# Overview

> 数据库和实例：
>
> 数据库：物理操作系统文件或其他形式文件类型的集合。在MySQL数据库中，数据库文件可以是`frm`、MYD、MYI、`ibd`结尾的文件。当使用NDB引擎时，数据库的文件可能不是操作系统的文件，而是内存中的文件。
>
> 实例：MySQL数据库实例由后台线程以及一个共享内存区组成，数据库实例才是真正用于操作数据库文件的。

在MySQL数据库中，实例和数据库通常一一对应，但是，在集群情况下，可能存在一个数据库被多个实例使用的情况。

MySQL实例在系统上的表现是一个进程。

MySQL体系结构：

![img](InnoDB.assets/403167-20190116145915277-683033214.jpg)

MySQL的存储引擎是插件式的，每个存储引擎开发者可以按照自己的意愿来进行开发。

## `InnoDB`存储引擎

`InnoDB`存储引擎支持事务，其设计目标主要面向 OLTP(on-line transaction processing) 的应用。其特点是行锁设计、支持外键，并支持非锁定读。

`InnoDB` 存储引擎将数据放在一个逻辑的表空间中，它使用多版本并发控制（MVCC）来提高并发性，并实现了SQL标准的四种隔离级别。同时，使用next-key locking策略来避免幻读现象。

`InnoDB`采用了聚簇索引来存储表中的数据，每张表按照主键顺序进行存储，如果没有显式为一张表指定主键，`InnoDB`存储引擎会为每一行生成一个6字节的ROWID作为主键。

## `MyISAM`存储引擎

`MyISAM`存储引擎不支持事务、表锁设计，支持全文索引，主要面向一些 OLAP(On-Line Analytical Processing) 数据库应用。

`MyISAM`存储引擎表由MYD和MYI组成，前者存放数据文件，后者存放索引。



# `InnoDB`存储引擎

## `InnoDB`体系架构

`InnoDB`存储引擎有多个内存块，共同组成了一个大的内存池，负责：

- 维护所有进程/线程需要访问的多个内部数据结构
- 缓存磁盘上的数据，同时对磁盘文件的数据修改之前在这里缓存
- 重做日志缓冲
- ….

<img src="InnoDB.assets/image-20210817155712100.png" alt="image-20210817155712100" style="zoom:50%;" />

后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最新的数据，此外，将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常时`InnoDB`能恢复到正常运行状态。

![InnoDB architecture diagram showing in-memory and on-disk structures.](InnoDB.assets/innodb-architecture.png)

## 后台线程

1. Master Thread

   Master Thread是核心后台线程，负责将缓冲池的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、undo页的回收等。

2. IO Thread

   `InnoDB`大量使用了AIO来处理写请求，IO Thread的工作主要是负责这些IO请求的回调，IO Thread包括read、write、insert buffer和log。一般来说，包括4个read thread，4个write thread，1个insert buffer thread和1个log thread。

3. Purge Thread

   事务提交后，其undo log可能不再需要，Purge Thread回收已经使用并分配的undo页，减轻Master Thread的压力。

4. Page Cleaner Thread

   回收脏页，减轻Master Thread的压力。

## 缓冲池

`InnoDB` 存储引擎是基于磁盘存储的，由于磁盘速度和 CPU 速度的差距，采用了缓冲池技术来提高数据库的性能。

当修改缓冲池中页的数据时，首先将数据写入缓冲池，然后通过 Checkpoint 机制刷新回磁盘。

查看缓冲池大小：

```mysql
mysql> show variables like 'innodb_buffer_pool_size';
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
1 row in set, 1 warning (0.01 sec)
```

从`InnoDB` 1.0.x开始，允许有多个缓冲池实例，每个页根据哈希值平均分配到不同的缓冲池实例中，减少数据库内部的资源竞争，增加数据库的并发处理能力。

查看缓冲池实例数：

```mysql
mysql> show variables like 'innodb_buffer_pool_instances';
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| innodb_buffer_pool_instances | 1     |
+------------------------------+-------+
1 row in set, 1 warning (0.00 sec)
```

可以在配置文件中修改该值。

### LRU List，Free List 和 Flush List

`InnoDB` 引擎的缓冲池中存储数据页和索引页组成的链表，采用了优化的 LRU 算法进行管理，将新读取到的页放到 LRU 列表的 midpoint 位置。

![img](InnoDB.assets/midpoint.png)

传统LRU链表的问题：

- 进行全表扫描时，可能会将访问频率很低的数据页装入缓存，扫描结束后导致命中率明显降低。
- 触发MySQL的预读机制时，会将可能使用的其他页加载到内存，此时可能会淘汰访问比较频繁的数据页，降低命中率。

优化后，缓冲池中5/8的空间用于存放热数据，称为 young 区，3/8的空间用于存放冷数据，称为 old 区。

```mysql
mysql> show variables like 'innodb_old_blocks_pct';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+
1 row in set, 1 warning (0.00 sec)
```

![Content is described in the surrounding text.](InnoDB.assets/innodb-buffer-pool-list.png)

当数据页第一次被加载到缓冲池时（用户执行一条SQL查询，或者预读机制），会放到 old 区的首部，当这个数据页再次被访问到时，如果该页在old区存在的时间超过了1秒，就把它移动到 young 区的首部（这会导致young区和old区的其他数据页全部后移一位）。

用户可以通过`innodb_old_blocks_time`参数控制数据页在冷数据区停留多久后转移到热数据区。

改进的LRU算法解决了之前提到的两个问题。

**针对全表扫描的问题：**

1. 扫描过程中，需要新插入的数据页，都放到old区
2. 一个数据页有多条记录，因此一个数据页会被访问多次
3. 由于是顺序扫描，因此一个页的连续的两次访问间隔时间会小于1s，不会转移到 young 区。
4. 该页扫描完后，不会再访问该页，因此该页很快就会被淘汰。

**针对预读机制：**预读到的页位于 old 区，如果它不是热点页，会很快被淘汰。

可以通过`show engine innnodb status\G`监控冷热数据，pages made young表示页从old 区转移到young区的数量，pages not made young表示在old区被淘汰的页个数。

Free List 存放缓冲池中可以使用的页，LRU List 存放使用 LRU 算法管理的数据页。

LRU List中的页被修改后，称该页为脏页，即缓冲池中的页和磁盘中的页的数据产生了不一致。这是数据库会通过CHECKPOINT机制将脏页刷新回磁盘，Flush List 中的页即为脏页列表（脏页同时存在于LRU List和Flush List）。

### Change Buffer

写缓冲，在MySQL5.5之前叫Insert Buffer，是一种特殊的数据结构，当辅助索引页不在缓冲池中，且需要对其进行修改（如UPDATE，DELETE，INSERT）时，`InnoDB`会将这些修改缓存在写缓冲之中，并在特定时机将缓冲的修改与物理页进行合并。

![Content is described in the surrounding text.](InnoDB.assets/innodb-change-buffer.png)

与聚簇索引不同，辅助索引通常不是唯一的，对辅助索引的修改操作随机性很强，修改设计的Page通常在索引树中不相邻，延迟对修改的合并可以避免大量的随机IO，提高数据库性能。

使用Change Buffer的两个条件：

- 索引为辅助索引(secondary index)
- 索引非唯一



change buffer写入磁盘的时机：

- 当一个被修改的页需要加载到缓冲池中时
- 一个后台线程认为数据库空闲时

- 数据库正常关闭时
- redo log写满时

merge 操作的时机：

- 原始数据页加载到 Buffer Pool 时
- 系统后台定时触发 merge 操作
- 数据库正常关闭时



> 对聚簇索引的操作需要进行唯一性检查，必须将索引页加载到缓冲池，因此只有对辅助索引的操作可以使用写缓冲。



### Redo Log Buffer

`InnoDB`存储引擎首先把Redo Log 存入到该缓冲中，然后以一定频率将其刷新到Redo Log文件。

- Master Thread 每隔1s将Redo Log缓冲刷新到Redo Log文件。
- 每个事务提交时会将Redo Log缓冲刷新到Redo Log文件。
- Redo Log缓冲剩余空间小于1/2时，会将Redo Log缓冲刷新到Redo Log文件。



## Checkpoint技术

如果每次一个数据页发生变化，就立即将缓冲池中的页刷新到磁盘，那么开销将非常大。同时，如果在从缓冲池将页的新版本刷新到磁盘时发生了宕机，那么数据就不能恢复了。为了避免发生数据丢失的问题，当前基于事务的数据库系统都普遍采用了 WAL(Write Ahead Log) 策略，即当事务提交时，先写redo log，再修改页。当由于发生宕机导致数据丢失时，通过重做日志来完成数据的恢复。

checkpoint机制的目的是：

- 缩短数据库的恢复时间（缓冲池和redo log文件很大时，恢复数据库耗时会增加）
- 缓冲池不够用时，脏页刷新
- Redo log不可用时，刷新脏页

当数据库宕机时，只需要对checkpoint之后的日志进行恢复即可。

当LRU算法淘汰的页是脏页时，需要强制执行checkpoint，将脏页刷新回磁盘。

`InnoDB`的checkpoint包括Sharp Checkpoint和Fuzzy Checkpoint。

Sharp Checkpoint发生在数据库关闭时将所有的脏页都刷新回磁盘。

Fuzzy Checkpoint包括：

- Master Thread Checkpoint：每隔一段时间异步刷新一定比例的页回磁盘。
- FLUSH_LRU_LIST checkpoint：保证LRU列表中有一定数量的空闲页可用（Page Cleaner Thread）。
- `Async/Sync Flush Checkpoint`：Redo Log不可用时，刷新一定数量的脏页回磁盘（Purge Thread）。
- Dirty Page too much checkpoint：脏页占缓冲池比例过高时，强制执行checkpoint。



## Master Thread工作方式





## `InnoDB`关键特性

### 两次写



### 自适应哈希索引



### 异步IO



### 刷新邻接页





# 文件

## 参数文件

MySQL实例启动时，会在以下路径一次查找配置文件，后查找到的配置会覆盖先查找到的配置：

`/etc/my.cnf		/etc/mysql/my.cnf		/usr/local/mysql/etc/my.cnf		~/.my.cnf`

MySQL数据库中的参数可以分为两类：

- 动态参数
- 静态参数

动态参数意味着可以在MySQL实例运行中进行更改，静态参数说明在整个实例生命周期内都不得进行修改。

对参数的修改可以设置为对当前会话有效或修改变量的全局值，对当前实例生命周期内的所有会话都生效。

## 日志文件

### 错误日志

错误日志文件对MySQL的启动、运行、关闭过程进行了记录。MySQL DBA在遇到问题时应该首先查看该文件来定位问题。

可以通过以下命令来定位该文件：

```shell
mysql> show variables like 'log_error';
+---------------+------------------------------------------------+
| Variable_name | Value                                          |
+---------------+------------------------------------------------+
| log_error     | D:\MySQL\mysql-5.7.30\data\DESKTOP-LE1O5PF.err |
+---------------+------------------------------------------------+
1 row in set, 1 warning (0.05 sec)
```

文件内容：

![image-20210818150048617](InnoDB.assets/image-20210818150048617.png)

### 慢查询日志

默认情况下，MySQL数据库并不启动慢查询日志，需要用户手动开启。不需要调优时不建议开启该功能，会影响MySQL数据库的性能。

```shell
mysql> show variables like 'slow_query_log%';
+---------------------+-----------------------------------------------------+
| Variable_name       | Value                                               |
+---------------------+-----------------------------------------------------+
| slow_query_log      | OFF                                                 |
| slow_query_log_file | D:\MySQL\mysql-5.7.30\data\DESKTOP-LE1O5PF-slow.log |
+---------------------+-----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

#开启命令
set global slow_query_log = 1;
```

通过long_query_time参数设置超过多少秒的查询视为慢查询，数据库会将这些查询语句记录在慢查询日志中。

```shell
mysql> SHOW VARIABLES LIKE 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set, 1 warning (0.00 sec)

#修改值
set global long_query_time = 4;
```

可以通过`log_output`参数选择将日志存储到文件中或者MySQL的slow_log表中，默认是存储在文件中（存储在表中会耗费更多的系统资源）。

```shell
mysql> show variables like '%log_output%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
1 row in set, 1 warning (0.00 sec)
```

**使用`mysql dumpslow`命令可以更方便的分析日志。**

### 查询日志

查询日志记录了所有客户端对MySQL数据库请求的信息，无论这些请求是否得到了正确的执行。

默认文件名为：主机名.log

```shell
mysql> show variables like '%general_log%';
+------------------+------------------------------------------------+
| Variable_name    | Value                                          |
+------------------+------------------------------------------------+
| general_log      | OFF                                            |
| general_log_file | D:\MySQL\mysql-5.7.30\data\DESKTOP-LE1O5PF.log |
+------------------+------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```

默认也是关闭的，需要调优时可选择开启。

### 二进制日志

Binary Log记录了对MySQL数据库执行更改的所有操作，例如建表或UPDATE操作，同时，使用Statement-based logging时，一些可能对数据库产生更改但未产生更改的语句也会被记录（例如没有删除任何一行的DELETE语句）。

二进制日志的作用：

- 在主从集群中，Master节点会将二进制日志发送给Worker节点，Worker节点执行二进制日志的内容来与master节点保持一致。
- 从一个备份恢复数据库时，在该备份的时间节点后的二进制日志中的操作会被执行。

Binary Log不会记录SELECT和SHOW命令，需要记录所有操作时应使用查询日志general log。**bin log只在事务提交完成后，但还没释放锁前进行一次写入。**

默认情况下bin log是二进制文件，无法直接查看，可以通过`mysqlbinlog`工具来查看。

bin_log的文件大小可能会大于`max_binlog_size`，因为一个事务的二进制日志不会只会写入到一个文件中。

记录二进制日志有两种方式：

1. Row-based logging：描述独立的一行记录被修改的细节。

   缺点：由于所有的执行的语句在日志中都将以每行记录的修改细节来记录，因此，可能会产生大量的日志内容，干扰内容也较多；比如一条update语句，如修改多条记录，则bin log中每一条修改都会有记录，这样造成bin log日志量会很大，特别是当执行alter table之类的语句的时候，由于表结构修改，每条记录都发生改变，那么该表每一条记录都会记录到日志中，实际等于重建了表。

   使用这种方式记录日志时，Worker节点根据bin log执行的SQL语句不会写入到general log

2. Statement-based logging：记录导致数据改变的SQL语句。

   缺点：为了保证SQL语句能在worker节点上正确执行，必须记录上下文信息，以保证所有语句能在worker得到和在master端执行时相同的结果；另外，主从复制时，存在部分函数（如sleep）及存储过程在worker上会出现与master结果不一致的情况，而相比Row level记录每一行的变化细节，绝不会发生这种不一致的情况
   
   使用该方式时，Worker节点收到的bin log中的SQL语句会写入到其general log中。

**基于Bin Log的主从复制过程：**

```
a.Master将数据改变记录到二进制日志(binary log)中
b.Worker上的IO进程连接Master，并请求从指定日志文件的指定位置（或者从最开始的日志）之后的日志内容
c.Master接收到来自Worker的IO进程的请求后，负责复制的IO进程会根据请求信息读取日志指定位置之后的日志信息，返回给Worker的IO进程。
  返回信息中除了日志所包含的信息之外，还包括本次传输中，Master端传输的最后的bin-log文件的名称以及bin-log的位置。
d.Worker的IO进程接收到信息后，将接收到的日志内容依次添加到Worker端的relay-log文件的最末端，并将读取到的Master端的bin-log的文   件名和位置记录到master-info文件中，以便在下一次读取的时候能够告诉Master需要读取从某个bin-log的哪个位置开始往后的日志内容。
e.Worker的Sql进程检测到relay-log中新增加了内容后，会马上解析relay-log的内容成为在Master端真实执行时候的那些可执行的内容，并在自身执行。
```



## 表结构定义文件

MySQL数据的存储是根据表进行的，每个表都会有与之对应的文件，但无论采用哪种存储引擎，MySQL都有一个以`frm`为后缀名的文件，记录该表的表结构定义，

`frm`文件还能存储视图的定义。

## `InnoDB`存储引擎文件

### 表空间文件

`InnoDB`采用将存储的数据按表空间进行存放的设计。在默认情况下，会有一个初始大小为10MB，名为ibdata1的文件，是默认的表空间文件，也叫系统表空间，用户可以通过参数`innodb_data_file_path`对其进行设置。

用户可以将多个文件组合为一个系统表空间：

```
[mysqld]
innodb_data_file_path = /db/ibdata1:2000M;/dr2/db/ibdata2:2000M:autoextend
```

此时，如果两个文件位于不同的磁盘，磁盘的负载可能被平均，因此能提高数据库的整体性能。

没有设置`innodb_file_per_table`为ON时，所有基于`InnoDB`的表的数据都会被记录到共享的系统表空间文件中。

当设置了参数`innodb_file_per_table`时，每个基于`InnoDB`存储引擎的表将被存储到一个独立的表空间中，命名为`表名.idb`。需要注意的是，这些单独的表空间文件仅存储该表的数据、索引和插入缓冲BITMAP等信息，其余信息如undo信息、系统事务信息、二次写缓冲等还是存在默认的表空间中。

默认开启：

```shell
mysql> show variables like '%innodb_file_per_table%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
1 row in set, 1 warning (0.00 sec)
```

独占表空间的优点：

- 使用独占表空间，删除表后可以回收磁盘空间；使用共享表时，删除表后共享表空间数据文件的大小不会改变，仅会对已删除空间进行标记，下次写入时可以覆盖该位置。
- 能够实现单表在不同数据库中移动
- 灾难恢复更容易



### 重做日志文件

在默认情况下，在`InnoDB`存储引擎的数据目录下会有两个名为`ib_logfile0`和`ib_logfile1`的文件，用来存储redo log，代表重做日志文件组下的2个重做日志文件。

`innodb_log_file_size`控制重做日志文件的大小，太大的话会导致数据库恢复变慢，太小的话会导致频繁的磁盘写入。



# 表

## 索引组织表

在`InnoDB`中，表都是按照主键顺序组织存放的，这种存储方式的表称为索引组织表。

当没有显式定义主键时，`InnoDB`引擎会按照以下方式创建或选择主键：

- 如果表中存在非空的unique索引字段，则设置为主键；
- 否则，自动创建一个6字节大小的主键。

存在多个非空unique索引时，选择第一个定义的非空unique索引作为主键，eg：

```mysql
create table a {
	a INT NOT NULL,
	b INT NOT NULL,
	UNIQUE KEY (b),
	UNIQUE KEY(a)
};
# 默认选择b作为索引
```

## `InnoDB`逻辑存储结构



![innodb逻辑存储结构](InnoDB.assets/2020032114540277.png)

`InnoDB`中，所有数据都逻辑的存放在一个空间中，即表空间。表空间由段 segment、簇（区） extent、页 page 组成。

**段**：分为索引段、数据段和回滚段等。索引段是非叶子节点部分，数据段是叶子节点部分，回滚段用于数据的回滚和多版本控制。一个段包含256个簇（256MB左右）。

**簇**：簇（区）是页的集合，一个区包含64个页，默认大小为1MB（64 * 16KB）。为了节省磁盘开销，创建表时默认大小为96KB。

**页**：页是`InnoDB`管理的最小单位，默认大小为16KB。

常见的页类型包括：

- 数据页（B-tree Node)

- undo页（undo Log Page）

- 系统页 （System Page）

- 事务数据页 （Transaction System Page）

- 插入缓冲位图页（Insert Buffer Bitmap）

- 插入缓冲空闲列表页（Insert Buffer Free List）

- 未压缩的二进制大对象页（Uncompressed BLOB Page）

- 压缩的二进制大对象页 （compressed BLOB Page）

**行**：`InnoDB` engine is row–oriented，数据是按照行存储的。

## `InnoDB`行记录格式

表的行记录格式决定它的列是怎样在物理上存储的，同一个物理页中的行越多，索引和查询的效率就越高。

`InnoDB`支持四种行记录格式：

![image-20211009143855864](InnoDB.assets/image-20211009143855864.png)

行记录最大长度：

1. 页大小（page size）为4KB、8KB、16KB和32KB时，行记录最大长度（maximum row length）应该略小于页大小的一半。
2. 默认页大小为16KB，因此行记录最大长度应该略小于8KB ，因此一个B+Tree叶子节点最少有2个行记录。



### 变长列

在`InnoDB`中，变长列（`variable-length column`）可能是以下几种情况

1. 长度不固定的数据类型，例如`VARCHAR`、`VARBINARY`、`BLOB`、`TEXT`等。
2. 对于长度固定的数据类型，如`CHAR`，如果实际存储占用的空间大于768Byte，`InnoDB`会将其视为变长列。
3. 变长编码下的`CHAR`

### 行溢出

1. 当行记录的长度没有超过行记录最大长度时，所有数据都会存储在当前页
2. 当行记录的长度超过`行记录最大长度`时，变长列（`variable-length column`）会选择外部溢出页（`overflow page`，一般是`Uncompressed BLOB Page`）进行存储
   - `Compact` + `Redundant`：保留前768 Byte 在当前页（`B+Tree叶子节点`），其余数据存放在`溢出页`。`768Byte`后面跟着`20Byte`的数据，用来存储`指向溢出页的指针`
   - `Dynamic` + `Compressed`：仅存储20 Byte 数据，存储`指向溢出页的指针`，这时比`Compact`和`Redundant`更高效，因为一个`B+Tree叶子节点`能`存放更多的行记录`

[![img](InnoDB.assets/overflow.png)](https://innodb-1253868755.cos.ap-guangzhou.myqcloud.com/overflow.png)

### Redundant 行记录格式

Redundant 行记录格式兼容 5.0 版本之前的页格式。

![img](InnoDB.assets/redundant_format.png)

字段偏移列表：

1. 按照列的顺序逆序放置
2. 列长度小于255Byte，用`1Byte`存储
3. 列长度大于255Byte，用`2Byte`存储

记录头信息：

| 名称                | 大小（bit） | 描述                                   |
| :------------------ | :---------- | :------------------------------------- |
| ()                  | 1           | 未知                                   |
| ()                  | 1           | 未知                                   |
| deleted_flag        | 1           | 该行是否已被删除                       |
| min_rec_flag        | 1           | 如果该行记录是预定义为最小的记录，为1  |
| **n_owned**         | 4           | 该记录拥有的记录数，用于`Slot`         |
| heap_no             | 13          | 索引堆中该条记录的索引号               |
| **n_fields**        | 10          | 记录中`列的数量`，一行最多支持`1023`列 |
| **1byte_offs_flag** | 1           | 偏移列表的单位为`1Byte`还是`2Byte`     |
| next_record         | 16          | 页中下一条记录的相对位置               |
| Total               | `48(6Byte)` | nothing                                |

隐藏列：

1. ROWID：没有显式定义主键`或`唯一非NULL的索引时，`InnoDB`会自动创建6 Byte 的 ROWID
2. Transaction ID：事务ID
3. Roll Pointer：回滚指针

实例：

```shell
mysql> CREATE TABLE t (
    -> a VARCHAR(9000)
    -> ) ENGINE=INNODB CHARSET=LATIN1 ROW_FORMAT=REDUNDANT;
Query OK, 0 rows affected (0.08 sec)

mysql> INSERT INTO t SELECT REPEAT('a',9000);
Query OK, 1 row affected (0.05 sec)
Records: 1  Duplicates: 0  Warnings: 0

$ sudo python py_innodb_page_info.py -v /var/lib/mysql/test/t.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0000>
page offset 00000004, page type <Uncompressed BLOB Page>
page offset 00000000, page type <Freshly Allocated Page>
Total number of page: 6:
Insert Buffer Bitmap: 1
Freshly Allocated Page: 1
File Segment inode: 1
B-tree Node: 1
File Space Header: 1
Uncompressed BLOB Page: 1
```

16 进制信息：

```shell
# Vim,:%!xxd
# page offset=3
0000c000: 17e8 3157 0000 0003 ffff ffff ffff ffff  ..1W............
0000c010: 0000 0000 408f 6113 45bf 0000 0000 0000  ....@.a.E.......
0000c020: 0000 0000 0113 0002 03b2 0003 0000 0000  ................
0000c030: 008b 0005 0000 0001 0000 0000 0000 0000  ................
0000c040: 0000 0000 0000 0000 0157 0000 0113 0000  .........W......
0000c050: 0002 00f2 0000 0113 0000 0002 0032 0801  .............2..
0000c060: 0000 0300 8b69 6e66 696d 756d 0009 0200  .....infimum....
0000c070: 0803 0000 7375 7072 656d 756d 0043 2700  ....supremum.C'.
0000c080: 1300 0c00 0600 0010 0800 7400 0000 14b2  ..........t.....
0000c090: 0300 0000 1408 cea3 0000 01f9 0110 6161  ..............aa
0000c0a0: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
......
0000c390: 6161 6161 6161 6161 6161 6161 6161 0000  aaaaaaaaaaaaaa..
0000c3a0: 0113 0000 0004 0000 0026 0000 0000 0000  .........&......
0000c3b0: 2028 0000 0000 0000 0000 0000 0000 0000   (..............
......

# page offset=4
00010000: 273a f701 0000 0004 0000 0000 0000 0000  ':..............
00010010: 0000 0000 408f 6113 000a 0000 0000 0000  ....@.a.........
00010020: 0000 0000 0113 0000 2028 ffff ffff 6161  ........ (....aa
00010030: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
......
00012050: 6161 6161 6161 0000 0000 0000 0000 0000  aaaaaa..........
00012060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
......
```

- 长度偏移列表（`4327 0013 000c 0006`）
  总共有`4`列，`a`列的长度超过了`255Byte`，偏移列表的单位为`2Byte`，所以`0xc07d~0xc084`为长度偏移列表

| 列序号 | 长度                   | 描述                   |
| :----- | :--------------------- | :--------------------- |
| 1      | `6` = 0x0006           | ROWID，隐藏列          |
| 2      | `6` = 0x000c-0x0006    | Transaction ID，隐藏列 |
| 3      | `7` = 0x0013-0x000c    | Roll Pointer，隐藏列   |
| 4      | `9000`(0x4327暂不理解) | a VARCHAR(9000)        |

- 记录头信息（`00 00 10 08 00 74`）

| 名称            | 值   | 描述                    |
| :-------------- | :--- | :---------------------- |
| n_fields        | 4    | 记录中列的数量          |
| 1byte_offs_flag | 0    | 偏移列表的单位为`2Byte` |

- ROWID（`00 00 00 14 b2 03`）
- Transaction ID（`00 00 00 14 08 ce`）
- Roll Pointer（`a3 00 00 01 f9 01 10`）
- a
  - `page offset=3`，前768Byte（`0xc09e~0xc39d`），在溢出页的长度为`0x2028`，即`8232`
  - `page offset=4`为`溢出页`，存放后8232Byte的数据(`0x1002e~0x12055`)

### Compact 行记录格式

Compact 行记录格式是在 MySQL 5.0 中引入的，其设计目标是高效的存储数据。

对比 Redundant：

- 减少了大约 20% 空间
- 在某些操作下会增加 CPU 的占用
- 在典型的应用场景下，比 Redundant 块

Compact 行记录的存储方式：

| 变长字段长度列表 | NULL标志位 | 记录头信息 | 列1数据 | 列2数据 | ……   |
| ---------------- | ---------- | ---------- | ------- | ------- | ---- |

变长字段长度列表是按照列的顺序逆序放置的，其长度为：

- 若列的长度小于 255 字节，用 1 字节表示。
- 若列的长度大于 255 字节，用 2 字节表示。

变长字段的长度最大不能超过 2 字节，MySQL 数据库中 VARCHAR 的最大长度限制为 65535 。

NULL 标志位记录该行数据中是否有 NULL 值，是一个位向量，可为 NULL 的列数量为 N ，则该标志位占用的字节数为`CEILING(N/8)` BYTE，列为 NULL 时不占用实际空间。

记录头信息固定占用5字节，含义为：

| 名称         | 大小 | 描述                                                         |
| ------------ | ---- | ------------------------------------------------------------ |
| ()           | 1    | 未知                                                         |
| ()           | 1    | 未知                                                         |
| deleted_flag | 1    | 该行是否被删除                                               |
| min_rec_flag | 1    | 为1，如果该记录是预先被定义好的最小的记录                    |
| n_owned      | 4    | 该记录拥有的记录数                                           |
| heap_no      | 13   | 索引堆中该条记录的排序记录                                   |
| record_type  | 3    | 000表示普通，001表示 B+ 树节点指针，010表示 infimum，<br>011表示 supremum，1xx保留 |
| next_record  | 16   | 页中下一条记录的相对位置                                     |
| total        | 40   |                                                              |

**实例：**

```mysql
mysql> CREATE TABLE t (
    -> a VARCHAR(10),
    -> b VARCHAR(10),
    -> c CHAR(10),
    -> d VARCHAR(10)
    -> ) ENGINE=INNODB CHARSET=LATIN1 ROW_FORMAT=COMPACT;
Query OK, 0 rows affected (0.03 sec)

mysql> INSERT INTO t VALUES ('1','22','22','333'),('4',NULL,NULL,'555');                                                               Query OK, 2 rows affected (0.02 sec)
Records: 2  Duplicates: 0  Warnings: 0

$ sudo python py_innodb_page_info.py -v /var/lib/mysql/test/t.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0000>
page offset 00000000, page type <Freshly Allocated Page>
page offset 00000000, page type <Freshly Allocated Page>
Total number of page: 6:
Freshly Allocated Page: 2
Insert Buffer Bitmap: 1
File Space Header: 1
B-tree Node: 1
File Segment inode: 1
```

十六进制信息：

```shell
# Vim,:%!xxd
# page offset=3
0000c000: 1f96 f8df 0000 0003 ffff ffff ffff ffff  ................
0000c010: 0000 0000 408f deaa 45bf 0000 0000 0000  ....@...E.......
0000c020: 0000 0000 0116 0002 00c3 8004 0000 0000  ................
0000c030: 00ac 0002 0001 0002 0000 0000 0000 0000  ................
0000c040: 0000 0000 0000 0000 015a 0000 0116 0000  .........Z......
0000c050: 0002 00f2 0000 0116 0000 0002 0032 0100  .............2..
0000c060: 0200 1e69 6e66 696d 756d 0003 000b 0000  ...infimum......
0000c070: 7375 7072 656d 756d 0302 0100 0000 1000  supremum........
0000c080: 2b00 0000 14b2 0a00 0000 1409 03c6 0000  +...............
0000c090: 020a 0110 3132 3232 3220 2020 2020 2020  ....12222
0000c0a0: 2033 3333 0301 0600 0018 ffc4 0000 0014   333............
0000c0b0: b20b 0000 0014 0903 c600 0002 0a01 1f34  ...............4
0000c0c0: 3535 3500 0000 0000 0000 0000 0000 0000  555.............


第1行记录（0xc078）
变长字段长度列表（03 02 01）
列a长度为1,列b长度为2,列c在LATIN1单字节编码下，长度固定，因此不会出现在该列表中,列d长度为3
NULL标志位（00）
在表中可以为NULL的可变列为a、b、d，0< 3/8 < 1，所以NULL标志位占用1Byte
00表示没有字段为NULL
记录头信息（00 00 10 00 2b）
本行记录结束的位置0xc078+0x2b=c0a3
ROWID（00 00 00 14 b2 0a）
Transaction ID（00 00 00 14 09 03）
Roll Pointer（c6 00 00 02 0a 01 10）
a（31）
字符1，VARCHAR(10)，1个字符只占用了1Byte
b（32 32）
字符22，VARCHAR(10)，2个字符只占用了2Byte
c（32 32 20 20 20 20 20 20 20 20）
字符22，CHAR(10)，2个字符依旧占用了10Byte
d（33 33 33）
字符333，VARCHAR(10)，3个字符只占用了3Byte

第2行记录（0xc0a4）
变长字段长度列表（03 01）
列a长度为1,列b、c为NULL，不占用空间，因此不会出现在该列表中，NULL标志位会标识那一列为NULL,列d长度为3
NULL标志位（06）
0000 0110，表示列b和列c为NULL
记录头信息（00 00 18 ff c4）
ROWID（00 00 00 14 b2 0b）
Transaction ID（00 00 00 14 09 03）
跟第1行记录在同一个事务内
Roll Pointer（c6 00 00 02 0a 01 1f）
a（34）
字符1，VARCHAR(10)，1个字符只占用了1Byte
b
VARCHAR(10)为NULL时，不占用空间
c
CHAR(10)为NULL时，不占用空间
d（35 35 35）
字符555，VARCHAR(10)，3个字符只占用了3Byte
```



### Compressed与Dynamic行记录格式

`InnoDB` Plugin引入了新的文件格式（file format，可以理解为新的页格式），对于以前支持的Compact和Redundant格式将其称为Antelope文件格式，新的文件格式称为Barracuda。Barracuda文件格式下拥有两种新的行记录格式Compressed和Dynamic两种。新的两种格式对于存放BLOB的数据采用了完全的行溢出的方式，在数据页中只存放20个字节的指针，实际的数据都存放在BLOB Page中，而之前的Compact和Redundant两种格式会存放768个前缀字节。

下图是Barracuda文件格式的溢出行：

![img](InnoDB.assets/990532-20170116140227130-2129734898.png)

Compressed行记录格式的另一个功能就是，存储在其中的行数据会以zlib的算法进行压缩，因此对于BLOB、TEXT、VARCHAR这类大长度类型的数据能进行非常有效的存储。

实例：

```mysql
mysql> CREATE TABLE t (
    -> a VARCHAR(9000)
    -> ) ENGINE=INNODB CHARSET=LATIN1 ROW_FORMAT=DYNAMIC;
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO t SELECT REPEAT('a',9000);                                                                                          Query OK, 1 row affected (0.02 sec)
Records: 1  Duplicates: 0  Warnings: 0

$ sudo python py_innodb_page_info.py -v /var/lib/mysql/test/t.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0000>
page offset 00000004, page type <Uncompressed BLOB Page>
page offset 00000000, page type <Freshly Allocated Page>
Total number of page: 6:
Insert Buffer Bitmap: 1
Freshly Allocated Page: 1
File Segment inode: 1
B-tree Node: 1
File Space Header: 1
Uncompressed BLOB Page: 1
```

16 进制信息：

```shell
# Vim,:%!xxd
# page offset=3
0000c000: 0006 f2d2 0000 0003 ffff ffff ffff ffff  ................
0000c010: 0000 0000 4090 bbcb 45bf 0000 0000 0000  ....@...E.......
0000c020: 0000 0000 011a 0002 00a7 8003 0000 0000  ................
0000c030: 0080 0005 0000 0001 0000 0000 0000 0000  ................
0000c040: 0000 0000 0000 0000 015e 0000 011a 0000  .........^......
0000c050: 0002 00f2 0000 011a 0000 0002 0032 0100  .............2..
0000c060: 0200 1d69 6e66 696d 756d 0002 000b 0000  ...infimum......
0000c070: 7375 7072 656d 756d 14c0 0000 0010 fff0  supremum........
0000c080: 0000 0014 b211 0000 0014 093d ee00 0001  ...........=....
0000c090: c201 1000 0001 1a00 0000 0400 0000 2600  ..............&.
0000c0a0: 0000 0000 0023 2800 0000 0000 0000 0000  .....#(.........
......

# page offset=4
00010000: 2371 f7ac 0000 0004 0000 0000 0000 0000  #q..............
00010010: 0000 0000 4090 bbcb 000a 0000 0000 0000  ....@...........
00010020: 0000 0000 011a 0000 2328 ffff ffff 6161  ........#(....aa
00010030: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
......
00012340: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00012350: 6161 6161 6161 0000 0000 0000 0000 0000  aaaaaa..........

```



1. `page offset=3`中没有前缀的`768Byte`，`Roll Pointer`后直接跟着`20Byte`的指针
2. `page offset=4`为`溢出页`，存储实际的数据，范围为`0x1002d~0x12355`，总共`9000`，即完全溢出

### UTF8与CHAR

1. `Latin1`与`UTF8`代表了两种编码类型，分别是定长编码和变长编码。
2. `UTF8`对`CHAR(N)`的的处理方式在`Redundant`和`Compact`（或Dynamic、Compressed）中是不一样的：
   - `Redundant`中占用`N * Maximum_Character_Byte_Length`
   - `Compact`中`最小化`占用空间
3. CHAR(N) 中的 N 实际代表的是字符的数量，而不是字节的数量，因此对于`'ab'`，其占用1个字节，`‘我们’`占用 2 个字节，但均是 CHAR(2) 类型。

## `InnoDB`数据页格式

**页结构：**

![img](InnoDB.assets/959658-20200218143934785-1152604727.png)

![img](InnoDB.assets/964149611_1582694852322_DF4BAE61C1D7C81A19ADF6826AD35DDF.png)

| 名称               | 中文名                     | 占用空间 | 简单描述                 |
| ------------------ | -------------------------- | -------- | ------------------------ |
| File Header        | 文件头部                   | 38字节   | 页的一些头信息           |
| Page Header        | 页面头部                   | 56字节   | 数据页的状态信息         |
| Infimum + Supremum | 逻辑上的最小记录和最大记录 | 26字节   | 两个虚拟的行记录         |
| User Records       | 用户记录                   | 不确定   | 实际存储的行记录内容     |
| Free Space         | 空闲空间                   | 不确定   | 页中尚未使用的空间       |
| Page Directory     | 页面目录                   | 不确定   | 页中的某些记录的相对位置 |
| File Trailer       | 文件尾部                   | 8字节    | 校验页是否完整           |

- **各个数据页**之间可以组成一个**双向链表**（就是B+树的各个页之间都按照索引值顺序用双向链表连接起来）
- 而**每个数据页中的记录**又可以组成一个**单向**链表
- 每个数据页都会为存储在它里边的记录生成一个**页目录**，该目录页是用**数组**进行管理，在通过**主键**查找某条记录的时候可以在页目录中使用**二分法快速定位**到对应的槽，然后再遍历该槽对应分组中的记录即可快速找到指定的记录
- 以**其他列**(非主键)作为搜索条件：只能从最小记录开始**依次遍历单链表中的每条记录**。

示意图：

![img](InnoDB.assets/964149611_1582694852901_DF4BAE61C1D7C81A19ADF6826AD35DDF.png)

### 插入记录

核心入口函数在`page_cur_insert_rec_low`。核心步骤如下:

1. 获取记录的长度。函数传入参数就有已经组合好的完整记录，所以只需要从记录的元数据中获取即可。
2. 首先从`PAGE_FREE`链表中尝试获取足够的空间。仅仅比较链表头的一个记录，如果这个记录的空间大于需要插入的记录的空间，则复用这块空间(包括heap_no)，否则就从`PAGE_HEAP_TOP`分配空间。如果这两个地方都没有，则返回空。这里注意一下，由于只判断Free链表的第一个头元素，所以算法对空间的利用率不是很高，估计也是为了操作方便。假设，某个数据页首先删除了几条大的记录，但是最后一条删除的是比较小的记录A，那么后续插入的记录大小只有比记录A还小，才能把Free链表利用起来。举个例子，假设先后删除记录的大小为4K, 3K, 5K, 2K，那么只有当插入的记录小于2K时候，这些被删除的空间才会被利用起来，假设新插入的记录是0.5K，那么Free链表头的2K，可以被重用，但是只是用了前面的0.5K，剩下的1.5K依然会被浪费，下次插入只能利用5K记录所占的空间，并不会把剩下的1.5K也利用起来。这些特性，从底层解释了，为什么InnoDB那么容易产生碎片，经常需要进行空间整理。
3. 如果Free链表不够，就从`PAGE_HEAP_TOP`分配，如果分配成功，需要递增`PAGE_N_HEAP`。
4. 如果这个数据页有足够的空间，则拷贝记录到指定的空间。
5. 修改新插入记录前驱上的next指针，同时修改这条新插入记录的指针next指针。这两步主要是保证记录上链表的连续性。
6. 递增`PAGE_N_RECS`。设置heap_no。设置owned值为0。
7. 更新`PAGE_LAST_INSERT`，`PAGE_DIRECTION`，`PAGE_N_DIRECTION`，设置这些参数后，可以一定程度上提高连续插入的性能，因为插入前需要先定位插入的位置，有了这些信息可以加快查找。详见查找记录代码分析。
8. 修改数据目录。因为增加了一条新的记录，可能有些目录own的记录数量超过了最大值(目前是8条)，需要重新整理一下这个数据页的目录(`page_dir_split_slot`)。算法比较简单，就是找到中间节点，然后用这个中间节点重新构建一个新的目录，为了给这个新的目录腾空间，需要把后续的所有目录都平移，这个涉及一次momove操作(`page_dir_split_slot`和`page_dir_add_slot`)。
9. 写redolog日志，持久化操作。
10. 如果有blob字段，则处理独立的off-page。



### 查找记录/定位位置

B+树索引本身不能找到具体的一条记录，只能找到数据所在的页，然后将页加载到内存中，根据页中的 Page Directory 在进行二分查找。

在`InnoDB`中，需要查找某条件记录，需要调用函数page_cur_search_with_match，但如果需要定位某个位置，例如大于某条记录的第一条记录，也需要使用同一个函数。定位的位置有PAGE_CUR_G，PAGE_CUR_GE，PAGE_CUR_L，PAGE_CUR_LE四种，分别表示大于，大于等于，小于，小于等于四种位置。 由于数据页目录的存在，查找和定位就相对简单，先用二分查找，定位周边的两个目录，然后再用线性查找的方式定位最终的记录或者位置。 

此外，由于每次插入前，都需要调用这个函数确定插入位置，为了提高效率，`InnoDB`针对按照主键顺序插入的场景做了一个小小的优化。因为如果按照主键顺序插入的话，能保证每次都插入在这个数据页的最后，所以只需要直接把位置直接定位在数据页的最后(`PAGE_LAST_INSERT`)就可以了。至于怎么判断当前是否按照主键顺序插入，就依赖`PAGE_N_DIRECTION`，`PAGE_LAST_INSERT`，`PAGE_DIRECTION`这几个信息了，目前的代码中要求满足5个条件：

1. 当前的数据页是叶子节点
2. 位置查询模式为PAGE_CUR_LE
3. 相同方向的插入已经大于3了(`page_header_get_field(page, PAGE_N_DIRECTION) > 3`)
4. 最后插入的记录的偏移量为空(`page_header_get_ptr(page, PAGE_LAST_INSERT) != 0`)
5. 从右边插入的(`page_header_get_field(page, PAGE_DIRECTION) == PAGE_RIGHT`)

## 分区表



# 索引

索引是应用程序设计和开发的一个重要方面。若索引太多，应用程序的性能可能会收到影响，而索引太少，对查询性能又会产生影响。找到一个合适的平衡点，对于应用程序的性能至关重要。

`InnoDB`存储引擎支持以下几种常见的索引：

- B+树
- 全文索引
- 哈希索引

`InnoDB`存储引擎支持的哈希索引是自适应的，会根据表的使用情况自动为表生成哈希索引，不能认为干预是否在一张表生成哈希索引。

## **为什么需要索引**

索引结构的选择基于这样一个性质：**大数据量时，索引无法全部装入内存**。

为什么索引无法全部装入内存？假设使用树结构组织索引，简单估算一下：

- 假设单个索引节点12B，1000w个数据行，unique索引，则叶子节点共占约100MB，整棵树最多200MB。
- 假设一行数据占用200B，则数据共占约2G。

假设索引存储在内存中。也就是说，每在物理盘上保存2G的数据，就要占用200MB的内存，索引:数据的占用比约为1/10。1/10的占用比算不算大呢？物理盘比内存廉价的多，以一台内存16G硬盘1T的服务器为例，如果要存满1T的硬盘，至少需要100G的内存，远大于16G。

考虑到一个表上可能有多个索引、联合索引、数据行占用更小等情况，实际的占用比通常大于1/10，某些时候能达到1/3。**在基于索引的存储架构中，索引:数据的占用比过高，因此，索引无法全部装入内存**。

由于无法装入内存，则必然依赖磁盘（或SSD）存储。而内存的读写速度远远快于磁盘，因此，核心问题是“**如何减少磁盘读写次数**”。

首先不考虑页表机制，假设每次读、写都直接穿透到磁盘，那么：

- 线性结构：读/写平均 O(n) 次
- 二叉搜索树（BST）：读/写平均 O(log~2~N) 次；如果树不平衡，则最差读/写 O(n) 次
- 自平衡二叉搜索树（AVL）：在BST的基础上加入了自平衡算法，读/写最大 O(log~2~N) 次
- 红黑树（RBT）：另一种自平衡的查找树，读/写最大 O(2*log~2~N) 次

假设使用自增主键，则主键本身是有序的，树结构的读写次数能够优化到树高，树高越低读写次数越少；自平衡保证了树结构的稳定。如果想进一步优化，可以引入B树和B+树。

## 数据结构与算法

### BST

二叉搜索数是一棵保证左子树中节点的值全部小于当前节点，右子树中节点的值全部大于当前节点的二叉树。BST的中序遍历会产生一个有序数组。

BST存在的问题是随即插入节点后，左右子树可能会不平衡，且在最差情况下会退化为单链表。

### AVL

AVL是一种平衡二叉树，保证左右子树高度差的绝对值小于等于1。

平衡二叉树可以保证数据查询的时间复杂度为O(log N)，但是再插入节点后，可能需要做多次旋转操作来维持平衡性，产生性能开销，但平衡二叉树多用于内存结构中，所以维护的开销相对较小。

红黑树是一种特殊的平衡二叉树，它是非严格平衡的。将红色节点的链接拉平后，它将与一个2-3树一一对应。

通过引入节点的颜色，红黑树降低了插入节点后平衡操作的次数与开销，相比 AVL 有更高的插入性能，在查询性能上稍微逊色，工业界对红黑树的使用更为广泛，如 Java 的 `HashMap`、Linux 的 `epoll` 等。

### B树

![zxcvvctomz](InnoDB.assets/zxcvvctomz.png)

B树的特点：

1. 关键字集合分布在整颗树中。
2. 任何一个关键字出现且只出现在一个节点中。
3. 搜索有可能在非叶子节点结束。
4. 其搜索性能等价于在关键字集合内做一次二分查找。
5. B树在插入删除新的数据记录会破坏B-Tree的性质，因为在插入删除时，需要对树进行一个分裂、合并、转移等操作以保持B-Tree性质。

### B+树

B+树是为磁盘或其他直接存取辅助设备设计的一种多路平衡查找树，B+树的所有记录都在叶子节点上，并且是顺序存放的，且它们之间通过指针双向连接。

B+树与B树的差异在于

- 有n棵子树的节点含有n个关键字（也有认为是n-1个关键字）。
- 所有的关键字全部存储在叶子节点上，且叶子节点本身根据关键字自小而大顺序连接。
- 非叶子节点可以看成索引部分，节点中仅含有其子树（根节点）中的最大（或最小）关键字。

![231sg4qoi0](InnoDB.assets/231sg4qoi0.png)

一般在数据库系统或文件系统中使用的B+Tree结构都在经典B+Tree的基础上进行了优化，增加了顺序访问指针。

![h4ruwfmjxf](InnoDB.assets/h4ruwfmjxf.png)

<img src="InnoDB.assets/image-20210901134745031.png" alt="image-20210901134745031" style="zoom: 50%;" />



## `MyISAM` 非聚簇索引

`MyISAM`引擎使用B+Tree作为索引结构，叶节点的data域存放的是数据记录的地址。下图是`MyISAM`索引的原理图：

![2dts9dzv31](InnoDB.assets/2dts9dzv31.png)

这里设表一共有三列，假设我们以Col1为主键，则上图是一个`MyISAM`表的主索引（Primary key）示意。可以看出`MyISAM`的索引文件仅仅保存数据记录的地址。在`MyISAM`中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复。如果我们在Col2上建立一个辅助索引，则此索引的结构如下图所示：

![e4sw3eagaz](InnoDB.assets/e4sw3eagaz.png)

同样也是一棵B+树，data域保存数据记录的地址。因此，`MyISAM`中索引检索的算法为首先按照B+Tree搜索算法搜索索引，如果指定的Key存在，则取出其data域的值，然后以data域的值为地址，读取相应数据记录。

`MyISAM`的索引方式也叫做“非聚集”的，之所以这么称呼是为了与`InnoDB`的聚集索引区分。

## `InnoDB`索引实现

`InnoDB`中的 B+ 树索引可以分为聚簇索引和辅助索引。

**聚簇索引**：将数据存储与索引放到了一块，索引结构的叶子节点保存了行数据

**非聚簇索引**：将数据与索引分开存储，索引结构的叶子节点指向了数据对应的位置

在`innodb`中，在聚簇索引之上创建的索引称之为辅助索引，`innodb`中的非聚簇索引都是辅助索引，如复合索引、前缀索引、唯一索引。辅助索引叶子节点存储的不再是行的物理位置，而是主键值，辅助索引访问数据总是需要二次查找。

![img](InnoDB.assets/ce9bedd0dc9013e14e5f450e2149704bef5.jpg)

1. `InnoDB`使用的是聚簇索引，将主键组织到一棵B+树中，而行数据就储存在叶子节点上，若使用"where id = 14"这样的条件查找主键，则按照B+树的检索算法即可查找到对应的叶节点，之后获得行数据。
2. 若对Name列进行条件搜索，则需要两个步骤：第一步在辅助索引B+树中检索Name，到达其叶子节点获取对应的主键。第二步使用主键在主索引B+树种再执行一次B+树检索操作，最终到达叶子节点即可获取整行数据。（重点在于通过其他键需要建立辅助索引）

**聚簇索引具有唯一性**，由于聚簇索引是将数据跟索引结构放到一块，因此一个表仅有一个聚簇索引。

**聚簇索引默认是主键**，如果表中没有定义主键，`InnoDB` 会选择一个**唯一且非空的索引**代替。如果没有这样的索引，`InnoDB` 会隐式定义一个主键来作为聚簇索引。

**使用聚簇索引的优势：**

1. 可以把相关数据保存在一起。        

   例如实现电子邮箱时，可以根据用户ID来聚集数据，这样只需要从磁盘读取少数的数据页就能获取某个用户的全部邮件。如果没有则每封邮件都有可能导致一次磁盘I/O。

2. 数据访问速度快。        

   聚簇索引将索引和数据保存在同一个B+Tree中，因此从聚簇索引中获取数据通常比在非聚簇索引中查找要快。

3. 使用覆盖索引扫描的查询可以直接使用页节点中的主键值。

**缺点：**

1. 聚簇索引最大限度地提高了IO密集型应用的性能，但如果数据全部都放在内存中，则访问的顺序就没么重要了，聚簇索引也就没什么优势了。
2. 插入速度严重依赖插入顺序。按照主键的顺序插入是加载数据到`InnoDB`表中速度最快的方式。但如果不是按照主键顺序加载数据，那么在加载完成后最好使用OPTIMIZE TABLE命令重新组织一下表。当对MySQL进行大量的增删改操作的时候，很容易产生一些碎片，这些碎片占据着空间，所以可能会出现删除很多数据后，数据文件大小变化不大的现象。新插入的数据仍然会利用这些碎片。但过多的碎片，对数据的插入操作是有一定影响的，此时，我们可以通过optimize来对表的优化。
3. 更新聚簇索引列的代价很高，因为会强制`InnoDB`将每个被更新的行移动到新的位置。
4. 基于聚簇索引的表在插入新行，或者主键或者主键被更新导致需要移动行的时候，可能面临“页分裂”的问题。当行的主键值要求必须将这一行插入到某个已满的页中时，存储引擎会将该页分裂成两个页面来容纳该行，这就是一次页分裂操作。页分裂会导致表占用更多的磁盘空间。
5. 二级索引（非聚簇索引）可能比想象的要更大，因为在二级索引的叶子节点包含了引用行的主键列。

**聚簇索引需要注意的地方：**

当使用主键为聚簇索引时，主键最好不要使用`uuid`，因为`uuid`的值太过离散，不适合排序且可能出现新增加记录的`uuid`，会插入在索引树中间的位置，导致索引树调整复杂度变大，消耗更多的时间和资源。

建议使用`int`类型的自增，方便排序并且默认会在索引树的末尾增加主键值，对索引树的结构影响最小。而且，主键值占用的存储空间越大，辅助索引中保存的主键值也会跟着变大，占用存储空间，也会影响到IO操作读取到的数据量。



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

### redo

重做日志用来实现事务的持久性，其由两部分组成：一是内存中的重做日志缓冲（redo log buffer），是易失的；二是重做日志文件（redo log file），是持久的。

`InnoDB`是事务的存储引擎，通过Force Log at Commit机制实现事物的持久性，即当事务提交时，必须先将事物的所有日志写入到重做日志文件中进行持久化。

为了确保每次日志都写入redo log file，在每次将redo log buffer写入redo log file后，`InnoDB`存储引擎都会调用一次`fsync`操作，将缓存池中的文件刷新到磁盘中，磁盘的性能决定了事务提交的性能。

用户可以手动控制日志写入磁盘的策略，让数据库在事务提交时不强制将log刷新到磁盘，但这样做会破坏ACID特性。

将`innodb_flush_log_at_trx_commit`设置为0时，表示每次提交事务不会将日志刷新到磁盘，而是等待Master线程每隔一秒进行一次刷新操作；为1时，表示执行commit时进行一次日志文件的`fsync`操作，将日志刷新到磁盘；为2时，表示仅将redo log在缓冲区进行修改。需要保证ACID中的持久性时，需要将该参数的值设置为1。

**某个事务未提交时，其redo log buffer也可能持久化到redo log文件中。**

在`InnoDB`中，重做日志都是以512字节进行存储的。若一个页中产生的重做日志大小大于512字节，则需要分割为多个redo log block进行存储。因为redo log block的大小与磁盘扇区一样，因此redo log的写入可以保证原子性。



**binary log和redo log的区别：**

- 任何MySQL存储引擎的写入性操作都会记录到bin log中，而redo log只记录`innodb`对数据的修改记录。
- redo log采用循环写入的方式记录；bin log采用追加写入的方式记录，一个文件写满后会生成新的文件。
- redo log适用于数据库崩溃后进行恢复，bin log适用于主从复制和数据恢复。

- bin log只在事务提交前进行一次写入，无论事务多大；而redo log在一条事务执行过程中可能会多次写入。
- bin log记录一条事务的逻辑操作日志，一条事务对应一个记录；redo log记录物理页的修改记录，一条事务可能对应多条日志。

**当binary log比redo log的记录更多时，MySQL会废弃一部分binary log的记录：**例如，①事务提交时，MySQL会将事务的一系列日志按顺序写入磁盘；②然后将事务提交给`InnoDB`。如果在①和②的中间机器宕机，则`InnoDB`会对事务进行回滚，但此时binary log中会存在多余的记录，`InnoDB`会检查binary log中的记录，并将无效记录删除。



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

| 隔离级别                        | 脏读（Dirty Read） | 不可重复读（`NonRepeatable` Read） | 幻读（Phantom Read） |
| ------------------------------- | ------------------ | ---------------------------------- | -------------------- |
| 未提交读（Read uncommitted）    | 可能               | 可能                               | 可能                 |
| 已提交读（Read committed）      | 不可能             | 可能                               | 可能                 |
| **可重复读（Repeatable read）** | **不可**能         | **不可**能                         | **可能**             |
| 可串行化（Serializable ）       | 不可能             | 不可能                             | 不可能               |