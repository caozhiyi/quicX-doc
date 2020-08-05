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
  - 第2节描述与Stream相关的核心概念
  - 第3节提供流状态的参考模型
  - 第4节概述了流量控制的操作
+ 连接是QUIC端点通信过程中的实体
  - 第5节描述了连接相关的核心概念   
  - 第6节描述了版本协商过程   
  - 第7节是关于连接建立过程的细节   
  - 第8节指点关键的服务拒绝环节机制   
  - 第9就描述了端点如何将连接迁移到另一个新的网络环境中   
  - 第10节列出了中断连接时的所有选项   
  - 第11节提供对错误处理的一般指导   
+ 包和帧是QUIC通信的基本单元
  - 第12节描述了关于包和帧的关键概念   
  - 第13节定义了传输，重传和确认的模型   
  - 第14节定义了包大小管理的规则   
+ 最后，展示了一些QUIC编码的细节
  - 第15节 版本   
  - 第16节 整型编码   
  - 第17节 包头   
  - 第18节 传输参数   
  - 第19节 传输帧   
  - 第20章 错误定义    
附录文件描述了本文未尽的一些QUIC细节，包括[拥塞控制](https://tools.ietf.org/html/draft-ietf-quic-recovery-27)，以及[QUIC-TLS](https://tools.ietf.org/html/draft-ietf-quic-recovery-27)。   

## 1.2 术语和定义
大写关键字 "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT","SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", “OPTIONAL”的使用与BCP14 [RFC2119] [RFC8174]相同。   
本文常用的术语定义如下：    
QUIC：本文描述的传输协议，QUIC只是一个名字，而不是一个缩略词。    
QUIC包：QUIC可以封装进UDP数据包的完整处理单元，多个QUIC包可以被放到一个UDP数据包中。   
ACK引发包：除了ACK，PADDING，CONNECTING外，其他包都包含的一种QUIC包，它使得接受方发送接受确认。    
乱序包：一种数据包，它的编号并不是接受方最大编号加一，当包传输延迟或者该包之前的包延迟或丢失，都可能造成包的乱序到达。