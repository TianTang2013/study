### 前言

在上一篇文章中分析了三个比较简单的解码器，今天接着分析最后一个常用的解码器：**LengthFieldBasedFrameDecoder**，这是一个基于长度字段的解码器。什么意思呢？就是在发送的数据中，使用一个字段来表示数据的长度，这样当接收方接收到数据后，先读出这个长度字段，读到了长度字段，那就知道了这次发送的数据有多长，这样就能解码出数据了。

### 属性介绍

在 **LengthFieldBasedFrameDecoder** 中有几个重要的属性，其中大分部属性与上一篇文章中提到的解码器中的属性相同，只有 4 个属性是长度字段解码器特有的。如下所示。

```java
// 长度字段的偏移量
private final int lengthFieldOffset;
// 长度字段的长度
private final int lengthFieldLength;
// 长度调整值
private final int lengthAdjustment;
// 跳过的初始字节数
private final int initialBytesToStrip;

// 长度字段结束的偏移量，是通过属性计算出来：lengthFieldOffset + lengthFieldLength
private final int lengthFieldEndOffset;

// ======= 下面属性与其他解码器中类似 =========
// 是否立即抛出失败信息，true表示立即
private final boolean failFast;
// 是否处于丢弃模式，true表示处于丢弃模式
private boolean discardingTooLongFrame;
// 需要丢弃的长度
private long tooLongFrameLength;
// 累计丢弃的字节数
private long bytesToDiscard;
```

既然说 **LengthFieldBasedFrameDecoder** 是基于长度字段的解码器，那么就肯定需要有字段告诉我长度在哪儿，也就是处于这一串数据的哪个位置？这个字段就是 **lengthFieldOffset**，它用来标识长度字段处于这一串数据中的哪一个地方。

另外，数据有长度，是用一个数字来存储的，这个**数字**也需要占用一定的字节数，因此 **lengthFieldLength** 就是用来标识这个数字占用多少字节的变量。

例如如下示例中，长度字段处于第 4 个位置，因此 **lengthFieldOffset = 3**。另外长度字段自身占用一个字节，且表示的值是：**0x0A**，用十进制表示就是 **10**，也就是消息体的长度是 **10** 个字节，从长度字段往后 **10** 个字节就是消息体的具体类容。

![长度字段解码器](https://user-gold-cdn.xitu.io/2019/12/31/16f5bc27926b0f17?w=1410&h=164&f=png&s=27114)

和行解码器和分隔符解码器类似，我们有时候希望解码出来的数据，不包含换行符或者分隔符，因此需要跳过一定的字节数后再解码数据。同样，在基于长度字段的解码器中，也会有这样类似的需求，因此 **initialBytesToStrip** 字段就是用来表示在解码时跳过多少个字节的数据。

通常我们发送的一串数据中，会包含**消息头**和**消息体**，在有些协议中喜欢直接使用长度字段来直接表示**消息体的长度**；而有些协议中，喜欢用长度字段来表示**整个数据的长度**，即：消息头的长度 + 消息体的长度，那么这个时候，数据的接收方在接收到消息后，想要知道消息体的内容，就无能为力了。怎么办呢？那就需要另外一个字段：**lengthAdjustment**。这个字段翻译过来就是长度的调整值，如果长度表示的是整个消息的长度，那么将 **lengthAdjustment** 设置为负数，用整个消息的长度加上这个负数，那就是消息体的长度了。

### 示例

在 **LengthFieldBasedFrameDecoder** 的源码中，官方给出了很多示例，下面结合官方给出的示例，来解释下上面 4 个重要属性的含义吧。

#### 示例 1

当长度字段处于数据的第一个位置，后面是消息体。长度字段占用两个字节（0x000C 这是一个 16 进制数，表示的是数值是 12，占用两个字节）。解码之后的数据如下右边所示。

```java
lengthFieldOffset = 0
lengthFieldLength = 2
lengthAdjustment  = 0
initialBytesToStrip = 0
BEFORE DECODE (14 bytes)         AFTER DECODE (14 bytes)
+--------+----------------+      +--------+----------------+
| Length | Actual Content |----->| Length | Actual Content |
| 0x000C | "HELLO, WORLD" |     | 0x000C | "HELLO, WORLD" |
+--------+----------------+      +--------+----------------+
```

#### 示例 2

当长度字段处于数据的第一个位置，后面是消息体。长度字段占用两个字节，由于 **initialBytesToStrip = 2**，因此在解码时需要跳过两个字节，所以解码之后的数据就不包含 0x000C 了，结果如下右边所示。

```java
lengthFieldOffset   = 0
lengthFieldLength   = 2
lengthAdjustment    = 0
initialBytesToStrip = 2
BEFORE DECODE (14 bytes)         AFTER DECODE (12 bytes)
+--------+----------------+      +----------------+
| Length | Actual Content |----->| Actual Content |
| 0x000C | "HELLO, WORLD" |      | "HELLO, WORLD" |
+--------+----------------+      +----------------+
```

#### 示例 3

前面两个示例中，长度字段直接表示的是消息体的长度，但是在有些场景下，有的协议会使用长度字段表示整个数据的长度（**消息头+消息体**），这个时候，就需要在这个长度数值上做一些调整了，在长度基础上加上一个负数（等价于减去一个正数），得到的结果就是消息体的长度了。例如如下示例中，长度值 0x000E 的值是 14，包含了 2 个字节的长度字段，因此令 **lengthAdjustment** 为-2，这样消息体的长度就是 14+(-2) =12 个字节。

```java
lengthFieldOffset   =  0
lengthFieldLength   =  2
lengthAdjustment = -2
initialBytesToStrip =  0
BEFORE DECODE (14 bytes)         AFTER DECODE (14 bytes)
+--------+----------------+      +--------+----------------+
| Length | Actual Content |----->| Length | Actual Content |
| 0x000E | "HELLO, WORLD" |      | 0x000E | "HELLO, WORLD" |
+--------+----------------+      +--------+----------------+
```

#### 示例 4

在部分场景下，在发送数据时，业务要求上可能需要在长度字段的前面添加一些请求头内容，例如如下所示，在数据的最前面添加了两个字节长度的消息头（0xCAFE），这样长度字段就不处于第一个位置了，因此就需要令 **lengthFieldOffset=2** 来标识长度字段的在数据中的偏移量是 2。

```java
lengthFieldOffset = 2
lengthFieldLength = 3
lengthAdjustment  = 0
initialBytesToStrip = 0
BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
+----------+----------+----------------+      +----------+----------+----------------+
| Header 1 |  Length  | Actual Content |----->| Header 1 |  Length  | Actual Content |
|  0xCAFE  | 0x00000C | "HELLO, WORLD" |     |  0xCAFE  | 0x00000C | "HELLO, WORLD" |
+----------+----------+----------------+      +----------+----------+----------------+
```

#### 示例 5

数据中包含了消息头，但是消息头不处于第一个位置，而是处于长度字段和消息体的中间。长度字段的含义是从长度字段往后数多少个长度的字节，这段数据就是消息体的内容。由于此时在中间夹杂了请求头，再这样计算消息体就不对了，因此就需要使用长度调整值，令 **lengthAdjustment=2**，这样就能准确知道消息体的内容了。

```java
lengthFieldOffset   = 0
lengthFieldLength   = 3
lengthAdjustment    = 2
initialBytesToStrip = 0
BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
+----------+----------+----------------+      +----------+----------+----------------+
|  Length  | Header 1 | Actual Content |----->|  Length  | Header 1 | Actual Content |
| 0x00000C |  0xCAFE  | "HELLO, WORLD" |      | 0x00000C |  0xCAFE  | "HELLO, WORLD" |
+----------+----------+----------------+      +----------+----------+----------------+
```

#### 示例 6

在有些场景下，有多个请求头，而且长度字段处于这些请求头的中间，例如如下示例，有两个请求头 **HDR1** 和 **HDR2**，长度字段处于这两个请求头中间，这个时候就需要使用 **lengthFieldOffset** 来标识长度字段处于第几个字节处。由于长度字段后面还有一个请求头，因此需要使用 **lengthAdjustment** 来进行长度的调整。最后，由于 **initialBytesToStrip** 等于 3，这表示解码是，跳过前面 3 个字节，因此解码后的数据不包含 HDR1 和长度字段（以为这两者在下面的示例中，刚好占用了 3 个字节）。

```java
lengthFieldOffset   = 1
lengthFieldLength   = 2
lengthAdjustment    = 1
initialBytesToStrip = 3
BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
+------+--------+------+----------------+      +------+----------------+
| HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
| 0xCA | 0x000C | 0xFE | "HELLO, WORLD" |     | 0xFE | "HELLO, WORLD" |
+------+--------+------+----------------+      +------+----------------+
```

#### 示例 7

和示例 6 一样，数据中包含多个请求头，不同的是，此时长度字段表示的长度是整个数据的长度（0x0010 的值用十进制表示就是 16），这个时候时候就需要令长度调整值为负数了，例如如下示例中，令 **lengthAdjustment=-3**，那么消息体+HDR2 的长度就是 13 了。

```java
lengthFieldOffset   =  1
lengthFieldLength   =  2
lengthAdjustment    = -3
initialBytesToStrip =  3
BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
+------+--------+------+----------------+      +------+----------------+
| HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
| 0xCA | 0x0010 | 0xFE | "HELLO, WORLD" |     | 0xFE | "HELLO, WORLD" |
+------+--------+------+----------------+      +------+----------------+
```

### 源码分析

看了那么多示例，接下来看看源码时如何实现。同样 **LengthFieldBasedFrameDecoder** 继承了抽象类 **ByteToMessageDecoder**，因此会重写抽象方法 **decode()**。

```java
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    // 调用重载的decode()方法解码
    Object decoded = decode(ctx, in);
    // 能成功解码出对象，就放进out中
    if (decoded != null) {
        out.add(decoded);
    }
}
```

核心逻辑在重载的 **decode(ctx, in)** 方法中，在 netty-4.1.16 以前的版本中，该方法的逻辑看起来特别复杂（从写法上），但从 4.1.16 版本开始，这段代码看起来就比较简洁了，下面给出该方法的源码，可以结合代码中注释阅读。

```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
    // 判断是否处于丢弃模式
    if (discardingTooLongFrame) {
        discardingTooLongFrame(in);
    }

    // 可读字节数小于长度字段的结束索引，肯定没法解码出数据
    //  lengthFieldEndOffset = lengthFieldOffset + lengthFieldLength
    if (in.readableBytes() < lengthFieldEndOffset) {
        return null;
    }

    // 获取到长度字段的位置
    int actualLengthFieldOffset = in.readerIndex() + lengthFieldOffset;

    // 根据长度字段的位置，以及长度字段的长度，计算出长度
    long frameLength = getUnadjustedFrameLength(in, actualLengthFieldOffset, lengthFieldLength, byteOrder);

    // 计算出的长度值小于0，肯定不合法，解码失败
    if (frameLength < 0) {
        failOnNegativeLengthField(in, frameLength, lengthFieldEndOffset);
    }

    // 根据lengthAdjustment进行长度调整
    frameLength += lengthAdjustment + lengthFieldEndOffset;

    if (frameLength < lengthFieldEndOffset) {
        failOnFrameLengthLessThanLengthFieldEndOffset(in, frameLength, lengthFieldEndOffset);
    }

    // 超过最大长度
    if (frameLength > maxFrameLength) {
        exceededFrameLength(in, frameLength);
        return null;
    }

    // never overflows because it's less than maxFrameLength
    int frameLengthInt = (int) frameLength;
    if (in.readableBytes() < frameLengthInt) {
        return null;
    }
    // 跳过的字节数大于长度，说明数据不合法，无法解码出数据
    if (initialBytesToStrip > frameLengthInt) {
        failOnFrameLengthLessThanInitialBytesToStrip(in, frameLength, initialBytesToStrip);
    }
    // 跳过指定的字节数
    in.skipBytes(initialBytesToStrip);

    // extract frame
    // 解码，就是根据索引范围，从字节数组中，读取出数据
    int readerIndex = in.readerIndex();
    int actualFrameLength = frameLengthInt - initialBytesToStrip;
    ByteBuf frame = extractFrame(ctx, in, readerIndex, actualFrameLength);
    // 移动读指针
    in.readerIndex(readerIndex + actualFrameLength);
    return frame;
}
```

在上面方法中，有一个需要注意的地方是，在计算数据的长度时，需要根据长度字段的位置，以及长度字段的长度来进行计算，其源码如下。

```java
protected long getUnadjustedFrameLength(ByteBuf buf, int offset, int length, ByteOrder order) {
    buf = buf.order(order);
    long frameLength;
    switch (length) {
    case 1:
        frameLength = buf.getUnsignedByte(offset);
        break;
    case 2:
        frameLength = buf.getUnsignedShort(offset);
        break;
    case 3:
        frameLength = buf.getUnsignedMedium(offset);
        break;
    case 4:
        frameLength = buf.getUnsignedInt(offset);
        break;
    case 8:
        frameLength = buf.getLong(offset);
        break;
    default:
        throw new DecoderException(
                "unsupported lengthFieldLength: " + lengthFieldLength + " (expected: 1, 2, 3, 4, or 8)");
    }
    return frameLength;
}
```

从源码中我们可以发现，长度字段的长度只能是 **1、2、3、4、8**，如果是其他值就会抛出异常。至于为什么只能是这几种值，在 netty 的源码注释中，并没有找到答案，本人想了很久，也没弄清楚为什么要这么做，有兴趣的朋友可以深入研究下。

### 总结

本位主要通过分析基于长度字段解码器的 4 个重要属性，解释了 **LengthFieldBasedFrameDecoder** 的实现原理，结合源码中给出 7 个示例，详细说明了每一个属性的使用场景以及用途，最后简单分析了 **decode()**方法的源码。

至此，netty 中 4 个比较常用的解码器的源码已经分析完毕了。事实上 netty 中还提供了很多其他解码器，这些解码器已经足够我们在实际应用中使用了，不需要我们再单独自定义，难度较大不说，还容易出 bug，除非万不得已。
