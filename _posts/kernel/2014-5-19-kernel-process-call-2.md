---
layout: post
title: "Linux 进程管理和调度——进程复制fork & clone"
date: 2014-5-19 16:34
category: "学习"
tags: ["Kernel"]

---

###进程相关的系统调用

####1.进程复制
传统的UNIX中用于复制进程的系统调用是fork。但它并不是Linux为此实现的唯一调用，实际上Linux实现了3个。

- fork是重量级调用，因为它建立了父进程的一个完整副本，然后作为子进程执行。为减少与该调用相关的工作量，Linux使用了写时复制（copy-on-write）技术，下文中会讨论。
- vfork类似于fork，但并不创建父进程数据的副本。相反，父子进程之间共享数据。这节省了大量CPU时间（如果一个进程操纵共享数据，则另一个会自动注意到）。vfork设计用于子进程形成后立即执行execve系统调用加载新程序的情形。在子进程退出或开始新程序之前，内核保证父进程处于堵塞状态。引用手册页vfork(2)的文字，"非常不幸，Linux从过去复活了这个幽灵"。由于fork使用了写时复制技术，vfork速度方面不再有优势，因此应该避免使用它。
- clone产生线程，可以对父子进程之间的共享、复制进行精确控制

__1. 写时复制__

内核使用了写时复制（Copy-On-Write，COW）技术，以防止在fork执行时将父进程的所有数据复制到子进程。该技术利用了下述事实：进程通常只使用了其内存页的一小部分。 在调用fork时，内核通常对父进程的每个内存页，都为子进程创建一个相同的副本。这有两种很不好的负面效应。

(1) 使用了大量内存。                            
(2) 复制操作耗费很长时间。                                            

如果应用程序在进程复制之后使用exec立即加载新程序，那么负面效应会更严重。这实际上意味着，此前进行的复制操作是完全多余的，因为进程地址空间会重新初始化，复制的数据不再需要了。

内核可以使用技巧规避该问题。并不复制进程的整个地址空间，而是只复制其页表。这样就建立了虚拟地址空间和物理内存页之间的联系，我在第1章简要地讲过，具体过程请参见第3章和第4章。因此，fork之后父子进程的地址空间指向同样的物理内存页。                                           

当然，父子进程不能允许修改彼此的页， 这也是两个进程的页表对页标记了只读访问的原因，即使在普通环境下允许写入也是如此。                             

假如两个进程只能读取其内存页，那么二者之间的数据共享就不是问题，因为不会有修改。                                            

只要一个进程试图向复制的内存页写入，处理器会向内核报告访问错误（此类错误被称作缺页异常）。内核然后查看额外的内存管理数据结构（参见第4章），检查该页是否可以用读写模式访问，还是只能以只读模式访问。如果是后者，则必须向进程报告段错误。读者会在第4章看到，缺页异常处理程序的实际实现要复杂得多，因为还必须考虑其他方面的问题，例如换出的页。                                                    

如果页表项将一页标记为"只读"，但通常情况下该页应该是可写的，内核可根据此条件来判断该页实际上是COW页。因此内核会创建该页专用于当前进程的副本，当然也可以用于写操作。直至第4章我们才会讨论复制操作的实现方式，因为这需要内存管理方面广泛的背景知识。                                                

COW机制使得内核可以尽可能延迟内存页的复制，更重要的是，在很多情况下不需要复制。这节省了大量时间                  

__2. 执行系统调用__

fork、vfork和clone系统调用的入口点分别是sys_fork、sys_vfork和sys_clone函数。其定义依赖于具体的体系结构，因为在用户空间和内核空间之间传递参数的方法因体系结构而异（更多细节请参见第13章）。上述函数的任务是从处理器寄存器中提取由用户空间提供的信息，调用体系结构无关的do_fork函数，后者负责进程复制。该函数的原型如下：

	kernel/fork.c  
	long do_fork(unsigned long clone_flags,  
		    unsigned long stack_start,  
			struct pt_regs *regs,  
			unsigned long stack_size,  
			int __user *parent_tidptr,  
			int __user *child_tidptr)
			
clone_flags是一个标志集合，用来指定控制复制过程的一些属性。最低字节指定了在子进程终止时被发给父进程的信号号码。其余的高位字节保存了各种常数，下文会分别讨论。

start_stack是用户状态下栈的起始地址。             

regs是一个指向寄存器集合的指针，其中以原始形式保存了调用参数。该参数使用的数据类型是特定于体系结构的struct pt_regs，其中按照系统调用执行时寄存器在内核栈上的存储顺序，保存了所有的寄存器（更详细的信息，请参考附录A）。            

stack_size是用户状态下栈的大小。该参数通常是不必要的，设置为0。               

parent_tidptr和child_tidptr是指向用户空间中地址的两个指针，分别指向父子进程的TID。NPTL（Native Posix Threads Library）库的线程实现需要这两个参数。我将在下文讨论其语义    

不同的fork变体，主要是通过标志集合区分。在大多数体系结构上， 典型的fork调用的实现方式与IA-32处理器相同。

	arch/x86/kernel/process_32.c  
	asmlinkage int sys_fork(struct pt_regs regs)  
	{  
		return do_fork(SIGCHLD, regs.esp, &regs, 0, NULL, NULL);  
	} 

唯一使用的标志是SIGCHLD。这意味着在子进程终止后发送SIGCHLD信号通知父进程。最初，父子进程的栈地址相同（起始地址保存在IA-32系统的esp寄存器中）。但如果操作栈地址并写入数据，则COW机制会为每个进程分别创建一个栈副本。

如果do_fork成功，则新建进程的PID作为系统调用的结果返回，否则返回错误码（负值）。

sys_vfork的实现与sys_fork只是略微不同，前者使用了额外的标志（CLONE_VFORK和CLONE_ VM，其语义下文讨论）。

sys_clone的实现方式与上述调用相似，差别在于do_fork如下调用:                  

	    arch/x86/kernel/process_32.c  
	    asmlinkage int sys_clone(struct pt_regs regs)  
	    {  
		    unsigned long clone_flags;  
		    unsigned long newsp;  
		    int __user *parent_tidptr, *child_tidptr;  
	    
		    clone_flags = regs.ebx;  
		    newsp = regs.ecx;  
		    parent_tidptr = (int __user *)regs.edx;  
		    child_tidptr = (int __user *)regs.edi;  
		    if (!newsp)  
			    newsp = regs.esp;  
		    return do_fork(clone_flags, newsp, 
	    &regs, 0, parent_tidptr, child_tidptr);  
	    } 


                



