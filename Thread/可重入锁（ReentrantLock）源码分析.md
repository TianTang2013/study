> 点击上方`菜鸟飞呀飞`或者扫描下方二维码，即可关注微信公众号。

【图】

### 问题
> 在阅读本文之前可以先思考一下以下两个问题

* `1.` ReentrantLock是如何在Java层面（非JVM层面）实现锁的？
* `2.` 什么是公平锁？什么是非公平锁？

### 简介
* `Lock`是JUC包下的一个接口，里面定义了获取锁、释放锁等和锁相关的方法，`ReentrantLock`是Lock接口的一个具体实现类，它的功能是可重入地独占式地获取锁。这里有两个概念，`可重入`和`独占式`。可重入表示的是同一个线程能多次获取到锁。独占式表示的是，同一时刻只能有一个线程获取到锁。
* `ReentrantLock`实现的锁又可以分为两类，分别是`公平锁`和`非公平锁`，分别由ReentrantLock类中的两个内部类`FairSync`和`NonfairSync`来实现。FiarSync和NonfairSync均继承了Sync类，而Sync类又继承了AbstractQueuedSynchronizer（AQS）类，所以ReentrantLock最终是依靠AQS来实现锁的。关于AQS的知识可以参考以下两篇文章。
* [队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw)  
* [队列同步器（AQS）源码分析](https://mp.weixin.qq.com/s/xIZlHydPxLmnClNckqKrZw)
* ReentrantLock有两个构造方法，一个无参构造方法，一个有参构造方法。通过无参构造方法创建对象时，创建的是非公平锁；有参构造方法中，可以传入一个boolean类型的变量，如果传入的是true，那么创建的是公平锁，如果传入的是false，那么创建的是非公平锁。
```java
/**
 * Creates an instance of {@code ReentrantLock}.
 * This is equivalent to using {@code ReentrantLock(false)}.
 */
public ReentrantLock() {
    sync = new NonfairSync();
}

/**
 * Creates an instance of {@code ReentrantLock} with the
 * given fairness policy.
 *
 * @param fair {@code true} if this lock should use a fair ordering policy
 */
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

### 公平锁
* 当向ReentrantLock的构造方法中传入true时，创建的是公平锁。公平锁的`lock()`和`unlock()`方法实现是由FairSync类中的`lock()`和`release()`方法实现的（其中release()方法定义在AQS中）。
```java
public static void main(String[] args) {
    ReentrantLock lock = new ReentrantLock(true);
    try{
        lock.lock();
        System.out.println("Hello World");
    }finally {
        lock.unlock();
    }
}
```
#### 获取锁
* 当调用lock.lock()时，会调用到FairSync.lock()方法，FairSync.lock()方法会调用AQS中的acquire()方法。
```java
static final class FairSync extends Sync {
    final void lock() {
        // 调用acquire()获取同步状态
        acquire(1);
    }
}
```
* 在AQS的acquire()方法中会先调用子类的tryAcquire()方法，此时由于我们创建的是公平锁，所以会调用FairSync类中的tryAcquire()方法。（关于acquire()方法的详细介绍，可以参考：[队列同步器（AQS）源码分析](https://mp.weixin.qq.com/s/xIZlHydPxLmnClNckqKrZw)）。

* tryAcquire()方法的作用就是获取同步状态（也就是获取锁），如果当前线程成功获取到锁，那么就会将AQS中的同步状态state加1，然后返回true，如果没有获取到锁，将会返回false。FairSync类上tryAcquire()方法的源码如下。
```java
protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        // 获取同步状态state的值（在AQS中，state就相当于锁，如果线程能成功修改state的值，那么就表示该线程获取到了锁）
        int c = getState();
        if (c == 0) {
            // 如果c等于0，表示还没有任何线程获取到锁

            /**
             * 此时可能存在多个线程同时执行到这儿，均满足c==0这个条件。
             * 在if条件中，会先调用hasQueuedPredecessors()方法来判断队列中是否已经有线程在排队，该方法返回true表示有线程在排队，返回false表示没有线程在排队
             * 第1种情况：hasQueuedPredecessors()返回true，表示有线程排队，
             *           此时 !hasQueuedPredecessors() == false，由于&& 运算符的短路作用，if的条件判断为false，那么就不会进入到if语句中，tryAcquire()方法就会返回false
             *
             * 第2种情况：hasQueuedPredecessors()返回false，表示没有线程排队
             *           此时 !hasQueuedPredecessors() == true, 那么就会进行&&后面的判断，就会调用compareAndSetState()方法去进行修改state字段的值
             *           compareAndSetState()方法是一个CAS方法,它会对state字段进行修改，它的返回值结果又需要分两种情况
             *           第 i 种情况：对state字段进行CAS修改成功，就会返回true，此时if的条件判断就为true了，就会进入到if语句中，同时也表示当前线程获取到了锁。那么最终tryAcquire()方法会返回true
             *           第 ii 种情况：如果对state字段进行CAS修改失败，说明在这一瞬间，已经有其他线程获取到了锁，那么if的条件判断就为false了，就不会进入到if语句块中，最终tryAcquire()方法会返回false。
             */
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                // 当前线程成功修改了state字段的值，那么就表示当前线程获取到了锁，那么就将AQS中锁的拥有者设置为当前线程，然后返回true。
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果c等于0，则表示已经有线程获取到了锁，那么这个时候，就需要判断获取到锁的线程是不是当前线程
        else if (current == getExclusiveOwnerThread()) {
            // 如果是当前线程，那么就将state的值加1，这就是锁的重入
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            // 因为此时肯定只有一个线程获取到了锁，只有获取到锁的线程才会执行到这行代码，所以可以直接调用setState(nextc)方法来修改state的值，这儿不会存在线程安全的问题。
            setState(nextc);
            // 然后返回true，表示当前线程获取到了锁
            return true;
        }
        // 如果state不等于0，且当前线程也等于已经获取到锁的线程，那么就返回false，表示当前线程没有获取到锁
        return false;
    }
}
```
* 在tryAcquire()方法中，先判断了同步状态state是不是等于0。
* `1.` 如果等于0，就表示目前还没有线程持有到锁，那么这个时候就会先调用`hasQueuedPredecessors()`方法判断同步队列中有没有等待获取锁的线程，如果有线程在排队，那么当前线程肯定获取锁失败（因为AQS的设计的原则是FIFO，既然前面有人已经在排队了，你就不能插队，老老实实去后面排队去），那么tryAcquire()方法会返回false。如果同步队列中没有线程排队，那么就让当前线程对state进行CAS操作，如果设置成功，就表示当前获取到锁了，返回true；如果CAS失败，表示在这一瞬间，锁被其他线程抢走了，那么当前线程就获取锁失败，就返回false。
* `2.` 如果不等于0，就表示已经有线程获取到锁了，那么此时就会去判断当前线程是不是等于已经持有锁的线程（`getExclusiveOwnerThread()方法的作用就是返回已经持有锁的线程`），如果相等，就让state加1，这就是所谓的重入锁，重入锁就是允许同一个线程多次获取到锁。
* `3.` state既不等于0，当前线程也不等于持有锁的线程，那么就返回false，表示当前线程获取到锁失败。
* 就这样，通过FairSync类的tryAcquire()方法，就实现了公平锁获取锁的逻辑。在不同的锁的实现中，tryAcquire()方法的逻辑是不一样的，例如在非公平锁中，NonfairSync类的tryAcquire()中，代码逻辑和FairSync类的tryAcquire()方法就不一样，在非公平锁的实现中，当线程尝试对state进行CAS操作之前，没有对同步队列中有没有线程在排队进行判断（即没有调用hasQueuedPredecessors()方法）。
* hasQueuedPredecessors()方法的作用是判断同步队列中有没有线程在排队，如果有，就返回true，如果没有，就返回false。其源码如下，在贴出的代码片段中，对该方法进行了详细的解释（要看懂该方法，需要对AQS的设计原理有一定的了解，可以阅读这一篇文章：[队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw)）。
```java
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;

    /**
     * 该方法的作用是判断队列中是否有线程在排队，返回true表示有线程排队，返回false表示没有线程排队
     *
     * 首先判断 h!=t，即判断头结点和尾结点是否相等，头结点和尾结点相等只有两种情况，
     *               第一：队列还没有被初始化，此时head = null, tail = null，此时线程不需要排队。
     *               第二：已经有一个线程获取到了锁，当第二个线程进来获取锁时，因为获取不到锁，所以会被需要入队，入队之前它需要先初始化队列，
     *                    在初始化时，在enq()方法中，第一步是先将tail = head，然后第二步再将当前线程代表的Node设置为tail。所以在这两步操作
     *                    的中间的一瞬间，是存在tail = head这个可能性的。此时对所有线程而言是也不需要排队
     * 当 h!=t 的结果为true时，接下来会判断((s = h.next) == null || s.thread != Thread.currentThread())，先判断(s = h.next) == null
     *
     * 判断(s = h.next) == null 时，先令 s = h.next ,表示获取到头结点的的下一个节点，即第二个节点，如果 s == null 则表示第二个节点为null，由于前面已经判断了h!=t，
     *                             说明此时已经有第二个线程进入到了队列中，只不过它还没来及将head节点的next指针的指向修改，所以此时线程线程需要排队，
     *                             因为||运算的短路原因，当(s = h.next) == null 的结果为true时，就不会进入到后面的判断了，而此时hasQueuedPredecessors()会返回true，表示线程需要排队。
     * 当s不为null时，会进行s.thread != Thread.currentThread() 的判断
     *                              它会判断s节点中的线程是否不等于当前线程，如果不等于当前线程，hasQueuedPredecessors()会返回true，说明当前线程需要排队。
     *                              因为当第二个节点不是当前线程，那么就说明当前线程应该至少是排在队列中的第三位，那么它需要排队。
     *                              如果s节点中的线程等于当前线程，那么说明当前线程是排在队列中的第二位(第一位是已经获取到锁的线程)，此时线程是不需要排队的，
     *                              因为可能在这一瞬间已经获取到锁的线程释放了锁，那么排在队列中第二位的线程还排啥子队哦，直接去尝试获取锁即可。
     *
     *
     *
     */

    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

#### 释放锁
* 当调用lock.unlock()方法释放锁时，会调用到AQS的release()方法，如果释放锁成功，就会将AQS中的state变量的值减为0。在release()方法中，会先调用到tryRelease()方法，然后还会调用unparkSuccessor()方法来唤醒同步队列中在等待获取锁的一个线程。tryRelease()方法的逻辑是由子类实现的，不同锁有不同的实现。但是对于ReentrantLock而言，公平锁和非公平锁的释放锁的逻辑是一样的，均是调用Sync类的tryRelease()方法。
* AQS的release()方法源码如下。
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        // 当释放锁成功以后，需要唤醒同步队列中的其他线程
        Node h = head;
        // 当waitStatus!=0时，表示同步队列中还有其他线程在等待获取锁
        if (h != null && h.waitStatus != 0)
            // 唤醒同步队列中下一个能尝试获取锁的线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
* Sync类的tryRelease()方法的源码和代码注释如下。
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    // 判断当前线程是不是持有锁的线程，如果不是就抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 因为上面一步已经确认了是当前线程持有锁，所以在修改state时，肯定是线程安全的
    boolean free = false;
    // 因为可能锁被重入了，重入了几次就需要释放几次锁，所以这个地方需要判断，只有当state=0时才表示完全释放了锁。
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
* tryRelease()方法会返回有没有释放锁成功的标识，当表示同步状态的变量state被减为0了，tryRelease()方法才会返回true，否则返回false，如果锁被重入了多次，就需要多次调用tryRelease()方法。当tryRelease()执行完以后，会回到AQS中的release()方法中，如果tryRelease()方法返回true，表示锁被释放了，这个时候如果同步队列中有线程在排队，那么就去调用unparkSuccessor()方法去唤醒同步队列中下一个离头结点最近的且有资格获取锁的线程。为什么是离头结点最近呢？因为AQS要保证FIFO。为什么是有资格呢？因为同步队列中有的线程的状态是被取消的，即Node节点的waitStatus=1，此时这种线程是不能被唤醒去抢锁的。另外，这里只是去唤醒线程去尝试获取锁，不保证能获取到。唤醒线程使用的方法是`LockSupport.unpark()`。
* unparkSuccessor()方法的源代码和注释如下。
```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        /**
         * 同步队列中的节点锁代表的线程，可能被取消了，此时这个节点的waitStatus=1
         * 因此这个时候利用for循环从同步队列尾部开始向前遍历，判断节点是不是被取消了
         * 正常情况下，当头结点释放锁以后，应该唤醒同步队列中的第二个节点，但是如果第二个节点的线程被取消了，此时就不能唤醒它了,
         * 就应该判断第三个节点，如果第三个节点也被取消了，再依次往后判断，直到第一次出现没有被取消的节点。如果都被取消了，此时s==null，所以不会唤醒任何线程
         */
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

### 非公平锁
* 当使用ReentrantLock的无参构造器或者有参构造器中传入false时，创建的是非公平锁。非公平锁的具体实现是由NonfairSync类来实现的，非公平锁的获取逻辑与公平锁的获取逻辑存在一点差异，锁的释放两者完全一样。

#### 获取锁
* 非公平锁的获取会先调用NonfairSync类的lock()方法。非公平锁不会去判断同步队列中有没有人排队，而是先直接去尝试修改state变量的值为1，如果修改成功，就表示线程获取到了锁，然后lock()方法结束。如果修改失败，就再去调用AQS的acquire()方法去尝试获取锁。
```java
static final class NonfairSync extends Sync {
    
    final void lock() {
        // 先尝试获取锁，如果获取锁失败，就再去调用AQS的acquire()方法
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
}
```
* 当调用到AQS的acquire()方法时，在acquire()方法中会调用AQS子类的tryAcquire()方法，此时调用的是NonfairSync类的tryAcquire()方法。源码如下。
```java
protected final boolean tryAcquire(int acquires) {
    // 调用nonfairTryAcquire()方法
    return nonfairTryAcquire(acquires);
}
```
* NonfairSync的tryAcquire()方法会调用nonfairTryAcquire()方法。
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 与公平锁的不同之处在于，公平在在进行CAS操作之前，会先判断同步队列中是否有人排队。
        // !hasQueuedPredecessors()
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
* 可以看到nonfairTryAcquire()方法的逻辑与FairSync的tryAcquire()方法的逻辑非常相似，唯一的区别就是当`c == 0`时，`非公平锁没有去判断同步队列中是否有人在排队`，而是`直接`调用CAS方法去设置state的值。`而公平锁则是先判断同步队列中是否有人排队`，如果没有人排队，才调用CAS方法去设置state的值。即非公平锁没有调用`hasQueuedPredecessors()`方法。其他的逻辑，公平锁与非公平锁的逻辑就一样了。

#### 释放锁
* 非公平锁的释放逻辑与公平锁的释放逻辑一样，最终都是调用Sync类的tryRelease()方法。


### 总结
* 本文主要介绍了如何通过`ReentrantLock`来创建公平锁和非公平锁，通过默认的无参构造方法创建的是非公平锁；当使用有参构造器创建时，如果参数传入的是false，则创建的是非公平锁，如果传入的是true，则创建的是公平锁。
* 实现公平锁的同步组件的类是`FairSync`，实现非公平锁的同步组件的类是`NonfairSync`，它们都是队列同步器AQS的子类。
* 通过分别分析FairSync和NonfairSync类的`tryAcquire()`方法，分析了公平锁和非公平锁的获取锁的流程。同时对比两者的实现代码，发现非公平锁在获取锁时不会判断同步队列中有没有线程在排队，而是直接尝试去修改state的值，而公平锁在获取锁时，会先判断同步队列中有没有线程在排队，如果没有才会尝试去修改state的值。
* 公平锁和非公锁在释放锁时的逻辑是一样的。
* 最后回答一下文章开头的两个问题。
* `1.` ReentrantLock是如何在Java层面（非JVM实现）实现锁的？
* ReentrantLock类通过`组合一个同步组件Sync来实现锁`，这个同步组件继承了AQS，并重写了AQS中的tryAcquire()、tryRelease()等方法，最终实际上还是通过AQS以及重写的tryAcquire()、tryRelease()等方法来实现锁的逻辑。ReentrantLock实现锁的方式，也是JUC包下其他类型的锁的实现方法，通过组合一个自定义的同步组件，这个同步组件需要继承AQS，然后重写AQS中的部分方法即可实现一把自定义的锁，通常这个同步组件被定义成内部类。
* `2.` 什么是公平锁？什么是非公平锁？
* 由`FairSync`同步组件实现的锁是公平锁，它获取锁的原则是，在同步队列中等待时间最长的线程获取锁，因此称它为公平锁。由`NonfairSync`同步组件实现的锁是非公平锁，它获取锁的原则是，同步队列外的线程在尝试获取锁时，不会判断队列中有没有线程在排队，而是上来就抢，抢到锁了就走，抢不到了才去排队，因此称它为不公平的。

【图】
