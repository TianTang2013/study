### 1. 回顾
在上一篇文章中分享了堆这种数据结构，同时提到，堆可以用来对数据排序，也可以用来解决Top N、定时任务、优先级队列等问题，今天要分享的是Java中优先级队列PriorityQueue的源码实现，看看堆在Java中的实际应用。需要说明的是，本文与上篇文章：**[重温《数据结构与算法》之堆与堆排序](https://mp.weixin.qq.com/s/4_5t-2qn11qC-qh328v3NA)** 密切相关。

### 2. PriorityQueue
优先级队列有两个常用的操作：向队列中添加元素、取出元素，这两个操作的方法为**add(E e)和poll()**，接下来将围绕这两个方法的源码展开。

PriorityQueue最底层采用数组来存放数据，它有很多构造方法，如果使用无参的构造方法，那么队列的最大容量将会采用默认值11，当一直向队列中添加元素时，如果达到了最大容量，那么将会进行扩容。

另外，优先级队列中添加的元素，一定是能比较的大小的元素，而如何比较大小呢？有两种选择，第一：在创建PriorityQueue时**指定一个Comparator类型的比较器**；第二：添加到队列中的元素**自身实现Comparable接口**。使用无参构造方法时，优先级队列内部的比较器为null，因此在这种情况下，添加到队列中的元素需要实现Comparable接口，否则将会出现异常。
```java
// 存放数据
transient Object[] queue;

// 默认的初始容量
private static final int DEFAULT_INITIAL_CAPACITY = 11;

public PriorityQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}
```

#### 2.1 添加元素（add(E e)）
向优先级队列中添加元素，实际上就是向堆中插入一个元素，当插入一个元素后，为了满足堆的性质（父结点的值要么都大于左右子结点，要么都小于左右子结点），因此可能需要堆化。下面是Java中PriorityQueue的add(E e)方法实现，你可以对照着上篇文章中堆插入数据的过程来看。
```java
public boolean add(E e) {
    // 直接调用offer(E e)
    return offer(e);
}

public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    // 判断是否需要扩容
    if (i >= queue.length)
        grow(i + 1);    // 扩容
    size = i + 1;       // 将以存放的数据个数+1
    if (i == 0)     // 第一个元素就不需要判断是否要堆化了，直接加入即可
        queue[0] = e;
    else
        siftUp(i, e);   // 从下往上堆化
    return true;
}
```

可以看到，核心代码在**siftUp()** 方法中，该方法从名字上就能推测出，是从下往上进行堆化。代码实现如下：
```java
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);    // 如果指定了比较器，则采用指定的比较器来判断元素的大小，从而进行堆化
    else
        siftUpComparable(k, x); // 没有指定比较器，那添加到队列中的元素必须实现了Comparable接口
}
```
这里我们以 **siftUpComparable()** 方法为例分析，其实这两个方法的实现逻辑一样，不同的是怎么比较两个元素的大小。
```java
// 从下往上堆化
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1; // 数组索引除以2，实际上就是计算父结点的索引
        Object e = queue[parent];
        if (key.compareTo((E) e) >= 0)  // 当前结点与父结点进行比较
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}
```
在从下往上堆化的过程中，先就算出父结点的位置，然后和父结点比较大小，根据比较的结果，判断是否还需要继续向上堆化。这段代码和上一篇文章中堆化的代码几乎一样，可以对照着来看。

#### 2.2 取出元素（poll()）
实际上从优先级队列中取出元素的过程，就是删除堆顶元素的过程。在删除完堆顶元素后，为了满足堆的性质，因此需要进行堆化。比较简单的做法就是，将数组中最后的一个元素搬到堆顶，然后再从上到下来进行堆化。poll()方法的源码实现如下：
```java
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    // 取出堆顶元素
    E result = (E) queue[0];
    // 取出数组的最后一个元素
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x); // 将数组的最后一个元素取出后，放到堆顶，然后从上往下堆化
    return result;
}
```
可以看到，核心代码实现在**siftDown()** 方法中，该方法的作用就是从上往下堆化。
```java
// 从上往下堆化
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);  // 有比较器
    else
        siftDownComparable(k, x);   // 元素自己实现Compareable接口
}
```
有比较器和无比较器的实现逻辑几乎一致，下面只以**siftDownComparable()** 方法为例，看看从上往下的堆化过程。
```java
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    // 叶子结点不需要堆化，因此需要计算出非叶子结点的位置
    int half = size >>> 1;        // loop while a non-leaf
    while (k < half) {
        // 计算左子结点的位置
        int child = (k << 1) + 1; // assume left child is least
        Object c = queue[child];
        // 右子结点的位置
        int right = child + 1;
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}
```
从上往下堆化的过程当中，**叶子节点是不需要进行堆化的**，因此代码中，先计算出了处于数组最后面的非叶子结点的位置：**int half = size >>> 1** (>>>的含义是无符号右移， **>>> 1** 实际上就是除以2)。 接着分别将当前结点的值与左右两个结点的值比较，判断是否需要交换。这段代码的逻辑和上篇文章中heapify()方法的逻辑是一致的，可以对照着来看。堆化完，最终堆顶的元素就又变成了优先级最大或者最小的元素了。

### 3 总结
优先级队列PriorityQueue的底层实现，实际上就是堆的实现，底层采用数组来存放数据，在插入数据时，采用的是从下往上进行堆化；取出元素时，实际上就是删除堆顶元素，这个过程采用的是从上往下进行堆化。
