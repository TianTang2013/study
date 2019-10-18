> 点击上方`菜鸟飞呀飞`或者扫描下方二维码，即可关注微信公众号。

【图】

[TOC]


> 在Java并发领域，总会提到原子操作，而Java作为一门高级语言，为了实现原子操作，提供了两种解决方案：1）加锁；2）通过CAS来实现，同时JDK在1.5开始，在JUC包下，并发编程大神Doug Lea提供了很多原子类，这些原子类都是通过CAS来实现的。接下来本文将主要介绍CAS相关的知识。

### 1. CAS简介
* CAS的全称是Compare And Swap，翻译过来就是比较并交换。假设内存中数据的值为V，旧的预期值为A，新的修改值为B。那么CAS操作可以分为三个步骤：1）将旧的预期值A与内存中的值V比较；2）如果A与V的值相等，那么就将V的值设置为B；3）返回操作是否成功。
* 在多处理器的机器上，当有多个线程对内存中的数据进行CAS操作时，处理器能保证只会有一个线程能修改成功。

### 2. CAS的实现原理
> 在Java中可以通过Unsafe类实现CAS操作，而Unsafe类最终调用的是native方法，即具体实现是由JVM中的方法实现的。而JVM中通过C++调用处理器的指令`cmpxchg`来实现的。

#### 2.1 JAVA中的CAS实现
* 在Java的`java.util.concurrent.atomic`包中提供了很多原子类，这些类的操作均是原子操作，它们都是根据Unsafe类的CAS方法来实现原子操作的，如：`compareAndSwapInt()、compareAndSwapLong()、compareAndSwapObject()`这三个方法均是CAS方法。
* 例如如下示例，通过AtomicInteger类的CAS操作实现了count线程安全的累加操作。
```java
public class Demo {

    // 创建一个Integer类型的原子类
    private static AtomicInteger count = new AtomicInteger(0);

    public static void main(String[] args) {
        List<Thread> threadList = new ArrayList<>(10);
        // 创建10个线程，每个线程中的run()方法将count累加10000次
        for (int i = 0; i < 10; i++) {
            threadList.add(new Thread(()-> {
                // 每个线程累加10000次
                for (int j = 0; j < 10000; j++) {
                    // 实现递增，incrementAndGet()方法等价于非原子操作里面的 count++
                    count.incrementAndGet();
                }
            }));
        }
        // 将10个线程启动
        for (int i = 0; i < 10; i++) {
            threadList.get(i).start();
        }

        // 让主线程休眠10秒钟，休眠10秒钟是为了让前面的10个线程全部执行完成，如果10秒钟不够，可以设置得更加长一点
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 打印count的值
        // 最终发现从控制台打印出来的结果为100000
        System.out.println(count.get());

    }
}
```
* 当上面10个线程同时并发的对count进行累加时，使用了AtomicInteger.incrementAndGet()方法来累加，这个方法保证了原子性。所以10个线程，每个线程累加10000次，最终count的结果为100000。接下来我们通过源码来看看AtomicInteger类是如何来实现原子操作的。
* AtomicInteger类的部分源码如下。从源码中我们发现，AtomicInteger调用了Unsafe类的getAndAddInt()方法。
```java
public class AtomicInteger extends Number implements java.io.Serializable {
   
    private static final Unsafe unsafe = Unsafe.getUnsafe();

    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
}
```
* Unsafe类的部分源码如下。从源码中可以发现，getAndAddInt()方法会循环调用compareAndSwapInt()方法（即CAS方法），如果CAS失败，则会通过调用getIntVolatile()方法从内存中获取新值，然后再进行CAS操作，直到成功。compareAndSwapInt()方法是一个native方法，具体的实现是由JVM来完成的。
```java
public final class Unsafe {

    private static final Unsafe theUnsafe = new Unsafe();

    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
            // 循环调用compareAndSwapInt()，直到设置成功
            // 如果设置失败，则通过getIntVolatile()方法从内存中获取新的值，然后再进行CAS操作
        } while (!compareAndSwapInt(o, offset, v, v + delta));
        return v;
    }

    // native方法由JVM实现
    public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);
}
```
* 要想看JVM的源码，就需要下载OpenJDK的源码，可以从github下载（[https://github.com/openjdk/jdk](https://github.com/openjdk/jdk)），进入github后，可以选择下载tag分支下不同版本的OpenJDK，这里笔者下载的是`jdk8-b120`版本的OpenJDK。
* Java中的native方法是通过JNI的方式调用到JVM中的，对于Unsafe类，在JVM的源码中会对应到unsafe.cpp文件，该文件的路径是：`jdk-jdk8-b120/hotspot/src/share/vm/prims/unsafe.cpp`。在unsafe.cpp文件中接近末尾的地方，可以看到如下代码（只截取了部分源码）。
```c++
static JNINativeMethod methods_18[] = {
    {CC"getObject",        CC"("OBJ"J)"OBJ"",   FN_PTR(Unsafe_GetObject)},
    {CC"putObject",        CC"("OBJ"J"OBJ")V",  FN_PTR(Unsafe_SetObject)},
    {CC"getObjectVolatile",CC"("OBJ"J)"OBJ"",   FN_PTR(Unsafe_GetObjectVolatile)},
    {CC"putObjectVolatile",CC"("OBJ"J"OBJ")V",  FN_PTR(Unsafe_SetObjectVolatile)},

    {CC"getAddress",         CC"("ADR")"ADR,             FN_PTR(Unsafe_GetNativeAddress)},
    {CC"putAddress",         CC"("ADR""ADR")V",          FN_PTR(Unsafe_SetNativeAddress)},

    {CC"allocateMemory",     CC"(J)"ADR,                 FN_PTR(Unsafe_AllocateMemory)},
    {CC"reallocateMemory",   CC"("ADR"J)"ADR,            FN_PTR(Unsafe_ReallocateMemory)},
    {CC"freeMemory",         CC"("ADR")V",               FN_PTR(Unsafe_FreeMemory)},

    {CC"objectFieldOffset",  CC"("FLD")J",               FN_PTR(Unsafe_ObjectFieldOffset)},
    {CC"staticFieldOffset",  CC"("FLD")J",               FN_PTR(Unsafe_StaticFieldOffset)},
    {CC"staticFieldBase",    CC"("FLD")"OBJ,             FN_PTR(Unsafe_StaticFieldBaseFromField)},
    {CC"ensureClassInitialized",CC"("CLS")V",            FN_PTR(Unsafe_EnsureClassInitialized)},
    {CC"arrayBaseOffset",    CC"("CLS")I",               FN_PTR(Unsafe_ArrayBaseOffset)},
    {CC"arrayIndexScale",    CC"("CLS")I",               FN_PTR(Unsafe_ArrayIndexScale)},
    {CC"addressSize",        CC"()I",                    FN_PTR(Unsafe_AddressSize)},
    {CC"pageSize",           CC"()I",                    FN_PTR(Unsafe_PageSize)},

    {CC"defineClass",        CC"("DC_Args")"CLS,         FN_PTR(Unsafe_DefineClass)},
    {CC"allocateInstance",   CC"("CLS")"OBJ,             FN_PTR(Unsafe_AllocateInstance)},
    {CC"monitorEnter",       CC"("OBJ")V",               FN_PTR(Unsafe_MonitorEnter)},
    {CC"monitorExit",        CC"("OBJ")V",               FN_PTR(Unsafe_MonitorExit)},
    {CC"tryMonitorEnter",    CC"("OBJ")Z",               FN_PTR(Unsafe_TryMonitorEnter)},
    {CC"throwException",     CC"("THR")V",               FN_PTR(Unsafe_ThrowException)},
    {CC"compareAndSwapObject", CC"("OBJ"J"OBJ""OBJ")Z",  FN_PTR(Unsafe_CompareAndSwapObject)},
    // 对应Unsafe.java文件中的compareAndSwapInt()方法，Unsafe_CompareAndSwapInt()方法时对应的C++方法
    {CC"compareAndSwapInt",  CC"("OBJ"J""I""I"")Z",      FN_PTR(Unsafe_CompareAndSwapInt)},
    {CC"compareAndSwapLong", CC"("OBJ"J""J""J"")Z",      FN_PTR(Unsafe_CompareAndSwapLong)},
    {CC"putOrderedObject",   CC"("OBJ"J"OBJ")V",         FN_PTR(Unsafe_SetOrderedObject)},
    {CC"putOrderedInt",      CC"("OBJ"JI)V",             FN_PTR(Unsafe_SetOrderedInt)},
    {CC"putOrderedLong",     CC"("OBJ"JJ)V",             FN_PTR(Unsafe_SetOrderedLong)},
    {CC"park",               CC"(ZJ)V",                  FN_PTR(Unsafe_Park)},
    {CC"unpark",             CC"("OBJ")V",               FN_PTR(Unsafe_Unpark)}
}
```
* 从上面的代码中可以发现，Unsafe类中的native方法，都会在这儿找到对应的c++方法。如上面我们提到的`compareAndSwapInt()、compareAndSwapLong()、compareAndSwapObject()`这三个方法，分别对应unsafe.cpp文件中的`Unsafe_CompareAndSwapInt()、Unsafe_CompareAndSwapLong()、Unsafe_SetOrderedObject()`。所以当我们调用`Unsafe.compareAndSwapInt()`方法时，最终会调用到unsafe.cpp文件中的`Unsafe_CompareAndSwapInt()`方法。Unsafe_CompareAndSwapInt()源码如下：
```c++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  // 核心代码是Atomic::cmpxchg()
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```
* `Unsafe_CompareAndSwapInt()`的核心代码是`Atomic::cmpxchg(x, addr, e)`，它的作用最终是调用处理器的`CMPXCHG`指令。针对不同的操作系统，`Atomic::cmpxchg()`有不同的实现。例如针对64位linux的系统，它最终调用的是`jdk-jdk8-b120/hotspot/src/os_cpu/linux_x86/vm/atomic_linux_x86.inline.hpp`文件中的`Atomic::cmpxchg()`方法；针对windows操作系统，它调用的是`jdk-jdk8-b120/hotspot/src/os_cpu/windows_x86/vm/atomic_windows_x86.inline.hpp`目录下的`Atomic::cmpxchg()`方法。
* 通常我们的服务器都是64位linux系统，所以下面以64位的linux系统为例，`Atomic::cmpxchg()`的源码如下。
```c++
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```
* `\__asm__`表示的是嵌入一段汇编代码，`LOCK_IF_MP(%4)`表示的是向缓存或者总线加锁，`cmpxchgl`是汇编语言中`比较并交换`，能保证原子操作。（关于硬件层面的指令，这里就不深入研究了，有兴趣的朋友可以去了解下）
* 所以最终在Java层面，是通过Unsafe类中提供的几个native修饰的CAS方法来实现CAS操作，然后这几个native方法会调用到JVM中的方法，最终通过处理器中的`CMPXCHG`指令来实现原子操作。

#### 2.2 处理器中的原子操作的实现
> CAS操作的目的是为了实现原子操作，在Java中通过CAS实现的原子操作，实际上最终还是落地在处理器上，由处理器来实现原子操作。那么处理器是如何实现原子操作的呢？

* 不同型号的处理器，针对原子操作有不同的实现方案。下面以x86架构的处理器举例。
* 对于单处理器的单核系统而言，要实现原子操作除了保证内存的读写操作需要地址对齐以外，还需要保证操作指令序列不被打断。对于简单的原子操作，CPU会提供单条指令如INC等指令，但是对于复杂的原子操作，需要执行多条指令，这个时候如果出现上下文切换，如：任务切换等行为，会影响到原子操作。因此此时需要采用自旋锁来保证操作指令不被中断。
* 对于多处理器而言，除了要保证单处理的原子操作以外，还要保证不受其他处理器的影响。由于每个CPU均有自己的L1、L2、L3缓存，所以当多个CPU同时向内存中写同一个数据时，最终会造成数据的不准确性。
【图】
* 如上图，对于`i++`的场景，进行两次i++操作，当2个CPU同时将`i`写入到内存时，最终内存中的结果是2，而我们期望的应该是3。显然这种操作结果是不正确的。
* 那么多处理器是如何实现原子操作的呢？对于x86架构的处理器，提供了总线加锁或者缓存加锁来实现原子操作。
* （1）总线加锁。对于上面i++操作的例子，最终导致结果不正确的原因是因为同时有多个CPU向内存中写共享变量。那么如果我们保证当一个CPU在向内存中改写共享变量时，其他的CPU不会向内存中改写共享变量，就能保证结果的正确性。众所周知，CPU和内存之间交换数据是通过总线(bus)进行的。当CPU1向内存中改写共享变量时，会执行一个Lock前缀的指令，这个指令会发出一个LOCK#信号，这个信号会锁住总线，总线锁住期间，其他的CPU不能访问总线，不能访问总线，就意味着其他的CPU就不能通过总线向内存中读写数据，所以此时总线锁住期间，发出LOCK#信号的CPU能独占共享内存，这样就能保证原子操作了。
【图】
* （2）使用缓存锁来保证原子性。总线加锁虽然能保证原子操作，但是它的效率太低了，因为总线锁住期间，其他的CPU都不能访问总线，这大大降低了多处理器的性能。实际上我们在保证原子性操作时，只需要保证的是某一个共享变量内存地址的原子操作即可，所以使用缓存锁定来实现原子操作。`缓存锁定是指它会锁住这块内存区域的缓存并回写到内存，并使用缓存一致性机制来确保修改的原子性，缓存一致性机制会阻止同时修改由两个以上处理器缓存的内存区域数据`。（这一句话引用的是《Java并发编程的艺术》一书汇总的第二章第2.1节第10页）这个时候由于不会锁住总线，即保证了原子操作，也保证了性能。目前处理器在某些场景下会使用缓存锁定来代替总线锁定。

### 3. CAS产生的问题
> CAS虽然解决了原子性，但同时它也存在着三大问题。

* （1）ABA问题。在CAS操作时会先检查值有没有变化，如果没有变化则执行更新操作，如果有变化则不执行更新操作。假设原来的值为A，后来更新成了B，然后又更新成了A，这个时候去执行CAS的检查操作时，内存中的值还是A，就会误以为内存中的值没有变化，然后执行更新操作，实际上，这个时候内存中的值发生过变化。那么怎么解决ABA的问题呢？可以在每次更新值的时候添加一个版本号，那么A->B->A就变为了1A->2B->3A，这个时候就不会出现ABA的问题了。在JDK1.5开始，JUC包下提供了AtomicStampedReference类来解决ABA的问题。这个类的compareAndSet()方法会首先检查当前引用是否等于预期的引用，然后检查当前标志是都等于预期标志，如果都相等，才会调用casPair()方法执行更新操作。`casPair()`方法最终也是调用了Unsafe类中的CAS方法。AtomicStampedReference类的compareAndSet()方法的源码如下。
```java
public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
    Pair<V> current = pair;
    // 先检查期望的引用是否等于当前引用，期望的标识是否等于当前的标识
    // 然后才执行CAS操作（casPair()也是调用了CAS方法）
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
         casPair(current, Pair.of(newReference, newStamp)));
}
```
```java
private boolean casPair(Pair<V> cmp, Pair<V> val) {
    return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
}
```
* （2）性能问题。CAS会采用循环的方式来实现原子操作，如果长时间的循环设置不成功，就会一直占用CPU，给CPU带来很大的执行开销，降低应用程序的性能。
* （3）只能保证一个共享变量的原子操作。对一个共享共享变量执行CAS操作时，能保证原子操作，但是如果同时对多个共享变量执行操作时，CAS就无法同时保证这多个共享变量的原子性。这个时候可以使用将多个共享变量封装到一个对象中，然后使用JUC包下提供的AtomicReference类来实现原子操作。另外一种方案就是使用锁。

### 4. 总结
* 本文以一个Demo开始，结合AtomicInteger类，逐渐深入源码，介绍了Java中CAS操作的实现；然后又介绍了处理器中原子操作的实现。
* 最后本文总结了CAS操作带来的3大问题，以及目前针对这些问题的解决方案。
* 在Java中要实现CAS操作，就脱离不开Unsafe类的使用，本文只是简单介绍了Unsafe类的使用，下一篇文章将详细分析Unsafe类的源码以及使用场景。

参考资料：
1. 《Java并发编程的艺术》

