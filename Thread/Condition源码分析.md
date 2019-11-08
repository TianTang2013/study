> 点击上方`菜鸟飞呀飞`，即可关注微信公众号。

[TOC]

> 队列同步器AQS是通过管程模型来实现的，在管程模型中，我们提到了两个队列：`入口等待队列`和`条件变量等待队列`。在AQS中`同步队列`对应管程模型中的`入口等待队列`，`条件等待队列`对应管程模型中的`条件变量等待队列`。关于AQS的同步队列的设计原理及源码实现可以阅读这两篇文章：[队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw) 和 [队列同步器（AQS）源码分析](https://mp.weixin.qq.com/s/xIZlHydPxLmnClNckqKrZw)。今天将详细分析AQS中等待队列的设计原理和源码。（关于管程的介绍可以参考这篇文章：[管程:并发编程的基石](https://mp.weixin.qq.com/s/6jvA5jnnMkr5l-IliDL8yw) ）

### 简介
* 在并发领域中需要解决的两个问题：`互斥`与`同步`，互斥指的是同一时刻只允许一个线程访问共享资源，这一点AQS的同步队列已经帮助我们解决了。同步指的是线程间如何进行通信和协作，那么AQS又是如何来解决同步问题的呢？答案就是今天的主角：`Condition`。
* Condition是JUC包下的一个接口，它的一个实现类`ConditionObject`是队列同步器AbstractQueuedSynchronizer（后面简称AQS）的一个内部类。它需要与Lock一起使用，通过`Lock.newCondition()`来创建实例。
* 在Object类中有三个native方法：wait()、notify()、notifyAll()，它们需要与synchronize关键字一起使用，用它来实现线程之间的通信。而Condition提供了`await()、signal()、signalAll()`三个方法，它们分别与Object类的这三个方法对应，也是用来实现线程之前的通信的，但是它们在功能和使用法方式上存在部分差异。具体差异可以参考下表。（表格来源于《Java并发编程的艺术》一书第5章第6节）

|对比项|Object|Condition|
|--|--|--|
|使用前提|使用synchronize获取到锁|使用Lock实例的lock()方法获取到锁|
|使用方式|object.wait()、object.notify()等|需要使用Lock接口的实例对象来创建，Lock.newCondition()|
|等待队列个数|支持1个|可以支持多个，使用Lock实例new多个Condition即可|
|线程进入等待队列后是否响应中断|不支持|支持|
|超时等待|支持|支持|
|线程释放锁后等待到将来的某个时间点|不支持|支持|
|唤醒等待队列中的一个线程|支持，notify()|支持，signal()|
|唤醒等待队列中的所有线程|支持，notifyAll()|支持，signalAll()|

### AQS中如何设计等待对列

#### 数据结构
* AQS中的同步队列的数据结构是Node元素形成的一个双向链表，同样，等待队列也是以Node元素形成的一个链表，只不过是单向链表。在 [队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw) 这篇文章中已经介绍过Node的属性了，今天再来复习一下它的属性以及用途。

|属性名|作用|
|--|--|
|Node prev|同步队列中，当前节点的前一个节点，如果当前节点是同步队列的头结点，那么prev属性为null|
|Node next|同步队列中，当前节点的后一个节点，如果当前节点是同步队列的尾结点，那么next属性为null|
|Node thread|当前节点代表的线程，如果当前线程获取到了锁，那么当前线程所代表的节点一定处于同步队列的队首，且thread属性为null，至于为什么要将其设置为null，这是AQS特意设计的。|
|int waitStatus|当前线程的等待状态，有5种取值。0表示初始值，1表示线程被取消，-1表示当前线程处于等待状态，-2表示节点处于等待队列中，-3表示下一次共享式同步状态获取将会无条件地被传播下去|
|Node nextWaiter|等待队列中，该节点的下一个节点|

* AQS`等待队列`的实现主要依靠的是Node节点中`nextWaiter`属性和`waitStatus`属性来实现的，其中`waitStatus=-2`时，表示`线程是处于等待队列中`。（同步队列依靠prev和next属性来实现双向链表）。Condition包含两个属性：`firstWaiter`和`lastWaiter`，分别表示等待队列的队首和队尾。等待队列遵循先进先出的原则（FIFO）。下图为Condition等待队列的示意图。
【图】

#### 实现原理
* synchronize实现锁的原理也是通过管程模型实现的，但是synchronize实现的锁，只支持一个等待队列，而Condition可以支持多个等待队列。在AQS中，等待队列和同步队列的示意图如下。
【图】

* 当线程通过Lock获取锁以后，调用Condition的await()方法时，当前线程会先将自己封装成一个Node节点，然后将这个节点`添加到等待队列`中；然后释放锁；最后调用`LockSopport.park()`方法将自己挂起。可以使用如下示意图表示。
【图】

* 当调用Condition的signal()方法时，首先会移除等待队列的`firstWaiter`节点，然后将`firstWaiter`节点加入到`同步队列`中。示意图如下。
【图】

* 如果是调用Condition的signalAll()方法，那么就会将Condition等待队列中`所有Node节点移到同步队列中`。

### 源码分析
> 理解了上面的等待队列的数据结构和实现原理，接下来就结合源码看看具体的实现。接下来将分析await()方法和signal()方法的源码。

#### await()方法
> 当调用condition.await()方法时，会调用到AbstractQueuedSynchornizer中的内部类ConditionObject的await()方法。该方法的源码如下。
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程加入到等待队列中
    Node node = addConditionWaiter();
    // 完全释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 判断同步节点是否在同步队列中，
    // 如果在当前线程释放锁后，锁被其他线程抢到了，此时当前节点就不在同步队列中了。
    // 如果锁没有被抢占到，节点就会再同步队列中(当前线程是持有锁的线程，所以它是头结点)
    while (!isOnSyncQueue(node)) {
        // 如果节点不在同步队列中，将当前线程park
        LockSupport.park(this);
        /**
         * 当被唤醒以后，接着从下面开始执行。醒来后会判断自己在等待过程中有没有被中断过。
         * checkInterruptWhileWaiting()方法返回0表示没有被中断过
         * 返回-1表示需要抛出异常
         * 返回1表示需要重置中断标识
         */
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 如果线程在同步队列中，那么就尝试去获取锁，如果获取不到，就会加入到同步队列中，并阻塞
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 当条件等待队列中还有其他线程在等待时，需要判断条件等待队列中有没有线程被取消，如果有，则将它们清除
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
* 在await()方法中，当前线程会先调用`addConditionWaiter()`方法，将自己封装成一个Node后，加入到等待队列中。然后调用`fullyRelease()`方法释放锁，接着通过判断当前线程是否在等待队列中，如果不在等待队列中，就将自己`park`。
* 在addConditionWaiter()方法中，主要干了两件事。一：将等待队列中处于取消状态的节点删除，即waitStatus=1的Node节点；二：将自己加入到等待队列，令Condition的`lastWaiter`属性等于当前线程所代表的节点。（注意：此时如果等待队列没有进行初始化时，会先进行初始化）。addConditionWaiter()方法的源码如下。
```java
/**
 * Adds a new waiter to wait queue.
 * @return its new wait node
 * 将当前线程加入到等待队列中
 */
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 创建一个节点，节点的waitStatus等于-2
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 如果等待队列还没有初始化，即等跌队列中还没有任何元素，那么此时firstWaiter和lastWaiter均为null
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node; // 令旧的队尾节点nextWaiter属性等于当前线程的节点，这样就维护了队列的前后关系
    lastWaiter = node;
    return node;
}
```
* 对于`fullyRelease()`方法，从方法名看，就知道它的作用是`完全释放锁`。这里为什么要完全释放锁呢？因为对于重入锁而言，`锁可能被重入了多次，此时同步变量state的值大于1`，而调用await()方法时，要让当前线程将锁释放掉，所以需要将state的值减为0，因此这里取名为fullyRelease()。fullyRelease()最终还是调用AQS的`release()`方法来释放锁。fullyRelease()方法的源码如下。
```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取同步变量state的值
        int savedState = getState();
        // 释放锁，注意此时将同步变量的值传入进去了，如果是重入锁，且被重入过，那么此时savedState的值大于1
        // 此时释放锁时，会将同步变量state的值减为0。（通常可重入锁在释放锁时，每次只会将state减1，重入了几次就要释放几次，在这里是一下子全部释放）。
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            // 当线程没有获取到锁时，调用该方法会释放失败，会抛出异常。
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```
* 在await()方法中，当释放完锁以后，紧接着通过`isOnSyncQueue()`判断当前线程的`节点是不是在同步队列`中，如果isOnSyncQueue()方法返回true，表示节点在同步队列中，如果返回false，表示当前节点不在同步队列中。当返回false时，`会进入到while循环中`，此时会调用`LockSupport.park()`方法，让当前线程挂起。由于当前线程在释放锁以后会唤醒同步队列中的线程去抢锁，如果有线程抢到锁，那么当前线程就肯定不会在同步队列了（同步队列的首节点变化了），所以此时isOnSyncQueue()方法大概率返回的是false，因此会进入到while()方法中。如果当前线程在同步队列中，或者从park()处醒来后（醒来后会出现在同步队列中，因为signal()或者signalAll()方法会将节点移到同步队列），就会执行到await()方法后面的逻辑，即`acquireQueued()`方法，该方法就是去尝试获取锁，如果获取到就会返回，获取不到就阻塞。
* `isOnSyncQueue()`的源码以及注释如下。
```java
final boolean isOnSyncQueue(Node node) {
    // 如果节点的waitStatus=-2时，节点肯定不在同步队列中，因为只有在等待队列时，才会为-2。
    // 如果节点的前驱节点为空时，有两种情况：
    // 1. 当前节点是同步队列的首节点，首节点是已经获取到锁的线程，可以认为线程不在同步队列中
    // 2. 当前节点在等待队列中，等待队列中的节点在创建时，没有给prev属性赋值
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 在上一个判断的前提条件下，如果后置节点不为空，那么当前节点肯定在同步队列中。
    if (node.next != null) // If has successor, it must be on queue
        return true;
    /*
     * node.prev can be non-null, but not yet on queue because
     * the CAS to place it on queue can fail. So we have to
     * traverse from tail to make sure it actually made it.  It
     * will always be near the tail in calls to this method, and
     * unless the CAS failed (which is unlikely), it will be
     * there, so we hardly ever traverse much.
     */
    // 以上情况均不属于，那么就从同步队列的尾部开始遍历，找到同步队列中是否含有node节点
    return findNodeFromTail(node);
}
```
* 关于await()方法，总结起来就是三个步骤：`1.` 将自己加到等待队列；`2.` 释放锁；`3.` 将自己park()。在Condition接口中还提供了几个重载的await()方法，它们在await()方法的基础上添加了部分功能，例如超时等待、不响应中断的等待等功能，但大致逻辑和await()方法类似，有兴趣的朋友可以去研究下。

#### signal()方法

* 当调用`condition.signal()`方法时，会调用到AbstractQueuedSynchornizer中的内部类`ConditionObject的signal()方法`。该方法的源码如下。
```java
public final void signal() {
    // 先判断当前线程有没有获取到锁，如果没有获取到锁就来调用signal()方法，就会抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```
* 在signal()方法中会判断当期线程是否是锁的拥有者，如果不是，就抛出异常。这就是为什么说调用await()和signal()、signalAll()方法的前提条件是：当前线程获取到了锁，因为这几个方法在执行具体逻辑之前都判断了当前线程是否等于持有锁的线程。从代码中可以看到，signal()方法的具体逻辑是在`doSignal()`方法中实现的。doSignal()方法的源码如下。
```java
private void doSignal(Node first) {
    do {
        // 令firstWaiter等于条件等待队列中的下一个节点。
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
        /**
         *调用transferForSignal()方法，是将节点从条件等待队列中移到同步队列中.
         * 当transferForSignal()返回true时，表示节点被成功移到同步队列了。返回false，表示移动失败，当节点所表示的线程被取消时，会返回false
         * 当transferForSignal()返回true时，do...while循环结束。返回false时，继续。为什么要这样呢？
         * 因为当transferForSignal()返回false表示条件等待队列中的，队列的头结点的状态时取消状态，不能将它移到同步队列中，随意需要继续从条件等待队列找没有被取消的节点。
         */
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```
* doSignal()方法的作用就是将等待队列的头节点`从等待队列中移除`，然后调用`transferForSignal()`方法将其`加入到同步队列`中，当transferForSignal()返回true时，表示节点被成功移到了同步队列中，返回false表示移动失败，只有一种情况会移动失败，那就是线程被取消了。transferForSignal()方法的源码如下。
```java
/**
 * Transfers a node from a condition queue onto sync queue.
 * Returns true if successful.
 * @param node the node
 * @return true if successfully transferred (else the node was
 * cancelled before signal)
 * 将节点从条件等待队列移到同步队列
 */
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    // 将节点的waitStatus的值从-2改为-1。这里如果出现CAS失败，说明节点的waitStatus值被修改过，在条件等待队列中，只有当线程被取消后，才会去修改waitStatus
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    // 当加入到同步队列后，需要将当前节点的前一个节点的waitStatus设置为-1，表示队列中还有线程在等待
    // 如果前驱节点的waitStatus大于0表示线程被取消，需要将当前线程唤醒
    // 或者修改前驱节点的waitStatus是失败，也需要去唤醒当前线程
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
* signal()方法总结：将等待队列的头节点移动到同步队列中。细心的朋友可能会发现，调用signal()方法，并不会让当前线程释放锁，所以当线程调用完signal()方法后，当前线程会将自己的业务逻辑执行完，直到在当前线程中调用了lock.unlock()方法，才会释放锁。

#### signalAll()方法
* `signalAll()`方法的作用就是`唤醒等待队列中的所有节点`，而signal()方法只唤醒等待队列的第一个节点。当调用condition.signalAll()方法时，会调用到AbstractQueuedSynchornizer中的内部类ConditionObject的signalAll()方法。该方法的源码如下。
```java
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        // 核心逻辑在doSignalAll()方法
        doSignalAll(first);
}
```
* 在signalAll()方法中会调用`doSignalAll()`方法。doSignalAll()方法的源码如下。
```java
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    // 通过do...while循环，遍历等待队列的所有节点
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        // transferForSignal()方法将节点移到到同步队列
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```
* 可以发现，doSignalAll()方法通过`do...while`循环，遍历等待队列的所有节点，循环调用`transferForSignal()`方法，将`等待队列中的节点全部移动到同步队列`中。

### 总结
* 本文主要介绍了AQS中等待队列的功能，对比了它与Object中对应方法的异同点。然后通过分析等待队列的数据结构，并结合示意图展示了await()方法和signal()方法的原理。最后通过具体的源码实现，详细分析了await()、signal()、signalAll()这三个方法。
* Condition使用起来十分方便，开发人员只需要通过`Lock.newCondition()`方法就能创建Condition实例，`多次调用`Lock.newCondition()方法，那么就会创建`多个条件等待队列`。
* Condition的应用十分广泛，我们经常接触的有界队列`LinkedBlockingQueue`就是通过Condition来实现的，有兴趣的朋友可以先去阅读下源码。

### 相关推荐
* [管程:并发编程的基石](https://mp.weixin.qq.com/s/6jvA5jnnMkr5l-IliDL8yw)
* [初识CAS的实现原理](https://mp.weixin.qq.com/s/oe046IRUbYeXpIpYLz19ew)
* [Unsafe类的源码解读以及使用场景](https://mp.weixin.qq.com/s/V8Rc3OlcI6D66ggCP2scRQ)
* [队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw)
* [队列同步器（AQS）源码分析](https://mp.weixin.qq.com/s/xIZlHydPxLmnClNckqKrZw)
* [可重入锁（ReentrantLock）源码分析](https://mp.weixin.qq.com/s/VHCVoBn3KBt95VMgT1wQDA)
* [公平锁与非公平锁的对比](https://mp.weixin.qq.com/s/hYLpsP_9Oxc9pSx3AGW5TQ)
* [通过源码看Bean的创建过程](https://mp.weixin.qq.com/s/WwjicbYtcjRNDgj2bRuOoQ)
* [Spring源码系列之容器启动流程](https://mp.weixin.qq.com/s/q6zs7xRjpcB4YxLw6w477w)
* [Spring中最！最！最！重要的后置处理器！没有之一！！！](https://mp.weixin.qq.com/s/f2vSH9YNmnNqdps05LEEHw)
* [@Import和@EnableXXX](https://mp.weixin.qq.com/s/y_2Z9m0gevp-cMkEIflrwA)
* [手写一个Redis和Spring整合的插件](https://mp.weixin.qq.com/s/AU0QpzD0xNslgeWEJ6ujQg)
* [为什么JDK的动态代理要基于接口实现而不能基于继承实现？](https://mp.weixin.qq.com/s/vLnjd80q9q1SNZy6yvzqYw)
* [FactoryBean——Spring的扩展点之一](https://mp.weixin.qq.com/s/NewVzdhA_BNq-LtOahxSAQ)
* [@Autowired注解的实现原理](https://mp.weixin.qq.com/s/qNuGgzPiOha0e1tCW46e8Q)
* [一次策略设计模式的实际应用](https://mp.weixin.qq.com/s/DOBnL1q6UMpWrcrKVvOkdw)
* [Spring如何解决循环依赖](https://mp.weixin.qq.com/s/W-GO189Shn0U78j7XHNxHQ)


