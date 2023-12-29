---
title: "『Paper Notes』Aries Recovery"
date: 2023-12-27T17:20:27+08:00
tags: ["数据库","论文"]
description: 不衰的经典 ARIES事务恢复
---

Aries描述了一个数据库Recovery算法，用以保证事务的【A】和【D】，主要包括俩部分：
- Runtime时的协议。包括WAL、checkpoint、steal、no-force等
- Recovery时的协议。包括状态重建、回滚算法等。

## steal？force？
Aries算法建立在Page模型数据库上。一个Page要么在磁盘上，要么同时存在磁盘和内存上，如果内存上的数据和磁盘上的数据不一致，则称其未Dirty Page。
内存中的Page由Buffer Manager管理，决定什么时候换出到磁盘以及换入到内存。（不多赘述
对于Page的换出换入，有多种策略：
- steal：还没有提交的事务写的数据也能被写入磁盘。Runtime友好
- no-steal：写入磁盘的数据必须是已提交事务写入的
- force：事务提交时必须将其所有改动写入磁盘。Recovery友好
- no-force：事务提交不必马上将改动写入磁盘

steal能让Buffer Manager的调度更加自由，且能提供更高的吞吐，如果有一百个事务都修改了同一个Page，如果采用no-steal策略，必须等这一百个事务都commit了，这个Page才能写盘；更糟糕的情况，如果一直都有事务来写这个page，那这个page就一直刷不了盘了。除非隔段时间lock住页面，禁止所有新事务都写入并等待已有事务全部提交。这显然会影响吞吐。

Aries选用的是steal+no-force，这是runtime性能最好的一个方案，也是大多数数据库的选择。



## 术语
Aries中的概念非常多...第一次看很难记得住。

- BM：Buffer Manager：管理内存中的Page，并决定其换出换入
- Log Record：一条日志记录。Aries中提及的Log Record有三种：
    - redo log：记录事务对数据页的修改操作，当数据库crash需要recover时，可以逐条redo log来对页面进行replay，还原出crash时系统的状态。因为Aries采用的是no-force策略，事务提交时不强制刷新脏页，为了保证数据不丢失，会要求事务的redo log必须刷盘。
    - undo log：记录回滚一个事务修改所需要的信息。因为aries采用steal策略，磁盘上的数据可能是还未提交的数据。recover时，需要知道怎么把这些数据还原回去。
    - CLR：Compensation Log Record。aries中的undo log是逻辑日志，而CLR可以认为是undo log的物理执行；或者说是undo log恢复时的redo log；还有一个功能是记录当前undo的进度，因为逻辑undo log是非幂等的，所以必须又一个地方记录undo执行到了什么位置。
- LSN：Log的序号，单调递增，标识一个log
- Prev LSN：Log Record的一个属性，记录同一事务的上一条log。因为一个事务的log不一定在物理上连续。依赖这个字段构建出一个事务的log列表。
- Page LSN：Page的一个属性，最近一次修改这个Page的Log对应的LSN。
- Rec LSN：Page的属性。上一次刷盘后，第一次修改Page的Log LSN。根据这个来判断需要从哪里开始redo
- Last LSN：事务的属性。事务的最新一个更新操作的Log LSN
- UndoNext LSN：只存在于CLR中，记录下一条应该执行的undo log。（前面说明CLR有记录undo进度的作用
- DPT：Dirty Page Table。记录所有的Dirty Page及其Rec LSN。Recover时以此来redo
- ATT：Active Transaction Table。记录了活跃事务以其对应的Last LSN。Recover时以此来undo

## 逻辑日志or物理日志
