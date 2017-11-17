---
layout: post
title:  "Alluxio的FileInStream"
date:   2017-11-17  +0800
categories: alluxio code
---
为了方便以后更新Alluxio的client，所以把FileInStream的相关结构记录,以下记录基于alluxio-client-fs-1.6.1，主要关注于Local的Block。如果希望对自己有帮助，您需要对Alluxio整体结构，NIO的DirectByteBuffer有一个基本了解。

# 整体结构
 一个完整FileInStream中最重要的是mCurrentBlockInStream的属性，其类型为BlockInStream，负责从块中读取数据。而一个Block中被划分为多个定长的Packet，如果是Local的Block，那么一个Packet就作为一个NIO的映射单元。 
 
 # 流水记录
首先创建FileInStream，需要必要要提供三个参数，分别为
 * URIStatus status 包含当前文件的元信息
 * InStreamOptions options 当前FileInStream的配置
 * FileSystemContext context 作为client连接Alluxio服务的通道 
 
下面针对几个主要方法，记录一下流水帐

```
int readInternal(byte[] b, int off, int len) throws IOException
 
 这个是读取数据最终的实现。首先做一些常规边界判断，接着就调用updateStreams()，更新BlockInStream对象，这也是一个比较重要的方法下面会介绍。
 然后调用BlockInStream的read方法，进行数据读取。最后不能忘记更新pos。其中涉及到的缓存由于是针对Remote的Block情况，于是略去了。
```
 
```
void seek(long pos) throws IOException 
 
 在FileInStream中，seek方法并没有做与seek相关的操作，和readInternal类似，边界的判断，updateStreams()更新BlockStream，
 主要的seek事情都在BlockInStream中做了，下面在介绍BlockInStream中会介绍到。还有缓存的内容，这部分做的事情很多，但是于Local不符，还是略去。
 要注意这个方法很重，频繁调用seek，非常影响读取，这点在BlockInStream中会看到。
```


```
void updateStreams() throws IOException
 更新mCurrentBlockInStream，mStreamBlockId和mCurrentCacheStream。由于File是分为多个Block块的，所以需要从一个Block读到另一个Block，这里做的就是这个切换。
 首先依据指向当前文件位置的mPos，计算出块的index，根据上面提到的status获得相应的BlockId。拿到BlockId后，通过mBlockStore获得一个BlockInStream。
```
以上是FileInStream的主要部分，下面记录下BlockInStream的流水内容。
从AlluxioBlockStore类中可以发现，BlockInStream对象并没有子类继承等，那么对于却别数据来自于本机还是网络其它机器，主要是通过packet（类型为DataBuffer）这一属性来做到的。
改属性有针对网络数据的读取。不过下面记录的主要还是还是Local的情况，也就是通NIO内存映射的实现。在BlockInStream中，实际还有一层packet，虽然类型为DataBuffer而没有一个Packet对象。
这里也一块给记录了。
对于BlockInStream，最重要的莫过于packet了，对于Alluxio来说最底层的数据来源。除此以外，还有mPacketReader重要一些，负责从数据上获得一个packet。


```
int read(byte[] b, int off, int len) throws IOException 
 读取Block中数据的方法，最终到packet中去读取。每一个packet映射Block中一段数据，然后从0位读到末尾。注意packet不支持seek方法，所有这里会有一个readPacket方法的调用，详细下面介绍。例如，有一个100MB的Block，packet大小为10MB。
 首先seek到Block中11MB的地方，然后调用read来读取的时候，这里产生一个映射Block上11MB到21MB的packet。然后从packet的0位置，开始读取。如果没有调用seek方法，会从0一直读到尾，然后开始活得一段新的packet。
 
```

```
void readPacket() throws IOException 
 这个方法的主要根据当前的mPos和packet的长度，来映射一段Block上的数据。当mCurrentPacket被置空的时候，开始新的生成新的packet。
 mCurrentPacket只会在读到末尾，和Block被调用seek的时候置空。注意，这里是用mPos来作为映射的开始位置，不是一段一段的映射。上面已经举例说明了。
```

```
void seek(long pos) throws IOException
 Block的seek方法，可是这个方法做到想ByteBuffer之类的轻量级，而是非常之重，对于Local里面牵扯到内存映射的高代价操作。
 上面的介绍中也有提到一些，下面详细说下其中的缘由。首先一个对于一个Block按照一个定长来做映射，内存占用是一个很重要的因素。
 如果将一个Block完全映射，那么内存占用会非常高(这里不是实际物理内存)，此外对于Remote的情况，一个packet也是非常灵活的选择。
 如果认为一个packet是读取的最小单位，同时不支持seek。那么Block调用seek后必须释放当前的packet。在seek后的位置重新映射一个packet来读取。
 要知道内存映射是代价是很高的。频繁seek和read的唯一结果就是速度极其缓慢。
```
以上是对于Client中FileInStream的大概介绍。总的来说，一个File被分为1-N个定长Block，一个Block每次依据mPos随机映射一段packet来读取数据。



