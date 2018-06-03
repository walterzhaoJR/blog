# TIDB和MySQL事务隔离级别对比

## 1事务ACID概念

* A(Atomicity) : 原子性，事务中的一系列操作要么全部完成，要么全部不完成，不能做了一半不做了。这个最好理解，比如转账不能扣完A的钱，不给B加钱。
* I(Isolation) : 隔离性，多个事务之间相互隔离的特性。首先不同业务对事务的隔离等级要求不一样，有的严格要求隔离，有的并不是那么严格。因此数据库系统都会实现多种隔离级别，从技术角度讲，每种隔离级别都需要不同的技术手段来保证，通常来说涉及各种锁和MVCC机制。
* D(Durability) : 持久性，事务一旦提交，所修改的数据就会被持久化，后面即使发生任何异常都不会出现数据丢失。这个容易理解，要么数据直接落盘，要么数据操作日志落盘。但是通常情况下数据库系统也一般会根据数据重要性提供多种持久化策略供客户端选择使用，比如对于重要数据，就会要求数据同步落盘之后才能算事务完成，这是最严格的持久化策略；而对于部分不重要数据，可能只会要求数据异步落盘就算事务完成。
* C(Consistency) : 一致性，要求事务必须始终保持系统处于一致的状态。比如A转账给B，转账前账户总额和转账后账户总和需要保持一致。



## 2隔离性简述

* 数据操作过程中利用数据库的锁机制或者多版本并发控制机制获取更高的隔离等级。但是，随着数据库隔离级别的提高，数据的并发能力也会有所下降。所以，如何在并发性和隔离性之间做一个很好的权衡就成了一个至关重要的问题。
* 数据库事务的隔离级别有4个，由低到高依次为Read uncommitted、Read committed、Repeatable read、Serializable，这四个级别可以逐个解决脏读、不可重复读、幻读这几类问题。
* 我们讨论隔离级别的场景，主要是在多个事务并发的情况下。
* sql 92标准定义了4种隔离级别，读未提交、读已提交、可重复读、串行化，见下表：

| Isolation Level  | Dirty Read   | Nonrepeatable Read | Phantom Read | Serialization Anomaly |
| ---------------- | ------------ | ------------------ | ------------ | --------------------- |
| Read uncommitted | Possible     | Possible           | Possible     | Possible              |
| Read committed   | Not possible | Possible           | Possible     | Possible              |
| Repeatable read  | Not possible | Not possible       | Not possible | Possible              |
| Serializable     | Not possible | Not possible       | Not possible | Not possible          |

## 3.TIDB的事务隔离级别(官方文档)

* 根据官方文档：TIDB实现了读已提交和可重复读。 
* TiDB 使用[percolator事务模型](https://research.google.com/pubs/pub36726.html)，当事务启动时会获取全局读时间戳，事务提交时也会获取全局提交时间戳，并以此确定事务的执行顺序，如果想了解 TiDB 事务模型的实现可以详细阅读以下两篇文章：[TiKV 的 MVCC（Multi-Version Concurrency Control）机制](https://pingcap.com/blog-cn/mvcc-in-tikv/)，[Percolator 和 TiDB 事务算法](https://pingcap.com/blog-cn/percolator-and-txn/)。 

### 3.1TIDB的可重复读

* 可重复读是 TiDB 的默认隔离级别，当事务隔离级别为可重复读时，只能读到该事务启动时已经提交的其他事务修改的数据，未提交的数据或在事务启动后其他事务提交的数据是不可见的。对于本事务而言，事务语句可以看到之前的语句做出的修改。
* 对于运行于不同节点的事务而言，不同事务启动和提交的顺序取决于从 PD 获取时间戳的顺序。
* 处于可重复读隔离级别的事务不能并发的更新同一行，当时事务提交时发现该行在该事务启动后，已经被另一个已提交的事务更新过，那么该事务会回滚并启动自动重试。

### 3.2与 ANSI 可重复读隔离级别的区别

* 尽管名称是可重复读隔离级别，但是 TiDB 中可重复读隔离级别和 ANSI 可重复隔离级别是不同的，按照[A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf)论文中的标准，TiDB 实现的是论文中的 snapshot 隔离级别，该隔离级别不会出现幻读，但是会出现写偏斜，而 ANSI 可重复读隔离级别不会出现写偏斜，会出现幻读。 

### 3.3与MySQL可重复读隔离级别的区别

* MySQL 可重复读隔离级别在更新时并不检验当前版本是否可见，也就是说，即使该行在事务启动后被更新过，同样可以继续更新。这种情况在 TiDB 会导致事务回滚并后台重试，重试最终可能会失败，导致事务最终失败，而 MySQL 是可以更新成功的。 MySQL 的可重复读隔离级别并非 snapshot 隔离级别，MySQL 可重复读隔离级别的一致性要弱于 snapshot 隔离级别，也弱于 TiDB 的可重复读隔离级别。 

### 3.4读已提交

* 读已提交隔离级别和可重复读隔离级别不同，它仅仅保证不能读到未提交事务的数据，需要注意的是，事务提交是一个动态的过程，因此读已提交隔离级别可能读到某个事务部分提交的数据。
* 不推荐在有严格一致要求的数据库中使用读已提交隔离级别。

### 3.5事务重试

* 对于 insert/delete/update 操作，如果事务执行失败，并且系统判断该错误为可重试，会在系统内部自动重试事务。 

## 4.Mysql和TIDB的隔离级别对比

### 4.1Read uncommitted

* 读未提交，顾名思义，就是一个事务可以读取另一个未提交事务的数据。

* 未提交读的数据库锁情况:

  * 事务在读数据的时候并未对数据加锁。
  * 事务在修改数据的时候只对数据增加行级共享锁。

* 表现

  * 事务1读取某行记录时，事务2也能对这行记录进行读取、更新；当事务2对该记录进行更新时，事务1再次读取该记录，能读到事务2对该记录的修改版本，即使该修改尚未被提交。
  * 事务1更新某行记录时，事务2不能对这行记录做更新，直到事务1结束。

  

![](http://hbasefly.com/wp-content/uploads/2017/07/1111.png)



* 刚开始，1号事务和2号事务看到的A都是A0，接着1号事务将A0更新为A1，再接着2号事务就读到A1新值，1号事务将A1回滚回了A0。

* 对比（set session transaction isolation level read uncommitted;）

  * MySQL 

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/78388fce90211fd4e23ccc3da6784821d162629ce321eeb547f101b51b4ee5b7392c264f1c005a337784a982fd1672e3?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=mysql_read_uncommitted.png&size=1024)

  * TIDB

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/f8920ab5305629ecb48964834372791887e24013891368a07edbee88117d54c03393d43a515781608ab9f890859e3e51?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=tidb_read_uncommitted.png&size=1024)

### 4.2 read committed

* 提交读，顾名思义，就是一个事务要等另一个事务提交后才能读取数据。 
* 提交读的数据库锁情况:

  *  事务对当前被读取的数据加 行级共享锁（当读到时才加锁），一旦读完该行，立即释放该行级共享锁；
  *  事务在更新某数据的瞬间（就是发生更新的瞬间），必须先对其加 行级排他锁，直到事务结束才释放。
* 很显然，Read Committed是与Read Uncommitted是相对的，意思是说1号事务可以在2号事务提交之后看到2号事务修改的数据。这种隔离级别可以避免脏读，但是又引入了一个新的问题：不可重复读，如下图所示：

![](http://hbasefly.com/wp-content/uploads/2017/07/1112.png)

* 但是从上面的例子中我们也看到，事务一两次读取的结果并不一致，所以提交读不能解决不可重复读的读现象。

* 简而言之，提交读这种隔离级别保证了读到的任何数据都是提交的数据，避免了脏读。但是不保证事务重新读的时候能读到相同的数据，因为在每次数据读完之后其他事务可以修改刚才读到的数据。

* 对比（set session transaction isolation level read committed;）

  * MySQL:

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/64f09c7fa6f602da6c5cb63bb2c491a94c86261b10749af329da6e08e2138837cd663a9cfb4539ad4acbb5f5be48537f?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=mysql_read_committed.png&size=1024)

  * TIDB:

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/0684e4e7b68cbdac7d7ec47659f7fe3ab5c3c2b07b269e0ff355c6066ae18af11af9a87296aedc2ef8f18782b385b5dc?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=tidb_read_committed.png&size=1024)

### 4.3.repeatable read

* 重复读，就是在开始读取数据（事务开启）时，不再允许修改操作 。
* 重复读可以解决不可重复读问题。不可重复读对应的是修改，即UPDATE操作。但是可能还会有幻读问题。因为幻读问题对应的是插入INSERT操作，而不是UPDATE操作。 
* 可重复读的数据库锁情况
  * 事务在读取某数据的瞬间（就是开始读取的瞬间），必须先对其加 行级共享锁，直到事务结束才释放。
  * 事务在更新某数据的瞬间（就是发生更新的瞬间），必须先对其加 行级排他锁，直到事务结束才释放。
* 从字面意思来看这种隔离级别修复了不可重复读这样的问题，表现如下图所示：

![](http://hbasefly.com/wp-content/uploads/2017/07/1113.png)

* 可以看出，无论1号事务如何更新A，2号事务在随后的进程中看到的A值都是事务开始第一次看到的A值（A0）。虽然解决了不可重复读的问题，但是还有一个问题－幻读： 

![](http://hbasefly.com/wp-content/uploads/2017/07/1114.png)

* 上图中1号事务在事务过程中插入了一个大于B0的新值B2，2号事务在插入操作前后读取B > 0的时候读到的值却不同。

* 对比1（set session transaction isolation level repeatable read;）

  * MySQL和TIDB行为一致：

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/19d22f6a71eb6d5982a4fef792ac1f7bfceb38171885ffb9bfb8b3ee1449d90a3b8835deef1b3e0cb048dc6d2afd91c2?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=repeatable.png&size=1024)

  

* 对比2（set session transaction isolation level repeatable read;）

  * MySQL:

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/02be2a64c6911e46d6df34e2180e891a5e0bc55d50f2a55c3c71a3c22b2ac1cd9354d3868b147ae9b63ae3f662d1798e?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=mysq_repeatable_a.png&size=1024)

  * TIDB:

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/4a0414a3dc0deb7e6f1a77d72f311388544c9327cb3591af8fa571c3e9fa4f82af4a9de8707a952391bd2ab3ee6ea88e?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=tidb_repeatable_a.png&size=1024)


* 对比3（set session transaction isolation level repeatable read;）

  * MySQL：

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/26a55772dbebc6fd49fe78db13abcbd7971303e27c700f80f5637f2a69fa7a5ac5c51bc6c81f02f20aa897f9976196cf?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=mysql_repeatable_b.png&size=1024)

  * TIDB：

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/4885301d7f2136469148e46ed08e0f06bad18a53a04d885328026214b40ebe60e2cf5b3919aca1f5b384ff096437fcdc?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=tidb_repeatable_b.png&size=1024)

* 对比4（set session transaction isolation level repeatable read;）

  * MySQL:

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/c7f533d5071d65f5cd67c532f27ffc8565692dc9793ba82376ba3eba43c9ca7f3cbbd0836f80253f7a8969c933787fbc?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=mysql_repeatable_c.png&size=1024)

  * TIDB:

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/6f3366dae1687fb448968acf5130d4b9df9b3b919a6d1a7b2a20871aa792997be937cf0847d3c0bdfe97e008c2bb48fc?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=tidb_repeatable_c.png&size=1024)

* 对比5（set session transaction isolation level repeatable read;）

  * MySQL:

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/62e9d7d27b1c06593ec32fa97d7cd972018995924a108d61aa54b441315bebccba98bb29de026409f383a91975ac7fcf?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=mysql_repeatable_d.png&size=1024)

  * TIDB:

  ![](https://picabstract-preview-ftn.weiyun.com:8443/ftn_pic_abs_v2/21e20b1a2f8a23d0eb3e797e1ce503037571dc69958c03ad3042219b77d2f9fb19b850915cacd93c2096461b26419789?pictype=scale&from=30113&version=2.0.0.2&uin=519362600&fname=tidb_repeatable_d.png&size=1024)



### 4.4Serializable

* 串性化是隔离最严格的一种形式，要求有读写冲突的事务必须严格串行执行。如下图所示，2号事务要读取1号事务修改的记录A，这就导致2号事务必须等待1号事务提交之后才能开启执行。通过这种形式可以避免之前所提到脏读、不可重复读和幻读。虽说如此，几乎所有数据库业务都不会开启这种隔离级别，因为这会带来严重的锁冲突。

* 可序列化的数据库锁情况
  * 事务在读取数据时，必须先对其加 表级共享锁 ，直到事务结束才释放
  * 事务在更新数据时，必须先对其加 表级排他锁 ，直到事务结束才释放。

![](http://hbasefly.com/wp-content/uploads/2017/07/1115.png)

#### 参考文献

*   [TiDB 事务隔离级别](https://www.pingcap.com/docs-cn/sql/transaction-isolation/)

