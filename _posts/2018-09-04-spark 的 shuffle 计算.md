---
layout: post
title:  "Spark 目录"
date:   2018-9-4  +0800
categories: spark 
---

# Spark的 Shuffle 计算
## 概述
由于分布式计算是数据分块，多机器同时计算的架构。但是当一个计算依赖的数据不是某一块数据，而是全局数据的时候。这类计算变难以分块计算，例如 join 操作。为了充分发挥集群机器的算力，能够并行计算。我们就会在这些依赖全局数据的计算前，增加一个 shuffle 操作。目的是按照某种逻辑将同一类数据汇集到一块中，这样变将原有无法并行计算的结构，转为可以并行计算的逻辑了。
Spark 中同样如此。所有需要进行 shuffle 操作的算子，都会加入到一个叫`ShuffledRDD`的对象中。`ShuffledRDD`的`getDependencies`方法返回到的是一个`ShuffleDependency`对象。这里的意思就是说，当前算子所需的 RDD 是一个经过 shuffle 计算的 RDD。而这个`ShuffleDependency`决定着如何来做 shuffle 操作。


## 构建 ShuffledRDD
上面解释了为什么，每一个操作，需要全局数据的时候，都需要进行先进行一次 shuffle 操作。其目的是把分散在全局中的同类数据聚集到一块中来，这样后续计算才能在同一台机器上继续执行。所以每个需要 shuffle 操作的算子，都会加到一个 ShuffledRDD对象中。该对象中比较重要的是`partitioner`属性和`getDependencies`方法。
`getDependencies`返回会返回一个`ShuffleDependency`类型的依赖。`DAGScheduler`每当看到这样的依赖，就会把此 RDD 和后续的 RDD 分开，并产生一个 ShuffleStage。而 shuffle stage 的 task 是由 `ShuffleMapTask`f负责执行。只有进行过 shuffle, shuffledRDD 的算子才可以继续执行（详见 DAGScheduler）。
 `partitioner`这个决定着数据的走向。在执行shuffleMapTask计算的时候，子 RDD 的每一个数据的 key首先会交给 `partitioner`，并获得一个partition id。最终数据依据partition id被写入到相应的 partition 范围中。注意这里的partition依然在本机的中，每台机器都会有一个相同partition id的partition，下面会讲到。至此一个 RDD 的构造就完成了。下图是一个简单的 shuffle 说明，并不难理解。
此外还有ShuffledRDD需要用到的ShuffleReader对象也非常重要，这个部分下面会讲到。

## Shuffle计算
在上面的构造 shuffledRDD中已经大概介绍了 Shuffle的计算，不再多述。这里主要剖析内部有价值的计算细节。
```
override def runTask(context: TaskContext): MapStatus = {
    val ser = SparkEnv.get.closureSerializer.newInstance()
    val (rdd, dep) = ser.deserialize[(RDD[_], ShuffleDependency[_, _, _])](
      ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
    var writer: ShuffleWriter[Any, Any] = null
    try {
      val manager = SparkEnv.get.shuffleManager
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
      writer.stop(success = true).get
    } catch {
    }
  }
```

上面是 ShuffleMapTask 计算的主要代码。计算逻辑非常简单
1. 获得计算的 RDD 对象，父RDD 对于此 RDD 的和依赖关系。
2. 依据依赖关系构建一个`ShuffleWriter`对象，该对象主要用途是实现具体的 shuffle 算法。
3. 获得此 RDD 的 iterator 对象，并传递给`ShuffleWriter`执行 shuffle 操作。


## ShuffleWriter
通过 ShuffleTask 的计算逻辑，我们知道ShuffleWriter是一个核心对象。ShuffleWriter有以下实现
* BypassMergeSortShuffleWriter
* SortShuffleWriter
* UnsafeShuffleWriter
每个实现都牵扯很多独特的细节问题，这里不去过多探讨实现细节（需要了解详见spark目录）
我们这里用BypassMergeSortShuffleWriter实现来解释 shuffle 的操作实现。因为BypassMergeSortShuffleWriter可以算是最朴素的实现，便于了解 shuflle 的实现细节

```
  public void write(Iterator<Product2<K, V>> records) throws IOException {
    for (int i = 0; i < numPartitions; i++) {
      final Tuple2<TempShuffleBlockId, File> tempShuffleBlockIdPlusFile =
    blockManager.diskBlockManager().createTempShuffleBlock();
      final File file = tempShuffleBlockIdPlusFile._2();
      final BlockId blockId = tempShuffleBlockIdPlusFile._1();
      partitionWriters[i] =
        blockManager.getDiskWriter(blockId, file, serInstance, fileBufferSize, writeMetrics);
    }

    while (records.hasNext()) {
      final Product2<K, V> record = records.next();
      final K key = record._1();
      partitionWriters[partitioner.getPartition(key)].write(key, record._2());
    }
  }
```

上面是 write方法去掉无关代码后的主干代码。逻辑如下
1. 首先按照后续RDD计算的需要，给每个块创建一个文件。
2. 从当前 RDD 中读取一条数据，获得其中的 key，并给`partitioner`计算partition index
3. 再依据index从第一步创建的块文件中，拿到对应的块文件
4. 将此条数据写入到文件中
5. 将全部的块文件进行合并，合并成一个文件。
6. 这个文件就是 shuffle 的结果了，后面 shuffledRDD 会通过ShuffleReader来读取数据。


## ShuffleReader
正如上面所说，ShuffleReader是 shuffle 计算过程中非常重要的一个对象。ShuffleRDD 的算子需要通过ShuffleReader向子 RDD拿数据时候。

下面是ShuffleRDD的 compute 方法。

```
override def compute(split: Partition, context: TaskContext): Iterator[(K, C)] = {
    val dep = dependencies.head.asInstanceOf[ShuffleDependency[K, V, C]]
    SparkEnv.get.shuffleManager.getReader(dep.shuffleHandle, split.index, split.index + 1, context)
      .read()
      .asInstanceOf[Iterator[(K, C)]]
  }

```

1. 通过shuffleManager对象，依据依赖关系拿到ShuffleReader
2. 调用ShuffleReader的 read 方法，获得shuffle 后的数据的 iterator

## 总结
至此 shuffle 的过程和涉及到对象就全部解释完了。这里只是一个 shuffle 的骨架结构，目的是提供一个宏观视角，起一个引导作用。所有尽量避免涉及到具体细节，需要细节的可以深入代码了解。




