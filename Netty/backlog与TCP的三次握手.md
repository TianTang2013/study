> 扫描下方二维码或者微信搜索公众号`菜鸟飞呀飞`，即可关注微信公众号，阅读更多`Spring源码分析`、`Java并发编程`和`Netty源码系列`文章。

![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144)

### 前言

前段时间在学习 netty 源码的时候，遇到了一个知识点：在服务端套接字创建的过程中，可以通过 **option()** 方法为服务端 channel 设置 TCP 相关的参数，例如：**ChannelOption.SO_BACKLOG**，该参数就是设置 tcp 的 **backlog** 属性的值（示例代码如下）。实际上不仅是在 netty 中可以设置，在 JDK 原生的 Socket 中也可以设置。那么 backlog 到底是个什么东西呢？它的作用是什么呢？

```java
// netty中设置backlog参数的示例代码
ServerBootstrap serverBootstrap = new ServerBootstrap();
// 将backlog设置为10
serverBootstrap.option(ChannelOption.SO_BACKLOG,10);
```

### TCP 三次握手

要想搞清楚 backlog 的含义以及用途，就不得不从 TCP 的三次握手开始说起了。

众所周知，TCP 是**面向连接**的，它是**靠谱**的传输协议。UDP 是面向**无连接**的，是**不靠谱**的传输协议。

那么面向连接又是什么意思呢？所谓的连接，并不是指通信的两端之间背后通过插一根物理网线而连接起来的，而是客户端和服务端之间通过**创建相应的数据结构**，来维护双方的状态，并通过这样的数据结构来保持面向连接的特性。所谓的连接就是客户端和服务端之间**数据结构状态的协同**，如果状态能对应的上，那么就说服务端和客户端之间创建了。

谈到 TCP 协议，大家都知道 TCP **三次握手、四次挥手**的特性，那么在三次握手之间到底干了哪些事情呢？和上面提到的数据结构又有什么关系呢？（关于 TCP 的四次挥手本文先不分析）

TCP 是一个靠谱的传输协议，而三次握手就是 TCP 用来保证靠谱的手段。下图是 TCP 三次握手的示意图。

![TCP三次握手](https://user-gold-cdn.xitu.io/2020/1/16/16fae24415af6ff9?w=1692&h=1232&f=png&s=319283)

首先，客户端和服务端都处于关闭状态，即 **CLOSED** 状态。

当服务端调用操作系统的 **bind()** 函数和 **listen()** 函数后，服务端就会处于 **LISTENING** 状态。

当客户端调用 **connect()** 函数主动向服务端发起连接时，就开始 TCP 的三次握手过程了。第一步：客户端会先向服务端发送一个 SYN 包，告诉服务端我要来连接了，发完这个 SYN 包后，客户端就会处于 **SYN_SENT** 状态。

第二步：当服务端收到客户端发来的 SYN 包后，需要返回给客户端一个 SYN 包以及确认收到 SYN 的 ACK 包，告诉客户端，我知道你要来连接了，发完这个包后，服务端就处于 **SYN_RCVD** 状态了。

第三步：当客户端收到了服务端发来的包后，客户端还需要告诉服务端，我收到了你发来的 SYN 包，即服务端的 SYN 的 ACK 确认包。发完这个包后，客户端就处于 **ESTABLISHED** 状态了。当服务端收到客户端的 ACK 包后，也会处于 **ESTABLISHED** 状态。

当双方均处于 **ESTABLISHED** 状态后，就可以进行数据传输了。

前面提到了，当创建 TCP 连接时，双方需要维护一定的数据结构来保证面向连接的特性。这些数据结构当中，就包含了上图中的两个数据结构：**半连接队列（syns queue）** 和**全连接队列（accept queue）**。

这两个队列干什么用的呢？当客服端发来一个 SYN 包后，服务端会将这个连接保存到 **syns queue** 队列当中，由于此时 TCP 连接还没有完全建立完成，因此称 **syns queue** 是一个**半连接**的队列。当服务端收到客户端发来的 ACK 包后，会将这个连接从 **syns queue** 队列中移到 **accept queue** 队列中，此时连接已经建立完成，所以称之为**全连接队列**。处于全队列中的连接，可以通过操作系统的 **accept()** 方法为其创建对应的套接字，这样服务端和客户端就能开始进行数据的传输了。

另外在 TCP 的头部协议中，有一个 **32 位长度的包序号**，这个包序号是用来**保证包的顺序**的，解决乱序问题。因为 TCP 是面向连接的嘛，需要保证每个连接和发送的包之间的对应关系，这样才不会混乱，客户端和服务端之间就能区分这个包是哪个连接发送过来的。这个包序号用示意图来表示，就是如下图所示。

![三次握手+包序号](https://user-gold-cdn.xitu.io/2020/1/16/16fae249d1465b42?w=1638&h=1256&f=png&s=319266)

第一次，客户端发送 SYN 包时，需要指定一个包序号 x。当服务端收到后，会返回一个 ACK 包，这个 ACK 的序号就是 x+1，表示的是我回应的是包序号是 x 的 SYN 包，同时服务端还会发送一个包序号为 y 的 SYN 包；当客户端收到服务端发送来需要为 y 的 SYN 包后，会回复一个 ACK 包，序号为 y+1，表示的是确认收到了需要为 y 的 SYN 包。

### backlog 与 TCP 的三次握手

说了这么多，backlog 和 TCP 到底有什么关系呢？在上面提到了两个队列：半连接队列（syns queue）和全连接队列（accept queue）。这两个队列是用来存放连接的，既然是队列，就会有大小限制，而 **backlog** 这个参数就是用来**限制全连接队列的大小的**。操作系统中有一个内核参数：**somaxconn**，默认是 **128**（和操作系统的版本有关系）。可以通过如下命令查看：

```java
sysctl -a|grep somaxconn
```

![somaxconn](https://user-gold-cdn.xitu.io/2020/1/16/16fae25262b04626?w=548&h=46&f=png&s=6356)

backlog 和 somaxconn 两者中的较小者：**min(backlog,somaxconn)**，就是全连接队列的最大容量限制。backlog 的值就是我们在代码中创建套接字时指定的值，如果用户没有指定，**Java 中默认是 50**。

与全连接队列类似，半连接队列也要一个容量限制：**max(64,/proc/sys/net/ipv4/tcp_max_syn_backlog)**。

**tcp_max_syn_backlog** 是操作系统的一个内核参数，可以通过如下命令查看具体的值。半连接队列的容量上限是 64 和 tcp_max_syn_backlog 参数值中的**较大者**。

```java
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
或者
sysctl -a|grep tcp_max_syn_backlog
```

在 TCP 的三次握手中，当客户端向服务端发起第一次握手时，即：发送第一个 SYN 包时，如果服务端的半连接队列没有满，那么就会将当前连接添加到半连接队列中。如果半连接队列已经满了，那么服务端就会直接丢弃这个 SYN 包。此时客户端等一段时间后，一直没有收到服务端发送回来的 ACK 包，所以客户端就会重发 SYN 包，如果服务端半连接队列还是已满状态，所以依旧会将 SYN 包丢弃，那么过一会，客户端还会重发 SYN 包，直到收到服务端发送来的 ACK 包**或者**重发次数达到上限，客户端才会停止重发。**SYN 包重发次数上限也是一个内核参数：tcp_syn_retries**。可以通过如下命令查看 **tcp_syn_retries**。如果客户端重发 SYN 包次数达到上限后，就会出现 **connection timeout** 异常。

```java
cat /proc/sys/net/ipv4/tcp_syn_retries
或者
sysctl -a|grep tcp_syn_retries
```

另外需要说明的是，**SYN-Flood** 攻击就是利用了第一次握手中**半连接队列的容量限制**的特点，恶意攻击者会不停的伪造 SYN 包，发送给服务端，此时连接是处于半连接状态。由于攻击者不定=地发，这样导致半连接队列很快被撑满。当正常的连接来连接服务端时，由于半连接队列已满，因此就无法完成 TCP 的三次握手操作，正常连接就无法连接到服务端。在操作系统中提供了一个内核参数：**tcp_syncookies**，该参数可以用来防止 SYN—Flood 攻击。可以通过如下命令来查看 tcp_syncookies 参数的值。

```java
cat /proc/sys/net/ipv4/tcp_syncookies
或者
sysctl -a|grep tcp_syncookies
```

当 **tcp_syncookies 的值为 1 的时候，表示开启 tcp_syncookies**，可以防止 SYN—Flood 攻击；当为 0 的时候表示禁止使用 syn cookies。虽然开启 tcp_syncookies 可以防止 SYN—Flood 攻击，但是由于 syn cookies 使用了加密的算法，会特别耗资源，对于负载较重的服务器，会额外增加压力，因此是否开启 tcp_syncookies 需要根据实际的业务来进行选择。

当在第三次握手时，服务端接收到客户端发来的 ACK 包后，在正常情况下(**全连接队列没有满**)，会把当前连接从半连接队列中移到全连接队列中。但是对于非正常情况，也就是全连接队列已经满了，服务端此时又会怎么处理呢？此时又有一个内核参数：**tcp_obort_on_onerflow**，服务端会根据这个内核参数来决定怎么处理。可以通过如下命令行来查看 **tcp_obort_on_onerflow** 参数的值。

```java
cat /proc/sys/net/ipv4/tcp_abort_on_overflow
 或者
sysctl -a|grep tcp_abort_on_overflow
```

当 **tcp_abort_on_overflow** 的值为 **0** 时，在全连接队列满了以后，服务端会直接丢弃掉客户端传来的 ACK 包。由于服务端将这个 ACK 包丢弃了，那么服务端会认为自己给客户端发送的 **SYN+ACK** 包一直没有响应，因此服务端会等待一会以后重新发送 SYN+ACK 包给客户端，这个重试次数也有一个上限，可以通过内核参数 **tcp_synack_retries** 来修改。通过如下命令可以查看 **tcp_synack_retries** 参数的值。如果服务端在重试期间，客户端由于设置的超时时间较短，TCP 三次握手没有完成，就会出现 connection timeout 异常。

```java
cat /proc/sys/net/ipv4/tcp_synack_retries
 或者
sysctl -a|grep tcp_synack_retries
```

当 **tcp_abort_on_overflow** 的值为 **1** 时，在全连接队列满了以后，服务端会直接向客户端发送一个 **RST** 通知，即 reset 包，表示**废除当前的握手过程**。此时客户端收到 RST 通知后就会出现 **connection reset by peer** 异常。

### 工具

知道了上面的理论，那么当出现全连接队列满时我们又该如何去排查呢？我们可以通过两个命令行工具来查看网络相关的性能指标：**netstat** 和 **ss**。

ss 命令的用法如下。

```properties
# -n表示显示数字地址和端口(不是名字)
# -l表示显示处于LISTEN状态的套接字
# -t表示显示TCP协议栈的信息
# -p表示显示进程信息
ss -lnt
```

![ss命令结果](https://user-gold-cdn.xitu.io/2020/1/16/16fae2bcaf4c46e8?w=3026&h=218&f=png&s=45801)

netstat 命令的用法如下。

```properties
# -n表示显示数字地址和端口(不是名字)
# -l表示显示处于LISTEN状态的套接字
# -p表示显示进程信息
netstat -lnp
```

关于 netstat 和 ss 命令的其他参数和用法，可以通过 **man** 命令或者**netstat -h**和**ss -h**去查看。

可以发现，netstat 和 ss 命令的查询结果类似。对于统计结果中，有两列需要特别注意一下：**Recv-Q 和 Send-Q。**

对于套接字处于**非 LISTEN 状态**的情况，**RECV-Q** 表示的是**接收队列**中积压的数据的字节数，即接收到了客户端发来的数据，但是还没有被读到具体的应用进程中；**Send-Q** 表示的是**发送队列**中积压数据的字节数，即数据还没有被接收方确认的消息的字节数。正常情况下，这两个数字应该为 0，如果一直不为 0，则表示接收队列或者发送队列中有数据积压了，CPU 可能处理过慢，这个时候就需要警惕了，防止出现异常。

对于处于 **LISTEN 状态**的套接字，**Recv-Q** 表示的是处于**全连接队列中的连接个数**，**Send-Q 表示的全连接队列的最大大小**，也就是 **min(backlog,somaxconn)** 的结果。当 Recv-Q 的值超过 Seend-Q 时，说明全连接队列已经满了，这个时候就会触发执行 **tcp_abort_on_overflow** 内核参数控制的逻辑。

这里需要额外提醒的是，当我使用 man 命令去看 netstat 的文档的时候，文档中给出的关于 Recv-Q 和 Send-Q 的解释，对于处于 LISTEN 状态的套接字，通过 netstat 命令统计出来的数据，**Recv-Q 指的是 syns queue 队列中的连接个数，即半连接队列的连接个数，Send-Q 指的是半连接队列大小的最大值，即：max(64,tcp_max_syn_backlog)的值**。然而当我使用命令 **netstat -lnp** 去查看处于 LISTEN 套接字的结果时，Recv-Q 和 Send-Q 始终返回的是 0。再使用 **ss -lnt** 命令去查看处于 LISTEN 套接字的结果时，Send-Q 返回的是我配置的 backlog 和 somaxconn 中的较小值，**这从侧面说明了，通过 ss 命令看到的 Recv-Q 和 Send-Q 描述的全连接队列相关的信息**。

然而这个实验结果，与 netstat 的官方文档是冲突的(官方文档中解释的是这两个指标描述的是半连接队列相关的信息)，在网上查了很多资料，两种说法都有，也和操作系统的版本有关系，我在两个不同的版本中，看到的文档解释的也不一样。最终我个人觉得是可能是 **ss 和 netstat 命令，关于这两个指标，他们描述的信息是不一样的，对于 netstat 命令查询出来的 Recv-Q 和 Send-Q 信息，和全连接队列、半连接队列没有任何关系。所以最终的结论是：对于处于 LISTEN 状态的套接字，使用 ss 命令得到的 Recv-Q 和 Send-Q 描述的是全连接队列相关的信息（非 LISTEN 状态的套接字，这两个指标描述的是发送队列和接收队列中，积压的数据量）**。

![netstat文档](https://user-gold-cdn.xitu.io/2020/1/16/16fae292e726248e?w=2982&h=248&f=png&s=60995)

当出现 **connection reset by peer** 等 TCP 连接异常时，我们可以通过如下命令来确认一下是否是三次握手中出现了半连接队列和全连接队列已满的问题。

```java
# -s参数表示查询时统计信息
# grep表示通过管道符过滤
# -i表示忽略大小写
# listen是我们要过滤的单词
netstat -s |grep -i listen
```

例如当查询如下结果时，就表明出现了全连接队列出现了溢出现象，**如果这个 overflow 的次数持续增加，这可能就说明 backlog 或者 somaxconn 的值配置小了，可以尝试去修改这两个参数的值**。

![overflow溢出图](https://user-gold-cdn.xitu.io/2020/1/16/16fae27753ca145c?w=930&h=88&f=png&s=17605)

另外还可以借助工具 **Wireshark+tcpdump** 工具来进行抓包分析。

### 总结

最后总结一下，本文从 backlog 参数出发，详细分析了 TCP 三次握手过程，以及在握手过程中涉及到的两个队列：半连接队列和全连接队列，当完成第一次握手后，会将连接放入到半连接队列中，当完成第三次握手后，会将连接放入到全连接队列中，当调用操作系统的 accept()函数时，会将连接从全连接队列中取出，从而创建一个客户端套接字，供后面的数据读写。

其中半连接队列的大小等于 **max(64,tcp_max_syn_backlog)**，**tcp_max_syn_backlog** 是一个内核参数。全连接队列的大小等于 **min(backlog，somaxconn)**，backlog 是我们创建套接字时通过代码指定的，java 中默认是 50，somaxconn 也是一个内核参数，默认是 128，不同的操作系统版本会有一定的区别。

另外内核参数 **tcp_syn_retries** 控制的是 SYN 包重试的次数(第一次握手中的 SYN 包)，**tcp_synack_retries** 控制的是 SYN+ACK 包重试的次数(第二次握手中的 SYN+ACK 包)。**tcp_syncookies** 参数设置为 1 可以防止 SYN-Flood 攻击，但开启后会占用服务端资源，需要酌情考虑。

最后，在第三次握手中，全连接队列满了以后，会根据内核参数 **tcp_abort_on_overflow** 来决定怎么处理连接，当设置为 0 时表示的是直接丢弃客户端发送来的 ACK 包，当设置为 1 时表示的是服务端会发送 RST 通知给客户端，客户端会出现 **connection reset peer** 异常。

关于内核参数的查看，可以使用 **sysctl -a** 查看所有内核参数，也可以在**/proc/sys/net/ipv4** 目录下查看网络相关的内核参数。通过 **ss 或者 netstat** 命令可以查看网络相关的统计值，这两个工具可以帮助我们快速定位线上问题。

另外可以通过修改**/etc/sysctl.conf** 文件来修改相关内核参数，然后使用 **sysctl -p** 使修改的参数生效。

### 推荐
* [如何从BIO演进到NIO，再到Netty](https://mp.weixin.qq.com/s/zcrclRhgK015FSjtuMZoCg)
* [Netty源码分析系列之Reactor线程模型](https://mp.weixin.qq.com/s/-aqIwWXYBkTy0dCNy0FtlA)
* [Netty源码分析系列之服务端Channel初始化](https://mp.weixin.qq.com/s/mwlzDmYZP6J_2Fr6TUvxKw)
* [Netty源码分析系列之服务端Channel注册](https://mp.weixin.qq.com/s/Tw0541dQlPKmEj8VinPucQ)
* [Netty源码分析系列之服务端Channel的端口绑定](https://mp.weixin.qq.com/s/p3ZouaQ_x6NgyVMWw4tJ0g)
* [Netty源码分析系列之NioEventLoop的创建与启动](https://mp.weixin.qq.com/s/qBRY8zl_LBrn3acCaRmEbg)
* [Netty源码分析系列之NioEventLoop的执行流程](https://mp.weixin.qq.com/s/RId4bKC0lwL_HofyqDV61Q)
* [Netty源码分析系列之新连接的接入](https://mp.weixin.qq.com/s/mKQXgVVhgI7Tbi9GQ4mGiQ)
* [Netty源码分析系列之TCP粘包、半包问题以及Netty是如何解决的](https://mp.weixin.qq.com/s/IihycUIcdTuigEmBbbkfBg)
* [Netty源码分析系列之常用解码器（上）](https://mp.weixin.qq.com/s/K-jzX1Vm5SYJEIrSTazk_g)
* [Netty源码分析系列之常用解码器（下）——LengthFieldBasedFrameDecoder](https://mp.weixin.qq.com/s/xoMZr6j6QPwGCeqodX49ww)
* [Netty源码分析系列之writeAndFlush()上](https://mp.weixin.qq.com/s/BBDlOJtJeRIC3S2ioJVknw)
* [Netty源码分析系列之writeAndFlush()下](https://mp.weixin.qq.com/s/XkTpxPz4V7NXNKbxeu7PhA)


![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144)
