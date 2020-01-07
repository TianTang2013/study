### 1.问题

当 netty 的服务端启动以后，就可以开始接收客户端的连接了。那么在 netty 中，服务端是如何来进行新连接的创建的呢？在开始进行源码阅读之前，可以先思考以下三个问题。

- 服务端是如何检测到有新的客户端请求接入的（后面简称新连接接入）？
- 在 JDK 原生的 NIO 中，服务端会通过**ServerSocketChannel.accept()** 来为新接入的客户端创建对应的客户端 channel，那么在 netty 中服务端又是如何来处理新连接的接入的呢？
- 在 netty 中网络 IO 的读写操作都是在 NioEventLoop 线程中进行的，那么客户端 channel 是如何和工作线程池中的 NioEventLoop 绑定的呢？

### 2.检测新连接接入

在上一篇文章[Netty 源码分析系列之 NioEventLoop 的执行流程](https://mp.weixin.qq.com/s/RId4bKC0lwL_HofyqDV61Q)中，分析了 NioEventLoop 线程在启动后，会不停地去循环处理网络 IO 事件、普通任务和定时任务。在处理网络 IO 事件时，当轮询到 IO 事件类型为 OP_ACCEPT 时(如下代码所示)，就表示有新客户端来连接服务端了，也就是检测到了新连接。这个时候，服务端 channel 就会进行新连接的读取。

```java
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
}
```

可以看到，当是 OP_ACCEPT 事件时，就会调用**unsafe.read()** 方法来进行新连接的接入。此时 unsafe 对象是 NioMessageUnsafe 类型的实例，为什么呢？因为只有服务端 channel 才会对 OP_ACCEPT 事件感兴趣，而服务端 channel 中 unsafe 属性保存的是 NioMessageUnsafe 类型的实例。

read()方法的源码很长，但它主要干了两件事，第一：调用 doReadMessages()方法来读取连接；第二：将读取到的连接通过服务端 channel 中的 pipeline 来进行传播，最终执行每一个 handler 中的 channelRead()方法。

### 3.创建客户端 channel

服务端 channel 在监听到 OP_ACCEPT 事件后，会为新连接创建一个客户端 channel，后面数据的读写均是通过这个客户端 channel 来进行的。而这个客户端 channel 是通过 doReadMessages()方法来创建的，该方法是定义在 NioServerSocketChannel 中的，下面是其源码。

```java
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());
    try {
        if (ch != null) {
            // 将原生的客户端channel包装成netty中的客户端channel：NioSocketChannel
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        // 异常日志打印等...
    }
    return 0;
}
```

在该方法中，首先会通过 javaChannel()获取到 JDK 原生的服务端 channel，即 ServerSocketChannel，这个原生的服务端 channel 是被保存在 NioServerSocketChannel 的**ch**属性中，在初始化 NioServerSocketChannel 时会对**ch**属性赋值（可以参考这篇文章：[Netty 源码分析系列之服务端 Channel 初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw)）。创建完 JDK 原生的服务端 channel 后，会通过 SocketUtils 这个工具类来创建一个 JDK 原生的客户端 channel，即 SocketChannel。SocketUtils 这个工具类的底层实现，实际上就是调用 JDK 原生的 API，即 ServerSocketChannel.accept()。

在创建完原生的 SocketChannel 后，netty 需要将其包装成 netty 中定义的服务端 channel 类型，即：NioSocketChannel。如何包装的呢？通过 new 关键字调用 NioSocketChannel 的构造方法来进行包装。在构造方法中，做了很多初始化工作。跟踪源码，发现会调用到 AbstractNioChannel 类的如下构造方法。

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    // 此时的parent = NioServerSocketChannel，ch = SocketChannel(JDK原生的客户端channel)，readInterestOp = OP_READ
    super(parent);
    // 保存channel和感兴趣的事件
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        // 设置为非阻塞
        ch.configureBlocking(false);
    } catch (IOException e) {
        // 异常处理...
    }
}
```

在该构造方法中，首先会保存原生的客户端 channel 和客户端 channel 感兴趣的事件，然后将客户端 channel 的阻塞模式设置为 false，表示不阻塞（在 NIO 网络编程中，这一步是必须的，否则启动会报错）。同时还会调用父类构造方法，父类就是 AbstractChannel。AbstractChannel 类的构造方法源码如下。

```java
protected AbstractChannel(Channel parent) {
    // parent的值为NioServerSocketChannel
    this.parent = parent;
    id = newId();
    // 对于客户端channel而言，创建的unsafe是NioSocketChannelUnsafe
    unsafe = newUnsafe();
    // DefaultChannelPipeline
    pipeline = newChannelPipeline();
}
```

在该构造方法中，对于客户端 channel 而言，parent 的值为 NioServerSocketChannel，也就是 netty 服务端启动时创建的服务端 channel。然后创建的 unsafe 是 NioSocketChannelUnsafe，最后会为客户端 channel 创建一个默认的 pipeline，此时 pipeline 的结构如下。（如果看过前几篇文章，可能会发现，服务端 channel 在创建时也会调用到该构造方法）

【图】

最终还会为 NioSocketChannel 创建一个 NioSocketChannelConfig 对象，这个对象是用来保存用户为客户端 channel 设置的一些 TCP 配置和属性，在创建这个 config 对象时，会将 TCP 的 TCP_NODELAY 参数设置为 true。TCP 在默认情况下，会将小的数据包积攒成大的数据包以后才发出去，而 netty 为了**及时**地 i 将较小的数据报发送出去，因此将 TCP_NODELAY 参数设置为 true，表示不延迟发送。

至此，新连接对应的客户端 channel 就创建完成了，后面网络数据的读写，都是基于这个 NioSocketChannel 来进行的。

### 4.绑定 NioEventLoop

当客户端的 channel 创建完成后，在 read()方法中，就会通过 pipeline.fireChannelRead(socketChannel)这一行代码，将客户端 channel 通过 pipeline 进行传播，依次执行 pipeline 中每一个 handler 的 channelRead()方法。（注意，这儿的 pipeline 是服务端 channel 中保存的 pipeline，在创建客户端 channel 时，也会为每个新建的客户端 channel 创建一个 pipeline，这里千万不要搞混了）

在服务端启动的时候，服务端 channel 中 pipeline 的结构图如下（详细解释可以参考这三篇文章： [Netty 源码分析系列之服务端 Channel 初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw) 、[Netty 源码分析系列之服务端 Channel 注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)和 [Netty 源码分析系列之服务端 Channel 的端口绑定](https://mp.weixin.qq.com/s/p3ZouaQ_x6NgyVMWw4tJ0g)）。

【图】

该 pipeline 中，对于 head 和 tail 而言，它俩的 channelRead()方法没做什么实际意义的工作，直接是向下一个节点传播了，这里重要的是 ServerBootstrapAcceptor 节点的 channelRead()方法。该方法的源码如下。

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    // 向客户端的channel中添加用户自定义的childHandler
    child.pipeline().addLast(childHandler);

    // 保存用户为客户端channel配置的属性
    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        // 将客户端channel注册到工作线程池，即从workerGroup中选择出一个NioEventloop，再将客户端channel绑定到NioEventLoop上
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

在该方法中，首先向客户端 channel 的 pipeline 中的节点中添加了一个 childHandler，这个 childHandler 是用户自己定义的，什么意思呢？如下图所示，用户通过 childHandler()方法自定义了一个 ChannelInitializer 类型的 childHandler，这个此时就会向客户端 channel 的 pipeline 中的节点中添加该 childHandler（这个地方很重要，后面会用到）。然后通过 setChannelOptions 保存用户为客户端 channel 配置的 TCP 参数和属性。

【图】

最重要的一步在 childGroup.register(child)，这一行代码会将客户端 channel 注册到 workerGroup 线程池中的某一个 NioEventLoop 上。（在服务端端口绑定的过程中，也是类似于调用 NioEventLoopGroup 的 register()方法，将服务端 channel 注册到 bossGroup 中的某一个 NioEventLoop 中）。

此时的 childGroup 是 workerGroup（Reactor 主从多线程线程模型中的从线程池），调用 register()方法时，会调用到如下方法。

```java
public ChannelFuture register(Channel channel) {
    // next()方法会从NioEventLoop中选择出一个NioEventLoop
    return next().register(channel);
}
```

next()方法会从 NioEventLoop 中选择出一个 NioEventLoop（关于 next()方法的详细介绍请参考： [Netty 源码分析系列之 NioEventLoop 的创建与启动](https://mp.weixin.qq.com/s/qBRY8zl_LBrn3acCaRmEbg)），由于 NioEventLoop 继承了 SingleThreadEventLoop，所以这儿最后调用的是 SingleThreadEventLoop 中的如下的 register()方法。

```java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    /**
     *  对于客户端channel而言
     *  promise是DefaultChannelPromise
     *  promise.channel()获取到的是NioSocketChannel
     *  promise.channel().unsafe()得到的是NioSocketChannelUnsafe
     *  由于NioSocketChannelUnsafe继承了AbstractUnsafe，所以当调用unsafe.register()时，会调用到AbstractUnsafe类的register()方法
     */
    // this为NioEventLoop
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

> 这里的 unsafe()获取到的是 NioSocketChannelUnsafe 对象，由于 NioSocketChannelUnsafe 继承了 AbstractUnsafe，所以当调用 unsafe.register()时，会调用到 AbstractUnsafe 类的 register()方法。该方法精简后的源码如下。

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // 省略部分代码....

    // 对客户端channel而言，这一步是给NioSocketChannel的eventLoop属性赋值
    AbstractChannel.this.eventLoop = eventLoop;

    // 判断是同步执行register0()，还是异步执行register0()
    if (eventLoop.inEventLoop()) {
        // 同步执行
        register0(promise);
    } else {
        try {
            // 提交到NioEventLoop线程中，异步执行
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            // 省略部分代码
        }
    }
}
```

实际上，服务端 channel 注册到 NioEventLoop 上时，也是调用的到了该方法（可以参考这篇文章： [Netty 源码分析系列之服务端 Channel 注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)）。

对于客户端而言，在该方法中，通过如下一行代码，就将客户端 channel 与一个 NioEventLoop 进行了绑定，这就回答了文章开头的第三个问题。

```java
AbstractChannel.this.eventLoop = eventLoop;
```

接着会判断当前线程是否等于传入的 eventLoop 中保存的线程，这里肯定不是。为什么呢？因为当前线程是 bossGroup 线程组中的线程，而 eventLoop 是 workerGroup 线程组中的线程，所以这里会返回 false，那么就会异步执行 register0()方法。register0()方法的源码如下。

```java
private void register0(ChannelPromise promise) {
    try {
        // 省略部分代码...
        boolean firstRegistration = neverRegistered;
        /**
         * 对于客户端的channel而言，doRegister()方法做的事情就是将服务端Channel注册到多路复用器上
         */
        doRegister();
        neverRegistered = false;
        registered = true;

        //会执行handlerAdded方法
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        //通过在pipeline传播来执行每个ChannelHandler的channelRegistered()方法
        pipeline.fireChannelRegistered();

           // 如果客户端channel已经激活，就执行下面逻辑。
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        // 省略部分代码...
    }
}
```

在 register0()方法中，有三步重要的逻辑，第一：doRegister()；第二：pipeline.invokeHandlerAddedIfNeeded()；第三：pipeline.fireChannelRegistered()。下面分别来看看这三步都干了哪些事情。

doRegister()就是真正将客户端 channel 注册到多路复用器上的一步。doRegister()调用的是 AbstractNioChannel 类中的 doRegister()方法，删减后源码如下。

```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            // 异常处理......
        }
    }
}
```

其中 javaChannel()获取的就是 JDK 中原生的 SocketChannel。

`eventLoop().unwrappedSelector()`获取的是 JDK 中原生的多路复用器 Selector（底层的数据结构被替换了）。（EventLoop 中的 unwrappedSelector 属性是在创建 NioEventLoop 时，初始化的，底层的数据结构也是这个时候被替换的）

所以`javaChannel().register(eventLoop().unwrappedSelector(), 0, this)`这一行代码，实际上就是调用 JDK 原生 SocketChannel 的`register(selector,ops,attr)`方法，然后将客户端 Channel 注册到了多路复用器 Selector 上。

注意这里在调 JDK 原生的 register()方法时，第三个参数传入的是 this，此时 this 代表的就是当前的 NioSocketChannel 对象。将 this 作为一个 attachment 保存到多路复用器 Selector 上，这样做的好处就是，后面可以通过多路复用器 Selector 获取到客户端的 channel。第二个参数传入的是 0，表示此时将客户端 channel 注册到多路复用器上，客户端 chennel 感兴趣的事件标识符是 0，即此时对任何事件都不感兴趣（在后面才会将感兴趣的事件设置为 OP_READ）。

当 doRegister()方法执行完以后，就会执行第二步：pipeline.invokeHandlerAddedIfNeeded()，这一步做的事情就是回调 pipeline 中 handler 的 handlerAdded()方法。

往下执行，代码会执行到 pipeline.fireChannelRegistered()，也就是前面我们提到的第三步。这一步做的事情就是传播 Channel 注册事件，如何传播呢？就是沿着 pipeline 中的头结点这个 handler 开始，往后依次执行每个 handler 的 channelRegistered()方法。

在前面我们提到过，会向客户端 channel 的 pipeline 中添加一个 ChannelInitializer 类型的匿名类，因此在传播执行 channelRegistered()方法的时候，就会执行到该匿名类的 channelRegistered()方法，从而最终会执行该匿名类中重写的 initChannel(channel)方法，即如下图所示的代码。关于是如何调用到 initChannel(channel)方法中的，可以参考这篇文章：[Netty 源码分析系列之服务端 Channel 注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)，里面进行了很详细的分析。不过读源码最佳方式还是亲自动手，Debug 调试一下你也许会体会更深，更容易理解。

【图】

再次回到 register0()方法中，最后会判断 isActive()是否为 true，此时由于客户端 channel 已经注册到多路复用器上了，因此会返回 true，而且由于此时客户端 channel 是第一次注册，所以会 pipeline.fireChannelActive()这一行代码，也就是又会通过客户端 channel 的 pipeline 向下传播执行所有 handler 的 channelActive()方法，最终会调用到 AbstractChannel 的 doBeginRead()方法(这一步的调用过程很复杂，建议直接 DEBUG)。doBeginRead 方法的源码如下。

```java
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;
    /**
     * 在客户端channel注册到多路复用器上时，将selectionKey的interestOps属性设置为了0
     * selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
     */
    final int interestOps = selectionKey.interestOps();
    /**
     * readInterestOp属性的值，是在NioSocketChannel的构造器中，被设置为SelectionKey.OP_READ
     */
    if ((interestOps & readInterestOp) == 0) {
        // 对于客户端channel而言，interestOps | readInterestOp运算的结果为OP_READ
        // 所以最终selectionKey感兴趣的事件为OP_READ事件，至此，客户端channel终于可以开始接收客户端的链接了。
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

至此，客户端 channel 感兴趣的就变成了 OP_READ 事件，那么接下来就可以进行数据的读写了。

### 5.总结

本文主要分析了当一个新连接进来后，netty 服务端是如何为这个新连接创建客户端 channel 的，又是如何将其绑定到 NioEventLoop 线程中的。客户端 channel 注册过程与服务端 channel 的注册过程非常相似，调用过程几乎一样，所以建议先阅读这篇文章[Netty 源码分析系列之服务端 Channel 注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)。
