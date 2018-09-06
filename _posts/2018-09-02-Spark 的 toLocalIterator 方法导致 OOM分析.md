---
layout: post
title:  "Spark driver 调用 toLocalIterator 方法导致 OOM分析"
date:   2018-09-02  +0800
categories: spark
---


# Spark driver 调用 toLocalIterator 方法导致 OOM分析
目前由于计算结构的问题，导致 spark 的计算结果需要返回 driver 进行落盘，导致数据量一大，很容易导致 OOM。下面分析一下问题的原因和处理办法。
首先看一下toLocalIterator 的代码实现。

```
 /**
   * Return an iterator that contains all of the elements in this RDD.
   *
   * The iterator will consume as much memory as the largest partition in this RDD.
   *
   * @note This results in multiple Spark jobs, and if the input RDD is the result
   * of a wide transformation (e.g. join with different partitioners), to avoid
   * recomputing the input RDD should be cached first.
   */
def toLocalIterator: Iterator[T] = withScope {
    def collectPartition(p: Int): Array[T] = {
      sc.runJob(this, (iter: Iterator[T]) => iter.toArray, Seq(p)).head
    }
    partitions.indices.iterator.flatMap(i => collectPartition(i))
  }

```

在注释中明确说了最多消耗的内存大小，等于 RDD 中最大块的大小。为什么这么说呢。看下面实现中有`(iter: Iterator[T]) => iter.toArray`这里将RDD中每一个partition 的 iterator 结果转换成了数组，也就是把 stream 数据都加载到内存，那么这里肯定会占用一整块 partition 大小的内存。

按照目前的情况，必须要将数据返回到 driver 端的。于是讨论出了以下3个方案。
## 方案一 提供逻辑小块
由于spark计算的原始数据的 RDD 是我们提供的，所以我们在这一层进行了处理。原来为一块的partition，根据数量的大小，被限制成了大约1000MB 一块的partition。这种方案的优点在于，因为只是逻辑上的分块，不产生额外的性能损失，可以解决大多数的etl场景。但是对于结果端有极端情况的 shuffle 计算的会导致小块合并成大块情况，就会破坏原有的分块，进而导致OOM
## 方案二 结果处调用 repartition
由于方案一中遇到的问题，我们很不情愿的采用了 repartition。在进行拉取数据到本地之前，再进行一次分块。目的是将大块分成小块。这里会有一定的性能损耗。但是这样的解决是可以避免 OOM

## 方案三 优化 result
::首先这个方案是不可行的::

之所以还列出该方案，是因为其中朴素的思想和设计到整个 job 的 result 处理流程，因此虽然方案不可行，但是还是列出来。
### 朴素的想法
当分析出 spark 是将一整块数据取到 driver 端，再返回一个 iterator 迭代器的之后，我们就开始思考是不是可以不要一次性返回一整块呢，用 iterator 很容易实现一个按需取数的结构。

开始天真的认为 spark 的 tolocalIterator 应该是通过input 和 output 这种方式返回到 driver端。driver 不断的调用 input 方法从而获得 result 的。后来发现其实不是这样的。
### 代码逻辑
我们着手处理这个问题，首先必须梳理清楚代码。
里面设计到的各个组件，可以参考我其它博文。这里我只暂时逻辑的主要部分上面在 toLocalIterator 中传递给 runTask 的方法是` (iter: Iterator[T]) => iter.toArray`这个代码会发送到 spark 服务器上并且执行。返回到结果的方法我们在spark 的 executor 类的run方法找到了答案

```
 val serializedResult: ByteBuffer = {
          if (maxResultSize > 0 && resultSize > maxResultSize) {
            logWarning(s"Finished $taskName (TID $taskId). Result is larger than maxResultSize " +
              s"(${Utils.bytesToString(resultSize)} > ${Utils.bytesToString(maxResultSize)}), " +
              s"dropping it.")
            ser.serialize(new IndirectTaskResult[Any](TaskResultBlockId(taskId), resultSize))
          } else if (resultSize > maxDirectResultSize) {
            val blockId = TaskResultBlockId(taskId)
            env.blockManager.putBytes(
              blockId,
              new ChunkedByteBuffer(serializedDirectResult.duplicate()),
              StorageLevel.MEMORY_AND_DISK_SER)
            logInfo(
              s"Finished $taskName (TID $taskId). $resultSize bytes result sent via BlockManager)")
            ser.serialize(new IndirectTaskResult[Any](blockId, resultSize))
          } else {
            logInfo(s"Finished $taskName (TID $taskId). $resultSize bytes result sent to driver")
            serializedDirectResult
          }
        }

        setTaskFinishedAndClearInterruptStatus()
        execBackend.statusUpdate(taskId, TaskState.FINISHED, serializedResult)

```

上面一大段代码的意思无非一个，将结果序列号，并且通过更新状态的方式返回。当时看到这里觉得完了，通过一个 RPC 调用的方式返回，那内存爆炸没法避免了啊。后来仔细一想，不可能这么做吧。于是分析代码发现如果结果过于巨大，spark 会通过 storage 将结果当成 block 处理，在 driver 再进行取数。
在 driver 跟到了结果的处理

```
*override def*taskSucceeded(index: Int, result: Any): Unit = {
  // resultHandler call must be synchronized in case resultHandler itself is not thread safe.
  synchronized {
    resultHandler(index, result.asInstanceOf[T])
  }
  *if*(/finishedTasks/.incrementAndGet() == totalTasks) {
    /jobPromise/.success(())
  }
}

```

这里的 resultHandler 的实现是`(index, res) => results(index) = res`
到这里基本断定，我们的需求不太容易实现了，因为在 spark 的 driver 中获得 result 前已经把结果都取出来。如果要改的话，那么代价太高。
虽然这个方法被证实为不可行，但是让我再一次见识了函数式编程的灵活性。
最后对 spark 的计算结果返回做一个小总结。首先在 runTask 方法的时候，driver 端肯定会传递一个最终的计算函数。如果这个函数是一个无返回值的，那么最终这个 task 是会返回一个空对象到 driver 端。但是如果这个函数是有返回值的，那么这个计算结果会随着状态更新一起返回到 driver 端。其中会对大数据量的结果进行处理，需要 drvier 端主动读取。












