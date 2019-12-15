> > 扫描下方二维码或者微信搜索公众号`菜鸟飞呀飞`，即可关注微信公众号，阅读更多`Spring源码分析`和`Java并发编程`文章。

【图】

### 简介
在JUC包下提供了很多线程安全的队列，通常称之为阻塞队列。这些阻塞队列在线程池中的应用十分广泛，搞懂阻塞队列的实现原理，对平时使用阻塞队列会有很大帮助。本文将结合源码主要分析下`LinkedBlockingQueue`这个阻塞队列的实现原理。

LinkedBlockingQueue是一个基于链表实现的阻塞队列，默认情况下，该阻塞队列的大小为`Integer.MAX_VALUE`，由于这个数值特别大，因此在很多地方称LinkedBlockingQueue是一个无界队列。在LinkedBlockingQueue进行初始化时，可以手动指定队列的大小，这样LinkedBlockingQueue就是一个有界队列了。

在看具体的源码之前，可以先思考一下，我们自己如何来实现一个LinkedBlockingQueue。

既然说LinkedBlockingQueue是线程安全的，那就要解决互斥和同步的问题，这一点我们可以通过Java中提供的锁来解决。Java中锁分两大类，一类是synchronized实现的隐式锁，另一类是并发编程大师Doug Lea基于AQS实现的锁。由于LinkedBlockingQueue是Doug Lea所编写的类，因此LinkedBlockingQueue底层使用的是AQS类型的锁，即：ReentrantLock。

对于队列而言，有两类操作：添加元素和取出元素。当队列中没有元素时，不能进行取元素的操作，直到队列中有元素时才可进行；当队列满了时，不能进行添加元素的操作，直到队列非满时才能进行添加操作。实际上，这就是生产者-消费者模式，而实现生产者-消费者模式，通常采用等待/通知的经典模式来实现。在Java中实现等待/通知又有两种类型：一种是基于Object类中的wait()/notify()方法来实现，另一种是基于AQS中Condition类的await()/signal()方法来实现。显然，LinkedBlockingQueue中使用的是Condition中的await()/signal()。

### 数据结构
LinkedBlockingQueue中包含几个十分重要的属性，这几个属性实现了LinkedBlockingQueue的线程安全和等待/通知的功能。下面将一一介绍这几个属性。

1. head和last。这两个属性的类型是Node类型，Node是LinkedBlockingQueue的一个内部类。每个Node又包含两个属性：item和next。item就是最终存元素的属性，next用来指向下一个节点，通过next属性就能在LinkedBlockingQueue内部维护一个单向链表。其中head和last分别表示链表的头部和尾部。需要特殊说明的是：在实际存储中，head的item属性始终是null，因此head不存放元素，它仅仅是表示一个链表的头结点。
```java
static class Node<E> {
    E item;
    Node<E> next;
    Node(E x) { item = x; }
}
```
2. capacity。int类型，它表示的队列的最大容量。默认情况下，capacity的值会被设置为Integer.MAX_VALUE。也可以手动指定它的值，当在LinkedBlockingQueue的构造器中传入一个int类型的值时，就会令capacity等于该值。

3. count。AtomicInteger类型，该属性的类型是一个原子类型，它表示的是当前队列中元素的个数。
4. takeLock。ReentrantLock类型。在LinkedBlockingQueue中，获取元素和添加元素采用了不同的锁，takeLock表示的是获取元素时使用的锁。
5. putLock。ReentrantLock类型，putLock表示的是添加元素时使用的锁。
6. notEmpty。非空等待队列，Condition类型，当队列为空时，不能再从队列中获取元素了，此时想从队列中获取元素的线程就需要等待，直到队列中有元素被添加进来。那么此时线程应该在哪儿等待呢？就是在notEmpty这个非空等待队列中等待。notEmpty属性的值，是通过takeLock这把锁来创建的。
```java
private final Condition notEmpty = takeLock.newCondition();
```
7. notFull。非满等待队列，Condition类型，当队列已满时，不能再向队列中添加元素了，此时向队列中添加元素的线程就需要等待，直到队列不满。那么线程应该在哪儿等待呢？就是在notFull这个非满等待队列中等待。notFull属性的值，是通过putLock这把锁来创建的。
```java
private final Condition notFull = putLock.newCondition();
```

### 源码分析
知道了LinkedBlockingQueue的内部数据结构，现在将结合具体的源码来分析下LinkedBlockingQueue的实现原理。将LinkedBlockingQueue的操作分为两类：存元素和取元素，存元素的方法有：put(e)，offer(e)，offer(e,time,unit)；取元素的方法有：take()，poll()，poll(time,unit)，peek()。

#### put(e)
当队列已满时，put(e)方法会一直阻塞线程，直到队列不满。当成功添加元素到队列中时，put(e)方法才会返回结束，该方法没有返回值。
```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    // 可中断的获取锁
    putLock.lockInterruptibly();
    try {
        
        // 如果队列满了，则进行等待，等待队列是非满状态
        while (count.get() == capacity) {
            notFull.await();
        }
        // 入队
        enqueue(node);
        // 队列元素个数自增，注意，由于这里调用的是getAndIncrement()方法，
        // 不是incrementAndGet()方法，所以返回的是自增之前的值。
        c = count.getAndIncrement();
        
        // 如果阻塞队列还没有满，就唤醒处于notFull等待队列中的线程
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    // 如果阻塞队列在没有添加元素之前，阻塞队列的元素个数为0，那么可能有线程处于notEmpty的等待队列中
    // 因此这里会唤醒处于notEmpty的等待队列中的线程
    if (c == 0)
        signalNotEmpty();
}
```
1. 当调用put(e)时，先获取putLock锁，然后判断队列是否已满，如果已经满了，就调用notFull.await()方法，让当前线程进入到notFull的等待队列中，当队列不满时，会调用notFull.signal()方法，唤醒notFull等待队列中的线程。
2. 当队列不满时，调用enqueue()方法将元素存入LinkedBlockingQueue内部维护的链表中。
3. 当元素入队成功后，再判断队列是否已满，如果未满，就唤醒notFull队列中的线程。
最后再判断队列在添加元素之前是否有元素，如果没有元素，那么可能有线程等待在notEmpty这个等待队列中，调用signalNotEmpty()方法就会唤醒处于notEmpty等待队列中的线程。

enqueue(node)方法的源码如下。
```java
private void enqueue(Node<E> node) {
    last = last.next = node;
}
```
enqueue(node)方法的源码比较简单，下面通过一个图来理解下元素入队过程。
【图】

#### offer(e)
offer(e)方法不会阻塞线程，当阻塞队列已经满了时，如果再向阻塞队列中添加元素，那么offer(e)方法会直接返回false，如果元素添加成功，则会返回true。源码如下。
```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    // 如果阻塞队列满了，直接返回false
    if (count.get() == capacity)    // ①
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            // 入队
            enqueue(node);
            c = count.getAndIncrement();
            // 未满通知
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    // 非空通知
    if (c == 0)
        signalNotEmpty();
    // 如果线程调用putLock.lock()没有获取到锁，那么此时c等于-1，因此会也会返回false
    return c >= 0;
}
```
offer(e)方法的源码与put(e)方法的源码大部分逻辑一致，不同点就在于代码中标记①的地方。offer(e)在判断队列已满时，会直接返回结束，未满时，才会进行获取putLock锁，然后进行元素添加操作。

#### offer(e,time,unit)
offer(e,time,unit)方法支持线程超时的存放元素，当阻塞队列已满时，当前线程最多等待time时间，如果在这段时间内依旧没有将元素存放入队列中，那么就会返回false。如果元素添加成功，就返回true。offer(e,time,unit)的源码如下。
```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    if (e == null) throw new NullPointerException();
    // 根据传入的time和时间单位，计算需要等待对少纳秒
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            // 如果超时，直接返回false
            if (nanos <= 0)
                return false;
            // 等待（最终是调用LockSupport.parkNanos(this, nanosTimeout)）
            nanos = notFull.awaitNanos(nanos);
        }
        // 入队
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        // 未满通知
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    // 非空通知
    if (c == 0)
        signalNotEmpty();
    return true;
}
```
offer(e,time,unit)方法与put(e)方法的逻辑也类似，不同的是，put(e)方法在等待时调用的是condition的await()方法，而offer(e,time,unit)调用的是awaitNanos(nanos)方法。awaitNanos(nanos)最终调用的是LockSupport的parkNanos(this, nanosTimeout)方法。关于Condition的源码分析，可以参考这篇文章：[Condition源码分析](https://mp.weixin.qq.com/s/cb8Xtr1wGorfArzNbfbrrw)。

#### take()
take()方法和put(e)方法相对应，take()方法用来从阻塞队列中取出元素。当阻塞队列中没有元素存在时，当前线程会一直等待，直到阻塞队列不为空，最终返回阻塞队列中存储的第一个元素。take()方法的源码如下。
```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        // 如果阻塞队列中一直没有元素，线程就一直等待，直到队列中有元素后调用notEmpty等待队列的signal()方法
        while (count.get() == 0) {
            notEmpty.await();
        }
        // 当阻塞队列中有元素后，会跳出上面的while循环，然后出阻塞队列
        x = dequeue();
        c = count.getAndDecrement();
        // 如果阻塞队列中还有元素，就唤醒等待在notEmpty等待队列中的线程
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    // 如果在元素出队列前，队列处于已满状态，那么从队列中移出一个元素后，队列就变为非满状态了
    // 此时就唤醒等待在notFull等待队列中的线程
    if (c == capacity)
        signalNotFull();
    return x;
}
```
1. 当调用take()方法时，先判断LinkedBlockingQueue中有没有元素，如果没有，就调用notEmpty.await()方法，让当前线程进入到notEmpty这个等待队列中等待。当LinkedBlockingQueue中有元素时，其他线程就会调用notEmpty.signal()方法，就会让当前线程醒来，继续执行后面的逻辑。
2. 如果LinkedBlockingQueue中有元素，就调用dequeue()方法从队列中取出元素。取出元素后，再判断队列中是否还有元素，如果还有，则唤醒处于notEmpty这个等待队列中的元素。
3. 最后判断LinkedBlockingQueue是否已满，如果没有满，就调用signalNotFull()方法，唤醒等待在notFull等待队列中的线程。

`dequeue()`方法会从LinkedBlockingQueue队列中取出存储的第一个元素，由于LinkedBlockingQueue队列中的head节点是不存储元素的，所以取出的是head.next这个节点的item属性的值。dequeue()方法的源码如下。
```java
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;
    Node<E> first = h.next;
    // 令head节点的next指针执行自己
    h.next = h; // help GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
```
在dequeue()方法中，令第二个节点变为新的head节点，并令老的head节点的next指针指向自己（为什么不让老的head节点的next指针指向null？理由后面再说）。dequeue()方法的这段代码实际上就是操作链表，修改指针指向。代码读起来可能比较费劲，下面结合图来理解下。
【图】

为什么不让老的head节点的next指针指向null？这是因为取元素操作和LinkedBlockingQueue通过迭代器的遍历所有元素的操作，这两个操作可能是同时进行的，如果这个地方将next的指针指向null了，那么在迭代器遍历时，就会出不可预知的错误。迭代器的具体源码，可以去看下LinkedBlockingQueue的内部类Itr的源代码。

#### poll()
poll()方法也是从LinkedBlockingQueue中取出元素，但是它不会阻塞线程，当LinkedBlockingQueue队列中没有元素时，poll()方法就会直接返回null；有元素时，就会先尝试获取锁，然后再取出元素。源码如下：
```java
public E poll() {
    final AtomicInteger count = this.count;
    // 如果队列为空，就立即返回null，不会阻塞线程
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        if (count.get() > 0) {
            // 取元素
            x = dequeue();
            c = count.getAndDecrement();
            // 非空通知
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    // 非满通知
    if (c == capacity)
        signalNotFull();
    return x;
}
```
poll()方法的逻辑和take()方法的逻辑基本一致，不同点在于，当LinkedBlockingQueue队列中没有元素时，poll()不会阻塞线程，take会阻塞线程。

#### poll(time,unit)
当LinkedBlockingQueue中没有元素时，poll(time,unit)也会阻塞线程，它支持的是超时阻塞。当在time时间内，没有获取到元素时，就会返回null。
```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            // 如果已经超时，直接返回null
            if (nanos <= 0)
                return null;
            // 等待(调用的是LockSupport.parkNanos(this, nanosTimeout))
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```
poll(time,unit)与take()的区别是，take是一直阻塞，poll(time,unit)是超时阻塞。

#### peek()
peek()方法也是从队列中获取元素，但是它只会获取队列中的第一个元素，且不会将元素从队列中移除。
```java
public E peek() {
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        // 取第一个元素
        Node<E> first = head.next;
        if (first == null)
            return null;
        else
            return first.item;
    } finally {
        takeLock.unlock();
    }
}
```
从peek()方法的源码中可以发现，peek()仅仅只是取出队列中的第一个元素，但是并没有修改链表的指针指向，因此它不会将元素从队列中删除。

### 总结
* LinkedBlockingQueue是一个线程安全的队列，它是一个底层基于链表实现的无界队列，当指定队列容量时，它是一个有界队列。
* 在LinkedBlockingQueue中，取元素和存元素使用的是两把锁，锁的类型是ReentrantLock。它通过使用Condition的等待/通知来实现了生产者-消费者模型。
* 由于LinkedBlockingQueue中存元素和取元素使用的是两把锁，存取操作可以同时进行，因此它的吞吐量高于ArrayBlockingQueue。

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
(https://mp.weixin.qq.com/s/qj2_x3WGIw7vg9zE-UNIVw)
* [线程池ThreadPoolExecutor的实现原理](https://mp.weixin.qq.com/s/q0Qt-ha9ps12c15KMW7NfA)
* [为什么《阿里巴巴Java开发手册》上要禁止使用Executors来创建线程池](https://mp.weixin.qq.com/s/EsJv8Uq6PS1PheJxP0HR9w)
* [并发编程系列之Future —— 最常用的性能优化手段](https://mp.weixin.qq.com/s/4gLjSGvwcuvElfnYRzDnSw)


【图】
