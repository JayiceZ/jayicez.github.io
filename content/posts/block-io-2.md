---
title: "Linux 块I/O子系统(二):请求流程"
date: 2022-11-30T12:20:27+08:00
tags: ["存储"]
description: Linux 块I/O子系统
---

# 块I/O子系统请求处理流程
## 基础流程

Linux通用块层提供给上层的接口函数是`submit_bio()`

```c
void submit_bio(int rw, struct bio *bio){
	bio->bi_rw |= rw;
	//省略一些参数检查 
	generic_make_request(bio);
}

void generic_make_request(struct bio *bio){
	...
	//获取目标块设备对应的request_queue，可以看到每个块设备都会有一个对应的request_queue结构
	struct request_queue* q= bdev_get_queue( bio-> bi_dev);
	
	//将bio添加到request_queue中
	q->make_request_fn(q, bio);
}
```

`make_request_fn()`由驱动层实现，比如对于SCSI设备来说，其实现为`blk_queue_bio()


```c
//块设备排入bio请求函数 
//该接口作为通用的块设备处理bio请求的方式，主要思路是尽可能的将bio合并request中或者生成新的request。 

static blk_qc_t blk_queue_bio(struct request_queue *q, struct bio *bio)
{
    //尝试从调度队列中找到一个request可以让bio合并进去，这里的返回值由调度器决定
	switch (elv_merge(q, &req, bio)) {
	//向后合并
	case ELEVATOR_BACK_MERGE:
		if (!bio_attempt_back_merge(q, req, bio))
			break;
		elv_bio_merged(q, req, bio);
		free = attempt_back_merge(q, req);
		if (free)
			__blk_put_request(q, free);
		else
			elv_merged_request(q, req, ELEVATOR_BACK_MERGE);
		//合并后，直接解锁
		return;
	//向后合并
	case ELEVATOR_FRONT_MERGE:
		if (!bio_attempt_front_merge(q, req, bio))
			break;
		elv_bio_merged(q, req, bio);
		free = attempt_front_merge(q, req);
		if (free)
			__blk_put_request(q, free);
		else
			elv_merged_request(q, req, ELEVATOR_FRONT_MERGE);
		return;
	default:
		break;
	}

    //如果找不到可以合并的，那就bio自己作为一个新的request
    req = get_request_wait(q, rw_flags, bio);
    
	//将该request加入调度队列
	add_acct_request(q, req, where);//默认where=ELEVATOR_INSERT_SORT
	
	//里面会调用request_fn()获取一个request来处理
	__blk_run_queue(q)

```
到此为止，bio以request的形态（也许是自己作为一个request，也许是被其他某个request给吞并了）进入`request_queue`	的调度队列中，接下来就会接受调度器的调度，并放入派发队列中。

`__blk_run_queue`会调用`requet_fn()`来获取派发队列中的request们来处理（哪些request能从调度队列进入派发队列呢？调度器说了算，给你返回啥你就处理啥）。

对于SCSI设备来说，`request_fn()`的实现为`scsi_request_fn`，其实现如下。
```c
static void scsi_request_fn(struct request_queue *q)
{
    struct scsi_device *sdev = q->queuedata;
    struct Scsi_Host *shost;
    struct scsi_cmnd *cmd;
    struct request *req;
    ...
    //派发队列中没有更多的request了，就退出
    for (;;) {
    	//从派发队列中中获取request
        req = blk_peek_request(q);
        
        // 按照scsi协议将request转化为scsi命令
        cmd = req->special;
        scsi_init_cmd_errh(cmd);
        
        //将scsi命令发到底层块设备中
        rtn = scsi_dispatch_cmd(cmd);
        ...
}
```
该函数的主要作用就是从请求队列取出请求，并把请求转换为对应的scsi命令，最终由scsi_dispatch_cmd(cmd)将命令交给下层控制器。

# 总结
至此，我们描述了一个块I/O子系统请求是如何从`bio`转化为`request`，然后再发给底层设备进行命令的执行。

但其实在块通用层，我们还有`plug/unplug机制`来作请求的batch提交，所以其实请求的转移会比本文描述的稍更复杂一些。

下一篇文章我们讲述一下`plug/unplug机制`。