> 扫描下方二维码或者微信搜索公众号`菜鸟飞呀飞`，即可关注微信公众号，阅读更多`Spring源码分析`、`Java并发编程`、`Netty源码系列`和`MySQL工作原理`文章。

![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144) 

### 1. 开篇

MySQL 在查询数据时，对于 InnoDB 存储引擎而言，会先将磁盘上的数据以页为单位，先将数据页加载进内存，然后以缓存页的形式存放在**Buffer Pool**中。Buffer Pool 是 InnoDB 的一块内存缓冲区，在 MySQL 启动时，会按照配置的缓存页的大小，将 Buffer Pool 缓存区初始化为许多个缓存页，默认情况下，缓存页大小为 16KB。

> 为了方便理解，对于磁盘上的数据所在的页，叫做数据页，当加载进 Buffer Pool 中后，叫做缓存页，这两者是一一对应的

在 MySQL 启动初期，Buffer Pool 中的这些缓存页都是处于空闲状态，也就是还没有被使用，而随着 MySQL 的运行时间越来越长，这些缓存页渐渐地都被使用了，用来存放从磁盘上加载的数据页，导致空闲的缓存页就越来越少。当某一时刻，所有空闲的缓存页都被使用了，那么这个时候，从磁盘加载到内存中的数据页该怎么办呢？

### 2. LRU算法

既然没有空闲缓存页了，而又想使用缓存页，那么最简单的办法就是淘汰一个缓存页。应该淘汰谁呢？当然是淘汰那个最近一段时间最少被访问过的缓存页了，这种思想就是典型的 LRU 算法了。

LRU 是 Least Recently Used 的简写，这个算法的实现就是淘汰最久未使用的数据，它通过维护一个链表，每当访问了某个数据时，就将这个数据加入到链表的头部，如果数据本身存在于链表中，那么就将数据从链表的中间移动到链表的头部，这样最终下来，在链表尾部的数据一定就是最久未被使用的数据了，因此可以将其淘汰。

将 LRU 算法应用到缓存页的淘汰策略上，那么就是在 InnoDB 存储引擎层内部，维护了一个链表，这些链表中的元素存储的就是指向缓存页的指针。

在 MySQL 启动的时候，这个链表为空。MySQL 启动以后，在进行数据查询时，InnoDB 会先判断要查询的数据所在的数据页，是否存在于 Buffer Pool 的缓存页当中，如果不存在，就从磁盘中读取对应数据页，存放到 Buffer Pool 一个空闲的缓存页当中，然后将这个缓冲页放入到链表的头部；如果要查询的数据已经存在于 Buffer Pool 当中了，那么就将对应的缓存页从链表的中间移动到链表头部。

这样随着 MySQL 的运行，空闲的缓存页越来越少，LRU 链表越来越长，直到某一时刻，剩余的空闲缓存页数为 0，当需要申请一个新的空闲缓存页的时候，就需要淘汰一个缓存页了，此时只需要把链表尾部的那个缓存页刷入到磁盘，然后清空缓存页里面的数据，这样就空余出一个新的缓存页了。

### 3. LRU 链表带来的问题

咋一看，似乎 LRU 链表完美解决了缓存页淘汰的问题。但理想很丰满，现实却很骨感，如果直接使用这种 LRU 算法的话，在 MySQL 中就会存在很大的问题。

#### 3.1 全表扫描

在 MySQL 中经常会出现全表扫描，一种是开发人员对索引的使用不当导致的，一种是业务如此，无法避免。

当出现全表扫描时，InnoDB 会将该表的数据页全部从磁盘文件加载进缓存页中，这些缓存页会被加入到 LRU 链表中。如果进行全表扫描的对象是一张非常大的表，可能是几十 GB 的数据，而且这张表记录的是类似于账户流水、操作日志等使用不频繁的数据，这个时候如果 LRU 链表已经满了，现在我们就要淘汰一部分缓存页，腾出空间来存放全表扫描出来的数据。这样就会因为全表扫描的数据量大，需要淘汰的缓存页多，导致在淘汰的过程中，极有可能将需要频繁使用到的缓存页给淘汰了，而放进来的新数据却是使用频率很低的数据，甚至是这一次使用之后，后面几乎再也不用，如操作日志等。

最终导致的现象就是，当我们在对这些使用不频繁的大表进行全表扫描之后，在一段时间内，Buffer Pool 缓存的命中率明显下降，SQL 的性能也明显下降，因为常用的缓存页被淘汰了，再进行查询时，需要从重新磁盘读取，发生磁盘 IO，性能下降。所以，如果 MySQL 只是简单的使用 LRU 算法，那么碰到全表扫描时，就会存在性能下降的问题，甚至在高并发场景下，成为性能瓶颈。

#### 3.2 预读

预读是 InnoDB 引擎的一个优化机制，当你从磁盘上读取某个数据页，InnoDB 可能会将与这个数据页相邻的其他数据页也读取到 Buffer Pool 中。

InnoDB 为什么要这样做呢？因为从磁盘读取数据时发生的磁盘 IO 是随机 IO，性能差。当你读取某一个数据页时，InnoDB 会猜测你可能也需要下一个数据页的数据，如果一次能连着读取多个数据页，那么对其他的数据页而言，这是顺序读取（节省了寻道时间、磁头旋转时间），相对较快，那这样就能提升一点性能。

在两种情况下会触发预读机制：

1. 顺序的访问了磁盘上一个区的多个数据页，当这个数量超过一个阈值时，InnoDB 就会认为你对下一个区的数据也感兴趣，因此触发预读机制，将下一个区的数据页也全都加载进 Buffer Pool。这个阈值由参数 **innodb_read_ahead_threshold** 控制，默认为 56。 可以通过如下命令查看：

```sql
show variables like 'innodb_read_ahead_threshold';
```

2. 如果 Buffer Pool 中已经缓存了同一个区数据页的个数超过 13 时，InnoDB 就会将这个区的其他数据页也读取到 Buffer Pool 中。这个开关由参数 **innodb_random_read_ahead** 控制，默认是关闭的。

```sql
show variables like 'innodb_random_read_ahead';
```

如果 MySQL 仅是简单的使用 LRU 算法，那么预读机制和全表扫描带来的问题类似，预读机制会将其他的数据页也加载进内存，当 LRU 链表满时，可能将我们频繁访问的缓存页给淘汰，从而导致性能下降。

### 4. 冷热分离

实际上，MySQL 确实没有直接使用 LRU 算法，而是在 LRU 算法上进行了优化。

从上面的全表扫描和预读机制的问题分析中，我们可以看到，根本原因就是从磁盘上新读取到的数据页，在加载进 Buffer Pool 时，可能将我们频繁访问的数据给淘汰，也就是出现了冷热数据的现象。因此，MySQL 的优化思路就是：对数据进行冷热分离，将 LRU 链表分成两部分，一部分用来存放冷数据，也就是刚从磁盘读进来的数据，另一部分用来存放热点数据，也就是经常被访问到数据。

其中，存放冷数据的区域占这个 LRU 链表的多少呢？这由参数 **innodb_old_blocks_pct** 控制，默认是 37%（约八分之三）。冷热分离的 LRU 链表示意图如下（图片来自于MySQL官方文档）。

```sql
show variables like 'innodb_old_blocks_pct';
```

【图】


优化过后的 LRU 链表，又是如何进行数据页的存放的呢？

**_当从磁盘读取数据页后，会先将数据页存放到 LRU 链表冷数据区的头部，如果这些缓存页在 1 秒之后被访问，那么就将缓存页移动到热数据区的头部；如果是 1 秒之内被访问，则不会移动，缓存页仍然处于冷数据区中。1 秒这个数值，是由参数 innodb_old_blocks_time 控制。_**

```sql
show variables like 'innodb_old_blocks_time';
```

当遇到全表扫描或者预读时，如果没有空闲缓存页来存放它们，那么将会淘汰一个数据页，而此时淘汰地是冷数据区尾部的数据页。冷数据区的数据就是不经常访问的，因此这解决了误将热点数据淘汰的问题。如果在 1 秒后，因全表扫描和预读机制额外加载进来的缓存页，仍然没有人访问，那么它们会一直待在冷数据区，当再需要淘汰数据时，首先淘汰地就是这一部分数据。

至此，基于冷热分离优化后的 LRU 链表，完美解决了直接使用 LRU 链表带来的问题。

### 5. LRU 链表的极致优化

实际上，MySQL 在冷热分离的基础上还做了一层优化。

当一个缓存页处于热数据区域的时候，我们去访问这个缓存页，这个时候我们真的有必要把它移动到热点数据区域的头部吗？

从代码的角度来看，将链表中的数据移动到头部，实际上就是修改元素的指针指向，这个操作是非常快的。但是为了安全起见，在修改链表的时候，我们需要对链表加上锁，否则容易出现并发问题。

当并发量大的时候，因为要加锁，会存在锁竞争，每次移动显然效率就会下降。因此 MySQL 针对这一点又做了优化，如果一个缓存页处于热数据区域，且在热数据区域的前 1/4 区域（注意是热数据区域的 1/4，不是整个链表的 1/4），那么当访问这个缓存页的时候，就不用把它移动到热数据区域的头部；如果缓存页处于热数据的后 3/4 区域，那么当访问这个缓存页的时候，会把它移动到热数据区域的头部。

### 6. 生产上的 MySQL 调优
理解了上面的原理，下面则基于这些原理，说一些MySQL可以优化的方案。

MySQL 的数据最终是存储在磁盘上的，每次查询数据时，我们先需要把数据加载进缓存，然后读取，如果每次查询的数据都已经存在于缓存了，那么就不用去磁盘读取，避免了一次磁盘 IO，这是我们最期望的。因此为了尽量在 LRU 链表中缓存更多的缓存页，我们**可以根据服务器的配置，尽量调大 Buffer Pool 的大小**。

另外，在进行增删改查的时候，需要涉及到对 Buffer Pool 中 LRU 链表、Free 链表、Flush 链表的修改，为了线程安全，我们需要进行加锁。因此为了提高并发度，MySQL 支持配置多个 Buffer Pool 实例。当有多个Buffer Pool实例时，就能将请求分别分摊到这些Buffer Pool中，减少了锁的竞争。

可以通过如下命令去查看 Buffer Pool 的大小以及 Buffer Pool 实例的个数。

```sql
# buffer pool大小
show variables like 'innodb_buffer_pool_size';
# buffer pool实例个数
show variables like 'innodb_buffer_pool_instances';
```

> Free 链表是维护空闲缓存页的列表，Flush 链表是维护脏页的链表。什么是脏页，感兴趣的同学可以先自己查阅相关资料，本公众号的后续文章也会介绍。

另外在实际应用中，在没有外部监控工具的情况下，我们该如何知道 MySQL 的一些状态信息呢？如：缓存命中率、缓存页的空闲数、脏页数量、LRU 链表中缓存页个数、冷热数据的比例、磁盘 IO 读取的数据页数量等信息。可以通过如下命令查看：

```sql
show engine innodb status;
```

这个命令的查询结果是一个很长的字符串，可以复制出来，放在文本文件中查看分析，部分信息截图如下：
【图】

如果看到 **youngs/s** 这个值较高，说明数据从冷数据区移到热数据的频率较大，因此可以适当调大热数据所占的比例，也就是减小**innodb_old_blocks_pct**参数的值，也可以调大**innodb_old_blocks_time**参数的值

如果看到 **non-youngs/s** 这个值较高，说明数据被加载进缓存当中后，没有被移动到热数据区，这是因为在 1s 内被访问了，这很可能是全表扫描造成的，这个时候就可以去检查一下代码，是不是SQL语句写得不恰当。

### 7. 总结

总结一下，本文详细说明了普通的 LRU 链表并不适用于 MySQL，全表扫描和预读机制均会导致热点数据被淘汰，从而导致性能下降的问题。MySQL 在 LRU 算法的基础上做了优化，将链表拆分为冷、热两部分，从而解决了冷热数据的问题。最后介绍了几种 MySQL 优化的方法，可以通过调到 Buffer Pool 的大小以及个数来提升性能，也可以结合 MySQL 的运行状态信息来决定是否需要调整 LRU 链表的冷热数据区的比例。

另外，将数据进行冷热分离的这种思路，非常值得借鉴。

最后，实践是检验理论的唯一标准，MySQL 相关的原理明白了，至于生产环境的 MySQL 应该如何优化，还需要结合实际情况以及机器的配置来决定如何配置 MySQL 的参数。

### 8. 参考资料

- 《高性能 MySQL》
- 极客时间林晓斌《MySQL 实战 45 讲》
- MySQL5.7 官方文档 Buffer Pool 章节

![微信公众号](https://user-gold-cdn.xitu.io/2019/11/25/16e9e8ae4b7faf0e?w=258&h=258&f=jpeg&s=27144)
