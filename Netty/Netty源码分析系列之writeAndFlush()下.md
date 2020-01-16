> 扫描下方二维码或者微信搜索公众号`菜鸟飞呀飞`，即可关注微信公众号，阅读更多`Spring源码分析`、`Java并发编程`和`Netty源码系列`文章。

![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144)

### 前言

在上一篇文章中（[Netty 源码分析系列之 writeAndFlush()上](https://mp.weixin.qq.com/s/BBDlOJtJeRIC3S2ioJVknw)）分析了 netty 将数据写出流程的前半部分：write()方法源码，知道了在这个过程中，数据只是被存放到了 NioSocketChannel 对象的 **ChannelOutboundBuffer** 缓冲区中，还没有被发送到操作系统的**套接字**中。只有当调用了 flush()方法后，才会真正将数据发送到套接字中。那么 flush()方法的源码又是如何执行的呢？这就是本文分析的重点。

还是以上篇文章的 demo 为例，客户端 channel 的 pipeline 的结构图如下所示。（关于 demo 的代码可以直接去查看上篇文章）

![Demo示例pipeline结构图](https://user-gold-cdn.xitu.io/2020/1/7/16f802d51e66ec6d?w=1442&h=328&f=png&s=52075)

### 源码分析

从 tail 节点的 writeAndFlush()方法开始，从 tail 节点的 write()方法向前传播执行，当 write()方法执行完之后，就会调用 **invokeFlush0()** 方法，该方法就是触发执行 flush()方法。上面的 pipeline 的结构中，由于 BizHandler 是 InBound 类型，在写数据的过程中不会触发执行它，另外由于 UserEncoder 我们没有重写 flush()方法，因此默认情况下，它啥也不干，直接再往前一个节点传播执行 flush 方法，因此最终调用的是 head 节点的 flush()方法。head 节点的 flush()方法源码如下。

```java
public void flush(ChannelHandlerContext ctx) {
    unsafe.flush();
}
```

直接调用的是 unsafe 对象的 flush()方法，由于此时是 NioSocketChannel 对象，因此 unsafe 是 **NioSocketChannelUnsafe** 对象的实例，又因为 NioSocketChannelUnsafe 继承自抽象类 AbstrractUnsafe，且 flush()方法定义在抽象类中，因此最终执行的是如下代码。

```java
public final void flush() {
    assertEventLoop();
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }
    // 改变待写队列的指针指向，
    outboundBuffer.addFlush();
    flush0();
}
```

在上面的代码片段中，主要有两行核心逻辑代码，第一处是调用 outboundBuffer 对象的 **addFlush()** 方法，改变写队列的三个指针指向；第二处是调用 **flush0()** 方法，真正将数据写到套接字缓冲区中。下面详细分析下这两处核心逻辑。

在上一篇文章中（（[Netty 源码分析系列之 writeAndFlush()上](https://mp.weixin.qq.com/s/BBDlOJtJeRIC3S2ioJVknw)））我们从源码中知道，outboundBuffer 这个缓冲区内部实际上是维护了一个队列，这个队列依靠三个指针来维护。**初始**状态下，这三个指针都是指向 **null**，当调用一次 write()方法向 outboundBuffer 缓冲区中写入一次数据对象后(这个数据对象会被封装成一个 Entry)，就会将 **unflushedEntry** 和 **tailEntry** 这两个指针指向刚刚写进来的这个数据所代表的 Entry，而此时 **flushedEntry** 这个指针仍然是指向 null。

**那么什么时候会改变 flushedEntry 指针的指向呢？** 那就是当调用 **addFlush()** 方法的时候，就会改变 flushedEntry 指针。addFlush()方法的作用就是将前面 write()写进到缓冲区的数据移动到刷新队列中，即：**将 unflushedEntry 指针指向的数据改变成由 flushedEntry 指针指向**，只有 flushedEntry 指针指向的数据才会真正地被发送到套接字中。用文字描述可能有点难以理解，可以结合下面的图来理解。

![指针变化示意图](https://user-gold-cdn.xitu.io/2020/1/11/16f93778a44ed863?w=1764&h=1290&f=png&s=275689)

（上一篇文章中也画了一张三个指针变化的关系图，但是由于疏忽，那张图的最后一行的指针指向画的不对，微信公众号没有提供修改的功能，所以阅读到这儿朋友注意下。）

下面看下 addFlush()方法的具体源码实现。

```java
public void addFlush() {
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            // 将flushed的指针指向第一个unFlushed的数据
            flushedEntry = entry;
        }
        do {
            // 记录目前有多少个数据等待被Flush
            flushed ++;
            // 现将promise设置为不可取消状态
            if (!entry.promise.setUncancellable()) {
                int pending = entry.cancel();
                // 递减待写的字节数，如果待写的字节数低于了最低水位线，那么就将channel的写状态设置为可写状态
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);

        // 所有unFlushed的数据均已经被标识成flushed了，所以unFlushed可以指针可以指向null了
        unflushedEntry = null;
    }
}
```

这段代码中，通过循环，将 unflushedEntry 指向的数据变为由 flushedEntry 指向，由于 unflushedEntry 执行的是第一个被 write()写进来的数据，因为 write()可能被多次调用，这样队列中就会有多个数据，因此使用了 do...while 循环，这样让 flushedEntry 指针指向第一个数据，然后再使用 **flushed** 这个变量来记录一下有多少个数据被从 unflushedEntry 指向的队列中移动到了 flushedEntry 指向的队列中，最后将 unflushedEntry 指向 null。

在数据被 write()进队列之后，被 flush()之前，数据是可以被取消的，可以通过 **promise.cancle()** 方法，取消数据的写出。但是当开始调用 flush()方法后，就不能再取消了，因为这一步即将将数据写入到操作系统的套接字中，所以再改变三个指针之前，需要将 promise 的状态设置为**不可取消状态：promise.setUncancellable()**。

当 setUncancellable()返回 false 时，表示的是 promise 之前已经被取消了，所以此时需要递减待写的字节数，如果缓冲区中待写的字节数低于了最低水位线，那么就将 channel 的写状态设置为可写状态。（前面在调用 write()方法向缓冲区中写数据时，会累计字节数，当超过最高水位线的时候，会将 channel 设置为不可写状态）。这里递减字节数使用的方法是 decrementPendingOutboundBytes()方法，其中 netty 默认的最高水位线是 64KB，最低水位线是 32KB。

第一处逻辑：addFlush()方法分析完了，接下来分析第二处核心逻辑：flush0()方法。当调用 flush0()方法时，首先会调用 AbstractNioUnsafe 类的 flush0()方法。源码如下。

```java
protected final void flush0() {
    // 如果channel注册了OP_WRITE事件，那么就会返回true
    if (!isFlushPending()) {
        super.flush0();
    }
}
```

在该方法中会先判断当前的 NioSocketChannel 是否注册了 **OP_WRITE** 事件，如果注册了 OP_WRITE 事件，那就不调用父类的 flush0()方法。如果没有注册，就调用父类的 flush0()方法。通过 isFlushPending()方法可以判断当前 channel 是否注册了 OP_WRITE 事件。其源码如下。

```java
private boolean isFlushPending() {
    SelectionKey selectionKey = selectionKey();
    return selectionKey.isValid() && (selectionKey.interestOps() & SelectionKey.OP_WRITE) != 0;
}
```

通常情况下，isFlushPending()方法会返回 false，因此会继续调用父类的 flush0()方法，即会调用 AbstractUnsafe 类的 flush0()方法，该方法的源码看着很长，实际上就一行核心逻辑：**doWrite(outboundBuffer)**；由于当前的 Channel 是 NioSocketChannel，所以会调用 NioSocketChannel 类的 doWrite()方法，**这个方法才是真正将数据写到套接字中**。源码很长，执行流程和逻辑见如下注释。

```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    // 获取JDK原生的SocketChannel
    SocketChannel ch = javaChannel();
    // 获取自旋写的次数
    int writeSpinCount = config().getWriteSpinCount();
    do {
        if (in.isEmpty()) {
            clearOpWrite();
            return;
        }

        int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
        // 将ByteBuf转换成ByteBuffer. ByteBuffer是JDK中的类
        ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
        /**
         * 获取ByteBuffer[]数组中，真正有几个ByteBuffer对象
         * （因为数组的长度可能是1024，但是数组中大部分元素时空的，所以不能直接通过nioBuffers.length来获取元素个数）
         * 而是需要通过in.nioBufferCount()来获取，这个方法返回的是ChannelOutboundBuffer对象的nioBufferCount属性值
         * 这个属性值又是什么时候被初始化的呢？是在上面调用in.nioBuffers()方法时进行赋值的
         */
        int nioBufferCnt = in.nioBufferCount();
        switch (nioBufferCnt) {
            case 0:
                // 如果ByteBuffers中没有内容可写，那么就调用普通写方法：doWrite0()，因为可能还有其他的内容可以写
                // 上面的in.nioBuffers()方法只会处理ByteBuf类型的数据，如果in中的数据全部不是ByteBuf类型，例如可能是FileRegion类型，
                // 那么此时nioBufferCnt就会为0，那么我们就需要调用doWrite0()方法进行普通写
                writeSpinCount -= doWrite0(in);
                break;
            case 1: {
                ByteBuffer buffer = nioBuffers[0];
                int attemptedBytes = buffer.remaining();
                // 采用原生的JDK的channel将数据写出去
                // 如果数据没有被写出去(可能是因为套接字的缓冲区满了等原因)，那么就会返回一个小于等于0的数
                final int localWrittenBytes = ch.write(buffer);
                // 如果小于等于0，说明数据没有被写出去，那么就需要将channel感兴趣的时间设置为OP_WRITE事件，方便下次多路复用器将channel轮询出来的时候，能继续写数据
                if (localWrittenBytes <= 0) {
                    incompleteWrite(true);
                    return;
                }
                // 根据上次写的数据量的大小，来调整下次可写的最大字节数
                adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                // 将flushedEntry指针指向的数据清空
                in.removeBytes(localWrittenBytes);
                --writeSpinCount;
                break;
            }
            default: {
                // 当ByteBuffer[]数组中有多个ByteBuffer待写的时候，就批量的写
                long attemptedBytes = in.nioBufferSize();
                // 调用JDK原生NIO的API，将数据写入到套接字中
                final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                if (localWrittenBytes <= 0) {
                    incompleteWrite(true);
                    return;
                }
                // Casting to int is safe because we limit the total amount of data in the nioBuffers to int above.
                // 根据上次写的数据量的大小，来调整下次最大的字节数限制
                adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                        maxBytesPerGatheringWrite);
                // 清除已经写出去的数据
                in.removeBytes(localWrittenBytes);
                --writeSpinCount;
                break;
            }
        }
    } while (writeSpinCount > 0);
    // 如果writeSpinCount的值小于0表示可能还有数据没有被写完，因此还需要继续注册OP_WRITE事件
    // 如果数据写完了，writeSpinCount的值会等于0或者大于0
    incompleteWrite(writeSpinCount < 0);
}
```

首先获取了 JDK 中原生 NIO 的 SocketChannel 对象，后面就是通过 JDK 的原生 API 将数据发送到套接字。

还从配置中获取了自旋的次数，这个次数默认是 16 次，我们可以通过配置客户端 channel 时，通过 DefaulSocketChannelConfig 对象配置。

这个自旋次数是用来干什么的呢？当我们使用 netty 向外发送数据时，有时候数据量比较少，但是有时候数据量会比较大。每次通过 JDK 的原生 API 向套接字中写数据时，每次只能写一部分。因此当数据量比较大的时候，我们需要分好几次调用 JDK 原生 API，才能将我们要发送的数据全部发送完，但是如果数据量特别大，那么循环调用 JDK 原生 API 的次数就越大，这样就会一直让 NioEventLoop 线程等在这儿，为了让其他任务得到 NioEventLoop 线程的执行，这里就需要设置一个最大的循环次数了，也就是我们配置的这个自旋写的次数，默认 16 次。当所有数据发送完成或者自旋次数超过 16 次时，这里的循环写均会被中断。

然后会从配置中获取最大能写多少字节的数据：**maxBytesPerGatheringWrite**，第一次默认是 Integer.MAX_VALUE。**后面这个值会动态的调整**。

在使用 JDK 的原生 API 写数据时，只接受 **ByteBuffer** 类型的数据，而我们前面的数据时 ByteBuf 类型，所以接下来需要将 ByteBuf 转换成 ByteBuffer 类型，怎么转换的呢？就是调用这一行代码：**in.nioBuffers()**。这一行代码后面分析。

接着通过 nioBufferCnt 的值，来判断进入哪一个 switch 分支。当要写出的数据对象不是 ByteBuf 时，例如文件类型（在 netty 中就是 FIleRegion 类型），nioBufferCnt 的值会为 0，所以会调用 doWrite0()方法进行数据的写出，最终也是通过 JDK 的原生 API 写入套接字中，对于文件对象，最终会使用 **tranferTo()** 方法，这样方法性能更优，具有零拷贝的特点。nioBufferCnt 为 0 的情况很少，因此不是今天讨论的重点。

当 nioBufferCnt 为 1 或者大于 1 时，执行的逻辑几乎一模一样，区别是最终会使用 JDK 中 NioSocketChannel 不同的 write()方法。当为 1 时，表示只有一个 ByteBuffer 对象可写，因此调用的是 **write(ByteBuffer buf)** 方法，当大于 1 时，表示有多个 ByteBuffer 可以写，因此调用的是 **write(ByteBuffer[] srcs, int offset, int length)** 方法。

由于两个分支逻辑几乎相同，因此选择其中一个分支进行分析，我们选择 case:1 这个分支。

```java
ByteBuffer buffer = nioBuffers[0];
int attemptedBytes = buffer.remaining();
// 采用原生的JDK的channel将数据写出去
// 如果数据没有被写出去(可能是因为套接字的缓冲区满了等原因)，那么就会返回一个小于等于0的数
final int localWrittenBytes = ch.write(buffer);
// 如果小于等于0，说明数据没有被写出去，那么就需要将channel感兴趣的时间设置为OP_WRITE事件，方便下次多路复用器将channel轮询出来的时候，能继续写数据
if (localWrittenBytes <= 0) {
    incompleteWrite(true);
    return;
}
// 根据上次写的数据量的大小，来调整下次可写的最大字节数
adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
// 将flushedEntry指针指向的数据清空
in.removeBytes(localWrittenBytes);
--writeSpinCount;
```

先获取到 buffer 中可写的字节数，然后调用 JDK 原生 API：ch.write(buffer)将数据发送到套接字中，该 API 会返回**成功写出去的字节数**，如果返回的数小于等于 0，说明数据没有被写出去，数据没有被写出去**可能是因为套接字的缓冲区满了等原因**，此时就需要将 channel 感兴趣的事件设置为 **OP_WRITE** 事件，方便下次多路复用器将 channel 轮询出来的时候，能继续写数据，所以会调用 **incompleteWrite(true)** 方法。

```java
protected final void incompleteWrite(boolean setOpWrite) {
    // 如果数据没有写完，就需要将channel感兴趣的事件设置为OP_WRITE事件
    if (setOpWrite) {
        setOpWrite();
    } else {
        // 如果数据已经写完了，那就清除channel的OP_WRITE事件
        clearOpWrite();
        eventLoop().execute(flushTask);
    }
}
```

如果数据写出去了，就会根据当前发送的数据量的大小，来动态调整最大字节数的限制。netty 可以说在数据的读写上将优化做到了极致，它会根据本次写出去的数据量的大小来**猜测**下一次数据量的大小，如果本次写出去的数据量大，那么 netty 就会认为你下次发送的数据会更大，就会将最大字节数限制扩大 2 倍；如果发送的数量小，netty 就认为你的需求小，所以就会将最大字节数限制缩小 2 倍。

```java
private void adjustMaxBytesPerGatheringWrite(int attempted, int written, int oldMaxBytesPerGatheringWrite) {
    // attempted 和written相等，那么就表示要写的数据全部写完了，因此netty就认为你要写的数据量很大，下次可能还会写更多的数据，
    // 因此将最大的字节数限制调整到当前数据量的两倍，也就是attempted向左移一位
    if (attempted == written) {
        if (attempted << 1 > oldMaxBytesPerGatheringWrite) {
            ((NioSocketChannelConfig) config).setMaxBytesPerGatheringWrite(attempted << 1);
        }
    }
    // 如果已经的数据小于attempted的半，且attempted大于4096，那么就将最大字节数限制缩小到attempted的一半。
    else if (attempted > MAX_BYTES_PER_GATHERING_WRITE_ATTEMPTED_LOW_THRESHOLD && written < attempted >>> 1) {
        ((NioSocketChannelConfig) config).setMaxBytesPerGatheringWrite(attempted >>> 1);
    }
}
```

将数据发送出去后，就需要改变 ChannelOutboundBuffer 这个缓冲区中 flushedEntry 的指针了。如何改变的呢？就是根据本次写出去的字节数，来反算出 flushedEntry 指针指向的 Entry 中数据有没有被写完，如果没有写完，就不移动指针，等待下次自旋写的时候会继续发送这个 entry 里面的数据。如果写完了，就判断 flushedEntry 队列后面还有没数据可写，如果有则将指针指向下一个 entry，如果没有，就指向 null。最后还会调用 decrementPendingOutboundBytes()方法来递减缓冲区中的字节数，如果字节数低于最低水位线，那么 channel 的状态将会被设置成可写状态。这一段的逻辑全部逻辑都是在 in.removeBytes()这一行代码中完成的，有兴趣的朋友可以深入研究下。

最后将自旋次数 writeSpinCount 减 1。

当 do...while 执行完成后，如果 writeSpinCount 的值小于 0 表示可能还有数据没有被写完，因此还需要继续注册 **OP_WRITE** 事件，如果数据写完了，writeSpinCount 的值会等于 0 或者大于 0。所以接下来会调用 incompleteWrite()方法，传入的参数是一个 boolean 值，表示数据有没有被写完。该方法在前面已经分析了一半，现在分析后面一半逻辑。

```java
protected final void incompleteWrite(boolean setOpWrite) {
    // 如果数据没有写完，就需要将channel感兴趣的事件设置为OP_WRITE事件
    if (setOpWrite) {
        setOpWrite();
    } else {
        // 如果数据已经写完了，那就清除channel的OP_WRITE事件
        clearOpWrite();
        eventLoop().execute(flushTask);
    }
}
```

当传入的参数为 true 时，表示数据没有写完，就需要将 channel 感兴趣的事件设置为 OP_WRITE 事件。当传入的参数为 false 时，表示数据写完了，需要清除 OP_WRITE 事件，同时让 NioEventLoop 执行一个异步任务。

好了，doWrite()方法的源码分析完了，下面来分析一下前面提到了 in.nioBuffers()方法的源码。该方法的作用就是将 ByteBuf 转变成 ByteBuffer（**ByteBuf 是 netty 中的类，ByteBuffer 是 JDK 提供的类**），方法的执行流程见下面代码合注释。

```java
public ByteBuffer[] nioBuffers(int maxCount, long maxBytes) {
    assert maxCount > 0;
    assert maxBytes > 0;
    long nioBufferSize = 0;
    int nioBufferCount = 0;
    final InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    ByteBuffer[] nioBuffers = NIO_BUFFERS.get(threadLocalMap);
    Entry entry = flushedEntry;
    while (isFlushedEntry(entry) && entry.msg instanceof ByteBuf) {
        // 当数据被write之后，flush之前，是有可能被取消的，因此这里需要判断数据是否被取消
        if (!entry.cancelled) {
            ByteBuf buf = (ByteBuf) entry.msg;
            final int readerIndex = buf.readerIndex();
            // 数据的字节长度
            final int readableBytes = buf.writerIndex() - readerIndex;

            if (readableBytes > 0) {
                // 超过了最大限制且nioBufferCount的数量不为0，则停止循环
                if (maxBytes - readableBytes < nioBufferSize && nioBufferCount != 0) {
                    break;
                }
                // 累计flush的字节数
                nioBufferSize += readableBytes;
                int count = entry.count;
                if (count == -1) {
                    // buf.nioBufferCount()，对于对外内存而言，会返回1
                    entry.count = count = buf.nioBufferCount();
                }
                // 判断ByteBuffer数组是否需要扩容
                // neededSpace表示的是需要的大小，去maxCount和nioBufferCount + count 这两者之间的最小值，是为了保证需要的数组大小最大不能超过1024
                int neededSpace = min(maxCount, nioBufferCount + count);
                // 如果需要的ByteBuffer数组的大小大于了 nioBuffers现在的数数组长度，就进行扩容
                if (neededSpace > nioBuffers.length) {
                    // 将ByteBuffer数组扩容
                    nioBuffers = expandNioBufferArray(nioBuffers, neededSpace, nioBufferCount);
                    // 然后扩容后的ByteBuffer数组存放在ThreadLocal中，这样就不用每次创建ByteBuffer数组了
                    NIO_BUFFERS.set(threadLocalMap, nioBuffers);
                }
                if (count == 1) {
                    ByteBuffer nioBuf = entry.buf;
                    if (nioBuf == null) {
                        // 将数据封装成ByteBuffer
                        entry.buf = nioBuf = buf.internalNioBuffer(readerIndex, readableBytes);
                    }
                    nioBuffers[nioBufferCount++] = nioBuf;
                } else {
                    nioBufferCount = nioBuffers(entry, buf, nioBuffers, nioBufferCount, maxCount);
                }
                if (nioBufferCount == maxCount) {
                    break;
                }
            }
        }
        entry = entry.next;
    }
    this.nioBufferCount = nioBufferCount;
    this.nioBufferSize = nioBufferSize;

    return nioBuffers;
}
```

这段代码很长，但主要干了两件事。第一，将 ByteBuf 转变成 ByteBuffer；第二，累加 nioBufferCount 的数量。在这期间，会涉及到 ByteBuffer 数组的扩容，具体细节，可以参考上面代码中的注释。

### 推荐
* [如何从BIO演进到NIO，再到Netty](https://mp.weixin.qq.com/s/zcrclRhgK015FSjtuMZoCg)
* [Netty源码分析系列之Reactor线程模型](https://mp.weixin.qq.com/s/-aqIwWXYBkTy0dCNy0FtlA)
* [Netty源码分析系列之服务端Channel初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw)
* [Netty源码分析系列之服务端Channel注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)
* [Netty源码分析系列之服务端Channel的端口绑定](https://mp.weixin.qq.com/s/p3ZouaQ_x6NgyVMWw4tJ0g)
* [Netty源码分析系列之NioEventLoop的创建与启动](https://mp.weixin.qq.com/s/qBRY8zl_LBrn3acCaRmEbg)
* [Netty源码分析系列之NioEventLoop的执行流程](https://mp.weixin.qq.com/s/RId4bKC0lwL_HofyqDV61Q)
* [Netty源码分析系列之新连接的接入](https://mp.weixin.qq.com/s/mKQXgVVhgI7Tbi9GQ4mGiQ)
* [Netty源码分析系列之TCP粘包、半包问题以及Netty是如何解决的](https://mp.weixin.qq.com/s/IihycUIcdTuigEmBbbkfBg)
* [Netty源码分析系列之常用解码器（上）](https://mp.weixin.qq.com/s/K-jzX1Vm5SYJEIrSTazk_g)
* [Netty源码分析系列之常用解码器（下）——LengthFieldBasedFrameDecoder](https://mp.weixin.qq.com/s/xoMZr6j6QPwGCeqodX49ww)
* [Netty源码分析系列之writeAndFlush()上](https://mp.weixin.qq.com/s/BBDlOJtJeRIC3S2ioJVknw)

### 总结

至此，writeAndFlush()方法的执行逻辑全部分析完了，本文主要分析的是 flush()方法的执行流程。综合本文和上篇文章来看，pipeline 中的 head 节点十分重要，在它里面实现了很多重要的功能，无论是 write()过程还是 flush()过程，都经过 head 节点做了最要的处理，最后通过 head 节点将数据发送到套接字。
