---
title: "『Paper Notes』Distributed Transactions at Scale in Amazon DynamoDB"
date: 2024-01-10T16:20:27+08:00
tags: ["数据库","论文"]
description: DynamoDB的事务实现
---

（论文第一章节就表明 我们DynamoDB和DynamoDB没关系！ 第二章节又提了一次我们名字差不多但架构完全不一样。233）

Transactions are submitted as single request.
DynamoDB不支持交互式事务，给出的原因是交互式事务往往是一个长事务，长时间持锁影响系统吞吐，而毕竟提供"predictable performance"是DynamoDB的一大设计原则。

DynamoDB所有的更新都是直接in-place，没实现MVCC，所以交互式事务对性能的影响会更大。


## 事务
事务请求需要经过coordinator走2pc，非事务请求直接在存储节点执行

To simplify the design and take advantage of low-contention workloads, DynamoDB uses an optimistic concurrency control scheme that avoids locking altogether.
DynamoDB采用乐观并发控制 OCC

用户接口上，提供了非事务的GetItem、PutItem、DeleteItem、UpdateItem接口，以及TransactGetItems和TransactWriteItems的事务接口。

其中TransactGetItems会提供一个一致性的快照视图，如果执行过程中遇到了写冲突会直接返回失败。

The TransactWriteItems may optionally include one or more preconditions on current values of the items. DynamoDB rejects the TransactWriteItems request if any of the preconditions are not met.
TransactWriteItems支持跨表事务，且支持condition写，文中给了一个用户下单的事务场景，其中的condition就是校验用户是否存在

## Timestamp Ordering
每个coordinator的时钟由AWS time-sync service分配，会有a few microseconds的误差。每个事务由其所在的coordinator节点分配事务ts，没有做commit-wait、read-wait之类。文中认为本地时钟也可以实现serializability，只不过更准确的时钟可以达到更高的事务成功率，以及事务的执行顺序更接近于事务发生时墙上时钟的顺序。
Although serializability holds even if the transaction coordinators do not have synchronized clocks, more accurate clocks result in more successful transactions and a serialization order that complies with real time.
确实，只不过做不到linearizability，目测应该是只能保证顺序一致性。


### write transaction protocol
item上存储一个write-ts，代表这个item最后一个更新事务的commit-ts。
除此之外，还在partition纬度维护一个max-delete-ts。因为如果对item的最新操作是一个删除，那么item没了自然也就没有write-ts了，所以需要额外记录一个delete-ts。

txn-ts、txn-id等事务元信息存储在事务所属的item上，像percolator那样
（但不知道是不是我理解错，论文说的似乎是每一个item？？ This metadata is attached to items that are part of the transaction

DynamoDB也采用2PC模型。prepare阶段存储节点主要在做以下检查：
- 所有的precondition需要满足
- 事务ts要大于写入item的write-ts
- 如果插入一个空的item，事务ts需要 > partition-max-delete-ts
- 没有其他prepare状态的先前事务正在准备写入这个item

（其实这里我有一点小疑问：这段伪代码展示的是对写入item的precondition，但上文的下单例子中展示的其实是"check A then write B"，那是不是得先去A加个SIREAD锁之类的，感觉对这段伪代码的逻辑会有一些改动。但论文没讲，好像默认就是只能check和write同一个item，但举的例子又不是，fine...

所有prepare成功后，再向所有参与者发起commit，全部成功后响应客户端。（没有异步resolve或者parallel commit之类的优化