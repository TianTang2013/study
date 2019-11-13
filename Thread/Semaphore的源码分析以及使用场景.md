### 简介
* `Semaphore`翻译过来就是信号的意思，在Java中通常称它为信号量，是JUC包下提供的一个并发工具类，它的作用就是`控制同时访问共享资源的线程数`。什么意思呢？
* 假如有个停车场，一共有100个车位，那么同一时刻，停车场内最多只能停放100辆车。当停车场没有空余车位时，停车场入口处的门卫就会阻止后面的车辆进入停车场，让其在门口排队等着。只有当停车场内有车开出停车场，释放停车位时，门卫才会允许其他的车辆进入，那么这个时候我们可以说是停车场的门卫控制着同时进入停车场的车辆数。在程序中，Semaphore对应这里的门卫，车辆对应程序中的线程，停车场的容量100对应访问共享资源的最大线程数，Semaphore控制着同一时刻进入停车场的车辆的数量。
* 在Semaphore中有一个概念叫做`许可证`，可以理解为`它是线程访问共享资源的凭证`，如果线程拥有一个许可证，那么它就可以去访问共享资源，如果没有许可证，就不能访问共享资源。许可证的作用和锁有点类似，但严格意义上来讲，它和锁存在一点区别，什么区别呢？文章后面再说。
* Semaphore的主要工作就是`许可证的发放与添加`，当有线程访问共享资源时，如果许可证还有剩余，就将许可证数量减一，线程可以访问共享资源；当许可证数量为0时，当前线程就无法访问共享资源，就让线程进入同步队列，开始等待。当有线程不再访问共享资源时，会释放一个许可证，也就是将许可证加一，如果同步队列中有线程在等待获取许可证，那么就唤醒一个线程。
* Semaphore的使用十分简单，使用`acquire()`和`release()`即可`获取`和`释放`许可。同时Semaphore还提供了其他的方法，如下表。

|方法名|作用|
|---|---|
|acquire()|可响应中断的获取1个许可证|
|acquireUninterruptibly()|不响应中断的获取1个许可证|
|acquire(int permits)|响应中断的获取指定数量的许可证|
|acquireUninterruptibly(int permits)|不响应中断的获取指定数量的许可证|
|release()|释放1个许可证|
|release(int permits)|释放指定数量的许可证|
|availablePermits()|返回剩余可用的许可证数量|
|drainPermits()|获取剩余可用的许可证|

### 源码分析
* Semaphore其实是一个`共享锁`，它的`底层实现是队列同步器（AQS）`，所以Semaphore也是通过组合一个同步组件来实现具体逻辑，这个组件需要继承AQS。在Semaphore中定义了一个内部类Sync，Sync继承了AQS。Semaphore也区分`公平性和非公平性`，它通过两个内部类`FairSync`和`NonfairSync`来是实现，这两个类继承了Sync。
* Semaphore提供了两个构造方法。当使用一个参数的构造方法时，创建的是非公平的Semaphore，参数permits表示许可证的数量。当使用两个参数的构造方法时，参数fair决定创建的是公平还是非公平的Semaphore，参数permits表示许可证的数量。
```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
* 由于Semaphore同时允许多个线程访问共享资源，因此它是共享锁，所以它最终调用的是AQS中和共享式锁相关的方法。
* 当调用Semaphore.acquire()方法时，会先调用到AQS的acquireSharedInterruptibly()方法，通过方法名就能猜出，acquireSharedInterruptibly()方法是响应中断的，从而acquire()方法能响应中断。
```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    // 响应中断
    if (Thread.interrupted())
        throw new InterruptedException();
    // tryAcquireShared()方法是尝试获取锁，方法逻辑由AQS的子类实现
    // 若返回值小于0则表示获取锁(许可证)失败。
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```
* 在acquireSharedInterruptibly()方法中，会先调用子类的tryAcquire()方法。公平的Semaphore（也就是FairSync）和非公共平的Samaphore（也就是NonfairSync），tryAcquire()方法的具体实现不一样。下面以FairSync的源码为例。
```java
static final class FairSync extends Sync {
    
    protected int tryAcquireShared(int acquires) {
        for (;;) {
            // 判断同步队列中是否有线程在排队，如果队列中有线程排队，就直接返回-1，表示获取锁失败
            if (hasQueuedPredecessors())
                return -1;
            // 获取当前同步变量的值
            int available = getState();
            // 将当前同步变量的值减去即将获取的许可证数量
            /**
             * 如果remaining小于0，就表示当前线程获取锁失败,因为许可证不够了，所以直接返回remaining，此时remaining是一个负数，负数表示获取共享锁失败
             * 如果remianing大于等于0，然后将进行CAS操作，修改成功，就表示当前线程获取锁成功，返回remaining，此时remaining是一个非负数。如果修改失败，就进入下一次循环
             */
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```
* 对于公平锁而言，在获取许可证之前，会先判断同步队列中是否有线程在排队，如果有，那么就直接返回-1，表示获取许可失败。因为前面有线程排队，为了保证公平性，所以此时当前线程不能插队，因此会返回-1。如果没有线程排队，那么就让此时同步变量state的值减去即将获取许可的数量，如果相减的结果小于0，表示此时许可证的数量不够了，那么就会返回一个负数，表示当前线程获取许可失败。如果相减的结果大于等于0，表示许可证数量足够，因此进行CAS操作，将state设置为剩余许可证的数量，最后如果CAS操作成功，就返回一个大于等于0的数，表示当前线程获取许可证成功。
* 当从tryAcquire()方法返回后，就会回到AQS的acquireSharedInterruptibly()方法中。如果线程获取到了许可证，那么就会直接返回。如果没有获取到许可证，就会执行doAcquireSharedInterruptibly()方法。剩下的就和其他共享锁的操作一模一样了，将当前线程加入到同步队列，然后将当前线程park。关于共享锁的详细逻辑，有兴趣的朋友可以自己去研究下，也可以阅读这两篇文章：[队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw) `和` [队列同步器（AQS）源码分析](https://mp.weixin.qq.com/s/xIZlHydPxLmnClNckqKrZw)，详细了解一下AQS的设计原理和源码分析。

* 对于非公平的Semaphore而言，在NonFairSync的tryAcquire()方法中会直接调用父类Sync的`nonfairTryAcquireShared()`方法。nonfairTryAcquireShared()方法的源码如下。它的逻辑几乎与公平锁一样，唯一的区别就是`非公平锁在减少许可证之前，没有调用hasQueuedPredecessors()方法判断队列中是否有线程排队`。
```java
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        // 与公平锁的区别就是，不会判断同步队列中是否有线程在排队
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```
* 当调用Semaphore.release()方法时，会调用到AQS的releaseShared()方法。releaseShared()方法的作用就是释放共享锁，其源码如下。
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
* 和其他类型的共享锁的释放一样，也是先调用子类的tryReleaseShared()。对于Semaphore而言，会调用Semaphore的内部类Sync的tryReleaseShared()方法。源码如下。
```java
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        // 利用CAS操作，将同步变量的值加release
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```
* tryReleaseShared()方法的逻辑比较简单，就是将state的值加releases，也就是添加许可证，然后通过for的死循环和CAS操作来保证原子性。当返回true时，表示释放许可证成功。回到AQS中的releaseShared()方法中，剩下的逻辑就和其他类型共享锁的释放一模一样了，唤醒同步队列中正在等待的线程。
* 从整体来看，相对而言，Semaphore的源码相对比较简单，尤其是明白AQS的设计原理和源码实现后，Semaphore的实现非常简单。

### 使用场景
* 虽然Semaphore的底层实现调用的是AQS中与共享锁相关的方法，但是根据Semaphore的特点，我们可以根据Semaphore来实现一个排他锁。因为Semaphore的构造方法中，我们可以传入一个int类型的参数，用来表示许可证的数量。当我们将这个参数传为1的时候，此时只有一个许可证，那么同一时刻就会只允许一个线程去访问共享资源，这个时候Semaphore就是一个排他锁了。不过与ReentrantLock不一样的地方是，Semaphore不支持重入，这一点我们从tryAcquireShared()方法的源码中就能看到。
* 用Semaphore实现排他锁那就太大材小用了，而且ReentrantLock实现的排他锁功能更加丰富，Semaphore还可以用于其他场景。
* 当许可证的数量大于1时，Semaphore就变成了一把共享锁。在实际工作中，我们经常会接触到池化技术，例如数据库连接池、redis连接池等等，这些池化技术出现的根本原因是，池中的资源是有限的，不能无限创建，当出现高并发的场景是，我们必须保证同一时刻最大不能超过指定数量的线程来得到这些资源，那么这个时候Semaphore就派上用场了。
* 在如下Demo示例中，创建了一个拥有2个许可证的信号量，表示同一时刻只允许两个线程访问数据库，然后启动了10个线程去模拟获取数据库的连接，然后对数据库进行操作，在demo中，为了模拟对数据库的操作，让线程休眠了两秒钟。
```java
public class SemaphoreDemo {

    public static void main(String[] args) {
        // 创建2个许可证，表示同一时刻只允许两个线程访问数据库
        Semaphore semaphore = new Semaphore(2);
        List<Thread> threads = new ArrayList<>(10);
        for (int i = 0; i < 10; i++) {
            int index = i;
            threads.add(new Thread(() -> {
                try {
                    // 在获取数据库连接之前先要获取到许可，这样就能保证统一时刻最大允许鬼固定的线程获取到数据库资源
                    semaphore.acquire();
                    // 获取数据链接
                    // 保存数据
                    // 让当前线程睡眠两秒，模拟业务处理的时间
                    Thread.sleep(2000);
                    System.out.println("线程T" + index + "操作成功");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    semaphore.release();
                }

            }, "T" + i));
        }
        for (Thread thread : threads) {
            thread.start();
        }

    }
}
```
* 运行main()方法，从控制台中我们可以发现，同一时刻最多只会有两个线程在打印。
* 根据Semaphore的特点，还可以用它来做简易版的限流器。当某一时刻系统的并发量较大的时候，可以简单的使用Semaphore来实现流量控制，只有从Semaphore中获取到许可证的连接，才让它继续访问系统，否则返回系统繁忙等提示。当然了，Semaphore的性能当然满足不了双十一这种高并发的场景，关于高性能的限流器，市面上有更好的解决方法，那就是`Guava RateLimiter`。有兴趣的朋友可以自行研究下，后面会有单独的文章分析。

### 总结
* Semaphore其实就是一个共享锁，底层通过使用AQS中共享锁相关的方法，来实现同一时刻对访问共享资源的线程数的控制。但是它和锁又存在一点区别，对于锁而言，在调用释放锁的方法之前，必须保证当前线程已经持有了锁，否则会抛出`IllegalMonitorStateException`异常。而对于Semaphore而言，可以在没有调用acquire()方法之前，就能直接调用release()方法，而且不会抛出异常，在调用release()方法的时候会直接将许可证的数量+1。可以参考如下示例理解。
```java
public static void main(String[] args) {
    Semaphore semaphore = new Semaphore(1);
    // availablePermits()方法的作用是获取可用的许可证数量
    // 打印结果为1
    System.out.println(semaphore.availablePermits());
    // 没有调用acquire()方法，而是直接调用release()方法
    semaphore.release();
    // 打印结果为2
    System.out.println(semaphore.availablePermits());
}
```
* Semaphore其实就是一个工具类，根据特性，初始化时如果指定许可证的数量为1，那么Semaphore就是一个排他锁了。如果许可证的数量大于1，那么Semaphore就是一个共享锁了，可以用来做流量控制等功能。

### 相关推荐
* [队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw)
* [队列同步器（AQS）源码分析](https://mp.weixin.qq.com/s/xIZlHydPxLmnClNckqKrZw)
* [读写锁ReadWriteLock的实现原理](https://mp.weixin.qq.com/s/KDQR4_MaR_NacE8tKfQILA)
* [可重入锁（ReentrantLock）源码分析](https://mp.weixin.qq.com/s/VHCVoBn3KBt95VMgT1wQDA)
* [相亲相爱的@Import和@EnableXXX](https://mp.weixin.qq.com/s/y_2Z9m0gevp-cMkEIflrwA)
