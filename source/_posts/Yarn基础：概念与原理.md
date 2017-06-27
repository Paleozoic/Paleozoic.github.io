---
title: Yarn基础：概念与原理
date: 2017-05-14 17:25:16
categories: [Hadoop,Yarn]
tags: [Yarn]
typora-root-url: ..
---
<Excerpt in index | 首页摘要>
简单介绍Yarn里面的一些基础知识和原理实现。<!-- more -->
<The rest of contents | 余下全文>
# 基本概念
Yarn也是Master-Slaves的设计模式，Master是ResourceManager，Slaves便是NodeManager。
## ResourceManager
负责集群资源统一管理和计算框架管理，主要包括调度与应用程序管理，RM是系统中将资源分配给各个应用的最终决策者
- 调度器
  根据容量、队列等限制条件，将系统中的资源分配给各个正在运行的应用程序
- 应用程序管理器
  负责管理整个系统中的所有应用程序，包括应用程序提交，与调度器协商资源，启动并监控AppMaster运行状态

## NodeManager
负责监控可用资源、报告故障情况和管理容器的生命周期。
## ApplicationMaster
AM理解为一个作业的“头”，它管理这个作业的声明周期，包括动态增加和减少资源的使用；管理执行流程（例如针对不同的Map输出调用Reducers）；
处理故障和计算偏差(computation skew)；以及执行其它的本地优化。
由于RM和NM的通信协议是可扩展的通信协议，所以AM支持任意语言的代码。
## Container
Yarn中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等。

# Yarn资源分配流程
给一个Spark通过Yarn Cluster提交作业的流程原理图，看图即可。
![SparkYarnCluster](/resources/img/spark/SparkYarnCluster.png)

# 引用
[Apache Hadoop YARN: Yet another resource negotiator](https://blog.acolyer.org/2017/01/09/apache-hadoop-yarn-yet-another-resource-negotiator/)
