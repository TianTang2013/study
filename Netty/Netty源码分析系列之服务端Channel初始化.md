### 问题
老规矩，Netty的源码很难、很复杂，为了更快的学懂新的知识，所以还是带着问题来学习源码。
Netty作为一款基于事件驱动的高性能网络框架，其底层实际上仍然使用的是JDK里面的NIO，Netty在JDK的NIO上做了大量优化，以及封装，降低了开发人员使用NIO的难度。

使用JDK原生的NIO进行网络编程时，首先得做两件事：1. 创建以及初始化ServerSocketChannel，2. 将ServerSocketChannel绑定到多路复用器Selector上。既然说Netty对JDK的NIO做了封装，那么在Netty中是什么时候进行这两步操作的呢？（在Netty中服务端的Channel是NioServerSocketChannel）

答案显然就是：在Netty服务端启动的时候进行的。本文接下来就结合源码来看看Netty是如何进行服务端Channel的初始化，关于绑定到Selector的源码分析会发布在下一篇文章中。

### 组件说明
在看Netty服务端启动流程的源码之前，先来简单介绍下Netty中相关的组件。这些组件很重要，后面会单独针对每一个组件写文章进行详细说明，今天只是先简单介绍一下。

首先是NioEventLoopGroup，看英文名，翻译过来就是一个NIO事件轮询组。可以简单理解为它是一个线程组，它里面包含了一组线程，这些线程后续用来执行客户端的接入、IO数据读写等任务。

NioEventLoop，可以简单理解为它是一个线程，多个NioEventLoop合起来就是一个NioEventLoopGroup。

Pipeline，它是由ChannelHandlerContext类型的元素组成的一个双向链表，ChannelHandlerContext里面包装了一个ChannelHandler。对于服务端而言，每当有新连接接入时，都会通过Pipeline来传播。这个组件十分重要，在实际工作中，使用Netty实现自己自定义的业务逻辑，就是通过修改Pipeline来实现的。

### 服务端启动
先来看一段服务端启动的demo代码。
【图】

在示例代码中，①~⑤都是为Netty服务端设置一些属性，比较简单。需要额外说明的是第②步。在第②步中，对服务端而言，需要设置channel的类型为NioServerSocketChannel。当调用channle(NioServerSocketChannel.class)方法时，会调用到AbstractBootstrap类的channel()方法，它干的事情就是创建一个一个ReflectiveChannelFactory工厂，并将ReflectiveChannelFactory实例赋值给AbstractBootstrap类的channelFactory属性。

对于ReflectiveChannelFactory而言，它有一个Constructor类型的属性，叫做constructor，它就是后续用来创建Channel的反射构造器。对于此处而言，constructor就是NioServerSocketChannel.class的反射构造器，通过它就能创建出一个NioServerSocketChannel的实例对象。
channel()方法的源码
【图】

ReflectiveChannelFactory构造器的源码
【图】

在第⑥步中，会调用ServerBootstrap的bind()方法，这个方法中传入了一个端口号：8080。从外面看，这个方法及其简单，但是它干的事情非常多。在这个方法中会初始化服务端的channel，注册channel到selector上，然后绑定端口，启动服务端。下面来通过分析它的源码，来看下服务端Channel是如何进行初始化和绑定操作的。

当调用ServerBootstrap.bind(8080)时，会调用到ServerBootstrap.doBind()，下面我简化了一下doBind()方法的源码，只保留了核心代码，如下。
【图】

在doBind()方法中调用了两个非常重要的方法，一个是initAndRegister()方法，它用来进行Channel的初始化和注册到Selector上的操作；另外一个是doBind0()方法，它用来进行端口号的绑定操作。今天只分析initAndRegister()方法的前半部分。

### channel初始化
initAndRegister()的源码如下。
【图】

在initAndRegister()方法中，会先通过调用channelFactory.newChannel()来创建一个Channle，对于服务端而言，这里创建的就是NioServerSocketChannel。那么它是如何创建的呢？

首先这里的channelFactory就是前面在②处提到的ReflectiveChannelFactory，它里面包含一个属性constructor。这个constructor就是`NioServerSocketChannel.class`的反射构造器，通过调用`constructor.newInstance()`方法，就会调用到NioServerSocketChannel类的无参构造方法。NioServerSocketChannel的无参构造方法如下。
```java
public NioServerSocketChannel() {
    // DEFAULT_SELECTOR_PROVIDER的值就是SelectorProvider.provider()
    // 先调用newSocket()
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```

在无参构造方法中会先调用newSocket()方法，该方法就是通过JDK原生的API创建一个ServerSocketChannel。
```java
private static ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
        // provider的值就是SelectorProvider.provider();
        // 这里就是直接调用JDK中NIO的原生API，创建ServerSocketChannel
        return provider.openServerSocketChannel();
    } catch (IOException e) {
        throw new ChannelException(
                "Failed to open a server socket.", e);
    }
}
```

当调用完newSocket()方法后，会返回一个ServerSocketChannel，然后接着会调用this(ServerSocketChannel)，即NioServerSocketChannel类的一个有参构造器。
```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    // 调用父类的构造器，SelectionKey.OP_ACCEPT的值为1，表示的是服务的channel感兴趣的事件是接收事件
    super(null, channel, SelectionKey.OP_ACCEPT);
    // config是用来保存服务端channel相关的配置
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

接着会一直往上调用父类的构造器，最终会调用到AbstractNioChannel类的构造器中。
```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    // 继续调用父类
    super(parent);
    this.ch = ch;
    // 保存感兴趣的事件，前面传过来的值是OP_ACCEPT，表示感兴趣的时间是接收事件，
    // 这里只是保存，并不是意味着此时就是要开始接收新连接了
    this.readInterestOp = readInterestOp;
    try {
        // 设置为非阻塞模式，这个JDK的原生的NIO是一致的
        ch.configureBlocking(false);
    } catch (IOException e) {
        // ......
    }
}
```

在这里会先调用父类的构造器，再保存channle、readInterestOp以及设置channel为非阻塞模式。注意这里只是保存了readInterestOp的值，并不是意味着此时就是要开始接收新连接了，因为此时服务端还没有绑定端口。再继续往上看父类的构造器干了哪些事情。
最终会调用到AbstractChannel的构造器。
```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```
在AbstractChannel的构造器主要干了三件事。
第一：给channel的id赋值，这个id就是一个唯一标识符，用处不大。
第二：通过newUnsafe()创建了一个NioMessageUnsafe对象，这个对象是服务端后面负责用来接收连接等操作的。看到Unsafe是不是立马能想到JDK中的Unsafe类，它的作用类似，可以直接操作内存，功能很强大。
第三：通过newChannelPipeline()创建了一个Pipeline，后面所有新连接的接入，都需要经过该Pipeline进行传播。Pipeline的初始化如下，会初始化一个头结点和尾结点。
```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```
此时Pipeline的结构如下。
【图】

此时，NioServerSocketChannel的构造方法终于执行完了，回到initAndRegister()方法中，此时channel对象已经有值了，接下来就是调用init()方法进行channel的初始化操作了。当调用init()方法时，会调用ServerBootstrap的init()方法。在init()方法中，先是获取到了前面②等步骤中设置的一些和服务端Channel相关的option、attrs（本文demo中没设置）等配置，以及和客户端channel相关的childOptions、childAttrs等配置，代码比较简单，这里就不一一展开了。

init()方法最核心的逻辑在最后一行：p.addLast()。前面在创建NioServerSocketChannel时，对channel中的pipeline进行的初始化操作，这里当调用p.addLast()时，会向pipeline中在添加一个ChannelHandler。下面是init()方法简化后的代码。
```java
void init(Channel channel) throws Exception {
    // ......
    ChannelPipeline p = channel.pipeline();
    // 添加了一个匿名类   
    p.addLast(new ChannelInitializer<Channel>() {
        // 后面会回调到initChannel()方法
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            // config.handler()获取到handler就是我们在步骤④中指定的NettyServerHandler类
            ChannelHandler handler = config.handler();
            if (handler != null) {
                // 添加到pipeline中
                pipeline.addLast(handler);
            }
            // 添加成功后，向NioEventLoop中添加一个任务，最终就执行到里面的run()方法
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    // 在run()方法中又向pipeline中添加了一个ChannelHandler
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```
此时向pipeline中添加的是一个匿名类：`new ChannelInitializer<Channel>(){}`，注意此时并不会执行这个匿名类中的initChannel()方法。添加完成后，pipeline的结构变成了如下图所示的结构。
【图】

当后面channel注册到Selector上后，会回调到ChannelInitializer的initChannel(ChannelHandlerContext ctx)方法上（注意这里initChannel()方法是一个重载方法）。源码如下。
```java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.add(ctx)) { // Guard against re-entrance.
        try {
            // 回调匿名类的initChannel()方法
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
            // We do so to prevent multiple calls to initChannel(...).
            exceptionCaught(ctx, cause);
        } finally {
            // 最后将pipeline中的这个匿名类删除
            ChannelPipeline pipeline = ctx.pipeline();
            if (pipeline.context(this) != null) {
                pipeline.remove(this);
            }
        }
        return true;
    }
    return false;
}
```
在initChannel(ChannelHandlerContext ctx方法上，就会回调到上面pipeline中匿名类的initChannel(final Channel ch)方法。在匿名类的initChannel()方法的逻辑中，会先向pipeline中添加一个我们在步骤④中指定的NettyServerHandler类，添加完后，pipeline就会变成如下结构。
【图】

当添加完NettyServerHandler后，接着通过`ch.eventLoop().execute(new Runnable(){})`向NioEventLoop中添加了一个任务，这个任务此时不会立刻执行，而是会在另外的线程中异步执行。当匿名类的initChannel(final Channel ch)方法执行完成后，就会回到initChannel(ChannelHandlerContext ctx)方法中，最后会执行finally语句块。在finally中执行了一段代码。
```java
ChannelPipeline pipeline = ctx.pipeline();
if (pipeline.context(this) != null) {
    pipeline.remove(this);
}
```
这段代码的作用，就是将前面创建的那个匿名类，从pipeline中移除。移除完以后pipeline的结构就变为了如下图所示。为什么要删除呢？因为这个匿名类的只是用来在启动阶段向pipeline中添加元素的，当服务器启动以后，pipeline的结构已经固定了，这个匿名类就没有存在的意义了，因此会把它删除。
【图】

由于前面向NioEventLoop中添加了一个任务，当这个任务被执行时，又会向pipeline中添加一个元素：ServerBootstrapAcceptor。这个类很重要，它就是后面用来负责所有新连接的接入的。因为后面所有的新连接都会先经过服务端channel的pipeline，而ServerBootstrapAcceptor又是pipeline中一个节点，所以后面所有的新连接都要经过ServerBootstrapAcceptor的处理。最终pipeline的结构如下图所示。（重要的事情说三遍：​这个图很重要，牢记！牢记！牢记！​）

【图】


### 总结
* 本文主要结合源码分析了服务端的NioServerSocketChannel的初始化过程，但没有分析NioServerSocketChannel如何注册到Selector上，以及服务端端口绑定的流程也没有分析，这两部分的内容会在后面两篇文章中详细分析。
* 在分析NioServerSocketChannel初始化的过程中，着重分析了pipeline的变化过程以及最终的结构，这对于后面分析新连接的接入过程会有很大的帮助。
* 最后，Netty源码很复杂，它的启动流程非常重要，代码也很复杂。类的继承关系十分复杂，所以建议读者亲自动手Debug调试，这对学习Netty会有很大帮助。
