---
title: "『Paper Notes』一种无异常系统调用·OSDI '10"
date: 2022-03-30T20:32:24+08:00
tags: ["操作系统","论文"]
description: Flexible System Call Scheduling with Exception-Less System Calls论文笔记
---

>『Paper Notes』是一个对一些计算机领域顶会文章进行学习与分析的系列。文章一般来自于OSDI、NSDI、SOSP等顶会，主要聚焦于分布式系统、操作系统、存储等领域。主要介绍论文中的一些要点。`

>由于本人的水平非常低，论文有可能是很久之前就已经发表的（比如说40年前，逃）。

原文：[Flexible System Call Scheduling with Exception-Less System Calls](https://www.usenix.org/legacy/events/osdi10/tech/full_papers/Soares.pdf)

`该文是一篇2010年在OSDI会议发表的论文`
### Idea
论文评估了传统的同步系统调用对系统资源敏感型应用（比如Nginx、MySQL等）的影响。发现了系统调用很大程度上对性能产生了影响，主要是因为CPU Pipeline的刷新与一些关键CPU结构（TLB，CPU Cache）的污染。
论文提出了一种新机制：少异常的系统调用（exception-less system call）。它通过实现操作系统工作调度的灵活性来提高处理器效率，从而减少对CPU结构的污染影响。且提出了`FlexSC`,一个基于Linux的exception-less system call实现。可以在无需修改用户程序的情况下，将MySQL的性能提升了40%、Apache的性能提升了110%。


系统调用是操作系统内核的接口，用于请求由操作系统提供的服务。一直以来，大多数操作系统上的系统调用都是基于异常（exception）实现：将参数写入相关的寄存器，然后通过特定的指令触发一个异常（如x86的`INT 0x80`，RISC-V的`ECALL`），事先在异常向量表中注册好的handler就会处理该异常，待处理完毕后会才将控制权返还给用户进程。

而这样传统的系统调用模型会对应用程序带来怎样的性能影响呢？论文中给出了实验结果（基于Intel i7）：
![image0](../exception-less_syscall/img.png)
IPC：instructions  per  cycles 每个周期最执行的指令，该值越高，系统性能越好
该图描述了在用户程序正常运行过程中，调用了一个系统调用，用户IPC由1.5跌至了近乎0.3，并在数万个cycles才恢复过来。


上下文切换所引起的CPU结构污染（CPU Cache、TLB）也是不可忽视的，论文给出了几个常用系统调用对这些结构的影响
![image1](../exception-less_syscall/img_1.png)
数字代表Cache Line的数量，一个Cache Line为64Bytes。
i-cache代表被淘汰出CPU L1 Cache的指令部分；d-cache代表被淘汰出CPU L1 Cache的数据部分；d-TLB代表被淘汰的TLB

该CPU的L1-d-cache的大小为32kB，可以看到这些系统调用平均污染了一半的d-Cache。而L2与L3被污染的数量更多，是因为L2与L3有着更激进的指令预取。

而最直观能衡量系统调用的开销就是测量其对应用程序的影响。实际上，系统调用带来的开销主要来源于两方面：
- 直接因素：触发异常后，CPU流水线的失效
  >ps：这是因为现代CPU并不是一次性将一条指令直接执行完。比如说一条指令的执行可能会涉及到取指令，译指令、读取数据、操作数据、数据写回 等流程，现代CPU并不会将第一条指令全部执行完了，再去执行第二条指令，而是会比如说取完第一条指令后，马上去取第二、三、四条指令，即使第一条指令并没有真正执行完。而潜在的一个问题就是，当取完了一堆指令之后，第一条指令的执行才出现异常（比方说系统调用），那么之前的指令预取就是无用功。`
- 间接因素：CPU结构的污染（上图已展示过）

论文做了一个实验，测试不同系统调用频率下IPC的结果，且这个系统调用的执行是被直接跳过的（也就是说这个系统调用啥也没做）。理想情况下，IPC不应该因为这个空系统调用而减少，而实际情况下，这个实验可以测试`直接因素`。而在此基础上，把空系统调用替换为`pwrite`，这个实验的结果减去`直接因素`就得到了`间接因素`。
![image2](../exception-less_syscall/img_2.png)

可以看到，对IPC产生主要影响的是间接因素。

基于这样的结果，论文提出了`无异常系统调用`。这是一种不需要通过异常，而主要通过`共享内存（Shared Memory）`实现的一种系统调用，它有两个优点：
- 批处理系统调用：在应用程序允许的情况下，延缓系统调用的执行并批处理
- Core绑定：应用程序和系统调用处理系统可以在不同的核心上，有助于提高`局部性（locality)`

这是传统模式与论文提出的模式系统调用的区别
![image3](../exception-less_syscall/img_3.png)


System Call Page是一个共享内存页。这个Page包含若干entry，每个entry代表着每次系统调用的信息，包含系统调用号、请求状态、参数、返回值等。在64位的操作系统上，论文将这个entry的格式定义为：
![image4](../exception-less_syscall/img_4.png)


每个entry的大小为64字节。这是因为：
- Linux ABI允许系统调用最多有6个参数与1个返回值，共56个字节。其他三个字段（系统调用号、参数个数、请求状态）可以压缩到8个字节内。
- 64字节是现代CPU流行的Cache Line大小。

用户程序发起系统调用，需要在这个Page上找到空闲的一行，并写上一个entry。当系统调用执行完毕时，该entry的状态栏会办成`finished`的状态，此时用户程序可以主动检查到这一结果。


### 实现：FlexSC
论文为这种机制提供了一个实现，名为`flexible system call`，简称`FlexSC`。
FlexSC的实现很简单。一是需要一个runtime，根据System Call Page的内容去完成系统调用；二是需要提供接口：`flexsc_register()`与`flexsc_wait()`


- flexsc_register()：是一个系统调用，通过页面映射获取一个（一组）System Call Page。虽然其也是一个系统调用，但一般来说只会在第一次进行调用，所有产生的性能影响很小

- flexsc_wait()：当用户程序发起了系统调用之后而没有更多的事情要做（必须要同步等待结果），那么用户程序应该调用该函数等待相关entry的状态变成`finished`


前面提到FlexSC的实现一是需要一个runtime，论文将其统称为System Call Threads。而在实现时，其面临两个问题：
- 在传统Linux实现中，系统调用执行时需要占用相关进程的虚拟地址空间
- System Call Threads执行系统调用时遇到资源等待该如何处理

对于第一个问题，System Call Thread的创建实际上是在`flexsc_register()`被调用时，且它会`clone()`与原始进程，以此满足虚拟地址空间的需求。这样就能服用Linux原有的逻辑

对于第二个问题，论文中提到，System Call Thread的数量实际上与System Call Page中entry的数量是一样的。但在任意时间点，只有一个System Call Thread在运行，只有在其遇到资源等待时，其他System Call Thread才会接力去执行其他entry。
生成这么多System Call Thread看似很昂贵，但是System Call Thread实际上很轻量，其只需要一个`task_struct`与很小的堆栈，一共只需要10KB内存，而页表、文件表等是与原进程共享的。论文认为这个开销还是可以接受的。


而如果将应用程序中的同步系统调用改造成论文提及的这两个api，无疑会引入巨大的工作量，而且并发模型也会发生很大的改变（因为论文中提及的这种模型其实更像一个异步的模型）。所以论文中提及了一种无侵入式的改造：
- 将以上所描述的逻辑作为程序运行的Runtime，而并非让用户程序去改造。利用动态绑定的能力，将`libc`中的系统调用重定向到论文提供的库中
- 当用户级线程发起系统调用后，将其切换到其他用户级线程（ps：其实就是我们现在津津乐道的协程了）
- 当没有可用的用户级线程时，检查System Call Page上是否有准备好的entry，以此来唤醒用户级线程
- 若没有，调用flexsc_wait()等待entry准备好。

如图所示

![image5](../exception-less_syscall/img_5.png)

>ps:其实这个调度的逻辑，跟我们所熟知的goroutine之类的，还是稍显简陋的==，不过毕竟这也是12年前的论文。
> 
> 而且exception-less也并不是完全的，只不过将异常从对性能要求高的应用程序转移到了runtime的进程中。

在此之后，论文还在Apache、MySQL上测试了FlexSC带来的性能优化，有兴趣的可以自行参考原文，这里就不赘述了。

