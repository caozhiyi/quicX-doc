# QUIC丢失检测与拥塞控制

## 摘要
本文描述了QUIC的丢失检测和拥塞控制机制。   

## 读者须知 
本草案的讨论在QUIC工作组邮件列表中进行(quic@ietf.org)，存档于https://mailarchive.ietf.org/arch/search/？email_list=quic
(https://mailarchive.ietf.org/arch/search/？email_list=quic）。
工作组信息可在https://github.com/quicwg(https://github.com/quicwg)；此草稿的源代码和问题列表可在https://github.com/quicwg/base-drafts/labels//recovery(https://github.com/quicwg/base-drafts/labels//recovery)。

## 版权
版权所有（c）2020 IETF Trust和文件作者。

## 1 介绍
QUIC是一种新的基于UDP的多路复用和安全传输。QUIC建立在几十年的运输和安全经验的基础上，并实施使其作为现代通用通讯协议的机制。   
QUIC实现了现有TCP拥塞控制和丢失恢复机制的机制，在RFCs、各种Internet草稿以及Linux-TCP实现中流行的那些机制。本文档描述了QUIC拥塞控制和丢失恢复，并在适用的情况下，描述了RFC、Internet草稿、学术论文和/或TCP实现中的TCP等效属性。   

## 2 符号约定
关键字 "**一定**", "**禁止**", "**要求**", "**应该**", "**不应**", "**推荐**", "**不推荐**", "**可以**", "**可选**"的使用与[BCP14](https://tools.ietf.org/html/bcp14) [RFC2119](https://tools.ietf.org/html/rfc2119) [RFC8174](https://tools.ietf.org/html/rfc8174)相同。    
本文件中使用的术语定义：
+ ACK引发帧: 除了**ACK**、**PADDING**和**CONNECTION_CLOSE**之外的所有帧都被认为是ACK引发帧。 
+ ACK引发包: 包含ACK引发帧的包在最大ACK延迟内从接收方获取ACK，称为ACK引发包。
+ In-flight: 当ACK引发包或者包含了**PADDING**帧的包发送了但是没有被确认，也没有被确认丢失或者与旧密钥一起被丢弃，则认为是In-flight的。

## 3 QUIC传输机制的设计
QUIC中的所有传输都使用包级别包头发送，该包头指示加密级别并包括包序列号（以下称为包号）。加密级别表示数据包编号空间，如[QUIC-TRANSPORT]中所述。在连接的生命周期内，数据包编号在数据包编号空间内从不重复。包号在一个空间内以单调递增的顺序发送，防止了产生歧义。   
这种设计避免了在传输和重传之间消除歧义的需要，并消除了QUIC解释TCP丢失检测机制的显著复杂性。   
QUIC包可以包含多个不同类型的帧。恢复机制确保需要可靠传输的数据和帧被确认或声明丢失，并在必要时以新的数据包发送。数据包中包含的帧类型影响恢复和拥塞控制逻辑：
+ 所有的包都要被确认，尽管那些不包含ACK引发帧的包只与包含ACK引发帧的包一起被确认。
+ 包含**CRYPTO**帧的长包头数据包对于QUIC握手的性能至关重要，并且使用较短的计时器进行确认。
+ 包含除了**ACK**或**CONNECTION_CLOSE**帧之外的帧的数据包计入拥塞控制限制，并被视为在传输中。
+ **PADDING**帧会导致数据包在传输过程中累加字节，而不会直接导致发送确认。

## 3.1 QUIC与TCP的相关区别
熟悉TCP的丢失检测和拥塞控制的读者会在这里找到与著名的TCP算法类似的算法。然而，QUIC和TCP之间的协议差异导致了算法上的差异。下面我们简要介绍这些协议的差异。  

### 3.1.1 单独的数据包编号空间
QUIC对每个加密级别使用单独的包编号空间，除了0-RTT和所有的1-RTT密钥使用相同的包编号空间。单独的数据包编号空间确保使用一个加密级别发送的数据包的确认不会导致以不同加密级别发送的数据包的虚假重新传输。拥塞控制和往返时间（RTT）测量在包数空间是统一的。   

### 3.1.2 单调递增包数
TCP将发送方的传输指令与接收方的发送指令相混淆，导致相同序列号的数据重发，从而导致“重传模糊”。QUIC把两者分开。QUIC使用数据包编号来指示传输顺序。在一个或多个流中发送应用数据，并且传送顺序由在流帧内编码的流偏移量确定。   
QUIC的包数在一个包数空间内严格递增，并直接根据传输顺序进行编码。较高的数据包编号表示该数据包的发送时间较晚，而较低的数据包编号表示该数据包更早地发送。当检测到包含ACK引发帧的包丢失时，QUIC用新的包号重新处理新包中的必要帧，从而消除在接收ACK时确认哪个包的模糊性。因此，可以进行更精确的RTT测量，可以很容易地检测到虚假的重传，并且诸如快速重传之类的机制可以普遍地应用，而仅仅基于包号。   
这个设计点大大简化了QUIC的丢失检测机制。大多数TCP机制都隐式地尝试根据TCP序列号推断传输顺序—这是一项非常重要的任务，尤其是当TCP时间戳不可用时。   

### 3.1.3 精确的耗时统计
当一个包丢失时，QUIC开始一个丢失的周期，在这个周期开始之后发送的包被确认时结束该计时周期。TCP等待序列号空间中的空隙被填充，因此，如果一个段连续多次丢失，丢失的耗时统计可能不会在多次往返中结束。因为每一个计时周期都应该只减少一次拥塞窗口限制，所以QUIC将对每一次经历丢失的往返执行一次，而TCP可能只在多个往返中执行一次。

### 3.1.4 没有接收拒绝
QUIC-ACK包含与TCP-SACK相似的信息，但QUIC不允许任何已确认的数据包被拒绝，这大大简化了双方的实现，并减少了发送方的内存压力。意思是QUIC确认的靠后一些的乱序包，发送方不用重新发送。   

### 3.1.5 更多的确认范围
QUIC支持许多ACK范围，而不是TCP的3个SACK范围。在高丢包率的环境中，这加快了恢复速度，减少了虚假的重传，并确保了无需依赖超时的快速重传过程。

### 3.1.6 延迟确认的显式修正
QUIC端点测量在接收数据包和发送相应确认之间产生的延迟，从而可以使对端计算更准确的往返时间估计（见[QUIC-TRANSPORT]第13.2节）。

## 4 RTT评估
在高层次上，端点要测量数据包从发送到接收到对端ACK确认之间的时间间隔(RTT time)。端点使用RTT样本和对端报告的主机延迟（见[QUIC-TRANSPORT]的第13.2节）来生成网络路径RTT的统计测量值。端点为每个路径计算以下三个值：在路径生命周期内观察到的最小值（min_rtt）、指数加权移动平均值（平滑的移动平均值）和观察到的rtt样本（rttvar）中的平均偏差（在本文的其余部分称为“变更值”）。

### 4.1 生成RTT样本
端点在接收到满足以下两个条件的ACK帧时生成RTT样本：
+ 最大的确认包号是新确认的
+ 新确认的包中至少有一个是ack引发包

RTT样本中latest RTT是自发送最大确认数据包以来经过的时间生成的：
```
latest_rtt = ack_time - send_time_of_largest_acked
```
RTT样本仅使用接收到的ACK帧中的最大确认包生成。这是因为对端只报告**ACK**帧中最大的确认包的ACK延迟。虽然所报告的ACK延迟不用于RTT样本测量，但它用于在平滑后的第4.3节和rttvar的后续计算中调整RTT样本。   
为了避免为单个包生成多个RTT样本，如果ACK帧没有新确认最大的已确认分组，则不应使用ACK帧来更新RTT估计值。   
在接收到没有新确认至少一个ACK引发包的**ACK**帧时，不能生成RTT样本。当只接收到非ACK引发包时，对端通常不会发送**ACK**帧。因此，只包含非ACK引发包的确认的ACK帧可以包括任意大的ACK延迟值。忽略这样的ACK帧可以避免后续smoothed_rtt和rttvar计算的复杂性。   
当在一个RTT中接收到多个ACK帧时，发送方可能会为每个RTT生成多个RTT样本。如[RFC6298]中所述，这样做可能会导致smoothed_rtt和rttvar的历史记录太片面。确保RTT估计保留足够的历史记录是一个开放的研究问题。   

### 4.2 评估最小RTT
"min_rtt"是给定网络路径所观察到的最小rtt。在第一个rtt样本上，min_rtt被设置为最新的latest_rtt，在随后的样本中设置为min_rtt和latest_rtt中较小的一个。在本文中，损失检测使用最小rtt来拒绝难以置信的小rtt样本。   
端点在计算"min_rtt"时只使用本地观察到的时间，并且不针对对等方报告的ACK延迟进行调整。这样做可以让端点完全根据观察到的情况为smoothed_rtt设置一个下限（见第4.3节），并限制由于对等方错误报告的延迟而导致的潜在低估。    
网络路径的RTT可能会随时间而改变。如果路径的实际RTT减小，则min_rtt将立即在第一个低样本上进行调整。如果路径的实际RTT增加，min_rtt将不再适用，允许在smoothed_rtt中包含小于比min_rtt大的样本。  

### 4.3 评估smoothed_rtt和rttvar
smoothed_rtt是端点的rtt样本的指数加权移动平均值，rttvar是rtt样本中的变量，使用平均变量进行估计。   
smoothed_rtt的计算使用在rtt样本调整对端主机延迟之后的路径延迟。如[QUIC-TRANSPORT]第19.3节所述，使用ACK帧的ACK延迟字段计算主机延迟。对于在ApplicationData包编号空间中发送的包，对等方将发送ack引发包的确认的任何延迟限制为不大于其在"max_ack_delay”传输参数中公布的值。因此，当对等端报告大于其最大"max_ack_delay”的Ack延迟时，延迟归因于对端无法控制的原因，例如对等端的调度器延迟或先前Ack帧的丢失。因此，任何超过对等方最大确认时延的延迟都被视为路径延迟的有效部分，并被纳入平滑的rtt估计中。   
当使用对等方报告的确认延迟调整RTT样本时，端点：
+ 对于在初始和握手数据包编号空间中发送的数据包，必须忽略Ack帧的Ack Delay字段。
+ 必须使用Ack帧的Ack Delay字段中报告的值与对等方的"max_Ack_Delay"传输参数中的较小值。
+ 如果产生的RTT样品小于min_rtt，则不得进行调整。这限制了误报对等方可能导致的对smoothed_rtt的低估。

在网络路径的第一个RTT示例中，smoothed_rtt设置为latest_rtt。   
smoothed_rtt和rttvar计算如下，类似于[RFC6298]。在网络路径的第一个RTT示例中：   
```
smoothed_rtt = latest_rtt
rttvar = latest_rtt / 2
```
在随后的RTT样本中，smoothed_rtt和rttvar演变如下：   
```
ack_delay = min(Ack Delay in ACK Frame, max_ack_delay)
adjusted_rtt = latest_rtt
if (min_rtt + ack_delay < latest_rtt):
adjusted_rtt = latest_rtt - ack_delay
smoothed_rtt = 7/8 * smoothed_rtt + 1/8 * adjusted_rtt
rttvar_sample = abs(smoothed_rtt - adjusted_rtt)
rttvar = 3/4 * rttvar + 1/4 * rttvar_sample
```

## 5 损失检测
QUIC发送者使用确认来检测丢失的数据包，并使用探测超时（见第5.2节）来确保接收到确认。本节介绍这些算法。   
如果数据包丢失，QUIC传输需要从丢失中恢复，例如通过重新传输数据、发送更新的帧或放弃该帧。有关更多信息，请参见[QUIC-TRANSPORT]第13.3节。  

### 5.1 基于确认的检测