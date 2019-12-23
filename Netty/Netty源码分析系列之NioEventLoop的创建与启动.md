### 前言

前三篇文章分别分析了 Netty 服务端 channel 的初始化、注册以及绑定过程的源码，理论上这篇文章应该开始分析新连接接入过程的源码了，但是在看源码的过程中，发现有一个非常重要的组件：**NioEventLoop**，出现得非常频繁，以至于影响到了后面源码的阅读，因此决定先分析下**NioEventLoop**的源码，再分析新连接接入的源码。关于**NioEventLoop**这个组件的源码分析，将会写两篇文章来分享。第一篇文章将主要分析**NioEventLoop 的创建与启动**，第二篇将主要分析**NioEventLoop 的执行流程**。

在开始之前，先来思考一下一下两个问题。

1. Netty 中的线程是何时启动的？
2. Netty 中的线程是如何实现串行无锁化的？

### 功能说明

NioEventLoop 从功能上，可以把它当做一个线程来理解，当它启动以后，它就会不停地循环处理三种任务（从类名上也能体现出循环处理的思想：**Loop**）。这三种任务分别是哪三种任务呢？

1. 网络 IO 事件；
2. 普通任务。通过调用**execute(Runnable task)** 来执行普通任务。
3. 定时任务。通过调用**schedule(Runnable task,long delay,TimeUnit unit)** 来执行定时任务。

NioEventLoop 类的继承关系特别复杂，它的 UML 图如下。
【图】

从图中可以看到，它实现了**ScheduledExecutorService**接口，因此它可以实现定时任务相关的功能；同时它还继承了**SingleThreadEventExecutor**类，从类名看，这是一个单线程的线程执行器。

### 创建流程

在 netty 中，我们通过**NioEventLoopGroup**来创建**NioEventLoop**，入口就是下面这一行代码。

```java
EventLoopGroup workerGroup = new NioEventLoopGroup()
```

当使用**NioEventLoopGroup**的无参构造器时，netty 会默认创建**2 倍 CPU 核数数量的 NioEventLoop**；当使用 NioEventLoopGroup 的有参构造方法时，向构造方法中传入一个 int 值，就表示创建指定个数的 NioEventLoop。无论是使用 NioEventLoopGroup 有参构造方法，还是无参构造方法，最终都会调用到 NioEventLoopGroup 类中的如下构造方法。

```java
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory) {
    // 调用父类
    super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}
```

这个构造方法有很多参数，此时每个参数的解释如下。

- nThreads：要创建的线程的数量，如果前面使用的是 NioEventLoopGroup 无参构造器，此时 nThreads 的值为 0，如果使用的是 NioEventLoopGroup 的有参构造方法，nThreads 的值为构造方法中传入的值。
- executor：线程执行器，默认是 null，这个属性的值会在后面创建 NioEventLoop 时，进行初始化。用户可以自定义实现 executor，如果用户自定义了，那么此时 executor 就不为 null，后面就不会再进行初始化。
- selectorProvider：SelectorProvider 类型，它是通过**SelectorProvider.provider()** 创建出来的，这是 JDK 中 NIO 相关的 API，会创建出一个 SelectorProvider 对象，这个对象的作用就是创建多路复用器 Selector 和服务端 channel。
- selectStrategyFactory：选择策略工厂，通过 DefaultSelectStrategyFactory.INSTANCE 创建，**INSTANCE**这个常量的值又是通过**new DefaultSelectStrategyFactory()** 来创建的。
- RejectedExecutionHandlers.reject()：返回的是一个拒绝策略，当向线程池中添加任务时，如果线程池任务队列已满，这个时候任务就会被拒绝，此时线程池就会执行拒绝策略。

接着又会调用父类的构造方法，**NioEventLoopGroup**直接继承了**MultithreadEventLoopGroup**类，此时会调用到**MultithreadEventLoopGroup**的如下构造方法。

```java
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

可以看到，构造方法的参数中，有一个 args 参数，是一个 Object 类型的可变数组。因此当在 NioEventLoopGroup 的构造方法调用到父类中时，**selectorProvider、selectStrategyFactory、RejectedExecutionHandlers.reject()** 都变成了 args 这个可变数组中的元素了。另外，我们从代码中可以知道，如果前面传递过来的 nThread 为 0，那么就令 nThread 的值等于**DEFAULT_EVENT_LOOP_THREADS**，而**DEFAULT_EVENT_LOOP_THREADS**这个常量的值就是 2 倍的 CPU 核数；如果前面传递过来的 nThread 不为 0，就使用传递过来的 nThread。

接着继续向上调用父类的构造器，MultithreadEventLoopGroup 继承了**MultithreadEventExecutorGroup**类，因此会调用到**MultithreadEventExecutorGroup**的如下构造方法。

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}
```

其中**nThreads、executor、args**这些参数就是前面传过来，这里就不多说了。然后通过**DefaultEventExecutorChooserFactory.INSTANCE**创建的是一个事件执行选择工厂，INSTANCE 常量的值是通过**new DefaultEventExecutorChooserFactory()** 创建出来的对象。
接着又通过 this 调用了 MultithreadEventExecutorGroup 类中的另一个构造方法，接下来这个构造方法就是核心代码了。该构造方法的代码很长，为了方便阅读，我进行了精简，精简后的源码如下。

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }

    if (executor == null) {
        /**
         * 创建线程执行器：ThreadPerTaskExecutor
         * newDefaultThreadFactory()会创建一个线程工厂，该线程工厂的作用就是用来创建线程，同时给线程设置名称：nioEventLoop-1-XX
         */
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }
    // 根据传进来的线程数，来创建指定大小的数组大小，这个数组就是用来存放NioEventLoop对象实例
    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        //出现异常标识
        boolean success = false;
        try {
            //创建nThreads个nioEventLoop保存到children数组中
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            // 异常处理...
        }
    }
    // 通过线程执行器选择工厂来创建一个线程执行器
    chooser = chooserFactory.newChooser(children);

    // 省略部分代码...
}
```

这个方法中，有三处主要的逻辑。第一处逻辑：当 executor 为空时，创建一个**ThreadPerTaskExecutor**类型的线程执行器；第二处逻辑：通过**newChild(executor, args)** 来创建**NIoEventLoop**；第三处：通过**chooserFactory.newChooser(children)** 来创建一个线程执行器的选择器。下面将逐步详细分析这三处逻辑。

#### 创建线程执行器

```java
if (executor == null) {
    /**
     * 创建线程执行器：ThreadPerTaskExecutor
     * newDefaultThreadFactory()会创建一个线程工厂，该线程工厂的作用就是用来创建线程，同时给线程设置名称：nioEventLoop-1-XX
     */
    executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
}
```

在第一处核心逻辑处，首先判断 executor 是否为空，如果用户没有自己指定，默认情况下，executor 是 null，因此就会通过 new 关键字来创建一个**ThreadPerTaskExecutor**类型的线程执行器。

在调用**ThreadPerTaskExecutor**的构造方法之前，先通过**new DefaultThreadFactory()** 创建了一个线程工厂，该线程工厂是**DefaultThreadFactory**类型，它实现了 ThreadFactory 接口，它的作用就是：当调用 threadFactory 的 newThread()方法时，就会创建出一个线程，同时给线程取一个有意义的名称，名称生成规则为：nioEventLoop-xx-xx。第一个 xx 的含义表示的 NiEventLoopGroup 的组号，在 netty 中可能同时创建 bossGroup 和 workerGroup 两个线程组，所以第一个 xx 表示线程组的序号。第二个 xx 表示的是线程在线程组中的序号。如：nioEventLoop-1-1 表示的是该线程是第一个 NioEventLoopGroup 线程组的第一个线程。

**ThreadPerTaskExecutor**类的源码比较简单，它实现了**Executor**接口，重写了**execute()** 方法，当每次调用**ThreadPerTaskExecutor**类的**execute()** 方法时，会创建一个线程，并启动线程。这里可能会有一个疑问：每次调用**execute()** 方法，都会创建一个线程，岂不是意味着会创建很多线程？实际上，在每个**NioEventLoop**中，只会调用一次**ThreadPerTaskExecutor**的**execute()** 方法，因此对于每个**NioEventLoop**而言，只会创建一个线程，且当线程启动后，就不会再调用**ThreadPerTaskExecutor**的**execute()** 方法了，也就不会造成在系统中创建多个线程。

```java
public final class ThreadPerTaskExecutor implements Executor {

    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        // threadFactory就是前面创建的DefaultThreadFactory
        // 通过线程工厂的newThread()方法来创建一个线程，并启动线程
        threadFactory.newThread(command).start();
    }
}
```

#### 创建 NioEventLoop

```java
// 根据传进来的线程数，来创建指定大小的数组大小，这个数组就是用来存放NioEventLoop对象实例
children = new EventExecutor[nThreads];

for (int i = 0; i < nThreads; i ++) {
    //出现异常标识
    boolean success = false;
    try {
        //创建nThreads个nioEventLoop保存到children数组中
        children[i] = newChild(executor, args);
        success = true;
    } catch (Exception e) {
        throw new IllegalStateException("failed to create a child event loop", e);
    } finally {
        // 异常处理...
    }
}
```

在执行第二处核心逻辑之前，先创建了一个**EventExecutor**类型的数组，数组的大小就是前面传进来的线程个数，然后将数组赋值给**children**属性，这个属性是 NioEventLoopGroup 的属性，NioEventLoopGroup 包含一组 NioEventLoop 线程，children 属性就是用来存放这一组 NioEventLoop 线程的。此时只是创建出了数组，但是数组中的元素都是 null，所以接下来通过 for 循环来为数组填充元素，通过**newChild(executor, args)** 创建出一个 NioEventLoop 对象，然后将对象赋值给数组中的元素。

当调用**newChild(executor, args)** 方法时，第一个参数 executor 就是上一步创建出来的**ThreadPerTaskExecutor**对象，第二个参数是一个可变数组，它的每一个元素是什么，有什么作用，在前面已经解释过了。**newChild(executor, args)** 定义在 NioEventLoopGroup 类中，源码如下。

```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    /**
     * executor: ThreadPerTaskExecutor
     * args:  args是一个可变数组的参数，实际上它包含三个元素，也就是前面传递过来的三个参数，如下：
     *          SelectorProvider.provider()是JDK中NIO相关的API，会创建出一个SelectorProvider，它的作用就是在后面创建多路复用器Selector和服务端channel
     *          DefaultSelectStrategyFactory.INSTANCE 是一个默认选择策略工厂，new DefaultSelectStrategyFactory()
     *          RejectedExecutionHandlers.reject()返回的是一个拒绝策略，当向线程池中添加任务时，如果线程池任务队列已满，这个时候任务就会被拒绝，然后执行拒绝策略
     *
     */
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
}
```

可以看见，在**newChild()** 中直接调用了 NioEventLoop 的构造方法。NioEventLoop 的构造方法源码如下。

```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
             SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler) {
    /**
     * executor: ThreadPerTaskExecutor
     * args:  args是一个可变数组的参数，实际上它包含三个元素，也就是前面传递过来的三个参数，如下：
     *          SelectorProvider.provider()是JDK中NIO相关的API，会创建出一个SelectorProvider，它的作用就是在后面创建多路复用器Selector和服务端channel
     *          DefaultSelectStrategyFactory.INSTANCE 是一个默认选择策略工厂，new DefaultSelectStrategyFactory()
     *          RejectedExecutionHandlers.reject()返回的是一个拒绝策略，当向线程池中添加任务时，如果线程池任务队列已满，这个时候任务就会被拒绝，然后执行拒绝策略
     *
     */
    super(parent, executor, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);
    if (selectorProvider == null) {
        throw new NullPointerException("selectorProvider");
    }
    if (strategy == null) {
        throw new NullPointerException("selectStrategy");
    }
    provider = selectorProvider;
    // openSelector()方法会创建一个多路复用器，但是这个多路复用器的selectedKey的底层数据接口被替换了
    final SelectorTuple selectorTuple = openSelector();
    //替换了数据结构selectedKeys   publicSelectedKeys的原生selector
    selector = selectorTuple.selector;
    //子类包装的selector  底层数据结构也是被替换了的
    unwrappedSelector = selectorTuple.unwrappedSelector;
    selectStrategy = strategy;
}
```

在 NioEventLoop 的构造方法中主要干了两件事，一是继续向上调用父类的构造方法，二是调用**openSelector()** 方法。在父类的构造方法中，会初始化两个任务队列：tailTasks 和 taskQueue，最终两个属性创建出来的都是**MpscQueue**类型的队列，同时还将传入的 executor 进行了一次包装，通过 ThreadExecutorMap 将其包装成了一个匿名类。（MpscQueue 是个什么东西呢？它是**many producer single consumer**的简写，意思就是同一时刻可以有多个生产者往队列中存东西，但是同一时刻只允许一个线程从队列中取东西。）

接着是调用**openSelector()** 来创建多路复用器 Selector，然后将多路复用器保存到 NioEventLoop 当中，在这一步 Netty 对多路复用器进行了优化。原生的 Selector 底层存放 SelectionKey 的数据结构是**HashSet**，**HashSet 在极端情况下，添加操作的时间复杂度是 O(n)** ，Netty 则将 HashSet 类型**替换成了数组类型**，这样添加操作的时间复杂度**始终是 O(1)** 。openSelector()方法的源码很长，下面以图片的方式贴出其源码，你也可以直接跳过源码，看我后面的总结。

【图】

openSelector()方法的源码很长，经过整理后，可以总结为如下几个步骤：

- 先调用 JDK 的 API 创建多路复用器 Selector：provider.openSelector();
- 通过**DISABLE_KEY_SET_OPTIMIZATION**属性判断是否禁用优化，如果为 true，则表示不进行底层数据结构的替换，即不优化，直接返回原生的 Selector。**DISABLE_KEY_SET_OPTIMIZATION**常量的含义是是否禁用优化：即是否禁止替换底层数据结构，默认为 false，不禁止优化。可以通过 **io.netty.noKeySetOptimization** 来配置。
- 通过反射加载 SelectorImpl：**Class.forName("sun.nio.ch.SelectorImpl",false,PlatformDependent.getSystemClassLoader())** ；
- 通过反射获取原生 SelectorImpl 中的**selectedKeys、publicSelectedKeys**属性（这两个属性的数据类型是**HashSet 类型**），然后再将这两个属性的访问权限设置为 true，接着再通过反射，将**selectedKeys、publicSelectedKeys**这连个属性的类型替换为 Netty 中自定义的数据类型：**SelectedSelectionKeySet**。该类型的底层数据结构是**数组类型**；
- 最后将**SelectedSelectionKeySet**封装到 netty 自定义的多路复用器**SelectedSelectionKeySetSelector**中，然后将 JDK 原生的 Selector 和 Netty 自定义的 Selector 封装到**SelectorTuple**中，再将**SelectorTuple**返回。注意：这里原生的 selector 的底层数据结构在返回时已经被替换成了数组。

至此，NioEventLoop 的创建已经完成了，总结一下创建 NioEventLoop 的创建过程干了哪些事。

- 初始化了两个队列：taskQueue 和 tailQueue，类型均为 MpscQueue。taskQueue 队列是用来存放任务的队列，后面 NioEventLoop 启动后，就会循环的从这个队列中取出任务执行；tailQueue 是用来存放一些收尾工作的队列。
- 将前面传入的**ThreadPerTaskExecutor**通过**ThreadExecutorMap**将其包装成了一个匿名类，然后保存到 NioEventLoop 的 executor 属性中，后面就能通过 NioEventLoop 来获取到线程执行器，然后执行任务了。
- 将拒绝策略：RejectedExecutionHandlers.reject()和选择策略工厂 DefaultSelectStrategyFactory.INSTANCE 保存到 NioEventLoop 中，方便后面从 NioEventLoop 中获取。
- 将 JDK 原生的多路复用器 Selector 保存到 NioEventLoop 的**unwrappedSelector**属性中，将 Netty 自定义的多路复用器 SelectedSelectionKeySetSelector 保存到 NioEventLoop 的**selector**属性中。unwrappedSelector 和 selector 底层的数据类型都是**数组**类型。

#### 线程执行器选择工厂

当 NioEventLoop 全部创建完成后，就会接着执行第三处核心逻辑，这一步做的工作是通过一个选择工厂来创建一个线程执行器的选择器，即给 chooser 属性赋值。看到这儿，可能有点懵，什么意思呢？为什么要创建这个选择器呢？

Netty 的 NioEventLoopGroup 包含了一组线程，即一组 NioEventLoop，当有新的连接接入到服务端后，后面需要对这个新连接来进行 IO 事件的读写，那这个时候需要使用一个 NioEventLoop 来和这个新连接绑定，也就是和客户端 channel 绑定，后续对这个客户端 channel 的数据读写都是基于绑定的这个 NioEventLoop 来进行的。既然有多个 NioEventLoop 线程，那么这个时候应该从线程组中选择哪一个 NioEventLoop 来和客户端 channel 绑定呢？

Netty 的做法是：轮询，第一个客户端 channel 来了后，取线程组中的第一个线程，即 children 数组中的第一个元素；然后当第二个线程来时，取数组中的第二个元素，以此类推，循环的从 children 数组中取 NioEventLoop。这个算法很简单，如何实现呢？就是每来一个客户端 channel，先获取计数器的值，然后用计数器的值对数组取模，然后再将计数器加一。

由于取模运算相对于位运算而言，是一个相对耗时的过程，因此 netty 对此进行了优化。当线程数是 2 的整数次方时，netty 就采用位运算的方式来进行取模运算；当线程数不是 2 的整数次方时，netty 就还是采用取模的方法去进行计算。这两种计算方法分别是由两个类来实现的：**PowerOfTwoEventExecutorChooser 和 GenericEventExecutorChooser**，这两个类都是 EventExecutorChooser 类型，翻译过来就是事件执行器的选择器。

而 chooser 就是这两个选择器的实例，究竟是**PowerOfTwoEventExecutorChooser**类型的实例还是**GenericEventExecutorChooser**类型的实例呢，这取决于 nThread 的数量。**newChooser(EventExecutor[] executors)** 方法的源码如下。

```java
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    // executors是 new NioEventLoop() 的对象数组,
    // executors.length的值就是前面nThread参数的值
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}
```

**isPowerOfTwo(int val)** 方法就是判断传入的值是否是 2 的整数次方。如何判断呢？又是通过位运算。下面代码可能不太直观，举个栗子：比如传入的参数是 8，那么 8 和-8 用二进制表示就是：

```properties
8:    00000000000000000000000000001000
-8:    11111111111111111111111111111000
```

将 8 和-8 进行与运算，结果还是 8，与原数值相等，因此 8 是 2 的整数次方。

```java
private static boolean isPowerOfTwo(int val) {
    return (val & -val) == val;
}
```

至此，NioEventLoopGroup 的创建过程就结束了，那么 NioEventLoop 的创建过程也就跟着结束了。那么问题来了，我们说 NioEventLoop 实际上就是一个线程，既然是线程，它就必须先启动，才能轮询地执行任务，而在整个创建过程的源码中，我们都没有看到 NioEventLoop 线程启动相关的代码，那么 NioEventLoop 是什么时候启动的呢？

### 启动

NioEventLoop 启动的触发时机有两个，一是在服务端启动的过程中触发，另一个是在新连接接入的时候。下面以服务端启动的过程为例子，进行分析。在服务端启动过程中，会执行如下一行代码。

```java
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```

**next()** 就是 chooser 的一个方法，chooser 有两种不同的实现：**PowerOfTwoEventExecutorChooser 和 GenericEventExecutorChooser**，这两种不同的实现对**next()** 方法有不同的实现逻辑，区别就是：是用位运算从 children 数组中取出一个 NioEventLoop，还是通过取模的方式从 children 数组中取出一个 NioEventLoop，但是最终都是返回一个 NioEventLoop。

所以这儿实际上是执行**NioEventLoop 的 register(channel)方法**，这个方法一直向下执行，最终会执行到如下代码：

```java
eventLoop.execute(new Runnable() {
    @Override
    public void run() {
        register0(promise);
    }
});
```

在这儿会调用 NioEventLoop 的 execute()方法，NioEventLoop 继承了 SingleThreadEventExecutor，execute(task)定义在 SingleThreadEventExecutor 类中，删减后的源码如下。

```java
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    // 判断当前线程是否和NioEventLoop中的线程是否相等，返回true表示相等
    boolean inEventLoop = inEventLoop();
    // 将任务加入线程队列
    addTask(task);
    if (!inEventLoop) {
        // 启动线程
        startThread();
        // 省略部分代码...
    }
    // 省略部分代码
}
```

可以看到，先判断当前线程是否和 NioEventLoop 中的线程是否相等，此时由于线程是 main 线程，**inEventLoop()** 会返回 false，所以会进入到 if 逻辑块中，并调用**startThread()** 方法来启动的线程。

```java
private void startThread() {
    // 处于为启动状态，才会去尝试启动线程
    if (state == ST_NOT_STARTED) {
        // 尝试将ST_NOT_STARTED设置为ST_STARTED
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                doStartThread();
                success = true;
            } finally {
                // 如果执行doStartThread()出现异常  将STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED回滚
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            }
        }
    }
}
```

在**doStart()** 中，会先判断 NioEventLoop 是否处于**未启动状态**，只有处于未启动状态才会去尝试启动线程。在启动线程之前，会先利用 CAS 方法，将状态标识为**启动状态**，CAS 成功后，然后再调用**doStartThread()** 方法。**doStartThread()** 方法精简后的源码如下。

```java
private void doStartThread() {
    assert thread == null;
    //真正的启动线程
    executor.execute(new Runnable() {
        @Override
        public void run() {
            // 将此线程保存起来
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                // 启动NioEventLoop
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                // 省略部分代码....
            }
        }
    });
}
```

可以发现，在**doStartThread()** 方法中，调用的 executor 属性的**execute()** 方法，注意，此时 executor 属性值是什么？在创建 NioEventLoop 时，创建了一个**ThreadPerTaskExecutor**类型的对象，然后再通过**ThreadExecutorMap**将其包装成了一个匿名类，最后将这个匿名类赋值给了 executor 属性。所以此时会调用匿名类的**execute(Runnable task)** 方法，而这个匿名类最最终还是调用的是**ThreadPerTaskExecutor**的**execute(Runnable task)** 方法。在前面已经简单分析了**ThreadPerTaskExecutor**的**execute(Runnable task)** 方法，现在为了方便阅读，再次贴出这部分代码。

```java
public void execute(Runnable command) {
    // threadFactory就是前面创建的DefaultThreadFactory
    // 通过线程工厂的newThread()方法来创建一个线程，并启动线程
    threadFactory.newThread(command).start();
}
```

可以看到，该**execute()** 方法，就是调用线程工厂的**newThread(command)** 方法来创建一个线程，然后调用线程的**start()** 的方法启动线程。当线程启动后，就会回调传入的 Runnable 任务的 run()方法，所以接着会回调到**doStartThread()** 方法中传入的 Runnable 的 run()方法。从**doStartThread()** 的源码中可以看到，在 run()方法中先将创建出来的线程保存了起来，然后会调用 **SingleThreadEventExecutor.this.run()** 。这一行代码就是启动 NioEventLoop 线程，该方法的源码很长，整个 NioEventLoop 的核心都在这个方法上，它实际上就是在一个无限 for 循环中，不停的去处理事件和任务。关于这个方法的源码会在下一篇文章详细分析。

至此，NioEventLoop 中的 Thread 线程已经启动了，同时会连带着 NioEventLoop 不停的在无限 for 循环中执行，也就是 NioEventLoop 启动起来了。

### 总结

- 本文以**new NioEventLoopGroup()** 为切入点，通过分析**NioEventLoopGroup**的源码，从而分析了**NioEventLoop**的创建过程，同时还介绍了 Netty 对 NIO 的优化。接着以服务端 channel 启动的流程为入口，分析了 NioEventLoop 是如何启动的。
- 默认情况下，netty 会创建 2 倍 CPU 核数数量的**NioEventLoop**线程，如果显示指定了数量，则创建指定数量的**NioEventLoop**。
- 最后回答下文章开头的两个问题。
- 第一个问题：Netty 中的线程是何时启动的？启动时机有两个，一个是在服务端启动的过程中触发，另一个是在新连接接入的时候，但是最终都是调用**ThreadPerTaskExecutor**类的**execute(Runnable command)** 方法，通过线程工厂来创建一个线程，然后调用线程的**start()** 方法启动线程，当线程启动后，又会回调传入的 Runnable 任务的 run()方法，在任务的 run()方法中通过调用**SingleThreadEventExecutor.this.run()** 来调用 NioEventLoop 的 run()方法，这样就启动了 NioEventLoop。
- 第二个问题：Netty 中的线程是如何实现串行无锁化的？从源码中我们可以知道，每个 NioEventLoop 中只包含一个线程，而每个 channel 只会绑定在一个 NioEventLoop 上，一但绑定上了，后面这个 channel 的所有 IO 操作都会交由这个 NioEventLoop 线程来处理，因此不会出现多个 NioEventLoop 线程来争夺处理 channel 的情况，因此说在 NioEventLoop 上，所有的操作都是串行处理的，不存在锁的竞争，即串行无锁化。可能有人会问，串行处理任务，岂不是降低了系统的吞吐量？显然不是的，因为 netty 中有多个 NioEventLoop 线程，多个 NioEventLoop 同时串行处理，这样服务既是多线程并行运行，各个线程间又不存在锁的竞争，大大提高了服务性能。
