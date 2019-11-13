### 简介
* CountDownLatch是JUC包下提供的一个工具类，它的作用是让一个或者一组线程等待其他线程执行完成后，自己再接着执行。从命名上可以猜出，它是通过倒着计数，最后打开门闩这把锁，即每一个线程忙完自己的工作后，让计数器递减一次，当计数器递减到0时，锁（门闩）被打开，主线程（或者等待的线程）接着执行自己的工作。
* 在实际工作中，我们可能会遇到需要利用多线程来处理问题的同时，还需要控制这些线程的执行顺序，这个时候我们可以选择使用Thread类中提供的join()方法，也可以使用今天即将介绍的CountDownLatch来解决，还可以使用JUC包下的另一个类CyclicBarrier来解决。

### 如何使用
* CountDownLatch的使用十分简单，它只有一个构造方法，在构造方法中需要传入一个int类型的参数，这个参数就是用来控制CountDownLatch需要递减多少次才释放锁（打开门闩）。CountDownLatch还提供了以下三个方法，详细信息见下表。

|方法名|方法作用|
|---|---|
|void await()|让调用该方法的线程阻塞，当CountDownLatch的计数器减为0时，才会让线程解阻塞|
|boolean await(long timeout, TimeUnit unit)|让调用该方法的线程超时阻塞，如果超过了指定的时间，CountDownLatch的计数器还没有减为0，那么线程就会直接返回|
|void countDown()|让CountDownLatch的计数器减1，当计数器的值减为0时，会让阻塞在CountDownLatch的线程解阻塞|

*  下面以一个简单的场景，简单介绍下CountDownLatch的用法。在学生时代，总会有各种各样的考试，每次考完试，各科老师都会进行阅卷，计算总分，总分排名。在这个过程中，各科的阅卷是同时进行的，由于每一科老师的阅卷速度不一样，因此计算总分和总分排名的人需要等到所有老师阅卷完成后才能进行。这个时候我们可以用CountDownLatch这个工具类在程序中进行模拟一下这个场景。把每一科的老师当做一个线程，由于每一科老师的阅卷速度不一样，因此采用让线程随机休眠一段时间，当每一科的老师阅卷完成后，就调用CountDownLatch的countDown()方法让计数器减一，在主线程中调用CountDownLatch的await()的方法，目的为了让主线程等待所有老师阅卷完成，当所有老师阅卷完成时，计数器就减为0了，主线程就会从await()方法处解阻塞，然后进行总分加和，排名等工作。示例的Demo如下。
```java
public class CountDownLatchDemo {

    public static void main(String[] args) {
        CountDownLatch countDownLatch = new CountDownLatch(6);
        List<String> teachers = Arrays.asList("语文老师","数学老师","英语老师","物理老师","化学老师","生物老师");
        Random random = new Random();
        // 创建6个线程，模拟6个科目的老师同时开始阅卷
        List<Thread> threads = new ArrayList<>(6);
        for (int i = 0; i < 6; i++) {
            threads.add(new Thread(()->{
                try {
                    int workTime = random.nextInt(6) + 1;
                    // 让线程睡眠一段时间，模拟老师的阅卷时间
                    Thread.sleep(workTime * 1000l);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "阅卷完成");
                // 每位老师阅卷完成后，就让计数器减1
                countDownLatch.countDown();
            },teachers.get(i)));
        }
        for (Thread thread : threads) {
            thread.start();
        }
        // 让主线程等待所有老师阅卷完成后，再开始计算总分，进行排名
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("所有科目的老师均已阅卷完成");
        System.out.println("开始计算总分，然后排名");
    }
}
```

### 源码分析
* 了解了CountDownLatch的使用，接下来就看下CountDownLatch的工作原理。
* CountDownLatch实际上是一种共享锁，它的底层是使用队列同步器AQS来实现的。CountDownLatch在构造方法中指定了计数器的初始值，即AQS中的同步变量state的值。在同一时刻它允许多个线程来访问state，每当调用一次countDown()方法的时候，让state的值减一；当调用await()方法时，会判断此时state的值是否为0，如果为0，就让当前线程返回，如果不为0，就让当前线程进入同步队列等待。
* 由于CountDownLatch是使用AQS来实现的，因此它的内部需要定义一个同步组件，该组件需要继承AQS（这种做法是AQS系列锁的通用做法），在CountDownLatch中定义了一个内部类Sync，Sync继承了AQS，并重写了AQS中的tryAcquireShared()、tryReleaseShared()方法。
* 当使用CountDownLatch的构造方法创建CountDownLatch时，在构造方法中会实例化Sync组件，并通过Sync的有参构造器，初始化AQS中同步变量state的值，值的大小就是CountDownLatch的构造方法中传入的int类型的数值，用来表示计数器的大小。
```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```
```java
private static final class Sync extends AbstractQueuedSynchronizer {

    Sync(int count) {
        setState(count);
    }
}
```
* 当调用CountDownLatch的countDown()方法时，会调用到sync.releaseShared(1)，即AQS的releaseShared()方法。releaseShared()方法的源码如下。
```java
public final boolean releaseShared(int arg) {
    // 尝试释放共享锁
    if (tryReleaseShared(arg)) {
        /**
         * 当释放锁完成后，同步状态state=0，此时说明后面的线程可以获取锁了
         * 如果此时同步队列中有人的等待，就唤醒后面的线程
         * 如果无人等待，就将首节点的waitStatus设置为-3，表示同步状态可以无条件的传播下去，即后面的线程都可以直接获取锁了
         */
        doReleaseShared();
        return true;
    }
    return false;
}
```
* 在releaseShared()方法中，会先调用子类的tryReleaseShared()方法，那么这里就会调用到CountDownLatch中的内部类Sync的tryReleaseShared()方法。源码如下。
```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    // 将同步状态进行减一，减一之后，同步状态变为0，就返回true，表示可以唤醒同步队列中正在等待的线程了
    for (;;) {
        int c = getState();
        // 在对state进行减一操作之前，会先判断一下state的值是否为0，如果state已经为0了，此时还有线程来对state进行减1，这个时候是不正常的操作，因此会返回false
        if (c == 0)
            return false;
        int nextc = c-1;
        // 利用CAS操作来设置state的值
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```
* tryReleaseShared()方法的作用就是让state的值减1，如果减一之后state的值变为0，就返回true，表示此时可以唤醒同步队列中正在等待的线程了，如果不为0，就返回false。当tryReleaseShared()方法结束时，就会回到AQS的releaseShared()方法中，如果tryReleaseShared()方法返回的是false，那么releaseShared()就直接返回结束了；如果tryReleaseShared()方法返回的是true，那么接着就会执行doReleaseShared()方法。doReleaseShared()方法是AQS中定义的一个模板方法，是用来处理共享锁的逻辑的，它的主要作用就是唤醒同步队列中正在等待的线程。
* 当调用CountDownLatch的await()时，会调用到sync.acquireSharedInterruptibly(1)，即AQS的acquireSharedInterruptibly()方法。acquireSharedInterruptibly()方法的源码如下。
```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    // 响应中断
    if (Thread.interrupted())
        throw new InterruptedException();
    // tryAcquireShared()方法是尝试获取锁
    // 对于CountDownLatch而言，当state=0时，会返回1，这表示锁被所有的线程都释放了，当state不等于0时，会返回-1，表示还有线程持有锁
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```
* 在acquireSharedInterruptibly()方法中会先调用子类的tryAcquireShared()方法，在这里调用的是CountDownLatch中的内部类Sync的tryAcquireShared()方法。tryAcquireShared()方法的逻辑十分简单，判断了state的值是否为0，如果为0，就返回1，不为0，就返回-1。当state为0时，表示的是计数器的值被减为了0，这表明所有线程已经完成了自己的工作，所以这个时候tryAcquireShared()方法返回1，那么回到AQS中时，acquireSharedInterruptibly()方法就会直接结束，当前线程不会阻塞。如果state不为0，就说明计数器的值还没被减为0，还有线程没有执行完自己的工作，没有调用countDown()方法，因此这个时候tryAcquireShared()方法返回-1，那么回到AQS中时，acquireSharedInterruptibly()方法不会直接结束，而是接着执行doAcquireSharedInterruptibly()方法。doAcquireSharedInterruptibly()方法是AQS的一个模板方法，该方法的主要作用就是处理共享锁相关逻辑，如果共享锁获取失败时，就让线程进入到同步队列中park。在此处，如果计数器的值不为0，那么当前线程调用await()方法后，就会进入同步队列中park，直到有其他线程调用countDown()方法将计数器减为0时，才会将当前线程唤醒，或者当前线程被中断。
```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

* await(long timeout, TimeUnit unit)方法的实现逻辑和await()方法类似，只不过在await()方法的基础上添加了一个超时判断，有兴趣的朋友可以自行研究下。整体来讲，CountDownLatch的原理相对而言比较简单，和共享锁的实现原理一样，只不过tryAcquireShared()方法和tryReleaseShared()方法的逻辑稍微有所改动。

### 总结
* 本文首先介绍了CountDownLatch的作用，它可以让一个线程或者一组线程等待其他线程执行完成后，自己才运行，可以实现控制线程执行的顺序的目的。然后本文通过一个阅卷、计算总分、排名的案例演示了CountDownLatch的使用方法。最后简单分析了CountDownLatch的源码实现，CountDownLatch的底层是根据AQS的共享锁的相关方法来实现的。
* CountDownLatch在使用过程中需要注意的是，最好使用`await(long timeout, TimeUnit unit)`来阻塞线程，因为如果处理任务的子线程一直不执行完，就会一直不调用countDown()方法，这样计数器就不会减为0，导致主线程一直阻塞等待，什么也干不了，所以推荐使用await(long timeout, TimeUnit unit)。如果在具体业务中，必须要求所有线程执行完后再执行主线程的话，那就使用await()方法，不过在子线程处理任务的代码中，最好使用try...catch...finally语句，然后在finally语句块中进行countDown()调用，否则很容易给自己挖下深坑。
* 本人有一次在使用CountDownLatch的过程中，由于使用的是await()方法，然后在子线程中也没有使用try...catch...finally，最后因为有一个线程在处理业务逻辑时报错了，最后导致这个线程没有调用countDown()方法，所以主线程一直阻塞，一直等在那儿。更惨的是，由于这是子线程中出了异常，也没有try...catch...finally，这个时候主线程会 `吞掉`子线程的堆栈异常信息，最终导致什么错误日志也不打印。由于这段代码是在服务启动的时候就会执行，所以当时的现象就是，服务始终起不来，错误日志不也打印。当时碰到这个问题的时候，这段代码在线上已经运行了好久，只有在测试环境才出现，所以压根就没往这个地方去想。再加上自身对多线程相关的知识掌握度几乎为0，查了好久都没找到原因，服务器重启了n次，眼看重启大法不也好使了，只能向同事请教，最后终于找到了这个错误。最后的解决办法就是在子线程中使用try...catch...finally，然后在finally语句块中调用countDown()。这个问题其实用jstack命令，在服务器上看看线程的堆栈就能查到是哪儿出问题了，但还是因为自身很菜，对多线程没有足够的了解，才花费了很长时间去解决。也正是因为这次的教训，让我决定开始去学习并发相关的源码。
