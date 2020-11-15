# QUIC丢失检测与拥塞控制

## 摘要
本文描述了QUIC的丢失检测和拥塞控制机制。   

## 读者须知 
本草案的讨论在QUIC工作组邮件列表中进行(quic@ietf.org)，存档于https://mailarchive.ietf.org/arch/search/？email_list=quic
(https://mailarchive.ietf.org/arch/search/？email_list=quic）。
工作组信息可在https://github.com/quicwg(https://github.com/quicwg)；此草稿的源代码和问题列表可在https://github.com/quicwg/base-drafts/labels//recovery(https://github.com/quicwg/base-drafts/labels//recovery)。

## 版权
版权所有（c）2020 IETF Trust和文件作者。

## 目录
<!-- TOC -->

- [QUIC丢失检测与拥塞控制](#quic丢失检测与拥塞控制)
  - [摘要](#摘要)
  - [读者须知](#读者须知)
  - [版权](#版权)
  - [目录](#目录)
  - [1 介绍](#1-介绍)
  - [2 符号约定](#2-符号约定)
  - [3 QUIC传输机制的设计](#3-quic传输机制的设计)
  - [3.1 QUIC与TCP的相关区别](#31-quic与tcp的相关区别)
    - [3.1.1 单独的数据包编号空间](#311-单独的数据包编号空间)
    - [3.1.2 单调递增包数](#312-单调递增包数)
    - [3.1.3 精确的耗时统计](#313-精确的耗时统计)
    - [3.1.4 没有接收拒绝](#314-没有接收拒绝)
    - [3.1.5 更多的确认范围](#315-更多的确认范围)
    - [3.1.6 延迟确认的显式修正](#316-延迟确认的显式修正)
  - [4 RTT评估](#4-rtt评估)
    - [4.1 生成RTT样本](#41-生成rtt样本)
    - [4.2 评估最小RTT](#42-评估最小rtt)
    - [4.3 评估smoothed_rtt和rttvar](#43-评估smoothed_rtt和rttvar)
  - [5 损失检测](#5-损失检测)
    - [5.1 基于确认的检测](#51-基于确认的检测)
      - [5.1.1 数据包阈值](#511-数据包阈值)
      - [5.1.2 时间阈值](#512-时间阈值)
    - [5.2 探测超时](#52-探测超时)
      - [5.2.1 计算PTO](#521-计算pto)
    - [5.3 握手和新的传输路径](#53-握手和新的传输路径)
      - [5.3.1 发送探测包](#531-发送探测包)
      - [5.3.2 损失检测](#532-损失检测)
    - [5.4 处理重试数据包](#54-处理重试数据包)
    - [5.5 丢弃密钥和包状态](#55-丢弃密钥和包状态)
  - [6 拥塞控制](#6-拥塞控制)
    - [6.1 显式拥塞通知](#61-显式拥塞通知)
    - [6.2 慢启动](#62-慢启动)
    - [6.3 拥塞避免](#63-拥塞避免)
    - [6.4 恢复期](#64-恢复期)
    - [6.5 忽略不可加密数据包的丢失](#65-忽略不可加密数据包的丢失)
    - [6.6 探测超时](#66-探测超时)
    - [6.7 持续拥塞](#67-持续拥塞)
    - [6.8 Pacing 发送速率](#68-pacing-发送速率)
    - [6.9 拥塞窗口利用不足](#69-拥塞窗口利用不足)
  - [7 安全考虑因素](#7-安全考虑因素)
    - [7.1 拥挤信号](#71-拥挤信号)
    - [7.2 交通分析](#72-交通分析)
    - [7.3 误报ECN标记](#73-误报ecn标记)
  - [8 IANA考虑](#8-iana考虑)
  - [9 引用](#9-引用)
    - [9.1 规范性引用文件](#91-规范性引用文件)
    - [9.2 资料性引用](#92-资料性引用)
  - [附录](#附录)

<!-- /TOC -->

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
QUICEndpoint测量在接收数据包和发送相应确认之间产生的延迟，从而可以使对端计算更准确的往返时间估计（见[QUIC-TRANSPORT]第13.2节）。

## 4 RTT评估
在高层次上，Endpoint要测量数据包从发送到接收到对端ACK确认之间的时间间隔(RTT time)。Endpoint使用RTT样本和对端报告的主机延迟（见[QUIC-TRANSPORT]的第13.2节）来生成网络路径RTT的统计测量值。Endpoint为每个路径计算以下三个值：在路径生命周期内观察到的最小值（min_rtt）、指数加权移动平均值（平滑的移动平均值）和观察到的rtt样本（rttvar）中的平均偏差（在本文的其余部分称为“变更值”）。

### 4.1 生成RTT样本
Endpoint在接收到满足以下两个条件的ACK帧时生成RTT样本：
+ 最大的确认包号是新确认的
+ 新确认的包中至少有一个是ack引发包

RTT样本中latest RTT是自发送最大确认数据包以来经过的时间生成的：
```
latest_rtt = ack_time - send_time_of_largest_acked
```
RTT样本仅使用接收到的ACK帧中的最大确认包生成。这是因为对端只报告**ACK**帧中最大的确认包的ACK延迟。虽然所报告的ACK延迟不用于RTT样本测量，但它用于在平滑后的第4.3节和rttvar的后续计算中调整RTT样本。   
为了避免为单个包生成多个RTT样本，如果ACK帧没有新确认最大的已确认包，则不应使用ACK帧来更新RTT估计值。   
在接收到没有新确认至少一个ACK引发包的**ACK**帧时，不能生成RTT样本。当只接收到非ACK引发包时，对端通常不会发送**ACK**帧。因此，只包含非ACK引发包的确认的ACK帧可以包括任意大的ACK延迟值。忽略这样的ACK帧可以避免后续smoothed_rtt和rttvar计算的复杂性。   
当在一个RTT中接收到多个ACK帧时，发送方可能会为每个RTT生成多个RTT样本。如[RFC6298]中所述，这样做可能会导致smoothed_rtt和rttvar的历史记录太片面。确保RTT估计保留足够的历史记录是一个开放的研究问题。   

### 4.2 评估最小RTT
"min_rtt"是给定网络路径所观察到的最小rtt。在第一个rtt样本上，min_rtt被设置为最新的latest_rtt，在随后的样本中设置为min_rtt和latest_rtt中较小的一个。在本文中，损失检测使用最小rtt来拒绝难以置信的小rtt样本。   
Endpoint在计算"min_rtt"时只使用本地观察到的时间，并且不针对对等方报告的ACK延迟进行调整。这样做可以让Endpoint完全根据观察到的情况为smoothed_rtt设置一个下限（见第4.3节），并限制由于对等方错误报告的延迟而导致的潜在低估。    
网络路径的RTT可能会随时间而改变。如果路径的实际RTT减小，则min_rtt将立即在第一个低样本上进行调整。如果路径的实际RTT增加，min_rtt将不再适用，允许在smoothed_rtt中包含小于比min_rtt大的样本。  

### 4.3 评估smoothed_rtt和rttvar
smoothed_rtt是Endpoint的rtt样本的指数加权移动平均值，rttvar是rtt样本中的变量，使用平均变量进行估计。   
smoothed_rtt的计算使用在rtt样本调整对端主机延迟之后的路径延迟。如[QUIC-TRANSPORT]第19.3节所述，使用ACK帧的ACK延迟字段计算主机延迟。对于在ApplicationData包编号空间中发送的包，对等方将发送ack引发包的确认的任何延迟限制为不大于其在"max_ack_delay”传输参数中公布的值。因此，当对等端报告大于其最大"max_ack_delay”的Ack延迟时，延迟归因于对端无法控制的原因，例如对等端的调度器延迟或先前Ack帧的丢失。因此，任何超过对等方最大确认时延的延迟都被视为路径延迟的有效部分，并被纳入平滑的rtt估计中。   
当使用对等方报告的确认延迟调整RTT样本时，Endpoint：
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
基于确认的丢失检测实现了TCP的快速重传[RFC5681]、早期重传[RFC5827]、FACK[FACK]、SACK丢失恢复[RFC6675]和RACK[RACK]。本节概述了这些算法是如何在QUIC中实现的。   
如果数据包满足以下所有条件，则确认该数据包丢失：
+ 数据包是未确认的，在飞行中，并且是在ACK确认的数据包之前发送的。
+ 它的数据包编号小于已确认的数据包（第5.1.1节），或者它在过去发送的时间足够长（第5.1.2节）。   

确认表示这之后发送的数据包已被发送，并且包和时间阈值为数据包重新排序提供了一定的容差。   
错误地确认数据包丢失会导致不必要的重传，并且可能由于拥塞控制器在检测到丢失时的动作而导致性能下降。检测虚假重传并以包或时间增加重新排序阈值的实现可以选择从较小的初始重新排序阈值开始，以最小化恢复延迟。

#### 5.1.1 数据包阈值
根据TCP丢失检测的最佳实践[RFC5681][RFC6675]，数据包重新排序阈值（kPacketThreshold）的建议初始值为3。QUIC实现时不应使用小于3的包阈值，以保持与TCP[RFC5681]一致。   
一些网络可能表现出更多的重新排序，导致发送方检测到虚假损失。实现者可以使用为TCP开发的算法，如TCP-NCR[RFC4653]，以提高QUIC的重新排序弹性。   

#### 5.1.2 时间阈值
一旦在同一包号空间内的之后数据包被确认，并且超过了过去发送阈值时间量，Endpoint应确认先前的数据包丢失。为了避免过早地声明数据包丢失，此时间阈值必须至少设置为Kgranulatity。时间阈值为：
```
max(kTimeThreshold * max(smoothed_rtt, latest_rtt), kGranularity)
```
如果在最大的已确认数据包之前发送的数据包还不能确认丢失，则应为剩余时间设置一个计时器。   
使用max（rttvar_sample，latest_rtt）可以防止以下两种情况：
+ 最新的RTT样本比smoothed_rtt样本低，可能是由于重新排序时确认遇到了一个较短的网络路径；
+ 最新的RTT样本高于smoothed_rtt，可能是由于实际RTT的持续增加，但smoothed_rtt还没有赶上。

建议的时间阈值（kTimeThreshold）以往返时间乘数表示，为9/8。    
实现可以尝试使用绝对阈值、来自先前连接的阈值、自适应阈值或包括RTT变化。较小的阈值会降低重新排序的弹性并增加虚假重传，而较大的阈值会增加丢失检测延迟。   

### 5.2 探测超时
当在预期的时间段内未确认ack引发包或握手尚未完成时，探测超时（PTO）触发会发送一个或两个探测数据报。PTO使连接能够从丢失中的尾包或确认中恢复。   
与丢失检测一样，探测超时是按数据包编号空间计算的。QUIC中使用的PTO算法实现了尾部损耗探针[RACK]、RTO[RFC5681]和TCP[RFC5682]的F-RTO算法的可靠性功能。超时计算基于TCP的重传超时时间[RFC6298]。   

#### 5.2.1 计算PTO
当发送ack引发包时，发送方为PTO周期调度定时器，如下所示：
```
PTO = smoothed_rtt + max(4*rttvar, kGranularity) + max_ack_delay
```
附录A.2和附录A.3中定义了kGranularity、smoothed_rtt、rttvar、rttvar和max_ack_delay。    
PTO周期是发送方等待已发送数据包确认的时间量。该时间段包括估计的网络往返时间（smoothed_rtt）、估计的变化（4*rttvar）和max_ack_delay，以说明接收器可能延迟发送确认的最大时间。当PTO为初始或握手数据包编号时，根据[QUIC-TRANSPORT]的13.2.1中的规定，max_ack_delay为0。PTO值必须至少设置为kGranulatity，以避免计时器立即过期。   
发送方在每次发送一个ack引发包时计算其PTO计时器。当ack引发包在多个包号空间中运行时，必须为最早超时的包号空间设置计时器，但ApplicationData除外，在握手完成之前必须忽略它；见[QUIC-TLS]第4.1.1节。不为ApplicationData启用PTO数据会优先完成握手，并阻止服务器在获得处理1-RTT数据包的密钥之前在PTO上发送1-RTT数据包。   
当PTO计时器过期时，PTO周期必须设置为其当前值的两倍。发送方速率的指数级降低很重要，因为连续的pto超时可能是由于严重拥塞导致的数据包或确认丢失造成的。即使在多个数据包数空间中存在ack引发包，探测超时也会在所有空间中呈指数级增加，以防止网络上出现过多负载。例如，初始包号空间中的超时使握手分包空间中的超时长度加倍。   
正在经历连续pto超时的连接的生命周期受到Endpoint空闲超时的限制。即PTO定时不可超过连接空闲超时时间。   
如果设置了时间阈值(第5.1.2节)丢失检测计时器，则不得设置探针计时器。预计时间阈值丢失检测计时器将比PTO早到期，并且不太可能错误地重新传输数据。   

### 5.3 握手和新的传输路径
新连接或新路径的初始探测超时应设置为初始RTT的两倍。通过同一网络恢复的连接应该使用前一个连接的最终平滑RTT值作为恢复连接的初始RTT。如果之前没有可用的RTT，初始RTT应设置为500ms，导致1秒的初始超时，如[RFC6298]中所建议的那样。   
一个连接可以使用发送一个**PATH_CHALLENGE**和接收一个**PATH_RESPONSE**之间的延迟来设置新路径的初始RTT（见附录A.2中的kInitialRtt），但该延迟不应被视为RTT示例。   
在服务器验证了路径上的客户端地址之前，它可以发送的数据量被限制为接收数据量的三倍，如[QUIC-TRANSPORT]第8.1节所述。如果无法发送数据，则在从客户端收到数据报之前，不得启用PTO警报。   
由于服务器可能会被阻塞，直到从客户端接收到更多的数据包，因此客户端有责任发送数据包以解除对服务器的阻塞，直到确定服务器已完成地址验证（见[QUIC-TRANSPORT]第8节）。也就是说，如果客户端没有收到对其握手或1-RTT包之一的确认，则客户端必须设置探测计时器。   
在握手完成之前，当生成的RTT样本很少甚至没有时，可能是由于客户端的RTT估计不正确而导致探测计时器过期的。为了允许客户机改进其RTT估计，它发送的新数据包必须是ack引发包。如果握手密钥对客户端可用，它必须发送一个握手包，否则它必须以至少1200字节的UDP数据报发送初始数据包。    
初始数据包和握手数据包永远不会被确认，但是当初始密钥和握手密钥被丢弃时，它们会从正在运行的字节中删除。

#### 5.3.1 发送探测包
当PTO计时器过期时，发送方必须在包号空间中发送至少一个ack引发包作为探测，除非没有可供发送的数据。一个Endpoint可以发送最多两个包含ack诱导包的全尺寸数据包，以避免由于单个丢失的数据包或从多个包号空间传输数据而导致的连续PTO的过期。   
除了在定时器过期的包号空间中发送数据外，发送方还应发送ack，确认从其他包号空间接收到的ack引发包，如有可能，合并包。实际上就是ack的捎带发送。    
当PTO计时器过期，并且存在新的或以前发送的未确认数据时，必须发送该数据。    
发送方可能没有新的或以前发送的数据要发送。作为一个例子，考虑以下事件序列：在**STREAM**帧中发送新的应用程序数据，被视为丢失，然后在新数据包中重新传输，然后又被确认原始传输的数据。当没有要发送的数据时，发送方应在单个数据包中发送**PING**或其他ack诱导帧，并重新启用PTO计时器。   
替代地，发送方可以将任何仍在飞行中的包标记为丢失，而不是发送ack诱导包。这样做可以避免发送额外的数据包，但会增加声明丢失的风险，从而导致拥塞控制器不必要地降低速率。    
连续的PTO周期呈指数级增长，因此，随着数据包在网络中的不断丢弃，连接恢复延迟呈指数级增加。在PTO过期时发送两个数据包可提高对数据包丢失的恢复能力，从而降低连续PTO事件的概率。   
在PTO上发送的探测包必须是ack引发包。如果可能的话，探测包应该携带新的数据。当新数据不可用时，当流控制不允许发送新数据时，或者为了偶然地减少丢失恢复延迟，探测包可以携带重传的未确认数据。实现可以使用替代策略来确定探测包的内容，包括基于应用程序的优先级发送新的或重新传输的数据。    
当PTO计时器多次过期且无法发送新数据时，实现必须在每次发送相同的有效负载还是发送不同的有效负载之间进行选择。发送相同的有效载荷可能更简单，并确保优先级最高的帧首先到达。每次发送不同的有效负载可以减少错误重新传输的机会。   

#### 5.3.2 损失检测
当接收到ACK帧新确认一个或多个包时，就可以确认已经发送的数据包是否丢失。   
PTO计时器过期事件不指示数据包丢失，并且不得导致先前未确认的数据包被标记为丢失。当接收到新确认包的确认时，丢失检测按照包和时间阈值机制进行；见第5.1节。

### 5.4 处理重试数据包
重试数据包会导致客户端发送另一个初始数据包，从而有效地重新启动连接过程。重试数据包表示已收到初始数据包，但未进行处理。不能将重试数据包视为确认，因为它不表示已处理数据包或指定数据包编号。    
接收到重试数据包的客户端将重置拥塞控制和丢失恢复状态，包括重置任何挂起的计时器。其他连接状态，特别是加密握手消息，被保留；参见[QUIC-TRANSPORT]第17.2.5节。   
客户端可以计算对服务器的RTT估计，作为从发送第一个初始值到接收到重试或版本协商包的时间段。客户端可以使用此值代替初始RTT估计的默认值。

### 5.5 丢弃密钥和包状态
当数据包保护密钥被丢弃时（见[QUIC-TLS]第4.10节），用这些密钥发送的所有数据包都不能再被确认，因为它们的确认不能再被处理。发送方必须丢弃与这些数据包相关联的所有恢复状态，并且必须将它们从正在传输的字节数中删除。Endpoint一旦开始交换握手包，就停止发送和接收初始包（见[QUIC-TRANSPORT]的17.2.2.1节）。此时，所有正在运行的初始数据包的恢复状态都将被丢弃。   
在RTT-0状态下，所有RTT-0恢复被丢弃。   
如果服务器接受0-RTT，但不缓冲在初始数据包之前到达的0-RTT数据包，则早期的0-RTT数据包将被声明为丢失，但这种情况并不常见。   
在用密钥加密的包被确认或声明丢失后，密钥将被丢弃。然而，一旦握手密钥可用，初始秘密可能会很快被销毁（见[QUIC-TLS]第4.10.1节）。

## 6 拥塞控制
本文档为QUIC[RFC6582]指定了一个Reno拥塞控制器。   
QUIC为拥塞控制提供的信号是通用的，并且设计为支持不同的算法。Endpoint可以单方面选择不同的算法来使用，比如Cubic[RFC8312]。   
如果Endpoint使用的控制器与本文件中指定的控制器不同，则所选控制器必须符合[RFC8085]第3.1节中规定的拥塞控制指南。   
本文档中的算法以字节为单位指定并使用控制器的拥塞窗口。   
如果某个Endpoint将导致"bytes_in_flight"（见附录B.2）大于拥塞窗口，则不得发送数据包，除非该数据包在PTO计时器过期时发送（见第5.2节）。   

### 6.1 显式拥塞通知
如果一条路径已经被验证支持ECN[RFC3168][RFC8311]，QUIC会将IP报头中的"Congestion Experienced"（CE）标识位作为为拥塞信号。本文档指定了当Endpoint的对端接收到"Congestion Experienced"的标识位的数据包时的响应。   

### 6.2 慢启动
QUIC在慢速启动时开始每个连接，并在ECN-CE计数器丢失或增加时退出慢启动。只要拥塞窗口小于ssthresh，QUIC就会重新进入慢速启动，这只会在声明持续拥塞之后发生。而在慢启动中，QUIC会在处理每个ACK确认时，通过确认的字节数来增加拥塞窗口的大小。

### 6.3 拥塞避免
慢启动退出后进入拥塞避免阶段。ewReno中的拥塞避免使用了一种加性增加乘性减少（AIMD）方法，该方法中，每个ACK确认都将使拥塞窗口增加一个最大包大小。当检测到丢失时，NewReno将拥塞窗口减半，并将慢启动阈值设置为新的拥塞窗口。   

### 6.4 恢复期
当检测到包的丢失或ECN-CE标记时，进入恢复期。当在恢复期间发送的包被确认时，恢复周期结束。这与TCP的恢复定义稍有不同，后者在确认开始恢复的丢失数据包时结束。   
恢复期将拥塞窗口减少限制为每往返一次。在恢复期间，无论ECN-CE计数器出现新的损失或增加，拥塞窗口都保持不变。

### 6.5 忽略不可加密数据包的丢失
在握手过程中，当数据包到达时，某些数据包保护密钥可能不可用。特别是，握手和0-RTT包在初始包到达之前不能被处理，而1-RTT包在握手完成之前不能被处理。Endpoint可以忽略握手、0-RTT和1-RTT包的丢失，这些包可能在对端拥有包保护密钥来处理这些包之前到达。

### 6.6 探测超时
探测包不能被拥塞控制器阻止。然而，发送方必须将这些数据包视为额外在飞(在传输中)，因为这些数据包在不造成数据包丢失的情况下增加了网络负载。请注意，发送探测包可能会导致发送方在传输中的字节超过拥塞窗口，直到接收到确认数据包丢失或传递的确认为止。    

### 6.7 持续拥塞
当接收到足够长的时间内发送的所有正在传输的包的丢失的ACK帧时，网络被认为正在经历持续的拥塞。通常，这可以通过连续PTO来建立，但是由于在发送新的ack引发包时PTO计时器被重置，因此必须使用显式的持续时间来解释PTO没有发生或被严重延迟的情况。持续时间计算如下：
```
(smoothed_rtt + 4 * rttvar + max_ack_delay) * kPersistentCongestionThreshold
```
例如，假设：
```
smoothed_rtt = 1 rttvar = 0 max_ack_delay = 0
kPersistentCongestionThreshold = 3
```
如果在time=0时发送ack引发包，则以下场景将说明持续拥塞：
```
+-----+------------------------+
| t=0 | Send Pkt #1 (App Data) |
+=====+========================+
| t=1 | Send Pkt #2 (PTO 1)    |
+-----+------------------------+
| t=3 | Send Pkt #3 (PTO 2)    |
+-----+------------------------+
| t=7 | Send Pkt #4 (PTO 3)    |
+-----+------------------------+
| t=8 | Recv ACK of Pkt #4     |
+-----+------------------------+
           表 1
```
当在t＝8处接收到包4的确认时，前三个包被确定为丢失。拥塞周期计算为最早和最新丢失数据包之间的时间：(3 - 0) = 3。持续性拥塞的持续时间等于（1*kPersistandClosessionThreshold）=3。因为达到了阈值，而且最老和最新的包之间没有任何包被确认，所以网络被认为经历了持续的拥塞。   
当建立持续拥塞时，发送方的拥塞窗口必须减少到最小拥塞窗口（kMinimumWindow）。这种在持续拥塞时关闭拥塞窗口的响应在功能上类似于TCP[RFC5681]中在尾部丢失探测（TLP）[RACK]之后发送方对重传超时（RTO）的响应。   

### 6.8 Pacing 发送速率
本文档没有指定一个pacer，但是建议发送方根据拥塞控制器的输入来调整所有正在传输的包的发送速度。例如，当与基于窗口的控制器一起使用时，pacer可以将拥塞窗口分布在平滑的RTT上，而pacer可能使用基于速率的控制器的速率估计。   
一个实现应该注意设计它的拥塞控制器，以便与pacer一起工作。例如，pacer可以包装拥塞控制器并控制拥塞窗口的可用性，或者pacer可以对拥塞控制器传递给它的数据包进行配速。及时发送ACK帧对于有效地恢复丢失非常重要。因此，应对包含其帧的ACK包进行速率调整，以避免其传送。   
将多个数据包发送到网络中而不在它们之间有任何延迟，这会导致数据包突发，从而导致短期的拥塞和丢失。
实现必须使用Pacing或将这种突发限制在初始拥塞窗口内，建议最小为10*max_datagram_size和max（2*max_datagram_size，14720）），其中max_datagram_size是连接数据报的当前最大大小，不包括UDP或IP开销。   
作为众所周知的、公开可用的flow pacer实现的示例，实现者可以参考Linux（3.11以后的版本）中的公平队列数据包调度器（fq qdisc）。   

### 6.9 拥塞窗口利用不足
当传输中的字节小于拥塞窗口且发送不受速度限制时，拥塞窗口利用率不足。当出现这种情况时，无论是慢速启动还是避免拥塞，都不应增加拥塞窗口。这可能是由于应用程序数据或流控制不足造成的。  
发送方可以使用[RFC7661]第4.3节中描述的pipeACK方法来确定是否充分利用了拥塞窗口。   
对数据包进行速度调整的发送方（见第6.8节）可能会延迟发送数据包，并且由于这种延迟而不能充分利用拥塞窗口。如果发送方可以充分利用拥塞窗口而不需要进行调步延迟，那么它不应该认为自己是应用程序受限的。   
发送方可以实现替代机制，以在使用不足的时段之后更新其拥塞窗口，例如在[RFC7661]中为TCP提议的那些机制。

## 7 安全考虑因素

### 7.1 拥挤信号
拥塞控制从根本上涉及到对来自未经验证的实体的信号（包括丢失和ECN码位）的消耗。路径攻击者可以伪造或更改这些信号。攻击者可以通过丢弃数据包使Endpoint降低其发送速率，或通过更改ECN码位来更改发送速率。   

### 7.2 交通分析
只携带ACK帧的包可以通过观察包大小来试探性地识别。确认模式可能会暴露有关链路特性或应用程序行为的信息。Endpoint可以使用**PADDING**帧或将确认与其他帧绑定，以减少泄漏的信息。   

### 7.3 误报ECN标记
接收方可以误报ECN标记来改变发送方的拥塞响应。抑制ECN-CE标记的报告可能会导致发送方增加其发送速率。这种增加可能会导致拥塞和丢包。   
发送方可以尝试通过标记他们用ECN-CE发送的偶发包来检测报告的抑制。如果使用ECN-CE发送的数据包在被确认时没有被报告为已被CE标记，则发送方应禁用该路径的ECN。   
报告额外的ECN-CE标记将导致发送方降低其发送速率，这在效果上类似于广告减少的连接流控制限制，因此这样做没有任何好处。   
Endpoint选择它们使用的拥塞控制器。虽然拥塞控制器通常将ECN-CE标记的报告视为等同于丢失[RFC8311]，但每个控制器的准确响应可能不同。因此，无法正确响应有关ECN标记的信息是很难检测到的。

## 8 IANA考虑
无

## 9 引用

### 9.1 规范性引用文件
[QUIC-TLS] Thomson, M., Ed. and S. Turner, Ed., "Using TLS to Secure
           QUIC", Work in Progress, Internet-Draft, draft-ietf-quic-
           tls-27, 9 March 2020,
           <https://tools.ietf.org/html/draft-ietf-quic-tls-27>.
[QUIC-TRANSPORT]
           Iyengar, J., Ed. and M. Thomson, Ed., "QUIC: A UDP-Based
           Multiplexed and Secure Transport", Work in Progress,
           Internet-Draft, draft-ietf-quic-transport-27, 9 March
           2020, <https://tools.ietf.org/html/draft-ietf-quic-
           transport-27>.
[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
           Requirement Levels", BCP 14, RFC 2119,
           DOI 10.17487/RFC2119, March 1997,
           <https://www.rfc-editor.org/info/rfc2119>.
[RFC8085]  Eggert, L., Fairhurst, G., and G. Shepherd, "UDP Usage
           Guidelines", BCP 145, RFC 8085, DOI 10.17487/RFC8085,
           March 2017, <https://www.rfc-editor.org/info/rfc8085>.
[RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
           2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
           May 2017, <https://www.rfc-editor.org/info/rfc8174>. 

### 9.2 资料性引用
[FACK]     Mathis, M. and J. Mahdavi, "Forward Acknowledgement:
           Refining TCP Congestion Control", ACM SIGCOMM , August
           1996.
[RACK]     Cheng, Y., Cardwell, N., Dukkipati, N., and P. Jha, "RACK:
           a time-based fast loss detection algorithm for TCP", Work
           in Progress, Internet-Draft, draft-ietf-tcpm-rack-07, 17
           January 2020, <http://www.ietf.org/internet-drafts/draft-
           ietf-tcpm-rack-07.txt>.
[RFC3168]  Ramakrishnan, K., Floyd, S., and D. Black, "The Addition
           of Explicit Congestion Notification (ECN) to IP",
           RFC 3168, DOI 10.17487/RFC3168, September 2001,
           <https://www.rfc-editor.org/info/rfc3168>.
[RFC4653]  Bhandarkar, S., Reddy, A. L. N., Allman, M., and E.
           Blanton, "Improving the Robustness of TCP to Non-
           Congestion Events", RFC 4653, DOI 10.17487/RFC4653, August
           2006, <https://www.rfc-editor.org/info/rfc4653>.
[RFC5681]  Allman, M., Paxson, V., and E. Blanton, "TCP Congestion
           Control", RFC 5681, DOI 10.17487/RFC5681, September 2009,
           <https://www.rfc-editor.org/info/rfc5681>.
[RFC5682]  Sarolahti, P., Kojo, M., Yamamoto, K., and M. Hata,
           "Forward RTO-Recovery (F-RTO): An Algorithm for Detecting
           Spurious Retransmission Timeouts with TCP", RFC 5682,
           DOI 10.17487/RFC5682, September 2009,
           <https://www.rfc-editor.org/info/rfc5682>.
[RFC5827]  Allman, M., Avrachenkov, K., Ayesta, U., Blanton, J., and
           P. Hurtig, "Early Retransmit for TCP and Stream Control
           Transmission Protocol (SCTP)", RFC 5827,
           DOI 10.17487/RFC5827, May 2010,
           <https://www.rfc-editor.org/info/rfc5827>.
[RFC6298]  Paxson, V., Allman, M., Chu, J., and M. Sargent,
           "Computing TCP’s Retransmission Timer", RFC 6298,
           DOI 10.17487/RFC6298, June 2011,
           <https://www.rfc-editor.org/info/rfc6298>.
[RFC6582]  Henderson, T., Floyd, S., Gurtov, A., and Y. Nishida, "The
           NewReno Modification to TCP’s Fast Recovery Algorithm",
           RFC 6582, DOI 10.17487/RFC6582, April 2012,
           <https://www.rfc-editor.org/info/rfc6582>.
[RFC6675]  Blanton, E., Allman, M., Wang, L., Jarvinen, I., Kojo, M.,
           and Y. Nishida, "A Conservative Loss Recovery Algorithm
           Based on Selective Acknowledgment (SACK) for TCP",
           RFC 6675, DOI 10.17487/RFC6675, August 2012,
           <https://www.rfc-editor.org/info/rfc6675>.
[RFC6928]  Chu, J., Dukkipati, N., Cheng, Y., and M. Mathis,
           "Increasing TCP’s Initial Window", RFC 6928,
           DOI 10.17487/RFC6928, April 2013,
           <https://www.rfc-editor.org/info/rfc6928>.
[RFC7661]  Fairhurst, G., Sathiaseelan, A., and R. Secchi, "Updating
           TCP to Support Rate-Limited Traffic", RFC 7661,
           DOI 10.17487/RFC7661, October 2015,
           <https://www.rfc-editor.org/info/rfc7661>.
[RFC8311]  Black, D., "Relaxing Restrictions on Explicit Congestion
           Notification (ECN) Experimentation", RFC 8311,
           DOI 10.17487/RFC8311, January 2018,
           <https://www.rfc-editor.org/info/rfc8311>.
[RFC8312]  Rhee, I., Xu, L., Ha, S., Zimmermann, A., Eggert, L., and
           R. Scheffenegger, "CUBIC for Fast Long-Distance Networks",
           RFC 8312, DOI 10.17487/RFC8312, February 2018,
           <https://www.rfc-editor.org/info/rfc8312>.

## 附录
。。。