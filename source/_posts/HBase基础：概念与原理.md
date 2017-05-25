---
title: HBase基础：概念与原理
date: 2017-05-04 22:40:13
categories: [Hadoop,HBase]
tags: [HBase]
---
<Excerpt in index | 首页摘要>
简单介绍HBase里面的一些基础知识和原理实现。<!-- more -->
<The rest of contents | 余下全文>
# HBase特点
HBase是一个分布式、列式存储、稀疏（HBase的列值为null可以不占用空间）的key-value数据库。
- 线性、模块化的可扩展性
- 读写强一致性（HBase是CP系统）
- 表可自动或手动配置分片
- RegionServers之间的自动故障切换支持
- HBase Tables可以很好地支持MR作业
- Java API Client简单易用
- Block cache 和 Bloom Filters用于实时查询
- Thrift网关和支持XML，Protobuf和二进制数据编码的RESTful Webservice
- 可扩展的 jruby-based (JIRB) shell
- Support for exporting metrics via the Hadoop metrics subsystem to files or Ganglia; or via JMX

# HBase Shell
比起HBase shell我个人更加愿意用Web写GUI来替代Shell。

# 基本概念
- rowkey：rowkey便是HBase Table的索引。
 * 散列：常用时间倒序，加盐散列等
 * 预分区：相当于手工设定负载均衡
 * rowkey一般根据业务设定，唯一性，字典ASCII排序
- Column Family：存储在HDFS上的一个单独文件中，列的空值不会被保存。
- Column
- Version Number：类型为Long，默认值是系统时间戳，可由用户自定义
- Value(Cell)：Byte array
- Region：Region是RegionServers负载均衡的最小单元，表按行分片得到Region。Region可自动分裂，也可以预分区。
- Table
>在一个HBase Table中：
 * 表是行的集合
 * 行是列族的集合
 * 列族是列的集合
 * 列是键值对的集合
下图可以是一个稀疏矩阵，且为null的列不会占用空间。注意：HBase的列不属于建表的元数据，这才使得列为null可以不占用存储空间，但代价是每个Cell都需要包含列的具体信息，还有版本号之类的其他信息。
![表结构](/resources/img/hbase/表结构.png)

# HBase架构
## HBase架构图
![HBase架构](/resources/img/hbase/HBase架构.png)
![HBase架构](/resources/img/hbase/HBase架构_1.png)
HBase是Master-Slave集群架构。
HBase由HMaster节点、HRegionServer节点、ZooKeeper集群节点组成，而底层存储则基于HDFS。
HBase Client通过RPC方式和HMaster、HRegionServer通信；一个HRegionServer可以存放X个HRegion；底层Table数据存储于HDFS中，而HRegion所处理的数据尽量和数据所在的DataNode在一起，实现数据的本地化；数据本地化并不是总能实现，比如在HRegion移动(如因Split)时，需要等下一次Compact才能继续回到本地化。
- HMaster
 * 管理HRegionServer，实现其负载均衡。
 * 管理和分配HRegion，比如在HRegion split时分配新的HRegion；在HRegionServer退出时迁移其内的HRegion到其他HRegionServer上。
 * 实现DDL操作（Data Definition Language，namespace和table的增删改，column familiy的增删改等）。
 * 管理namespace和table的元数据（实际存储在HDFS上）。
 * 权限控制（ACL）。
- HRegionServer
 * 存放和管理本地HRegion。
 * 读写HDFS，管理Table中的数据。
 * Client直接通过HRegionServer读写数据（从HMaster中获取元数据，找到RowKey所在的HRegion/HRegionServer后）。
- ZooKeeper集群是协调系统
 * 存放整个 HBase集群的元数据以及集群的状态信息。
 * 实现HMaster主从节点的failover。(主从故障自动切换)

## HRegion(即Region)
HBase使用RowKey将表水平切割成多个HRegion，从HMaster的角度，每个HRegion都纪录了它的StartKey和EndKey（第一个HRegion的StartKey为空，最后一个HRegion的EndKey为空），由于RowKey是排序的，因而Client可以通过HMaster快速的定位每个RowKey在哪个HRegion中（BloomFilter）。HRegion由HMaster分配到相应的HRegionServer中，然后由HRegionServer负责HRegion的启动和管理，和Client的通信，负责数据的读(使用HDFS)。每个HRegionServer可以同时管理1000个左右的HRegion（这个数字怎么来的？没有从代码中看到限制，难道是出于经验？超过1000个会引起性能问题？来回答这个问题：感觉这个1000的数字是从BigTable的论文中来的（5 Implementation节）：Each tablet server manages a set of tablets(typically we have somewhere between ten to a thousand tablets per tablet server)）。
- RegionServers负载Region
HBase的Table按行分片（分区）为Region，每个Region属于一个RegionServers。每个RegionServers管理着多个Region。
RegionServer是HBase的数据服务进程。负责处理用户数据的读写请求。
由这个图就可以了解到rowkey为何要散列，因为散列后rowkey落在不同的Region，才能在不同的RegionServer负载均衡。
并且Region可以在RegionServer之间发生转移。
![RegionServers负载Region](/resources/img/hbase/RegionServers负载Region.png)
再看此图：Table被水平（按行）切分为Region，其依据便是Rowkey。属于一种**RangePartitioner**。每个Region都有一个StartRowkey和EndRowkey。
![RegionServers负载Region](/resources/img/hbase/RegionServers负载Region_1.png)
- 元数据Region(Meta Region)
Meta Region记录了每一个UserRegion的路由信息。
Client读写Region数据的路由：
 * 找寻Meta Region地址。
 * 再由Meta Region找寻UserRegion地址。
- 用户Region(User Region)

## HMaster
HMaster没有单点故障问题，可以启动多个HMaster，通过ZooKeeper的Master Election机制保证同时只有一个HMaster出于Active状态，其他的HMaster则处于热备份状态。一般情况下会启动两个HMaster，非Active的HMaster会定期的和Active HMaster通信以获取其最新状态，从而保证它是实时更新的，因而如果启动了多个HMaster反而增加了Active HMaster的负担。前文已经介绍过了HMaster的主要用于HRegion的分配和管理，DDL(Data Definition Language，既Table的新建、删除、修改等)的实现等，既它主要有两方面的职责：
- 协调HRegionServer
     * 启动时HRegion的分配，以及负载均衡和修复时HRegion的重新分配。
     * 监控集群中所有HRegionServer的状态(通过Heartbeat和监听ZooKeeper中的状态)。
- Admin职能
    * 创建、删除、修改Table的定义。
![HMaster](/resources/img/hbase/HMaster.png)

## ZooKeeper：协调者
ZooKeeper为HBase集群提供协调服务，它管理着HMaster和HRegionServer的状态(available/alive等)，并且会在它们宕机时通知给HMaster，从而HMaster可以实现HMaster之间的failover，或对宕机的HRegionServer中的HRegion集合的修复(将它们分配给其他的HRegionServer)。ZooKeeper集群本身使用分布式一致性协议(PAXOS协议)保证每个节点状态的一致性。
![zookeeper](/resources/img/hbase/zookeeper.png)

## How The Components Work Together

# Minor Compaction/Major Compaction

# HBase读流程

# HBase写流程

# [HBase的ACID](http://hbase.apache.org/acid-semantics.html)
ACID是指原子性(Atomicity)，一致性(Consistency)，隔离性(Isolation)和持久性(Durability)
- 原子性(Atomicity)：整个事务中的所有操作，要么全部完成，要么全部不完成，不可能存在中间状态。
- 一致性(Consistency)：事务提交后，所有读操作都可以读到事务提交后的一致的数据。
- 隔离性(Isolation)：并发事务之间相互隔离，互不影响。
- 持久性(Durability)：在事务提交后，该事务对数据库所作的更改便持久的保存在存储介质之中，并且不可变。
在HBase里面，提供有限度的ACID特性：
- 原子性：行级锁，保证行更改时的原子行，对某一行的更改操作要么全部成功，要么全部失败。
- 一致性：1行数据存在于Region，而Region属于1个RegionServer。该行数据更新后，所有读操作都会定位到此Region读取唯一的最新的数据。所以数据一致。
- 隔离性：行与行的事务互不影响，同行事务由于“写行”操作的原子性，那么并发写必然是串行的，也必然是隔离的。
- 持久性：事务一旦提交，WAL数据便认为不可破坏。即使宕机导致MemStore数据丢失，从WAL也能恢复数据。
* HBase写时流程：
  * 锁行，相同行无法并发写操作
  * 获取WriteNumber（相当于事务版本号）
  * 写WAL
  * 写MemStore
  * 更新ReadPoint为最新的WriteNumber
  * 释放锁
* HBase读时流程：
  * 打开Scanner
  * 获取最新ReadPoint
  * 顺序读取BlockCache、MemStore、HFile，直至取得数据
  * 关闭Scanner

# HBase的锁
## 行锁RowLock
用于实现“写行”时的原子性。RowLock有Lease租约的机制，超时会自动释放行锁。（即使是超时释放行锁，整个操作也是要么全部成功，要么全部失败，保持原子性）
> MVCC:Multi-Version Concurrency Control 多版本并发控制
参考HBase读写流程的WriteNumber和ReadPoint，读写版本分离。

## MemStore锁
对Store的写操作会调用Memstore的相关操作，在对memstore做snapshot以及清除snapshot的时候会阻塞其他操作(如add、delete、getNextRow)。

## Region锁
在做更新操作时，需要判断资源是否满足要求，如果到达临界点，则申请进行flush操作，等待直到资源满足要求（参见Region中的checkResource）
Region update更新锁(updatesLock)，在internalFlushCache时加写锁，导致在做Put、delete、increment操作时候阻塞(这些操作加的是读锁)。
Region close保护锁(lock)，在Region close或者split操作的时(加写锁)，阻塞对region的其他操作(加读锁)，比如compact、flush、scan和其他写操作。

## StoreFile锁
flush过程包括，
    a 、prepare(基于memstore做snapshot)
    b、flushcache(基于snapshot生成临时文件)
    c、commit(确认flush操作完成，rename临时文件为正式文件名称，清除mem中的snapshot)
其中在flush过程的commit阶段，compact过程的completeCompaction阶段(rename临时compact文件名、清理旧的文件)，close store(关闭store)，bulkLoadHFile，会阻塞对store的写操作。

# 协处理器Coprocessor
## Observer(类似触发器)

## EndPoint(类似与存储过程)

# 二级索引
## Coprocessor Observer实时同步构建

## 写队列异步构建

## ETL(MapReduce/Spark/DataX等)批量离线构建

# 引用
[深入HBase架构解析（一）](http://www.blogjava.net/DLevin/archive/2015/08/22/426877.html)
[深入HBase架构解析（二）](http://www.blogjava.net/DLevin/archive/2015/08/22/426950.html)
[An In-Depth Look at the HBase Architecture](https://mapr.com/blog/in-depth-look-hbase-architecture/#.VdMxvWSqqko)
[Hbase原理、基本概念、基本架构](http://blog.csdn.net/woshiwanxin102213/article/details/17584043)
[Apache HBase Internals: Locking and Multiversion Concurrency Control](https://blogs.apache.org/hbase/entry/apache_hbase_internals_locking_and)
[总结一下HBase各种级别的锁以及对读写的阻塞](http://blog.csdn.net/yangbutao/article/details/12950083)
[]()
