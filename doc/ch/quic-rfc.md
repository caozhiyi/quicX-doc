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
  - 第2节 描述与Stream相关的核心概念
  - 第3节 提供Stream状态的参考模型
  - 第4节 概述了流量控制的操作
+ 连接是QUIC端点通信过程中的实体
  - 第5节 描述了连接相关的核心概念   
  - 第6节 描述了版本协商过程   
  - 第7节 是关于连接建立过程的细节   
  - 第8节 指点关键的服务拒绝环节机制   
  - 第9节 描述了端点如何将连接迁移到另一个新的网络环境中   
  - 第10节 列出了中断连接时的所有选项   
  - 第11节 提供对错误处理的一般指导   
+ 包和帧是QUIC通信的基本单元
  - 第12节 描述了关于包和帧的关键概念   
  - 第13节 定义了传输，重传和确认的模型   
  - 第14节 定义了包大小管理的规则   
+ 最后，展示了一些QUIC编码的细节
  - 第15节 版本   
  - 第16节 整型编码   
  - 第17节 包头   
  - 第18节 传输参数   
  - 第19节 传输帧   
  - 第20节 错误定义    
附录文件描述了本文未尽的一些QUIC细节，包括[拥塞控制](https://tools.ietf.org/html/draft-ietf-quic-recovery-27)，以及[QUIC-TLS](https://tools.ietf.org/html/draft-ietf-quic-recovery-27)。   

### 1.2 术语和定义
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
   
### 1.3 符号约定   
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
### 2.1 Steam的类型和ID
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
| 0x3  | 服务端发起, 单向 |
   
在每种类型中，Stream ID都是递增创建的，无序使用的流ID会导致该类型的所有流同时打开编号较低的流ID(存疑)。    
客户端打开的第一个双向流的Stream ID为0。     
    
### 2.2 发送和接收数据    
Stream帧封装应用发送的数据，一个EndPoint使用Stream ID和偏移量字段来使数据有序。    
EndPoint一定保证将Stream 数据有序的传输给上层应用，这就需要EndPoint缓存无序到达的数据，直到流量控制的上限。     
QUIC对Stream乱序传输的数据并没有特别的考虑，但是，在实现上也可以将乱序的数据透传给上层应用。   
一个EndPoint可能会在一个Stream上接收到多次相同偏移的数据，其中一些早已经接受过的数据可以丢弃掉，当数据发生重传时，它的偏移量**禁止**被修改。当一个EndPoint在一个Stream上相同偏移接收到不同数据时，会视为连接错误：PROTOCOL_VIOLATION。   
对QUIC而言，Stream是独立的有序传输数据的抽象，当数据被传输、丢包后重新传输或传送到应用程序时，Stream帧边界不期望被持有。   
EndPoint**禁止**在任何还没有被对端设置流控限制的Stream上发送数据，流控的详细内容见第四节。   
    
### 2.3 Stream优先级
如果分配给流的资源的优先级正确，Stream 复用可以对应用程序性能产生显著影响。    
QUIC并没有自己的信息优先级机制，它依赖使用QUIC的上层应用来设置不同的优先级。    
QUIC的实现**应该**提供一种方式来使上层应用可以设置Stream的优先级，并且**应当**使用应用层的信息来设置不同Stream的传输优先级。

### 2.4 Stream所需操作
应用层在使用QUIC Stream时，**一定**要执行某些操作，本文并没有定义具体的API，但是这个版本的QUIC任何实现都应该提供本节中描述的Stream操作。   
在Stream发送部分，应用层应该能够：   
+ 写数据，了解何时流控信息被设置然后开始发送写入的数据。   
+ 结束Stream(完整的终止)，将发送设置了FIN标识的Stream帧。   
+ 重置Stream(突然终止)，当Stream尚未处于终端状态时，发送RESET_STREAM帧。   
   
在接收端。应用程序能够：   
+ 读取数据。   
+ 终止读取并且请求关闭，可能会发送STOP_SENDING帧。   
   
Stream的状态改变需要通知到应用层，包括：   
- 对端打开或关闭了一个Stream
- 对端终止读取一个Stream
- 新的数据可读
- 流控引起的数据可写或不可写 

## 3 Stream状态
本节讲述Stream的发送或接收组件。有两种状态机：   
+ 数据发送端
+ 数据接收端
  
单向的流直接使用合适的状态机，但是双向的Stream每个端都有两种状态机。在大多数情况下，单向或双向Stream对状态机的使用都是相同的。双向Stream的打开要复杂一些，因为不论是接收端还是发送端，都需要打开两个方向的数据传输。   
一个EndPoint**必须**在一个Stream type上递增的创建Stream ID。   
注意： 这些状态的信息量很大，本文使用Stream状态来描述何时以及如何发送不同类型的帧的规则，以及接收到不同类型帧时预期的反应。理解了这些状态机对实现QUIC非常有用。实现时可以定义不同的状态机，只要其实现与本文定义一致。   

### 3.1 Stream的发送状态
下图展示了Stream发送数据到对端的部分状态： 
```
    o    
    | 创建 Stream (Sending)    
    | 对端创建双向Stream    
    v    
+-------+    
| Ready | Send RESET_STREAM    
|       |-----------------------.    
+-------+                       |    
    |                           |       
    | Send STREAM /             |    
    | STREAM_DATA_BLOCKED       |    
    |                           |    
    | Peer Creates              |    
    | Bidirectional Stream      |    
    v                           |    
+-------+                       |    
| Send  | Send RESET_STREAM     |    
|       |---------------------->|    
+-------+                       |    
    |                           |    
    | Send STREAM + FIN         |    
    v                           v    
+-------+                   +-------+
| Data  | Send RESET_STREAM | Reset |
| Sent  |------------------>| Sent  |
+-------+                   +-------+
    |                           |
    | Recv All ACKs             | Recv ACK
    v                           v
+-------+                   +-------+
| Data  |                   | Reset |
| Recvd |                   | Recvd |
+-------+                   +-------+
```
在发送端Endpoint启动(客户端类型为0和2，服务端类型为1和3)由应用层打开的Stream，"Ready"状态标识新创建的Stream已经可以接收应用层的数据，这时将数据进行缓存以等待发送。   
发送第一个**STREAM**或**STREAM_DATA_BLOCKED**帧时Stream会进入"Send"状态。 实现上可以将设置Stream ID延后知道Stream发送了第一个STREAM帧进入了"Send"状态，这样可以更好的确认Stream的优先级。   
由对端方发起的双向Stream接收状态机创建后，发送状态机以"Ready"状态开始。   
在"Send"状态，EndPoint通过**STREAM**帧发送或重传数据，EndPoint发送过程遵守由对端创建的流量限制，并一直接收和处理对端的**MAX_STREAM_DATA**帧。如果被连接或Stream的流控限制(见4.1节)导致阻塞数据发送，"Send"状态的EndPoint将发送**STREAM_DATA_BLOCKED**帧。  
当应用层的数据被全部发送完并且发送了**FIN**的**STREAM**帧，Stream的发送端进入"Data Sent"状态，之后EndPoint只会做一些必要的数据重传不再接收新的应用层数据。在这个状态，EndPoint不用检查流控限制或者发送**STREAM_DATA_BLOCKED**帧，但是还是可能会接收到**MAX_STREAM_DATA**帧，直到对端接收到最终的Stream数据，不过此时可以安全的忽略对端发送的**MAX_STREAM_DATA**帧。   
一旦所有的数据都被确认，Stream发送端进入"Data Recvd"状态，这是一个终端状体。   
在"Ready","Send"或"Data Sent"状态，应用层都可以终止Stream的数据发送。或者，EndPoint可能接收到对端的**STOP_SENDING**帧。无论哪种情况，EndPoint都会发送一个**RESET_STREAM**帧，之后进入"Reset Sent"状态。   
EndPoint可能会在Stream上第一次就发送**RESET_STREAM**帧，这将导致Stream打开发送部分并直接进入"Reset Sent"状态。   
一旦**RESET_STREAM**帧被确认，Stream的发送端会进入"Reset Recvd"的终端状态。
### 3.2 Stream的接收状态
下图展示了从对端接收数据的部分状态：
```
    o
    | Recv STREAM / STREAM_DATA_BLOCKED / RESET_STREAM
    | Create Bidirectional Stream (Sending)
    | Recv MAX_STREAM_DATA / STOP_SENDING (Bidirectional)
    | Create Higher-Numbered Stream
    v
+-------+
|  Recv | Recv RESET_STREAM
|       |-----------------------.
+-------+                       |
    |                           |  
    | Recv STREAM + FIN         |
    v                           |
+-------+                       |
| Size  |   Recv RESET_STREAM   |
| Known |---------------------->|
+-------+                       |
    |                           |
    |Recv All Data              |
    v                           v
+-------+ Recv RESET_STREAM +-------+
| Data  |--- (optional) --->| Reset |
| Recvd |   Recv All Data   | Recvd |
+-------+<-- (optional) ----+-------+
    |                           |
    | App Read All Data         | App Read RST
    v                           v
+-------+                   +-------+
| Data  |                   | Reset |
| Read  |                   | Read  |
+-------+                   +-------+
```
Stream接收部分不跟踪发送部分无法观测的状态，如"Ready"状态。Stream的接收部分跟踪向应用程序传递的数据。   
当接收到对端发送的第一个**STREAM**，**STREAM_DATA_BLOCKED**或**RESET_STREAM**帧(客户端类型为0和2，服务端类型为1和3)时，创建Stream的接收部分。当接收到对端发送的**MAX_STREAM_DATA**，**STOP_SENDING**帧时也会创建Stream的发送部分。Stream发送部分的起始状态是"Recv"。    
当EndPoint创建双向发送Stream的发送部分进入"Ready"时，Stream的接收部分进入"Recv"状态。
当接收到对端发送的**MAX_STREAM_DATA**，**STOP_SENDING**EndPoint打开一个双向Stream。一个未打开的Stream接收到**MAX_STREAM_DATA**表示对端已经打开了Stream并且提供了流控设置。一个未打开的Stream接收到**STOP_SENDING**表示对端希望从这个Stream接收数据。当发送丢包或重传时，这两种包都可能比**STREAM**和**STREAM_DATA_BLOCKED**先到达。    
在Stream创建之前，相同类型的Stream较小Stream ID**一定**被创建过。这样确保Stream的创建顺序在两个EndPoint上是一致的。   
在"Recv"状态，EndPoint接收**STREAM**和**STREAM_DATA_BLOCKED**帧，缓存接收的数据，组合排序然后传输给应用。当数据交付给应用时，可以复用缓存。EndPoint发送**MAX_STREAM_DATA**帧通知对端发送更多的数据。   
当接收到设置了**FIN**的**STREAM**帧时，就获知了Stream的最终大小。Stream的接收部分进入"Size Known"状态。在此状态，EndPoint不再发送**MAX_STREAM_DATA**给对端，只是接收数据。    
一旦Stream接收到了所有的数据，接收部分就进入了"Data Recvd"状态，这可能与接收到**FIN**的**STREAM**进入"Size Known"是同一帧。这之后，任何**STREAM**帧和**STREAM_DATA_BLOCKED**帧都可以被丢弃。    
"Data Recvd"状态维持到应用接收完所有的数据，当所有数据都交付完成时，Stream进入"Data Read"状态，这是一个终端状态。    
在"Recv"或"Size Know"状态接收到**RESET_STREAM**帧，Stream会进入"Reset Recvd"状态，这可能会导致中断向应用交付数据的过程。    
可能在所有数据都接收完成时收到**RESET_STREAM**帧(在"Data Recvd"状态)，同样，也可能在接收到**RESET_STREAM**帧之后又接收到剩余的数据携带帧(在"Reset Recvd"状态)，这时实现可以自己选择处理方式。    
发送**RESET_STREAM**表示EndPoint不再发送数据，然后并没有要求接收到**RESET_STREAM**后中断数据接收。一种实现可能会切断Stream的数据接收过程然后丢弃掉所有已经接收缓存的数据，然后发送**RESET_STREAM**信号(通知上层)。当所有的数据都已经接收并缓存以等待应用读取时，**RESET_STREAM**也可能被忽略，这时Stream的接收部分依然处于"Data Recvd"状态。    
一旦应用接收到Stream重置的信号，Stream的接收部分进入"Reset Read"的终端状态。    
### 3.3 Stream的帧类型
Stream的发送方只会发送三种类型的帧，他们影响着发送端和接收端Stream的状态：   
+ STREAM(见19.8节)
+ STREAM_DATA_BLOCKED(见19.13节)
+ RESET_STREAM(见19.4节)

Stream在终端状态时(“Data Recvd”或“Reset Recvd”)，**禁止**发送任何携带上述类型的帧。一个发送者在发送了**RESET_STREAM**之后，**禁止**再发送**STREAM**或**STREAM_DATA_BLOCKED**帧。因为，在终端状态或者“Reset Sent”状态，由于可能出现的延迟到达，接收端在任何状态都会接收这三种类型的帧。   

Stream的接收端发送:    
+ MAX_STREAM_DATA
+ STOP_SENDING   

Stream的接收端在”Recv”状态时只发送**MAX_STREAM_DATA**帧，接收端在未收到**RESET_STREAM**时，在任何状态都可以发送**STOP_SENDING**帧，也就是除了”Reset Recvd”,”Reset Read”状态。然而，在”Data Recvd”状态发送**STOP_SENDING**并没有什么意义，因为所有的数据都已经被接收。因为包的延迟到达，Stream的发送端在任何时候都有可能收到这两种帧。
### 3.4 双向Stream状态
一个双向的Stream由发送和接收两部分组成，实现上可以将发送和接送部分的状态组合为双向Stream的状态。最简单的模型，当发送和接收部分都不处于终端状态时，Stream处于”Open”状态，当发送和接收都处于终端状态时，Stream处于”Closed”状态。    
下图展示了一种更为复杂的组合Stream状态，其近似于HTTP/2。由Stream的发送和接收部分的多个状态映射为一个复合状态。注意，这只是这种映射的一种可能；这种映射要求在转换到“closed”或“half closed”状态之前确认所有数据。
```
+----------------------+----------------------+-----------------+
| 发送部分状态           |  接收部分状态          |  复合状态        |
+======================+======================+=================+
| No Stream/Ready      | No Stream/Recv *1    | idle            |
+----------------------+----------------------+-----------------+
| Ready/Send/Data Sent | Recv/Size Known      | open            |
+----------------------+----------------------+-----------------+
| Ready/Send/Data Sent | Data Recvd/Data Read | half-closed     |
|                      |                      | (remote)        |
+----------------------+----------------------+-----------------+
| Ready/Send/Data Sent | Reset Recvd/Reset    | half-closed     |
|                      | Read                 | (remote)        |
+----------------------+----------------------+-----------------+
| Data Recvd           | Recv/Size Known      | half-closed     |
|                      |                      | (local)         |
+----------------------+----------------------+-----------------+
| Reset Sent/Reset     | Recv/Size Known      | half-closed     |
| Recvd                |                      | (local)         |
+----------------------+----------------------+-----------------+
| Reset Sent/Reset     | Data Recvd/Data Read | closed          |
| Recvd                |                      |                 |
+----------------------+----------------------+-----------------+
| Reset Sent/Reset     | Reset Recvd/Reset    | closed          |
| Recvd                | Read                 |                 |
+----------------------+----------------------+-----------------+
| Data Recvd           | Data Recvd/Data Read | closed          |
+----------------------+----------------------+-----------------+
| Data Recvd           | Reset Recvd/Reset    | closed          |
|                      | Read                 |                 |
+----------------------+----------------------+-----------------+
```
注(*1):    
如果Stream没有创建，或者Stream的接收状态处于"Recv"而没有接收到任何帧，则视为"idle"状态。
### 3.5 请求的状态转换
如果一个应用不再对Stream接收的数据感兴趣，可以终止Stream的读取然后设置一个应用的error code。如果Stream处于"Recv"或"Size Known"状态，传输应该通过发送**STOP_SENDING**帧来提示对端关闭Stream的发送部分。这表示接收端的应用不再读取之后接收到的数据，但并不代表会忽略掉已经接收到的数据。    
在发送**STOP_SENDING**后接收到的**STREAM**帧依然会计入连接和流量控制中，即使这些帧可能会被丢弃。    
一个**STOP_SENDING**帧要求接收端返回一个**RESET_STREAM**帧，如果Stream处于"Ready"或"Send"状态，在接收到**STOP_SENDING**帧后**一定**要发送一个**RESET_STREAM**帧做为反馈。如果Stream处于"Data Sent"状态并且未传输的数据被申明为丢弃，EndPoint**应该**发送一个**RESET_STREAM**帧来代替重传。    
如果Stream在接收到**STOP_SENDING**的时候已经处于"Data Sent"状态，EndPoint如果想要终止先前发送的**STREAM**帧的重传，则要先发送一个**RESET_STREAM**帧。    
**STOP_SENDING** **应该**只能被没有被对端重置的Stream发送，**STOP_SENDING**对处于"Recv"或"Size Known"状态的Stream是非常有效的。    
如果前一个**STOP_SENDING**丢失，则EndPoint需要重传另一个**STOP_SENDING**，然而，一旦Stream接收到了所有数据或接收到了**RESET_STREAM**，也就是说，Stream已经不处于"Recv"或"Size Known"状态，就没有必要再发送**STOP_SENDING**帧了。    
如果EndPoint希望终止双向Stream两个方向上的数据传输，可以发送一个**RESET_STREAM**来终止通知对端自己不再发送，然后再发送一个**STOP_SENDING**来通知对端不要再向自己发送。    

## 4 流量控制
非常有必要来限制数据的数量以使接收端可以有效缓存，一是防止一个快速的发送端压垮一个较慢的接收端，二是防止恶意的发送端发送大量数据以占用接收端过量的内存。为了使接收端能够控制连接使用的内存以压缩发送端的发送数据量，Stram的流控既有单独的控制也有聚合的控制。一个QUIC的接收端在任何时候都控制着发送端最大数据量的发送限制。详细描述见4.1节和4.2节。    
同样，为了控制一个连接上的并发，QUIC的EndPoint控制着对端可以启动的最大Stream数量限制。详细描述见4.5节。    
**CRYPTO**帧数据的发送与Stream的流控方式不同，QUIC依赖加密协议的实现来避免数据的过度缓存(见[QUIC-TLS](https://tools.ietf.org/html/draft-ietf-quic-tls-27))。实现上QUIC应该提供一种接口来告知其缓存上限，这样就不会在多层上存在冗余缓存。    

### 4.1 数据流控制
QUIC采用了一种类似于HTTP/2的基于信用的流控制方案，接收端会公布它在Stream和对应连接上准备好接收的字节数，这就导致QUIC有两层的流量控制：    
+ Stream层的流控，它通过限制在任意Stream上发送的数据量来避免一个单个的Stream消耗连接上过多的缓存。
+ 连接层的流控，通过限制所有Streams的**STREAM**帧的数据量来避免发送端发送的数据超出连接的整个缓存大小。    

接收端在握手(见7.3节)的时候通过发送传输参数来设置所有Stream的初始流量控制信息，通过发送**MAX_STREAM_DATA**帧(见19.10节)或**MAX_DATA**帧(见19.9节)到对端去调整流量限制。    
Stream的接收端通过发送设置了适当Stream ID的**MAX_STREAM_DATA**帧来通知对端流量控制信息，一个**MAX_STREAM_DATA**帧携带了Stream的所允许的绝对最大数据偏移，接收端可以使用当前所消耗的数据偏移来确定流量控制的偏移量。接收端**可能**会在多个包上发送**MAX_STREAM_DATA**帧来确保接收端在超出流控限制之前接收到了流控通知，即使其中的一些包丢失。    
接收端通过发送**MAX_DATA**帧来调整连接的流量控制信息，它携带所有Stream绝对偏移量之和。接收端维护累计所有Stream已经接收数据量的和，以用来检测是否超出了流量限制。接收端可以使用所有Stream上消耗的字节总数来确定要公布的流量限制。    
接收端通过发送**MAX_STREAM_DATA**帧和**MAX_DATA**帧来通告一个更大的偏移值。如果发布了没有增加偏移的偏移值，也没有影响。发送端**必须**忽略偏移量未增加的**MAX_STREAM_DATA**帧和**MAX_DATA**帧。
如果发送端违反了连接层或Stream层的流量限制，接收端**一定**会使用**FLOW_CONTROL_ERROR**错误来终止连接。    
如果发送端消耗完了对端通知的流量限制偏移，那么就无法再发送数据而进入阻塞状态，这时**应该**发送**STREAM_DATA_BLOCKED**帧或**DATA_BLOCKED**帧来通知对端还有数据未发送但是被流量限制阻塞了。如果阻塞的时间超过了设置的空闲超时时间(见10.2节)，那么连接就可能会关闭，即使还有数据没有发送完。为了防止连接关闭，受流量限制的发送端**应该**在没有未确认的在途数据的时候定期发送**STREAM_DATA_BLOCKED**或**DATA_BLOCKED**帧。    

### 4.2 流量限制增加
实现要决定何时通过**MAX_STREAM_DATA**和**MAX_DATA**发送多少偏移限制，本节提供一些考虑因素。    
为了避免阻塞发送端，接收端可以在一个交互回合中多次发送或尽早发送**MAX_STREAM_DATA**和**MAX_DATA**以便从丢包中快速恢复。    
控制帧会增加连接的开销，因此，极小的变动就频繁的发送**MAX_STREAM_DATA**和**MAX_DATA**帧是不明智的。另一方面，如果更新的频次较低，需要发送更大的限制偏移来避免发送端阻塞。在确定公布的流量限制偏移有多大时，应该在资源开销和传输开销之间有一个权衡。    
接收端可以基于包的往返时间和接收端消耗数据的速率来自动调整发送的偏移限制和频率，类似与常见的TCP实现。作为一种优化，EndPoint可以在有其他包要发送或者对端已经被流量限制阻塞时发送相关的**MAX_STREAM_DATA**和**MAX_DATA**帧，以确保流控不会导致发送额外的控制包。    
被阻塞的发送端并不强制要求发送**STREAM_DATA_BLOCKED**和**DATA_BLOCKED**帧，因此，接收端在发送**MAX_STREAM_DATA**和**MAX_DATA**帧之前**禁止**等待**MAX_STREAM_DATA**和**MAX_DATA**帧。因为这样做的话可能会阻塞连接上的其他发送端，另外，就算发送端发送了这些帧，等待这些帧也会导致发送端最少阻塞一个RTT周期。    
如果一个阻塞了的发送端接收到了新的偏移通告，则可能在响应中发送大量的数据，导致短期的拥塞。在6.9节有如何避免这种拥塞的相关讨论。

### 4.3 处理Stream的取消
EndPoints应该在Stream取消时就流量限制的偏移量达成一致，以避免超出流量限制和死锁。    
在接收**RESET_STREAM**的过程中，EndPoint会终止对应Stream的状态并忽略Stream上新达到的数据。如果**RESET_STREAM**帧不携带偏移量，两个EndPoint可能会在计算连接流量控制的偏移上产生不一致。为了解决这个问题，**RESET_STREAM**帧包含了Stream最终发送数据的数量。在接收**RESET_STREAM**的过程中，接受端可以确切的知道在**RESET_STREAM**之前已经接收了多少数据，发送端**必须**使用Stream的最终偏移大小来计算整个连接层流量控制的偏移大小。    
**RESET_STREAM**帧会突然终止Stream的一个方向，对于一个双向的Stream来说，**RESET_STREAM**对另一个方向的流量控制没有影响。两端都必须保持在未终止方向上的流量控制，直到该方进入终端状态或者其中一个EndPoint发送了**CONNECTION_CLOSE**。

### 4.4 Stream最终大小
最终大小是Stream消耗流量控制偏移的数量。如果每个相邻的byte都被只发送了一次，那么最终大小就是发送byte的数量。通常，最终大小比Stream上已经发送的最大偏移高一个。如果还没有数据发送，则是0。    
对于一个已经被重置的Stream，最终大小在**RESET_STREAM**中携带，或者，最终大小是偏移量加上用FIN标志标记的Stream帧的长度。如果是单向Stream的接收端，则为0。

当接受端的Stream在进入”Size Known”或”Reset Recvd”状态时，会知晓最终的大小。EndPoint**禁止**发送超过最终大小的数据。    
一旦Stream的最终大小被确定，就不能再修改。如果接收到了**RESET_STREAM**或**STREAM**帧导致最终大小发生变化，则EndPoint**应该**发送一个**FINAL_SIZE_ERROR**错误作为响应(见11节)。接收端**应该**将超过最终大小的数据视为**FINAL_SIZE_ERROR**错误，即使在Stream关闭之后。生成这些错误并不是强制性的，而是因为EndPoint生成这些错误也意味着EndPoint需要为关闭Stream保持最终大小状态，这是重要的状态承诺。

### 4.5 控制并发
Endpoint限制着对端打开Stream的并发数据。只有Stream ID比(max_stream * 4 + initial_stream_id_for_type)小的Stream可以被打开。初始的限制在在握手时通过传输参数设置，之后的限制通过**MAX_STREAMS**帧来调整。单独的限制适用于单向流和双向流。    
如果通过握手时的传输参数或者**MAX_STREAMS**帧接收到的最大Stream数量超过了2^60，这将导致最大的Stream ID不能按照变长整数编码(见16节)。一旦接受到这样的数字，则连接**一定**要立马关闭连接并返回**STREAM_LIMIT_ERROR**错误(见10.3节)。   
EndPoint**禁止**超过他们对端设置的限制。一旦收到帧携带的Stream ID超过了限制，则**一定**要视为连接错误并返回**STREAM_LIMIT_ERROR**。    
一旦接受端通过**MAX_STREAMS**接收到了一个Stream数量限制，则再接收到小于这个的限制会被忽略。接收端**一定**要忽略任何不会将限制增大的**MAX_STREAMS**帧。    
与Stream和连接的流量控制一样，实现中要自己定义什么时候通过**MAX_STREAMS**帧限制多少Stream并发到对端。实现可能会选择在Stream ID接近限制时增加限制，以保持对等方可用的流的数量大致一致。    
如果EndPoint由于对端的限制不能再打开新的Stream，则**应该**发送**STREAMS_BLOCKED**帧(见9.14节)，这对程序调试非常有用。EndPoint**一定不要**等待**STREAMS_BLOCKED**帧之后再调整对端Stream数量限制，如果这样做的话意味着对端至少要阻塞一个RTT周期，如果对端选择不发送**STREAMS_BLOCKED**帧，则可能会阻塞更长时间。    

## 5 连接
QUIC的连接建立过程结合了版本协商和加密传输握手以减少连接建立时的延迟，这将在第七节讲述。一旦连接建立，则可以迁移到任何一个不同ip和端口的EndPoint上，这将在第九节讲述。最后，第十节将说明连接的中断过程。

### 5.1 连接ID
每个连接都有一组连接标识或者连接ID，以用来标识一个连接。连接ID由EndPoint独立选择，每个EndPoint选择对端使用的连接ID。连接ID的主要作用是确保下层协议(UDP,IP)协议的地址变更不会导致QUIC连接的数据传输到错误的EndPoint上。每个EndPoint都使用一个特定于实现(或特定于部署的)方式来选择连接ID。    
相同的连接ID不能在同一连接上多次发出。    
具有长数据头的包包含源连接ID和目的连接ID字段，这些字段被用来为新的连接设置连接ID，具体细节在7.2节介绍。    
具有短数据头的包(17.3节)只包括目的地连接ID，而忽略显式长度。目的连接ID字段的长度应该被对端所知道。EndPoint使用一种基于路由的负载均衡器来就连接ID的长度或编码方案达成一致，固定部分可以编码一个显式的长度，这允许整个连接ID的长度不同，并且仍然由负载平衡器使用。
一个版本协商包(17.2.1节)原路返回客户端选择的连接ID，以保证包可以正常路由到客户端，且客户端可识别该包。    
当不需要连接ID路由到正确的端点时，可以使用长度为零的连接ID。然而，相同IP和端口上的多路复用连接使用零长度的连接ID时会导致连接迁移失效。NAT的重新绑定和客户端的端口重用都会导致此问题产生，因此，除非确认这些协议的功能没有启动，否则不要使用长度为零的连接ID。     
当一个EndedPoint使用一个非零长度的连接ID时，它需要确认对端有一个可以选择的连接ID来发送数据。这些连接ID由EndPoint使用**NEW_CONNECTION_ID**帧来提供。    

### 5.1.1 发布连接ID
每个连接ID都有一个关联的序列号用来消除重复消息。在握手阶段，EndPoint发出的初始连接ID在长数据包头(见17.2节)的Source Connection ID字段中，初始连接ID的序列号为0，如果发送了preferred_address参数，则提供的连接ID为1。    
之后附加的连接ID通过**NEW_CONNECTION_ID**帧发送给对端。每个新发布的连接ID**必须**加1。客户端在初始包中随机选择的连接ID和重试包提供的任何连接ID都不会被分配序列号，除非服务器选择将它们保留为初始连接ID。   
当一个EndPoint发出了一个连接ID，则在连接期间它**必须** **接收**携带这个连接ID的包直到对端通过**RETIRE_CONNECTION_ID**帧宣布这个ID失效。已经发出而且没有失效的连接ID时活跃的，任何活动的连接ID在任何时候都是有效的，可以在任何包类型中使用，这包括服务器通过preferred_address参数发出的连接ID。    
EndPoint应该确保它的对端有足够数量可用和未使用的连接ID，EndPoint存储接收到的连接ID以供将来使用，并在活跃的连接上通过active_connection_id_limit参数来宣布愿意存储的活跃ID数量。EndPoint**一定不要**提供过多的连接ID超过对端的限制，当EndPoint接收到超过通过active_connection_id_limit参数宣布的ID限制的连接ID时，**必须**用**CONNECTION_ID_LIMIT_ERROR**错误关闭这个连接。    
当对端退出一个连接ID时，EndPoint必须提供一个新的连接ID，如果一个EndPoint提供的连接ID少于对端的活动连接ID限制，那么当它接收到一个带有以前未使用的连接ID的数据包时，它**可能**会提供一个新的连接ID。EndPoint可以限制为每个连接发出的连接ID的频率或总数，以避免连接ID用完的风险(请参阅第10.4.2节)。    
启动连接迁移并需要non-zero-length连接ID的EndPoint应确保其对端可用的连接ID池允许在迁移时使用新的连接ID，因为如果池耗尽，对端将关闭连接。   

### 使用和注销连接ID
EndPoint在连接存在的任何时间都可以修改它使用的连接ID，EndPoint使用连接ID响应对端的迁移。(9.5节)    