---
title: "Transaction Repair for MVCC"
date: 2024-10-22T17:40:57+08:00
draft: false
toc: false
description: 局部修复read-set以实现Serializable
tags: 
  - Database
---


原文：https://dl.acm.org/doi/pdf/10.1145/3035918.3035919
很多OCC实现Serializable时都使用了类似write-snapshot-isolation的方案, 即事务执行需要4个流程：read-modify-validate-commit。

其中validate阶段需要校验自己的read-set在start_ts和commit_ts之间没有被其他已提交的事务破坏，否则的话就是出现了read-write-conflict，需要abort然后重试。

而如何做这个validate呢？
- SQL Server的Hekaton直接用predication重读一遍，看这两次的结果是否一致，简单粗暴  [Hekaton: SQL Server’s Memory-Optimized OLTP Engine。crdb叫read refresh](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/06/Hekaton-Sigmod2013-final.pdf)
- HyPer不维护read-set，而是维护predications，validate时判断是否有新提交的数据命中了自己的predications，有则validate失败。[Fast Serializable Multi-Version Concurrency Control for Main-Memory Database Systems](https://15721.courses.cs.cmu.edu/spring2019/papers/04-mvcc2/p677-neumann.pdf)

这个文章的核心思想是：在validate失败时进行局部repair而不是直接整个abort。

以一个转账的事务为例：

![1729593574089](https://github.com/user-attachments/assets/03b43813-bec1-463a-9ab5-53f88453717b)


其中，每个橙色阴影就是一个predicate，比如图中有P1、P2、P3三个predicate。每个predicate都有一个所属的closure，图中表示为灰色（或深灰色）。closure的执行依赖于predicate读到的read-set，所以如果一个predicate的read-set失效的话，其所属的closure需要重跑。

再者，可以构建出一个predicate tree，比如图中P1是根节点，在执行P1的closure时会产生了个新的predicate P2和P3，这两个就会是P1的子节点。

validate函数的输入：事务的predicate tree和一个validate ts

输出：两个数组，分别装着 通过和没有通过validate的predicate nodes。（如果一个predicate validate失败，那么其下所有的子predicate节点都算作失败）

伪代码：

![image](https://github.com/user-attachments/assets/be56f2f9-504c-4ca6-805a-41bfec798c3d)

repair过程，简单来说就是对所有validate失败的predicate节点进行剪枝，包括他们closure执行的结果，也要从undo buffer中去掉，然后以新的时间戳重跑这些失败的节点。

![image](https://github.com/user-attachments/assets/f87a0f31-1759-4bb6-9725-d4b1295700c9)

文中提到的节点优化：
1. 独占repair。因为repair和其他事务是并行的，repair时的新ts可能依然会validate失败，这会导致一直重试，为了避免这种情况，会尝试使用悲观锁，阻塞其他事务，确保repair成功（额...
2. attribute-level的validation，降低fail rate
3. reuse read set。就是说在repair的时候，可以重用前面validate时的read-set，只根据新的ts生成delta
