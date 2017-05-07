---
title: HBase基础：概念与原理
date: 2017-05-04 22:40:13
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





# 二级索引

# 引用
[深入HBase架构解析（一）](http://www.blogjava.net/DLevin/archive/2015/08/22/426877.html)
[深入HBase架构解析（二）](http://www.blogjava.net/DLevin/archive/2015/08/22/426950.html)
[An In-Depth Look at the HBase Architecture](https://mapr.com/blog/in-depth-look-hbase-architecture/#.VdMxvWSqqko)
[Hbase原理、基本概念、基本架构](http://blog.csdn.net/woshiwanxin102213/article/details/17584043)
[]()
