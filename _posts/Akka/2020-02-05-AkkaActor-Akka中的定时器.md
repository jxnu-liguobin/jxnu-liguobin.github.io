---
title: Akka中的定时器
categories:
  - AkkaActor
tags: [akka 2.5.23]
description: 本文是关于Akka的定时调度功能的基本使用
---

* 目录
{:toc}

# 基本使用

在Akka中，想实现定时调度有多种方法，分别是使用ActorSystem的scheduler、继承Timers特质并使用其TimerScheduler实例或使用第三方库[akka-quartz-scheduler](https://github.com/enragedginger/akka-quartz-scheduler)等。

> 依赖 "com.typesafe.akka" %% "akka-actor" % "2.5.23"

## 实现Timer特质

这是一个完整的例子，copy自官网

[定时器比较](https://dreamylost.cn/%E5%85%B6%E4%BB%96/%E5%85%B6%E4%BB%96-%E4%B8%89%E7%A7%8D%E5%AE%9A%E6%97%B6%E5%99%A8%E7%9A%84%E4%BD%BF%E7%94%A8.html)

```scala
package io.github.dreamylost.timer

import akka.actor.{ Actor, ActorSystem, Props, Timers }

import scala.concurrent.duration._

/**
 * actor自己处理定时
 *
 * @see https://doc.akka.io/docs/akka/current/actors.html#actors-timers
 * @author 梦境迷离
 * @since 2020-01-30
 * @version v1.0
 */
object TimerTest extends App {

  object MyActor {

    case object TickKey

    case object FirstTick

    case object Tick

  }

  class MyActor extends Actor with Timers {

    import MyActor._

    //启动计时器，在之后将“msg”发送一次给自己
    timers.startSingleTimer(TickKey, FirstTick, 500.millis)

    //TimerScheduler它不是线程安全的
    def receive = {
      case FirstTick =>
        // 2.6 startTimerWithFixedDelay
        println(s"=======${System.currentTimeMillis()}========")
        //timer 定时发送的消息 延迟
        timers.startPeriodicTimer(TickKey, FirstTick, 1.second)
      case Tick =>
    }
  }

  override def main(args: Array[String]): Unit = {
    import MyActor._
    val s = ActorSystem("testtimer")
    val actor = s.actorOf(Props(classOf[MyActor]))
    actor ! FirstTick

  }
}
```

Actor中为其自身安排定期任务或发送单个消息时，使用。

* 更加方便安全，生命周期与当前actor相关，actor重启或停止时，定时任务将取消
* TimerScheduler是非线程安全，不允许其他actor使用

## 使用ActorSystem的Scheduler

```scala
package io.github.dreamylost.timer

import akka.actor.{ Actor, ActorSystem, Props }

import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

/**
 *
 * @author liguobin@growingio.com
 * @version 1.0,2020/2/4
 */
object ScheduleTest extends App {

  val Tick = "tick"

  class TickActor extends Actor {
    def receive = {
      case Tick => println(System.currentTimeMillis())
    }
  }

  val system = ActorSystem()

  val tickActor = system.actorOf(Props(classOf[TickActor]))

  val cancellable = system.scheduler.schedule(Duration.Zero, 500.milliseconds, tickActor, Tick)

  //取消
  //  cancellable.cancel()
}
```

> 当ActorSystem终止时，将执行所有计划的任务，即任务可能在超时之前执行。

* 固定延迟 后续执行之间的延迟将始终（至少）为给定延迟。使用scheduleWithFixedDelay。
* 固定速率 随时间推移的执行频率将达到给定的时间间隔。使用scheduleAtFixedRate。

如果不确定要使用哪一个，则应选择scheduleWithFixedDelay。

* 对于固定延迟，如果执行时间较长或调度由于某种原因而延迟的时间超过指定时间，则它将无法补偿任务或消息之间的延迟。
* 对于固定速率，如果先前的任务执行时间过长，它将补偿后续任务的延迟。例如，如果给定的时间interval是1000毫秒，而一个任务需要200毫秒来执行，则下一个任务将安排在800毫秒后运行。
在这种情况下，实际执行间隔将与传递给scheduleAtFixedRate方法的间隔不同（scheduleAtFixedRate和scheduleWithFixedDelay是2.6的方法名称）
* 如果任务的执行时间比的时间长interval，则后续任务将在前一个任务完成后立即开始（不会有重复执行）
* 固定速率执行适用于对绝对时间敏感的重复活动或执行固定数量执行的总时间很重要的定期活动，例如倒计时定时器，它十秒钟每秒滴答一次。
* scheduleAtFixedRate长时间的垃圾回收暂停后，可能会导致计划的任务或消息突发，这在最坏的情况下可能会导致不希望的系统负载。scheduleWithFixedDelay通常是首选。

## 使用akka-quartz-scheduler

前面两种均为高吞吐量而设计，在精确性上以及长任务上有妥协，默认最长是八个月（Int max值）左右。且由于使用了Hashed Wheel Timer，所以精度上也有不足。因此，对于需要高精度触发或长期调度的程序，则不应该选择Akka自带的定时调度功能。

第三方框架quartz是一个更加复杂强大的任务调度框架，akka中的使用如下

> 依赖 "com.enragedginger" %% "akka-quartz-scheduler" % "1.8.1-akka-2.5.x"


```scala
package io.github.dreamylost.quartz

import akka.actor.{ Actor, ActorSystem, Props }
import com.typesafe.akka.extension.quartz.QuartzSchedulerExtension

/**
 * akka quartz 拓展的更完善的定时处理
 *
 * @see https://github.com/enragedginger/akka-quartz-scheduler
 * @author 梦境迷离
 * @since 2020-01-30
 * @version v1.0
 */
object QuartzSchedulerActor extends App {

  case object Tick

  val system = ActorSystem("user-system")
  val cleaner = system.actorOf(Props[CleanupActor])
  val scheduler = QuartzSchedulerExtension(system)
  val s = QuartzSchedulerExtension(system).schedule("Every30Seconds", cleaner, Tick)
  println(s)

  //配置文件在application.conf
  class CleanupActor extends Actor {
    override def receive: Receive = {
      case Tick =>
        println("tick ...")
    }
  }

}
```

application.conf
```
akka {
  quartz {
    schedules {
      Every30Seconds {
        description = "A cron job that fires off every 30 seconds"
        expression = "*/30 * * ? * *" #每30秒触发一次
        # calendar = "OnlyBusinessHours" #排除了从上午8点到下午6点之间触发的触发器
      }
    }
  }
}

```

查看默认的定时器的实现，在Akka actor源码中，查看reference.conf（以Akka 2.5.23为例）
```
... 
scheduler {
    # The LightArrayRevolverScheduler is used as the default scheduler in the
    # system. It does not execute the scheduled tasks on exact time, but on every
    # tick, it will run everything that is (over)due. You can increase or decrease
    # the accuracy of the execution timing by specifying smaller or larger tick
    # duration. If you are scheduling a lot of tasks you should consider increasing
    # the ticks per wheel.
    # Note that it might take up to 1 tick to stop the Timer, so setting the
    # tick-duration to a high value will make shutting down the actor system
    # take longer.
    tick-duration = 10ms

    # The timer uses a circular wheel of buckets to store the timer tasks.
    # This should be set such that the majority of scheduled timeouts (for high
    # scheduling frequency) will be shorter than one rotation of the wheel
    # (ticks-per-wheel * ticks-duration)
    # THIS MUST BE A POWER OF TWO!
    ticks-per-wheel = 512

    # This setting selects the timer implementation which shall be loaded at
    # system start-up.
    # The class given here must implement the akka.actor.Scheduler interface
    # and offer a public constructor which takes three arguments:
    #  1) com.typesafe.config.Config
    #  2) akka.event.LoggingAdapter
    #  3) java.util.concurrent.ThreadFactory
    implementation = akka.actor.LightArrayRevolverScheduler

    # When shutting down the scheduler, there will typically be a thread which
    # needs to be stopped, and this timeout determines how long to wait for
    # that to happen. In case of timeout the shutdown of the actor system will
    # proceed without running possibly still enqueued tasks.
    shutdown-timeout = 5s
  }
... 
```

该配置中对使用场景也做了简单的说明。可以看到字段implementation即是定时器的具体实现。目前默认的是LightArrayRevolverScheduler。
该类与Netty的HashedWheelTimer相似，都是基于一种叫哈希轮的数据结构实现的定时任务调度算法。
感兴趣可以看看整理的一个PPT [定时轮](https://github.com/jxnu-liguobin/scala-examples/blob/master/scala-other/src/main/scala/io/github/dreamylost/timer/TimingWheels.ppt)