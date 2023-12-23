---
title: "Linux 块I/O子系统(一):关键对象"
date: 2022-11-28T17:40:57+08:00
tags: ["hugo", "website", "blog"]
description: s
---

# 概述
块设备是以固定长度的块为单元进行数据读写的存储设备。可以分为物理块设备（如硬盘、内存等），与逻辑块设备（硬盘分区、Device Mapper）。

Linux中负责块设备IO请求的模块称为块IO子系统。

这里祭出一张很经典的Linux I/O栈全貌图
[![zr3Zbd.jpg](https://s1.ax1x.com/2022/12/03/zr3Zbd.jpg)](https://imgse.com/i/zr3Zbd)

其中红框的部分就是块I/O子系统模块，其可以被分为如下三层：
[![zr3VDH.jpg](https://s1.ax1x.com/2022/12/03/zr3VDH.jpg)](https://imgse.com/i/zr3VDH)
- 通用块层：为各种类型块设备建立统一的模型，并为上层暴露I/O接口。上层通过`submit_bio()`向块I/O子系统提交请求。
- I/O调度层：为了优化寻址操作，减少磁盘寻道次数，内核不会简单地按照I/O请求的顺序来执行，而是在接受通用块层的I/O请求后，会对请求进行合并、排序与调度。再提交给块设备驱动层。
- 块设备驱动层：在收到调度之后的请求后，将其翻译成对应的协议后，提交给块设备。

总得来说，块I/O子系统处理I/O的流程是：上层应用（如文件系统）调用通用块层提供的接口提交I/O请求（在Linux中以`bio`结构来描述），I/O调度器将`bio`包装为`request`结构，并对后者进行调度，选出最合适的`request`发到块设备驱动层，驱动层按照对应的协议（SCSI、SATA等）构建出符合协议的请求，再交由块设备进行解析与执行。

# 块I/O子系统关键对象
## 请求对象
块I/O子系统中的一个重点就是理解在请求过程中涉及的对象。按照上节的描述，主要包含两个请求：
- 上层应用提交到通用块层的请求，以`bio`结构表示。
- 通用块层提交到块设备驱动层的请求，以`request`结构表示。

而它们的关系是：
一个`request`请求包含一个或多个`bio`请求，这取决于多个`bio`请求的数据在硬盘上是否连续、是否符合电梯算法等。也就是说，一次SCSI命令的执行有可能响应了多个上层应用的请求。这就是所谓的请求合并。
而`bio`请求能否被合并，向前合并还是想后合并，是`I/O调度器`的决策。Linux系统主要支持3种I/O调度器
- CFQ调度器
- NOOP调度器
- 最后期限调度器（DeadLine调度器）

### bio
bio结构中的关键域
```c
struct bio{
	...
	struct block_device* bi_bdev; //指向块设备描述符的指针。以此来定位到块设备。
	sector_t bi_sector; //该I/O操作在磁盘上的起始扇区编号
	unsigned long bi_rw; //I/O操作的标志。低位表示读写，高位表示该I/O请求的优先级。
	
	/*
	指向`bio_vec`（代表了一段内存segment）数组的指针。这是为了支持向量I/O（或叫聚散I/O）
	如readv(3)、writev(3)。比如对于读操作来说，从硬盘读出的数据可能会放到内存上地址不连
	续的多个buf上，这里就是记录这些不连续的内存segment。
	*/
	struct bio_vec* bi_io_vec; 
	...
}
```


其中`bio_vec`结构为：
```c
struct bio_vec {
	struct page *bv_page; //指向该segment对应的page描述符
	unsigned int bv_len;  //该segment的长度
	unsigned int bv_offset; //该segment在page中的起始偏移量
}
```
bio的结构总览如下
[![zr3FgO.jpg](https://s1.ax1x.com/2022/12/03/zr3FgO.jpg)](https://imgse.com/i/zr3FgO)
### request
`request`结构的关键域
```c
struct request{
	...
	struct request_queue *q; //指向该request所在的request_queue，是一个重要的结构
	struct bio* bio; //上文提到，一个request结构可能包含多个bio结构，所以此处为bio链表的头
	struct bio* biotail;  //bio链表的尾
	struct gendisk* rq_disk; //指向对应硬盘描述符的指针。每个硬盘插入机器然后被系统识别后，都会创建一个gendisk结构
	sector_t _sector; //请求起始扇区编号
}
```

`request`结构会被添加进`request_queue`中，在里面经过调度后，被块设备驱动层消费到。

## 队列
此外，块I/O子系统的请求提交是以队列的模型实现的，在现在的Linux版本中，并存着单队列与多队列（blkmq）的模型。
[![zr3kvD.jpg](https://s1.ax1x.com/2022/12/03/zr3kvD.jpg)](https://imgse.com/i/zr3kvD)
这里先介绍单队列模式，在代码中以`request_queue`表示，所有`bio`请求都会被提交至此，在里面转化为`request`继而被调度。
### request_queue

`request_queue`是个抽象的概念，只是代表了一种`生产者/消费者`的模型，其由块设备驱动层实现，实际的实现不一定是队列，也不一定只有一个队列。

`request_queue`结构的关键域

```c
struct request_queue{
	...
	evevator_t* elevator; //指向I/O调度器描述符的指针。可以看到驱动层在实现时可以选择使用哪个调度器。
	make_request_fn* make_request_fn //外部通过这个函数将bio传到request_queue中
	request_fn_proc* request_fn //块设备驱动通过这个函数从request_queue中获取request
	...
}
```
其中有两个关键的函数：
- `int(make_request_fn)(struct request_queue *q, struct bio *bio)` : 将bio添加进队列中，由通用块层进行调用。
- `void(request_fn_proc)(struct request_queue *q)`:从队列里消费数据，获取到`request`。由块设备驱动层调用。

这两个接口的实现都是由块设备驱动来完成，比如说`make_request_fn()`内部会以怎样的方式将`bio`转化为`request`、转化后如何调度选出合适的`request`作为`request_fn()`的返回值，都需要驱动来实现，并在块设备初始化时完成相关函数的注册。

`request_queue`的经典的实现中，包含了`I/O调度队列`与`派发队列`
- I/O调度队列：这个队列保存`request`结构，每个`bio`结构都会先尝试在I/O调度队列中找到合适的`request`并将自己合并进去。如果没找到，再把自己单独作为一个`request`加入进去。另外，I/O调度队列同样只是一个代表`生产者消费者`的模型，比如`最后期限调度器`中就用了4个队列来实现这个模块。
- 派发队列：每个块设备对应一个派发队列。I/O调度器会对I/O调度队列中的`request`按照特定的算法进行调度，然后会将选出的`request`加入到派发队列中。块设备驱动层会从派发队列中消费。`request_fn()`就是从该队列中获取请求。

# 总结
至此，我们描述了块I/O子系统最关键的三个对象
- bio
- request
- request_queue

下一篇文章我们拆解一下I/O请求从提交到被响应，经过了哪些流程。

