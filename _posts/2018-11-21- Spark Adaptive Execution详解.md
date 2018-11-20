---
layout: post
title:  "SparkSQL Adaptive Execution详解"
date:   2018-11-18  +0800
categories:  spark
---


# SparkSQL Adaptive Execution详解


## 原有问题

在使用 Spark SQL 时，会通过 spark.sql.shuffle.partitions 指定 Shuffle 时 Partition 个数。通过数据量的大小，估计一个合理的Partition 个数。Partition大小的最佳实际是保持一块100M 左右（Adaptive Execution 功能中块大小默认64M），所以一般会用表的数据量均量和单块大小相除，得到一个Partition 的数量值，在这个基础上微调。
在这里存在一个特别头痛的问题，Partition数量值不好指定。

* Partition数值指定过大的话，说明每块的数量很少。例如，两张数量不大的表做 join，最终很容易会发现大量的 reduce 任务的执行，如果减小Partition数量值，会有一个明显的性能提升。
*  Partition数值指定过小的话，那么每块的数量会很多，这样内存压力会变大，因此可能会有许多 Spill 数据到磁盘的操作。存在数据倾斜的可能更高，难以发挥多机器的优势。

总而言之， 指定 Shuffle 时 Partition 个数的做法在生产环境中非常麻烦，没有办法保证不同的 Shuffle 场景都是一个最优的大小。

## 自适应 Partition 原理
从上述的问题中发现，问题都是由于partition的大小变动而引起的。同时计算partition 的个数实际上是依赖于一块partition的大小。
因此我们希望的是，partition的大小作为一个常量，partition的个数作为一个变量。
所有这里自适应的原理非常简单。

* 指定期望partition的大小
* 统计 shuffle 后partition的实际大小
* 已经期望partition的大小，对partition进行逻辑上的合并操作。
* 最终得到一个合理的 reduce 数量

## 代码分析
### EnsureRequirements

在说自适应 Partition 之前，我们需要明白什么地方存在自适应

每一个Spark Plan操作都会有一个Distribution对象。此对象是用来描述当前 Spark Plan 期望的输入数据（子 Spark Plan 的输出）的分布情况。如果子Spark Plan 的输出数据不满足条件，就会在当前父与子之间加入一个`ShuffleExchangeExec`用来确保数据满足输入条件。而Adaptive Execution 功能就是从这里增加到执行过程中的。

咱们到 EnsureRequirements 中找到下面这段代码

```
private def withExchangeCoordinator(
      children: Seq[SparkPlan],
      requiredChildDistributions: Seq[Distribution]): Seq[SparkPlan] = {
...
    ShuffleExchangeExec(targetPartitioning, child, Some(coordinator))
...
  }
```

其中` ShuffleExchangeExec(targetPartitioning, child, Some(coordinator))`这行代码中的`coordinator `是我们今天的主角，从这里Adaptive Execution 就开始准备控制 reduce 数量了。

### ExchangeCoordinator
首先看一下ExchangeCoordinator的构造函数的传参。


```
class ExchangeCoordinator(
    numExchanges: Int,
    advisoryTargetPostShuffleInputSize: Long,
    minNumPostShufflePartitions: Option[Int] = None)
  extends Logging 
```



参数意思分别为
* numExchanges 表明有多少 ShuffleExchangeExec会注册其中
* targetPostShuffleInputSize 期望 shuffle 后的块的大小，通过这个参数来确定 reduce 的 partition 数量
* minNumPostShufflePartitions 通过这个参数确保块的最小值

### coordinator流程

* 当我们开始执行物理计划的时候， coordinator中注册的 `ShuffleExchangeExec`会调用 `postShuffleRDD` 获得相应的shuffle 后的`ShuffledRowRDD`
* 如果coordinator 能够确定如何分配 shuffle 数据，那么会立刻返回shuffle 后的`ShuffledRowRDD`
* 如果 coordinator 无法确定的话，它会让注册的`ShuffleExchangeExec`提交它们的 shuffle 操作，然后基于 shuffle 结果的统计数据再去确定shuffle 后的`ShuffledRowRDD`

下面是如何确定`ShuffledRowRDD`的策略
* 首先获取指定的期望 shuffle 后 partition的大小，也就是构造函数中的targetPostShuffleInputSize
* 再获取shuffle 后实际的partition 大小的统计数据
* 依据期望大小，将实际的 shuffle 后的 partition 按照下标顺序打包成一个单独的逻辑 partition。逻辑 partition 的大小应该尽量接近但不超过期望的大小

举例：
我们有两个 stages 并且获得以下的 shuffle 数据统计

```
  stage 1: [100 MB, 20 MB, 100 MB, 10MB, 30 MB]
  stage 2: [10 MB,  10 MB, 70 MB,  5 MB, 5 MB]
```
假定目标输入大小为128M，那么我们将有4个shuffle 后数据，如下所示

```
   - reduce输入 partition 0: pre-shuffle partition 0 (size 110 MB)
   - reduce输入 partition 1: pre-shuffle partition 1 (size 30 MB)
   - reduce输入 partition 2: pre-shuffle partition 2 (size 170 MB)
   - reduce输入 partition 3: pre-shuffle partition 3 and 4 (size 50 MB)
```

## 使用方法

通过 spark.sql.adaptive.enabled=true 启用 Adaptive Execution 

通过 spark.sql.adaptive.shuffle.targetPostShuffleInputSize 可设置每个 Reducer 读取的目标数据量。


## 实践情况

分析上面来看，指定 Partition 的大小是比指定的Partition个数更为合理的选择。数据量不特别多时候性能提升明显。
但这并不代表Adaptive Execution就是一个万金油。因为指定了 Partition 大小意味着，数据量增多的情况下，Partition 个数会变多，这一点需要注意。
例如，我们在实际的 ETL 操作的过程中，中间有落盘操作，如果直接将Partition 落盘为一个实际块的话，会导致大量的文件的创建，而对于分布式文件系统，文件的创建通常是很重的操作，因此这并不是一个好的实现，必须要对块进行再次的处理后才能落盘。

## 参考
[Issue](https://issues.apache.org/jira/browse/SPARK-9850)

[Adaptive Execution 让 Spark SQL 更高效更智能](http://www.jasongj.com/spark/adaptive_execution/)

