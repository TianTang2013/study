> 扫描下方二维码或者微信搜索公众号·菜鸟飞呀飞·，即可关注微信公众号，阅读更多`Spring源码分析`和`Java并发编程`文章。

### Runnable与Callable
众所周知，当我们使用线程来运行`Runnable`任务时，是不支持获取返回值的，因为Runnable接口的run()方法使用`void修饰`的，方法不支持返回值。而在很多场景下，我们一方面需要通过线程来异步执行任务，以便提升性能，另一方面还期望能获取到任务的执行结果。尤其是在RPC框架中，异步获取任务返回值，几乎是每一个RPC接口要实现的功能。这个时候，使用Runnable显然就无法满足我们的需求了，因此`Callable`就出现了。

Callable与Runnable类似，它是一个接口，也只有一个方法：`call()`，不同的是Callable的call()方法有是有返回值的，返回值的类型是一个泛型，泛型由创建Callable对象时指定。
```java
public interface Callable<V> {
    V call() throws Exception;
}
```
Runnable对象可以传入到Thread类的构造方法中，通过Thread来运行Runnable任务，而Callable接口则不能直接传入到Thread中来运行，Callable接口通常结合线程池来使用。线程池ThreadPoolExecutor中除了提供execute()方法来提交任务以外，还提供了submit()的三个重载方法来提交任务，这三个方法均有返回值。
ThreadPoolExecutor类继承了抽象类`AbstractExecutorService`，在AbstractExecutorService中定义了submit()重载的三个方法。具体定义如下。

|方法名|说明|
|--|--|
|Future<?> `submit(Runnable task)`|该方法虽然返回值对象是Future，但是由于提交的是Runnable类型的任务，所以使用`Future.get()`获取结果时会返回null。|
|Future<T> `submit(Runnable task,T result)`|方法的返回值对象是Future，通过`Future.get()`获取具体的返回值时，结果与方法的第二个参数result相等。|
|Future<T>  `submit(Callable task)`|该方法的参数是一个`Callable类型`的对象，方法有返回值。调用Future.get()获取到值就是Callable接口的call()方法返回的值|

可以看到，submit()的三个重载方法的返回值均是Future类型的对象，那么Future又是何方神圣呢？

### Future与FutureTask
当任务提交到线程池后，我们可能需要获取任务的返回值，或者想要知道任务有没有执行完成，甚至有时候因为特殊情况需要取消任务，那么这个时候应该怎么办呢？

在JUC包下，为我们提供了一个工具类：`Future`。Future是一个接口，它提供了5个方法，当一个任务通过submit()方法提交到线程池后，线程池会返回一个Future类型的对象，我们可以通过Future对象的这5个方法来获取任务在线程池中的状态。这些方法定义如下。

|方法名|说明|
|--|--|
|`boolean cancel(boolean mayInterruptIfRunning)`|用来取消任务，mayInterruptIfRunning参数用来表示是否需要中断线程，如果传true，表示需要中断线程，那么就会将任务的状态设置为`INTERRUPTING`；如果为false，那么就会将任务的状态设置为`CANCELLED`（关于任务的状态`INTERRUPTING`和`CANCELLED`后面会说明）|
|`boolean isCancelled()`|判断任务是否已经被取消了，返回true表示被取消了|
|`boolean isDone()`|判断任务是否已经完成|
|`V get() `|获取任务的返回值，会一直阻塞当前线程，直到获取到任务的返回值|
|`V get(long timeout, TimeUnit unit)`|以超时的形式获取任务的返回值，如果在超时时间内没获取到任务的返回值，那么抛出TimeoutException异常|

Future接口有一个具体的实现类：`FutureTask`。事实上线程池ThreadPoolExecutor的三个submit()重载方法，返回的Future类型的对象，都是FutureTask的实例对象。FutureTask的UML图如下。
【图】

从UML图上可以看到，FutureTask是直接实现了`RunnableFuture`接口，而RunnableFuture接口又继承了Runnable和Future接口，因此FutureTask既是Runnable类型，又是Future类型。
当调用submit()方法来向线程池中提交任务时，无论提交的是Runnable类型的任务，还是提交的是Callable类型的任务，最终都是将任务封装成一个FutureTask对象。下面以`Future<T> submit(Callable<T> task)`方法为例，来看下源码。
```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    // 调用newTaskFor()将Callable任务封装成一个FutureTask
    RunnableFuture<T> ftask = newTaskFor(task);
    // 执行任务
    execute(ftask);
    return ftask;
}

// newTaskFor
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    // 直接new一个FutureTask对象
    return new FutureTask<T>(callable);
}
```
当submit()方法提交的是Runnable任务时，会调用`newTaskFor()方法的另一个重载方法`来将任务封装成一个FutureTask对象，所以说最终线程池中执行的都是FutureTask类型的任务。（注意，Runnable类型的对象，最终会通过`Executors.callable()`方法，将Runnable对象封装为一个Callable类型的对象。Executors.callable()的原理是使用`适配器模式`，适配器为 `RunnableAdapter`类）

知道了Future与这些类之间的关系后，下面就来分析下线程池是如何执行一个FutureTask任务的？以及又是如何通过Future.get()方法就能获取到任务的返回值的？

### 设计原理与数据结构

想要弄明白Future接口的几个方法的原理，那么就必须先搞明白FutureTask的数据接口，以及其设计的原理。

既然想要在线程池外部通过其他线程获取到池中任务的状态，而线程池中的任务都是FutureTask类型，那么在FutureTask这个对象中，肯定存在和任务状态有关的变量。

在FutureTask中定义了十分重要的属性。如下表所示。

|属性|含义|
|--|--|
|int state|state变量用来保存任务的状态，它的取值有0~6，7个值，每个值分别表示不同的含义，具体含义见下方说明|
|Callable<V> callable|callable表示的是我们提交的任务，Runnable类型的任务会通过Executors.callable()来转变为Callable|
|Object outcome|用来保存Callable的call()方法的返回值|
|Thread runner|保存执行当前任务的线程|
|WaitNode waiters|用来保存等待获取任务返回值的线程的等待队列，当我们在主线程中调用Future.get()方法时，就会将主线程封装成一个WaitNode。当有多个线程同时调用Future.get()方法时，WaitNode会通过next属性来维护一个链表|

state的取值有7种。每种取值的含义如下代码注释。
```java
private volatile int state;
// 任务的初始状态，当新建一个FutureTask任务时，state值默认为NEW
private static final int NEW          = 0;
// 任务处于完成中，什么是完成中呢？有两种情况
// 1. 任务正常被线程执行完成了，但是还没有将返回值赋值给outcome属性
// 2. 任务在执行过程中出现了异常，被捕获了，然后处理异常了，在将异常对象赋值给outcome属性之前
private static final int COMPLETING   = 1;
// 任务正常被执行完成，并将任务的返回值赋值给outcome属性之后，会处于NORMAL状态
private static final int NORMAL       = 2;
// 任务出了异常，并将异常对象赋值给outcome属性之后
private static final int EXCEPTIONAL  = 3;
// 调用cancle(false)，任务被取消了
private static final int CANCELLED    = 4;
// 调用cancle(true)，任务取消，但是在线程中断之前
private static final int INTERRUPTING = 5;
//  调用cancle(true)，任务取消，但是在线程中断之后
private static final int INTERRUPTED  = 6;
```
虽然任务的状态有7中取值，但大致可以将其分为三类：初始状态、中间状态、最终状态。这些状态的变化关系，如下图所示。

【图】

* 当一个任务被提交到线程池后，它的初始状态为`NEW`；当任务被正常执行完成后，会先将任务的状态设置为`COMPLETING`；然后将任务的返回值（即Callable的call()方法的返回值）赋值给FutureTask的`outcome`属性，当赋值完成后，再将任务的状态设置为`NORMAL`。这是一个任务正常执行的流程，也就是对应图中`①`所示的线路。
* 当任务被提交到线程池后，线程在执行任务中出现了异常，那么会现将任务的状态由`NEW`设置为`COMPLETING`；然后将异常对象赋值给`outcome`属性，当赋值完成后，再将任务状态设置为`EXCEPTIONAL`。这是任务出现异常的情况，也就是对应图中`②`所示的线路。
* 当任务被提交到线程池后，如果调用Future对象的cancle()方法，当cancle()传入的传入的参数为false时，会直接将任务的状态由`NEW`设置为`CANCELLED`，也就是对应图中`③`所对应的路线。
* 当cancle()方法传入的参数为true时，会先将任务状态设置为`INTERRUPTING`；然后调用执行当前任务的线程的`interrupt()`方法，最后再设置任务状态为`INTERRUPTED`，也就是图中`④`所对应的线路。

### 源码分析
当调用submit()方法提交任务到线程池后，会先调用`newTaskFor()`方法将任务封装成一个FutureTask对象，然后调用execute()方法来执行任务。在execute()方法中会先启动Worker线程，当线程启动后，会调用线程的runWorker()方法。在`runWorker()`方法中最终会调用到`task.run()`方法，也就是FutureTask的run()方法。关于这一步详细的源码分析可以参考这篇文章：[线程池ThreadPoolExecutor的实现原理](https://mp.weixin.qq.com/s/q0Qt-ha9ps12c15KMW7NfA)

下面只分析下FutureTask.run()方法。在run()方法中，最终会调用callable属性的call()方法。当任务正常执行完后，会调用FutureTask的`set()`方法来更新任务的状态以及保存任务的返回值，最后唤醒获取任务结果的处于等待中的线程。如果出现异常，将会调用`setException()`方法来更新任务状态，保存异常，唤醒等待中的线程。下面是run()方法的源码，我对源码进行了删减，只保留了核心逻辑。
```java
public void run() {
    ......
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 执行任务
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                // 出异常时将state置为EXCEPTIONAL
                setException(ex);
            }
            if (ran)
                // 设置任务状态为COMPLETING，然后保存返回值，最后再设置为NORMAL
                set(result);
        }
    } finally {
        // 其他处理
        ......
    }
}
```
对于`set()`和`setException()`方法，比较简单，就是通过CAS来更新任务的状态，然后将任务的返回值赋值给`outcome`属性，最后调用`finishCompletion()`方法唤醒`waiters`这个属性构成的等待队列中的线程。（关于CAS相关的原理和知识，可以参考这两篇文章：[初识CAS的实现原理](https://mp.weixin.qq.com/s/oe046IRUbYeXpIpYLz19ew) 和 [Unsafe类的源码解读以及使用场景](https://mp.weixin.qq.com/s/V8Rc3OlcI6D66ggCP2scRQ)）

接下来结合具体的源码来分析下`Future.get()`方法的执行过程。当调用Future.get()方法时，会调用FutureTask的get()方法。在get()方法，首先判断任务有没有完成，如果已经完成了，就直接返回结果，如果没有完成，则进行等待。
```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    // 如果状态处于NEW或者COMPLETING状态，表示任务还没有执行完成，需要等待
    if (s <= COMPLETING)
        // awaitDone()进行等待
        s = awaitDone(false, 0L);
    // 返回结果
    return report(s);
}
```
通过调用`report(s)`方法返回结果，在report()方法中，会先判断任务是不是处于`NORMAL`状态，即任务是否是被正常执行完成，只有正常执行完成了，才会返回结果，否则抛出对应的异常。
```java
private V report(int s) throws ExecutionException {
    Object x = outcome;
    // 只有任务正常结束时，才会返回
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```
当任务处于`NEW或者COMPLETING`状态时，表示任务正处于执行中或者任务的返回值还没有被赋值给outcome属性，所以这个时候，还不能返回结果，因此需要进入等待状态，即调用`awaitDone()`方法。在awaitDone()方法中，有一个无限for循环，先判断任务是否是处于COMPLETING状态。如果处于COMPLETING状态，就让当前线程先放弃CPU的调度权（为什么要放弃CPU的调度权呢？因为从COMPLETING变为NORMAL状态，或者其他状态，是一段很短的过程，让当前线程先放弃CPU的调度权，以便让其他线程得到CPU资源，而CPU的时间片也是一段很短的时间，当下次线程在获取到CPU资源的时候，此时任务的状态大概率会变为NORMAL或者其他最终状态，由于代码是处于for循环中的，所以会进入下一次循环）。如果当前任务不是处于COMPLETING状态，就会让线程进行park等待，具体是park超时等待呢，还是非超时等待呢？由awaitDone()方法传入的参数决定。（当调用park()方法后，这些线程又是在什么时候被唤醒的呢？当任务的状态变为`最终状态`后，会调用`finishCompletion()`方法，唤醒这些处于等待中的线程。）

下面是awaitDone()方法的部分源码，我对源码进行了删减，只保留的主要逻辑。
```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    ......
    for (;;) {
        ......
        // 任务处于COMPLETING中，就让当前线程先暂时放弃CPU的执行权
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        ......                               
        else if (timed) {
            // 超时时间计算
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            // 等待一段时间
            LockSupport.parkNanos(this, nanos);
        }
        else
            // 等待
            LockSupport.park(this);
    }
}
```

### 总结
* 本文主要介绍了Runnable接口和Callable接口的区别，前者没有返回值，能被Thread直接执行；后者有返回值，不能被Thread直接执行需要通过线程池来执行。
* 接着介绍了Future接口的5个方法，以及它的实现类FutureTask的几个重要属性以及数据结构。无论是Runnable还是Callable对象，当提交到线程池后，均是被封装成一个FutureTask对象后执行。对于Future的使用场景，在Netty和Dubbo均有大量的应用。
* 最后结合源码详细介绍了FutureTask的get()方法和run()方法的实现原理。
