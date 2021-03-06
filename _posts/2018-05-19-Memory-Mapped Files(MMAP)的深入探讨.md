---
layout: post
title:  "Memory-Mapped Files(MMAP)的深入探讨"
date:   2018-5-19  +0800
categories: java
---


关于Memory-Mapped文件的问题，问题多集中在在于unmap。官方有个05年至今还是Open状态的bug（参考1），说的就是unmap方法，其中有个回复说在JDK10中会解决，所有非常期待JDK10。
对于mmap的使用至今没有找到一个来源可靠的资料，能给个大概的使用。
网上对于java的MMAP文件使用资料不多，而且有大量今天看来是错误使用的。官方的API中对于mmap文件的使用，并未提供任何的unmap方法。对此给出的解释（参考1）和猜测的是一样的，还没有办法给出一个安全高效的unmap方法。

JVM在做垃圾回收的时候，实际上会对一个mmap的对象进行释放，相关实验代码参照 Java 文件读写相关，以及MappedByteBuffer的注释。
基于以上说明，和官方bug系统的回复。能够大概得出一个正确的使用。
mmap文件的使用，在当前情况下不用释放，像正常对象一样使用，gc会管理释放。
不能多次map同一文件。
 一个文件被map到内存后会占用到用户空间，此时并没有做实际的文件载入，当访问到该内存时候会进行缺页处理载，将相应的页数据入物理内存。所以对于用户空间有限制的系统，会导致文件map 失败，而报内存不足的异常。

MMAP过程分析
Java中的Memory-Mapped文件最终还是到系统的内存映射上。
内存映射的实现过程，总的来说可以分为三个阶段：（这里简化了很多细节，详细的见参考2）
（一）进程启动映射过程，并在虚拟地址空间中为映射创建虚拟映射区域
1、进程在用户空间调用库函数mmap
2、在当前进程的虚拟地址空间中，寻找一段空闲的满足要求的连续的虚拟地址（这个地方如果系统做了空间大小限制，同时gc后仍然没有足够空间，会抛内存不足的异常）
3、为此虚拟区分配一个结构，接着对这个结构的各个域进行了初始化
4、将新建的虚拟区结构插入进程的虚拟地址区域链表或树中

（二）调用内核空间的系统调用函数mmap（不同于用户空间函数），实现文件物理地址和进程虚拟地址的一一映射关系
5、为映射分配了新的虚拟地址区域后，通过待映射的文件指针，链接到内核“已打开文件集”中该文件的文件结构体。
6、通过该文件的文件结构体，调用内核函数mmap。
7、内核mmap函数通过虚拟文件系统定位到文件磁盘物理地址。
8、通过remap_pfn_range函数建立页表，即实现了文件地址和虚拟地址区域的映射关系。此时，这片虚拟地址并没有任何数据关联到主存中。

（三）进程发起对这片映射空间的访问，引发缺页异常，实现文件内容到物理内存（主存）的拷贝
注：前两个阶段仅在于创建虚拟区间并完成地址映射，但是并没有将任何文件数据的拷贝至主存。真正的文件读取是当进程发起读或写操作时。
9、进程的读或写操作访问虚拟地址空间这一段映射地址，通过查询页表，发现这一段地址并不在物理页面上。因为目前只建立了地址映射，真正的硬盘数据还没有拷贝到内存中，因此引发缺页异常。
10、缺页异常进行一系列判断，确定无非法操作后，内核发起请求调页过程。
11、调页过程先在交换缓存空间（swap cache）中寻找需要访问的内存页，如果没有则调用nopage函数把所缺的页从磁盘装入到主存中。
12、之后进程即可对这片主存进行读或者写的操作，如果写操作改变了其内容，一定时间后系统会自动回写脏页面到对应磁盘地址，也即完成了写入到文件的过程。

## Java 中使用 MMAP
正如[参考1](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=4724038)中提到的，没有一个可靠的技术来确保，安全地 unmap 一个文件映射。实际使用中 mmap 的方式的确有一定的风险。稍有不慎就会导致访问非法内存，从而导致 JVM 的崩溃。
这里提供一个较为准确的使用方案。
### 不理会方式
这个是应该是官方提倡的方式，毕竟不提供 unmap 方法，开发者只能不管不问了。使用起来只需要确保不再使用的对象没有引用即可。JVM 会在 gc 的时候去释放这块内存映射。
下面是`FileChannelImpl#map()`中的具体实现，在第一次抛`OutOfMemoryError`的时候，会将异常捕获，并且主动调用 gc 触发垃圾回收。
```
try {
    var7 = this.map0(var6, var13, var15);
} catch (`OutOfMemoryError` var30) {
    System.gc();

    try {
        Thread.sleep(100L);
    } catch (InterruptedException var29) {
        Thread.currentThread().interrupt();
    }

    try {
        var7 = this.map0(var6, var13, var15);
    } catch (OutOfMemoryError var28) {
        throw new IOException("Map failed", var28);
    }
}

```

### 主动释放
下面的代码可以直接使用

```
/**
   * Forces to unmap a direct buffer if this buffer is no longer used. After calling this method,
   * this direct buffer should be discarded. This is unsafe operation and currently a work-around to
   * avoid huge memory occupation caused by memory map.
   *
   * <p>
   * NOTE: DirectByteBuffers are not guaranteed to be garbage-collected immediately after their
   * references are released and may lead to OutOfMemoryError. This function helps by calling the
   * Cleaner method of a DirectByteBuffer explicitly. See <a
   * href="http://stackoverflow.com/questions/1854398/how-to-garbage-collect-a-direct-buffer-java"
   * >more discussion</a>.
   *
   * @param buffer the byte buffer to be unmapped, this must be a direct buffer
   */
  public static synchronized void cleanDirectBuffer(ByteBuffer buffer) {
    Preconditions.checkNotNull(buffer, "buffer");
    Preconditions.checkArgument(buffer.isDirect(), "buffer isn't a DirectByteBuffer");
    try {
      if (sByteBufferCleanerMethod == null) {
        sByteBufferCleanerMethod = buffer.getClass().getMethod("cleaner");
        sByteBufferCleanerMethod.setAccessible(true);
      }
      final Object cleaner = sByteBufferCleanerMethod.invoke(buffer);
      if (cleaner == null) {
        if (buffer.capacity() > 0) {
          LOG.warn("Failed to get cleaner for ByteBuffer: {}", buffer.getClass().getName());
        }
        // The cleaner could be null when the buffer is initialized as zero capacity.
        return;
      }
      if (sCleanerCleanMethod == null) {
        sCleanerCleanMethod = cleaner.getClass().getMethod("clean");
      }
      sCleanerCleanMethod.invoke(cleaner);
    } catch (Exception e) {
      LOG.warn("Failed to unmap direct ByteBuffer: {}, error message: {}",
                buffer.getClass().getName(), e.getMessage());
    } finally {
      // Force to drop reference to the buffer to clean
      buffer = null;
    }
  }
```

主动的方式需要注意的是，一定要在确保当前 DirectByteBuffer 是不再使用的，否则会引起 JVM 崩溃。
## 写在最后
文件的内存映射是非常快速的一种访问方式。而且在对于通过指定位置访问文件的方式，也有着非常棒的性能，因此它带来的速度的提升吸引着我们承担 JVM 崩溃的风险。所有为了避免JVM 崩溃，一地要牢记释放后的对象不能再次访问。


参考1：http://bugs.java.com/bugdatabase/view_bug.do?bug_id=4724038
参考2：http://www.cnblogs.com/huxiao-tee/p/4660352.html
参考3：Linux设备驱动程序