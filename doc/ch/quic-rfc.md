# QUIC : 基于UDP的多路复用安全传输

## 摘要
本文描述了QUIC传输协议的核心部分，附件文档中描述QUIC的丢包检测，拥塞控制，以及TLS密钥使用。   


## 读者须知   

本草案在QUIC工作组的邮件列表中讨论，邮件见https://mailarchive.ietf.org/arch/search/?email_list=quic   
工作组信息见https://github.com/quicwg   
本草案的源码和问题列表见https://github.com/quicwg/base-drafts/labels/-transport   


## 目录

## 1.简介
QUIC是一种多路复用和安全的通信协议   
它提供以下特性：   
+ 流的复用   
+ 连接级别的流量控制   
+ 低延迟的建连耗时   
+ 连接迁移和NAT重新绑定的弹性   
+ 验证和加密的头部信息和负载  
    
QUIC底层使用UDP以避免客户端操作系统和通信中间设备的改动。QUIC认证所有的头部信息，加密传输的大部分数据，包括它的控制信号，以避免对通信的中间设备产生依赖。     
### 1.1文档结构
本文描述了QUIC协议的核心部分，本文结构如下：   
+ Stream是QUIC提供的基础服务抽象
  - 第2章 描述与Stream相关的核心概念
  - 第3章 提供流状态的参考模型
  - 第4章 概述了流量控制的操作
+ 连接是QUIC端点通信过程中的实体
  - 第5章 描述了连接相关的核心概念   
  - 第6章 描述了版本协商过程   
  - 第7章 是关于连接建立过程的细节   
  - 第8章 指点关键的服务拒绝环节机制   
  - 第9章 描述了端点如何将连接迁移到另一个新的网络环境中   
  - 第10章 列出了中断连接时的所有选项   
  - 第11章 提供对错误处理的一般指导   
+ 包和帧是QUIC通信的基本单元
  - 第12章 描述了关于包和帧的关键概念   
  - 第13章 定义了传输，重传和确认的模型   
  - 第14章 定义了包大小管理的规则   
+ 最后，展示了一些QUIC编码的细节
  - 第15章 版本   
  - 第16章 整型编码   
  - 第17章 包头   
  - 第18章 传输参数   
  - 第19章 传输帧   
  - 第20章 错误定义    
附录文件描述了本文未尽的一些QUIC细节，包括[拥塞控制](https://tools.ietf.org/html/draft-ietf-quic-recovery-27)，以及[QUIC-TLS](https://tools.ietf.org/html/draft-ietf-quic-recovery-27)。   

## 1.2 术语和定义
关键字 "**一定**", "**禁止**", "**要求**", "**应该**", "**不应**", "**推荐**", "**不推荐**", "**可以**", "**可选**"的使用与[BCP14](https://tools.ietf.org/html/bcp14) [RFC2119](https://tools.ietf.org/html/rfc2119) [RFC8174](https://tools.ietf.org/html/rfc8174)相同。   
本文常用的术语定义如下：    
QUIC：本文描述的传输协议，QUIC只是一个名字，而不是一个缩略词。    
QUIC包：QUIC可以封装进UDP数据包的完整处理单元，多个QUIC包可以被放到一个UDP数据包中。   
ACK引发包：除了ACK，PADDING，CONNECTING外，其他包都包含的一种QUIC包，它使得接受方发送接受确认。    
乱序包：一种数据包，它的编号并不是接受方最大编号加一，当包传输延迟或者该包之前的包延迟或丢失，都可能造成包的乱序到达。   
Endpoint：QUIC连接中生成，接收，处理QUIC包的实体。QUIC中只有客户端和服务器两种Endpoint。   
客户端：主动建立连接的Endpoint。    
服务器：接收连接请求的Endpoint。     
Address：ip版本，ip 地址，UDP协议，和UDP端口的元组代表了网络通信的一个终端。    
连接ID：在EndPoint中用一个唯一的ID来标识一个连接，每个EndPoint发送包到对端的时候都会将这个值写入包内。   
Stream：在QUIC连接之上，传输有序数据的单向或双向通道，一个QUIC连接可以同时携带多个Stream。   
应用：使用QUIC发送和接收数据的实体。   
   
## 1.3 符号约定   
本文使用的包和帧的格式定义类似Section 3.1 of [RFC2360](https://tools.ietf.org/html/rfc2360#section-3.1)，约定如下：   
[x]：表示x是可选的   
x (A): 表示x长度为A bits   
x (A/B/C): 表示x的长度为A，B或C bits   
x (i)…：表示x使用变长编码   
x (*): 表示x是变长的   

## 2 Streams   
Stream 是QUIC提供给应用的轻量级有序数据传输的功能抽象。Stream 可以是单向的也可以是双向的，QUIC单向流的另一种观点是几乎无限长的“消息”抽象。   
Streams可以被创建用来发送数据，与Streams管理相关的其他过程(结束，取消，管理流控)，都是为了施加最小的开销而设计的。比如，一个Stream帧可以创建，关闭Streams或者携带数据。Streams也可以是长存的，直到QUIC连接结束。   
Steams可以被任意一端EndPoint创建，可以交替着和其他Streams一起发送数据，也可以被取消。QUIC没有提供任何实际的机制来保证不同Streams的数据传输有序。不同Streams间的数据是乱序传输的。   
QUIC允许操作任意数量的Streams，在任意一个Streams上发送任意数量的数据，这取决于流控和流量限制。   
## 2.1 Steam的类型和ID
Streams可以是单向或双向的，单向的Streams只能往一个方向发送数据：从Stream的发起方到对端。双向的Stream可以在两个方向上发送数据。   
在一个连接内，一个数字值作为Stream ID来标识一个Stream，Stream ID是一个62Bit长度的整数，它在一个连接内是独特唯一的。Stream ID按照变长整数进行编码，QUIC EndPoint **禁止**在一个连接内复用 Stream ID。   
Stream ID的最后一个bit位用来标识发起方，客户端发起的Stream，设置为0，服务器发起的Stream，设置为1。   
Stream ID的倒数第二个bit位用来标识Stream数据发送方向，双向时为0，单向时为1。   
Stream ID最后两位的bit组合共有四种类型，定义如下：   
| Bits | Stream Type      |
| :--: | :--------------: |
| 0x0  | 客户端发起, 双向 |
| 0x1  | 服务端发起, 双向 |
| 0x2  | 客户端发起, 单向 |
| 0x3  | 客户端发起, 双向 |
   
在每种类型中，Stream ID都是递增创建的，无序使用的流ID会导致该类型的所有流同时打开编号较低的流ID(存疑)。    
客户端打开的第一个双向流的Stream ID为0。     
    
## 2.2 发送和接收数据    
Stream帧封装应用发送的数据，一个EndPoint使用Stream ID和偏移量字段来时数据有序。    
EndPoint一定保证将Stream 数据有序的传输给上层应用，这就需要EndPoint缓存无序到达的数据，直到流量控制的上限。     
QUIC对Stream乱序传输的数据并没有特别的考虑，但是，在实现上也可以将乱序的数据透传给上层应用。   
一个EndPoint可能会在一个Stream上接受到多次相同偏移的数据，其中一些早已经接受过的数据可以丢弃掉，当数据发生重传时，它的偏移量**禁止**被修改。当一个EndPoint在一个Stream上相同偏移接收到不同数据时，会视为连接错误：PROTOCOL_VIOLATION。   
对QUIC而言，Stream是独立的有序传输数据的抽象，当数据被传输、丢包后重新传输或传送到应用程序时，Stream帧边界不期望被持有。   
EndPoint**禁止**在任何还没有被对端设置流控限制的Stream上发送数据，流控的详细内容见第四章。   
    
## 2.3 Stream优先级
如果分配给流的资源的优先级正确，流复用可以对应用程序性能产生显著影响。    
QUIC并没有自己的信息优先级机制，它依赖上层使用QUIC的应用来设置不同的优先级。    
QUIC的实现**应该**提供一种方式来使上层应用可以设置Stream的优先级，并且**应当**使用应用层的信息来设置不同Stream的传输优先级。

## 2.4 Stream所需操作
应用层在使用QUIC Stream时，**一定**要执行某些操作，本文并没有定义具体的API，但是这个版本的QUIC任何实现都应该提供本节中描述的Stream操作。   
在Stream发送部分，应用层应该能够：   
+ 写数据，了解何时流控信息被设置然后开始发送写入的数据。   
+ 结束Stream(完整的终止)，将发送设置了FIN标识的Stream帧。   
+ 重置Stream(突然终止)，当Stream尚未处于终端状态时，发送RESET_STREAM帧。   
   
在接受端。应用程序能够：   
+ 读取数据。   
+ 终止读取并且请求关闭，可能会发送STOP_SENDING帧。   
   
Stream的状态改变需要通知到应用层，包括对端打开或关闭了一个Stream，对端终止读取一个Stream，新的数据可读，流控引起的数据可写或不可写。   


