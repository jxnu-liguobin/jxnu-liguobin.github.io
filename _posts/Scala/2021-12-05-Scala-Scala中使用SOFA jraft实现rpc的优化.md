---
title: Scala中使用SOFA jraft实现rpc的优化
categories:
- Scala
tags: [Scala]
topmost: true
---

* 目录
{:toc}


## 背景

项目基于sofa jraft构建，顺便使用了其自带的rpc服务，协议使用`protobuf`，使用jraft创建一个rpc服务`RaftRpcServerFactory.createRaftRpcServer(serverId.getEndpoint)`，并
新增的rpc接口，这通常需要定义自己的Processor并继承`com.alipay.sofa.jraft.rpc.RpcRequestProcessor`，然后创建一个实例，使用`rpcServer.registerProcessor`将实例暴露的rpc注册到`RpcServer`中。这里的待改善问题是当我们的接口变多时，Processor并不容易管理，同时个人认为，定义Processor的过程是繁琐和枯燥的，几乎都是一个模板。而我很懒哈哈，不想一个个写，除了业务，不想来回写。下面看看怎么简化这个流程吧。

下面使用[bitlap](https://github.com/bitlap/bitlap)的一个创建会话的Processor来说明

```scala
// networkService是业务逻辑所在类
class OpenSessionProcessor(private val networkService: NetworkService, executor: Executor = null)
  extends BitlapRpcProcessor[BOpenSessionReq](executor, BOpenSessionResp.getDefaultInstance) { // BitlapRpcProcessor将RpcRequestProcessor的handleRequest方法处理分为正常处理processRequest和processError异常处理

  override def processRequest(request: BOpenSessionReq, done: RpcRequestClosure): Message = {
    import scala.jdk.CollectionConverters.MapHasAsScala
    val username = request.getUsername
    val password = request.getPassword
    val configurationMap = request.getConfigurationMap
    val sessionHandle = networkService.openSession(username, password,
      configurationMap.asScala.toMap)
    BOpenSessionResp.newBuilder()
      .setSessionHandle(sessionHandle.toBSessionHandle())
      .setStatus(success()).build()
  }

  override def processError(rpcCtx: RpcContext, exception: Exception): Message = {
    BOpenSessionResp.newBuilder().setStatus(error(exception)).build()
  }

  override def interest(): String = classOf[BOpenSessionReq].getName
}
```

这里面其实没什么代码，主要是代码在`networkService`中，但是这里需要创建`OpenSessionProcessor`这个类，算是个模板类，这里虽然使用`BitlapRpcProcessor`做了一次优化，但是效果一般。（毕竟我是个懒人，能少写一行代码都是好的，哈哈）

`BitlapRpcProcessor`的定义如下：

```scala
abstract class BitlapRpcProcessor[T <: Message](executor: Executor, override val defaultResp: Message)
  extends RpcRequestProcessor[T](executor, defaultResp)
    with ProcessorHelper with LazyLogging {

  override def handleRequest(rpcCtx: RpcContext, request: T) {
    try {
      val msg = processRequest(request, new RpcRequestClosure(rpcCtx, this.defaultResp))
      if (msg != null) {
        rpcCtx.sendResponse(msg)
      }
    } catch {
      case e: Exception =>
        logger.error(s"handleRequest $request failed", e)
        rpcCtx.sendResponse(processError(rpcCtx, e))
    }
  }

  def processError(rpcCtx: RpcContext, exception: Exception): Message
}
```
## 使用宏方法进行优化

根据如上所示，我们的目的是通过一种方法宏的处理，在不创建新的类文件的情况下创建`com.alipay.sofa.jraft.rpc.RpcRequestProcessor`的实例。这个宏定义为`Processable`

考虑：
1. 泛型、类型安全
2. 业务处理
3. 自定义拓展

一般Processor是使用`RpcRequestProcessor`的构造函数派生子类。这里的2个构造函数分别是执行请求的和protobuf `Message`类型的响应消息的默认实例
```java
    public RpcRequestProcessor(Executor executor, Message defaultResp) {
        super();
        this.executor = executor;
        this.defaultResp = defaultResp;
    }
```

### Processable宏的初步设计

下面使用**黑盒宏**来实现。

**宏的定义**

```scala
  def apply[Req <: Message, Service, Executor <: java.util.concurrent.Executor]
      (service: Service, defaultResp: Message, executor: Executor)
      (
      processRequest:   (Service, RpcRequestClosure, Req) ⇒ Message,
      processException: (Service, RpcContext, Exception) ⇒ Message): CustomRpcProcessor[Req] 
      = macro ProcessableMacro.processorImpl[Req, Service, Executor]
```

泛型说明：
- `Req` protobuf定义的类型，用于request的消息类型，必须是`com.google.protobuf.Message`的子类。
- `Service` 用户自定义的服务接口，用于处理业务逻辑，可以为任意类型。
- `Executor` 用于传递给`RpcRequestProcessor`的构造函数，必须是`java.util.concurrent.Executor`的子类。

参数说明：
- `processRequest:   (Service, RpcRequestClosure, Req) ⇒ Message` 一个处理请求的函数，可以实现任意业务逻辑，最重要的参数。
- `processException: (Service, RpcContext, Exception) ⇒ Message` 一个处理异常的函数。
- `service:          Service` 操作业务所需要的实例对象。
- `defaultResp:      Message` protobuf定义的类型的默认实例，用于传递给`RpcRequestProcessor`的构造函数。
- `executor:         Executor` 用于传递给`RpcRequestProcessor`的构造函数，必须是`java.util.concurrent.Executor`的子类。

> 返回的`Message`通常是自己定义的用于响应的protobuf对象的子类

初步的设计照搬了`OpenSessionProcessor`的实现，只是使用宏创建类的实例对象，所以乍一看参数很多，不便使用。
考虑到大多数情况下并不需要这么灵活的定义，还是可以再简化一下宏定义的。先看protobuf例子。

**示例**

对于现有protobuf文件：
```protobuf
message BOpenSession {
    message BOpenSessionReq {
        string username = 1;
        string password = 2;
        map<string, string> configuration = 3;
    }
    message BOpenSessionResp {
        string status = 1;
        map<string, string>  configuration = 2;
        string session_handle = 3;
    }
}
```

使用`Processable`宏:
```scala
  val openSession = Processable[BOpenSessionReq, NetService, Executor](
    (service, rpcRequestClosure, req) => {
      import scala.jdk.CollectionConverters.MapHasAsScala
      val username = req.getUsername
      val password = req.getPassword
      val configurationMap = req.getConfigurationMap
      val ret = service.openSession(username, password, configurationMap.asScala.toMap)
      BOpenSessionResp.newBuilder().setSessionHandle(ret).build()
    },
    (service, rpcContext, exception) => {
      BOpenSessionResp.newBuilder().setStatus(exception.getLocalizedMessage).build()
    },
    new NetService, BOpenSessionResp.getDefaultInstance, null
  )
```

### Processable宏的改进 一

本次改进不是为了拓展，而是为了在一般情况下，宏方法容易使用，目标当然是减少参数传递，但是哪些参数可以减少呢？
下面列了2个参数一般都是默认值，所以就可以简化它。需要注意，这里简化并不是就不支持传递这些参数了，因为Scala object的`apply`方法是能重载的，所以是共存的。
为什么我们要使用`apply`方法？object的`apply`能使得我们使用`Processable[T](xx)`的形式来调用，而不需要`Processable[T].toProcessor(xx)`，是不是更清爽了，哈哈。

**宏的定义**

```scala
  def apply[Service, Req <: Message, Resp <: Message](service: Service)
      (
      processRequest:   (Service, RpcRequestClosure, Req) ⇒ Message,
      processException: (Service, RpcContext, Exception) ⇒ Message): CustomRpcProcessor[Req] = 
      macro ProcessableMacro.processorWithDefaultRespImpl[Service, Req, Resp]
```

- `executor` 直接使用`null`，不支持传入自定义参数。
- `defaultResp` 直接使用`Resp.getDefaultInstance`创建默认对象 ，不支持传入自定义参数。

与第一次定义很类似，仅是省略了`executor`和`defaultResp`参数，但是泛型参数都保留了，这是为了类型安全。这次由于没有传`defaultResp`，所以需要使用泛型`Resp`指定默认值的类型，其实内部仍是是使用了`getDefaultInstance`。这里也能观察到，灵活性和便捷性是不可都得的。

**示例**

```scala
  val openSession = Processable[NetService, BOpenSessionReq, BOpenSessionResp](new NetService)(
    (service, _, req) => {
      import scala.jdk.CollectionConverters.MapHasAsScala
      val username = req.getUsername
      val password = req.getPassword
      val configurationMap = req.getConfigurationMap
      val ret = service.openSession(username, password, configurationMap.asScala.toMap)
      BOpenSessionResp.newBuilder().setSessionHandle(ret).build()
    },
    (_, _, exception) => {
      BOpenSessionResp.newBuilder().setStatus(exception.getLocalizedMessage).build()
    }
  )
```

### Processable宏的改进 二

到上面为止，其实差不多可以了，再次简化就只能是连`service`都不传了。可以做到吗？答案是肯定的，我们可以用运行时反射来创建对象。
虽然之前都做的是编译期反射，这次结合编译期和运行期来看看具体应用。

- 为`Service`泛型反射出对象，不再需要传 `service` 参数
- 仅支持非抽象类且必须含有默认无参构造函数

Scala如何反射一个类来创建类的对象呢？

我们定义一个`Creator`，通过参数`T:WeakTypeTag`反射。`WeakTypeTag`由编译器创建，使用`T`的`tpe`属性可以反射`T`。
`WeakTypeTag`力求尽可能是具体的类型，即如果`TypeTag`可用于引用的类型参数或抽象类型，则它们用于将具体类型嵌入`WeakTypeTag`。
否则`WeakTypeTag`将包含对抽象类型的引用。当人们期望`T`可能是部分抽象的，但需要特别小心来处理这种情况时，这种行为是有用的。
但是，如果`T`应该是完全已知的，则应该使用`TypeTag`，它静态地保证了这个属性。`TypeTag`它不包含任何对未解析类型参数或抽象类型的引用。

Scala的抽象语法树除了三个字段外，是不可变的。这三个就是`symbol`，`pos`，`tpe`。对于编译器而言，类型检查不是一步到位的，所以`pos`，`tpe`，`symbol`这种属性，可能在某阶段是没有值的。
而在typechecked后就能获取到实际值。这在编译期反射中很有用。

```scala
class Creator[T: WeakTypeTag] {

  def createInstance(args: AnyRef*)(ctor: Int = 0): T = {
    val tt = weakTypeTag[T]
    currentMirror.reflectClass(tt.tpe.typeSymbol.asClass).reflectConstructor(
      tt.tpe.members.filter(m =>
        m.isMethod && m.asMethod.isConstructor
      ).iterator.toSeq(ctor).asMethod
    )(args: _*).asInstanceOf[T]
  }
}
```

有了反射功能，我们只需要将`NetService`传入作为`Service`的类型，在宏中使用运行时反射构造对象即可。
```scala
  // 这里即使与上面相比少了个service参数，但是因为编译器识别时有点问题，会和上面那个重载的apply定义冲突，所以把泛型的位置改了下，把Service泛型放到最后。
  val openSession = Processable[BOpenSessionReq, BOpenSessionResp, NetService](
    (service, rpc, req) => {
      import scala.jdk.CollectionConverters.MapHasAsScala
      val username = req.getUsername
      val password = req.getPassword
      val configurationMap = req.getConfigurationMap
      val ret = service.openSession(username, password, configurationMap.asScala.toMap)
      BOpenSessionResp.newBuilder().setSessionHandle(ret).build()
    },
    (service, rpc, exception) => {
      BOpenSessionResp.newBuilder().setStatus(exception.getLocalizedMessage).build()
    }
  )
```
到目前为止，再一般情况下，我们甚至只需要提供2个函数就能实现任意Processor的定义了，再也不用创建类了，哈哈。


宏的实现是比较难懂的，这里没有贴代码，感兴趣的可以看看源码。https://github.com/jxnu-liguobin/scala-macro-tools/tree/master/src/main/scala/io/github/dreamylost/sofa。

> 如果对你有帮助可以点个star。


