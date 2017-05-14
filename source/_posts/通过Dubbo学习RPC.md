---
title: 通过Dubbo学习RPC
date: 2017-05-09 23:00:17
categories: [RPC,Dubbo]
tags: [RPC,Dubbo]
---
<Excerpt in index | 首页摘要>
RPC基础，通过Dubbo的设计学习RPC框架的基本组成。<!-- more -->
<The rest of contents | 余下全文>

# Dubbo依赖关系
![Dubbo依赖关系](/resources/img/rpc/dubbo依赖关系.jpg)
由dubbo可以看到一个基本的RPC框架设计和依赖。
- Provider向注册中心注册服务(pub)
- Consumer订阅注册中心消息(sub)
- Provider向注册中心注册服务时会被注册中心推送至Consumer
- Consumer通过注册中心获取到服务的注册信息，比如调用地址等。Consumer通过调用地址列表做负载均衡（客户端负载均衡），然后调用Provider
- 监控

# Dubbo支持协议(序列化以及网络传输)
![dubbo-protocol](/resources/img/rpc/dubbo-protocol.jpg)
dubbo将对象序列化，包括header(codec)（序列化编码方式，可选）和body(serialization)（对象序列化后的内容，二进制或者字符串）。
Client通过网络传输，将序列化内容发送给服务端。
## dubbo协议
Dubbo缺省协议采用单一长连接和NIO异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。
Dubbo缺省协议不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。
- 序列化协议：
- 网络传输协议：

## rmi协议
- 序列化协议：
- 网络传输协议：

## hessian协议
- 序列化协议：
- 网络传输协议：

## http协议
- 序列化协议：
- 网络传输协议：

## webservice协议
- 序列化协议：
- 网络传输协议：

## thrift协议
- 序列化协议：
- 网络传输协议：

## memcached协议
- 序列化协议：
- 网络传输协议：

## redis协议
- 序列化协议：
- 网络传输协议：

# 注册中心（服务注册与发现）(zookeeper)

# 负载均衡
## 服务端负载均衡(nginx)

## 客户端负载均衡(dubbo)


# 监控


# 引用
[Dubbo开发者指南](http://dubbo.io/Developer+Guide-zh.htm)
[Dubbo架构设计详解](http://shiyanjun.cn/archives/325.html)
