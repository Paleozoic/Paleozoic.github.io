---
title: HDFS基础：概念与原理
date: 2017-05-03 00:11:32
tags: [HDFS]
---
<Excerpt in index | 首页摘要>
简单介绍HDFS里面的一些基础知识和原理实现。<!-- more -->
<The rest of contents | 余下全文>

# HDFS特点
- 支持：
 * 处理超大文件(Very large files)：处理几百MB、TB甚至PB级别的数据。
 * 流式地访问数据(Streaming data access)：HDFS设计模式基于“一次写入，多次读取”(write-once, read-many-times)。数据集由数据源生成或复制至HDFS，然后被用于分析。一般来说，这种数据分析应该基于整个数据集。所以对于HDFS来说，设计上应该考虑读取整个数据集的性能而不是读取其中一条数据的性能。
 * 商用硬件(Commodity hardware)：可以运行于廉价的商用硬件集群上。
- 不支持：
 * 低延迟数据访问(Low-latency data access)：HDFS不适用于低延迟的数据访问，比如数百ms的响应时间。HDFS强调的是吞吐量，如果需要实时的数据访问，基于HDFS的HBase才是合适的技术。
 * 海量小文件(Lots of small files)：HDFS会将文件系统的元数据加载到NameNode的内存。NameNode的内存限制了文件存储的数量。每个文件的元数据大约占用150 bytes空间。
  PS：在HDFS 2.x，已经支持NameNode分片扩展(NameNode Federation)，类似于Redis Cluster。
 * 多用户写入，文件任意修改(Multiple writers, arbitrary file modifications)：在HDFS文件只能被一个用户写入，并且只能是文件末尾追加。不支持多用户写入以及通过偏移量修改文件任意位置。

# 基本概念
## block块
block是HDFS存储（读写）文件的最小单位（类似于windows文件系统NFS的簇，或Linux文件系统的块）。
block的默认大小在Hadoop 1.x是64M，在Hadoop 2.x是128M。
HDFS将文件分割存储在各个block中，而block则分布在各个DataNode中。
HDFS默认是3个副本。
HDFS中文件大小如果小于block，并不会占用整个block的空间。
HDFS将集群当做一个整体，每个DataNode划分了很多的block（类比Windows将磁盘看做一个整体，在磁盘上划分了很多的簇），数据便存储在这些block里面。
注意：NFS的簇是Windows操作系统的逻辑概念，HDFS的块是Hadoop应用的逻辑概念。
二者所属层次不同，NFS是操作系统提供支持，HDFS是应用提供支持。不过二者硬件底层都是磁盘提供物理存储支持。
>为什么HDFS的block单位如此大？
- 文件被切分时，块的数量少，HDFS访问定位（相当于“磁盘寻道”）、聚合（从各个block读取数据）都会较快。
- bolck的大小设置由磁盘的传输速率决定。假设磁盘寻道时间是10ms，传输速度是100 MB/s。让寻道时间为传输时间的1%，那么传输时间是1s。
所以块大小应为：传输速率*传输时间，即100MB/s*1s = 100M

## NameNode
HDFS是典型的Master-Slave(Worker)模式，NameNode便是Master。
NameNode管理这文件系统的命名空间，维护整个文件系统的文件目录树以及这些文件的索引目录（概括来说是维护HDFS的元数据）。
NameNode的元素据包括：
- 文件存储路径
- NameNode描述
- 副本数量
- 数据块数量
- 数据块存储的物理位置（主机名）
- CRC校验判断文件块是否损坏

HDFS元数据数据以2种形式持久化在本地磁盘中：
- 文件系统镜像(FileSystem Image)：
- 编辑日志(Edit Log)：

## NameNode扩展(NameNode Federation)

## DataNode
DataNode是具体读写任务的执行节点：存储文件块，被客户端和NameNode调用。同时通过心跳周期性地向NameNode报告文件块的信息。

## SecondaryNameNode
负责定期合并系统文件镜像(namespace image)

## JournalNodes

# HDFS HA(Since 2.x)
如果配置了HA，则JournalNodes会取代SecondaryNameNode

# HDFS副本策略（机架感知rack-aware）
- 第一个副本在本地机器
- 第二个副本在远端机架
- 第三个副本在本地机架的不同节点
- 其他副本随机放置

# HDFS就近读取策略
HDFS会就近读取文件的副本，而所谓的远近由拓扑距离定义。
同节点，距离为0；同一机架不同节点，距离为2；
相同数据中心不同机架，距离为4；
不同数据中信，距离为6.

# HDFS的安全模式
NameNode启动后会进入安全模式。
处于安全模式的NameNode不会进行数据块的“写”操作，只能进行“读”操作。
NameNode从所有的DataNode接收心跳信号和块状态报告，状态报告包括了某个DataNode所有的block的信息。
每个block都有一个指定的最小副本数，当NameNode确认某个block的副本数量达到最小副本数时，则认为该block是副本安全的。
NameNode确认一定百分比（可配置）的block是副本安全后，NameNode会退出安全模式。
之后，NameNode对于副本数量不足的block会补充合适的副本。

# 文件安全(NameNode安全)

# HDFS HA
