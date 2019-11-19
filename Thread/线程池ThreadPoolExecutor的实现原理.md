### 前言
&emsp;&emsp;无论是在工作中，还是在书本中，我们都可以听到或者看到关于线程在使用时的一些建议：不要在代码中自己直接创建线程，而是通过线程池的方式来使用线程。使用线程池的理由大致可以总结为以下几点。
* `1.` 降低资源消耗。线程是操作系统十分宝贵的资源，当多个人同时开发一个项目时，在互不知情的情况下，都自己在代码中创建了线程，这样就会导致线程数过多，而且线程的创建和销毁，在操作系统层面，需要由`用户态切换到内核态`，这是一个`费时费力`的过程。而使用线程池可以避免频繁的创建线程和销毁线程，线程池中线程可以重复使用。
* `2.` 提高响应速度。当请求到达时，由于线程池中的线程已经创建好了，使用线程池，可以省去线程创建的这段时间。
* `3.` 提高线程的可管理性。线程是稀缺资源，当创建过多的线程时，会造成系统性能的下降，而使用线程池，可以对线程进行统一分配、调优和监控。
&nbsp;
&emsp;&emsp;线程池的使用十分简单，但是会用不代表用得好。在面试中，基本不会问线程池应该怎么用，而是问线程池在使用不当时会造成哪些问题，实际上就是考察线程池的实现原理。因此搞明白线程池的实现原理是很有必要的一件事，不仅仅对面试会有帮助，也会让我们在平时工作中避过好多坑。

### 实现原理
&emsp;&emsp;在线程池中存在几个概念：核心线程数、最大线程数、任务队列。核心线程数指的是线程池的基本大小；最大线程数指的是，同一时刻线程池中线程的数量最大不能超过该值；任务队列是当任务较多时，线程池中线程的数量已经达到了核心线程数，这时候就是用任务队列来存储我们提交的任务。
&emsp;&emsp;与其他池化技术不同的是，线程池是基于`生产者-消费者`模式来实现的，任务的提交方是生产者，线程池是消费者。当我们需要执行某个任务时，只需要把任务扔到线程池中即可。线程池中执行任务的流程如下图如下。
【图】
* `1.` 先判断线程池中线程的数量是否超过核心线程数，如果没有超过核心线程数，就创建新的线程去执行任务；如果超过了核心线程数，就进入到下面流程。
* `2.` 判断任务队列是否已经满了，如果没有满，就将任务添加到任务队列中；如果已经满了，就进入到下面的流程。
* `3.` 再判断如果创建一个线程后，线程数是否会超过最大线程数，如果不会超过最大线程数，就创建一个新的线程来执行任务；如果会，则进入到下面的流程。
* `4.` 执行拒绝策略。
> &emsp;&emsp;`在没看线程池的具体实现之前，我一直存在这样的疑惑：为什么是先判断任务队列有没有满，再判断有没有超过最大线程数？正常逻辑不是应该先尽可能的创建线程，让线程去处理任务吗？当任务实在是太多了，线程处理不过来了，再将任务添加到任务队列吗？知道我看了线程池的具体代码实现后，我才知道答案。问题答案在文末。`

### ThreadPoolExecutor
&emsp;&emsp;在JUC包下，已经提供了线程池的具体的实现：`ThreadPoolExecutor`。ThreadPoolExecutor提供了很多的构造方法，其中最复杂的构造方法有7个参数。
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
}
```
* `corePoolSize`：该参数表示的是线程池的核心线程数。当任务提交到线程池时，如果线程池的线程数量还没有达到corePoolSize，那么就会新创建的一个线程来执行任务，如果达到了，就将任务添加到任务队列中。
* `maximumPoolSize`：该参数表示的是线程池中允许存在的最大线程数量。当任务队列满了以后，再有新的任务进入到线程池时，会判断再新建一个线程是否会超过maximumPoolSize，如果会超过，则不创建线程，而是执行拒绝策略。如果不会超过maximumPoolSize，则会创建新的线程来执行任务。
* `keepAliveTime`：当线程池中的线程数量大于corePoolSize时，那么大于corePoolSize这部分的线程，如果没有任务去处理，那么就表示它们是空闲的，这个时候是不允许它们一直存在的，而是允许它们最多空闲一段时间，这段时间就是keepAliveTime，时间的单位就是TimeUnit。
* `unit`：空闲线程允许存活时间的单位，`TimeUnit`是一个枚举值，它可以是纳秒、微妙、毫秒、秒、分、小时、天。
* `workQueue`：任务队列，用来存放任务。该队列的类型是阻塞队列，常用的阻塞队列有`ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue、PriorityBlockingQueue`等。
&nbsp;
`ArrayBlockingQueue`是一个基于数组实现的阻塞队列，元素按照先进先出（FIFO）的顺序入队、出队。因为底层实现是数组，数组在初始化时必须指定大小，因此ArrayBlockingQueue是有界队列。
&nbsp;
`LinkedBlockingQueue`是一个基于链表实现的阻塞队列，元素按照先进先出（FIFO）的顺序入队、出队。因为顶层是链表，链表是基于节点之间的指针指向来维持前后关系的，如果不指链表的大小，它默认的大小是`Integer.MAX_VALUE`，即$2^{32}-1$，这个数值太大了，因此通常称LinkedBlockingQueue是一个无界队列。当然如果在初始化的时候，就指定链表大小，那么它就是有界队列了。
&nbsp;
`SynchronousQueue`是一个不存储元素的阻塞队列。每个插入操作必须得等到另一个线程调用了移除操作后，该线程才会返回，否则将一直阻塞。吞吐量通常要高于LinkedBlockingQueue。
&nbsp;
`PriorityBlockingQueue`是一个将元素按照优先级排序的阻塞的阻塞队列，元素的优先级越高，将会越先出队列。这是一个无界队列。

* `threadFactory`：线程池工厂，用来创建线程。通常在实际项目中，为了便于后期排查问题，在创建线程时需要为线程赋予一定的名称，通过线程池工厂，可以方便的为每一个创建的线程设置具有业务含义的名称。
* `handler`：拒绝策略。当任务队列已满，线程数量达到maximumPoolSize后，线程池就不会再接收新的任务了，这个时候就需要使用拒绝策略来决定最终是怎么处理这个任务。`默认情况下使用AbortPolicy，表示无法处理新任务，直接抛出异常`。在ThreadPoolExecutor类中定义了四个内部类，分别表示四种拒绝策略。我们也可以通过实现`RejectExecutionHandler`接口来实现自定义的拒绝策略。
> `AbortPocily`：不再接收新任务，直接抛出异常。
> `CallerRunsPolicy`：提交任务的线程自己处理。
> `DiscardPolicy`：不处理，直接丢弃。
> `DiscardOldestPolicy`：丢弃任务队列中排在最前面的任务，并执行当前任务。（排在队列最前面的任务并不一定是在队列中待的时间最长的任务，因为有可能是按照优先级排序的队列）

&emsp;&emsp;在使用ThreadPoolExecutor时，可以使用`execute(Runnable task)`方法向线程池中提交一个任务，没有返回值。也可以使用`submit()`方法向线程池中添加任务，有返回值，返回值对象是Future。submit()方法有三个重载的方法。
* `submit(Runnable task)`，该方法虽然返回值对象是Future，但是使用`Future.get()`获取结果是null。
* `submit(Runnable task,T result)`，方法的返回值对象是Future，通过`Future.get()`获取具体的返回值时，结果与方法的第二个参数result相等。
* `submit(Callable task)`，该方法的参数是一个`Callable类型`的对象，方法有返回值。
* 关于Future的原理有兴趣的朋友可以自己先了解下，后面一篇文章会专门介绍。


### 源码分析
&emsp;&emsp;了解了ThreadPoolExecutor的基本用法，下面将结合源码来分析下线程池的代码实现。从上面的线程池的原理中，我们可以发现，线程池的原理相对比较简单，代码实现起来应该不难，看源码主要是为了学习他人写的优秀代码，尤其是编程大师Doug Lea写的代码。
&nbsp;
&emsp;&emsp;对于一个线程池，除了上面介绍的几个重要属性以外，我们还需要一个变量来表示线程池状态，线程池也需要有运行中、关闭中、已关闭等状态。如果要我们去实现一个线程池，可能第一反应就是用一个单独的变量来表示线程池的状态，再用另一个变量来表示线程池中线程的数量。这样的确可以，不过Doug Lea并不是这样实现的，他将两者用一个变量来表示（不得不感叹下，大佬就是大佬，想法果然和他人不一样）。那么问题来了，如何用一个变量来表示两个值？阅读过读写锁源码的朋友可能就能立想到另外这一中方案（关于读写锁的介绍可以阅读这一篇文章： [读写锁ReadWriteLock的实现原理](https://mp.weixin.qq.com/s/KDQR4_MaR_NacE8tKfQILA)）。在读写锁的实现中将一个int型的数值，按照高低位来拆分，高位表示一个数，低位再表示另一个数，在线程池的实现中，Doug Lea再次使用了按位拆分的技巧。
&nbsp;
&emsp;&emsp;在线程池中，使用了一个原子类AtomicInteger的变量来表示线程池状态和线程数量，该变量在内存中会占用4个字节，也就是32bit，其中高3位用来表示线程池的状态，低29位用来表示线程的数量。线程池的状态一共有5中状态，用3bit最多可以表示8种状态，因此采用高3位来表示线程池的状态完全能满足需求。示意图如下。
【图】

&emsp;&emsp;在看线程池的核心实现逻辑之前，先简单看下ThreadPoolExecutor中关于各种变量和方法的定义。因为在核心逻辑中，经常会用到它们，而且这些方法和变量大量用到了位运算，看起来不是特别直观，所以提前熟悉它们的功能对于看核心逻辑有很大的帮助。相关源码和注释如下。
```java
// 高2位表示线程池的状态，其他29位表示线程的数量，即worker的数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 计数的位数，29
private static final int COUNT_BITS = Integer.SIZE - 3;
// 线程池的最大大小，2^29 -1，
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
// 线程池的状态，高三位表示线程的运行状态，
// RUNNING: 111
private static final int RUNNING    = -1 << COUNT_BITS;
// SHUTDOWN: 000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// STOP: 001
private static final int STOP       =  1 << COUNT_BITS;
// TIFYING(整理): 010
private static final int TIDYING    =  2 << COUNT_BITS;
// TERMINATED: 011
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
// 计算线程池的状态，计算结果的低29位全为0，因此最终结果就是线程池的状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 工作线程的数量，计算结果的高3位全是0，因此最终结果就是工作线程的数量
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 根据线程池的状态和工作线程的数量，计算ctl，实际上就是将两者合并成ctl
private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 * Bit field accessors that don't require unpacking ctl.
 * These depend on the bit layout and on workerCount being never negative.
 */

// 线程池的状态是否小于s
private static boolean runStateLessThan(int c, int s) {
    return c < s;
}

// 线程池的状态大于等于s
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}

// 判断线程池是否处于运行状态
private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```
&emsp;&emsp;有了前面的基础，接下来分析下核心实现。再使用线程池时我们可以通过`execute(Runnable task)`来提交一个任务到线程池，因此我们从核心入口execute(Runnable task)方法开始分析。execute()方法的源码如下。在源码中我添加了部分注释，以供参考。
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    
    int c = ctl.get();
    // workerCountOf(c)方法时计算工作线程的数量，
    // 1. 工作线程数是否小于核心线程数，如果小于核心线程数，就创建新的线程去执行任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        // 任务执行失败后，重新获取ctl的值，供下面的逻辑计算
        c = ctl.get();
    }
    // 2. 当工作线程数大于等于核心线程数时，线程池是运行状态，且任务添加到队列成功
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 如果线程池状态不是RUNNING状态，就执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果线程池中线程的数量为0，就调用addWorker()方法创建新的worker线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 3. 线程池非运行状态，或者任务入队失败，就尝试创建新的worker线程
    else if (!addWorker(command, false))
        // 4. 如果创建新的worker线程失败，就执行拒绝策略。
        reject(command);
}
```
&emsp;&emsp;`execute()`方法的逻辑大致分为4部分，分别对应线程池原理部分所提到的4个步骤。
* 先通过`workerCountOf(c)`方法计算当前线程池中线程的数量，然后与初始化线程池时指定的核心线程数相比较，如果小于核心线程数，就调用`addWorker()`方法，实际上就是创建新的线程来执行任务。addWorker()方法的源码后面分析。
* 如果当前线程数小于核心线程数，那么就通过`workQueue.off(command)`方法，将任务添加到任务队列中。如果此时任务队列还没有满，那么就会添加成功，workQueue.off(command)就会返回true，那么就会进入到if逻辑块中，进行一些其他的判断。如果此时任务队列已经满了，workQueue.off(command)方法就会返回false，那么就会执行后面的3和4。
* 当任务队列满了以后，就会再次调用`addWorker()`方法，在addworker()方法中会在创建新的线程之前，判断线程数会不会超过最大线程数，如果会，addworker()就会返回false。如果不会，就会创建新的线程去执行任务。
* 如果addWorker()返回true，表示不能再创建新的线程了，那么此时就会执行到4，即调用reject()方法，执行拒绝策略。reject()方法的逻辑比较简单，就是调用了我们指定的`handler`的`rejectedExecution()`方法。其源码如下。
```java
final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
```
&emsp;&emsp;从上面的分析中，可以看出，在好几处逻辑中均调用了`addWorker()`方法，这说明该方法十分重要。事实也是如此，该方法不仅重要，代码实现还十分复杂。该方法需要两个参数，第一个参数就是我们传入的Runnable任务，第二参数是一个boolean值，传入true表示的是当前线程池中的线程数还没有达到核心线程数，传false表示当前线程数已经大于等于核心线程数了。`addWorker()`方法的源码很长，这里我将其分为两个部分，下面先看前面一部分的源码。
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 1. 当线程池的状态大于SHUTDOWN时，返回false。因为线程池处于关闭状态了，就不能再接受任务了
        // 2. 当线程池的状态等于SHUTDOWN时，firstTask不为空或者任务队列为空，返回false
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            // 1. 线程数大于等于理论上的最大数(2^29-1)，则直接返回false。（因为线程数不能再增加了）
            // 2. 根据core来决定线程数应该和谁比较。当线程数大于核心线程数或者最大线程数时，直接返回false。
            // （因为当大于核心线程数时，表示此时任务应该直接添加到队列中(如果队列满了，可能入队失败)；当大于最大线程数时，肯定不能再新创建线程了，不然设置最大线程数有毛用）
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 线程数+1，如果设置成功，就跳出外层循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 再次获取线程池的状态，并将其与前面获取到的值比较，
            // 如果相同，说明线程池的状态没有发生变化，继续在内循环中进行循环
            // 如果不相同，说明在这期间，线程池的状态发生了变化，需要跳到外层循环，然后再重新进行循环
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    // 省略后半部分代码
    // ......
}
```
&nbsp;
&emsp;&emsp;在这部分代码中，我们可以看到一个很陌生的语法：`retry...break retry...continue retry`。这种写法真的是太少见了，少见到我第一眼看到的时候，以为是我不小心碰到键盘，把源码给改了。这种语法有点类似于C语言里面的goto（已经被遗弃），其作用就是在for循环之前定义一个retry，然后在循环中，使用`break retry`时，就会让代码`跳出循环，并不再进入循环`；当使用`continue retry`时，表示`跳出当前循环，立马进入到下一次循环`。由于这里使用了两层for循环，因此为了方便`在内层循环中一下子跳出到外层循环`，就使用了retry这种语法。需要说明的是，这里的retry并不是Java里面的关键字，而是随机定义的一个字符串，我们也可以写成a,b,c等，但是后面的break和continue后面的字符串需要和前面定义的这个字符串对应。
&emsp;&emsp;我们可以看到，前半部分的核心逻辑就是使用了两个无限for和一个CAS操作来设置线程池的线程数量。如果线程池的线程数修改成功，就中断循环，进入后半部分代码的逻辑，如果修改失败，就利用for循环再一次进行修改，这样的好处是，既实现了线程安全，也避免使用锁，提高了效率。在这一部分代码中，进行了很多判断，这些判断主要是校验线程池的状态以及线程数，个人认为不是特别重要，我们主要抓住核心逻辑即可。
&emsp;&emsp;当成功修改线程数量以后，就会执行addWorker()方法的后半部分代码，其源码如下。
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    // 省略前半部分代码
    // ......
    
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 创建一个新的worker线程
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                // 再次获取线程池的状态，因为在获取锁期间，线程池的状态可能改变了
                int rs = runStateOf(ctl.get());

                // 如果线程池状态时运行状态或者是关闭状态但是firstTask是空，就将worker线程添加到线程池
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 判断worker线程是否已经启动了，如果已经启动，就抛出异常
                    // 个人觉得这一步没有任何意义，因为worker线程是刚new出来的，没有在任何地方启动
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {

        // 如果启动失败，就将worker线程从线程池移除，并将线程数减1
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
&emsp;&emsp;在这一部分代码中，通过`new Worker(firstTask)`创建了一个Worker对象，Worker对象`继承了AQS`，同时`实现了Runnable接口`，它是线程池中真正干活的人。我们提交到线程的任务，最终都是`封装成Worker对象，然后由Worker对象来完成任务`。先简单看下Woker的构造方法，其源码如下。在构造方法中，首先设置了同步变量state为-1，然后通过ThreadFactory创建了一个线程，`注意在通过ThreadFactory创建线程时，将Worker自身也就是this，传入了进去，也就是说最后创建出来的线程对象，它里面的target属性就是指向这个Worker对象`。
```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable{

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 将this传入进去，最后就是使线程的target属性等于当前的worker对象
        this.thread = getThreadFactory().newThread(this);
    }
}
```
&emsp;&emsp;当通过`new Worker(firstTask)`创建完worker对象后，此时线程已经被创建好了，在启动线程之前，先通过`t.isAlive()`判断线程是已经启动，如果没有启动，才会调用线程的`start()`方法来启动线程。这里有一点需要注意的是，创建完worker对象后，调用了`mainLock.lock()`来保证线程安全，因为这一步`workers.add(w)`存在并发的可能，所以需要通过获取锁来保证线程安全。
&emsp;&emsp;当调用线程的start()方法之后，如果线程获取到CPU的执行权，那么就会执行线程的run()方法，在线程的run()方法中，会执行线程中target属性的run()方法。这里线程的target属性就是我们创建的worker对象，因此最终会执行到Worker的run()方法。
```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable{

    public void run() {
        runWorker(this);
    }
}
```
&emsp;&emsp;在Worker类的`run()`中，直接调用了`runWorker()`方法。所以Worker执行任务的核心逻辑就是在`runWorker()`方法中实现的。其源码如下。
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 如果worker线程的firstTask不为空或者能从任务队列中获取到任务，就执行
        // 否则就会一直阻塞到getTask()方法处
        while (task != null || (task = getTask()) != null) {
            // 保证worker串行的执行线程
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // beforeExecute()是空方法，由子类具体实现
                // 该方法的目的是为了让任务执行前做一些其他操作
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 真正执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // afterExecute()空方法，由子类具体实现
                    // 该方法的目的是为了让任务执行后做一些其他操作
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
&nbsp;
&emsp;&emsp;`runWorker()`的源码看起来也比较长，但核心逻辑就一行，即：`task.run()`，最终执行了我们提交的Runnable任务。在runWorker()方法中通过一个`while循环来让Worker对象一直来执行任务`。当传入的task对象不为空或者通过`getTask()`方法能从任务队列中获取到任务时，worker就会一直执行。否则将在finally语句块中调用`processWorkerExit`退出，让线程中断，最终销毁。
&emsp;&emsp;`getTask()`方法是一个阻塞的方法，当能从任务队列中获取到任务时，就会立即返回一个任务。如果获取不到任务，就会阻塞。它支持超时，当超过线程池初始化时指定的线程最大存活时间后，就会返回null，`从而导致worker线程退出while循环，最终线程销毁`。
&emsp;&emsp;到这儿线程池的`execute()`方法就分析完了。最后简单分析下线程池的`shutdown()`方法和`shutdownNow()`方法。当调用shutdown()方法时，会令线程池的状态为`SHUTDOWN`，然后中断空闲的线程，`对于已经在执行任务的线程并不会中断`。当调用shutdownNow()方法时，会令线程池的状态为`STOP`，然后在`中断所有的线程，包括正在执行任务的线程`。
* shutdown()方法的源码如下。
```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 权限校验
        checkShutdownAccess();
        // 将线程池的状态设置为SHUTDOWN状态
        advanceRunState(SHUTDOWN);
        // 中断空闲的worker线程
        interruptIdleWorkers();
        // 空方法，由子类具体去实现，例如ScheduledThreadPoolExecutor
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    // 尝试将线程池的状态设置为TERMINATED
    tryTerminate();
}
```
* shutdownNow()方法的源码如下。
```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 线程池状态设置为STOP
        advanceRunState(STOP);
        // 中断所有线程，包括正在执行任务的线程
        interruptWorkers();
        // 清除任务队列中的所有任务
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
&nbsp;
* 在实际工作当中，对于究竟应该使用哪一种方法去中断线程池，应该结合具体的任务来决定，如果`要求任务必须执行完成，那么就是用shutdown()方法`。通常也建议使用shutdown()方法，更加优雅。

### 总结
* 本文主要介绍了线程池的实现原理，其原理主要分为4个核心步骤，先判断线程数量是否超过核心线程数，然后再判断任务队列是否已经满了，再判断线程数会不会超过设置的最大线程数，最后执行拒绝策略。接着本文详细介绍了线程池的几个核心参数`corePoolSize,maximumPoolSize,keepAliveTime,unit,workQueue,threadFactory,handler`以及它们的各自的意义，然后结合源码实现，详细分析了任务的执行过程。
* 无论是读写锁的实现，还是线程池的实现，Doug Lea都使用了将一个int型的变量按照高低位拆分的技巧，这种思想很值得学习，不仅是因为设计巧妙，还因为在计算机中位运算的执行效率更高。
* 最后解释下关于文章开头提到的一点疑惑：为什么是先判断任务队列有没有满，再判断线程数有没有超过最大线程数？而不是先判断最大线程数，再判断任务队列是否已满？
* 答案和具体的源码实现有关。因为当需要创建线程的时候，都会调用`addWorker()`方法，在addWorker()的后半部分的逻辑中，会调用`mainLock.lock()`方法来获取全局锁，而获取锁就会造成一定的资源争抢。如果先判断最大线程数，再判断任务队列是否已满，这样就会造成线程池原理的4个步骤中，第1步判断核心线程数时要获取全局锁，第2步判断最大线程数时，又要获取全局锁，这样相比于先判断任务队列是否已满，再判断最大线程数，就可能会`多出一次获取全局锁的过程`。因此在设计线程池，为了`尽可能的避免因为获取全局锁而造成资源的争抢`，所以会先判断任务队列是否已满，再判断最大线程数。
* 另外一个疑惑就是：`LinkedBlockingQueue`的吞吐量比`ArrayBlockingQueue`的吞吐量要高。前者是基于链表实现的，后者是基于数组实现的，正常情况下，不应该是数组的性能要高于链表吗？
* 最后看了一下这两个阻塞队列的源码才发现，这是因为LinkedBlockingQueue的`读和写操作使用了两个锁`，takeLock和putLock，读写操作`不会造成资源的争抢`。而ArrayBlockingQueue的`读和写使用的是同一把锁`，读写操作`存在锁的竞争`。因此LinkedBlockingQueue的吞吐量高于ArrayBlockingQueue。

### 推荐
* [管程:并发编程的基石](https://mp.weixin.qq.com/s/6jvA5jnnMkr5l-IliDL8yw)
* [初识CAS的实现原理](https://mp.weixin.qq.com/s/oe046IRUbYeXpIpYLz19ew)
* [Unsafe类的源码解读以及使用场景](https://mp.weixin.qq.com/s/V8Rc3OlcI6D66ggCP2scRQ)
* [队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw)
* [队列同步器（AQS）源码分析](https://mp.weixin.qq.com/s/xIZlHydPxLmnClNckqKrZw)
* [可重入锁（ReentrantLock）源码分析](https://mp.weixin.qq.com/s/VHCVoBn3KBt95VMgT1wQDA)
* [公平锁与非公平锁的对比](https://mp.weixin.qq.com/s/hYLpsP_9Oxc9pSx3AGW5TQ)
* [Condition源码分析](https://mp.weixin.qq.com/s/cb8Xtr1wGorfArzNbfbrrw)
* [读写锁ReadWriteLock的实现原理](https://mp.weixin.qq.com/s/KDQR4_MaR_NacE8tKfQILA)
* [Semaphore的源码分析以及使用场景](https://mp.weixin.qq.com/s/zoP6rdG_PyLJ9RflV_DLew)
* [并发工具类CountDownLatch的源码分析以及使用场景](https://mp.weixin.qq.com/s/2W3nc1G-I78WEb1ABRO5YQ)
* [并发工具类CyclicBarrier的源码分析以及使用场景](https://mp.weixin.qq.com/s/UooC-Rk1ncyqBDY3po5n7w)
* [wait()和notify()一定成对出现吗？如何解释Thread.join()](https://mp.weixin.qq.com/s/qj2_x3WGIw7vg9zE-UNIVw)


