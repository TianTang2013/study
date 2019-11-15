### 问题
* 在join()方法中最终会调用到对象的wait()方法，而wait()方法通常是和notify()或者notifyAll()方法成对出现的。而在使用join()方法时，我们压根就没有写notify()或者notifyAll()方法，这是为什么呢？
* 如果你知道答案，那么本文将对你没有任何帮助，你可以直接跳过本文。

### 前言
* 在前面两篇文章中分析了CyclicBarrier和CountDownLatch的用法和使用原理，它们都是用来控制线程的执行顺序的，这两个类是JUC包下提供的两个工具类，是由并发大佬Doug Lea开发的。实际上在Java语言当中，也提供了相关的API用来控制线程的执行顺序，那就是Thread类中的join()方法，它是由Sun公司（现被Oracle收购）的开发人员开发的。

### 如何使用
* join()方法的作用是：在当前线程A中调用另外一个线程B的join()方法后，会让当前线程A阻塞，直到线程B的逻辑执行完成，A线程才会解阻塞，然后继续执行自己的业务逻辑。可以通过如下Demo示例感受下其用法。
* 在Demo示例中，”main线程“因为肚子饿了想吃饭，因此让”保姆（线程thread）“ 去做饭，只有饭做好了才能开始吃饭，因此”main线程“需要等待”保姆（线程thread）“ 完全执行完了（饭做好了）才能开始吃饭，因此在”main线程“中调用”保姆（线程thread）“的join()方法，让”保姆（线程thread）“ 把饭做完了再通知自己去吃饭。
```java
public class JoinDemo {

    public static void main(String[] args) {
        System.out.println("肚子饿了，想吃饭饭");

        Thread thread = new Thread(() -> {
            System.out.println("开始做饭饭");
            try {
                // 让线程休眠10秒钟，模拟做饭的过程
                Thread.sleep(10000l);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("饭饭做好了");
        });
        thread.start();

        try {
            // 等待饭做好
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 只有饭做好了，才能开始吃饭
        System.out.println("开始吃饭饭");

    }
}
```

### 原理
* join()方法的使用很简单，下面来看下它的实现原理。
* join()方法的作用，其本质实际上就是`线程之间的通信`。CyclicBarrier和CountDownLatch这两个类的作用的本质也是线程之间的通信，在源码分析中，我们可以发现，这两个类的底层最终是通过AQS来实现的，而AQS中是通过LockSupport类的`park()`和`unpark()`方法来实现线程通信的。而Thread类的join()则是通过Object类的`wait`和`notify()、notifyAll()`方法来实现的。
* 当调用thread.join()时，为什么能让线程阻塞呢？它是如何实现的呢？答案就在join()方法的源码当中。join()方法有3个重载方法，其他两个方法支持超时等待，当超过指定时间后，如果子线程还没有执行完成，那么主线程就会直接醒来。当调用join()方法中，会直接调用`join(0)`方法，`参数传0表示不限时长地等待`。join(long millis)方法的源码如下。
```java
public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    // 当millis为0时，表示不限时长地等待
    if (millis == 0) {
        // 通过while()死循环，只要线程还活着，那么就等待
        while (isAlive()) {
            wait(0);
        }
    } else {
        // 当millis不为0时，就需要进行超时时间的计算，然后让线程等待指定的时间
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```
* 从上面的源码中，可以发现，join(long millis)方法的签名中，加了`synchronized`关键字来保证线程安全，join(long millis)最终调用的是Object对象的wait()方法，让主线程等待。这里需要注意的是，synchronized关键字实现的隐式锁，锁的是`this`，即thread这个对象（因为synchronized修饰的join(long millis)方法是一个成员方法，因此锁的是实例对象），在Demo中，我们是在main线程中，调用了thread这个对象的join()方法，所以这里调用wait()方法时，调用的是thread这个对象的wait()方法，所以是main线程进入到等待状态中。（`调用的是thread这个对象的wait()方法，这一点很重要，因为后面唤醒main线程时，需要用thread这个对象的notify()或者notifyAll()方法`）。
* 那么问题来了，既然调用了wait()方法，那么notify()或者notifyAll()方法是在哪儿被调用的？然而我们找遍了join()方法的源码以及我们自己写的Demo代码，都没有看到notify()或者notifyAll()方法的身影。我们都知道，wait()和notify()或者notifyAll()`肯定是成对出现的`，单独使用它们毫无意义，那么在join()方法的使用场景下，notify()或者notifyAll()方法是在哪儿被调用的呢？答案就是`jvm`。
* Java里面的Thread类在JVM上对应的文件是thread.cpp。thread.cpp文件的路径是`jdk-jdk8-b120/hotspot/src/share/vm/runtime/thread.cpp`（笔者本地下载的是openJDK8的源代码）。在thread.cpp文件中定义了很多和线程相关的方法，Thread.java类中的native方法就是在thread.cpp中实现的。
* 根据join()方法的描述，它的作用是先让thread业务逻辑执行完成，然后才让main线程开始执行。所以我们可以猜测notify()或者notifyAll()方法应该是在线程执行完run()方法后，JVM对线程做一些收尾工作时调用的。在JVM中，当每个线程执行完成时，会调用到thread.cpp文件中`JavaThread::exit(bool destroy_vm, ExitType exit_type)`方法（该方法的代码在1730行附近）。exit()方法的源码很长，差不多200行，删除了无用的代码，只保留了和今天主题有关的内容。源码如下。
```c++
void JavaThread::exit(bool destroy_vm, ExitType exit_type) {
    assert(this == JavaThread::current(),  "thread consistency check");
    // ...省略了很多代码

    // Notify waiters on thread object. This has to be done after exit() is called
    // on the thread (if the thread is the last thread in a daemon ThreadGroup the
    // group should have the destroyed bit set before waiters are notified).
    
    // 关键代码就在这一行，从方法名就可以推断出，它是线程在退出时，用来确保join()方法的相关逻辑的。而这里的this就是指的当前线程。
    // 从上面的英文注释也能看出，它是用来处理join()相关的逻辑的
    ensure_join(this);

    // ...省略了很多代码

}
```
* 可以看到，主要逻辑在`ensure_join()`方法中，接着找到`ensure_join()`方法的源码，源码如下。（`ensure_join()`方法的源码也在thread.cpp文件当中。有兴趣的朋友可以使用JetBrains公司提供的Clion这款智能工具来查看JVM的源代码，和IDEA产不多，按住option（或者Alt键）+鼠标左键，也能跟踪到源代码中）。
```c++
static void ensure_join(JavaThread* thread) {
  // We do not need to grap the Threads_lock, since we are operating on ourself.
  Handle threadObj(thread, thread->threadObj());
  assert(threadObj.not_null(), "java thread object must exist");
  ObjectLocker lock(threadObj, thread);
  // Ignore pending exception (ThreadDeath), since we are exiting anyway
  thread->clear_pending_exception();
  // Thread is exiting. So set thread_status field in  java.lang.Thread class to TERMINATED.
  java_lang_Thread::set_thread_status(threadObj(), java_lang_Thread::TERMINATED);
  // Clear the native thread instance - this makes isAlive return false and allows the join()
  // to complete once we've done the notify_all below
  java_lang_Thread::set_thread(threadObj(), NULL);


  // 核心代码在下面这一行
  // 是不是很惊喜？果然见到了和notify相关的单词，不用怀疑，notify_all()方法肯定就是用来唤醒。这里的thread对象就是我们demo中的子线程thread这个实例对象
  lock.notify_all(thread);


  // Ignore pending exception (ThreadDeath), since we are exiting anyway
  thread->clear_pending_exception();
}
```

### 总结
* 本文主要介绍了join()方法的作用是用来控制线程的执行顺序的，并结合Demo演示了其用法，然后结合源代码分析了join()方法的实现原理。join(long millis)方法因为使用了synchronized关键字修饰，所以是一个同步方法，它锁的对象是this，也就是实例对象本身。在join(long millis)方法中调用了实例对象的wait()方法，而notifyAll()方法是在jvm中调用的。在实际开发过程中，join()使用的比较少，我们通常会使用JUC包下提供的工具类`CountDownLatch`或者`CyclicBarrier`，因为后两者的功能更加强大。
* 在join(long millis)方法的源码中，隐藏了一个经典的编程范式。如下。
```java
// 这种写法是等待/通知的经典范式。
while(条件判断){
    wait();
}
```
