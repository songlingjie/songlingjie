---
title: Tcp小结
date: 2021-09-16 17:04:23
tags: 网络
categories: 
  - 技术笔记
  - 网络
banner_img: /img/tcp_end.jpg
index_img: /img/tcp_end.jpg
---

#### tcp 是基于连接、可靠的传输层的应用协议

- 基于连接
    - 通过3次握手建立连接
        - 客户端（包序号：seq=1 标志位：syn ）发起请求
        - 服务端（回复收到N字节之前的数据 ack=1 包序号 seq=1 标志位 syn ）回复请求
        - 客户端 （包序号：seq=1 ack=1）回复服务端，建立请求
        - 建立连接后服务端分配 连接缓存  变量信息

    - 通过4次挥手断开连接
        - 服务端（报序号 seq=x 标志位 fin）
        - 客户端（ack=x）
        - 客户端（seq=y ack=x 标志位 fin）
        - 服务端（ack=y）
    
- 可靠
    - 基于回退N补和选择重传协议，保证数据到达

- 拥塞控制
    - 保证发送端的速率 (cwnd= 拥塞窗口大小， rwnd=接受窗口大小，发送速率min(cwnd, rwnd))
        - 1、慢启动
            - 初始发送速率为 MSS 每收到一个ack 发送速率就翻倍，指数增加,ssthresh = cwnd/2
        - 2、拥塞避免
            - 遇到超时 或者 3个冗余ACK 发送速率减为 1 
        - 3、快递重启
            - 每个RTT 增加一个MSS 逐渐增加至ssthresh