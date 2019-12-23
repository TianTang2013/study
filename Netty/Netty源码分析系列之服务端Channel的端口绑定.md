> 扫描下方二维码或者微信搜索公众号`菜鸟飞呀飞`，即可关注微信公众号，阅读更多`Spring源码分析`和`Java并发编程`文章。

![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144)

### 问题

本文内容是接着前两篇文章写的，有兴趣的朋友可以先去阅读下两篇文章： [Netty 源码分析系列之服务端 Channel 初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw) 和 [Netty 源码分析系列之服务端 Channel 注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)

由于 Netty 是对 JDK 原生 NIO 的封装，对比 JDK 原生 NIO 的写法，我们可以先思考一下以下两个问题。

1. 在 JDK 原生 NIO 写法中，会调用 **serverSocketChannel.bind(new InetSocketAddress(port))** 进行端口号的绑定，那么在 Netty 中，绑定端口号的操作又是在什么地方实现的呢？

2. 在 JDK 原生的 NIO 写法中，在将 ServerSocketChannel 注册到多路复用器 Selector 上时，就将 ServerSocketChannel 感兴趣的事件设置为了 **OP_ACCEPT** 事件，而 Netty 中，将 channel 注册到 Selector 时，将 channel 感兴趣的事件设置的是 **0** ，即对任何事件都不感兴趣。那么在 Netty 中，又是什么时候将服务端 channel 感兴趣的事件设置为 **OP_ACCEPT** 的呢？

JDK 原生 NIO 的写法如下。

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
// 轮询器，不同的操作系统对应不同的实现类
Selector selector = Selector.open();
// 绑定端口
serverSocketChannel.bind(new InetSocketAddress(8080));
serverSocketChannel.configureBlocking(false);
// 将服务端channel注册到轮询器上，并告诉轮询器，自己感兴趣的事件是ACCEPT事件
serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT);
```

### 端口绑定

当调用 **serverBootstrap.bind(port)** 时，会调用到 **AbstractBootstrap.doBind(final SocketAddress localAddress)** 方法。在 doBind(localAddress) 方法中，会先调用 initAndRegister() 方法来初始化服务端 Channel 以及将服务端 channel 注册到多路复用器上，该方法详细的源码分析已经在前两篇文章中[Netty 源码分析系列之服务端 Channel 初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw) 和 [Netty 源码分析系列之服务端 Channel 注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ) 分析过了，今天将接着分析 **doBind(localAddress)** 后面的逻辑。doBind(localAddress)方法的源码如下

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    //初始化和注册
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }
    // 无论是进入if还是进入else，最终均会执行doBind0()
    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // Registration future is almost always fulfilled already, but just in case it's not.
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                // cause！= null 表示channel在初始化和注册过程中出席那了异常
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.registered();
                    // 没有异常，执行doBind0()
                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

当 initAndRegister() 执行完成后，会返回一个 **ChannelFuture**。ChannelFuture 实现了 Future 接口（关于 Future 的详细介绍可以参考这篇文章：[并发编程系列之 Future —— 最常用的性能优化手段](https://mp.weixin.qq.com/s/4gLjSGvwcuvElfnYRzDnSw)），调用 **ChannelFuture.isDone()** 能判断 initAndRegister()方法有没有执行完成（**因为 initAndRegister()方法中使用线程异步执行了其他逻辑，所以当 initAndRegister()方法返回时，它里面的逻辑不一定都已经执行完成了**）。

如果 initAndRegister() 方法已经完全执行完了，就进入到 if 逻辑块中，然后调用 **doBind0()** ；如果 initAndRegister() 没有执行完，那就进入 else 逻辑块中。在 else 中，通过添加一个监听器来监听 initAndRegister() 是否执行完，当执行完时，该监听器就能立马知道 initAndRegister() 已经执行完成了，然后还是调用 **doBind0()** 方法。

因此无论是进入 if 逻辑块，还是 else 逻辑块，最终都是调用 **doBind0()** 方法。doBind0()方法的源码如下。

```java
private static void  doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
    /**
     * 此时的channel就是NioServerSocketChannel
     * channel.eventLoop()获取到的就是服务端channel绑定的NioEventLoop线程
     */
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                // 调用NioServerSocketChannel的bind()方法
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

此时的 channel 是 NioServerSocketChannel 的实例，channel.eventLoop()获取到的是服务端 channel 绑定的 NioEventLoop 线程（在 initAndRegister()方法中绑定的）。因此 doBind0() 方法实际上就是通过 NioEventLoop 线程来异步执行 **channel.bind()** ，channel.bind() 方法的源码如下。

```java
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
}
```

channel.bind()实际上就是调用 channel 中的 pipeline 的 bind()方法。pipeline.bind()方法的源码如下。

```java
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return tail.bind(localAddress, promise);
}
```

pipeline.bind()方法中，调用的是 pipeline 中 tail 结点的 bind()方法。在 tail.bind()方法中，它没做其他逻辑处理，只是调用了 pipeline 中下一个节点的 bind()方法（这里是从 pipeline 的尾部开始向前找节点），因此最终会调用到 head 节点的 bind()方法。head 节点的 bind()方法源码如下。

```java
public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
    unsafe.bind(localAddress, promise);
}
```

对于服务端 channel 而言，这里的 unsafe 就是 **NioMessageUnsafe** 类型的实例（unsafe 在 NioServerSocketChannel 的构造方法中被实例化的）。NioMessageUnsafe 类的 bind()方法被定义在 AbstractChannel 中，精简后的源代码如下。

```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    // 省略部分代码...

    // 此时由于服务端channel还没哟绑定端口号，所以isActive()返回的是false
    boolean wasActive = isActive();
    try {
        // 真正绑定端口号的操作
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }
    // 经过doBind(localAddress)这一步，服务端channel已经绑定了端口号，所以isActive()会返回true
    // !wasActive && isActive() 运算的结果为true
    if (!wasActive && isActive()) {
        // 通过线程异步执行任务：在pipeline中传播执行handler的channelActive()方法
        invokeLater(new Runnable() {
            @Override
            public void run() {
                // 通过pipeline来传播执行handler的channelActive()方法
                pipeline.fireChannelActive();
            }
        });
    }

    safeSetSuccess(promise);
}
```

可以看到，在 bind()方法中，调用了 doBind(localAddress) 方法，看到 do 开头的方法名，终于可以高兴一下了，因为通常以 do 开头的方法，都是干实际事情的方法。实事的确如此，在 doBind()方法中，就做了端口号绑定的操作。NioServerSocketChannel 类的 doBind()方法的源码如下。

```java
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

其中 **javaChannel()** 获取的是 JDK 原生的 channel，即 ServerSocketChannel，然后调用 JDK 中 NIO 的原生 API：**bind(localAddress, backlog)** ，这一步执行完，Netty 中服务端的 channel 就和端口号实现了绑定，解决了文章开头中的第一个问题。然而第二个问题还没解决，所以继续往后看。

当 doBind()方法执行完后，回到 AbstractChannel.bind()方法中，此时由于服务端 channel 已经绑定了端口号，所以 **!wasActive && isActive()** 的计算结果为 true，因此会进入到 if 逻辑块中。在 if 逻辑块中，又异步执行了 **pipeline.fireChannelActive()** ，通过方法名就能猜出，这一步干的事情就是在 pipeline 中传播执行 handler 的 chanelActive()方法。 pipeline.fireChannelActive() 的源码如下。

```java
public final ChannelPipeline fireChannelActive() {
    // 从head节点开始传播
    AbstractChannelHandlerContext.invokeChannelActive(head);
    return this;
}
```

可以看到，会从 pipeline 的 head 节点开始传播执行，invokeChannelActive(head)最终会调用到 handler 的 channelActive(ChannelHandlerContext ctx)。下面看下 head 节点的 channelActive(ctx) 方法，由于 head 节点是 HeadContext 类型，所以代码找到 HeadContext.channelActive(ctx)，其源码如下。

```java
public void channelActive(ChannelHandlerContext ctx) {
    // 将channelActive时间传播给pipeline中下一个handler
    ctx.fireChannelActive();
    // 如果服务端channel的autoRead属性为1，就表示自动读数据，那么这个时候，就开始读数据
    // 默认情况下，autoRead属性为1，即开启自动读功能
    readIfIsAutoRead();
}
```

就两行代码，首先第一行代码，就是继续向 pipeline 中传播执行 channelActive(ctx)方法，这里就不再继续看了（实际上也没有必要看，因为此时服务端的 pipeline 中就四个节点，其他几个节点的 channelActive(ctx)方法中，没有做重要的逻辑处理，因此可以不用看）。

再看第二行代码，它调用了 **readIfIsAutoRead()** 方法，方法名翻译过来就是：如果是自动读状态，就开始读操作，默认情况下，自动读的开关是开启的，即默认开始读。在这儿可能有点懵逼，究竟要读什么东西呢？所以继续跟进 **readIfIsAutoRead()** 方法的源码。

```java
private void readIfIsAutoRead() {
    // 默认自动读
    if (channel.config().isAutoRead()) {
        // 读数据
        channel.read();
    }
}
```

由于 autoRead 默认等于 1，所以 **channel.config().isAutoRead()** 会返回 true，因此进入 if 逻辑块，调用 channel.read()方法。此时是服务端 channel，因此调用的是 NioServerSocketChannel 的 read()方法。该方法定义在 AbstractChannel 中，源码如下。

```java
public Channel read() {
    // 从pipeline中开始传播，一次调用pipeline中的handler的read()方法
    pipeline.read();
    return this;
}
```

卧槽，又看到了 pipeline，前车之鉴，我们可以猜到，这个 read()方法，又会沿着 pipeline 去传播执行。接着看 pipeline 的 read()方法，源码如下。

```java
public final ChannelPipeline read() {
    tail.read();
    return this;
}
```

发现它会调用 tail 节点的 read()方法，然后沿着 tail 节点，向前传播执行 read()，最终会执行到 head 节点的 read()方法中，由于本人看过源代码，知道 pipeline 中除了 head 节点外，其他节点的 read() 方法没做重要逻辑处理，所以直接跳到 head 节点的 read()方法中。其源码如下。

```java
public void read(ChannelHandlerContext ctx) {
    // 对于服务端的channel而言，unsafe的值为NioMessageUnsafe类型的实例
    // 对于客户端channel而言，unsafe的值为NioSocketChannelUnsafe类型的实例
    unsafe.beginRead();
}
```

可以看见，在 head 节点的 read()方法中，会调用 **unsafe 属性的 beginRead()** 方法。对于服务端的 channel 而言，unsafe 的值为 **NioMessageUnsafe** 类型的实例；对于客户端 channel 而言，unsafe 的值为 NioSocketChannelUnsafe 类型的实例。由于这里是服务端 channel，所以会调用 NioMessageUnsafe 的 beginRead() 方法。该方法定义在 AbstractUnsafe 类中，其源码如下。

```java
public final void beginRead() {
    assertEventLoop();

    if (!isActive()) {
        return;
    }

    try {
        // 真正开始读数据
        doBeginRead();
    } catch (final Exception e) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireExceptionCaught(e);
            }
        });
        close(voidPromise());
    }
}
```

在代码中又可以看见以 **do** 开头的方法：**doBeginRead()** ，终于可以高兴一下了，总算见到核心代码了，doBeginRead()方法的源码如下。

```java
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;
    /**
     * 在服务端channel注册到多路复用器上时，将selectionKey的interestOps属性设置为了0
     * selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
     */
    final int interestOps = selectionKey.interestOps();
    /**
     * readInterestOp属性的值，是在NioServerSocketChannel的构造器中，被设置为SelectionKey.OP_ACCEPT，即16
     * public NioServerSocketChannel(ServerSocketChannel channel) {
     *    super(null, channel, SelectionKey.OP_ACCEPT);
     *    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
     * }
     */
    if ((interestOps & readInterestOp) == 0) {
        // 对于服务端channel而言，interestOps | readInterestOp运算的结果为16，即SelectionKey.OP_ACCEPT
        // 所以最终selectionKey感兴趣的事件为OP_ACCEPT事件，至此，服务端channel终于可以开始接收客户端的链接了。
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

先获取到了 selectionKey 的 **interestOps** 属性的值，该属性的值为 **0** 。为什么呢？因为在将服务端 channel 注册到多路复用器上时，传入的值就是 **0** 。如下图所示。


![register()](https://user-gold-cdn.xitu.io/2019/12/15/16f09760da927e1e?w=1656&h=90&f=png&s=96911)

然后获取到了 **readInterestOp** 的值，该属性的值为 **16**，即 **SelectionKey.OP_ACCEPT** 。为什么呢？因为在 NioServerSocketChannel 的构造方法中，给它赋的值就是 **SelectionKey.OP_ACCEPT**。如下图所示。


![构造方法](https://user-gold-cdn.xitu.io/2019/12/15/16f097649bd74ea7?w=1432&h=280&f=png&s=91163)

最后，进行或运算： **interestOps | readInterestOp** 得到结果是 **16**，即 **OP_ACCEPT**，然后将结果设置给 **selectionKey**，这样服务端 channel 感兴趣的事件就是 **OP_ACCEPT** 事件，这样当有新的客户端来连接时，服务端 channel 就可以通过 selectionKey 感知到了，这就回答了文章开头的第二个问题。

到这里，Netty 服务端的启动流程的源码终于分析完成了。感叹一下，**太 TM 复杂了！！！**，虽然我们使用 Netty 的时候，代码特别简单，但是它的底层逻辑却是相当复杂，真的是**哪有什么岁月静好，不过是有人替你负重前行**。

### 总结

- 本文详细分析了 Netty 中如何将服务端 channel 和端口号绑定的过程，以及 Netty 是如何将服务端 channel 感兴趣的事件设置为 **OP_ACCEPT** 的。
- 结合前两篇文章[Netty 源码分析系列之服务端 Channel 初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw) 和 [Netty 源码分析系列之服务端 Channel 注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)，至此，Netty 服务端的启动流程就分析完了，后面的文章将会分析新连接接入的源码以及编解码等过程的源码。
- 原创不易，看到这里，如果觉得本文对你有帮助，可以点击一下文章中的广告，感激不尽。

### 推荐
* [如何从BIO演进到NIO，再到Netty](https://mp.weixin.qq.com/s/zcrclRhgK015FSjtuMZoCg)
* [Netty源码分析系列之Reactor线程模型](https://mp.weixin.qq.com/s/-aqIwWXYBkTy0dCNy0FtlA)
* [Netty源码分析系列之服务端Channel初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw)
* [Netty源码分析系列之服务端Channel注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)

![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144)
