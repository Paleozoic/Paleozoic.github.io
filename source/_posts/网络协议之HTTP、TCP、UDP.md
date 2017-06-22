---
title: 网络协议之HTTP、TCP、UDP
date: 2017-05-15 00:33:47
categories: [计算机网络]
tags: [HTTP,TCP,UDP]
typora-root-url: ..
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

| 层     | 描述                                       | 例子                       |
| ----- | ---------------------------------------- | ------------------------ |
| 应用层   | 这层协议是为了满足特定应用的通信需求而设计的，通常定义一个服务接口        | HTTP、FTP、STMP            |
| 表示层   | 这层协议将以一种网络表示传输数据，这种表示与计算机使用的表示无关，两种表示可能完全不相同。如果需要，可以在这一层对数据进行加密 | TLS安全、CORBA数据表示          |
| 会话层   | 在这层要实现可靠性和适应性，例如故障检测和自动恢复                | SIP                      |
| 传输层   | 这是处理消息（而不是数据包）的最低一层。消息被定位到与进程相连的通信端口上。这层协议是可以面向连接的，也可以是无连接的。 | TCP、UDP                  |
| 网络层   | 在特定网络的计算机间传输数据包，在一个WAN或一个互连网络中，这一层负责生成一个通过路由器的路径。在单一的LAN中不需要路由 | IP、ATM虚电路                |
| 数据链路层 | 负责再有直接物理连接的结点间传输数据包。在WAN中，传输是在路由器间或路由器或主机间进行的。在LAN中，传输是在任意一对的主机间进行的 | Ethernet MAC、ATM信元传送、PPP |
| 物理层   | 指驱动网络的电路和硬件。它通过发送模拟信号传输二进制数据序列，用电信号的振幅或频率调制信号（在电缆电路上），光信号（在光纤电路上），或其他电磁信号（在无线电和微波电路上） | Ethernet基带信号、ISDN        |

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
- 同步序号：Synchronize Sequence Numbers，SYN，用来标识TCP发送端向接收端发送的数据字节流，它表示在这个报文段中的第一个数据字节。一共32位，SYN>2<sup>32</sup>－1后从0开始计数。
- 确认序号：Acknowledgement Number，ACK，上次已成功收到数据字节序号加1
- TCP状态位
  * SYN：SYNchronous，同步标识
  * ACK：ACKnowledgement，确认标识
  * URG：URGent，紧急指针（urgent pointer）有效
  * PSH：PuSH，接收方应尽快将这个报文段交给应用层
  * RST：ReSeT，重建连接
  * FIN：FINish，发送端完成发送任务

## TCP三次握手与四次挥手
![TCP_open_close](/resources/img/network/TCP_open_close.jpg)
![TCP_open_close](/resources/img/network/TCP_open_close1.png)
看到[一篇blog](http://www.cnblogs.com/guguli/p/4520921.html)通过FSM(Finite State Machine,有限状态机)伪代码描述TCP传输过程。
> PS：此blog的FSM图和算法都是中文描述，非常清晰。

先看FSM图：
![TCP_FSM](/resources/img/network/TCP_FSM.png)
![TCP_FSM](/resources/img/network/TCP_FSM1.png)
- TCB：Transmit Control Block，传输控制模块，它用于记录TCP协议运行过程中的变量。对于有多个连接的TCP，每个连接都有一个TCB。
  TCB结构的定义包括这个连接使用的源端口、目的端口、序号、应答序号、对方窗口大小、己方窗口大小、TCP状态、TCP的重传有关变量。
- RTS：Request-to-send
- CTS：Clear-to-send
- Segment：传输的数据报文

| State       | Description                              |
| ----------- | ---------------------------------------- |
| CLOSED      | No connection exists                     |
| LISTEN      | Passive open received; waiting for SYN   |
| SYN-SENT    | SYN sent; waiting for ACK                |
| SYN-RCVD    | SYN+ACK sent; waiting for ACK            |
| ESTABLISHED | Connection established; data transfer in progress |
| FIN-WAIT-1  | First FIN sent; waiting for ACK          |
| FIN-WAIT-2  | ACK to first FIN received; waiting for second FIN |
| CLOSED-WAIT | First FIN received, ACK sent; waiting for application to close |
| TIME-WAIT   | Second FIN received, ACK sent; waiting for 2MSL time-out |
| LAST-ACK    | Second FIN sent; waiting for ACK         |
| CLOSING     | Both sides decided to close simultaneously |

```java
TCP_Main_Module (Segment){
  Search the TCB Table //搜索TCB Table
  if (corresponding TCB is not found){//TCB不存在
    Create a TCB with the state CLOSED //创建TCB，状态为CLOSED
  }
  Find the state of the entry in the TCB table //获取TCB的state
  switch (state){
      case CLOSED state:{  //CLOSED状态
          if ("passive open" message received){ //收到“被动打开”报文，进入LISTEN状态
            go to LISTEN state.
          }
          if ("active open" message received){//收到“主动打开”报文
            send a SYN segment //发送SYN信号
            go to SYN-SENT state //进入SYN-SENT状态
          }
          if (any segment received){//收到报文
            send an RST segment //发送RST(Reset)信号
          }
          if (any other message received){//收到其他报文
            issue an error message //报告错误响应
          }
          break;
      }
      case LISTEN state:{
          if ("send data" message received) {
            Send a SYN segment //发送SYN信号
            Go to SYN-SENT state //进入SYN-SENT状态
          }
          if (any SYN segment received){
            Send a SYN + ACK segment
            Go to SYN-RCVD state
          }
          if (any other segment or message received){
            Issue an error message
          }
          break;
      }
      case SYN-SENT state:{
          if (time-out){
            Go to CLOSED state
          }
          if (SYN segment received){
            Send a SYN + ACK segment
            Go to SYN-RCVD state
          }
          if (SYN + ACK segment received){
            Send an ACK segment
            Go to ESTABLISHED state
          }
          if (any other segment or message received){
            Issue an error message
          }
          break;
      }
      case SYN-RCVD state:{
        if (an ACK segment received){
          Go to ESTABLISHID state
        }
        if (time-out){
          Send an RTS segment
          Go to CLOSED state
        }
        if ("close" message received){
          Send a FIN segment
          Go to FIN-WAIT-I state
        }
        if (RTS segment received){
           Go to LISTEN state
        }
        if (any other segment or message received){
          Issue an error message
        }
        break;
      }
      case ESTABLISHED state:{
        if (a FIN segment received){
          Send an ACK segment
          Go to CLOSED-WAIT state
        }
        if ("close" message received){
          Send a FIN segment
          Go to FIN-WAIT-I
        }
        if (a RTS or an SYN segment received){
          Issue an error message
        }
        if (data or ACK segment received){
          call the input module
        }
        if ("send" message received){
          call the output module
        }
        break;
      }
      case FIN-WAIT-1 state:{
        if (a FIN segment received){
          Send an ACK segment
          Go to CLOSING state
        }
        if (a FIN + ACK segment received){
          Send an ACK segment
          Go to FIN-WAIT state
        }
        if (an ACK segment received){
          Go to FIN-WAIT-2 state
        }
        if (any other segment or message received){
          Issue an error message
        }
        break;
      }
      case FIN-WAIT-2 state:{
        if (a FIN segment received){
          Send an ACK segment
          Go to TIME-WAIT state
        }
        break;
      }
      case CLOSING state:{
        if (an ACK segment received){
          Go to TIME-WAIT state
        }
        if (any other message or segment received){
          Issue an error message
        }
        break;
      }
      case TIME-WAIT state:{
        if (time-out){
          Go to CLOSED state
        }
        if (any other message or segment received){
          Issue an error message
        }
        break;
      }
      case CLOSED-WAIT state:{
        if ("close" message received){
          Send a FIN segment
          Go to LAST-ACK state
        }
        if (any other message or segment received){
          Issue an error message
        }
        break;
      }
      case LAST-ACK state:{
        if (an ACK segment received){
          Go to CLOSED state
        }
        if (any other message or segment received){
          Issue an error message
        }
        break;
      }
    }
} // end module
```
### 初始化
Server监听指定端口，等待Clinet的申请建立TCP连接

### 三次握手（建立连接）
#### 握手过程
- 第一次握手：Client发送SYN seq=x给Server请求确认。Client状态：SYN_SEND

- 第二次握手：Server回复ACK=x+1确认收到Client的请求。Client->Server连接建立。并发送SYN seq=x，请求Client建立连接。Server状态：SYN_RECV

- 第三次握手：Client回复ACK=y+1确认收到Server的请求。Server->Client连接建立。至此双方连接建立完毕，可以开始传输数据
  由于TCP是全双工的，需要至少三次握手来同步双方的SYN序号，如此才能保证Client<->Server的数据是有序、准确的。(Client顺序发送，Server顺序接收)

  所以说，三次握手是保证TCP可靠传输的前提。

#### 网络异常分析
可通过上面的FSM伪代码分析握手失败的处理。
- 第[1,2]次握手失败，Client收不到Server的SYN+ACK，超时后关闭连接
- 第[2,3]次握手失败，Server收不到Client的ACK，超时后发送RTS报文段，进入CLOSED状态
- 超时CLOSED：Lease机制的运用，超时释放资源，防止死锁。或者说资源被占用时间过长。
- SYN Flood攻击：攻击者恶意申请大量的TCP连接，使Server存在大量等待SYN-ACK的TCP连接，占用系统资源。这些资源都只能等待超时后释放。

### 数据传输
#### 传输过程
如图所示：
- 首先执行三次握手：1-3步，初始化了Client的seq(SYN)=1，Server的ack(SYN)=1
- Client发送数据包，seq=1，ACK=1，数据包Len=1440
- Client发送数据包，seq=1441，ACK=1，数据包Len=1440
- Server接收数据包，返回seq=1，ACK=1441
- Client发送数据包，seq=1441+1440=2881，ACK=1，数据包Len=1440
- Client发送数据包，seq=2881+1440=4321，ACK=1，数据包Len=1440
- Server接收数据包，返回seq=1，ACK=2881
- ……(重复)
- 注意：Client顺序发送数据包（seq number），Server必然顺序ACK数据包（ack number）。但不是一应一答的阻塞。（**猜想**应存在缓冲区，先缓存一个时间窗口的数据包，Server再顺序ACK）
  ![TCP_数据传输](/resources/img/network/TCP_transport_example.jpg)


由上图可以看出：Seq和Ack共同保证了数据的可靠性（顺序传输和顺序确认）。而三次握手分别初始化了Seq和Ack，这便是可靠传输的前提。

#### 流量控制（接收方流量控制）

#### 目的

接收方通过接收窗口，让发送方的发送速率不要太快，要让接收方来得及接收。突出的是端到端的流量控制。

接收窗口：rwnd，receiver window/advertised window

假设Client(Sender)->Server(Receiver)发送数据，Server会告诉Client一个接收窗口，比如rwnd=400字节，那Client会向Server发送400字节的数据。

假设Client每次向Server发送100字节的数据，第一次Server收到100字节后会返回rwnd=300字节...一直循环至rwnd=0。

此时Client会等待Server重新发送一个rwnd。

- 如果Server->Client发送的rwnd=200丢失了呢？Client会不会一直等待Server发送非0 rwnd？而Server也在等待Client发送数据？（死锁出现）如何解决死锁？

  TCP维护了一个计时器。TCP接到rwnd=0，就启动一个Timer。Timer到期会向Server发送请求非0 rwnd。若Server返回的rwnd仍为0，则重置Timer，直至rwnd>0。


- 如果是Client->Server发送的数据丢失了呢？

  Server会重新发送rwnd给Client。

![TCP_flow_control](/resources/img/network/TCP_flow_control.png)

#### 拥塞控制（发送方流量控制）

在某段时间，若对网络中某资源的需求超过了该资源所能提供的可用部分，网络的性能就要变坏——产生拥塞(congestion)。 出现资源拥塞的条件： 对资源需求的总和 > 可用资源。若网络中有许多资源同时产生拥塞，网络的性能就要明显变坏，整个网络的吞吐量将随输入负荷的增大而下降。目的是防止网络过载，强调的是整个网络环境的资源控制。

拥塞窗口：cwnd，congestion window

- 慢启动

  发送方维持一个拥塞窗口，拥塞窗口的大小取决于网络的拥塞程度，根据此动态变化。发送方维持cwnd=rwnd。考虑到接收方的流量控制，cwnd=rwnd

- 拥塞避免

- 拥塞发生

- 快速恢复


### 四次挥手（断开连接）
- 第一次挥手：Client发送seq=x+2，ACK=y+1。请求Server关闭连接。Client状态：FIN-WAIT-1
- 第二次挥手：Server确认ACK x+3。Server状态：CLOSED-WAIT
- 第三次挥手：Server发送seq=y+1，请求Client关闭连接。Server状态：LAST-ACK
- 第四次挥手：Client收到Server的关闭连接请求。进入状态TIME-WAIT。并发送ACK=y+2确认关闭连接。

# HTTP(HyperText Transfer Protocol,超文本传输协议)

# 基于Java NIO的封装
- Mina
- Netty

# 引用
[计算机网络11--OSI参考模型](http://blog.csdn.net/u014581901/article/details/50733888)
[第11章 UDP:用户数据报协议](http://docs.52im.net/extend/docs/book/tcpip/vol1/11/)
[第17章 TCP：传输控制协议](http://docs.52im.net/extend/docs/book/tcpip/vol1/17/)
《分布式系统：概念与设计》
《TCP/IP Protocol Suite Fourth Edition》
[TCP 的那些事儿（上）](http://coolshell.cn/articles/11564.html)
[TCP 的那些事儿（下）](http://coolshell.cn/articles/11564.html)
[<再看TCP/IP第一卷>TCP/IP协议族中的最压轴戏----TCP协议及细节](http://www.cnblogs.com/guguli/p/4520921.html)
[传输控制协议](https://zh.wikipedia.org/wiki/传输控制协议)
[TCP流量控制](http://www.cnblogs.com/13224ACMer/p/6414620.html)
[TCP拥塞控制](http://www.cnblogs.com/13224ACMer/p/6415892.html)
