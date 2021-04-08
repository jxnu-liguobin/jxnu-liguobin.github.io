---
title: Scala2中宏的实际应用
categories:
- Scala
tags: [宏编程]
description: 使用Scala2中的宏生成指定接口的子类的对象来减少冗余代码。
---


虽然Scala3已经重写了宏，但是考虑到Scala3的变化实在太大，并且不兼容Scala2，所以可预见的是在相当一段时间内，企业级应用并不会升级到Scala3。
目前使用的最新版本一般是2.13.3。我们推荐大家学习新技术，但是考虑到技术的复杂度和新人上手的难度，一般仅会在基础库内部使用，对外并不暴露。且会尽可能不去使用Scala中比较难学的框架或技术。但有时使用一些技术能极大的增加开发效率或更有利于构建可拓展和可伸缩的系统，这时引入一些复杂的技术也许是值得的。
我们使用宏最广泛的地方是实现Java protobuf与case class的转化，该实现其实很复杂，这里我介绍一个刚刚实现的，更容易上手的简单例子。

这个例子是使用宏生成指定接口的子类的对象。看上去很难，其实很简单。只需要了解宏插值器即可。

需求是使用模板（只是一个包含字符串占位符的文件，不是严格意义上的模板）自动生成crud类。包括service，serviceImpl，repository，repositoryImpl，table(scalikejdbc的syntax)，domain，gRpc dto，gRpc api。
它们每个类都有一个CodeGenerator的实例，分别生成不同的代码。每个`Generator`实现如下：

```scala
object GrpcServiceGenerator extends CodeGenerator {

  override val templateName = "service/service_grpc"//模板的路径

  override def generate(args: TemplateArgs): Path = {
    val result = explain(args)
    Helper.writeSourceResult(result)
  }

  override def remove(args: TemplateArgs): Path = {
    val result = explain(args)
    Helper.deleteSourceCode(result.file)
  }

  override def explain(args: TemplateArgs): SourceResult = {
    //handleSourceResult是默认的通用处理逻辑，能处理大多数的模板渲染。  
    handleSourceResult(args.name, args.rootPackage, args.rootPath, args.isDataCenter)
  }

}
```

这样为每个类都写一个`CodeGenerator`的子类即可。每个`CodeGenerator`都有一个模板路径和`generate`，`remove`，`explain`三个方法，很好理解它们的含义。如果我们就这样实现，也能达到我们的目的，同时使得数据和模板的分离，后续只需继续添加模块，增加子类即可。

但是，这里有个待优化的地方，每个`CodeGenerator`基本相同，我们都要编写？麻不麻烦？
那，使用宏替代手动编写`CodeGenerator`，怎么解决模板的`explain`逻辑可能不同呢？比如，现在我的gRpc渲染需要特殊处理：
```scala
object GrpcGenerator extends CodeGenerator {

  override val templateName = "grpc/grpc_proto"

  override def generate(args: TemplateArgs): Path = {
    val result = explain(args)
    Helper.writeSourceResult(result)
  }

  override def remove(args: TemplateArgs): Path = {
    val result = explain(args)
    Helper.deleteSourceCode(result.file)
  }

  override def explain(args: TemplateArgs): SourceResult = {
    import args._
    //无法使用默认的渲染逻辑，需要自己编写
    val grpcName = getGrpcName(name)
    val grpcPackage = getGrpcPackage(rootPackage)
    val arguments = Helper.toRendererArguments(rootPackage, name, "grpcPackage" -> grpcPackage, "grpcName" -> grpcName)
    val code = TemplateRenderer.render(templateName, arguments)
    val path = Helper.resolvePath(getGrpcRootPackage(rootPackage), rootPath, "v1")
    val sourcePath = Helper.getProtoSourceCodePath(path, name)
    SourceResult(sourcePath, code)
  }

  private[grpc] def getGrpcRootPackage(rp: String): String = {
    "grpc" + rp.substring(rp.lastIndexOf('.'))
  }

  private[grpc] def getGrpcPackage(rp: String): String = {
    rp.substring(rp.indexOf('.') + 1)
  }

  private[grpc] def getGrpcName(n: String): String = {
    n.substring(0, 1).toUpperCase + n.substring(1)
  }

}
```

1.解决第一个问题。使用宏来避免编写重复的`Generator`，其实也很简单，如下：
```scala
object CodeGenerateBuilder {

  import scala.language.experimental.macros //Scala2宏是不稳定的，必须导入才能使用。
  import scala.reflect.macros.blackbox

  //一个Generator需要一些固定的参数来初始化，分别是Generator名称，模板路径，生成crud文件的后缀，生成crud文件的包名(不包含组织)
  def apply[G <: CodeGenerator](name: String, template: String, classSuffix: String, codePath: String): G = macro applyImpl

  //必须传入context，这里使用黑盒的context，该方法的参数名与apply参数名相同，且使用c.Tree包裹，表达式使用c.Expr包裹。
  //宏入门可以看看https://dreamylost.cn/scala/Scala-marco%E4%BB%8B%E7%BB%8D.html#%E9%BB%91%E7%9B%92%E4%BE%8B%E5%AD%90
  //暂时不妨把Tree就当做Scala的ast。
  def applyImpl(c: blackbox.Context)(name: c.Tree, template: c.Tree, classSuffix: c.Tree, codePath: c.Tree): c.universe.Tree = {
    import c.universe._
    //表示该值是一个类型名称，注意，不能直接使用字符串。
    val className = TypeName(name.toString())
    q"""
      import io.growing.boxer.generator.{ Helper, CodeGenerator, SourceResult }
      import io.growing.boxer.generator.TemplateArgs
      
      import java.nio.file.Path

      class $className extends CodeGenerator {
    
        override val templateName = $template
        
        override val classSuffix = $classSuffix
        
        override val codePath = $codePath
      
        override def generate(args: TemplateArgs): Path = {
          val result = explain(args)
          Helper.writeSourceResult(result)
        }
      
        override def remove(args: TemplateArgs): Path = {
          val result = explain(args)
          Helper.deleteSourceCode(result.file)
        }
      
        override def explain(args: TemplateArgs): SourceResult = {
          handleSourceResult(args)
        }
    }
    new $className //生成类后，new匿名对象返回。
    """
  }
}
```
这样我们就能自动生成`CodeGenerator`的任何子类的对象了。由于这里使用宏，我们就可以不使用`object`定义`CodeGenerator`的实现，因为宏的生成是在编译期间，且只会在调用宏时生成对象。需要注意的是，该对象的运行时的`class name`并不等于上面的宏参数传入的`className`，它的运行时类名类似为`u0022ServiceImplGenerator$u0022$1`这种，两边多了一些怪东西，暂时没有深究是啥。

因为宏是编译期间的技术，所以你不可能在定义宏的时候同时使用宏，这绝对会出现错误，幸好编译器会提示你。我们需要将宏实现放到一个独立的项目或模块，同时依赖该模块。在`build.sbt`中，需要配置`dependsOn`。

接下来就可以调用宏了，代码如下：
```scala
val GrpcServiceGenerator = CodeGenerateBuilder[CodeGenerator]("GrpcServiceGenerator", "service/service_grpc", "Service", "service")
```
> 再次注意：除非是在测试模块中，否则绝对无法在定义宏的模块调用宏。

这里`CodeGenerateBuilder()`将调用`apply`方法，根据上面可知，`apply`方法对应的宏实现是`applyImpl`方法。

最后，我们使用宏生成的GrpcServiceGenerator对象来渲染模块：
```scala
val sourceResult = GrpcServiceGenerator.explain(args)
```

2.解决第二个问题。上面所说的如果想定义自己的`explain`方法怎么办？也很简单，我们给宏传入函数即可！
在`CodeGenerateBuilder`中再定义一个`apply`的重载方法，并调用第二个宏实现`applyImpl2`方法，如下
```scala
def apply[G <: CodeGenerator](name: String, template: String, classSuffix: String, codePath: String, fun: TemplateArgs ⇒ SourceResult): G = macro applyImpl2
//需要注意的是，宏不能使用命名参数和可选参数，这将无法编译。
def applyImpl2(c: blackbox.Context)(name: c.Tree, template: c.Tree, classSuffix: c.Tree, codePath: c.Tree, fun: c.Expr[TemplateArgs ⇒ SourceResult]): c.universe.Tree = {
    import c.universe._
    val className = TypeName(name.toString())
    q"""
      import io.growing.boxer.generator.{ Helper, CodeGenerator, SourceResult }
      import io.growing.boxer.generator.TemplateArgs
      
      import java.nio.file.Path

      class $className extends CodeGenerator {
    
        override val templateName = $template
        
        override val classSuffix = $classSuffix
        
        override val codePath = $codePath
      
        override def generate(args: TemplateArgs): Path = {
          val result = explain(args)
          Helper.writeSourceResult(result)
        }
      
        override def remove(args: TemplateArgs): Path = {
          val result = explain(args)
          Helper.deleteSourceCode(result.file)
        }
      
        override def explain(args: TemplateArgs): SourceResult = {
          //两个宏的唯一不同是这里我们使用传进来的fun函数，而不是默认的handleSourceResult方法。
          $fun(args)
        }
    }
    new $className
    """
  }
```

调用宏与上面相同，只不过我们现在需要编写一个函数作为参赛传进去：
```scala
def customHandle(args: TemplateArgs): SourceResult = {
    import args._

    def getGrpcRootPackage(rp: String): String = {
    "grpc" + rp.substring(rp.lastIndexOf('.'))
    }

    def getGrpcPackage(rp: String): String = {
    rp.substring(rp.indexOf('.') + 1)
    }

    def getGrpcName(n: String): String = {
    n.substring(0, 1).toUpperCase + n.substring(1)
    }

    val scopeId = if (args.isDataCenter) "data_center_id" else "project_id"
    val scope = if (args.isDataCenter) "DataCenter" else ""
    val grpcName = getGrpcName(name)
    val grpcPackage = getGrpcPackage(rootPackage)
    val arguments = Helper.toRendererArguments(rootPackage, name,
    "grpcPackage" -> grpcPackage,
    "grpcName" -> grpcName,
    "scopeId" -> scopeId,
    "scope" -> scope)
    val code = TemplateRenderer.render("grpc/grpc_proto", arguments)
    val path = Helper.resolvePath(getGrpcRootPackage(rootPackage), rootPath, "v1")
    val prefix = if (isDataCenter) "data-center-" else ""
    val sourcePath = Helper.getProtoSourceCodePath(path, prefix + name.toLowerCase())
    SourceResult(sourcePath, code)
}

val GrpcGenerator = CodeGenerateBuilder[CodeGenerator]("GrpcGenerator", "grpc/grpc_proto", "", "", customHandle(_))
```

使用宏生成的`GrpcGenerator`对象与之前相同：
```scala
val sourceResult = GrpcGenerator.explain(args)
```

这样下来我们通常不需要编写任何多余代码，仅当`explain`是特别逻辑时才需要自己编写一个函数给宏调用。**代码还有待优化，仅供参考。**

有的人可能会问，既然使用宏自动生成，为什么不直接生成crud类，还搞这么多的`Generator`干嘛？那说明你还是年轻了。哈哈。

前面我们讲到，模板本身只是一个带有占位符的字符串，这意味着修改模板将会极其容易。如果我们完全使用宏生成，岂不是每次都得改宏实现，所有需要新增或修改模板的人都要知道如何编写宏？很显然，调试宏是麻烦的（`reify`可以打印`Tree`），而编写宏也不是每个人都熟练的，这个维护成本难以估计，为了达到平衡，适当使用才是最佳的。
还有一个问题是，直接生成crud等类，需要很多参数，这在调用宏时并不方便。而使用`Generator`既容易理解，更重要的是可以做一些预处理。