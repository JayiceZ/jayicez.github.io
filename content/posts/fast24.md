---
title: "FAST'24 速读"
date: 2024-03-19T21:20:27+08:00
tags: ["存储","论文"]
description: 用txyz粗读一下FAST'24感兴趣的论文
---


(笔记向)

扫了一眼题目，感兴趣的大概有几篇：
- Combining Buffered I/O and Direct I/O in Distributed File Systems
- IONIA: High-Performance Replication for Modern Disk-based KV Stores
- MIDAS: Minimizing Write Amplification in Log-Structured Systems through Adaptive Group Number and Size Configuration
- What's the Story in EBS Glory: Evolutions and Lessons in Building Cloud Block Store（Best Paper）

用txyz.ai辅助粗读了一下

## What’s the Story in EBS Glory: Evolutions and Lessons in Building Cloud Block Store
今年的Best Paper

文章描述了阿里云块存储EBS的三代架构

EBS1：一个虚拟磁盘VD被绑定到BlockServer上，路由由BlockManager维护。
一个VD被切割成64MB的chunk，ChunkManager维护Chunk到ChunkServer的映射，每个Chunk对应ChunkServer本地64MB的ext4文件，三副本。
BlockManager和ChunkManager都用Paxos做HA

[![pF4eIJ0.jpg](https://s21.ax1x.com/2024/03/24/pF4eIJ0.jpg)](https://imgse.com/i/pF4eIJ0)

缺陷：
1. 一个VD只能由一个BlockServer提供服务，存在瓶颈
2. 内核网络协议栈和HDD难以提供高质服务
3. 三副本，成本高

EBS2：
1. 去掉了ChunkManager和ChunkServer，使用Append-only的Pangu作为存储底座。
2. 因为底层存储是Append Only的，所以上层BlockServer需要维护索引（LSM Tree，也是基于Pangu）来实现业务的随机写
3. 对VD划分Segment，每个Segment会隶属一个BlockServer，消除了单BlockServer的瓶颈。Segment Group按照2MB切块，round-robin分配在Segment上。
每个Segment对应多个512MB的Pangu DataFile（3副本）

因为底层是Append-Only的，所以会产生空洞数据，需要GC对Data File进行rewrite，GC会将数据压缩后写入EC DataFile（所以这是个后台EC）

[![pF4e5iq.jpg](https://s21.ax1x.com/2024/03/24/pF4e5iq.jpg)](https://imgse.com/i/pF4e5iq)

[![pF4ehon.jpg](https://s21.ax1x.com/2024/03/24/pF4ehon.jpg)](https://imgse.com/i/pF4ehon)

缺点：
1. 三副本转EC的过程存在网络流量放大。

（但在线EC不好弄，因为小IO会有读写放大，攒一个大IO可以解决，但会提高延迟，块存储恰好又是性能敏感的系统

EBS3：
1. 在BlockServer内增加一个Fusion Write Engine的组件，聚合多个VD的写入，攒满16KB或者超过8us就用FPGA进行压缩，然后将压缩结果写入EC Journal File中（只在恢复的时候会使用到），同时也把数据写到缓存中，批量写入EC Data File后，更新索引、删除缓存
2. 高速网络

[![pF4efds.jpg](https://s21.ax1x.com/2024/03/24/pF4efds.jpg)](https://imgse.com/i/pF4efds)

（唔...这个将多租户的数据一起batch写入的想法也比较自然，用户量上来了的话攒到batch的概率很大。但是对于单个用户来说肯定就要牺牲一点latency

   
﻿
## Combining Buffered I/O and Direct I/O in Distributed File Systems
文章主要对比了buffered io和direct io不同workload下的一些表现，并做了个名为autoIO的引擎，能根据IO size、内存压力等因素来动态切换direct和buffered。
﻿
首先对比了在不同IO size下，DIO和BIO的表现：


﻿[![pFWl0xK.jpg](https://s21.ax1x.com/2024/03/19/pFWl0xK.jpg)](https://imgse.com/i/pFWl0xK)

﻿
随着io size加大，DIO会超越BIO。
论文认为的原因是在小size下，DIO suffer from 存储设备的高延迟，而BIO则收益于Page Cache的加速。但是在大size下，BIO会在维护cache上有很大的开销，比如分配内存页、LRU锁、内存拷贝。
﻿
软件层的开销占比，page cache的维护是大头：

﻿ [![pFWlaP1.jpg](https://s21.ax1x.com/2024/03/19/pFWlaP1.jpg)](https://imgse.com/i/pFWlaP1)

﻿
然后列举了一些workload的适用场景

﻿[![pFWlw26.jpg](https://s21.ax1x.com/2024/03/19/pFWlw26.jpg)](https://imgse.com/i/pFWlw26)

﻿
总的来说，就是大的顺序IO用DIO、内存压力大用DIO
﻿
提出了一个autoIO引擎，根据workload来切换io模式，算法：

[![pFWld8x.jpg](https://s21.ax1x.com/2024/03/19/pFWld8x.jpg)](https://imgse.com/i/pFWld8x)

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

IONIA的写入是客户端并行地写入quorum副本，实现blind write，1RTT就直接返回，写入的排序和apply被推出到后台执行

对于读操作来说。如果是读leader，读的这个key可能已经apply了，则直接读；也可能还没有排序并同步给follower，则需要等这个过程结束
如果是读follower，需要并行地读follower状态机和leader的meta，并在client侧校验从follower读到的这个key是否足够新鲜，否则重试。

（PS：如果写后马上读，那应该有不小概率需要重试？）

leader以全内存的方式维护这个meta，所以只能维护最近一段时间的。那么在查询的时候，如果发现key不在meta中，那么leader会返回一个当前最大的index来保证不破坏线性一致性


## RFUSE
背景：FUSE性能不够好，希望开发一个兼容原有FUSE的框架，叫RFUSE

RFUSE的核心在于更高效的用户态与内核态的信息传输：

[![pF4uprD.jpg](https://s21.ax1x.com/2024/03/24/pF4uprD.jpg)](https://imgse.com/i/pF4uprD)

1. 参考io uring，引入了Ring Channel做内核和用户的通信，这个Ring Channel是Per Core的，分配在同一NUMA 节点上，且通过mmap让内核和用户态共享的。
2. 通信机制是混合了轮训+事件唤醒。也就是一段时间（50us）轮训不到消息的话，会进入sleep等待被唤醒。
3. RingBuffer之间负载均衡，cost sensitive。

嗯...核心思想好像就差不多这些了，其他都是测试结果

（PS：rfuse有代码开源出来：github.com/snu-csl/rfuse 用了fuse的项目应该都会被老板要求测一下吧，逃





