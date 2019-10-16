<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [计算机网络](#计算机网络)
	- [应用层](#应用层)
		- [1.HTTP长连接和短连接](#1http长连接和短连接)
		- [2.HTTP报文](#2http报文)
		- [3.HTTP返回码](#3http返回码)
	- [传输层](#传输层)
		- [1.UDP or TCP](#1udp-or-tcp)
		- [2.TCP三次握手、四次挥手](#2tcp三次握手四次挥手)
		- [3.典型状态转移图](#3典型状态转移图)
		- [4.拥塞控制](#4拥塞控制)

<!-- /TOC -->
# 计算机网络
本文记录一些计算机网络相关的知识，用于复习
主要关注应用层和传输层

## 应用层
### 1.HTTP长连接和短连接
* 短连接每次请求维护一个全新的连接，每个连接都需要分配TCP缓冲区等，web服务器负担严重。且请求的每个对象，需要两倍的RTT
* 长连接会一直维护，直到达到设定时长没有请求才会关闭

从编写的webserver里看也基本是这样，长连接qps是短连接的3-4倍，符合上述理论
### 2.HTTP报文
![HTTP请求报文](/assets/HTTP请求报文.png)
![HTTP响应报文](/assets/HTTP响应报文_6pia0ymuj.png)
### 3.HTTP返回码
| 返回码类别 | 描述 |
| :---- | :----|
| 1** | 服务器收到请求，要求客户端继续操作 |
| 2** | 成功，请求接收并处理 |
| 3** | 重定向，需进一步操作以完成请求 |
| 4** | 客户端错误，请求错误或无法完成 |
| 5** | 服务器错误，处理中发生错误 |
常用的返回码：
* 200 OK: 请求成功，信息返回在响应报文中
* 301 Moved Permanently: 请求被转移到其他URL，并传回新的URL
* 400 Bad Request: 通用差错代码，服务器不理解请求
* 404 Not Found: 请求文档不在服务器中
* 505 HTTP Version Not Supported: 服务器不支持HTTP协议
## 传输层
### 1.UDP or TCP
* TCP包括面向连接以及可靠数据传输服务以及拥塞控制机制(有利于网络整体)
* UDP只提供必要服务的轻量级传输协议服务，不保证报文一定到达，也不保证报文一定按顺序到达。

因此，在需要可靠传输的场景中，选用TCP，在能够容忍丢失，但对速度下限要求较高的服务(网络电话、视频等多媒体服务)采用UDP。
### 2.TCP三次握手、四次挥手
* 三次握手
A->B 说明A能够发送数据
->B && B->A B能接收消息，且能发送
A->B 证明A能够发送
三次握手也就是客户端服务端互相确认对方收发没问题
* 四次挥手
A->B FIN 告知B，A将关闭发送
B->A ACK B知道A将关闭发送
B->A FIN B将关闭发送
A->B ACK 连接两端关闭
四次挥手是因为TCP是双向的数据流动，只有双方都不再发送的时候，连接才会彻底关闭
### 3.典型状态转移图
**client**: CLOSED->SYN_SENT->ESTABLISHED->FIN_WAIT_1->FIN_WAIT_2->TIME_WAIT->CLOSED
TIME_WAIT状态使TCP可客户能够重传缺失的最后ACK，如果没有，则服务端将一直等待这个ACK。
**server**: CLOSED->LISTEN->SYN_RCVD->ESTABLISHED->CLOSE_WAIT->LAST_ACK->CLOSED
### 4.拥塞控制
发送方限制未确认的数据不大于rwnd和cwnd的最小值，即:`LastByteSent - LastByteAcked <= min{cwnd, rwnd}`。

TCP以慢启动开始，如果发生丢包，则cwnd重设为1，ssthresh为cwnd/2。如果cwnd>=ssthresh，则cwnd不翻倍，而是更为谨慎的增大(+MSS)。如果连续收到3个冗余ACK，则进入快速恢复状态(ssthresh = cwnd/2, cwnd = ssthresh)。
