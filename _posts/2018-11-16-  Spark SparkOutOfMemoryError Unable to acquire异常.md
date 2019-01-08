---
layout: post
title:  " Spark SparkOutOfMemoryError Unable to acquire异常"
date:   2018-11-16  +0800
categories:  spark
---



异常信息如下

```
Caused by: org.apache.spark.memory.SparkOutOfMemoryError: Unable to acquire 28 bytes of memory, got 0
	at org.apache.spark.memory.MemoryConsumer.throwOom(MemoryConsumer.java:157)
	at org.apache.spark.memory.MemoryConsumer.allocatePage(MemoryConsumer.java:119)
	at org.apache.spark.util.collection.unsafe.sort.UnsafeExternalSorter.acquireNewPageIfNecessary(UnsafeExternalSorter.java:383)
	at org.apache.spark.util.collection.unsafe.sort.UnsafeExternalSorter.insertRecord(UnsafeExternalSorter.java:407)
	at org.apache.spark.sql.execution.UnsafeExternalRowSorter.insertRow(UnsafeExternalRowSorter.java:135)
```

这是一个内存溢出的问题，但不是严格意义上 JVM 无法分配的内存不足，而是 Spark 计算得出的内存不足。
目前在一个 shuffle 场景会遇到这个异常。具体场景如下，数据经过shuffle 后的partition很多，然后后续调用了一个剧烈的Coalesce操作，比如Coalesce(1)。

解决这个问题需要对UnsafeExternalSorter和Spark内存管理有一定了解。我们知道UnsafeExternalSorter对象会在内存不足的时候主动 spill 数据到磁盘来释放内存，但是在这里仿佛失效了。实际上 spill 操作并没有失效，而是由于 spark 避免其它调用者对数据操作，而保留了一个内存页。

```
public long spill() throws IOException {
      synchronized (this) {
       ...
          for (MemoryBlock page : allocatedPages) {
            if (!loaded || page.pageNumber !=
                    ((UnsafeInMemorySorter.SortedIterator)upstream).getCurrentPageNumber()) {
              released += page.size();
              freePage(page);
            } else {
              lastPage = page;
            }
          }
          allocatedPages.clear();
        }
        ...
      }
    }
```

可以看到在 spill 操作中，如果当前数据已经加载同时当前页号和 iterator 的数据来源页号是一样，那么就将这个页设为最后页，这个页并不会释放。
在UnsafeExternalSorter对象构造函数中注册了一个 clean 方法


```
// Register a cleanup task with TaskContext to ensure that memory is guaranteed to be freed at
// the end of the task. This is necessary to avoid memory leaks in when the downstream operator
// does not fully consume the sorter's output (e.g. sort followed by limit).
taskContext.addTaskCompletionListener(context -> {
  cleanupResources();
});

```

这意味着如果不主动调用 cleanupResource 方法，那么sort对象中的最后保留的 page 内存页将不会被释放。

没有Coalesce操作的时候，每个 partition 是一个task来计算，此时内存并不会有很大问题，假设 JVM 中32个 Cores 在计算， Page 默认大小为128M的情况下，内存损耗顶峰也不到4G，这个数量并不算多。但是Coalesce（1）彻底不同了，会让 N 块 partitions 放到一个 Task 中来计算。那导致内存顶峰大小为 N*128M，而 shuffle 中的 N 值默认的情况为200，实际可能更大，那么就很容易内存溢出了。

## 问题解决
首先从计算逻辑上分析，Coalesce（1）这个操作是否为必要操作？如果不是，那么尽快减少使用，Coalesce不会把数据块做 shuffle 操作，但是会让数据块在一个 Task 执行，同时后续的计算成为单块的任务了。
这里使用Coalesce（1）是因为数据量不大，但是 shuffle 后数据量太多，导致 task 计算次数过多，所有使用 Coalesce 将数据拉成1块，但是没想到还有这么一茬。目前我们去掉了Coalesce，并从 join 逻辑上面改变 join 方式，彻底解决了这个问题。

最后，我们可以通过下面的配置方法来解决，但是还是存在性能损耗，并且不能根治
1. 减小 Page 默认大小，参数为 spark.buffer.pageSize，比如设为32M，这样减少内存峰值。
2. 减少 partition 数量，比如 shuffle 的数量，或者从数据源入手减少数量等等。






