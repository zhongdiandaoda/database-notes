# Overview

> 数据库和实例：
>
> 数据库：物理操作系统文件或其他形式文件类型的集合。在MySQL数据库中，数据库文件可以是`frm`、MYD、MYI、`ibd`结尾的文件。当使用NDB引擎时，数据库的文件可能不是操作系统的文件，而是内存中的文件。
>
> 实例：MySQL数据库实例由后台线程以及一个共享内存区组成，数据库实例才是真正用于操作数据库文件的。

在MySQL数据库中，实例和数据库通常一一对应，但是，在集群情况下，可能存在一个数据库被多个实例使用的情况。

MySQL实例在系统上的表现是一个进程。

MySQL实例启动时，会在以下路径一次查找配置文件，后查找到的配置会覆盖先查找到的配置：

`/etc/my.cnf		/etc/mysql/my.cnf		/usr/local/mysql/etc/my.cnf		~/.my.cnf`

MySQL体系结构：

![img](InnoDB.assets/403167-20190116145915277-683033214.jpg)

MySQL的存储引擎是插件式的，每个存储引擎开发者可以按照自己的意愿来进行开发。

## `InnoDB`存储引擎

`InnoDB`存储引擎支持事务，其设计目标主要面向 OLTP(on-line transaction processing) 的应用。其特点是行锁设计、支持外键，并支持非锁定读。

InnoDB 存储引擎将数据放在一个逻辑的表空间中，它使用多版本并发控制（MVCC）来提高并发性，并实现了SQL标准的四种隔离级别。同时，使用next-key locking策略来避免幻读现象。

InnoDB采用了聚簇索引来存储表中的数据，每张表按照主键顺序进行存储，如果没有显式为一张表指定主键，InnoDB存储引擎会为每一行生成一个6字节的ROWID作为主键。

## `MyISAM`存储引擎

MyISAM存储引擎不支持事务、表锁设计，支持全文索引，主要面向一些 OLAP(On-Line Analytical Processing) 数据库应用。

MyISAM存储引擎表由MYD和MYI组成，前者存放数据文件，后者存放索引。



# `InnoDB`存储引擎

## InnoDB体系架构

InnoDB存储引擎有多个内存块，共同组成了一个大的内存池，负责：

- 维护所有进程/线程需要访问的多个内部数据结构
- 缓存磁盘上的数据，同时对磁盘文件的数据修改之前在这里缓存
- 重做日志缓冲
- ….

<img src="InnoDB.assets/image-20210804170535952.png" alt="image-20210804170535952" style="zoom:80%;" />

后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最新的数据，此外，将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常时InnoDB能恢复到正常运行状态。

### 后台线程

1. Master Thread

   Master Thread是核心后台线程，负责将缓冲池的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、undo页的回收等。

2. IO Thread

   InnoDB大量使用了AIO来处理写请求，IO Thread的工作主要是负责这些IO请求的回调，IO Thread包括read、write、insert buffer和log。

3. Purge Thread

   回收已经使用并分配的undo页，减轻Master Thread的压力。

4. Page Cleaner Thread

   回收脏页，减轻Master Thread的压力。

### 内存

1. 缓冲池

   InnoDB 存储引擎是基于磁盘存储的，由于磁盘速度和 CPU 速度的差距，采用了缓冲池技术来提高数据库的性能。

   当修改缓冲池中页的数据时，首先将数据写入缓冲池，然后通过 Checkpoint 机制刷新回磁盘。

   ![image-20210804172344245](InnoDB.assets/image-20210804172344245.png)

2. LRU List，Free List 和 Flush List

   InnoDB 引擎的缓冲池采用了优化的LRU算法进行管理，将新读取到的页放到LRU列表的midpoint位置。

   传统LRU链表的问题：

   - 进行全表扫描时，可能会将访问频率很低的数据页装入缓存，扫描结束后导致命中率明显降低。
   - 触发MySQL的预读机制时，会将可能使用的其他页加载到内存，此时可能会淘汰访问比较频繁的数据页，降低命中率。

   优化后，缓冲池中63%的空间用于存放热数据，37%的空间用于存放冷数据。

   用户可以通过`innodb_old_blocks_time`参数控制数据页在冷数据区停留多久后转移到热数据区。

   Free List 存放缓冲池中可以使用的页，LRU List 存放使用 LRU 算法管理的数据页。

   LRU List中的页被修改后，称该页为脏页，即缓冲池中的页和磁盘中的页的数据产生了不一致。这是数据库会通过CHECKPOINT机制将脏页刷新回磁盘，Flush List 中的页即为脏页列表（脏页同时存在于LRU List和Flush List）。

3. Redo Log 缓冲

   InnoDB存储引擎首先把Redo Log 存入到该缓冲中，然后以一定频率将其刷新到Redo Log文件。

   - Master Thread 每个1s将Redo Log缓冲刷新到Redo Log文件。
   - 每个事务提交时会将Redo Log缓冲刷新到Redo Log文件。
   - Redo Log缓冲剩余空间小于1/2时，会将Redo Log缓冲刷新到Redo Log文件。

## Checkpoint技术

如果每次一个数据页发生变化，就立即将缓冲池中的页刷新到磁盘，那么开销将非常大。同时，如果在从缓冲池将页的新版本刷新到磁盘时发生了宕机，那么数据就不能恢复了。为了避免发生数据丢失的问题，当前基于事务的数据库系统都普遍采用了Write Ahead Log策略，即当事务提交时，先写redo log，再修改页。当由于发生宕机导致数据丢失时，通过重做日志来完成数据的恢复。

checkpoint机制的目的是：

- 缩短数据库的恢复时间（缓冲池和redo log文件很大时，恢复数据库耗时会增加）
- 缓冲池不够用时，脏页刷新
- Redo log不可用时，刷新脏页

当数据库宕机时，只需要对checkpoint之后的日志进行恢复即可。

当LRU算法淘汰的页是脏页时，需要强制执行checkpoint，将脏页刷新回磁盘。

InnoDB的checkpoint包括Sharp Checkpoint和Fuzzy Checkpoint。

Sharp Checkpoint发生在数据库关闭时将所有的脏页都刷新回磁盘。

Fuzzy Checkpoint包括：

- Master Thread Checkpoint：每隔一段时间异步刷新一定比例的页回磁盘。
- FLUSH_LRU_LIST checkpoint：保证LRU列表中有一定数量的空闲页可用（Page Cleaner Thread）。
- Async/Sync Flush Checkpoint：Redo Log不可用时，刷新一定数量的脏页回磁盘（Purge Thread）。
- Dirty Page too much checkpoint：脏页占缓冲池比例过高时，强制执行checkpoint。

## Master Thread工作方式





## InnoDB关键特性

### 插入缓冲







































# 索引

`InnoDB`存储引擎支持以下几种常见的索引：

- B+树
- 全文索引
- 哈希索引

B+树索引并不能找到一个给定键值的具体行，它只能找到被查找数据行所在的页，然后数据库把页读入到内存，再在内存中进行查找。

## AVL

平衡二叉树，指左右子树高度差绝对值不大于1的二叉搜索树。

插入一个节点后，可能需要多次左旋或右旋来保持AVL的平衡性，代价很大。

## B+树

B+树的所有记录都在叶子节点上，并且是顺序存放的。

B+树的索引分为聚簇索引和辅助索引。

### 聚簇索引

`InnoDB`存储引擎表是索引组织表，表中数据按照主键顺序存放。聚簇索引就是按照每张表的主键构建一棵二叉树，同时，叶子节点存放的是整张表的行记录数据，也将聚簇索引的叶子节点称为数据叶，每个数据叶都通过一个双向链表进行链接。





# 锁









# 事务

事务是数据库区分于文件系统的重要特性之一。事务会把数据库从一种一致状态转换为另一种一致状态。在数据库提交工作时，可以确保要么所有修改都已经保存了，要么所有修改都不保存。

`InnoDB`存储引擎中的事务完全符合ACID的特性：

- 原子性 atomicity
- 一致性 consistency
- 隔离性 isolation
- 持久性 durability

1 原子性

原子性指整个数据库事务是不可分割的工作单位，只有使事务中所有的数据库操作都执行成功，才算整个事务成功。任意一条SQL语句执行失败，已经执行成功的SQL语句也必须撤销，数据库状态应该退回到执行事务前的状态。

2 一致性

一致性是指事务将数据库从一种一致的状态转变为下一种一致的状态。在事务开始之前和事务结束以后，数据库的完整性约束没有被破坏。例如，表中的一个字段姓名满足唯一性，如果一个事务对姓名进行了修改，但是在事务提交或者回滚后，表中的姓名变得不唯一了，这就破坏了事务的一致性要求。因此，事务是一致性的单位，如果事务中某个动作失败了，系统会撤销事务，恢复初始状态。

3 隔离性

事务的隔离性要求同一时间，只允许一个事务请求同一数据，不同的事务之间彼此没有任何干扰。

4 持久性

事务一旦提交，其结果就是永久性的，即使发生宕机等故障，数据库也能将数据恢复。持久性保障事务系统的高可靠性，但不保障高可用性，如RAID损坏等。

## 事务的实现

事务的隔离性由锁来实现，事务的原子性，一致性和持久性通过数据库的redo log和undo log来实现。

### redo

重做日志用来实现事务的持久性，其由两部分组成：一是内存中的重做日志缓冲（redo log buffer），是易失的；二是重做日志文件（redo log file），是持久的。

`InnoDB`是事务的存储引擎，通过Force Log at Commit机制实现事物的持久性，即当事务提交时，必须先将事物的所有日志写入到重做日志文件中进行持久化。

为了确保每次日志都写入redo log file，在每次将redo log buffer写入redo log file后，`InnoDB`存储引擎都会调用依次`fsync`操作，将文件系统缓存中的文件刷新到磁盘中，磁盘的性能决定了事务提交的性能。

用户可以手动控制日志写入磁盘的策略，让数据库在事务提交时不强制将log刷新到磁盘，但这样做会破坏ACID特性。将`innodb_flush_log_at_trx_commit`为1时，每次提交事务都会将日志刷新到磁盘；为0时，表示事务提交后不强制写入到磁盘，而是由master线程每1秒进行依次日志文件的`fsync`操作；为2时，表示每次redo log写入文件系统的缓存，由操作系统决定缓存何时刷新到磁盘。

在`InnoDB`中，重做日志都是以512字节进行存储的。若一个页中产生的重做日志大小大于512字节，则需要分割为多个redo log block进行存储。因为redo log block的大小与磁盘扇区一样，因此redo log的写入可以保证原子性。

重做日志块的结构：

![image-20210815145107748](InnoDB.assets/image-20210815145107748.png)



### undo

undo用来支持事务的回滚操作，存放在数据库内部的undo段中。undo segment位于共享表空间内。

undo操作会将数据库逻辑的恢复到执行事务之前的状态，所有修改都被逻辑地取消了（防止物理将数据库恢复到先前的状态时影响到其他事务）。例如，对每个INSERT，执行一个DELETE；对每个UPDATE，执行一个相反的UPDATE。

undo的了另一个作用时MVCC，当用户读取一行记录时，若该记录已被其他事务占用，当前事务可以通过undo读取之前的行版本信息，实现非锁定读取。



## 事务控制语句



## 隐式提交的SQL语句



## 事务的隔离级别

**1、READ UNCOMMITTED**

在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。

本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。读取未提交的数据，也被称之为脏读（Dirty Read）。

**2、READ COMMITTED**

这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。
它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。
这种隔离级别会导致不可重复读（`Nonrepeatable Read`）的问题，因为同一事务的其他实例在该实例处理期间可能会有新的commit，所以相同的 select 可能返回不同结果。

**3、REPEATABLE READ**

这是MySQL的默认事务隔离级别，一个事务读取数据后，会对数据加锁，其他事务无法进行UPDATE操作。
不过理论上，这会导致另一个棘手的问题：幻读 （Phantom Read）。
幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。由于其他事务的INSERT操作，当前事务会产生幻读现象。
`InnoDB`和Falcon存储引擎通过多版本并发控制（MVCC，Multi Version Concurrency Control）机制解决了该问题。

**4、SERIALIZABE**

这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。
简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。

```sql
[窗口A]:

mysql> SET GLOBAL tx_isolation='SERIALIZABLE';
Query OK, 0 rows affected (0.00 sec)

mysql> quit;
Bye

[root@vagrant-centos65 ~]# mysql -uroot -pxxxx(重新登录)

mysql> SELECT @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| SERIALIZABLE   |
+----------------+
1 row in set (0.00 sec)

mysql> select * from test.user;
+----+------+
| id | name |
+----+------+
|  2 | b    |
|  4 | d    |
+----+------+
2 rows in set (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test.user values (5, 'e');
Query OK, 1 row affected (0.00 sec)

[窗口B]:

mysql> quit;
Bye

[root@vagrant-centos65 ~]# mysql -uroot -pxxxx(重新登录)

mysql> SELECT @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| SERIALIZABLE   |
+----------------+
1 row in set (0.00 sec)

mysql> select * from test.user;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

[窗口A]:

mysql> commit;
Query OK, 0 rows affected (0.01 sec)

[窗口B]:

mysql> mysql> select * from test.user;
+----+------+
| id | name |
+----+------+
|  2 | b    |
|  4 | d    |
|  5 | e    |
+----+------+
3 rows in set (0.00 sec)
```



四种隔离级别的问题：

| 隔离级别                        | 脏读（Dirty Read） | 不可重复读（NonRepeatable Read） | 幻读（Phantom Read） |
| ------------------------------- | ------------------ | -------------------------------- | -------------------- |
| 未提交读（Read uncommitted）    | 可能               | 可能                             | 可能                 |
| 已提交读（Read committed）      | 不可能             | 可能                             | 可能                 |
| **可重复读（Repeatable read）** | **不可**能         | **不可**能                       | **可能**             |
| 可串行化（Serializable ）       | 不可能             | 不可能                           | 不可能               |