
> 上一篇文章介绍了工具类`CountDownLatch`的原理和使用场景（[并发工具类CountDownLatch的源码分析以及使用场景](https://mp.weixin.qq.com/s/2W3nc1G-I78WEb1ABRO5YQ)），今天将介绍JUC包下另一个十分常用的并发工具类`CyclicBarrier`，翻译过来就是可`循环`使用的`屏障`。


### 简介
* CyclicBarrier的功能与CountDownLatch的功能十分类似，也是`控制线程的执行顺序`，但是它与CountDownLatch的区别是，CyclicBarrier是让一组线程阻塞在同一屏障（同步点）处，直到最后一个线程到达屏障（也就是屏障的计数器减为0），屏障才会打开，这些阻塞在屏障处的线程才会继续往下执行。CountDownLatch是让一组或者一个线程等待其他线程执行完后，当前线程才继续执行。另外一点区别就是，CyclicBarrier的计数器减为0后，可以重置计数器，从而可以再次使用，这一点通过类名中含有`Cyclic`（循环）就能看出。而CountDownLatch的计数器减为0后，不会重置，因此不能重复使用。

### 示例
* CyclicBarrier的使用也十分简单，只需要new一个CyclicBarrier创建一个实例对象，它的构造方法中需要传入一个`int类型`的参数，用来指定`屏障的大小`（即当多少个线程到达屏障后，屏障打开），然后在线程中调用`await()`方法即可让线程阻塞在屏障处。
* 如下Demo示例中，通过10个线程模拟了十个短跑运动员。在现实生活中，100米赛跑的时候，运动员需要听到发令枪响之后才能起跑，所有运动员的起跑时间是在同一个时间的，不能抢跑。发令枪就相当于程序中的CyclicBarrier，当所有人准备好，听到发令枪响（达到屏障）时，才能开始起跑。
```java
public class CyclicBarrierDemo {

    public static void main(String[] args) {
        Random random = new Random();
        CyclicBarrier cyclicBarrier = new CyclicBarrier(10);
        List<Thread> threads = new ArrayList<>(10);
        for (int i = 0; i < 10; i++) {
            threads.add(new Thread(()->{
                int time = random.nextInt(5) + 1;
                try {
                    // 通过线程休眠来模拟每位运动员的准备时间
                    Thread.sleep(time * 1000);
                    System.out.println(Thread.currentThread().getName() + "准备就绪");
                    // 运动员准备就绪后，就示意发令员自己准备好了，即调用await()方法
                    cyclicBarrier.await();
                    System.out.println("起跑枪响，"+Thread.currentThread().getName() + "起跑");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }

            },"运动员"+(i+1)));
        }
        
        for (Thread thread : threads) {
            thread.start();
        }

    }
}
```

### 实现原理
* `CyclicBarrier`和`CoutDownLatch`的底层实现也存在一点区别，CountDownLatch底层是`直接`通过组合一个继承了AQS的同步组件来实现的，而CyclicBarrier`并没有直接`借助AQS的同步组件，而是通过`组合ReentrantLock这把锁`来实现的（ReentrantLock的底层实现依然使用的AQS来实现的，归根结底，CyclicBarrier的底层实现也是AQS）。
* 由于CyclicBarrier是使用ReentrantLock来实现的，因此它有个属性是`lock`。在CyclicBarrier中还维护了一个计数器：`count`。由于CyclicBarrier可以重复使用，即计数器减为0后，将其重置，因此还需要借助另外一个变量来存放count的初始值，这个变量就是`parties`。CyclicBarrier中有个属性是generation，其类型是一个CyclicBarrier的内部类Generation，它的作用是用来实现`await(long timeout,TimeUnit unit)`方法的超时等待的功能（后面分析源码时会详细解释）。当CyclicBarrier重置时，也会重新令generation重置赋值。CyclicBarrier的属性和方法见下表。

|属性或者方法|作用|
|---|---|
|ReentrantLock lock |用来保证线程安全，防止多个线程同时修改count时，出现线程不安全的情况|
|int count|计数器，当调用await()方法时，会令count减1|
|int parties|记录计数器的初始值|
|Generation generation|当计数器重置时，也会重置该属性。当出现超时等待时，会令generation中的broken属性为true。|
|Condition trip|等待队列|
|Runnable barrierCommand|CyclicBarrier支持当计数器减为0后，先执行一个Runnable任务，然后执行阻塞在屏障处的线程|
|await()|让线程等待在阻塞在屏障处，并令计数器减1，不支持超时等待|
|await(long timeout, TimeUnit unit)|让线程等待在阻塞在屏障处，最大等待timeout的单位时间，并令计数器减1|
|reset()|重置屏障|

* CyclicBarrier有两个有参构造器，如下。
```java
// parties用来指定计数器的大小
// barrierAction是一个Runnable，当计数器减为0时，会先执行barrierAction，然后再打开屏障
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}

// parties用来指定计数器的大小
public CyclicBarrier(int parties) {
    // 调用有两个参数的有参构造方法
    this(parties, null);
}
```
* 当执行`CyclicBarrier cyclicBarrier = new CyclicBarrier(10);`这一行代码时，会初始化计数器count的值和parties。传入的参数10，表示当有10个线程到达屏障时，才会打开屏障。
* 当调用cyclicBarrier.await()时，在await()方法中会直接调用`dowait()`方法。dowait()方法的源码如下。
```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;
        // 如果有线程调用了await(long timeout,TimeUnit unit)方法，且出现了超时等待，那么此时g.broken就为true，因此会抛出异常
        if (g.broken)
            throw new BrokenBarrierException();

        // 如果线程被中断，那么就直接中断屏障（让所有等待的线程醒来）
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        // 计数器递减
        int index = --count;
        // 如果递减后的结果为0，说明所有线程达到屏障
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                // 判断有没有需要优先执行的任务，有就执行
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                // 在nextGeneration()会唤醒等待队列中的所有线程，边让计数器的count值重置
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        // 如果计数器没有减到0，就让当前线程进入到等待队列中等待
        for (;;) {
            try {
                // timed是用来标识是否是超时等待
                if (!timed)
                    // 调用condition的await()方法，进入到等待队列
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```
* 在dowait()方法中会先判断generation.broken是否为true，在第一次进入时，肯定为false（默认值就是false），该字段的值只有当线程调用await(long timeout,TimeUint unit)方法，且出现了超时情况，才会为true。
* 然后将计数器count减1，如果减一之后count为0，表示此时所有线程都已经到达了屏障，此时就可以打开屏障，让阻塞的线程继续执行了。但是在打开屏障之前，会先因判断barrierCommand是否为空，如果不为空，就先执行barrierCommand。然后才调用nextGeneration()方法。nextGeneration()的主要作用是`唤醒等待队列中的所有线程，并重置计数器`。其源码如下。
```java
private void nextGeneration() {
    // signal completion of last generation
    // 唤醒等待队列中所有在等待的线程
    trip.signalAll();
    // set up next generation
    count = parties;
    generation = new Generation();
}
```
* 如果count减一之后的值不为0，就表示还有线程`没有到达屏障，还不能打开屏障`，因此就需要令当前线程加入到等待队列中，即会调用`trip.await()`，让线程等待。
* 如果出现等待超时了，就会执行到for循环中的`catch语句块`中，在catch语句块中调用了breakBarrier()方法，breakBarrier()方法的主要作用就是`将generation的broken属性设置true`。那么当执到`if(g.broken)`就会判断成立，然后抛出异常，这样就实现了超时等待功能。
```java
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```
* 如果没有出现超时等待，当计数器减为0时，就会唤醒trip等待队列中的所有线程，使其进入到Lock的同步队列中，接下来就是在Lock的同步队列中，一个节点一个节点的线程被唤醒，然后当线程从`trip.await()`方法处醒来，继续执行后面的逻辑。关于ReentrantLock的详细分析可以参考这两篇文章：[可重入锁（ReentrantLock）源码分析](https://mp.weixin.qq.com/s/VHCVoBn3KBt95VMgT1wQDA) `和` [公平锁与非公平锁的对比](https://mp.weixin.qq.com/s/hYLpsP_9Oxc9pSx3AGW5TQ)
* 至于Cyclicbarrier的await(long timeout,TimeUnit unit)方法的实现，最终也是调用dowait()方法，因此这里就不再详细说明。整体来说，CyclicBarrier的源码实现相对比较简单。

### 与CountDownLatch的区别
* CyclicBarrier与CountDownLacth存在几点区别。首先CyclicBarrier是让所有线程到达屏障后再一起执行后面的逻辑，而CountDownLatch是让一个线程或者一组线程等待其他线程执行完后，自己再接着执行。第二，CyclicBarrier的计数器可以重置，因此可以重复使用，而CountDownLatch的计数器不能重置，不可以重复使用。第三，CyclicBarrier可以在所有线程达到屏障后，先执行一个Runnable任务，然后才打开屏障，这个功能在特殊场景下很有用处。第四，虽然两者最终底层实现都是根据AQS来实现的，但是CyclicBarrier是通过ReentrantLock这个互斥锁来间接使用AQS实现的，而CountDownLatch是直接使用AQS的共享锁来实现的。
* 关于CyclicBarrier的构造方法中支持传入一个Runnable类型的参数，下面还是以文章开头的Demo，演示一下其用法。在上面的Demo中，运动员都准备好站到起跑线后，此时应该是发令员先鸣枪，然后运动员才开始起跑，也就是线程开始执行，那么这个鸣枪的动作就是在屏障打开之前，那么我们一个通过Runnable来实现。示例代码如下。
```java
public class CyclicBarrierDemo {

    public static void main(String[] args) {
        Random random = new Random();
        // 在CyclicBarrier构造方法中，第二个参数传入一个Runnable。
        CyclicBarrier cyclicBarrier = new CyclicBarrier(10, new Runnable() {
            @Override
            public void run() {
                System.out.println("==============  各就位！！！预备！！！砰！============");
            }
        });
        List<Thread> threads = new ArrayList<>(10);
        for (int i = 0; i < 10; i++) {
            threads.add(new Thread(()->{
                int time = random.nextInt(5) + 1;
                try {
                    Thread.sleep(time * 1000);
                    System.out.println(Thread.currentThread().getName() + "准备就绪");
                    cyclicBarrier.await();
                    System.out.println(Thread.currentThread().getName() + "起跑");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }

            },"运动员"+(i+1)));
        }

        for (Thread thread : threads) {
            thread.start();
        }

    }
}
```
* 从打印结果中可以看到，当所有运动员就绪后，会先打印出`==============  各就位！！！预备！！！砰！============`这一行后，才会让其他线程继续执行。

### 总结
* 本文详细介绍了CyclicBarrier的功能，以及如何使用，然后结合源码分析了CyclicBarrier的实现原理，并从功能上和底层实现原理上，对比了CyclicBarrier和CountDownLatch的区别，最后总结一下，在大部分场景下，CountDownLatch能实现的功能，都能使用CyclicBarrier实现。

### 推荐
* [读写锁ReadWriteLock的实现原理](https://mp.weixin.qq.com/s/KDQR4_MaR_NacE8tKfQILA)
* [Semaphore的源码分析以及使用场景](https://mp.weixin.qq.com/s/zoP6rdG_PyLJ9RflV_DLew)
* [并发工具类CountDownLatch的源码分析以及使用场景](https://mp.weixin.qq.com/s/2W3nc1G-I78WEb1ABRO5YQ)
* [队列同步器（AQS）的设计原理](https://mp.weixin.qq.com/s/a04VUQMHZAX8e9b3MVs8zw)
* [队列同步器（AQS）源码分析](https://mp.weixin.qq.com/s/xIZlHydPxLmnClNckqKrZw)
