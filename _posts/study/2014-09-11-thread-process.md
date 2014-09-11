---
layout: post
title: "进程和线程"
modified: 2014-09-11 10:50:56 +0800
tags: [thread]
image:
  feature: 
  credit: 
  creditlink: 
comments: ture
share: ture
---
##概念

###1.进程的概念

进程是表示资源分配的基本单位，又是调度运行的基本单位。例如，用户运行自己的程序，系统就创建一个进程，并为它分配资源，包括各种表格、内存空间、磁盘空间、I／O设备等。然后，把该进程放人进程的就绪队列。进程调度程序选中它，为它分配CPU以及其它有关资源，该进程才真正运行。所以，进程是系统中的并发执行的单位.      
在Mac、Windows NT等采用微内核结构的操作系统中，进程的功能发生了变化：它只是资源分配的单位，而不再是调度运行的单位。在微内核系统中，真正调度运行的基本单位是线程。因此，实现并发功能的单位是线程。

###2.线程的概念

线程是进程中执行运算的最小单位，亦即执行处理机调度的基本单位。如果把进程理解为在逻辑上操作系统所完成的任务，那么线程表示完成该任务的许多可能的子任务之一。例如，假设用户启动了一个窗口中的数据库应用程序，操作系统就将对数据库的调用表示为一个进程。假设用户要从数据库中产生一份工资单报表，并传到一个文件中，这是一个子任务；在产生工资单报表的过程中，用户又可以输人数据库查询请求，这又是一个子任务。这样，操作系统则把每一个请求――工资单报表和新输人的数据查询表示为数据库进程中的独立的线程。线程可以在处理器上独立调度执行，这样，在多处理器环境下就允许几个线程各自在单独处理器上进行。操作系统提供线程就是为了方便而有效地实现这种并发性

引入线程的好处      

- 易于调度。
- 提高并发性。通过线程可方便有效地实现并发性。进程可创建多个线程来执行同一程序的不同部分。
- 开销少。创建线程比创建进程要快，所需开销很少。。
- 利于充分发挥多处理器的功能。通过创建多线程进程（即一个进程可具有两个或更多个线程），每个线程在一个处理器上运行，从而实现应用程序的并发性，使每个处理器都得到充分运行。 

##他们的不同

- 进程是资源分配的基本单位，线程是cpu调度，或者说是程序执行的最小单位。在Mac、Windows NT等采用微内核结构的操作系统中，进程的功能发生了变化：它只是资源分配的单位，而不再是调度运行的单位。在微内核系统中，真正调度运行的基本单位是线程。因此，实现并发功能的单位是线程。

- 进程有独立的地址空间，比如在linux下面启动一个新的进程，系统必须分配给它独立的地址空间，建立众多的数据表来维护它的代码段、堆栈段和数据段，这是一种非常昂贵的多任务工作方式。而运行一个进程中的线程，它们之间共享大部分数据，使用相同的地址空间，因此启动一个线程，切换一个线程远比进程操作要快，花费也要小得多。当然，线程是拥有自己的局部变量和堆栈（注意不是堆）的，比如在windows中用_beginthreadex创建一个新进程就会在调用CreateThread的同时申请一个专属于线程的数据块（_tiddata)。

- 线程之间的通信比较方便。统一进程下的线程共享数据（比如全局变量，静态变量），通过这些数据来通信不仅快捷而且方便，当然如何处理好这些访问的同步与互斥正是编写多线程程序的难点。而进程之间的通信只能通过进程通信的方式进行。

- 由b，可以轻易地得到结论：多进程比多线程程序要健壮。一个线程死掉整个进程就死掉了，但是在保护模式下，一个进程死掉对另一个进程没有直接影响。

- 线程的执行与进程是有区别的。每个独立的线程有有自己的一个程序入口，顺序执行序列和程序的出口，但是线程不能独立执行，必须依附与程序之中，由应用程序提供多个线程的并发控制。


##应用情况

- 需要频繁创建销毁的优先用线程。
实例：web服务器。来一个建立一个线程，断了就销毁线程。要是用进程，创建和销毁的代价是很难承受的。
- 需要进行大量计算的优先使用线程。
所谓大量计算，当然就是要消耗很多cpu，切换频繁了，这种情况先线程是最合适的。
实例：图像处理、算法处理
- 强相关的处理用线程，若相关的处理用进程。
什么叫强相关、弱相关？理论上很难定义，给个简单的例子就明白了。       
一般的server需要完成如下任务：消息收发和消息处理。消息收发和消息处理就是弱相关的任务，而消息处理里面可能又分为消息解码、业务处理，这两个任务相对来说相关性就要强多了。因此消息收发和消息处理可以分进程设计，消息解码和业务处理可以分线程设计。
- 可能扩展到多机分布的用进程，多核分布的用线程。
- 都满足需求的情况下，用你最熟悉、最拿手的方式。

至于”数据共享、同步“、“编程、调试”、“可靠性”这几个维度的所谓的“复杂、简单”应该怎么取舍，只能说：没有明确的选择方法。一般有一个选择原则：如果多进程和多线程都能够满足要求，那么选择你最熟悉、最拿手的那个。