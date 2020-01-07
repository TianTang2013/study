### 问题

在上一篇文章中分析到了 Netty 服务端是如何进行新连接的接入的，那么当新连接接入后，就可以开始数据的读写操作了。在进行数据读写操作时，对于 TCP 连接而言，netty 就需要解决 TCP 中粘包、半包的问题，这将是本文今天重点分析的内容。在开始阅读本文之前，可以先思考一下以下两个问题。

- 1. 什么是 TCP 的粘包、半包问题？UDP 协议存在粘包半包吗？
- 2. netty 是如何解决的

### 什么是粘包拆包

在 TCP/IP 协议模型中，TCP 和 UDP 协议属于传输层协议，这两个协议在数据传输过程中存在很大的差异。

对于 UDP 协议而言，它传输的数据是基于数据报来进行收发的，在 UDP 协议的头中，会有一个 16bit 的字段来表示 UDP 数据报文的长度，在应用层能很好的将不同的数据报文区分开。可以理解为，UDP 协议传输的数据是有边界的，因此它不会存在粘包、半包的问题。

而对于 TCP 协议而言，它传输数据是基于字节流传输的。应用层在传输数据时，实际上会先将数据写入到 TCP 套接字的缓冲区，当缓冲区被写满后，数据才会被写出去，这就可能造成粘包、半包的问题。而且当接收方接收到数据后，实际上接收到的是一个字节流，所谓的流，可以理解为河流一样。既然是流，多个数据包相互之间是没有边界的，而且在 TCP 的协议头中，没有一个单独的字段来表示数据包的长度，这样在接收方的应用层，从字节流中读取到数据后，是没办法将两个数据包区分开的。

### 粘包、半包示意图

当发送方连续向接收方发送两个完整的数据包时，如果使用 TCP 协议进行传输，就可能存在以下几种情况。下图中 packet1 和 packet2 分别表示发送方发送的两个完整的数据包。

第一种情况，没有发生粘包、半包的现象，即接收方正常接收到两个独立的完整数据包 packet1、packet2，这种情况是属于正常情况。如图 1 所示。


![正常包](https://user-gold-cdn.xitu.io/2019/12/29/16f5166c22945a7e?w=1294&h=302&f=png&s=59839)

第二种情况，发生了粘包现象，即发送方将数据包 packet1 写入到自己的 TCP 套接字的缓冲区后，TCP 并没有立即将数据发送出去，因为此时缓冲区可能还没有慢。接着发送方又发送了一个数据包 packet2，仍然是先写入到 TCP 套接字的缓冲区，此时缓冲区满了，然后 TCP 才将缓冲区的数据一起发送出去，这时候接收方接收到的数据看起来只有一个数据包。在 TCP 的协议头中，没有一个单独的字段来表示数据包的长度，这样接收方根本就无法区分出 packet1 和 packet2，这就是所谓的粘包问题。另外，当接收方的 TCP 层接收到数据后，由于应用层没有及时从 TCP 套接字中读取数据，也会造成粘包现象。如图 2 所示。


![粘包](https://user-gold-cdn.xitu.io/2019/12/29/16f5166fcce55f0f?w=1244&h=312&f=png&s=60474)

第三种情况，发生了半包现象，即发送方依旧是先后发送了两个数据包 packet1 和 packet2，但是 TCP 在传输时，分了几次传输，每次传输的内容中包含的不是 packet1 和 packet2 的完整包，只是 packet1 或者 packet2 的一部分，就相当于把两个数据包的内容拆分了，因此也称之为拆包现象。如图 3 所示。


![半包](https://user-gold-cdn.xitu.io/2019/12/29/16f51672f014857c?w=1234&h=698&f=png&s=158054)

### 产生粘包、半包的原因

从上面的示意图中，我们大致可以知道产生粘包、半包的主要原因如下。

- 粘包原因
  - 1. 发送方每次写入的数据小于套接字缓冲区大小；
  - 2. 接收方读取套接字缓冲区的数据不够及时。
- 半包原因
  - 1. 发送方写入的数据大于套接字缓冲区的大小；
  - 2. 发送的数据大于协议的 MSS 或者 MTU，必须拆包。（MSS 是 TCP 层的最大分段大小，TCP 层发送给 IP 层的数据不能超过该值；MTU 是最大传输单元，是物理层提供给上层一次最大传输数据的大小，用来限制 IP 层的数据传输大小）。

但归根结底，产生粘包、半包的根本原因是因为 TCP 是基于字节流来传输数据的，数据包相互之间没有边界，导致接收方无法准确的分辨出每一个单独的数据包。

### netty 如何解决粘包拆包问题

作为一个应用层的开发者，我们无法去改变 TCP 基于字节流来传输数据的特性，除非我们自定义一个类似于 TCP 的协议，但是难度太大，设计出来的性能还不一定比现有的 TCP 协议性能好，况且目前 TCP 协议的使用十分广泛。而 netty 作为一款高性能的网络框架，必然就要有对 TCP 协议的支持，既然支持 TCP 协议，那就要解决 TCP 中粘包、半包的问题，否则如果开发人员自己去解决，那就费时费力了。

netty 中通过提供一系列的编解码器来解决 TCP 的粘包、半包问题，顾名思义，编解码器就是通过将从 TCP 套接字中读取的字节流通过一定的规则，将其进行编码或者解码，编码成二进制字节流或者解析出一个个完整的数据包。在 netty 中提供了很多通用的编解码器，对于解码器而言，它们均继承自抽象类**ByteToMessageDecoder**；对于编码器而言，它们均继承与抽象类**MessageToByteEncoder**。

今天主要先简单分析下抽象类解码器**ByteToMessageDecoder**类的源码，对于具体的解码器实现将在后面两篇文章中详细分析其原理，对于编码器而言，编码过程与解码过程恰好相反，因此就不再单独赘述，有兴趣的朋友可以自行阅读，欢迎分享。

**ByteToMessageDecoder**实际上就是一个 ChannelHandler，它的具体实现类需要被添加到 pipeline 中才会起作用。当将解码器添加到 pipeline 中后，当出现 **OP_READ** 事件时，就会通过 pipeline 传播执行所有 handler 的 **channelRead()** 方法。在抽象类解码器中，就定义了 channelRead()方法。其源码如下。

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        // 用来存放解码出来的数据对象，可以将它当做一个集合
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            first = cumulation == null;
            if (first) {
                //第一次直接赋值
                cumulation = data;
            } else {
                //累加数据
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            }
            // 调用解码的方法
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            // 省略部分代码...

            // size是解码出来的数据对象的数量，
            int size = out.size();
            decodeWasNull = !out.insertSinceRecycled();
            // 向下传播，如果size为0，就表示没有解码出一个对象，因此不会向下传播，而是等到下一次继续读到数据后解码
            fireChannelRead(ctx, out, size);
            out.recycle();
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}
```

netty 中解码器的核心逻辑就是先通过一个累加器将读到的字节流数据累计，然后调用解码器的实现类对累加到的数据进行解码，如果能解码出一个数据对象，就表示读到了一个完整的数据包，那就将解码出来的数据对象沿着 pipeline 向下传播，交由业务代码去执行后面的业务逻辑。如果没能解码出一个数据对象，那就表示还没有读到一个完整的数据包，就不向下进行传播，而是等待下一次继续有数据读，继续累加字节流数据，直到累加到的数据能解码出一个数据对象，然后再向下传播。

这里有 3 个比较重要的点，第一：字节流数据的累加器；第二：具体的解码操作，这一步是在具体的解码器实现类中完成的；第三：解码出来的数据对象，在由具体的解码器将字节流数据解码出数据对象后，会将对象存放到一个 **list** 中，然后将数据对象沿着 pipeline 向下传播。接下来我们结合上面的源码，来分析下这几个步骤。

首先会判断 **msg** 是否是 **ByteBuf** 类型的对象，对于已经解码出来的字节流数据，此时不会是 ByteBuf 类型，因此也不需要进行解码，从而进入 else 逻辑中，直接将数据对象向下进行传播。对于没有被解码的字节流数据，此时 msg 就是 ByteBuf 类型，因此会进入到 if 逻辑块中进行解码。

在 if 逻辑中，先定义了一个**out**对象，这个对象可以简单的把它当做一个集合对象，它用来存放成功解码出来的数据对象，也就是上面第三点中提到的 **list**。接下来会碰到 **cumulation** 这个对象，它就是一个字节流数据累加器，默认值为**MERGE_CUMULATOR**，通过判断它是否为空，从而知道是否是第一次读取数据，如果为空，表示前面没有累加数据，因此直接让 msg 等于 **cumulation**，意思就是将当前的字节流数据 msg 全部累加到累加器 cumulation 中；如果累加器不为空，表示前面累加器中存在一部分数据（前面出现了 TCP 半包现象），因此需要将当前读到的字节流数据 msg 累加到累加器中。

如何累加字节流数据到累加器中的呢？那就是调用累加器的 **cumulate()** 方法，这里采用的是策略设计模式，默认的累加器为**MERGE_CUMULATOR**，其源码如下。

```java
public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
    try {
        final ByteBuf buffer;
        // 如果因为空间满了写不了本次的新数据 就扩容
        // cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes() 可以装换为如下：
        // cumulation.writeIndex() + in.readableBytes()>cumulation.maxCapacity
        // 即 写指针的位置+可读的数据的长度，如果超过了ByteBuf的最大长度，那么就需要扩容
        if (cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes()
                || cumulation.refCnt() > 1 || cumulation.isReadOnly()) {
            // 扩容
            buffer = expandCumulation(alloc, cumulation, in.readableBytes());
        } else {
            buffer = cumulation;
        }
        // 将新数据写入
        buffer.writeBytes(in);
        return buffer;
    } finally {
        // 释放内存，防止OOM
        in.release();
    }
}
```

在代码中可以看到，在将数据累加到累加器中之前，会先判断是否需要扩容，如果需要扩容，就调用 **expandCumulation()** 方法先进行扩容。最后调用 **writeBytes()** 方法将数据写入到累加器中，然后将累加器返回。关于扩容的方法，由于这里的累加器是**MERGE_CUMULATOR**，因此其底层就是进行内存复制。在 netty 中还提供了另一种类型的累加器：**COMPOSITE_CUMULATOR**，它扩容的时候不需要进行内存复制，而是通过组合 ByteBuf，即**CompositeByteBuf**类来实现扩容的。

那么问题来了，显然基于内存复制的操作会更慢一点，那 netty 为什么会默认使用基于内存复制的累加器呢？netty 源码里面给的解释如下：

```properties
/**
 * Cumulate {@link ByteBuf}s by add them to a {@link CompositeByteBuf} and so do no memory copy whenever possible.
 * Be aware that {@link CompositeByteBuf} use a more complex indexing implementation so depending on your use-case
 * and the decoder implementation this may be slower then just use the {@link #MERGE_CUMULATOR}.
 */
```

大致意思就是：累加器只是累加数据，具体的解码操作是由抽象解码器的实现类来做的，对于解码器的实现类，此时我们并不知道解码器的实现类具体是如何进行解码的，可能它基于 **CompositeByteBuf** 类型数据结构，解码起来会更慢，并不一定比直接使用 ByteBuf 快，效率不一定高，因此 netty 默认就直接使用了**MERGE_CUMULATOR**，也就是基于内存复制的累加器。

当将数据累加到累计器后，就会调用**callDecode(ctx, cumulation, out)** 来进行解码了。其精简后的源码如下。

```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        // 只要有数据可读，就循环读取数据
        while (in.isReadable()) {
            int outSize = out.size();
            // 如果out中有对象，这说明已经解码出一个数据对象了，可以向下传播了
            if (outSize > 0) {
                // 向下传播，并清空out
                fireChannelRead(ctx, out, outSize);
                out.clear();
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }
            // 记录下解码之前的可读字节数
            int oldInputLength = in.readableBytes();
            // 调用解码的方法
            decodeRemovalReentryProtection(ctx, in, out);
            if (ctx.isRemoved()) {
                break;
            }
            // 如果解码前后，out中对象的数量没变，这表明没有解码出新的对象
            if (outSize == out.size()) {
                // 当没解码出新的对象时，累计器中可读的字节数在解码前后也没变，说明本次while循环读到的数据，
                // 不够解码出一个对象，因此中断循环，等待下一次读到数据
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    continue;
                }
            }
            // out中的对象数量变了，说明解码除了新的对象，但是解码前后，累计器中的可读数据并没有变化，这表示出现了异常
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            }

            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```

该方法的第二个参数 **in** 就是前面我们提到的累积器，第三个参数 **out**，就是前面提到的存储成功解码出来的数据对象。该方法的逻辑可以参考上面代码中的注释，解码的核心代码在这一行：

```java
// 调用解码的方法
decodeRemovalReentryProtection(ctx, in, out);
```

这个方法的源代码如下。在它的代码中，会就会真正去调用解码器子类的 decode()方法，进行数据解码。

```java
final void decodeRemovalReentryProtection(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
        throws Exception {
    decodeState = STATE_CALLING_CHILD_DECODE;
    try {
        //调用子类的解码方法
        decode(ctx, in, out);
    } finally {
        boolean removePending = decodeState == STATE_HANDLER_REMOVED_PENDING;
        decodeState = STATE_INIT;
        if (removePending) {
            handlerRemoved(ctx);
        }
    }
}
```

**decode(ctx, in, out)** 方法是一个抽象方法，它由解码器的具体实现类去实现具体的逻辑。显然，在父类中，调用一个抽象方法，抽象方法具体的逻辑由子类自己实现，这里用到的是模板方法设计模式。netty 中常用的解码器有如下几种，如下所示。关于子类解码的核心逻辑，后面两篇文章分析。

```properties
FixedLengthFrameDecoder（基于固定长度的解码器）
LineBasedFrameDecoder （基于行分隔符的解码器）
DelimiterBasedFrameDecoder （基于自定义分割符的解码器）
LengthFieldBasedFrameDecoder （基于长度字段的解码器）
```

最后回到**channelRead()** 方法的 finally 语句块中（代码如下），会先获取 **out** 中解码出来的数据对象的数量，然后调用 **fireChannelRead()** 方法将解析出来的数据对象向下进行传播处理。如果 size 为 0，就表示没有解码出一个对象，因此不会向下传播，而是等到下一次继续读到数据后解码。

```java
finally {
    // 省略部分代码...

    // size是解码出来的数据对象的数量，
    int size = out.size();
    decodeWasNull = !out.insertSinceRecycled();
    // 向下传播，如果size为0，就表示没有解码出一个对象，因此不会向下传播，而是等到下一次继续读到数据后解码
    fireChannelRead(ctx, out, size);
    out.recycle();
}
```

### 总结

本文先介绍了什么是 TCP 的粘包、半包现象，就是将多个独立的数据包合成或者拆分成多个数据包发送，以及产生粘包、半包现象的根本原因是 TCP 协议是基于字节流传输数据的。然后结合源码介绍了 netty 通过编解码器是如何来解决粘包、半包问题。最后关于具体的解码操作的源码分析，将会在后面两篇文章中分析。
