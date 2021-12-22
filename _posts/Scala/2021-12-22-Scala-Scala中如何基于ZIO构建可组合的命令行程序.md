---
title: Scala中如何基于ZIO构建可组合的命令行程序
categories:
- Scala
tags: [Scala]
topmost: true
---

* 目录
{:toc}

## 设计一个命令行程序

大多数命令行程序都是无状态的，这是理所当然的，因为它们可以很容易地集成到脚本中并通过shell管道链接。然而，对于本文，我们需要一个稍微复杂一点的程序。
让我们写一个SQL命令行程序。用户将通过文本命令与之交互，根据不同的SQL命令创建不同的程序命令以输出不同的字符，同时我们还希望可以循环输入。
对于这些问题中的每一个，我们将创建一个独立的模块，该模块依赖于其他模块，如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/36794cbb3457411e869c2b6c2d3823c7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5qKm5aKD6L-356a7,size_13,color_FFFFFF,t_70,g_se,x_16#pic_center)


## 基本程序

ZIO 应用程序的基本构建块是`ZIO[R, E, A]`类型，它描述了有效的计算，其中：

- `R` 是运行效果所需的环境类型
- `E` 是效果可能产生的错误类型
- `A` 是效果可能产生的值的类型

>  effect == “效果” == “副作用”

ZIO 是围绕对接口编程的想法而设计的。我们的应用程序可以分成更小的模块，任何依赖都表示为环境类型`R`的约束。首先，我们必须将`ZIO`的依赖添加到构建工具(maven)中：
```xml
<dependency>
    <groupId>dev.zio</groupId>
    <artifactId>zio_2.13</artifactId>
    <version>1.0.12</version>
</dependency>
-- 宏注解需要，仅zio1.0可用
<dependency>
    <groupId>dev.zio</groupId>
    <artifactId>zio-macros-core_2.13</artifactId>
    <version>0.6.2</version> 
</dependency>
```

我们将从一个简单的程序开始打印“hello world”并逐渐扩大。
```scala
object Main extends App {

  val program = console.putStrLn("hello world")

  def run(args: List[String]): ZIO[Console, Nothing, ExitCode] =
    for {
      r <- program.foldM(
      error => console.putStrLn(s"Execution failed with: $error") *> ZIO.succeed(1)
      , _ => ZIO.succeed(0)
    )(CanFail.canFail[IOException]).exitCode
  } yield r
}
```

为了让我们的生活更轻松，ZIO提供了`App`特性。我们需要做的就是实现`run`方法。在我们的例子中，我们可以忽略程序运行时使用的参数，并返回一个简单的程序打印到控制台。
该程序将在`DefaultRuntime`中运行，它提供具有阻塞(Blocking)、时钟(Clock)、控制台(Console)、随机数(Random)和系统(System)服务的默认环境。我们可以使用直接运行该方法，它将输出`hello world`。

## 自下而上构建程序

ZIO的核心设计目标之一是可组合性。它允许我们构建解决较小问题的简单程序并将它们组合成更大的程序。所谓的“自下而上”的方法并不是什么新鲜事 —— 它一直是航空业许多成功实施的支柱。孤立地测试和研究小组件，然后根据它们众所周知的特性，将它们组装成更复杂的设备，这样更便宜、更快、更容易。这同样适用于软件工程。当我们启动我们的应用程序时，根据输入的`SQL`不同，我们需要执行不同的逻辑，我们将输入定义为`CLICommand`：

```scala
sealed trait CLICommand

object CLICommand {
  case class ExecuteStatement(sql: String, args: Map[String,String] = Map.empty) extends CLICommand

  case class ExecuteDDL(sql: String, args: Map[String,String] = Map.empty) extends CLICommand

  case class ExecuteNativeSQL(sql: String, args: Map[String,String] = Map.empty) extends CLICommand

  case object Invalid extends CLICommand
}
```

接下来，我们将定义我们的第一个模块`CLICommandParser`，它将负责将用户输入转换为我们的域模型，即`CLICommand`。

```scala
trait CLICommandParser {
  // 其伴生对象只是service的容器
  val cliCommandParser: CLICommandParser.Service[Any]
}

object CLICommandParser {

  trait Service[R] {
    def parse(input: String): ZIO[R, Nothing, CLICommand]
  }
}
```

这遵循ZIO文档中的[模块模式](https://zio.dev/next/datatypes/contextual/#module-pattern-10)。`CLICommandParser`是一个模块，它只是 `CLICommandParser.Service` 的容器。ZIO有两种方式来编写服务。这里其实是使用第一种方式（但有些不同，主要是为了结构清晰），有一些样板内容。

> 注意：按照惯例，我们将保存引用的值命名为与模块相同的服务名称，仅第一个字母小写(`CLICommandParser`的引用命名为`cliCommandParser`)。这是为了在混合多个模块以创建环境时避免名称冲突。

服务只是一个普通的接口，定义了它提供的功能。

> 注意：按照惯例，我们将服务放置在模块的伴随对象中，并将其命名为`Service`。这是为了在整个应用程序中具有一致的命名方案`<Module>.Service[R]`。也是[zio-macros](https://github.com/zio/zio-macros)项目中一些宏所需要的结构。

接下来，我们可以定义我们的`Live`实现如下：

```scala
trait CLICommandParserLive extends CLICommandParser {

  val cliCommandParser: CLICommandParser.Service[Any] = new CLICommandParser.Service[Any] {
    def parse(input: String): UIO[CLICommand] = {
      UIO.succeed(parseArgs(input.split(" "))) map {
        case Some(conf) if conf.cmd == "stmt" => CLICommand.ExecuteStatement(conf.sql, conf.kwargs)
        case Some(conf) if conf.cmd == "ddl" => CLICommand.ExecuteDDL(conf.sql, conf.kwargs)
        case Some(conf) if conf.cmd == "sql" => CLICommand.ExecuteNativeSQL(conf.sql, conf.kwargs)
        case _ => CLICommand.Invalid
      }
    }
  }

  private def parseArgs(args: Seq[String]): Option[Config] = {
    Cli.parse(args.toArray).version("1.0-SNAPSHOT").withCommand(new bql) { bql =>
      bql.sql.tail match {
        // bsql --kvargs=key=1,key2=value2 select * from table
        case (head: String) :: (sql: List[String]) if head.startsWith("--kvargs") =>
          val propertiesStr = head.replaceFirst("--kvargs=", "")
          println(s"properties => $propertiesStr")
          val args = propertiesStr.split(",").foldLeft[mutable.Map[String, String]](new mutable.HashMap[String, String]())((map, e) => {
            if (e.contains("=")) {
              val Array(k, v) = e.split("=")
              map += k -> v
            } else {
              map += "args" -> e
            }
          }).toMap
          val conf = Config(cmd = if (args.isEmpty) "sql" else "stmt", sql = sql.mkString(" "), args)
          println(s"cli conf => $conf")
          conf
        // bsql select * from table
        case sql: List[String] if sql.isEmpty => Config(cmd = "sql", sql = sql.mkString(" "))
        case _ => Config("sql", "select 1")
      }
    }
  }
}
```

上面主要实现2个作用：`kvargs`不为空，则`cmd`设置为`stmt`，否则设置为`sql`，也即`cliCommandParser`服务根据输入`cmd`最终得到了不同的领域命令。

## 将纯函数提升到效果系统中

您会注意到`parse`表示包装纯函数的效果。有一些函数式程序员不会将此函数提升到效果系统中，以在您的代码库中明确区分纯函数和效果。
然而，这需要一支纪律严明且技能娴熟的团队，其好处值得商榷。虽然这个函数本身不需要被声明为有效果的，但通过让它这样使得我们可以在测试与这个模块协作的其他模块时模拟变得更简单。
通过构建较小的效果并根据需要将它们组合成较大的效果，渐进式设计应用程序也容易得多，而无需隔离副作用。这对习惯于命令式编程风格的程序员特别有吸引力。

## 将模块组合成一个更大的应用程序

以相同的方式，定义`Terminal`模块：

```scala
@accessible(">") // 用于初始启动的调用（实现将对功能的调用委托给环境），> 是一个object。
trait Terminal {
  val terminal: Terminal.Service[Any]
}

object Terminal {

  trait Service[R] {
    val getUserInput: ZIO[R, IOException, String]

    def display(cliContext: CLIContext): ZIO[R, IOException, Unit]
  }
}
```

但是，我们不想重新发明轮子。因此，我们将在我们的`TerminalLive`实现中重用内置的控制台服务。

```scala
trait TerminalLive extends Terminal {

  val console: Console.Service

  final val terminal = new Terminal.Service[Any] {

    lazy val getUserInput = console.getStrLn.orDie

    def display(cliContext: CLIContext) =
      for {
        _ <- console.putStr(TerminalLive.ANSI_CLEARSCREEN)
        _ <- console.putStrLn(cliContext.toString)
      } yield ()
  }
}

object TerminalLive {
  val ANSI_CLEARSCREEN: String = "\u001b[H\u001b[2J"
}
```

我们通过添加`Console.Service`类型的抽象值来定义依赖关系。编译器在构造使用`TerminalLive`实现的环境时将要求我们提供该服务。
请注意，这里我们再次依赖于约定，我们希望服务保存在以模块命名的变量中。实现非常简单，但问题是…我们如何测试它？我们可以使用`TestConsole`间接测试行为，但这很脆弱，在规范中不能很好地表达我们的意图。这就是[ZIO Mock framework](https://zio.dev/docs/howto/howto_mock_services)的用武之地。基本思想是表达我们对协作服务的期望，并最终构建该服务的模拟实现，该实现将在运行时检查我们的假设是否正确。

## 最终程序

```scala
object BitlapCLI extends App {

  val program: ZIO[RunSQL, Nothing, Unit] = {
    def exec(sqlContext: CLIContext): ZIO[RunSQL, Nothing, Unit] = {
      RunSQL.>.exec(sqlContext).foldM(
        _ => UIO.unit,
        newSqlContext => exec(newSqlContext)
      )
    }

    exec(CLIContext.apply(""))
  }

  def run(args: List[String]) =
    for {
      env <- prepareEnvironment
      out <- program.provide(env).foldM(
        error => console.putStrLn(s"Execution failed with: $error").exitCode
        , _ => UIO.succeed(0)
      )(CanFail.canFail[IOException]).exitCode
    } yield out

  private val prepareEnvironment =
    UIO.succeed(
      new ControllerLive
        with TerminalLive
        with RunSQLLive
        with CLICommandParserLive {
        override val console = Console.Service.live
      }
    )
}
```

我跳过了许多服务的细节，您可以在[bitlap](https://github.com/bitlap/bitlap/pull/109/files)存储库中查找完成的代码。我们不必显式地声明程序的完整环境类型。它只需要`RunSQL`，但一旦我们提供`RunSQLLive`，编译器就会要求我们提供`Terminal`和`Controller`服务。
当我们提供这些功能的`Live`实现时，它们又添加了自己的更多依赖项。通过这种方式，我们在Scala编译器的慷慨帮助下以增量方式构建最终环境，如果我们忘记提供任何必需的服务，它将输出可读且准确的错误。

本文参考 https://scalac.io/blog/write-command-line-application-with-zio
代码https://github.com/bitlap/bitlap/tree/cli-zio/integration/cli/src/main/scala/org/bitlap/cli （未真实使用，但可以运行了）

![在这里插入图片描述](https://img-blog.csdnimg.cn/13f3b59c557f40efa47eff4765379c90.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5qKm5aKD6L-356a7,size_20,color_FFFFFF,t_70,g_se,x_16)
