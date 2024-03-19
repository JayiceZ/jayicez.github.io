---
title: "FAST'24 速读"
date: 2024-03-19T21:20:27+08:00
tags: ["存储","论文"]
description: 用txyz粗读一下FAST'24感兴趣的论文
---


扫了一眼题目，感兴趣的大概有几篇：
- Combining Buffered I/O and Direct I/O in Distributed File Systems
- IONIA: High-Performance Replication for Modern Disk-based KV Stores
- MIDAS: Minimizing Write Amplification in Log-Structured Systems through Adaptive Group Number and Size Configuration
- What's the Story in EBS Glory: Evolutions and Lessons in Building Cloud Block Store（Best Paper）

用txyz.ai辅助粗读了一下
﻿
## Combining Buffered I/O and Direct I/O in Distributed File Systems
文章主要对比了buffered io和direct io不同workload下的一些表现，并做了个名为autoIO的引擎，能根据IO size、内存压力等因素来动态切换direct和buffered。
﻿
首先对比了在不同IO size下，DIO和BIO的表现：
﻿

﻿[![pFWl0xK.jpg](https://s21.ax1x.com/2024/03/19/pFWl0xK.jpg)](https://imgse.com/i/pFWl0xK)

﻿
随着io size加大，DIO会超越BIO。
论文认为的原因是在小size下，DIO suffer from 存储设备的高延迟，而BIO则收益于Page Cache的加速。但是在大size下，BIO会在维护cache上有很大的开销，比如分配内存页、LRU锁、内存拷贝。
﻿
软件层的开销占比，page cache的维护是大头：

﻿ [![pFWlaP1.jpg](https://s21.ax1x.com/2024/03/19/pFWlaP1.jpg)](https://imgse.com/i/pFWlaP1)

﻿

﻿
﻿
然后列举了一些workload的适用场景

﻿[![pFWlw26.jpg](https://s21.ax1x.com/2024/03/19/pFWlw26.jpg)](https://imgse.com/i/pFWlw26)

﻿

﻿
总的来说，就是大的顺序IO用DIO、内存压力大用DIO
﻿
提出了一个autoIO引擎，根据workload来切换io模式，算法：

[![pFWld8x.jpg](https://s21.ax1x.com/2024/03/19/pFWld8x.jpg)](https://imgse.com/i/pFWld8x)

﻿

﻿
最后就是benchmark结果：遥遥领先
﻿
## IONIA: High-Performance Replication for Modern Disk-based KV Stores
这篇文章介绍了一种名为IONIA的新型复制协议，专为现代基于SSD的写优化键值存储（WOKV）设计。现有协议无法满足SSD-based WO-KV存储的需求，存在着性能和可用性方面的改进空间，IONIA利用SSD-based WO-KV存储的独特特性，通过存储感知设计实现高吞吐量的1RTT写入和可扩展的1RTT读取，同时不影响可用性。
﻿
论文认为现有复制协议无法发挥写优化存储引擎（如基于ssd的lsm）的性能，因为状态机为了避免不确定性，会采取顺序apply的方式，只能做到比较低的吞吐，无法发挥lsm的性能。而且读只能从leader读，或者由follower来执行read index机制（引入多一次RTT）。
而且写入会有两次RTT，client->leader->follower
﻿
 [![pFWlDKO.jpg](https://s21.ax1x.com/2024/03/19/pFWlDKO.jpg)](https://imgse.com/i/pFWlDKO)
﻿

﻿
（PS：我没get到的是，CURP协议应该是可以做到不冲突key的并行写入的，为什么write throughout被定义为low呢？）


TO BE FINISHED