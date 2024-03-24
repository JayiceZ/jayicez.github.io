---
title: "SOCC'23  Fast and Elastic Metadata Management for Distributed File Systems"
date: 2024-03-19T15:20:27+08:00
tags: ["文件系统","论文"]
description: 字节HDFS namespace优化
---
## Motivation
HDFS fedoration不行
将fs元数据构建在分布式数据库上是一个解决扩展性的手段，但是严重牺牲了效率和性能，他们测试出：三个NameNode+2个DB服务器的性能 = 原本的一个NameNode

而且论文认为，将fs逻辑做进DB里面是一个颠覆性地工作，目前还没有被证明这是一个普遍适用的方法

```
building the file system logic into the DDBMS requires rip-and-replace upgrades of existing
 file system technology and has yet to be shown to be a generally applicable approach
```
## 架构
论文提出了一个三层架构的模式来代替HDFS的NameNode：
第一层是Proxy，第二层是Cacheing，第三层是DB

[![pF4u0o9.jpg](https://s21.ax1x.com/2024/03/24/pF4u0o9.jpg)](https://imgse.com/i/pF4u0o9)

* Proxy：根据path将请求路由到对应的NameNode节点
* NameNode：每个NameNode节点负责元数据的一个子集，全内存。即是cache，也是buffer
* DB：share nothing的分布式db，承载所有元数据。但上面的数据不一定是最新的，新的数据有可能在NameNode的Buffe中还没下刷

### DB层
元数据的存储方式：
FileScale在inode表中存目录的full path：

然后pname+name作为表的primary key
论文认为的好处是减少了底层数据库的递归查询（说的应该就是多层lookup），一个where pname like prefix%语句就能找到子树的所有节点
rename也是通过一个where pname like prefix% 语句来更新

[![pF4uDiR.jpg](https://s21.ax1x.com/2024/03/24/pF4uDiR.jpg)](https://imgse.com/i/pF4uDiR)

（PS：那一个上层目录的rename不是废了吗。。。）

理论上，定制一个DB和FileScale结合会有更高的性能，但是论文认为为特定场景定制东西短期会有收益，但是长期来看可能跟不上DB领域的前沿成果，特别是分布式数据库领域的研究很活跃，不断有新的进展，所以他们希望做到如果未来出现一个更牛逼的DB，可以随时切换过来用
```
it is well-known that purpose-building anything for a specific application yields big performance 
benefits in the short term, but generally fails to keep up with technology developments as research progress
```
但用使用通用DB的缺点是开销大，所以在架构上必须减轻对DB的适用，所以有了Cache层

Cache层

每个NameNode节点保存元数据的一个子集，充当cache和buffer的作用。但所有NameNode节点的并集不一定能覆盖所有元数据，也就是说，有一部分元数据可能是不存在于任何NameNode的，这部分元数据是直接操作DB
buffer啥时候flush下去？2个时机：
1. 定期刷
2. 需要有分布式事务的时候刷（比如说一个元数据操作涉及了三个NameNode的数据，那么需要被修改的数据就要先从cache刷到db，然后在db做分布式事务）

因为Cache层要做纯内存的写buffer，所以也需要有WAL。WAL写在远程文件系统上（？）。
DB层就是定时去读这些增量WAL，来更新自己的数据

### Proxy层
Proxy需要按照请求路径，将请求定位到Cache节点。怎么知道路由信息呢？这个信息是存在Zookeeper上的，path前缀 - > cache node

这个路由信息会被proxy缓存起来，不用每次都去查zookeeper

而子树分裂和merge时，zookeeper的信息会被修改，从而导致proxy上的缓存是不正确的，然后请求会到错误的Cache节点上。
怎么解决这个问题？Cache节点维护一些元数据，记录最近的子树变动。遇到错误的请求能重定向到正确的Cache节点


个人解读：
子树分裂merge的可能会导致长尾的因素：
1. 修改路由后的一段时间，肯定会有请求拿着旧的路由访问到错误的cache节点，然后再被重定向
2. cache节点的buffer中，如果有还没flush到db但要被分裂出去的元数据，就需要把这部分数据给flush。这里也会有一个长尾。
3. 子树分裂后放在一个新的Cache节点上，此时Cache节点没有缓存，会有冷启动的问题，也有长尾/