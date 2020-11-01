---
title:  Timing wheel心跳机制
categories:
- 工程
tags:
- 工程
---

在web服务中，断开空闲连接是一种减少资源浪费的一种手段，由此就有了心跳机制来判断一个连接是否空闲。

<!-- more --> 

# 一种简单粗暴的方式：
1. 服务端每个连接保存一个最后一次操作的时间戳，每次这个连接对应fd可读时（客户端发来请求），就更新一下时间戳。

2. 服务端会起一个定时任务: close掉在时间戳(now – heart_beat)时刻之前的fd。

这种方式需要不断的遍历已有连接，检查是否过期。

本文介绍的是，George Varghese 和 Tony Lauck 1996 年的论文《Hashed and Hierarchical Timing Wheels: data structures to efficiently implement a timer facility》中提出了一种时间轮(Timing wheel)管理time out事件的方式。

# 原理

下图是一个时间轮模型，假设当前心跳间隔是4S，将时间轮分为4分，每个格子表示当前格子的剩余寿命(s)。
![](/images/time_wheel_1.png)
每隔1S，pointer滚动一次，先清理掉0号格子存放的所有连接，然后当前时刻进来的连接放入(heart_beat – 1)号格子格子。

## 例子
当前时刻conn 1连入，此时conn1剩余寿命3S，放入3号格子
![](/images/time_wheel_2.png)
1S后，此时conn1剩余寿命2S
![](/images/time_wheel_3.png)
当conn1剩余寿命为0S时，此连接会被清理。如果恰好这一秒conn进行操作了，那么会放入3号格子另一个conn1，如果时间轮上所有的conn1都被清理，那么这个连接会被关闭。

# 实现：
C++以及一些指针友好型语言实现比较简单，轮子转动一次格子的指针引用数-1即可，当某个格子指针引用数为0时，代表格子时间到了，会析构掉。
事例代码可见: [journey-c(basket网络库)](https://github.com/lyuc0924/basket/tree/master/forward)中workthread的实现。
