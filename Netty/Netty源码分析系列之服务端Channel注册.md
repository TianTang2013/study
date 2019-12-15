> 扫描下方二维码或者微信搜索公众号`菜鸟飞呀飞`，即可关注微信公众号，阅读更多`Spring源码分析`和`Java并发编程`文章。

【图】

### 问题
1. 在JDK的原生NIO的写法中，通过`serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT)`将服务端channel注册到多路复用器selector上，那么在Netty中，又是如何将NioServerSocketChannel注册到多路复用器上的呢？在注册过程中，Netty又额外做了哪些事情呢？
2. 在上一篇文章[Netty源码分析系列之服务端Channel初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw)中，分析了在init(channel)方法中，向pipeline添加了一个匿名类：ChannelInitializer，在该匿名类的initChannel(channel)方法中，执行了很重要的逻辑：向pipeline中添加了两个handler。但是上一篇文章是直接说了initChannel(channel)的执行结果，没有说代码是如何回调到这个匿名类的initChannel(channel)方法上的，本文接下来将详细说明这一点。

### 上篇回顾
当调用`ServerBootstrap.bind(port)`时，代码会执行到`AbstractBootstrap.doBind(localAddress)`方法，在doBind()方法中又会调用`initAndRegister()`方法。initAndRegister()方法简化后的代码如下。
```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        /**
         * newChannel()会通过反射创建一个channel，反射最终对调用到Channel的构造器
         * 在channel的构造器中进行了Channel很多属性的初始化操作
         * 对于服务端而言，调用的是NioServerSocketChannel的无参构造器。
         */
        channel = channelFactory.newChannel();
        /**
         * 初始化
         * 调用init方法后，会想服务端channel的pipeline中添加一个匿名类，这个匿名类是ChannelInitializer
         * 这个匿名类非常重要，在后面channel的register过程中，会回调到该匿名类的initChannel(channel)方法
         */
        init(channel);
    } catch (Throwable t) {
        // 省略部分代码...
    }
    ChannelFuture regFuture = config().group().register(channel);
    // 省略部分代码...
    return regFuture;
}
```

`initAndRegister()`方法的作用是初始化服务端channel，即NioServerSocketChannel，并将服务端channel注册到多路复用器上。在上一篇文章[Netty源码分析系列之服务端Channel初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw)中分析了initAndRegister()方法的前半部分，即服务端channel初始化的过程。在服务端channel初始化的过程中通过`channelFactory.newChannel()`会反射调用到NioServerSocketChannel的无参构造器，最终会创建一个NioServerSocketChannel实例，在构造方法中会对NioServerSocketChannel的很多属性进行初始化，例如：pipeline、unsafe。接着调用`init(channel)`方法，会为NioServerSocketChannel设置`options、attr`等属性，最重要的是向NioServerSocketChannel的pipeline中添加了一个ChannelInitinalizer类型的匿名类。

当执行完init(channel)方法后，代码接着就会执行到`ChannelFuture regFuture = config().group().register(channel);`。这一行代码就是本文今天分析的重点，它的主要功能就是将服务端Channel注册到多路复用器上。

### register(channel)源码
```java
ChannelFuture regFuture = config().group().register(channel);
```

在这一行代码中，`config()`获取到的是ServerBootstrapConfig对象，这个对象保存了我们为Netty服务端配置的一些属性，例如设置的bossGroup、workerGroup、option、handler、childHandler等属性均被保存在ServerBootstrapConfig这个对象中。（bossGroup、workerGroup表示的是Reactor的主从线程池，也就是我们通过如下代码创建的NioEventLoopGroup）

```java
// 负责处理连接的线程组
NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
// 负责处理IO和业务逻辑的线程组
NioEventLoopGroup workerGroup = new NioEventLoopGroup(8);
```

`config().group()`获取到的是我们为服务端设置的bossGroup。因此`config().group().register(channel)`实际上调用的是NioEventLoopGroup的register(channel)方法。由于NioEventLoopGroup继承了MultithreadEventLoopGroup，register(channel)定义在MultithreadEventLoopGroup类中，所以会调用`MultithreadEventLoopGroup.register(channel)`。（Netty中类的继承关系十分复杂，所以在看源码过程中，最好使用IDEA的debug方式去看源码，否则有时候都不知道某个方法的具体实现究竟是在哪个类中）。

MultithreadEventLoopGroup.register(channel)方法的源码如下。
```java
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```

NioEventLoopGroup是一个线程组，它包含了一组NioEventLoop。`next()`方法就是从NioEventLoopGroup这个线程组中通过轮询的方法取出一个NioEventLoop（NioEventLoop可以简单理解为它是一个线程），然后通过NioEventLoop来执行register(channel)。

当调用`NioEventLoop.register(channel)`时，实际上调用的是SingleThreadEventLoop类的register(channel)。SingleThreadEventLoop.register(channel)源码如下。

```java
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```

此时的channel为NioServerSocketChannel，this为NioEventLoop。先创建了一个ChannelPromise对象，然后将channel和NioEventLoop保存到这个ChannelPromise对象中，这是为了后面方便从ChannelPromise中获取到channel和NioEventLoop。

接着调用的是SingleThreadEventLoop中的另一个register(channelPromise)重载方法。源码如下。

```java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    /**
     *  对于服务端而言
     *  promise是DefaultChannelPromise
     *  promise.channel()获取到的是NioServerSocketChannel
     *  promise.channel().unsafe()得到的是NioMessageUnsafe
     *  由于NioMessageUnsafe继承了AbstractUnsafe，所以当调用unsafe.register()时，会调用到AbstractUnsafe类的register()方法
     */
    // this为NioEventLoop
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

在register(final ChannelPromise promise)中，`promise.channel()`获取到的就是NioServerSocketChannel（前面已经提到过，在创建ChannelPromise时，就将NioServerSocketChannel保存到了promise中，因此这儿能获取到）。

`promise.channel().unsafe()`实际上就是`NioServerSocketChannel.unsafe()`，它获取到的是`NioMessageUnsafe`对象。这个对象又是什么时候将其保存到NioServerSocketChannel的呢？在反射调用NioServerSocketChannel的构造器时，在NioServerSocketChannel父类的构造器中会进行unsafe属性的初始化，对于服务端channel而言，unsafe属性是NioMessageUnsafe实例对象；对于客户端channel而言，unsafe属性时NioSocketChannelUnsafe实例对象（记住这一点很重要，后面新连接的接入、数据的读写都是基于这两个Unsafe来实现的）。

由于这里是服务端channel，所以`promise.channel().unsafe().register(this, promise)`实际上就是调用NioMessageUnsafe类的register(this, promise)方法。NioMessageUnsafe继承了`AbstractUnsafe`类，`register(this, promise)`方法实际上定义在AbstractUnsafe类中，NioMessageUnsafe类并没有重写该发方法，因此最终会调用到`AbstractUnsafe.register(this, promise)`。终于到核心代码了！！！

`AbstractUnsafe.register(this, promise)`方法删减后的源码源码如下。

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // 省略部分代码....

    // 对服务端channel而言，这一步是给NioServerSocketChannel的eventLoop属性赋值
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

上面这段代码的逻辑，可以分为两部分。第一：通过`AbstractChannel.this.eventLoop = eventLoop`给NioServerSocketChannel的eventLoop属性赋值，这样后面服务端的channel就绑定在这个NioEventLoop上了，所有的操作均由这个线程来执行。第二：通过`eventLoop.inEventLoop()`来判断是同步执行register0()方法，还是异步执行register0()方法。

`eventLoop.inEventLoop()`的逻辑比较简单，就是判断当前线程和eventLoop中保存的线程是否相等，如果相等，就同步执行register0()；如果不相等，就异步执行register0()。此时由于当前线程是main()线程，肯定与eventLoop中的线程不相等，因此会通过eventLoop来异步执行register0()。

至今为止，依旧没看到服务端channel绑定到多路复用器上的代码。由此可见，绑定操作应该在register0()上，下面再看看register0()的代码。

```java
private void register0(ChannelPromise promise) {
    try {
        // 省略部分代码...
        boolean firstRegistration = neverRegistered;
        /**
         * 对于服务端的channel而言，doRegister()方法做的事情就是将服务端Channel注册到多路复用器上
         */
        doRegister();
        neverRegistered = false;
        registered = true;

        //会执行handlerAdded方法
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        //通过在pipeline传播来执行每个ChannelHandler的channelRegistered()方法
        pipeline.fireChannelRegistered();

           // 如果服务端channel已经激活，就执行下面逻辑。
           // 由于此时服务端channel还没有绑定端口，因此isActive()会返回false，不会进入到if逻辑块中
        if (isActive()) {
            // 省略部分代码
        }
    } catch (Throwable t) {
        // 省略部分代码...
    }
}
```

在register0()方法中，有三步重要的逻辑，第一：doRegister()；第二：pipeline.invokeHandlerAddedIfNeeded()；第三：pipeline.fireChannelRegistered()。下面分别来看看这三步都干了哪些事情。

`doRegister()`。doRegister()就是真正将服务端channel注册到多路复用器上的一步。doRegister()调用的是AbstractNioChannel类中的doRegister()方法，删减后源码如下。
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

其中`javaChannel()`获取的就是JDK中原生的ServerSocketChannel。

`eventLoop().unwrappedSelector()`获取的是JDK中原生的多路复用器Selector（底层的数据结构被替换了）。（EventLoop中的unwrappedSelector属性是在创建NioEventLoop时，初始化的，底层的数据结构也是这个时候被替换的）

所以`javaChannel().register(eventLoop().unwrappedSelector(), 0, this)`这一行代码，实际上就是调用JDK原生ServerSocketChannel的`register(selector,ops,attr)`方法，然后将服务端Channel注册到了多路复用器Selector上。

注意这里在调JDK原生的register()方法时，第三个参数传入的是this，此时this代表的就是当前的NioServerSocketChannel对象。将this作为一个attachment保存到多路复用器Selector上，这样做的好处就是，后面可以通过多路复用器Selector获取到服务端的channel。第二个参数传入的是0，表示此时将服务端channel注册到多路复用器上，服务端chennel感兴趣的事件标识符是0，即此时对任何事件都不感兴趣。（真正开始对接收事件感兴趣是在服务端channel监听端口之后）。

当doRegister()方法执行完以后，就会执行第二步：`pipeline.invokeHandlerAddedIfNeeded()`。这一步做的事情就是回调pipeline中handler的`handlerAdded()`方法。invokeHandlerAddedIfNeeded()方法的源码如下。

```java
final void invokeHandlerAddedIfNeeded() {
    assert channel.eventLoop().inEventLoop();
    if (firstRegistration) {
        firstRegistration = false;
        // 只有在第一次注册时候才会执行这儿的逻辑
        // 回调所有Handler的handlerAdded()方法
        callHandlerAddedForAllHandlers();
    }
}
```

只有当channel时第一次注册时，才会执行`callHandlerAddedForAllHandlers()`方法。核心逻辑在`callHandlerAddedForAllHandlers()`上。

```java
private void callHandlerAddedForAllHandlers() {
    final PendingHandlerCallback pendingHandlerCallbackHead;
    synchronized (this) {
        assert !registered;

        // 该通道本身已注册。
        registered = true;

        pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
        this.pendingHandlerCallbackHead = null;
    }
    // pendingHandlerCallbackHead属性的值是什么时候被初始化的？
    PendingHandlerCallback task = pendingHandlerCallbackHead;
    while (task != null) {
        task.execute();
        task = task.next;
    }
}
```

从`callHandlerAddedForAllHandlers()`方法的源码中我们可以发现，它的核心逻辑就是这个`while(task!=null)`循环，然后再循环中执行task。task的初始值就是`this.pendingHandlerCallbackHead`，即`DefaultChannelPipeline.pendingHandlerCallbackHead`。那么问题来了，pendingHandlerCallackHead属性的值是什么时候被初始化的。（接下来就会开始懵逼了，代码会各种跳跃，这个时候就要发挥IDEA的debug功能了）

在`initAndRegister()`方法中，会调用`init(channel)`方法，在init(channel)方法中通过pipeline.addLast()向pipeline中添加了一个ChannelInitializer类型的匿名类，在本文的`”上篇回顾“`这一部分中就提到说这一步非常重要，现在就来说下它究竟有多重要了。

在调用pipeline.addLast()方法时，最终会调用到DefaultChaannelPipeline的`addLast(EventExecutorGroup group, String name, ChannelHandler handler)`方法，该方法的源码如下。

```java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);
        newCtx = newContext(group, filterName(name, handler), handler);
        addLast0(newCtx);

        // 默认为false，当为false时，表示的channel还没有被注册到eventLoop中
        if (!registered) {
            //判断handlerState属性等于0  并且设置为1
            // pending:待定，听候
            newCtx.setAddPending();
            // 将ChannelHandlerContext封装成一个PendingHandlerCallback（实际上就是一个Runnable类型的Task）
            // 然后添加到回调队列中
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        //返回NioEvenGroup
        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            callHandlerAddedInEventLoop(newCtx, executor);
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

在上面这段代码中， `addLast0(newCtx)`就是真正将handler添加到pipeline中，但是在添加完成后，还执行了一个方法：`callHandlerCallbackLater(newCtx, true)`。将方法名称翻译过来就是：在稍晚一点回调执行handler的回调方法。下面看下`callHandlerCallbackLater(newCtx, true)`方法的具体逻辑。

```java
private void callHandlerCallbackLater(AbstractChannelHandlerContext ctx, boolean added) {
    assert !registered;
    // added为true
    PendingHandlerCallback task = added ? new PendingHandlerAddedTask(ctx) : new PendingHandlerRemovedTask(ctx);
    PendingHandlerCallback pending = pendingHandlerCallbackHead;
    if (pending == null) {
        pendingHandlerCallbackHead = task;
    } else {
        // Find the tail of the linked-list.
        while (pending.next != null) {
            pending = pending.next;
        }
        pending.next = task;
    }
}
```

参数ctx是个什么东西呢？它就是前面我们创建的ChannelInitializer这个匿名类封装成的AbstractChannelHandlerContext对象；参数added此时传入的是true。所以创建出来的task是`PendingHandlerAddedTask(ctx)`，然后我们可以发现，将创建出来的task最后赋值给了pendingHandlerCallbackHead属性。

回到前面的`callHandlerAddedForAllHandlers()`方法中，我们知道了pendingHandlerCallbackHead属性的值就是PendingHandlerAddedTask(ctx)，所以执行`task.execute()`时，就是执行PendingHandlerAddedTask对象的execute()。在execute()方法中会调用到`callHandlerAdded0(ctx)`方法，接着调用到`ctx.callHandlerAdded()`，ctx对象就是ChannelInitializer这个匿名类封装成的AbstractChannelHandlerContext对象。持续跟进`ctx.callHandlerAdded()`方法的源码，最终发现，它最终就会调用handler对象的handlerAdded()方法。到这里，终于找到了`handlerAdded()`方法是在哪儿回调的了。至此`pipeline.invokeHandlerAddedIfNeeded()`方法最终执行完了。

回到register0()方法中，当`pipeline.invokeHandlerAddedIfNeeded()`方法执行完成后，接着往下执行，代码会执行到`pipeline.fireChannelRegistered()`，也就是前面我们提到的第三步。这一步做的事情就是传播Channel注册事件，如何传播呢？就是沿着pipeline中的头结点这个handler开始，往后依次执行每个handler的`channelRegistered()`方法。

通过一步步跟进，可以看到`pipeline.fireChannelRegistered()`最终会调用到AbstractChannelContextHandler对象的`invokeChannelRegistered()`。该方法的源码如下。

```java
private void invokeChannelRegistered() {
    if (invokeHandler()) {
        try {
            // handler()就是获取handler对象
            ((ChannelInboundHandler) handler()).channelRegistered(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRegistered();
    }
}
```

其中handler()获取的就是当前AbstractChannelContextHandler对象中包装的handler对象，例如前面创建的`ChannelInitializer的匿名类`。然后调用handler对象的`channelRegistered(this)`方法，因此最终会调用到`ChannelInitializer的匿名类`的`channelRegistered(this)`方法。下面我们看看ChannelInitializer的匿名类的`channelRegistered(this)`方法又做了哪些事情。

```java
public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    // Normally this method will never be called as handlerAdded(...) should call initChannel(...) and remove
    // the handler.
    // 主要看initChannel(ctx)方法的逻辑
    if (initChannel(ctx)) {
        // we called initChannel(...) so we need to call now pipeline.fireChannelRegistered() to ensure we not
        // miss an event.
        ctx.pipeline().fireChannelRegistered();

        // We are done with init the Channel, removing all the state for the Channel now.
        removeState(ctx);
    } else {
        // Called initChannel(...) before which is the expected behavior, so just forward the event.
        ctx.fireChannelRegistered();
    }
}
```

可以看到，会先调用`initChannel(ctx)`方法，然后再调用`ctx.pipeline().fireChannelRegistered()`或者`ctx.fireChannelRegistered()`，后面这两个方法通过方法名就能看到，它就是继续向pipeline中传播执行handler的channelRegistered()方法，可以不用再看了。这里重点看看initChannel(ctx)方法。注意：这里的initChannel(ctx)方法的参数类型是`ChannelHandlerContext`类型，在后面还会出现一个initChannel(channel)方法，它的参数类型是`Channel`，这里特意提醒一下，不要搞混这两个重载方法。

initChannel(ctx)方法的源码如下。

```java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.add(ctx)) { // Guard against re-entrance.
        try {
            //ChannelInitializer匿名类的initChannel(channel)的方法 将在这里被调用
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
            // We do so to prevent multiple calls to initChannel(...).
            exceptionCaught(ctx, cause);
        } finally {
            //删除此节点
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

可以看到，在initChannel(ctx)方法中，先调用了initChannel(channel)方法。由于前面创建的ChannelInitializer匿名类中重写了initChannel(channel)方法，因此这个时候会调用到重写的initChannel(channel)方法。为了方便查看，下面通过截图给出ChannelInitializer匿名类实例创建时的代码。

【图】

在重写的initChannel(channel)方法中，我们可以看出，先向pipeline中添加了我们在main()中为服务端设置的handler，然后又通过`ch.eventLoop().execute()`这行代码，以异步的方式，向pipeline中添加了ServerBootstrapAcceptor类型的handler，ServerBootstrapAcceptor这个handler就是后面负责客户端接入的handler，非常重要。

到这里终于可以感叹一下，文章开头的第二问：什么时候回调intiChannel(channel)，现在终于知道了。

然而，还没结束。回到initChannel(ctx)的方法中，我们发现，在finally语句块中，做了一个很重要的操作，那就是将ChannelInitializer匿名类所代表的handler，从pipeline中移除了。这是因为这个匿名类存在的目的，就是为了在服务端channel初始化和注册的过程中，为其初始化pipeline的结构，现在服务端channel的初始化和注册工作已经完成了，而且服务端channel只会在服务启动时初始化一次，因此这个匿名类后面就没有存在的意义了，从而将其从pipeline中移除，所以最终服务端channel中的pipeline的结构图如下。

【图】

至此，initAndRegister()方法终于分析完了。

### 总结
* 本文主要分析了initAndRegister()方法的后半部分，即服务端channel注册到多路复用器的过程，其最终调用的是JDK中NIO包下，ServerSocketChannel的register方法，将服务端channel注册到多路复用器Selector上。
* 然后本文还通过对代码一步一步跟进，详细说明了ChannelInitializer匿名类实例中initChannel(channel)的回调过程，最终形成了服务端channel中的pipeline的结构。
* 至此，Netty服务端channel的初始化和注册已经完成，但是服务端的启动流程还没结束，还剩下最后一步：服务端channel和端口号绑定，端口绑定的流程，下一篇文章分析。


### 推荐
