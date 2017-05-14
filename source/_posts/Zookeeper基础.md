---
title: Zookeeper基础
date: 2017-05-06 20:19:22
categories: [Hadoop,Zookeeper]
tags: [Zookeeper]
---
<Excerpt in index | 首页摘要>
简单介绍Zookeeper的知识。<!-- more -->
<The rest of contents | 余下全文>
## Zookeeper是什么？
Zookeeper是一个分布式协调服务器，基于paxos实现了Zab一致性算法。
## Zookeeper提供了什么？
- 文件系统
> 对于zk文件系统，在zk文件系统的目录树中，每个目录都被乘坐znode。
  * znode可有子目录，并且可存放数据，但是EPHEMERAL类（临时znode）znode除外。
  * znode有版本机制，即znode可以存储多个版本的数据。
  * znode类型：
    * PERSISTENT(持久化目录节点):client与zk断开连接后，该节点依旧存在
    * PERSISTENT_SEQUENTIAL(持久化顺序编号目录节点):client与zk断开连接后，该节点依旧存在。且创建新节点时，会被顺序编号
    * EPHEMERAL(临时目录节点):区别于PERSISTENT，client断开连接后，znode会被删除
    * EPHEMERAL_SEQUENTIAL(临时顺序编号目录节点):区别于PERSISTENT_SEQUENTIAL，client断开连接后，znode会被删除
- 发布/订阅通知机制
  * client可以订阅（watch）指定的znode，当znode发生改变时，zk会发布消息通知client。（Redis也有类似的pub/sub机制）基于这个机制zk和redis都可以实现分布式锁以及服务注册与发现（比如dubbo）
## Zookeeper提可以做什么？
- 服务注册与发现：dubbo
- 配置管理：disconf
- 集群管理：HBase Storm等
- 分布式锁
- 分布式队列
- 进程屏障Barrier
### 服务注册与发现
作为注册中心，zk大致流程如下，可以实现客户端的负载均衡：
![Zookeeper的服务注册与发现](/resources/img/zookeeper/Zookeeper的服务注册与发现.png)
我没有看过dubbo源码，但是根据配置文件，可以大致推出以下结论：
- dubbo:reference：消费者将会监听zookeeper下关于reference下的znode，消费者获得他所有reference的服务注册在zk的信息，维护成一个表。以此实现客户端的负载均衡。
- dubbo:service：生产者将service注册到指定znode
- registry：指定了注册中心的地址，服务注册和发现的address
- dubbo:protocol：指定了RPC的传输和序列化协议
```xml
<!-- 提供方应用信息，用于计算依赖关系 -->
<dubbo:application name="hello-world-app"  />
<!-- 使用multicast广播注册中心暴露服务地址 -->
<dubbo:registry address="zookeeper://224.5.6.7:1234" />
<!-- 用dubbo协议在20880端口暴露服务 -->
<dubbo:protocol name="dubbo" port="20880" />
<!-- 声明需要暴露的服务接口 -->
<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService" />
<!-- 声明需要引用的服务接口 -->
<dubbo:reference interface="com.foo.BarService" actives="10" />
<!-- 和本地bean一样实现服务 -->
<bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl" />
```
