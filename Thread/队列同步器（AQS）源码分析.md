> 点击上方`菜鸟飞呀飞`或者扫描下方二维码，即可关注微信公众号。

【图】

[toc]

> 在上一篇文章中分析了队列同步器（AQS）的设计原理，这一篇文章将结合具体的源代码来分析AQS。如果理解了上一篇文章中介绍的同步队列的数据结构，那么源码看起来相对比较好理解。读过Spring源码的朋友应该能感受到Spring中方法和变量的命名是非常规范的，见名知意。与Spring相比，JDK中某些类的源码，命名则不是那么规范，里面存在各种简写，单个字母命名变量，有时候看起来十分烧脑。其中AQS中就存在这种情况，所以如果不了解AQS的设计原理，直接看源码，就会显得比较困难。

* AQS中获取锁和释放锁的方法很多，今天以acquire()和release()为例来进行源码分析，其他方法的逻辑与这两个方法逻辑基本一样。

```java
public static void main(String[] args) {
    // 公平锁
    ReentrantLock lock = new ReentrantLock(true);
    try{
        // 加锁
        lock.lock();
        System.out.println("Hello World");
    }finally {
        // 释放锁
        lock.unlock();
    }
}
```

* 以上面demo为例，先创建了一个公平锁，然后调用lock()方法来加锁，最后调用unlock()方法释放锁。在lock()方法中会调用到Sync的lock()方法，由于我们创建的是公平锁，所以这个时候调用的是FailSync的lock()方法。
```java
static final class FairSync extends Sync {

    final void lock() {
        // 调用acquire()获取同步状态
        acquire(1);
    }
}
```

## 获取锁：acquire()
* 在FailSync的lock()方法中就会调用到AQS的模板方法：`acquire()`。最终的加锁逻辑就是在acquire()方法中实现的。`acquire()`的源码如下：
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 调用selfInterrupt()方法重新设置中断标识
        selfInterrupt();
}
```
* 这个方法就3行代码，但是逻辑相当复杂。可以拆分为四段逻辑：第一步：调用tryAcquire()方法；第二步：执行addWaiter()方法；第三步：执行acquireQueued()方法；第四步：执行selfInterrupt()方法。下面将围绕这4步展开分析。我在`acquire()`这个方法上简单写了些注释，也可以直接参考。
```java
public final void acquire(int arg) {
    /*
     * acquire()的作用是获取同步状态，这里同步状态的含义等价于锁
     * tryAcquire()方法是尝试获取同步状态，如果该方法返回true，表示获取同步状态成功；返回false表示获取同步状态失败
     * 第1种情况：当tryAcquire()返回true时，acquire()方法会直接结束。
     *           因为此时!tryAcquire(arg) 就等于false，而&&判断具有短路作用，当因为&&判断前面的条件判断为false，&&后面的判断就不会进行了，所以此时就不会执行后面的if语句，此时方法就直接返回了。
     *           这种情况表示线程获取锁成功了
     * 第2中情况：当tryAcquire()返回false时，表示线程没有获取到锁，
     *           这个时候就需要将线程加入到同步队列中了。此时 !tryAcquire() == true，因此会进行&&后面的判断，即acquireQueued()方法的判断，在进行acquireQueued()方法判断之前会先执行addWaiter()方法。
     *
     *           addWaiter()方法返回的是当前线程代表的节点，这个方法的作用是将当前线程放入到同步队列中。
     *           然后再调用acquireQueued()方法，在这个方法中，会先判断当前线程代表的节点是不是第二个节点，如果是就会尝试获取锁，如果获取不到锁，线程就会被阻塞；如果获取到锁，就会返回。
     *           如果当前线程代表的节点不是第二个节点，那么就会直接阻塞，只有当获取到锁后，acquireQueued()方法才会返回
     *
     *           acquireQueued()方法如果返回的是true，表示线程是被中断后醒来的，此时if的条件判断成功，就会执行selfInterrupt()方法，该方法的作用就是将当前线程的中断标识位设置为中断状态。
     *           如果acquireQueued()方法返回的是false，表示线程不是被中断后醒来的，是正常唤醒，此时if的条件判断不会成功。acquire()方法执行结束
     *
     * 总结：只有当线程获取到锁时，acquire()方法才会结束；如果线程没有获取到锁，那么它就会一直阻塞在acquireQueued()方法中，那么acquire()方法就一直不结束。
     *
     */

    // tryAcquire()方法是AQS中定义的一个方法，它需要同步组件的具体实现类来重写该方法。因此在公平锁的同步组件FairSync和非公平锁的同步组NonfairSync中，tryAcquire()方法的实现代码逻辑是不一样的。
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 调用selfInterrupt()方法重新设置中断标识
        selfInterrupt();
}
```

### 第一步：tryAcquire()
* tryAcquire()是由AQS的子类重写的方法，所以代码逻辑和具体的锁的实现有关系，由于本次Demo中使用的是公平锁，所以它最终会调用FairSync类的tryAcquire()方法。tryAcquire()方法的作用就是获取同步状态（也就是获取锁），如果当前线程成功获取到锁，那么就会将AQS中的同步状态state加1，然后返回true，如果没有获取到锁，将会返回false。FairSync类上tryAcquire()方法的源码如下：
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
* 就这样，通过tryAcquire()方法，就实现了线程能否获取到锁的功能和逻辑。在不同的锁的实现中，tryAcquire()方法的逻辑是不一样的，上面的示例是以公平锁的实现举例的。

### 第二步：addWaiter()
> &emsp;&emsp;addWaiter()方法不一定会执行到，它依赖第一步tryAcquire()的执行结果。如果线程获取锁成功，那么tryAcquire()方法就会返回true，此时就不会进行后面的第二步、第三步、第四步了。只有当线程获取锁失败了，tryAcquire()方法会返回false，这个时候需要将线程加入到同步队列中，所以需要执行后面的步骤。

* addWaiter()方法的逻辑相对比较简单，它的作用就是将当前线程封装成一个Node，然后加入到同步队列中。如果同步队列此时还没有初始化（也就是AQS的head、tail属性均为null），那么它还会进行同步队列的初始化。源码如下：
```java
private Node addWaiter(Node mode) {
    // 先根据当前线程，构建一个Node节点
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 然后需要把当前线程代表的节点添加到队列的尾部，此时队尾可能是null，因为可能这是第一次出现多个线程争抢锁，队列还没有初始化，即头尾均为null
    Node pred = tail;
    if (pred != null) {
        // 如果队尾不为null,就直接将当前线程代表的node添加到队尾，修改node节点中prev的指针指向
        node.prev = pred;
        // 利用CAS操作设置队尾，如果设置成功，就修改倒数第二个节点的next的指针指向，然后方法return结束
        // 在多个线程并发的情况下，CAS操作可能会失败，如果失败，就会进入到后面的逻辑
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 如果队尾为null或者CAS设置队尾失败，就调用enq()方法将当前线程代表的节点加入到同步队列当中
    enq(node);
    // 将当前线程代表的节点返回
    return node;
}
```
* 在addWaiter()方法中会先判断尾结点是不是null，如果是null，就表示同步队列还没有被初始化，那么就不会进入到if的逻辑中了，将会调用`enq(node)`方法进行同步队列的初始化和当前线程的入队操作。如果尾结点不为null，就表示同步队列已经被初始化了，只需要将当前线程所代表的节点加入到同步队列队尾即可。由于加入到队尾这个操作存在并发的可能性，所以需要利用CAS操作（即`compareAndSetTail()`方法）来保证原子性，如果CAS成功，就将旧的队尾节点的next指针指向新的队尾节点，这样就完成了节点加入到队尾的操作了，同时addWaiter()方法return结束。如果CAS操作失败，那么就会调用`enq(node)`方法来实现入队操作。
* `enq(node)`方法的源码如下：
```java
private Node enq(final Node node) {
    // 无限循环
    for (;;) {
        Node t = tail;
        // 如果尾结点为null，说明这是第一次出现多个线程同时争抢锁，因此同步队列没有被初始化，tail和head此时均为null
        if (t == null) { // Must initialize
            // 设置head节点，注意此时是new了一个Node节点，节点的next,prev,thread属性均为null
            // 这是因为对于头结点而言，作为队列的头部，它的next指向本来就应该是null
            // 而头结点的thread属性为空，这是AQS同步器特意为之的
            // 头结点的prev属性此时为空，当进行第二次for循环的时候，tail结点就不为空了，因此不会进入到if语句中，而是进入到else语句中，在这里对头结点的prev进行了赋值
            if (compareAndSetHead(new Node()))
                // 第一次初始化队列时，头结点和尾结点是同一个结点
                tail = head;
        } else {
            // 队尾结点不为null，说明队列已经进行了初始化，这个时候只需要将当前线程表示的node结点加入到队列的尾部

            // 设置当前线程所代表的节点的prev的指针，使其指向尾结点
            node.prev = t;
            // 利用CAS操作设置队尾结点
            if (compareAndSetTail(t, node)) {
                // 然后修改队列中倒数第二个节点的next指针，使其指向队尾结点
                t.next = node;
                // 然后返回的是队列中倒数第二个节点（也就是旧队列中的尾结点）
                return t;
            }
        }
    }
}
```
* 在`enq(node)`方法中进行了一个for(;;)的无限循环，在循环中会先判断队尾是不是null，是null就表示同步队列没有被初始化，就采用CAS操作（即`compareAndSetHead()`方法）来初始化队列，并让tail = head。注意：在创建头结点时，头结点的thread、prev、next属性均为null，在进行第二次循环时，next属性会被赋值，但是thread、prev属性始终是null。
* 当同步队列初始化完成后，后面进入for循环的时候，就不会进入到if语句块中了，而是进入到else中，此时也是利用CAS操作（即`compareAndSetTail()`方法）将当前线程所代表的Node节点设置为队尾，然后修改之前队尾的next指针，维护好队列节点中的双向关联关系，就将旧的队尾节点返回。
* 在执行完enq()方法后，就会回到addWaiter()方法中，然后addWaiter()方法就会将当前线程所代表的Node节点返回。

### 第三步：acquireQueued()
* 在执行完addWaiter()方法后，就会执行acquireQueued()方法。该方法的作用就是让线程以不间断的方式获取锁，如果获取不到，就会一直阻塞在这个方法里面，直到获取到锁，才会从该方法里面返回。
* acquireQueued()方法的源码如下：
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 无限for循环
        for (;;) {
            // 获取当前线程所代表节点的前一个节点
            final Node p = node.predecessor();
            // 如果前一个节点是头结点（即当前线程是队列中的第二个节点），那么就调用tryAcquire()方法让当前线程去尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 如果当前线程获取锁成功，那么就将当前线程所代表的节点设置为头结点(因为AQS的设计原则就是：队列中占有锁的线程一定是头结点)
                setHead(node);
                // 然后将原来的头节点的next指针设置为null，这样头节点就没有任何引用了，有助于GC回收。
                p.next = null; // help GC
                failed = false;
                // 返回interrupted所代表的值，表示当前线程是不是通过中断而醒来的
                return interrupted;
            }
            // 如果前一个节点不是头结点 或者 前一个节点是头结点，但是当前节点调用tryAcquire()方法获取锁失败，那么就会执行到下面的if判断
            /**
             * 在if判断中，先调用了shouldParkAfterFailedAcquire()方法。
             * 在第一次调用shouldParkAfterFailedAcquire()时，肯定返回false(为什么会返回false，可以看下shouldParkAfterFailedAcquire()方法的源码)
             * 由于当前代码是处于无限for循环当中的，所以当后面出现第二次代码执行到这儿时，会再次调用shouldParkAfterFailedAcquire()方法，此时这个方法会返回true。
             * 当shouldParkAfterFailedAcquire()返回true时，if判断中就会再调用parkAndCheckInterrupt()方法，该方法会将当前线程进行阻塞，直到这个线程被唤醒或者被中断。
             * 因此当线程获取不到锁时，就会一直阻塞到这儿。直到被其他线程唤醒，才会继续向下执行，当线程想来后，再次进入到当前代码的无限for循环中，除非线程获取到锁，才会return返回
             */
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 如果在try中出现异常，那么此时failed就会为true，就会取消获取锁
        // failed的初始值为true，如果try语句中不出现异常，最终线程获取到锁后，就会将failed置为false，那么下面的if判断条件就不会成立，也就不执行cancelAcquire()方法了
        // 如果上面的try语句存在异常，那么就不会将failed设置为false，那么就会执行cancelAcquire()方法
        if (failed)
            cancelAcquire(node);
    }
}
```
* 在acquireQueued()方法中也是使用了无限for循环：for(;;)。在该方法中会先获取到当前线程的前一个节点。
* 1. 如果前一个节点是头结点，那么就会让当前线程调用tryAcquired()方法尝试获取锁。如果获取锁成功，就将当前线程设置为头结点，这里设置头结点时只需要调用setHead(head)方法即可，因为肯定只有一个线程获取到锁，所以不用考虑并发的问题。然后旧的头结点的next引用置为null，便于GC。接着acquireQueued()方法直接返回。如果当前线程调用tryAcquire()方法获取锁失败，就会进入到下面2中的逻辑。
* 2. 如果前一个节点不是头结点或者是头结点但是获取锁失败，就会进入到后面的逻辑中，即执行到`if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())`。
* 在if判断中，会先执行`shouldParkAfterFailedAcquire()`方法，该方法用来判断当前线程是否应该park，即是否应该将当前线程阻塞。如果返回true，就表示需要阻塞，那么就会接着执行parkAndCheckInterrupt()方法，在`parkAndCheckInterrupt()`方法中，就会将当前线程阻塞起来，直到当前线程被唤醒并获取到锁以后，acquireQueued()方法才会返回。parkAndCheckInterrupt()方法的源码如下：
```java
private final boolean parkAndCheckInterrupt() {
    // 将当前线程挂起
    LockSupport.park(this);
    return Thread.interrupted();
}
```
* shouldParkAfterFailedAcquire()方法是一个比较有意思的方法，在第一次调用它的时候，它会返回false，由于acquireQueued()方法里有一个无限for循环，所以会进行第二次调用shouldParkAfterFailedAcquire()方法，这个时候它会返回true。shouldParkAfterFailedAcquire()方法的源码如下：
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 在初始情况下，所有的Node节点的waitStatus均为0，因为在初始化时，waitStatus字段是int类型，我们没有显示给它赋值，所以它默认是0
    // 后面waitStatus字段会被修改

    // 在第一进入到shouldParkAfterFailedAcquire()时，所以waitStatus是默认值0（因为此时没有任何地方对它进行修改),所以ws=0
    int ws = pred.waitStatus;

    if (ws == Node.SIGNAL)
        // 如果ws = -1，就返回true
        // 如果第一次进入shouldParkAfterFailedAcquire()方法时，waitStatus=0,那么就会进入到后面的else语句中
        // 在else语句中，会将waitStatus设置为-1，那么当后面第二次进入到shouldParkAfterFailedAcquire()方法时，就会返回true了
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        // 如果ws既不等于-1，也不大于0，就会进入到当前else语句中
        // 此时会调用CAS方法将pred节点的waitStatus字段的值改为-1
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
* 经过shouldParkAfterFailedAcquire()方法后，队列中所有节点的状态的示意图如下。

【图】

* 最终对于acquireQueued()方法而言，只有线程获取到了锁或者被中断，线程才会从这个方法里面返回，否则它会一直阻塞在里面。

### 第四步：selfInterrupt()
* 当线程从acquireQueued()方法处返回时，返回值有两种情况，如果返回false，表示线程不是被中断才唤醒的，所以此时在acquire()方法中，if判断不成立，就不会执行selfInterrupt()方法，而是直接返回。如果返回true，则表示线程是被中断才唤醒的，由于在parkAndCheckInterrupt()方法中调用了Thread.interrupted()方法，这会将线程的中断标识重置，所以此时需要返回true，使acquire()方法中的if判断成立，然后这样就会调用selfInterrupt()方法，将线程的中断标识重新设置一下。最后acquire()方法返回。

## 释放锁：release()
* 释放锁的代码实现相对比较简单，当持有锁的线程释放锁后，会唤醒同步队列中下一个`处于等待状态`的节点去尝试获取锁。还是以上面的Demo为例，当调用lock.unlock()时，最终会调用到AQS中的release()方法，release()方法的源码如下。
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        // 当释放锁成功以后，需要唤醒同步队列中的其他线程
        Node h = head;
        // 当waitStatus!=0时，表示同步队列中还有其他线程在等待获取锁
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
* 在release()方法中会先调用AQS子类的tryRelease()方法，在本文的示例中，就是调用ReentrantLock类中Sync的tryRelease()方法，该方法就是让当前线程释放锁。方法源码如下。
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
* 在tryRelease()方法中，会先检验当前线程是不是持有锁的线程，如果不是持有锁的线程，就会抛出异常（这一点很容易理解，没有获取到锁，却去调用释放锁的方法，这个时候就告诉你出错了）。然后再判断state减去1后是否等于0，如果等于0，就表示锁被释放了，最终tryRelease()方法就会返回true。如果state减去1后不为0，表示锁还没有被释放，最终tryRelease()方法就会返回false。因为对于重入锁而言，每获取一次锁，state就会加1，获取了几次锁，就应该释放几次，这样当完全释放完时，state才会等于0。
* 当tryRelease()返回true时，在release()方法中，就会进入到if语句块中。在if语句块中，会先判断同步队列中是否有线程在等待获取锁，如果有，就调用`unparkSuccessor()`方法来唤醒下一个处于等待状态的线程；如果没有，tryRelease()就会直接返回true。
* 如何判断同步队列中是否有线程在等待获取锁的呢？是通过`h != null && h.waitStatus != 0`这一个判断条件来判断的。
* 1. 当头节点等于null时，这个时候肯定没有线程在等待获取锁，因为同步队列的初始化是当有线程获取不到锁以后，将自己加入到同步队列的时候才初始化同步队列的（即给head、tail赋值），如果头结点为null，说明同步队列没有被初始化，即没有线程在等待获取锁。
* 2. 在acquire()方法中我们提到了shouldParkAfterFailedAcquire()方法，该方法会让等待获取锁的节点的waitStatus的值等于-1，所以当`h != null && h.waitStatus != 0`时，可以认为同步队列中有线程在等待获取锁。
* 如果同步队列中有线程在等待获取锁，那么此时在release()方法中调用`unparkSuccessor()`方法去唤醒下一个等待状态的节点。unparkSuccessor()方法的源码如下
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
* 在unparkSuccessor()方法中，会找到头结点后`第一个waitStatus小于0`的节点，然后将其唤醒。为什么是`waitStatus小于等于0`？因为waitStatus=1表示线程是取消状态，没资格获取锁。最后调用LockSupport.unpark()方法唤醒符合条件的线程，此时线程被唤醒后，就回到上面`获取锁流程中`的parkAndCheckInterrupt()方法处，接着就执行后面的逻辑了。

## 总结
* 本文通过分析获取锁acquire()和释放锁release()方法的源码，讲解了线程获取锁和入队、出队的流程。对于AQS中其他获取锁和释放锁的方法，如可响应中断的获取锁，支持超时的获取锁以及共享式的获取锁，其方法的源码和逻辑和acquire()方法的逻辑很相似，有兴趣的朋友可以自己去阅读下源码。
* 在本文中提到了公平锁、非公平锁以及重入锁的概念，关于这些会在下一篇文章ReentrantLock的源码分析中详细介绍。




