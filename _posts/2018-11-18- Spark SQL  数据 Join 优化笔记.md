---
layout: post
title:  "SparkSQL Join优化笔记"
date:   2018-11-18  +0800
categories:  spark
---


# Spark SQL  数据Join优化笔记


## 问题描述
从开始定位SparkOutOfMemoryError这一异常之前，就有两个问题困扰着我。一个就是有 Spill 机制的存在，为什么还会 OOM 呢？这个问题在之前的文字中已经解释了根本原因。还有一个问题是数据量不大的情况为什么会进行 sort merge join 呢？这样我在小数据量的情况下，shuffle 操作带来了很多不必要的损耗。

## 代码分析

在SparkStrategy中的JoinSelection类中找到将关联 logical 转换成具体 Sort merge join的代码

```
  case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if RowOrdering.isOrderable(leftKeys) =>
        joins.SortMergeJoinExec(
          leftKeys, rightKeys, joinType, condition, planLater(left), planLater(right)) :: Nil
```

有疑问的代码并不在这里，而是下面这部分代码


```
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastBySizes(joinType, left, right) =>
        val buildSide = broadcastSideBySizes(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))

```

其中最为关键的是 canBroadcastBySizes(joinType, left, right)这个调用返回的是 false！首先我数据量并不大，设置可以说是非常小，这里竟然返回是 false！

接着往这个函数内部跟踪，发现我们提供的 RDD 的统计数据的计算竟然是 Int 最大值，换句话说我的数据量非常大。为什么 spark 采用 sort merge join 的原因找到了，但是为什么我们提供的 RDD 的统计值这么大呢？

这里我们要从 Dataset 的构造开始找起，由于我们的数据是在内存中的，而且有一个可用RDD 封装，于是我们调用的构造函数是

```
Dataset<Row> data = sparkSession.createDataFrame(rdd, schema);
```

因为传入 dataset 中的数据和操作，最后都会被转换为Logical Plan，包括 SQL 语句(当然最终构造会成为QueryExecution传入 Dataset 构造函数)。那么我的跟踪目标就是锁定在了 RDD 转换的 Logical Plan 上面。于是发现了一个 Logical RDD 对于统计数据计算的实现。

```
override def computeStats(): Statistics = Statistics(
    sizeInBytes = BigInt(session.sessionState.conf.defaultSizeInBytes)
  )
```

也就是说 RDD 获得的 Logical Plan 的统计值估算的是一个定值。那么当我用这个 Logical Plan 做 join 操作，并转换成为一个 Spark Plan 时，必然会被转换成为sort merge join 了。
在查看 dataset 构造方法并确认了相关函数后，我们采用了下面的构造方法。

```
Dataset<Row> data = sparkSession.createDataFrame(sparkRowList, schema);

```

函数名是一样的，但是第一个参数sparkRowList，是原始数据，不再是 RDD。这个对于的 logical plan 是 LocalRelation，其computeStats方法是依据数据量来计算的，join 也会顺利的采用了BroadcastHashJoin。


## 总结
Spark 内部逻辑复杂，一个参数类型的不同就会带来截然不同的性能。具我同事所说，前期也遇到过类似问题，同一个方法，参数数据本质一样，只是改变参数类型，效果完全不同。因此可见深入了解 Spark 内部实现的必要性。



