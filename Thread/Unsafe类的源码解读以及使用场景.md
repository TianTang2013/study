> 点击上方`菜鸟飞呀飞`或者扫描下方二维码，即可关注微信公众号。

[图]

[TOC]

> &emsp;&emsp;在上一篇文章《初始CAS的实现原理》中，提到了Unsafe类相关方法，今天这篇文章将详细介绍Unsafe类的源码。
> &emsp;&emsp;为什么要单独用一篇文章介绍Unsafe类呢？这是因为在看源码过程中，经常会碰到它，例如JUC包下的原子类、AQS、Netty等源码中，最终都会看见Unsafe类的使用。搞清楚Unsafe类的使用，对以后看源码会有很大的帮助。

### 1. Unsafe类简介
* Unsafe类是`rt.jar`中`sun.misc`包下的类，从类名就能看出来，这个类是不安全的，但是它的功能十分强大。相比C和C++的开发人员，作为一名Java开发人员是十分幸福的，因为在Java中程序员在开发时不需要关注内存的管理，对象的回收，因为JVM全部都帮助我们完成了。如果Java开发人员需要自己手动去操作内存，那么可以通过Unsafe类去进行申请，这也是Unsafe类被定义为`不安全`的类的原因，因为一不小心就容易出现`忘记释放内存`等问题。
* Unsafe类中方法很多，但大致可以分为8大类。CAS操作、内存操作、线程调度、数组相关、对象相关操作、Class相关操作、内存屏障相关、系统相关。`笔者画了一张脑图，因为图片占用空间较大，为了不影响阅读，我把这张图放在了文章末尾，以供参考。`

### 2. 如何获取Unsafe类的实例
* Unsafe类被final修饰了，表示Unsafe不能被继承；同时Unsafe的构造方法用private修饰，表示外部无法直接通过构造方法去创建实例。实际上Unsafe是一个单例对象，下面是Unsafe类的部分源码。
```java
// 类被final修饰，表示不能被继承
public final class Unsafe {

    // 构造器被私有化
    private Unsafe() {}

    private static final Unsafe theUnsafe = new Unsafe();

    public static Unsafe getUnsafe() {
        Class<?> caller = Reflection.getCallerClass();
        if (!VM.isSystemDomainLoader(caller.getClassLoader()))
            throw new SecurityException("Unsafe");
        return theUnsafe;
    }
}
```
* 虽然Unsafe是一个单例，但是我们在自己开发的类中无法通过Unsafe.getUnsafe()获取到Unsafe的实例，在程序运行时会抛出`SecurityException`异常。例如如下示例：
```java
public class Demo {

    public static void main(String[] args) {
        Unsafe unsafe = Unsafe.getUnsafe();
    }
}
```
* 运行main()方法，最终在控制台出现如下运行时异常：
```java
Exception in thread "main" java.lang.SecurityException: Unsafe
    at sun.misc.Unsafe.getUnsafe(Unsafe.java:90)
    at com.tiantang.study.Demo.main(Demo.java:14)
```
* 为什么会出现`SecurityException`异常呢？这是因为在Unsafe类的`getUnsafe()`方法中，它做了一层校验，判断当前类（Demo）的类加载器（ClassLoader）`是不是启动类加载器（Bootstrap ClassLoader）`，如果不是，则会抛出`SecurityException`异常。在JVM的类加载机制中，自定义的类使用的类加载器是`应用程序类加载器（Application ClassLoader）`，所以这个时候校验失败，会抛出异常。
* 那么如何才能获取到Unsafe类的实例呢？有两种方案。
* 第一方案：将我们自定义的类（如Demo类）所在的jar包所在的路径通过-Xbootclasspath参数添加到Java命令中，这样当程序启动时，Bootstrap ClassLoader会加载Demo类，这样校验就通过了。显然这种方式比较麻烦，而且不太实用，因为在项目中，可能需要在很多地方都使用Unsafe类，如果通过Java命令行这种方式去指定，就会很麻烦，而且容易出现纰漏。
* 第二种方案：通过反射来创建Unsafe类的实例（`反射反射，程序员的快乐`）。反射的代码可以参考如下示例：
```java
public static void main(String[] args) {
    try {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        // 将字段的访问权限设置为true
        field.setAccessible(true);
        // 因为theUnsafe字段在Unsafe类中是一个静态字段，所以通过Field.get()获取字段值时，可以传null获取
        Unsafe unsafe = (Unsafe) field.get(null);
        // 控制台能打印出对象哈希码
        System.out.println(unsafe);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 3. Unsafe功能介绍以及实际应用
* 下面将Unsafe类的API分为8大类，针对每一类操作的API方法以及常见的应用场景作介绍。

#### 3.1 CAS操作
* 在Java的锁中，经常会出现CAS操作，它们最终都调用了Unsafe类中的CAS操作方法，`compareAndSwapInt()、compareAndSwapLong()、compareAndSwapObject()`这三个CAS方法都是native方法，具体实现是在JVM中实现，它们的作用是比较并交换，这个操作是原子操作。关于CAS更详细的讲解可以参考这篇文章：[初识CAS的实现原理](https://mp.weixin.qq.com/s/oe046IRUbYeXpIpYLz19ew)。
* Unsafe在队列同步器AQS（AbstractQueuedSynchronizer）、原子类中都有应用，现在以队列同步器AQS为例，看看AQS当中时如何使用Unsafe类的。
* 在AQS中获取同步状态时，如果当前线程能获取到锁，那么就会去尝试修改同步状态state的值，这个时候就用到了Unsafe类。compareAndSetState()是AQS类中的一个方法，它实际调用的是Unsafe类的compareAndSwapInt()方法。
```java
 protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

#### 3.2 内存操作
* Unsafe能直接操作内存，它能直接进行申请内存、释放内存、内存拷贝等操作。值得注意的是Unsafe直接申请的内存是堆外内存。何谓堆外内存呢？堆外是相对于JVM的内存来说的，通常我们应用程序运行后，创建的对象均在JVM内存中的堆中，堆内存的管理是JVM来管理的，而堆外内存指的是计算机中的直接内存，不受JVM管理。因此使用Unsafe类来申请对外内存时，要特别注意，否则容易出现内存泄漏等问题。
* Unsafe类对内存的操作在网络通信框架中应用广泛，如：Netty、MINA等通信框架。在java.nio包中的DirectByteBuffer中，内存的申请、释放等逻辑都是调用Unsafe类中的对应方法来实现的。下面是DirectByteBuffer类的部分源码。
```java
DirectByteBuffer(int cap) {                   

    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        // 调用unsafe申请内存
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    // 初始化内存
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;

}
```
* Netty作为一个高性能框架，它有一个特点就是“零拷贝”，操作的是堆外内存。在操作堆外内存时，它最终使用的DirectByteBuffer来对堆外内存进行操作的。例如Netty框架中`io.netty.buffer.UnpooledUnsafeDirectByteBuf`类申请内存时的源码如下：
```java
public class UnpooledUnsafeDirectByteBuf extends AbstractReferenceCountedByteBuf {
    protected ByteBuffer allocateDirect(int initialCapacity) {
        // 调用ButeBuffer来申请堆外内存，ButeBuffer是java.nio包下的内
        return ByteBuffer.allocateDirect(initialCapacity);
    }
}
```
* `java.nio.ByteBuffer`类是通过DirectByteBuffer类来操作内存，DirectByteBuffer又是通过Unsafe类来操作内存，所以最终实际上Netty对堆外的内存的操作是通过Unsafe类中的API来实现的。

#### 3.3 线程调度相关
* Unsafe中提供了两个和线程调度相关的native方法，分别是park()和unPark()，它们的作用分别是阻塞线程、唤醒线程。在JUC包下实现的锁中，通常会用到`LockSupport.park()、LockSupport.unpark()`方法来进行线程间的通信。LockSupport中的这些方法最终调用的是Unsafe类的park()和unPark()。下面是LockSupport类的部分源代码。
```java
public class LockSupport {
    
    // UNSAFE是Unsafe类的实例
    public static void park() {
        // 阻塞线程
        UNSAFE.park(false, 0L);
    }

    public static void unpark(Thread thread) {
        if (thread != null)
            // 唤醒线程
            UNSAFE.unpark(thread);
    }

}
```

#### 3.4 数组相关
* Unsafe类中和数组相关的方法有两个：`arrayBaseOffset()、arrayIndexScale()`。
```java
// 返回数组中第一个元素在内存中的偏移量
public native int arrayBaseOffset(Class<?> arrayClass);
// 返回数组中每个元素占用的内存大小,单位是字节
public native int arrayIndexScale(Class<?> arrayClass);
```
* 根据这两个方法能计算出数组中的每一个元素在内存中的偏移量。下面通过AtomicIntegerArray类的源码来说明Unsafe类对数组的操作。AtomicIntegerArray类的部分源码如下：
```java
public class AtomicIntegerArray implements java.io.Serializable {
    private static final long serialVersionUID = 2862133569453604235L;

    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // 获取数组中第一元素在内存中的偏移量
    private static final int base = unsafe.arrayBaseOffset(int[].class);
    private static final int shift;
    private final int[] array;

    static {
        // 获取数组中每个元素占用的内存大小
        // 对于int类型的元素，占用的是4个字节大小，所以此时返回的是4
        int scale = unsafe.arrayIndexScale(int[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }

    private static long byteOffset(int i) {
        // 根据数组中第一个元素在内存中的偏移量和每个元素占用的大小，
        // 计算出数组中第i个元素在内存中的偏移量
        return ((long) i << shift) + base;
    }
}
```
#### 3.5 对象相关操作
* Unsafe类中提供了很多操作对象实例的方法。这些方法基本都是成对出现的，例如`getObject()和putObject()`，一个是从内存中获取给定对象的指定偏移量的Object类型对象，一个是向内存中写。与此类似的还有getInt()、getLong()...等方法。还有一组加了volatile语义的方法，例如：`getObjectValotile()、putObjectVolatile()`，它们的作用就是使用volatile语义获取值和存储值。什么是volatile语义呢？就是读数据时每次都从内存中取最新的值，而不是使用CPU缓存中的值；存数据时将值立马刷新到内存，而不是先写到CPU缓存，等以后再刷新回内存。部分方法注释如下：
```java
//从对象o的指定地址偏移量offset处获取变量的引用，与此类似方法有：getInt，getLong等等
public native Object getObject(Object o, long offset);
//对对象o的指定地址偏移量offset处设值，与此类似方法有：putInt，putLong等等
public native void putObject(Object o, long offset, Object x);
//从对象o的指定地址偏移量offset处获取变量的引用，使用volatile语义读取,与此类似方法有：getIntVolatile，getLongVolatile等等
public native Object getObjectVolatile(Object o, long offset);
//对对象o的指定地址偏移量offset处设值，使用volatile语义存储，与此类似方法有：putIntVolatile，putLongVolatile等等
public native void putObjectVolatile(Object o, long offset, Object x);
```
* 和对象相关操作的方法还有一个十分常用的方法：`objectFieldOffset()`。它的作用是获取对象的某个非静态字段`相对于该对象`的偏移地址，它与`staticFieldOffset()`的作用类似，但是存在一点区别。staticFieldOffset()获取的是静态字段相对于类对象（即类所对应的Class对象）的偏移地址。静态字段存在于方法区中，静态字段每次获取的偏移量的值都是相同的。
```java
// 获取对象的某个非静态字段相对于该对象的偏移地址
public native long objectFieldOffset(Field f);
```
* `objectFieldOffset()`的应用场景十分广泛，因为在Unsafe类中，大部分API方法都需要传入一个offset参数，这个参数表示的是偏移量，要想直接操作内存中某个地址的数据，就必须先找到这个数据在哪儿，而通过offset就能知道这个数据在哪儿。因此这个方法应用得十分广泛，下面以AtomicInteger类为例：在静态代码块中，通过`objectFieldOffset()`获取了value属性在内存中的偏移量，这样后面将value写入到内存时，就能根据offset来写入了。
```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            // 在static静态块中调用objectFieldOffset()方法，获取value字段在内存中的偏移量
            // 因为后面AtomicInteger在进行原子操作时，需要调用Unsafe类的CAS方法，而这些方法均需要传入offset这个参数
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
}
```

#### 3.6 Class相关操作
* Unsafe类中提供了一些和Class操作相关的方法，例如获取静态字段在内存中的偏移量的方法：`staticFieldOffset()`，获取静态字段的对象指针：`staticFieldBase()`。
```java
// 获取给定静态字段的偏移量
public native long staticFieldOffset(Field f);

// 获取给定静态字段的对象指针
public native Object staticFieldBase(Field f);
```
* 另外在JDK1.8开始，Java开始支持lambda表达式，而lambda表达式的实现是由字节码指令`invokedynimic`和`VM Anonymous Class`模板机制来实现的，`VM Anonymous Class`模板机制最终会使用到Unsafe类的defineAnonymousClass()方法来创建匿名类。对这一块感兴趣的朋友可以去查阅一下相关的资料，欢迎分享。
```java
 // 定义一个匿名内部类
 public native Class<?> defineAnonymousClass(Class<?> hostClass, byte[] data, Object[] cpPatches);
```

#### 3.7 内存屏障相关
* Unsafe类从JDK1.8开始，提供了三个和内存屏障相关的API方法。分别是`loadFence()、 storeFence() 、fullFence()`。
```java
// 禁止load操作重排序
public native void loadFence();

// 禁止store操作重排序
public native void storeFence();

// 禁止load和store操作重排序
public native void fullFence();
```

#### 3.8 系统相关
* Unsafe类中提供了两个和系统相关的API方法。
```java
// 获取指针的大小，单位是字节。
// 对于64位系统，返回8，表示指针大小是8字节
// 对于32位系统，返回4，表示指针大小是4字节
public native int addressSize();

// 返回内存页的大小，单位是字节。返回值一定是2的多少次幂
public native int pageSize();
```
* 例如如下示例，是在笔者电脑上运行的结果:
```java
 public static void main(String[] args) {
    Unsafe unsafe = null;
    try {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        unsafe = (Unsafe) field.get(null);
    } catch (Exception e) {
        e.printStackTrace();
    }
    // 指针大小
    System.out.println(unsafe.addressSize());
    // 内存页大小
    System.out.println(unsafe.pageSize());
}
```
* 控制台打印结果如下。笔者电脑的指针大小为8字节，内存页大小为4096字节，即4KB。
```java
8
4096
```
* 这两个方法在`java.nio.Bits`类中有实际应用。Bits作为工具类，提供了计算所申请内存需要占用多少内存页的方法，这个时候需要知道硬件的内存页大小，才能计算出占用内存页的数量。因此在这里借助了Unsafe.pageSize()方法来实现。`Bits`类的部分源码如下。
```java
class Bits { 
    static int pageSize() {
        if (pageSize == -1)
            // 获取内存页大小
            pageSize = unsafe().pageSize();
        return pageSize;
    }

    // 根据内存大小，计算需要的内存页数量
    static int pageCount(long size) {
        return (int)(size + (long)pageSize() - 1L) / pageSize();
    }  
}
```

### 4. 总结
* 本文详细介绍了Unsafe类的使用，以及各类API方法的作用和应用场景。对于Java中并发编程，Java的源码里面存着这大量的Unsafe类的使用，主要使用的是和CAS操作相关的三个方法，所以搞清楚这三个方法，对看懂Java并发编程的源码有很大帮助。
* 另外Unsafe类中`objectFieldOffset(Field f)`这个方法很常用，它是获取字段在内存中的偏移量，通常和Unsafe类中的其他方法结合使用。通过这个方法能知道要修改的数据在内存中的位置，然后再通过Unsafe类中其他方法来根据数据在内存中的位置从而来修改数据。
* 看完本篇文章，相信你现在应该能看懂JUC包中的很多源码了。
* 关于CAS相关的介绍可以参考另一篇文章。[初识CAS的实现原理](https://mp.weixin.qq.com/s/oe046IRUbYeXpIpYLz19ew)

【图】



