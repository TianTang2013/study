相信很多人在面试过程中，总被问到有没有 SQL 调优经验，对于工作经验年限较少的求职者，通常都是在面试之前从网上百度一些答案，提前背熟，然后面试的时候直接将提前背好的答案说出来。笔者作为一名菜鸟，在刚满一年工作经验的时候，出去面试，就是这么干的。记得去某家公司面试的时候，被面试官问到 order by 在排序的时候执行较慢，这个时候该如何优化？我当初想都没想，就回答说给 order by 子句中的字段加上索引（当然这答案也是我提前从网上百度来的），接着面试官问为什么加索引就能提高 order by 的执行效率的时候，我就懵逼了，这我哪知道为什么啊，百度也没告诉我啊。后来面试自然也就黄了。

现在回想一下，当初是真的菜啊。不过话说回来，为什么给 order by 子句中的字段加索引就能加快 SQL 的执行？它一定能提高 SQL 的效率吗？为了搞清楚这些问题，我们就得从 order by 的实现原理上说起了。

### 示例表

为了方便说明举例，我们先创建一个示例表，建表语句如下。

```sql
CREATE TABLE `user` (
`id` BIGINT ( 11 ) AUTO_INCREMENT COMMENT '主键id',
`id_card` VARCHAR ( 20 ) NOT NULL COMMENT '身份证号码',
`name` VARCHAR ( 64 ) NOT NULL COMMENT '姓名',
`age` INT ( 4 ) NOT NULL COMMENT '年龄',
PRIMARY KEY ( `id` ),
INDEX ( `name` )
) ENGINE = INNODB COMMENT '用户表';

insert into `user`(id_card,name,age) values
('429006xxxxxxxx2134','张三',22),
('429006xxxxxxxx2135','李四',26),
('129006xxxxxxxx3136','王五',28),
('129106xxxxxxxx3337','赵六',17),
('129106xxxxxxxx3349','孙XX',43),
('129106xxxxxxxx3135','马大哈',39),
('129106xxxxxxxx3134','王一',55),
('139106xxxxxxxx2236','张三',7),
('139106xxxxxxxx2130','张三',31),
('439106xxxxxxxx2729','张三',29),
('439106xxxxxxxx2734','李明',78),
('429106xxxxxxxx1734','张三',96),
('129106xxxxxxxx1737','张三',89),
('129106xxxxxxxx1132','张三',3),
('129106xxxxxxxx1197','张三',11),
('129106xxxxxxxx1184','张三',14);

```

我们创建了一张用户表，表中有 4 个字段：id 是自增的主键，id_card 表示用户的身份证号码，name 表示用户姓名，age 表示用户年龄，另外我们除了主键索引 id 外，还为 name 字段创建了一个普通索引。最后向表中插入了几条数据，方便后面举例。

假设我们现在有如下需求：按照年龄从小到大，查询前三个姓名叫做张三的用户的身份证号码、姓名、年龄。对应的 SQL 语句应该这么写：

```sql
select id_card,name,age from user where name = '张三' order by age limit 3;
```

这条 SQL 语句逻辑比较简单，很好理解，语句在执行时会使用到 name 索引树，并且会进行排序，我们可以通过关键字 Explain 来查看一下 SQL 的执行计划。

```sql
explain select id_card,name,age from user where name = '张三' order by age limit 3 \G
```

【图 1】

上图中 key 这一行的值为 name，这表示本次查询会使用到 name 索引；Extra 这一行的值为 Using filesort，这表示本次查询需要用到排序操作。接下来我们来看下，MySQL 的排序过程。

### 全字段排序

首先 MySQL 会为每个查询线程分配一块内存，叫做 sort_buffer，这块内存的作用就是用来排序的。这块内存有多大呢？由参数 **sort_buffer_size** 控制，可以通过如下命令来查看和修改：

```sql
# 查看sort_buffer的大小
show variables like 'sort_buffer_size';
# 修改sort_buffer的大小
set global sort_buffer_size = 262144;
```

针对上面我们提到的示例，我们来看下这个排序流程：

1. 首先 MySQL 为对应的线程分配一块大小为 sort_buffer_size 的内存，然后确认要向内存中放入哪些字段，由于本文示例中要查询的是 id_card、name、age 这三个字段，因此这块 sort_buffer 中要存放的字段是 id_card、name、age；
2. 通过前面的执行计划我们已经知道了该条 SQL 语句在执行时会使用 name 索引树，所以存储引擎会先在 name 索引树中找到第一个 name="张三"的叶子结点，然后返回叶子结点中存放的主键 id 值；
3. 根据上一步中返回的主键 id，进行回表，回到主键索引树上查找到对应 id 的数据，并取出 id_card、name、age 这三个字段的值，返回给 MySQL 的 server 层，放入到 sort_buffer 中；
4. 继续在 name 索引树中查找下一条 name="张三"的结点，重复步骤 2、3，直到在 name 索引树上找到第一条 name 不等于张三时，停止查找。
5. 前面 4 步已经查找到了所有 name 为张三的数据，接下来就是在 sort_buffer 中，将所有数据根据 age 进行排序；
6. 从排好序的数据中，取前三条返回。

整个过程的示意图如下：

【图 2】

这个排序过程，因为是将查询的所有字段都放入到了 sort_buffer 中（id_card、name、age），因此也被称之为全字段排序。

看到这里，有人肯定就会有疑问了，如果我们查询的数据量太大，符合 name="张三"的数据有很多条，以至于 sort_buffer 这块内存无法装下所有数据，那这个时候我们肯定无法在 sort_buffer 内存中实现所有的排序了，那又该怎么办呢？答案就是借助磁盘文件来进行排序。

借助磁盘文件进行排序的时候，通常使用归并排序算法。当从主键 id 索引树中查询到数据时，将数据放入到 sort_buffer 中，当 sort_buffer 中快满时，就在 sort_buffer 中对这部分数据进行排序，然后将排好序的数据临时存放进磁盘的一个小文件中，然后再继续从主键索引树中查询数据，再在 sort_buffer 中进行排序，写入磁盘的临时文件中，循环操作，直到所有数据读取完。最后将磁盘上的这些数据有序的小文件，合并成一个有序的大文件，这样就完成了所有数据的排序操作。

从这个过程中，我们可以发现，在要排序的数据的大小一定额情况下，如果 sort_buffer 的大小越小，即 sort_bufer_size 越小，那么我们在借助磁盘排序时，需要的临时文件越多，那么发生 IO 的次数就越多，性能也就越差。

虽然我们知道了全字段排序原理，也能通过查询数据库的配置知道 sort_buffer_size 的大小，但是我们该如何知道我们的 SQL 在执行排序时有没有借助磁盘文件进行排序呢？我们可以通过 MySQL 中 information_schema 库中 optimizer_trace 表来查看 SQL 执行的优化信息，但是默认情况下 optimizer_trace 的开关是关闭的，因为它记录的是 SQL 相关的优化信息，会额外消耗 MySQL 的资源。我们可以通过如下命令来查看和修改 optimizer_trace 的状态

```sql
# 查看
show variables like 'optimizer_trace';
# 临时针对当前数据库连接修改(连接断开后，下次再连接数据库时，该值还是false)
set optimizer_trace = "enabled=on";
# 针对所有数据库连接修改
set global optimizer_trace = "enabled=on";
```

开启了 optimizer_trace 统计信息之后，我们就可以从这张表中查看 SQL 执行的相关信息了。以文章前面示例为例，我们一次执行以下 SQL：

```sql
# 开启统计
set optimizer_trace = "enabled=on";
# 执行查询SQL
select id_card,name,age from user where name = '张三' order by age limit 3;
# 查询统计信息
select * from information_schema.optimizer_trace \G
```

最终我们可以看到如下图所示的统计信息（我只截取了一部分）

【图 3】

图中的 number_of_tmp_files 这一行表示的是排序时使用到的临时文件的个数，如果为 0，则表示的是本次排序没有借助磁盘文件排序，如果大于 0，则表示借助了磁盘文件排序。由于我电脑上安装的 MySQL，默认 sort_buffer_size 为 256KB，例子中查询的数据量也比较小，所以完全可以在 sort_buffer 内存中完成排序，不需要借助磁盘文件。

如果想要演示借助磁盘文件排序，可以先将 sort_buffer_size 设置为一个很小的值，然后再执行查询操作，最后再看看 optimizer_trace 中统计的信息。由于本文的数据太少，而 sort_buffer_size 最小可以被设置为 32KB，不能比 32KB 还小，所以最终还是使用的是内存排序，因此就不再做示范了，你可以在工作中的测试环境中试试，验证一下。

### rowid 排序

看懂了上面的全字段排序，可能有人就会有疑问了，我们实际上只需要对 name 字段排序，为什么还需要把 id_card 字段和 age 字段也放入到 sort_buffer 中呢？而且 sort_buffer 本身是有内存大小限制的，sort_buffer 中放入的字段越多，那它能存放的数据条数就越少，如果要对多条数据排序，那就很有可能需要用到磁盘文件排序了，显然磁盘文件排序没有内存排序快。

既然知道了全字段排序的缺点，那么我们该怎么改进呢？这一点 MySQL 的开发人员早就已经考虑到了，因此就有了另外一种排序方式，暂且叫它 rowid 排序吧（为什么是暂且呢？因为我在 MySQL 官方文档中并没有找到这种说法，这种说法是在极客时间上《MySQL 实战 45 讲》中看到的，不过具体叫什么不重要，重要的是知道其中的原理）。

rowid 排序原理的大致思路就是，不会将 SQL 语句中 select 后面的所有字段都放入到 sort_buffer 中，而是只会将需要被排序的字段和主键 id 放入到 sort_buffer 中，对应到本文的例子中就是：将 name 字段和主键 id 字段放入到 sort_buffer 中。

> 在实际开发过程当中，有的表我们没有创建主键索引，那这个时候 MySQL 会判断表中有没有唯一索引，如果有唯一索引，那就会将这个唯一索引当做主键；如果也没有唯一索引，那么 MySQL 会默认为每一行数据生成一个 rowid，这个 rowid 作用和主键作用一样，那么在排序的时候，放入到 sort_buffer 中的字段就是被排序的字段和 rowid 了，这也是为什么叫它 rowid 排序的由来了。

前面我们说了全字段排序会将不需要排序的字段也放入到 sort_buffer 中，这些字段会占用内存，当这些字段多到一定程度时，MySQL 会认为全字段排序这种排序方式可能会需要借助磁盘文件排序，会影响性能，因此就将排序方式改为 rowid 排序。那么这个“一定程度”到底是个什么程度呢？它是由参数 **max_length_for_sort_data** 控制的，这个参数表示的是当需要放入到 sort_buffer 中的字段的长度之和超过这个参数值时，就会使用 rowid 排序。你可以通过如下命令查看该参数的值。

```sql
show variables like 'max_length_for_sort_data';
```

max_length_for_sort_data 参数的默认值为 1024 个字节，对于本文中的示例，由于 id_card、name、age 这三个字段的总长度加起来肯定是小于 1024 个字节的，没有超过 max_length_for_sort_data 的限制，因此不会使用 rowid 排序。

为了看看 rowid 的排序流程，我先将 max_length_for_sort_data 的值设置为一个较小的值，例如 16 个字节，这样 id_card、name、age 这三个字段的长度之和就超过了这个限制，因此后面排序时会使用 rowid 排序算法。

```sql
# 限制设置为16个字节
set max_length_for_sort_data = 16;
# 查询数据
select id_card,name,age from user where name = "张三" order by age limit 3
```

以上面的查询 SQL 为例，rowid 的排序流程如下：

1. 首先 MySQL 为对应的线程分配一块大小为 sort_buffer_size 的内存，然后确认要向内存中放入哪些字段，由于示例中要查询的是 id_card、name、age 这三个字段，而这三个字段的长度之和超过了 max_length_for_sort_data 的限制，所以采用 rowid 排序， 因此这块 sort_buffer 中要存放的字段是 age 和主键 id；
2. 存储引擎先在 name 索引树中找到第一个 name="张三"的叶子结点，然后返回叶子结点中存放的主键 id 值；
3. 根据上一步中返回的主键 id，进行回表，回到主键索引树上查找到对应 id 的记录，并取出 age 字段的值，返回给 MySQL 的 server 层，放入到 sort_buffer 中；
4. 继续在 name 索引树中查找下一条 name="张三"的结点，重复步骤 2、3，直到在 name 索引树上找到第一条 name 不等于张三时，停止查找。
5. 前面 4 步已经查找到了所有 name 为张三的数据，并将要排序的字段 age 和主键 id 放入到了 sort_buffer 中，接下来就是在 sort_buffer 中，将所有数据根据 age 进行排序；
6. 从排好序的数据中，取前 3 条数据。由于我们要查询的数据是 id_card、name、age 这三个字段，此时 sort_buffer 中只有 id 和 age 字段，因此此时还需要根据取到的三条数据的 id，会到主键索引树上读取 id_card、name、age 的值；
7. 最后将 id_card、name、age 字段的数据返回。

这个流程的示意图如下：

【图 4】

从这个流程中，我们可以发现，相比全字段排序而言，rowid 排序的回表次数更多。

同样，我们也可以查看一下 rowid 排序时，optimizer_trace 中记录的信息。执行语句如下：

```sql
select * from information_schema.optimizer_trace\G
```

【图 5】

从图中我们可以看到，sort_mode 这一行显示的 rowid，这说明本次排序使用的是 rowid 排序，而对于全字段排序，则显示的不是 rowid。

### order by 优化思路

理解了全字段排序和 rowid 排序的原理，现在我们可以思考一下该如何优化排序的 SQL。

#### 1. 调整 sort_buffer_size 大小

首先，无论是全字段排序还是 rowid 排序，它们都会受到 sort_buffer 内存大小的影响，如果数据过多，就到导致借助磁盘文件排序。借助磁盘文件排序，很产生磁盘 IO，性能差，显然这不是我们所期望的，我们应该尽量避免。**如果参数 sort_buffer_size 太小，而 MySQL 服务器的配置又较高，我们可以尝试将 sort_buffer_size 设置得大一点**。

#### 2. max_length_for_sort_data

当查询字段的长度超过 max_length_for_sort_data 的限制后，MySQL 就会采用 rowid 排序，但是 rowid 排序会产生更多的回表次数，这可能会造成磁盘读，也会降低查询性能，所以为了避免 MySQL 使用 rowid 排序，我们可以将 max_length_for_sort_data 参数的值适当调大一点。

max_length_for_sort_data 参数的值，MySQL 默认是 1024，即 1KB。我个人觉得这个值已经很大了，1024 个字节已经可以包含很多字段了，按照平均每个字段 8 个字节来算（变长的 varchar 类型除外），差不多可以容纳 256 个字段了。如果你的查询 SQL 中要查询的字段长度超过了 1024 个字节，这极有可能是 SQL 写得有问题了，我们可以尝试去优化 SQL，而不是去调整 MySQL 的系统参数。例如通过减少查询字段，分多次查询，或者通过中间表来优化 SQL（这个地方也再次印证了尽量不要使用 select \* 这类 SQL 语句的说法）。总之我个人觉得，max_length_for_sort_data 参数的值尽量不要调整。

说到这里，《高性能 MySQL》一书中，在第八章开头，作者曾提到过，MySQL 虽然为我们提供了很多可以配置的系统参数，但是这些参数大部分我们都可以直接采用默认值，只有极少数需要我们根据实际场景去调整。如果我们过多的去调整参数，并且还没有经过实际生产验证，极有可能起到反作用。

#### 3. 使用联合索引

因为数据是乱序的，所以我们要对数据进行排序，那如果假设数据本身就是有序的，那么我们就不用再对数据进行排序操作了，避免了后面 sort_buffer 内存大小、磁盘文件排序等一系列问题了。我们都知道，MySQL 中索引的数据结构使用的是 B+Tree，该数据结构的一大特点就是索引值是有序存放的，那么我们可以利用有序性这一特点，来避免排序工作。

对于本文示例中的 SQL，如果 age 字段在索引树上本身就是有序的，那么我们就不用再额外在 sort_buffer 中排序了，因此我们可以考虑建立一个 name 和 age 的联合索引：index(name,age)。

再继续思考，因为我们需要查询 id_card、name、age 这三个字段的信息，而 index(name,age)这个联合索引上只有 name 和 age 的字段值，这意味着虽然我们可以通过这个联合索引避免掉排序操作，但是我们还需要回到主键索引树上取 id_card 字段的值，也就是需要回表，回表可能又造成磁盘读，所以我们还有优化空间。

如果看过我这篇文章的朋友《[MySQL 索引的工作原理](https://mp.weixin.qq.com/s/scEl3uU2OGLW6eR-d5SrCQ)》，这个时候可能会想到，避免回表操作最常用的手段就是使用覆盖索引技术，所以这个时候我们可以创建 name、age、id_card 这三个字段的联合索引：**index(name,age,id_card)**，这样联合索引树上存的数据，已经全部满足了我们要查询的数据，所以不需要再进行回表操作了。SQL 语句如下：

```sql
# 我们先删除前面为name字段创建的索引
alter table user drop index `name`;
# 创建name、age、id_card的联合索引
alter table user add index(`name`,`age`,`id_card`);
# 使用explain关键字，查看一下SQL的执行计划
EXPLAIN select id_card,name,age from user where name = "张三" order by age limit 3\G;
```

SQL 的执行计划如下图：

【图】

从图中，我们可以发现，Extra 这一行变成了 Using index，没有了 Using filesort。**Using index 表示使用到了覆盖索引，没有 Using filesort 表示本次 SQL 的执行不需要用到 sort_buffer 进行排序操作**。

需要额外说明的是，本文示例 SQL 中，只查询了 name="张三"的数据，所以我们能保证在联合索引 index(name,age,id_card)上，age 字段的值是有序的。如果我们的查询条件为 name in ("张三","王五")，那么就无法保证 age 字段是有序的了，因为联合索引中，是先保证第一列有序，再依次保证后面的列有序，所以这个时候还是得排序。如果还想利用这个特性，这个时候我们可以分两次查询，然后在应用程序的内存中进行数据的排序，如：

```sql
# 分两次查询
select id_card,name,age from user where name = '张三' order by age limit 3;
select id_card,name,age from user where name = '李四' order by age limit 3;
# 然后在应用程序中自己排序
```

### 解答开篇

在文章开篇处我提到了一个问题：给 order by 子句中的字段添加索引，就一定能加快 SQL 的执行效率吗？现在我们来做个实验，示例表和数据还是前面的，不同的是我们将除了主键以外的索引都删除，然后再为 age 字段创建一个索引。最终 user 表中，id 为主键索引，age 列有一个普通索引。然后我们用 Explain 关键字查看一下如下 SQL 的执行计划:

```sql
explain select id_card,name,age from user order by age limit 3 \G
```

执行计划如下图所示。

【图】

从图中我们可以发现，key 那一行的值为 null，这说明没有使用到 age 索引；type 这一行的值为 ALL，这说明进行了全表扫描；Extra 这一行的值为 Using filesort，这说明在 sort_buffer 进行了排序。

看到这里是不是有点毁三观啊，我们为 age 列创建了索引，怎么就没有使用呢？

这是因为我们要查询 id_card、name、age 这三个字段，而 age 索引树上没有存放这些信息，所以最终还是回表到主键索引树上查询这些信息。这个时候 MySQL 认为，虽然 age 索引树上 age 字段值是有序的，可以避免排序操作，但是它需要回表到主键索引树去取其他字段的信息，MySQL 认为这个回表操作所消耗的性能大于避免排序操作所节省的性能，所以干脆就直接扫描主键索引树了，而不使用 age 索引树了。

继续实验，我们的 user 表中的数据太少了，一共只有 16 条，现在我们增加一点数据，我写了个简单的存储过程，向数据库中插入了 10 万条数据（为了简单，每条数据的 name 和 id_card 的值都是瞎编的）。

```sql
delimiter ;;
create procedure fakeData()
BEGIN
DECLARE
    i INT;
SET i = 1;
WHILE
    ( i <= 100000 ) DO
    INSERT INTO user(id_card,name,age)
VALUES
    ( '429006xxxxxxxx2135', CONCAT('AA',i), i%100 ); # 身份证号码都是一样的（实际情况显然不是这样），姓名为AA+i，年龄为对i除以100取模
SET i = i + 1;
END WHILE;
END
delimiter ;;

# 执行存储过程
call fakeData();
```

现在表里面大概有 10 万行数据了，我们在用 Explain 查看一下上面的查询过程：

```sql
explain select id_card,name,age from user order by age limit 3 \G
```

查询的执行计划如下图所示：

【图】

从图中我们可以看到，type 为 index，key 为 age，这说明使用了 age 这个索引，并且 Extra 这一行显示为 null，说明也不需要额外排序。同样的 SQL 语句，因为表中的数据量不一样，看到的却是不同的执行计划，这是为什么呢？

这是因为表中的数据量比较多，id 主键索引树这个时候有 10 万行数据了，如果对 id 索引树进行全表扫描的话，MySQL 会认为这个过程会比较费时。而通过走 age 索引树，取排序前三的 age 所对应的 id，然后回表到主键索引树取数据，这个过程相比较直接对 id 索引树进行全表扫描执行得快，所以就决定走 age 索引树了，也就是我们在执行计划中看到的。

现在我们再看一下下面这个 SQL 语句的执行计划：

```sql
explain select id_card,name,age from user order by age limit 1000 \G
```

【图】

这条 SQL 语句和前面的也是类似，唯一的区别就是 limit 取了前 1000 条记录，结果我们看到执行计划的截图当中，type 为 ALL，key 为 null，这些说明这条 SQL 语句没有使用 age 索引，并且进行了全表扫描，Extra 这一行为 Using filesort，说明需要在内存中进行排序。

这又是为什么呢？这是因为如果使用 age 索引树的话，就得回表，回到主键索引树中取数据。而 limit 为 1000，表示要取 1000 条数据，这就要回 1000 次表，MySQL 任务这个过程回表次数太多，消耗太大，还不如直接对主键索引树进行全表扫描，所以没有选择 age 索引。

看了这三个例子，同样的 SQL 逻辑，不同的是表中的数据量和要返回的数据条数，居然看到的执行计划不一样，有的使用了索引，有的没有使用索引，产生这些现象的原因在于 MySQL 优化器是如何选择的。因此在实际开发过程中，一条 SQL 语句究竟有没有使用索引，我们需要先通过 Explain 查看了执行计划才能确定，所以以后不要谈到 SQL 优化，上来不管三七二十一就说创建索引了。

### 总结

本文主要讲解了 order by 排序的两种方法，分别为全字段排序和 rowid 排序，排序过程受系统参数 sort_buffer_size 和 max_length_for_sort_data 的影响。当查询的数据量过大时，超过 sort_buffer_size 的大小，那么就会借助磁盘文件进行排序。如果查询的字段过多，每一行记录的查询字段长度之和超过 max_length_for_sort_data 后，MySQL 会认为数据量过大，可能会超过 sort_buffer_size，因此会选择使用 rowid 排序。

开发人员如何知道一条 SQL 语句的排序使用的是全字段排序还是 rowid 排序呢？排序过程中有没有借助磁盘文件排序呢？可以通过查看 optimizer_trace 来查看，**number_of_tmp_files** 表示借助临时磁盘文件的个数，从 **sort_mode** 这一行中，可以知道是那种排序，如果显示的是 rowid，那就是 rowid 排序，否则是全字段排序。默认情况下，optimizer_trace 的开关是关闭的，因为统计这些信息，需要额外消耗 MySQL 服务器的资源。

接着我们根据掌握的 order by 的排序原理，提供了几种优化 order by 语句的思路，可以通过调整 MySQL 的系统参数 sort_buffer_size 和 max_length_for_sort_data，或者创建联合索引来提高 SQL 的执行效率。

最后，我通过几个例子，证明了 即使为 order by 子句中的字段创建了索引，在执行时就一定会选择索引，MySQL 的优化器会根据实际情况来抉择是否使用索引。而我们在实际开发过程中，如果要对 SQL 语句优化，也应该是如此，结合实际场景，借助 Explain 等工具，经过分析后再决定如何优化 SQL。

### 参考资料

- 《高性能 MySQL》
- 极客时间林晓斌《MySQL 实战 45 讲》
