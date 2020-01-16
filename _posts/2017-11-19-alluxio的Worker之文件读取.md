---
layout: post
title:  "alluxio的Worker之文件读取"
date:   2017-11-19  +0800
categories: alluxio
---
这个系列，从一个场景作为突破口，最后再从整体来看一遍。

首先在《Alluxio的FileInStream记录》一文中将Local模式的介绍了一遍。但是有个疑问:如果文件是写在UFS（底层文件存储）上的话，那么文件改如何读取呢？开始我以为会像Local模式一样，通过NIO的MappedByteBuffer读取UFS上的文件，并且同时cache到Mem中。再仔细看完代码后，发现事实并非如此，在FileInStream中，只要是非Local模式的，都会通过Netty走网络，即使是在本机的UFS上面也是如此。主要是Local模式外加设置USER_SHORT_CIRCUIT_ENABLED参数来使用直接读取文件，这样做是绕过Alluxio，是属于特殊情况。此外对于Local来说，文件结构是在Alluxio的控制之下的，而对于UFS的则无法控制而且也负责很多，统一的办法就是利用Netty和数据Buffer来读取数据。这部分内容的代码如下

```
public static BlockInStream create(...) {
    if (Configuration.getBoolean(PropertyKey.USER_SHORT_CIRCUIT_ENABLED)
        && !NettyUtils.isDomainSocketSupported(address)
        && blockSource == BlockInStreamSource.LOCAL) {
        return createLocalBlockInStream(context, address, blockId, blockSize, options);
      }
    }
    return createNettyBlockInStream(context, address, blockSource,blockSize,options);
  }
  这是BlockInStream类中的create方法，修改了无关内容，从if的条件判断可以看出上面所说的内容。
```
到这里client端的事情基本结束，因为再往下是《Alluxio的FileInStream记录》文中介绍的内容，当然这里在packet读取部分，是通过Netty走的是网络传输。既然如此，就不再赘述了，希望了解细节的朋友，可以结合《Alluxio的FileInStream记录》一文和代码来深入了解。

那么client端既然已经结束了，实际读取的工作肯定是由worker来做了。这里有放出一个问题:worker是以什么样的方式来读取数据，以及如何将数据载入内容的？

再往下继续之前，需要介绍一下worker的结构，以及netty的简单内容。否则下面会有些小难。首先说下netty，这是一个以事件为基础的java网络框架，简单，高效，灵活，统一。用过原生nio的小伙伴，看到netty一定会有种相见恨晚的感觉。这里有一个ChannelPipeline对象，这是一个拦截流经 Channel 的入站和出站事件的 ChannelHandler 实例链

[](!images/alluxio/netty_pipeline.png)
如上图可见，一个I/O事件在pipline上进行传播。那么，在worker中也有这么一个对象，负责来处理client的请求。

```
    // Block Handlers
    pipeline.addLast("blockReadHandler",new BlockReadHandler(NettyExecutors.BLOCK_READER_EXECUTOR,
            mWorkerProcess.getWorker(BlockWorker.class), mFileTransferType));
```
这是在worker项目下的PipelineHandler类中的代码。这里的BlockReadHandler就是主要处理client的读取Block的请求。实际做事的是BlockPacketReader的内部类。由于pipline你不能阻塞在这里的，否则后面的处理器会迟迟得不到调用的。这里在BlockReadHandler收到读取请求，被调用channelRead方法时候。会创建一个BlockPacketReader以一个独立线程的方式，进行数据读取操作。


```
  /**
    *  Invoked when the current {@link Channel} has read a message from the peer.
    */
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object object) throws Exception {
    Protocol.ReadRequest msg =((RPCProtoMessage)object).getMessage().asReadRequest();
    validateReadRequest(msg);
    //创建一个BlockPacketReader并执行。
    mPacketReaderExecutor.submit(createPacketReader(mContext, ctx.channel()));
    mContext.setPacketReaderActive(true);
    
  }
  依然只留主要部分代码。这个channelRead会在收到消息时候被调用。
```
那么下面继续回到worker的UFS文件读取。

BlockPacketReader的父类PacketReader有一个run方法，无外乎就是从文件所在位置，读取并且传输给client端。注意这里是完全异步的，没有阻塞存在。client端读取好数据交给netty，netty传输给client，client收到数据后是放到队列中，client读取数据是从队列中取数。这个过程中每个人都是做自己的时间，不会等待网络传输。


在run方法中，通过UnderFileSystemBlockReader类将文件中的数据读取到buffer中，再交由netty传输给client端。从文件中读取数据的同时，会将数据写入Mem中。当数据传输完之后，worker会将这个Mem的Block提交，成为正式的block。下面我们来详细看一下。
这里需要说一下Block如何可用。首先调用BlockStore的createBlock方法，创建一个临时Block。然后调用getBlockWriter方法，获得改临时Block的写对象。一直到写入数据完成，这时改Block对外是不见的，只有这个worker自身知道存在。最终调用commitBlock方法将这个Block提交，这样这个Block就称为可见可用状态了。

下面三段代码，分别对应是初始化，读数据并写入Alluxio中，关闭提交Block。我已经将代码所在的类标明了，为了容易阅读，我把无关紧要的代码都去除了，只留下主要的代码。需要看具体代码的话，按照所在类查找实际代码来阅读。


**第一步.临时Block的初始化**
```
  
  //这里是UnderFileSystemBlockReader中更新BlockWriter的方法。无关代码依然去除了
  private void updateBlockWriter(long offset) throws IOException {
    //首先创建了一个临时Block
    mLocalBlockStore.createBlock(mBlockMeta.getSessionId(), mBlockMeta.getBlockId(), loc,mInitialBlockSize);
    //获得了改临时Block的写方法。
    mBlockWriter = mLocalBlockStore.getBlockWriter(mBlockMeta.getSessionId(), mBlockMeta.getBlockId());
      
  }
```
**第二步.读取UFS文件数据，同时写入临时Block**
```
  //这是UnderFileSystemBlockReader中读取数据的方法。这里只是将数据读出，并且写入到到BlockerWriter中。
  public ByteBuffer read(long offset, long length) throws IOException {
 
    byte[] data = new byte[(int) bytesToRead];
    read = mUnderFileSystemInputStream.read(data, bytesRead, (int) 
    ByteBuffer buffer = ByteBuffer.wrap(data, (int) (mBlockWriter.getPosition() - offset),(int) (mInStreamPos - mBlockWriter.getPosition()));
    //同时写入Alluxio的Blocker中
    mBlockWriter.append(buffer.duplicate());
    return ByteBuffer.wrap(data, 0, bytesRead);
  }
```
**第三步.数据读取完毕后，那么将临时Block提交，使其对外可见。**

```
  //这是在DefaultBlockerWorker类中的代码
  public void closeUfsBlock(long sessionId, long blockId) {
      mUnderFileSystemBlockStore.closeReaderOrWriter(sessionId, blockId);
      try {
      //数据写完后，BlockerWorker关闭UFS的block，同时进行了Alluxio的Block提交工作
        commitBlock(sessionId, blockId);
      } catch (BlockDoesNotExistException e) {
    }
    mUnderFileSystemBlockStore.releaseAccess(sessionId, blockId);
  }
```

至此，对于UFS上的文件读取就完成了，同时这个Block文件也成为了Alluxio的文件了。下次倘若再读取这个文件，就不会进入UFS的文件读取了。

这里有一个小问题:我们都知道Alluxio会Mem的文件按照规则进行eviction操作。那么如果一个Block被evict到了Disk上面后，倘若此时再进行读取数据，是按照什么逻辑呢？是当成Local通过short circuit的直接读取方式，还是通过netty来呢？