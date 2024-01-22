---
title: "VLDB'12 Serializable Snapshot Isolation in PostgreSQL"
date: 2024-01-21T17:20:27+08:00
tags: ["数据库","论文"]
description: PostgreSQL的SSI工程实现
---

论文介绍了PG在Snapshot Isolation基础上实现Serializability的工程细节，实现了[SIGMOD'08 Serializable Isolation for Snapshot Database](https://courses.cs.washington.edu/courses/cse444/08au/544M/READING-LIST/fekete-sigmod2008.pdf)（后面简称为SSI论文）中提出的SSI，也是第一个实现了SSI的生产级数据库

并且在此基础上，在只读事务、内存占用与降低false positive方面做了一些优化。实验表明这个SSI的实现只比原来的SI多造成了不到7%的开销，且远远小于基于S2PL实现的开销。




# 背景
论文介绍了一些基础知识，比如SI、Write-Skew、S2PL等问题，略过。

Adya提出了事务间的三种dependency：
- wr-dependency：如果T1写了一个数据，T2读到了这个数据，那么存在一个wr dependency，从T1指向T2。也暗示着T1的序列化点在T2之后
- ww-dependency：T1写了一个数据a，T2写了一个数据b覆盖了数据a。那么存在一个ww-dependency，从T1指向T2。也暗示着T1的序列化点在T2之后
- rw-anti-dependency：T1写了一个数据新版本，T2读到了数据的旧版本。那么存在一个T2指向T1的anti-dependency。因为T2并没有读到T1写的数据，所以暗示着T1的序列化点在T2之后。

其中rw-anti-dependency是个特殊的dependency。因为在SI中，wr依赖出现的话，T1肯定已经提交（否则T2读不到T1的数据呀），ww依赖也同理，否则会有写写冲突。只有rw反依赖是T1和T2在并发执行的。而这也是实现SSI的一个关键点

Adya提到write-skew的出现必然伴随着一个事务依赖环+2个rw反依赖。

而在[Making Snapshot Isolation Serializable](https://dsf.berkeley.edu/cs286/papers/ssi-tods2005.pdf)论文中对这一结论进行了补充：
1. 这两个rw反依赖肯定是连续的 即 T1 --rw--> T2 --rw--> T3
2. T3最先提交

图3

在[SIGMOD'08 Serializable Isolation for Snapshot Database](https://courses.cs.washington.edu/courses/cse444/08au/544M/READING-LIST/fekete-sigmod2008.pdf)中将两个连续的rw-anti-dependency称为危险结构，通过引入一个SIREAD锁来破话这个危险结构，防止write-skew的发生，进而实现Serializable

但这样破坏危险结构的方式是会有false positive的，很简单，你破坏两个连续的rw反依赖，但人家不一定是成环的呀。

论文评估了这个方案与其他Serializable的方案，认为SSI比S2PL和OCC的并发度要好，因为他们直接禁止了rw冲突的产生，而SSI允许一些rw冲突。

[Precisely Serializable Snapshot Isolation](https://www.cs.umb.edu/~srevilak/srevilak-dissertation.pdf)提出了PSSI，它会直接检测这个环而不是危险结构，这样大大false positive，但因为需要追踪所有依赖，开销实在太大了。

# SSI的变体
PG的SSI对SSI论文里提到的算法做了一些改进

包括

## 只读事务优化



