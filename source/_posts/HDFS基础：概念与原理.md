---
title: HDFS基础：概念与原理
date: 2017-05-03 00:11:32
categories: [Hadoop,HDFS]
tags: [HDFS]
typora-root-url: ..
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
NameNode的元数据包括：
- 文件存储路径
- NameNode描述
- 副本数量
- 数据块数量
- 数据块存储的物理位置（主机名）
- CRC校验判断文件块是否损坏

HDFS元数据数据以2种形式持久化在本地磁盘中：
- 文件系统镜像(FileSystem Image)：整个文件系统的元数据快照。
- 编辑日志(Edit Log)：NameNode启动后，client对文件系统的改动日志。

## NameNode扩展(NameNode Federation)
NameNode受限与内存，那么【分片】就很有必要了。
**单NameNode架构：**
![HA](/resources/img/hdfs/federation-background.gif)
- Namespace：有且只有一个Namespace/NameNode
- Block Storage Service：
 * Block Management：
   * 通过处理注册和心跳维护DataNodes状态
   * 处理block reports和维护block位置
   * 支持block的增删改查
   * 管理副本放置、块复制以及删除多余副本
 * Storage：DataNodes提供把blocks存储在本地磁盘并给予读写权限的功能。

**NameNode Federation架构：**
![HA](/resources/img/hdfs/federation.gif)
- Namespace Volume：Namespace和它的Block Pool合称为Namespace Volume
 * Namespace：Namespace由一个或多个NameNode组成
 * Block Pool：每个NameNode都有一个pool，每个pool都维护这一些列属于该NameNode的blocks。一个Namespace下的所有pool组成Block Pool。Block Pool维护了整个Namespace的blocks信息。
- ClusterID：用来标识集群中的所有节点。在格式化NameNode的时候，需要指定ClusterID，否则会自动生成。ClusterID会被用于将其他NameNode格式化进集群。
- [ViewFs](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/ViewFs.html)：用于管理HDFS的Namespaces（或者说Namespace Volumes）。

> NameNode Federation是如何进行分片横向扩展的？是否类似于Redis Cluster的Hash分片？
> 显然，由于划分了Namespace，通过Hash(filePath)的方式来分片，那么具有同样业务IO特性的文件有可能跨Namespace，性能降低。
> HDFS使用的方法是ViewFs,类似于Unix/Linux系统的**client-side mount table**。指定Namespace的Pathname Pattern（将Pathname Pattern挂载在某个Namespace），然后将具有相同Pathname Pattern的文件将会定位到指定的Namespace。
> 下图表示将/data、/project、/user、/tmp 一共4个路径挂载到4个不同的命名空间。
> ![ViewFs:client-side mount table](/resources/img/hdfs/viewfs_TypicalMountTable.png)

## DataNode
DataNode是具体读写任务的执行节点：存储文件块，被客户端和NameNode调用。同时通过心跳周期性地向NameNode报告文件块的信息。

## SecondaryNameNode
负责定期合并edit logs到系统文件镜像(namespace image or called FileSystem Image)，以减少edit log的空间。
SecondaryNameNode进行checkpoint备份的数据必然是滞后于NameNode的，如果NameNode完全挂掉，包括磁盘（即丢掉了最新的edit logs）。那么还是会丢掉一部分数据。
Hadoop 2.x通过JournalNodes实现了热备。
PS：SecondaryNameNode并不能实现高可用，NameNode依然是单点的。NameNode故障重启加载元数据（冷启动）也要消耗大量时间。

# HDFS副本策略（机架感知rack-aware）
- 第一个副本在本地机器
- 第二个副本在远端机架
- 第三个副本在本地机架的不同节点
- 其他副本随机放置

# HDFS就近读取策略
HDFS会就近读取文件的副本，而所谓的远近由拓扑距离定义。
同节点，距离为0；同一机架不同节点，距离为2；
相同数据中心不同机架，距离为4；
不同数据中心，距离为6.

# HDFS的安全模式
NameNode启动后会进入安全模式。
处于安全模式的NameNode不会进行数据块的“写”操作，只能进行“读”操作。
NameNode从所有的DataNode接收心跳信号和块状态报告，状态报告包括了某个DataNode所有的block的信息。
每个block都有一个指定的最小副本数，当NameNode确认某个block的副本数量达到最小副本数时，则认为该block是副本安全的。
NameNode确认一定百分比（可配置）的block是副本安全后，NameNode会退出安全模式。
之后，NameNode对于副本数量不足的block会补充合适的副本。

# HDFS High Availability
>JournalNodes
>一组用于NameNode Active和NameNode Standby通信的进程。通过JournalNodes实现HA，也就实现了HDFS的高可用。
>注意JournalNodes必须大于3，且最好是奇数，因为NameNode Active的editlog必须写入超过一半的JournalNodes才算写入成功（分布式一致性算法）。

高可用满足：
- edit logs共享，NameNode Standby可以读取NameNode Active全部的edit logs，之后与NameNode Active保持edit logs同步。
- DataNodes必须向NameNode Standby和NameNode Active同时发送block reports，因为block的映射信息存储于NameNode的内存而不是磁盘。
- 客户端操作HDFS时，故障切换NameNode对于用户透明。
- SecondaryNameNode的checkpoint工作由NameNode Standby取代，NameNode Standby会周期性地checkpoint NameNode Active的namespace。
  ![HA](/resources/img/hdfs/HDFS高可靠性.png)

# Block Caching
通常DataNode从磁盘读取block，但对于频繁访问的文件块，可以显式缓存在DataNode的**堆外内存**中。
默认一个block只缓存在1个DataNode的内存中，不过这个参数可以基于文件进行配置。
比如，需要将一个小表用于join操作，那么用户可以指定NameNode缓存指定的文件（并且指定缓存过期时间），NameNode就会生成一个cache加入到cache pool（落在物理上就是DataNode将block写入堆外内存）。cache pool用于管理缓存的权限和资源使用。

# HDFS的checkpoint
checkpoint指的是SecondaryNameNode合并edit log的过程。
- 1.SecondaryNameNode请求回滚NameNode正在写入的editlogs，并将new editlogs写到一个新文件中去（图中edits_inprogress_20）。NameNode更新它所有的seen_txid文件。
> seen_txid是存放transactionId的文件，format之后是0，它代表的是NameNode里面的edits_*文件的尾数，NameNode重启的时候，会按照seen_txid的数字， 顺序从头跑edits_0000001~到seen_txid的数字
- 2.SecondaryNameNode通过HTTP GET请求获取最新的fsimage and edits files（图中fsimage_0和dits_1-19）。
- 3.SecondaryNameNode加载fsimage至内存，合并editlog。得到一个新的fsimage file（图中fsimage_19.ckpt）。
- 4.SecondaryNameNode通过HTTP PUT将新的fsimage传输到NameNode，保存为一个临时.ckpt文件。
- 5.NameNode重命名临时.ckpt文件使其生效。
  checkpoint的最终结果是：NameNode得到一个最新的fsimage和一个较小的新的editlog文件（checkpoint过程客户端对HDFS的写操作日志）
  ![checkpoint](/resources/img/hdfs/checkpoint.png)

# HDFS读流程
- 1.Client调用FileSystem.open(filePath)
- 2.DistributedFileSystem用RPC去NameNode获取文件块的位置
- 3.open方法返回得到FSDataInputStream，用于流式读取
- 4.Client调用FSDataInputStream.read读取拓扑距离最近的DataNode
- 5.直至FSDataInputStream读取到文件最后的一个block
- 6.读取完毕，关闭输入流

![HDFS读流程](/resources/img/hdfs/read.png)
# HDFS写流程
- 1.Client调用FileSystem.create(filePath)
- 2.DistributedFileSystem通过RPC在NameNode创建一个新的空文件（即没有blocks的文件）
- 3.create方法返回得到FSDataOutputStream，用于流式写入。FSDataOutputStream会将Client写入的数据分割成packets，组成一个`data queue`。`data queue`被`DataStreamer`消费，
  并请求NameNode通过副本策略选择合适的节点来存放数据副本。这些节点组成一个管道pipeline（图中展示的是3个副本的写入操作）。
- 4.`DataStreamer`将packets写入第一个节点，第一个节点将数据复制到第二个节点，依次类推。
- 5.`DFSOutputStream`还维护了一个`ack queue`，`ack queue`里面存放的是未被ack的packets。1个packet被所有节点ack后，就会从`ack queue`里面移除。
- 6.写入完毕，关闭输出流。
  ![HDFS写流程](/resources/img/hdfs/write.png)

> TODO:关于HDFS读写流程就图只能简单解读，如果需要更加深入的了解，可以通过HDFS原生的Client结合HDFS源码来细细研读。
> 还可以结合SpringHadoop研究一下优秀的代码封装(TextFileReader和TextFileWriter)。
> HDFS相关jar是：
```xml
<dependency>
  <groupId>org.springframework.data</groupId>
  <artifactId>spring-data-hadoop-store</artifactId>
  <version>2.4.0.RELEASE</version>
</dependency>
```

# [HDFS命令](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html)
- User Commands
- Administration Commands
- Debug Commands
