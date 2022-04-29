---
title: Tcp中的GBN
date: 2021-09-09 15:50:24
tags: 网络
categories: 
  - 技术笔记
  - 网络
banner_img: /img/tcp_gbn.jpeg
index_img: /img/tcp_gbn.jpeg
---
Tcp 中的 GBN(go back n ,回退N步)
```php
$pckData = [1,2,3,4,5,6];
// 发送端 用到的变量
$n = 4; // 窗口大小
$base = 1; // 下一个待确认的分组 表示 此分组之前的 数据组都已收到
$nextseq = 1;// 下一个待发送的数据组
$wNum = 4; // 最大允许发送的 数据组 表示 nextSeq <= wNum

// 接收端用到的变量
$expectedseqnum = 0; // 表示 此前的分组都已收到，下个收到的分组 应该是 $expectedseqnum +1， 如果不是 就忽略返回 ack = $expectedseqnum

// step 1  发送 编号为 1  的数据报
$c = [[1,2,3,4],[5,6]];
$base = 1;
$nextseq = 2;
$wNum = 4;

// step 2  发送 编号为 1  的数据报
$c = [[1,2,3,4],[5,6]];
$base = 1;
$nextseq = 3;
$wNum = 4;

// step 3  发送 编号为 3  的数据报
$c = [[1,2,3,4],[5,6]];
$base = 1;
$nextseq = 4;
$wNum = 4;

// 收到接受方 ack = 1($expectedseqnum = 1)
$base = 2;
$wNum = 5;// 窗口向右滑动 一

// step 2  发送超时

// 接受方售后编号为 3  的数据报 丢弃 返回 ack =1

// step 3  发送 编号为 3  的数据报
$c = [1,[2,3,4,5],[5,6]];
$base = 1;
$nextseq = 5;
$wNum = 5;
//$nextseq == $wNum  启动 计时 没有收到接收端 返回的ack =2 超时，重新发送 2，3，4，5
```

![72068A22-4A3F-48CE-B11B-37F24CAFFC64](https://tva1.sinaimg.cn/large/e6c9d24ely1h1ptqc0a1oj20ej0laab8.jpg)

