---
title: "Linux 块I/O子系统(三):plug/unplug机制"
date: 2022-12-02T17:34:21+08:00
tags: ["存储"]
description: Linux 块I/O子系统
---


# plug/unplug机制

先看一段文件系统使用`submit_iob()`提交I/O请求的代码

```c
static ssize_t __blkdev_direct_IO(struct kiocb *iocb, struct iov_iter *iter, int nr_page){

	...
	blk_start_plug(&plug);
	for(;;){
		//对bio作初始化，先忽略
		...
		
		submit_bio();
	}
	blk_finish_plug(&plug);
}	
```

可以看到在调用`submit_bio()`的前后，好像还有一个启停`plug`的行为，这玩意是个啥呢？

其实，这是块I/O子系统的**蓄流/泄流（plug/unplug）机制**。

- 蓄流，就是块I/O子系统并不会把上层提交的`bio`马上交给`request_queue`，而是会先将其保存在一个`plug队列`（**每个进程独有一个**）里，在合适的时机清空队列，将里面的请求批量下发给`request_queue`，其实就是一种batch的模式。`plug`队列里存的也是`request`。

- 泄流，就是将`plug队列`里存起来的请求下发到`request_queue`中，并调用`request_fn()`来处理请求。因为`plug`队列里请求可能涉及不同块设备，所以一次泄流可能会调用多次`request_fn()`（涉及几个块设备，就会被调用几次）


### 蓄流
所以,`make_request_fn()`（其实现为上文的`blk_queue_bio`）的实现并不是简单地将`bio`合并到`request_queue`中，在此之前，其实还会将其尝试加入到`plug`队列里保存起来，新的代码如下：


```c
//块设备排入bio请求函数 
//该接口作为通用的块设备处理bio请求的方式，主要思路是尽可能的将bio合并request中或者生成新的request。 

static blk_qc_t blk_queue_bio(struct request_queue *q, struct bio *bio)
{
+	//尝试将bio与plug队列中的request合并
+	if (!blk_queue_nomerges(q)) {
+		//如果合并成功，直接返回
+		if (blk_attempt_plug_merge(q, bio, &request_count, NULL))
+			return;
+	} else
+		request_count = blk_plug_queued_count(q);

		
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
    
+    //尝试把这个新的request加入到plug队列中
+	if (plug) {
+		if (!request_count || list_empty(&plug->list))
+			trace_block_plug(q);
+		else {
+			struct request *last = list_entry_rq(plug->list.prev);
+	
+			//如果plug队列中的请求书大于阈值（16），则说明缓存得太多了，要先把它们flush到request_queue中
+			if (request_count >= BLK_MAX_REQUEST_COUNT ||
+			    blk_rq_bytes(last) >= BLK_PLUG_FLUSH_SIZE) {
+	
+				// flush到request_queue中，也即泄流
+				blk_flush_plug_list(plug, false);
+				trace_block_plug(q);
+			}
+		}
+		// 插入到plugd队列里
+		list_add_tail(&req->queuelist, &plug->list);
+		blk_account_io_start(req, true);
+		return;
	}	
+	//没有启用plug，才会将request直接加入request_queue中
	//将该request加入调度队列
	add_acct_request(q, req, where);//默认where=ELEVATOR_INSERT_SORT
	__blk_run_queue(q)

```

所以其总体的流程为
- 尝试将`bio`合并进`plug队列`中
	- 可以，直接返回
	- 不可以
		-   尝试将`bio`直接合并进`request_queue`中
			- 可以，直接返回
			- 不可以
				-	为此`bio` 创建一个单独的`request`，判断是否启用`plug`机制
					-	启用了，插入`plug`队列，返回
					-	没启用，插入`request_queue`队列，最终调用`request_fn()`来处理请求（不一定是这里生成的请求）

所以说，蓄流发生在文件系统向通用块层提交I/O请求时

### 泄流

泄流其实就是对应着`blk_flush_plug_list()`方法，它会将`plug队列`里的`request`flush到`request_queue`中，并调用`request_fn()`来处理

泄流有三种出现的时机：
- 上层应用如文件系统手动调用`blk_finish_plug()`时。

		blk_finish_plug(&plug)
			--blk_flush_plug_list(plug, false)
- 有新的`request`加入`plug队列`，却发现`plug队列`的请求数量太多（16个），此时会先对队列进行泄流。这个好理解，不能缓存太久
- 发生进程切换时，为防止被切走的进程上的I/O请求饥饿，切走之前也会泄流一次。但这里的泄流会异步进行（前面两个都是同步进行），后面会提到。
		
		schedule
    		 --sched_submit_work 
       		--blk_schedule_flush_plug()
            		--blk_flush_plug_list(plug, true)



`blk_flush_plug_list()`的实现为：
```c
void blk_flush_plug_list(struct blk_plug *plug, bool from_schedule)
{
    //对plug队列里的request进行排序，最终发往同一个设备的请求都会挨在一起，如1 1 1 3 3 4 4 4
    list_sort(NULL, &list, plug_rq_cmp);

    local_irq_save(flags);

    while (!list_empty(&list)) {
        rq = list_entry_rq(list.next);
        list_del_init(&rq->queuelist);
        BUG_ON(!rq->q);
        //判断前后两个请求是否发往同一个设备，若不是，说明同一个设备的请求都已经flush到request_queue了
        if (rq->q != q) {
            if (q)
            	//泄流，处理上一个设备的request_queue
                queue_unplugged(q, depth, from_schedule);
            q = rq->q;
            spin_lock(q->queue_lock);
        }
        //将该请求加入到request_queue中
        if (rq->cmd_flags & (REQ_FLUSH | REQ_FUA))
            __elv_add_request(q, rq, ELEVATOR_INSERT_FLUSH);
        else             
            __elv_add_request(q, rq, ELEVATOR_INSERT_SORT_MERGE);
    }
    //泄流最后一个队列
    if (q)
        queue_unplugged(q, depth, from_schedule);
    local_irq_restore(flags);
}
```
这个函数做的事情：
- 对plug队列进行排序，使同一个设备的请求都挨在一起
- 从队列中获取`request`，插入到`request_queue`中
- 判断某个设备的请求是否都已经flush了
	- 是，对该`request_queue`调用`queue_unplugged（）`

```c

static void queue_unplugged(struct request_queue *q, unsigned int   depth, bool from_schedule) __releases(q->queue_lock)
{
    if (from_schedule)
        // 异步泄流        
        blk_run_queue_async(q);
    else 
    	// 同步泄流        
        __blk_run_queue(q);
    spin_unlock(q->queue_lock);
}

```
这个函数根据`from_schedule`为true，说明是异步泄流，反之同步泄流。
#### 同步泄流
同步泄流最终会执行：
```c
inline void __blk_run_queue_uncond(struct request_queue *q)
{
    if (unlikely(blk_queue_dead(q)))
        return;
     q->request_fn_active++;
     q->request_fn(q);
     q->request_fn_active--;
}
```

最终会调用`request_fn()`来处理请求，该实现前文已经贴过了。


#### 异步泄流

而对于异步泄流，之前也提到了，异步泄流主要是进程切换的时候会出现。


其最终会调用到
```c
void blk_run_queue_async(struct request_queue *q)
{
    if (likely(!blk_queue_stopped(q) && !blk_queue_dead(q)))
        mod_delayed_work(kblockd_workqueue, &q->delay_work, 0);
}
```
其实就是注册一个异步任务，


# 总结
以上，我们分析了块I/O子系统中`plug/unplug`机制优化。

但至此，I/O调度层对我们来说还是一个黑盒，它是如何判断`bio`是否可以被合并、如何决定哪些`request`可以被放进派发队列中然后被`request_fn()`获取到，我们都还不得而知。

接下来我们会分析一下I/O调度层，分析一下里面几种I/O调度器的实现。