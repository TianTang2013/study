[TOC]

### 1.前言

在上一篇文章中分析了**NioEventLoop**的创建以及启动过程的源码，在文章结尾处提到，当**NioEventLoop**线程启动以后，会一直在一个无限 for 循环中一直循环，至死方休，那么在循环中，**NioEventLoop**到底在循环处理什么呢？这将是本文分析的重点。同时，还是先思考一下以下两个问题。

1. 众所周知，Netty 巧妙的避免了 JDK 的空轮询的 BUG，那么什么是 JDK 的空轮询 BUG？
2. Netty 又是如何避免的？

### 2.NioEventLoop.run()

在 NioEventLoop 通过 doStartThread()启动线程后，会执行这一行代码。这一行代码最终会调用的是 NioEventLoop 的 run()方法。

```java
SingleThreadEventExecutor.this.run();
```

run()方法的源码很长，但主要逻辑主要分为三部分，第一部分：通过调用 select()从操作系统中轮询到网络 IO 事件；第二部分：处理 IO 事件；第三部分：处理 nioEventLoop 的任务队列中的普通任务和定时任务。我对 run()方法的源码进行了删减，精简后的源码如下。

【图】

在 run()方法中，会通过选择策略（**selectStrategy** ）来计算 switch 语句中的条件值，在计算的时候，会先通过 **hasTasks()** 方法来判断 taskQueue 和 tailQueue 中是否有任务等待被执行，如果有任务，则将调用 selectNow()方法从操作系统中来轮询网络 IO 事件；如果没有任务，则将调用 select(timeout)方法来轮询网络 IO 事件。

为什么要这样做呢？因为 Netty 中为了保证任务被及时执行，selectNow()方法是个非阻塞方法，如果操作系统中没有已经准备好的网络 IO 事件，那么就会立即返回，有已经准备好的网络 IO 事件，那么就会将这些网络 IO 事件查询出来并立马返回。而 select(timeout)方法也是从操作系统中轮询网络 IO 事件，但是它是一个阻塞方法，当 netty 中有任务等待被执行时，使用阻塞方法，显然会造成任务被执行不及时的问题。

如果**selectStrategy**计算出来的值为-1，那么就会执行到下面这一行代码。

```java
select(wakenUp.getAndSet(false))
```

这行代码首先会将 **wakenUp** 的值置为 false。wakenUp 字段表示的含义是是否需要唤醒 selector，在每次进行新的轮询时，都会将 wakenUp 设置为 false。然后调用 select()方法，从操作系统中轮询出来网络 IO 事件。

接着在 run()方法中会对 **ioRatio** 的值进行判断，ioRatio 的含义又是什么呢？在 Netty 中，NioEventLoop 每一次循环其实主要干两类事，一是处理网络 IO 事件，二是执行任务（包括普通任务和定时任务），但是处理这两类任务的所消耗的时间是不一样的。而且有些系统可能期望分配给处理网络 IO 事件的时间多一点，有些系统可能期望分配给处理任务的时间多一些，那么 netty 就需要提供一个变量来控制执行这两类事的所花的时间的占比，这个变量就是 **ioRatio**，翻译过来就是 IO 的时间占比。默认情况下，ioRatio 的值为 50，即处理网络 IO 的时间和处理任务的时间各占一半。所以默认情况下，会进入到 else 语句块中，在 else 语句块中，先进行了网络 IO 的处理(**processSelectedKeys()**)，然后进行任务的处理(**runAllTasks(timeoutNanos)**)。

接下来将对 run()方法中的三个主要部分进行详细分析。

### 3.轮询事件(select())

如果**selectStrategy**计算出来的值为-1，那么就会执行 select(oldWakenUp)方法。该方法的源码又是很长，为了方便阅读，我进行了删减，其中省略了解决 JDK 空轮询 bug 相关的代码，这一部分代码会在后面详细说明。精简后的源码如下。

```java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        // 当定时任务队列中没有任务时，select操作的截止时间为： 当前时间 + 1秒
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

        for (;;) {
            // 1.定时任务截止时间快到了（<0.5ms），就跳出循环
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            if (timeoutMillis <= 0) {
                // 如果还没有进行过select()操作，就执行一次selectNow()操作，selectNow()方法不会阻塞线程。
                // 如果进行过select()操作，就直接跳出循环
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }
            // 轮询过程中，可能会有新的任务加入到任务队列中，就中断本次轮询，并将wakeUp设置为true
            // 为什么呢？因为netty为了保证任务能够及时执行
            if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                selector.selectNow();
                selectCnt = 1;
                break;
            }
            // 代码执行到这一行，说明任务队列中没有任务，且当前时间距离所有定时任务的执行时间大于0.5ms，那么就执行一次阻塞的select()
            // 阻塞时间就是当前时间距离第一个定时任务的执行时间
            int selectedKeys = selector.select(timeoutMillis);
            selectCnt ++;

            // 如果碰到以下任意一种情况，就中断循环
            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                break;
            }
            // JDK空轮询bug处理
        }
        // 省略部分代码
    } catch (CancelledKeyException e) {
        // 异常处理
    }
}
```

首先会计算出 select 的截止时间：**selectDeadLineNanos**，如果任务队列中没有任务，截止时间就等于当前时间加 **1 秒**。为什么要计算截止时间呢？当然是不能让线程一直在这儿从操作系统中获取网络 IO 事件啊，后面还需要处理 IO 事件和执行任务。

然后又会计算出定时任务的到期时间，如果当时任务的到期时间距离当前时间小于**0.5ms**，那么就会执行 selectNow()操作，然后返回。因为定时任务马上就要被执行了，不能再让线程执行 select()这种阻塞操作了，而是使用能立马从操作系统中返回的 selectNow()方法。

接着又会判断任务队列中有没有任务，如果有任务，就将 wakenUp 先设置为 **true**，表示要将 selector 唤醒；然后又是执行 selectNow()操作，从操作系统中轮询出已经注册的 IO 事件，最后通过 break 终止 for 循环。

如果在 **0.5ms** 内不会有定时任务快到期，也没有普通任务要执行，那么就表明可以进行阻塞式的 select()操作了，调用多路复用器的 select(timeoutMillis)方法，从操作系统中阻塞式地轮询出网络 IO 事件，该方法会返回轮询到事件数量，然后赋值给 selectedKeys 变量。

接下来就进行以下判断，如果满足以下任意一种情况，那么就终止当前 for 循环。

- 1. selectedKeys != 0，这一个条件表示从操作系统中轮询到了 IO 事件，既然轮询到了 IO 事件，那就立即返回，然后去处理这些 IO 事件吧。
- 2. oldWakenUp 为 true。
- 3. wakeUp.get()为 true （因为 wakeUp 为 true 时，就表示要唤醒 select 操作，因此对于第二点和第三点，当这两种情况成立时，需要立即返回，终止 select 操作）。
- 4. hasTasks()，这一条件表名任务队列中有任务，既然有任务等着被执行，那就不能阻塞在操作系统上了，所以需要立即返回。
- 5. 有定时任务可以执行了
     
如果上面的条件都不成立，那么代码就会继续向下执行，后面这一部分代码就是规避 JDK 中关于空轮询 bug 的代码。

### 4.JDK 空轮询的 BUG

在说 Netty 是如何避免 JDK 的空轮询的 BUG 之前，先来说下什么是 JDK 的空轮询 BUG，JDK 的官方是这样定义描述的。

- [JDK-6670302 : (se) NIO selector wakes up with 0 selected keys infinitely](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6670302 "JDK-6670302 : (se) NIO selector wakes up with 0 selected keys infinitely")
- [JDK-6403933 : (se) Selector doesn't block on Selector.select(timeout) (lnx)](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6403933 "JDK-6403933 : (se) Selector doesn't block on Selector.select(timeout) (lnx)")

> This is an issue with poll (and epoll) on Linux. If a file descriptor for a connected socket is polled with a request event mask of 0, and if the connection is abruptly terminated (RST) then the poll wakes up with the POLLHUP (and maybe POLLERR) bit set in the returned event set. The implication of this behaviour is that Selector will wakeup and as the interest set for the SocketChannel is 0 it means there aren't any selected events and the select method returns 0.

什么意思呢？在部分 Linux 的 2.6 的 kernel 中，poll 和 epoll 对于突然中断的连接 socket 会对返回的 eventSet 事件集合置为 POLLHUP，也可能是 POLLERR，eventSet 事件集合发生了变化，这就可能导致 Selector 会被唤醒，这样就会让 select()方法立即返回，不会阻塞。而通常 select()方法是写在一个 while(true)循环中的，这样就会导致应用程序一直不停地执行 select()，也就是导致 CPU 空转，**CPU 使用率最终会飙升到 100%**

这是与操作系统机制有关系的，JDK 虽然仅仅是一个兼容各个操作系统平台的软件，但很遗憾在 JDK5 和 JDK6 最初的版本中（严格意义上来将，JDK 部分版本都是），这个问题并没有解决，而将这个帽子抛给了操作系统。虽然 JDK 官方说在 JDK1.7 版本中已经解决了这个问题，但是实际上，空轮询的 bug 依旧存在，在 1.7 中只是降低了发生空轮询 bug 的概率。

事实上，netty 也没有解决空轮询的问题，因为这是操作系统自身的机制问题，netty 是无法解决操作系统的问题的。netty 只是巧妙的避开了空轮询的问题，让发生空轮询的 bug 的概率为 0。（严格意义上来讲，不是解决，而是规避）

那么 netty 是如何在代码层面来规避空轮询的 bug 的呢？答案就在如下代码中（只贴出了 select(oldWakenUp)方法中处理空轮询相关部分的代码）。

```java
for (; ; ) {
    // ...
    selectCnt ++;
    // 省略其他和空轮询无关的代码
    long time = System.nanoTime();
    // 下面这一行代码等价于： time - currentTimeNanos >= TimeUnit.MILLISECONDS.toNanos(timeoutMillis)
    // 什么意思呢？当前时间 - 运行之前的时间，如果大于 select阻塞的时间，这就说明进行了select的阻塞操作，没有进行JDK的空轮询
    if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
        selectCnt = 1;
    } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
            selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
        // 如果selectCnt的值大于等于512（这个数值可配置），说明系统可能进行了空轮训，因此创建新的selector，并用新的selector来替换之前的selector
        selector = selectRebuildSelector(selectCnt);
        selectCnt = 1;
        break;
    }
    currentTimeNanos = time;
}
```

可以看到，每进行一次 for 循环，**selectCnt** 就会加 1，selectCnt 变量就是用来记录发生 for 循环次数的。

在代码中，会先判断当前时间减去运行之前的时间，如果大于 select 阻塞的时间，这就说明进行了 select 的阻塞操作，没有进行 JDK 的空轮询，那么就将重新 selectCnt 计为 1。否则，就是**有可能**发生了一次空轮询。

因此接着判断 selectCnt 的值是否大于 **SELECTOR_AUTO_REBUILD_THRESHOLD** 常量的值，该常量的含义是发生空轮询是 CPU 空转次数的阈值。默认值是 512，可以通过如下属性来配置。

```properties
io.netty.selectorAutoRebuildThreshold
```

如果 CPU 连续空转的次数超过了该阈值，那么 netty 就认为当前系统出现了空轮询的 BUG，那么它就会重新创建一个新的多路复用器，然后替换当前的多路复用器 selector。

**selectRebuildSelector(selectCnt)** 方法就是用来重新创建新的多路复用器的，它最终会调用到如下方法。

```java
private void rebuildSelector0() {
    final Selector oldSelector = selector;
    final SelectorTuple newSelectorTuple;

    if (oldSelector == null) {
        return;
    }
    try {
        // 重新创建新的Selector
        newSelectorTuple = openSelector();
    } catch (Exception e) {
        return;
    }
    int nChannels = 0;
    for (SelectionKey key: oldSelector.keys()) {
        Object a = key.attachment();
        try {
            if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
                continue;
            }
            int interestOps = key.interestOps();
            // 取消key
            key.cancel();
            // 重新将channel注册到新的多路复用器上
            SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
            // 绑定新SelectionKey与channel的关系
            if (a instanceof AbstractNioChannel) {
                // Update SelectionKey
                ((AbstractNioChannel) a).selectionKey = newKey;
            }
            nChannels ++;
        } catch (Exception e) {
            // 异常处理
        }
    }
    //赋值最新selector
    selector = newSelectorTuple.selector;
    unwrappedSelector = newSelectorTuple.unwrappedSelector;
    try {
        // 关闭旧的selector
        oldSelector.close();
    } catch (Throwable t) {
        // 异常
    }
}
```

该方法的功能就是重新创建一个新的多路复用器 Selector。由于旧的多路复用器上注册了很多 channel，以及 channel 上又绑定了 SelectionKey。所以要用新的多路复用器替换旧的多路复用器之前，还需要将旧的 selector 上的 channel 和 selectionKey 全部重新注册到新的 selector 上。因此 rebuildSelector0()方法还做了以下几项工作。

- 1. 从旧的多路复用器 Selector 上取出所有的 SelectionKey，
- 2. 取消每个 SelectionKey 在旧的 Selector 上注册的事件
- 3. 然后将每个 SelectionKey 对应的 channel 重新注册到新的 selector 上
- 4. 重新绑定新的 SelectionKey 和 channel 的关系
- 5. 关闭旧的 selector

至此，netty 就巧妙地规避了 JDK 中空轮询的 bug。

### 5.处理网络 IO(processSelectedKeys())

当 select(wakenUp)方法执行完成后，就会去处理从操作系统中轮询到的网络 IO 事件。处理的网络 IO 事件的方法就是 processSelectedKeys()。无论 netty 有没有开启对 Selecor 的优化，最终都是循环调用如下方法进行网络 IO 事件的处理，精简后的源码如下。

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    // 判断当前的SelectionKey是有有效，如果无效，就不进行网络IO的数据处理了
    if (!k.isValid()) {
        // 省略部分代码
        return;
    }
    try {
        int readyOps = k.readyOps();
        // 如果是OP_CONNECT事件，则让当前channel完成连接事件
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            // 完成连接
            unsafe.finishConnect();
        }
        // 如果是写事件，则先进行写事件，因为先进行写事件，可以在写事件结束后，释放一些内存
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // 写数据
            ch.unsafe().forceFlush();
        }
        // 读事件或者接收事件或者是0
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // 读数据
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

首先判断当前的 SelectionKey 是否是有效的，如果无效，则不会进行网络 IO 的数据处理，而是直接返回。

然后判断是否是 **OP_CONNECT** 事件，即连接事件。如果是连接事件，则会让 channel 先完成连接事件。

接着先判断是否是 **OP_WRITE** 事件，即写事件。如果是写事件，则通过调用 unsafe 的 forceFlush()方法进行数据的写操作。

最后，如果是 **OP_READ** 或者 **OP_ACCEPT** 或者 **readyOps 为 0**，即读事件或者接收事件或者是没有事件，则通过调用 unsafe 的 read()数据进行读数据操作。（本质上：**OP_ACCEPT**（新连接接入）事件也是一种读操作，服务端 channel 从通道中读取是哪个客户端来接入）

> 对于服务端 channel 而言，这里的 unsafe 是 **NioMessageUnsafe** 类的实例；对于客户端 channel 而言，这里的 unsafe 是 **NioSocketChannelUnsafe** 的实例。关于 unsafe 的读数据操作和写数据操作，不是本文的重点，后续会有专门的文章去解读，敬请关注。

这里可能会存在两个疑惑。

- 1. 为什么是先判断 **OP_WRITE** 事件，而不是先判断 **OP_READ** 等事件呢？这是因为如果存在写事件，那么先进行写事件，可以在写事件结束后，可以释放一些内存，这对系统性能更优。
- 2. 为什么 readyOps 等于 0 时，也需要进行 read()操作？因为 readyOps 为 0，可能是 JDK 处理错误，如果不对 readyOps 为 0 进行处理，那么该 selectionKey 就会一直被轮询出来，这就会导致自旋。而如果对其进行调用了 read()方法，那么后面就不会从操作系统中轮询出当前的 SelectionKey 了。

### 6.处理任务(runAllTasks())

处理完网络 IO 事件后，就会执行任务队列中的任务，由于 **ioRatio** 默认为 **50**，所以最后调用的是 runAllTasks(timeoutNanos)方法，分配给执行任务的时间与分配给处理网络 IO 的时间是相等的。该方法的源码如下。

```java
protected boolean runAllTasks(long timeoutNanos) {
    // 将定时任务中到期的任务添加到taskQueue中
    fetchFromScheduledTaskQueue();

    Runnable task = pollTask();
    if (task == null) {
        // 如果没有获取到任务，就执行tailQueue队列中的任务，最后直接返回
        afterRunningAllTasks();
        return false;
    }
    // 计算本次任务循环的截止时间
    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    // 循环执行任务
    for (; ; ) {
        // 真正执行任务
        safeExecute(task);
        runTasks++;
        // 如果连续执行了64次任务，就判断一下是否超时
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }
        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }
    // 执行tailQueue中的任务
    afterRunningAllTasks();
    // 更新NioEventLoop中最近一次执行任务的时间
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

首先**fetchFromScheduledTaskQueue()** 会将定时任务队列中到期的任务取出，然后放入到 taskQueue 中。如果任务队列中获取不到任务，就直接返回。如果有任务，就计算一下执行任务的截止时间，接着开始在无限 for 循环中循环取任务、执行任务了，直到任务全部被执行完或者超时了。最后会更新一下 nioEventLoop 中最近执行任务的时间。

**safeExecute(task)** 是真正执行任务的方法，因为队列中的任务都是 Runnable 类型，所以 safeExecute(task)实际上就是调用 Runnable 的**run()** 方法。

在循环执行任务的过程中，如何判断超时呢？netty 不是每执行一次任务就判断一下是否超时，而是连续执行 64 次任务后才判断一次是否超时。为什么要这样做呢？因为在判断超时时，会调用系统的**nanoTime()** 方法，每次调用该方法，实际上也是一种浪费性能的操作。尤其是当系统负载较高时，获取系统时间的方法会是一个耗时的操作。

### 7.总结

本文主要分析了 NioEventLoop 中 run()方法的执行步骤，它会先从操作系统中轮询出所有的网络 IO 事件，然后进行网络 IO 事件的处理，最后进行任务队列中任务的处理，任务包括普通任务和定时任务。在从操作系统中轮询网络 IO 事件时，netty 通过对阻塞时间以及轮询次数的判断，来判断是否发生了空轮询的 BUG，如果是，就采用重新创建一个多路复用器的思路，巧妙的规避了 JDK 中空轮询的 BUG。

最后关于网络 IO 数据的读写操作，会在后面新连接接入的篇章中详细说明。
