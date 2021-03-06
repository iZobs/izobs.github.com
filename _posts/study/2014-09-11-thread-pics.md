---
layout: post
title: "进程间通信和线程间通信"
modified: 2014-09-11 10:22:10 +0800
tags: [thread]
image:
  feature: 
  credit: 
  creditlink: 
comments: ture
share: ture
---

##1.进程间通讯

###管道(pipe)
管道是一种半双工的通信方式，数据只能单向流动，而且只能在具有亲缘关系的进程间使用。进程的亲缘关系通常是指父子进程关系.

> 管道的特点:

管道的主要局限性正体现在它的特点上： 只支持单向数据流； 
只能用于具有亲缘关系的进程之间； 没有名字； 
管道的缓冲区是有限的（管道制存在于内存中，在管道创建时，为缓冲区分配一个页面大小）； 管道所传送的是无格式字节流，这就要求管道的读出方和写入方必须事先约定好数据的格式，比如多少字节算作一个消息（或命令、或记录）等等；

###信号(signal)
信号是在软件层次上对中断机制的一种模拟，它是比较复杂的通信方式，用于通知进程有某事件发生，一个进程收到一个信号与处理器收到一个中断请求效果上可以说是一致的。

内核为进程生产信号，来响应不同的事件，这些事件就是信号源。信号源可以是：异常，其他进程，终端的中断（Ctrl-C，Ctrl+\等），作业的控制（前台，后台进程的管理等），分配额问题（cpu超时或文件过大等），内核通知（例如I/O就绪等），报警（计时器）。

###消息队列

消息队列是消息的链接表，它克服了上两种通信方式中信号量有限的缺点，具有写权限得进程可以按照一定得规则向消息队列中添加新信息；对消息队列有读权限得进程则可以从消息队列中读取信息。

消息队列克服了信号传递信息少，管道只能支持无格式字节流和缓冲区受限的缺点。

###共享内存
可以说这是最有用的进程间通信方式。它使得多个进程可以访问同一块内存空间，不同进程可以及时看到对方进程中对共享内存中数据得更新。这种方式需要依靠某种同步操作，如互斥锁和信号量等。

>缺点

共享内存针对消息缓冲的缺点改而利用内存缓冲区直接交换信息，无须复制，快捷、信息量大是其优点。但是共享内存的通信方式是通过将共享的内存缓冲区直接附加到进程的虚拟地址空间中来实现的．因此，这些进程之间的读写操作的同步问题操作系统无法实现。必须由各进程利用其他同步工具解决。另外，由于内存实体存在于计算机系统中．所以只能由处于同一个计算机系统中的诸进程共享。不方便网络通信

###嵌套字
socket也是一种进程间的通信机制，不过它与其他通信方式主要的区别是：它可以实现不同主机间的进程通信。一个套接口可以看做是进程间通信的端点（endpoint），每个套接口的名字是唯一的；其他进程可以访问，连接和进行数据通信

> 特点

##2.线程间通讯

###线程互斥

互斥意味着“排它”，即两个线程不能同时进入被互斥保护的代码。Linux下可以通过pthread_mutex_t 定义互斥体机制完成多线程的互斥操作，该机制的作用是对某个需要互斥的部分，在进入时先得到互斥体，如果没有得到互斥体，表明互斥部分被其它线程拥有，此时欲获取互斥体的线程阻塞，直到拥有该互斥体的线程完成互斥部分的操作为止

###线程同步
同步就是线程等待某个事件的发生。只有当等待的事件发生线程才继续执行，否则线程挂起并放弃处理器。当多个线程协作时，相互作用的任务必须在一定的条件下同步。    
　　Linux下的C语言编程有多种线程同步机制，最典型的是条件变量(condition variable)。pthread_cond_init用来创建一个条件变量，其函数原型为




