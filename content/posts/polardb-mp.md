---
title: "PolarDB-MP 随记"
date: 2024-10-13T17:40:57+08:00
draft: false
toc: false
description: PolarDB的multi master
tags: 
  - Database
---

各家share-storage db都交出了multi-master的答卷，如Aurora-MM，Taurus-MM等。这次来看看与其他log as database架构不同、认为网络不是瓶颈的PolarDB是怎么解决这个问题的。

Taurus实现多主架构的核心点有2个：
1. 引入一个Global Lock Manager模块来做分布式锁，协调多主之间的并发访问。这种PCC方案与Aurora-MM的OCC相比能降低写写冲突时的abort rate。、
2. 向量时钟与标量时钟混用。其中用标量时钟来描述一个Page的log顺序，相当于LSN。因为写Page要去GLM中去拿X-Lock，所以这个因果关系很容易通过GLM这个中心点来传播；向量时钟用于在不同master之间定序，比如master a和master b会向c批量发log，这些log涉及到不同Page，如何描述这之间的偏序呢？通过向量时钟。

PolarDB则是引入了一个remote buffer pool，在这个rbp上做了一套类似MESI protocol，去协调各个master的一致性。其中这个rdb有三个核心组件：

- Transaction Fusion 做事务管理
- Buffer Fusion 做Page的一致性视图管理
- Lock Fusion做锁管理


## Transaction Fusion
每个master都有一个本地事务表（TIT），记录本地的事务，自己的TIT可以被其他master节点读到。这里其实和普通的事务可见性流程没什么区别，只是这个事务表不一定在本地，可能需要去其他master节点读。
事务要修改一行时，需要给这行标记上这次事务的txn_id, 以便其他事务根据这个txn_id来判断事务状态。事务提交后，获取一个ts写到本地TIT中，并写到这行的元数据里（但这个操作是lazy的，如果本事务提交，而其他事务没读到commit_ts, 那么它需要去txn_id所在的master上读一下TIT来判断事务状态）。


[![pFiziOf.md.jpg](https://github.com/user-attachments/assets/6f101d1c-780e-4935-84cf-a96614c9a7d7)


可见性判断伪代码

[![pFiziOf.md.jpg](https://github.com/user-attachments/assets/c8bf1fe5-8032-44e4-83b8-d967520072c1)

## Buffer Fusion

Remote Buffer Pool的核心，集中存储Page。

同时每个master也有自己的Local Buffer Pool，每个page会额外存两个字段：
- valid：指示local buffer pool中的这个page是否有效, master刷脏的时候会将其他master lbp中对应的page给invalid掉，使后者下次读这个page时去访问rdp。
- r_addr: rdp中这个page的地址

[![pFiziOf.md.jpg](https://github.com/user-attachments/assets/fa04002f-3350-4799-99bd-b9264979ab8b)

## Lock Fusion

记录锁信息，包括Page和raw的，master本地同样也会维护锁信息。

申请锁：
1. 判断本地是否已经持有这个锁，有则直接返回
2. RDMA访问lock fusion请求锁，没其他事务正在占用则请求成功。否则在lock fusion处记录一下wait-for关系，并通知持有锁的master节点，我在等你，你用完了记得马上释放。
3. 获得锁后，记录在本地

释放锁：
1. 判断是否有事务在等锁，如果没有，则将锁保持在本地，不通知lock fusion。如果有，则要释放给lock fusion， 后者根据wait-for关系唤醒等待的事务
2. 如果修改了Page，则释放锁时需要刷脏并invalid其他master节点上的page。



总得来说感觉这是个成本很高的架构。。。而且remote buffer pool应该也是需要做HA的，感觉这里还是很恶心的


