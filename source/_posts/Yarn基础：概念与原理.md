---
title: Yarn基础：概念与原理
date: 2017-05-14 17:25:16
categories: [Hadoop,Yarn]
tags: [Yarn]
---
<Excerpt in index | 首页摘要>
简单介绍Yarn里面的一些基础知识和原理实现。<!-- more -->
<The rest of contents | 余下全文>
# 基本概念
## ResourceManager
负责集群资源统一管理和计算框架管理，主要包括调度与应用程序管理
- 调度器
根据容量、队列等限制条件，将系统中的资源分配给各个正在运行的应用程序
- 应用程序管理器
负责管理整个系统中的所有应用程序，包括应用程序提交，与调度器协商资源，启动并监控AppMaster运行状态

## NodeManager
节点资源管理监控和容器管理，RM是系统中将资源分配给各个应用的最终决策者。
## AppMaster
各种计算框架的实现（例如MRAppMaster）向ResourceManager申请资源，通知NodeManager管理相应的资源。
## Container
Yarn中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等。
