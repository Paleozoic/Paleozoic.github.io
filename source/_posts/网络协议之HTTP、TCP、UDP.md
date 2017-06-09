---
title: 网络协议之HTTP、TCP、UDP
date: 2017-05-15 00:33:47
categories: [计算机网络]
tags: [HTTP,TCP,UDP]
---
<Excerpt in index | 首页摘要>
介绍网络协议：HTTP、TCP、UDP<!-- more -->
<The rest of contents | 余下全文>
# 写在前面
此篇blog仅作了解计算机网络的一些基础知识。如果对于网络协议有更深入的追求。
强烈推荐[TCP/IP详解：卷I：协议](http://www.52im.net/topic-tcpipvol1.html)

# OSI模型
OSI即Open System Interconnection，开放系统互联。
OSI定义了7层协议栈。

|层|描述|例子|
|--|--|--|
|应用层|这层协议是为了满足特定应用的通信需求而设计的，通常定义一个服务接口|HTTP、FTP、STMP|
|表示层|这层协议将以一种网络表示传输数据，这种表示与计算机使用的表示无关，两种表示可能完全不相同。如果需要，可以在这一层对数据进行加密|TLS安全、CORBA数据表示|
|会话层|在这层要实现可靠性和适应性，例如故障检测和自动恢复|SIP|
|传输层|这是处理消息（而不是数据包）的最低一层。消息被定位到与进程相连的通信端口上。这层协议是可以面向连接的，也可以是无连接的。|TCP、UDP|
|网络层|在特定网络的计算机间传输数据包，在一个WAN或一个互连网络中，这一层负责生成一个通过路由器的路径。在单一的LAN中不需要路由|IP、ATM虚电路|
|数据链路层|负责再有直接物理连接的结点间传输数据包。在WAN中，传输是在路由器间或路由器或主机间进行的。在LAN中，传输是在任意一对的主机间进行的|Ethernet MAC、ATM信元传送、PPP|
|物理层|指驱动网络的电路和硬件。它通过发送模拟信号传输二进制数据序列，用电信号的振幅或频率调制信号（在电缆电路上），光信号（在光纤电路上），或其他电磁信号（在无线电和微波电路上）|Ethernet基带信号、ISDN|

两台主机在网络中通信的示意图：
![OSI_通信示意图](/resources/img/network/OSI_通信示意图.png)

OSI模型详情图：
![OSI](/resources/img/network/TCP_IP.png)

# UDP(User Datagram Protocol,用户数据报协议)
## 特点
- 面向无连接。即发送数据前不需要建立连接
- 不可靠传输，可能发生丢包，并且传输可能乱序
- UDP占用较少系统资源，协议信息简单
常见运用：网络视频、语音聊天、直播等

## UDP封装
IP是网络层协议，UDP是网络层上一层的传输层协议。所以UDP数据报与IP首部会组成一个IP数据报：
![UDP封装](/resources/img/network/UDP_data.png)

## UDP首部
![UDP首部](/resources/img/network/UDP_head.png)
- 源端口号：发送进程
- 目的端口号：接收进程
- UDP长度：指UDP首部和UDP数据的字节长度
- UDP校验和：可选，UDP报文可没有UDP校验和。覆盖UDP首部和UDP数据

# TCP(Transmission Control Protocol,传输控制协议)
## 特点
- TCP为应用层提供全双工服务。这意味数据能在两个方向上独立地进行传输。
- 面向连接，可靠传输，保证传输顺序、完整

## TCP封装
IP是网络层协议，TCP是网络层上一层的传输层协议。所以TCP数据报与IP首部会组成一个IP数据报：
![TCP封装](/resources/img/network/TCP_data.png)

## TCP首部
![TCP首部](/resources/img/network/TCP_head.png)
- 源端口号：发送进程
- 目的端口号：接收进程
- TCP校验和：必选，TCP报文必须有TCP校验和。覆盖TCP首部和TCP数据
- 序号：SYN，用来标识TCP发送端向接收端发送的数据字节流，它表示在这个报文段中的第一个数据字节。一共32位，SYN>2<sup>32</sup>－1后从0开始计数。
- 确认序号：ACK，上次已成功收到数据字节序号加1
- URG：紧急指针（urgent pointer）有效
- PSH：接收方应尽快将这个报文段交给应用层
- RST：重建连接
- FIN：发送端完成发送任务

## TCP三次握手与四次挥手
![TCP_open_close](/resources/img/network/TCP_open_close.jpg)
### 初始化
Server监听指定端口，等待Clinet的申请建立TCP连接

### 三次握手（建立连接）
- 第一次握手：Client发送SYN seq=x给Server请求确认。Client状态：SYN_SEND
- 第二次握手：Server回复ACK=x+1确认收到Client的请求。Client->Server连接建立。并发送SYN seq=x，请求Client建立连接。Server状态：SYN_RECV
- 第三次握手：Client回复ACK=y+1确认收到Server的请求。Server->Client连接建立。至此双方连接建立完毕，可以开始传输数据
由于TCP是全双工的，需要至少三次握手来确定双方的SYN序号，如此才能保证Client<->Server的数据是有序、准确的。

### 网络异常分析
- 第一次握手失败，Client周期性重发，直至Server ACK
- 第二次握手失败，Server周期性重发
- 第三次握手失败，Server周期性重发

### 数据传输

### 四次回收（断开连接）

# HTTP(HyperText Transfer Protocol,超文本传输协议)

# 基于Java NIO的封装
- Mina
- Netty


# 引用
[计算机网络11--OSI参考模型](http://blog.csdn.net/u014581901/article/details/50733888)
[第11章 UDP:用户数据报协议](http://docs.52im.net/extend/docs/book/tcpip/vol1/11/)
[第17章 TCP：传输控制协议](http://docs.52im.net/extend/docs/book/tcpip/vol1/17/)
《分布式系统：概念与设计》
[TCP 的那些事儿（上）](http://coolshell.cn/articles/11564.html)
