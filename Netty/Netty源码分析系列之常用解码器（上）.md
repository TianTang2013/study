> 扫描下方二维码或者微信搜索公众号`菜鸟飞呀飞`，即可关注微信公众号，阅读更多`Spring源码分析`和`Java并发编程`文章。

![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144)

### 前言

在上一篇文章中，只分析了 netty 如何通过编解码器解决了 TCP 粘包、半包的问题，没有具体分析解码器是如何来对数据进行解码的，今天本文将具体分析这些解码器的工作原理。

netty 为我们提供了几个十分常用的解码器，这几个解码器几乎能满足我们所有的场景，这几个解码器根据难易程度，从上到下，如下表所示。

```properties
FixedLengthFrameDecoder（基于固定长度的解码器）
LineBasedFrameDecoder （基于行分隔符的解码器）
DelimiterBasedFrameDecoder （基于自定义分割符的解码器）
LengthFieldBasedFrameDecoder （基于长度字段的解码器）
```

前三个解码器比较简单，也容易理解，最后一个解码器相对而言比较复杂，也不容易理解，但它能满足的场景却是最多的。由于前三个解码器比较简单，因此将它们的源码放在一篇文章中分析，也就是今天本文的主要内容。最后一个解码器的源码将在后面一篇文章中单独分析。

### FixedLengthFrameDecoder

根据类名，翻译过来就能知道这是一个基于固定长度的解码器，什么意思呢？就是在初始化这个解码器时，指定一个 int 类型的数值：**frameLength**，后面在解码时，每当读到 **frameLength** 长度的字节时，就解码出一个数据对象。例如：当发送方发送了四次数据，分别是 A、BC、DEFG、HI，一共 9 个字节，如果我们指定解码器的固定长度 **frameLength = 3**，那么就表示每 3 个字节解一次码，那么解码出来的结果结果就是：**ABC**、**DEF**、**GHI**。

```java
+---+----+------+----+          +-----+-----+-----+
| A | BC | DEFG | HI |   ->    | ABC | DEF | GHI |
+---+----+------+----+          +-----+-----+-----+
```

基于固定长度的解码器的源码和注释如下，比较简答，就不做展开分析了，参考源码中的注释即可。

```java
public class FixedLengthFrameDecoder extends ByteToMessageDecoder {

    // 表示每次解码多长的数据
    private final int frameLength;

    public FixedLengthFrameDecoder(int frameLength) {
        checkPositive(frameLength, "frameLength");
        // 指定每次解码的字节数
        this.frameLength = frameLength;
    }

    @Override
    protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        // 解码
        Object decoded = decode(ctx, in);
        // 如果解码出来的数据对象不为空，就将其保存到out这个集合中
        if (decoded != null) {
            out.add(decoded);
        }
    }

    protected Object decode(
            @SuppressWarnings("UnusedParameters") ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        // 如果可读取的数据小于每次解码的长度，那就直接返回null
        if (in.readableBytes() < frameLength) {
            return null;
        } else {
            // 读取指定长度的字节数据，然后返回
            return in.readRetainedSlice(frameLength);
        }
    }
}
```

### LineBaseFrameDecoder

**LineBaseFrameDecoder**是基于行分割符的解码器。什么意思呢？就是每当读到行分隔符（\n 或者\r\n）的时候，就解析出一个数据对象。例如如下示例。

```java
+---+-------+----------+------+          +-----+-----+-------+
| A | B\nC | DE\r\nFG | HI\n |   ->      | AB  | CDE | FGHI |
+---+-------+----------+------+          +-----+-----+-------+
```

通过上面的示例，**LineBaseFrameDecoder**的原理看起来比较简单，但实际上，在具体实现上并没有像上面表现得那么简单。在**LineBaseFrameDecoder**定义了几个十分重要的成员变量。如下所示。

```java
// 解码的最大长度
private final int maxLength;

// 当通过换行符读出来的数据超过maxLength规定的长度后，是否立即抛出异常。true表示立即
private final boolean failFast;

// 解析数据时是否跳过换行符\r\n或者\n，true表示跳过，false表示不跳过
private final boolean stripDelimiter;

// 当超过maxLength的长度后，就不能解码，需要丢弃数据，此时会将discarding设置为true，表示丢弃数据
private boolean discarding;

// 记录已经丢弃了对少字节的数据
private int discardedBytes;

// 最后一次扫描的位置
private int offset;
```

当在解码数据时，会先找到**换行符**的位置，然后计算从当前读指针的位置到换行符位置的长度，如果这个长度大于 **maxLength**，那么就表示这是一个无效数据，不能进行解码，需要丢弃。例如我们设置 **maxLength = 4**时，在如下图所示的示例中，只会解码出两个正确的数据包：**AB**、**CDEF**，而对于**GHIJBCA**，因为它的长度为 6，超过了 **maxLength** 规定的长度，因此会被丢弃。

![行解码器示例](https://user-gold-cdn.xitu.io/2019/12/30/16f5769be800ea0b?w=1792&h=210&f=png&s=37357)

在解码数据时，对于解码之后的数据是否保留换行符\r\n 或者\n，可以通过 **stripDelimiter** 属性控制，true 表示跳过换行符，解码出来的数据不保留换行符。另外当通过换行符读取出来的数据长度超过 **maxLength** 后，就需要丢弃数据，那么什么时候丢弃数据呢？是立即丢弃数据？还是等到下次读数据时丢弃呢？这可以通过 **failFast** 属性来控制，true 表示立即丢弃。同时，如果需要丢弃数据，就会将 **discarding** 属性这是为 true。

下面结合源码看下换行符解码器的解码过程。换行符解码器继承了上一篇文章中提到的抽象类解码器 **ByteToMessageDecoder**，重写了抽象方法 **decode()**。

```java
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    // 解码
    Object decoded = decode(ctx, in);
    // 能解码出来数据，就将解码出来的结果存放到out中
    if (decoded != null) {
        out.add(decoded);
    }
}
```

可以看到，核心逻辑在另一个重载的 **decode()** 方法中。这个重载方法的源码很长，将其整理了一下，整体骨架如下。

```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    // 返回\n或者\r\n的下标位置
    final int eol = findEndOfLine(buffer);
    if (!discarding) {
        // 找到换行符
        if (eol >= 0) {
            // 解码...
            return frame;
        } else {
            // ...
            return null;
        }
    } else {
        // 找到了换行符
        if (eol >= 0) {
            // ...
        } else {
            // ...
        }
        return null;
    }
}
```

首先会寻找出\n 或者\r\n 的下标位置，这个查找过程比较简单，就是遍历字节数组。如果没找到换行符，那么就会返回一个小于 0 的数值，如果找到了换行符，就会返回一个大于等于 0 的数。其中，如果找到的是\n，那么返回的是\n 的索引值；如果找到的是\r\n，那么返回的是\r 的索引值。

接着剩下的逻辑，就可以分为两部分了：**是否处于丢弃模式**。第一部分就是不处于丢弃模式下执行的逻辑，即：**discarding = false**，那么 !discarding 就为 true；第二部分就是处于丢弃模式下执行的逻辑。对于这两部分，每一部分又可以分为两种情况：**找到了换行符** 和 **没有找到换行符** ，因此这里实际上是四种逻辑。当第一调用解码器的解码方法时，此时 **discarding = false** ，即处于非丢弃模式。

第一种情况：**非丢弃模式且找到换行符（eol >= 0）**，这种情况对应执行的具体代码如下所示。首先会计算出读指针到换行符之间数据的长度 length，然后判断这段的数据长度是否超过 maxLength 的限制，如果超过了，则表示数据是非法的，因此需要丢弃这段数据。如何丢弃呢？就是将 ByteBuf 的读指针移到换行符之后，然后调用 fail()方法，进行异常处理。如果数据没有超过最大限制，那么就表示数据是合法的，可以进行正常解码。接着在解码时，会根据 **stripDelimiter** 来判断是否保留换行符，最后将解码后的数据赋值给 frame，然后返回。这种情况是最理想的状况。

```java
// 非丢弃模式且找到换行符
if (eol >= 0) {
    final ByteBuf frame;
    // 计算出要截取的数据长度
    final int length = eol - buffer.readerIndex();
    // 判断是\r\n还是\n，如果是\r\n返回2，如果是\n返回1
    final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;

    // 如果读取的数据长度，超过最大长度，那么就不能读这一段数据，需要跳过这段数据，即将读指针直接\n之后。
    if (length > maxLength) {
        // 跳过这段数据
        buffer.readerIndex(eol + delimLength);
        // 进入失败模式
        fail(ctx, length);
        return null;
    }

    // 是否跳过换行符
    if (stripDelimiter) {
        // 读数据时不读取换行符
        frame = buffer.readRetainedSlice(length);
        // 跳过换行符
        buffer.skipBytes(delimLength);
    } else {
        // 读取到的数据包含换行符
        frame = buffer.readRetainedSlice(length + delimLength);
    }
    return frame;
}
```

第二种情况，**非丢弃模式但没有找到换行符（eol < 0）** ，这种情况对应执行的具体代码如下所示。因为此时没有找到换行符，所以肯定是解码不出来数据的。但是由于我们有 maxLength 的限制，所以此时需要判断一下当前 buffer 中可读的数据是否超过了这个最大限制。如果超过了，那数据肯定就不合法了，所以这段数据 **全部** 需要被丢弃，什么时候丢弃呢？是现在立即丢弃还是下一次来解码数据时丢？这取决于 failFast 成员变量的值，true 表示立即丢弃，false 则表示下一次解码时丢弃。

```java
else {
    // 没有找到换行符，则判断可读的数据长度是否超过最大长度，如果超过，则需要丢弃数据
    final int length = buffer.readableBytes();
    if (length > maxLength) {
        // 设置丢弃的长度为本次buffer的可读取长度
        discardedBytes = length;
        // 修改读指针，跳过这段数据
        buffer.readerIndex(buffer.writerIndex());
        // 设置为丢弃模式
        discarding = true;
        offset = 0;
        // 是否快速进入丢弃模式
        if (failFast) {
            fail(ctx, "over " + discardedBytes);
        }
    }
    return null;
}
```

第三种情况，**丢弃模式且找到了换行符（eol >= 0）**，对应的代码如下。由于在父类解码其中会循环调用子类的解码方法 decode()，所以当前面出现需要丢弃数据时，就会进入丢弃模式中。此时虽然找到了换行符，由于前一次的数据是需要被丢弃的，所以此时，会将本次找到的换行符之前的数据全部丢弃（包括上一次循环中需要被丢弃的数据），最后将丢弃模式设置为 false，因为此时已经将数据丢弃过了，下一次循环读的时候，就是正常解码判断了。

```java
// 丢弃模式且找到了换行符
if (eol >= 0) {
    // 以前丢弃的数据长度+本次可读的数据长度
    final int length = discardedBytes + eol - buffer.readerIndex();
    // 拿到分隔符的长度
    final int delimLength = buffer.getByte(eol) == '\r' ? 2 : 1;
    // 跳过丢弃的数据
    buffer.readerIndex(eol + delimLength);
    // 设置丢弃数据长度为0
    discardedBytes = 0;
    // 设置非丢弃模式
    discarding = false;
    if (!failFast) {
        fail(ctx, length);
    }
}
```

第四种情况，**丢弃模式且没有找到换行符（eol < 0）**，对应的代码如下。此时因为没有找到换行符，所以肯定不能正确解码，而且又处于丢弃模式，因此本次读到的数据全部都是无效的，都需要被丢弃，然而在这一段的代码中，我们发现，并没有立即丢弃数据，为什么呢？因为还要丢弃掉下一次读取到的数据的前半部分，如果此时将数据丢弃了，那么下一次读取数据时，可能找到的数据长度小于 maxLength 规定的长度，这样我们就会将它拿去解码，实际上这段数据是不可用的。

```java
else {
    // 没找到换行符
    // 以前丢弃的数据 + 本次所有可读的数据
    discardedBytes += buffer.readableBytes();
    // 跳过本次所有可读的数据
    buffer.readerIndex(buffer.writerIndex());
    // 我们跳过缓冲区中的所有内容，需要再次将偏移量设置为0。
    offset = 0;
}
```

无论是第三种情况，还是第四种情况，只要处于丢弃模式，都不能正常解码，所以最后返回的是 null，即没有解码出对象。

从前面的分析中，可以看到，当要丢弃数据时，会调用 fail()方法，它有几个重载的方法，但最终都会调用到如下重载的方法。

```java
private void fail(final ChannelHandlerContext ctx, String length) {
    ctx.fireExceptionCaught(
            new TooLongFrameException(
                    "frame length (" + length + ") exceeds the allowed maximum (" + maxLength + ')'));
}
```

实际上就是创建了一个异常，这个异常信息，在实际应用中我们可能会经常见到，然后通过 pipeline 向下传播这个异常，最终会调用到 handler 的 **exceptionCaught()** 方法。

整体来讲，**LineBaseFrameDecoder**这个基于换行符的解码器，实现思路上比较简单，就是根据\r\n 或者\n 来进行数据分割，如果分割数来的数据大于规定的最大长度，那就将数据丢弃，否则表示解码成功。

### DelimiterBasedFrameDecoder

**DelimiterBasedFrameDecoder** 是一个基于分隔符的解码器，也就是根据用户自己指定的分割符来进行解码，如果用户将分割符定义为 **分号;** ，那就表示根据分号来解码数据。另外，用户可以同时指定多个分割符，只要在读取数据时，碰到了其中任意一个分隔符，就可以进行一次解码。例如下图所示，指定分割符为**英文逗号、感叹号、分号、换行符**，那么解码结果为：**AB**、**CDEF**、**AGHI**、**BCA**。

![基于分隔符的解码器示例](https://user-gold-cdn.xitu.io/2019/12/30/16f576a2295295af?w=1748&h=160&f=png&s=31079)

另外，当且仅当，只指定两个分隔符，且是\r\n 和\n 时，基于分隔符的解码器就变成了一个基于行的解码器。那么在解码时，就直接使用行分割器 **LineBaseFrameDecoder** 解码。

在 **DelimiterBasedFrameDecoder** 定义了几个十分重要的属性，这几个属性的含义以及用途与 **LineBaseFrameDecoder** 解码器中定义的成员变量几乎一样。下面给出了这些成员变量的含义以及作用。

```java
// 分隔符数组，因为可以同时制定过个分隔符，所以使用数组来存放
private final ByteBuf[] delimiters;

// 最大长度限制
private final int maxFrameLength;

// 是否跳过分隔符，true表示跳过
private final boolean stripDelimiter;

// 是否立即丢弃，true：立即
private final boolean failFast;

// 是否处于丢弃模式
private boolean discardingTooLongFrame;

// 累计丢弃的字节数
private int tooLongFrameLength;

/** Set only when decoding with "\n" and "\r\n" as the delimiter.  */
// 如果分隔符是\r\n和\n时，就直接使用基于行的解码器
private final LineBasedFrameDecoder lineBasedDecoder;
```

可以看到，与 **LineBaseFrameDecoder** 解码器不同的是，分隔符解码器多了两个成员变量，一个是 **delimiters** 属性，这是一个数组，用来存放用户自定义的分隔符，因为用户可以自定义多个分隔符，所以使用数组来存放。另外一个就是 **lineBasedDecoder** 属性，这个属性表示的是基于行的解码器，delimiters 数组中的分隔符当且仅当为 \r\n 和 \n 时，分隔符解码器就变成了行解码器，该属性的值在**LineBaseFrameDecoder**构造方法中被初始化。源码如下。

```java
public DelimiterBasedFrameDecoder(
        int maxFrameLength, boolean stripDelimiter, boolean failFast, ByteBuf... delimiters) {

    // 省略其他代码...
    if (isLineBased(delimiters) && !isSubclass()) {
        lineBasedDecoder = new LineBasedFrameDecoder(maxFrameLength, stripDelimiter, failFast);
        this.delimiters = null;
    } else {
        // 省略其他代码...
        lineBasedDecoder = null;
    }
    // 省略其他代码...
}
```

在构造方法中，会通过 **isLineBased(delimiters)** 方法来判断分隔符是否是 \r\n 和 \n，如果是，就创建一个行解码器，然后赋值给 **lineBasedDecoder** 属性；否则就令 **lineBasedDecoder** 属性为空。**isLineBased()** 方法的源码如下。

```java
private static boolean isLineBased(final ByteBuf[] delimiters) {
    // 当分隔符数组中只包含\r\n和\n时，才返回true
    if (delimiters.length != 2) {
        return false;
    }

    ByteBuf a = delimiters[0];
    ByteBuf b = delimiters[1];
    // 保证令a = \r\n，令b= \n
    if (a.capacity() < b.capacity()) {
        a = delimiters[1];
        b = delimiters[0];
    }
    return a.capacity() == 2 && b.capacity() == 1
            && a.getByte(0) == '\r' && a.getByte(1) == '\n'
            && b.getByte(0) == '\n';
}
```

从 **isLineBased()** 方法的源码中就可以看到，当且仅当分隔符是 \r\n 和 \n 时，才返回 true，也就是将分隔符解码器变成基于行的解码器。

同样，分隔符解码器重写了父类中的抽象方法 decode()。

```java
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    // 调用decode()的重载方法解码
    Object decoded = decode(ctx, in);
    // 如果能成功解码，就添加到out中
    if (decoded != null) {
        out.add(decoded);
    }
}
```

核心逻辑在重载的 **decode(ctx,in)** 方法中。同样该方法的源码很长，我进行了精简，为了方便阅读，将代码稍微进行了改动，大致骨架如下。

```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    // 首先判断基于行的解码器是否被初始化了，如果被初始化了，就表明分隔符就是\r\n和\n，直接只用行解码器进行解码
    if (lineBasedDecoder != null) {
        return lineBasedDecoder.decode(ctx, buffer);
    }
    int minFrameLength = Integer.MAX_VALUE;
    ByteBuf minDelim = null;
    // 遍历所有的分隔符，然后找到最小位置的分割符
    for (ByteBuf delim: delimiters) {
        // 找到最小分隔符的位置
    }
    // 如果找到了分割符
    if (minDelim != null) {

        //如果处于丢弃模式
        if (discardingTooLongFrame) {

            return null;
        }else{
            return frame;
        }
    } else {
        //如果没有找到分割符
        //判断是否处于非丢弃模式
        if (!discardingTooLongFrame) {

        } else {

        }
        return null;
    }
}
```

首先会判断 **lineBasedDecoder** 是否为空，如果不为空，就表示分隔符是 \r\n 和 \n，那就直接使用行解码器解码；否则进入后面的逻辑。

与前面分析的行解码器的不同的是，分割符解码器中，因为分割符可以指定多个，所以我们首先需要找到在可读的数据中，最先出现的分隔符是哪一个，以及出现在什么位置，怎么找呢？遍历每一个分隔符，然后分别找到它们在可读数据中的索引，最后看哪一个分隔符的 **索引最小** ，那就是谁是最先出现的分隔符。

后面的逻辑几乎和行解码器一样，也是先分两种情况：**找到分隔符了**、**没有找打分隔符**。然后对于前面每一种情况，再细分为是否处于丢弃模式，所以一共 4 种情况。与行解码器不同的是，行解码器先是判断是否处于丢弃模式，再判断是否找到了分隔符，处理思路都一致。依然是只有找到了分隔符，且解码器不处于丢弃模式，才能解码出数据，否则将返回 null。关于里面具体的细节，就不展开说明了，与前面分析的行解码器是一样的。

### 总结

本文接着上一篇文章，分析了在上篇文章中提到了三种常用的解码器：固定长度的解码器、行解码器、基于分隔符的解码器，这三种解码器比较简单，结合文章中的图片和示例，很容易理解，其实和我们平时在开发中使用的 **str.spilt()** 方法的原理类似，不同点在于，需要对丢弃模式进行判断。下一篇文章将分析 **基于长度字段的解码器**，该解码器稍微比今天分析的三种解码器会复杂一点，但却是最通用的一种解码器。

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

![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144)
