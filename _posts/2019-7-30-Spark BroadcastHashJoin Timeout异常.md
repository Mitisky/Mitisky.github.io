---
layout: post
title:  "Spark BroadcastHashJoin Timeout异常简单分析笔记"
date:   2019-7-30  +0800
categories:  spark异常
---

异常信息如下

```
java.util.concurrent.TimeoutException: Futures timed out after [300 seconds]
	at scala.concurrent.impl.Promise$DefaultPromise.ready(Promise.scala:219)
	at scala.concurrent.impl.Promise$DefaultPromise.result(Promise.scala:223)
	at org.apache.spark.util.ThreadUtils$.awaitResult(ThreadUtils.scala:201)
	at org.apache.spark.sql.execution.exchange.BroadcastExchangeExec.doExecuteBroadcast(BroadcastExchangeExec.scala:136)
```
这个问题在很久之前也出现过，但是当时并没有把超时当回事，因为这个出现的时候发生了OOM的异常，于是简单认为这里的超时是因为GC导致的。直到一个客户频繁发生超时异常，并且无OOM发生，于是感觉这个问题不是这么简单的。

## 问题分析
处理这样的问题必须深入代码了解具体做什么。因为这里BroadcastHashJoin的中异步计算抛出的异常，而其中的计算是对于小表进行broadcast时候超时的。
在`relationFuture`的实现中，发现了下面的代码

```
      val beforeCollect = System.nanoTime()
          // Use executeCollect/executeCollectIterator to avoid conversion to Scala types
          val (numRows, input) = child.executeCollectIterator()
          if (numRows >= 512000000) {
            throw new SparkException(
              s"Cannot broadcast the table with more than 512 millions rows: $numRows rows")
          }
          

```

代码中的`child.executeCollectIterator()`的非常扎眼，原来这里还有一次需要提交任务的计算。那如果此时spark系统繁忙的话，此任务迟迟得不到计算资源，因此非常容易超时。

## 解决方案
超时的操作是发生在join执行计划的构造过程中，也就是实际的join计算还没有发生呢。同时BroadcastHashJoin对于小表本身就是有大小限制的，因此可以理解为什么这里会设置一个超时限制。但是对于系统繁忙的时候这里存在一定超时的可能性。
因此最简单处理方式就是直接增加超时的时间，让这里的任务有足够时间去获得计算资源并完成。




