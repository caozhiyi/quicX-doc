
#使用TLS保护QUIC草案
## 摘要
本文描述了如何使用传输层安全性（TLS）来保护QUIC。


## 1 简介
本文档描述了如何使用TLS[TLS13]保护QUIC[QUIC-TRANSPORT]。    
TLS 1.3为以前版本的连接建立提供了关键的延迟改进。在没有数据包丢失的情况下，大多数新的连接都可以在一次往返中建立和保护；在同一客户机和服务器之间的后续连接上，客户机通常可以立即发送应用程序数据，即使用零往返设置。    
本文描述了TLS如何作为QUIC的安全组件。

## 2 符号约定

## 2.1 TLS概述
TLS为两个端点提供了一种在不受信任的介质（即Internet）上建立通信方式的方法，以确保它们交换的消息不能被观察、修改或伪造。    
在内部，TLS是一个分层协议，其结构如图1所示。
```
          +-------------+------------+--------------+---------+
Handshake |             |            |  Application |         |
Layer     |  Handshake  |   Alerts   |     Data     |   ...   |
          |             |            |              |         |
          +-------------+------------+--------------+---------+
Record    |                                                   |
Layer     |                      Records                      |
          |                                                   |
          +---------------------------------------------------+

                            Figure 1: TLS Layers
```
每个握手层消息（例如，握手、警报和应用程序数据）由记录层作为一系列类型化的TLS记录进行传输。记录被单独地加密保护，然后通过可靠的传输（通常是TCP）传输，它提供排序和有保证的传输。    
TLS认证密钥交换发生在两个端点之间：客户端和服务器。客户端启动交换，服务器响应。如果密钥交换成功完成，客户机和服务器都将同意一个密钥。TLS支持预共享密钥（PSK）和Diffie-Hellman在有限域或椭圆曲线（（EC）DHE）密钥交换上。PSK是0-RTT的基础，后者在（EC）DHE密钥被破坏时提供了完美的前向保密（PFS）。    
完成TLS握手后，客户机将识别并验证服务器的身份，并且服务器可以选择性地识别和验证客户机的身份。TLS支持服务器和客户端的基于X.509[RFC5280]证书的身份验证。    
TLS密钥交换能够抵抗攻击者的篡改，并且它产生的共享机密不能被任何一个参与的对等方控制。    
TLS提供了两种QUIC感兴趣的基本握手模式： 
+ 一种完整的1-RTT握手，在这种握手中，客户机能够在往返一次后发送应用程序数据，服务器在收到来自客户机的第一条握手消息后立即作出响应。
+ 一种0-RTT握手，在这种握手中，客户机使用它以前了解到的有关服务器的信息来立即发送应用程序数据。攻击者可以重放此应用程序数据，因此它不能携带任何非幂等操作的自包含触发器。   
 
图2显示了使用0-RTT应用程序数据的简化TLS握手。请注意，这忽略了在QUIC中没有使用的EndOfEarlyData消息（见第8.3节）。同样，QUIC不使用ChangeCipherSpec和KeyUpdate消息；ChangeCipherSpec在TLS 1.3中是冗余的，QUIC定义了自己的密钥更新机制（第6节）。
```
 Client                                             Server

 ClientHello
 (0-RTT Application Data)  -------->
                                                ServerHello
                                      {EncryptedExtensions}
                                                 {Finished}
                          <--------      [Application Data]
 {Finished}                -------->

 [Application Data]        <------->      [Application Data]

 () Indicates messages protected by Early Data (0-RTT) Keys
 {} Indicates messages protected using Handshake Keys
 [] Indicates messages protected using Application Data
    (1-RTT) Keys

            图 2: TLS 0-RTT 握手过程
```
使用多种加密级别保护数据：
+ 初始密钥
+ 早期数据（0-RTT）密钥
+ 握手密钥
+ 应用数据（1-RTT）密钥

应用程序数据可能只出现在早期数据和应用程序数据级别。握手和警报消息可以出现在任何级别。   
0-RTT握手只有在客户机和服务器之前进行过通信的情况下才可能。在1-RTT握手中，客户端在接收到服务器发送的所有握手消息之前无法发送受保护的应用程序数据。    

## 3 协议概述
