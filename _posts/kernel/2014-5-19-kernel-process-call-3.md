---
layout: post
title: "Linux 进程管理和调度——内核线程"
date: 2014-5-19 19:34
category: "学习"
tags: ["Kernel"]

---

###内核线程
内核线程是直接由内核本身启动的进程。内核线程实际上是将内核函数委托给独立的进程，与系统中其他进程"并行"执行（实际上，也并行于内核自身的执行）。 内核线程经常称之为（内核）守护进程。它们用于执行下列任务。            

- 周期性地将修改的内存页与页来源块设备同步（例如，使用mmap的文件映射）。   
- 如果内存页很少使用，则写入交换区。    
- 管理延时动作（deferred action）。
- 实现文件系统的事务日志。
- 基本上，有两种类型的内核线程。
- 类型1：线程启动后一直等待，直至内核请求线程执行某一特定操作。
- 类型2：线程启动后按周期性间隔运行，检测特定资源的使用，在用量超出或低于预置的限制值时采取行动。内核使用这类线程用于连续监测任务。
 
 内核线程与用户线程的相同点是：

- 都由do_fork()创建，每个线程都有独立的task_struct和内核栈；
- 都参与调度，内核线程也有优先级，会被调度器平等地换入换出。

不同之处在于：

- 内核线程只工作在内核态中；而用户线程则既可以运行在内核态，也可以运行在用户态；   
- 内核线程没有用户空间，所以对于一个内核线程来说，它的0~3G的内存空间是空白的，它的current->mm是空的，与内核使用同一张页表；而用户线程则可以看到完整的0~4G内存空间。

在Linux内核启动的最后阶段，系统会创建两个内核线程，一个是init，一个是kthreadd。其中init线程的作用是运行文件系统上的一系列"init"脚本，并启动shell进程，所以init线程称得上是系统中所有用户进程的祖先，它的pid是1。kthreadd线程是内核的守护线程，在内核正常工作时，它永远不退出，是一个死循环，它的pid是2。


内核初始化工作的最后一部分是在函数rest_init()中完成的。在这个函数中，主要做了4件事情，分别是：创建init线程，创建kthreadd线程，执行schedule()开始调度，执行cpu_idle()让CPU进入idle状态。经过简化的代码如下：

<pre class="prettyprint" id="c">

 static noinline void __init_refok rest_init(void)
  __releases(kernel_lock)
 {
  kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
  pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
  schedule();
  cpu_idle();
 }
 
 </pre>
 
      <asm-arch/processor.h>   
      int kernel_thread(int (*fn)(void *), void * 
      arg, unsigned long flags)  
 
 kernel_thread的第一个任务是构建一个pt_regs实例，对其中的寄存器指定适当的值，这与普通的fork系统调用类似。接下来调用我们熟悉的do_fork函数。
 
 
	p = do_fork(flags | CLONE_VM | CLONE_UNTRACED, 
	0, &regs, 0, NULL, NULL);  
	

参数int (*fn) (void *) 是一个函数指针，即第一个参数是一个函数，此函数是此进程执行时执行的函数，此函数返回值为int型，参数是一个void　指针。

参数 void * arg是一个void 型指针，是传递给第一个参数所代表函数的参数，即子进程执行时函数的参数。

参数 flags 是创建新进程的标志位，在内核文件对此有定义，可能取值如下所列：

<pre class="prettyprint" id="c">
/include/sched.h:

#define CSIGNAL  0x000000ff     /*进程退出信号*/

#define CLONE_VM     0x00000100    /*进程间共享虚拟区 */

#define CLONE_FS      0x00000200    /*进程间共享文件系统信息*/

#define CLONE_FILES 0x00000400    /*进程间共享文件*/

#define CLONE_SIGHAND  0x00000800    /*共享信号处理函数及阻塞信号*/

#define CLONE_PTRACE 0x00002000         /*持续追踪子进程* /

#define CLONE_VFORK  0x00004000        /*子进程内存空间释放时唤醒父进程 */

#define CLONE_PARENT    0x00008000    /* 克隆进程共用同一父进程*/

#define CLONE_THREAD   0x00010000    /*相同的进程组*/

#define CLONE_NEWNS     0x00020000    /*新命名空间组*/

#define CLONE_SYSVSEM 0x00040000    /*共享系统的V SEM_UNDO变量*/

#define CLONE_SETTLS     0x00080000    /*为子进程创建新的TLS值*/

#define CLONE_PARENT_SETTID    0x00100000    /*设置父进程的TID值 */

#define CLONE_CHILD_CLEARTID  0x00200000    /*清除子进程的TID值*/

#define CLONE_DETACHED             0x00400000    /*此标志位没有被使用 */

#define CLONE_UNTRACED             0x00800000    /*进程禁止追踪*/

#define CLONE_CHILD_SETTID       0x01000000    /*设置子进程的TID值*/

#define CLONE_STOPPED         0x02000000    /*从停止状态启动*/

#define CLONE_NEWUTS          0x04000000    /*新uts命名组*/

#define CLONE_NEWIPC           0x08000000    /*新ipcs*/

#define CLONE_NEWUSER              0x10000000    /*新用户命名空间*/

#define CLONE_NEWPID           0x20000000    /*新pid命名空间*/

#define CLONE_NEWNET          0x40000000    /*新网络命名空间*/

#define CLONE_IO              0x80000000          /*克隆输入输出上下文内容 */

 

/*
* List of flags we want to share for kernel threads,
* if only because they are not used by them anyway.
*/
#define CLONE_KERNEL    (CLONE_FS | CLONE_FILES | CLONE_SIGHAND)

此函数返回一个整形变量，此值为新创建的进程的进程号。

 </pre>
 
该函数返回整型数，如果返回的值小于0则表示内核线程创建失败，否则创建成功。

由该函数创建的进程不需要在模块清除时注销，可能运行过就自动销毁了
 
因为内核线程是由内核自身生成的，应该注意下面两个特别之处。        

(1) 它们在CPU的管态（supervisor mode）执行，而不是用户状态（参见第1章）。   

(2) 它们只可以访问虚拟地址空间的内核部分（高于TASK_SIZE的所有地址），但不能访问用户空间。        

回想上文的内容，可知task_struct中包含了指向mm_structs的两个指针     
      sched.h>   
      struct task_struct {   
      ...   
	      struct mm_struct *mm, *active_mm;  
      ...  
      } 


创建内核线程更现代的方法是辅助函数kthread_create

	kernel/kthread.c   
	struct task_struct *kthread_create(int (*threadfn)(void *data),  
					    void *data,  
					    const char namefmt[],  
					    ...) 
 
 
threadfn：函数指针，指向内核线程所执行的函数                   

data：不定类型的指针，指向内核线程所需要的参数                 

namefmt：内核线程名字              

…：类似于printf参数              

例如内核线程的名字带有不确定数字，可以像printf函数一样将数字写进内核线程名字。                 

####kthread_create

	struct task_struct *kthread_create(int (*threadfn)(void *data),

				      void *data,

				      const char namefmt[], ...)

	      __attribute__((format(printf, 3, 4)));

该函数返回创建的内核线程的进程描述符。

各参数含义：

threadfn：函数指针，指向内核线程所执行的函数

data：不定类型的指针，指向内核线程所需要的参数

namefmt：内核线程名字

…：类似于printf参数

例如内核线程的名字带有不确定数字，可以像printf函数一样将数字写进内核线程名字。

 

这个函数创建的内核线程不能立即执行，需要调用wake_up_process()函数来使线程执行，为此定义了宏kthread_run，他是kthread_create()函数和wake_up_process()的封装。

例子：
<pre class="prettyprint" id="c">

static struct task_struct *test_task;
static __init int test_init_module(void)
{

       int err;

       int num = 1;

       test_task = kthread_create(test_thread,NULL,”test_task-%d”, num);

       if(IS_ERR(test_task))

{

       printk(KERN_NOTICE “create new kernel thread failed\n”);

       err = PTR_ERR(test_task);

       test_task = NULL;

       return err;

}

wake_up_process(test_task);

return 0;

}

</pre>
模块退出时，需要结束所创建线程的运行，使用下面的函数：

      int kthread_stop(struct task_struct *k);
      int kthread_should_stop(void);

注意：在调用kthread_stop函数时，线程函数不能已经运行结束。否则，kthread_stop函数会一直进行等待。
为了避免这种情况，需要确保线程没有退出，其方法如代码中所示：
<pre class="prettyprint" id="c">
thread_func()
{

    // do your work here
    // wait to exit
    while(!thread_should_stop())
    {
           wait();
    }
}
exit_code()
{
     kthread_stop(_task);   //发信号给task，通知其可以退出了
}
</pre>

这种退出机制很温和，一切尽在thread_func()的掌控之中，线程在退出时可以从容地释放资源，而不是莫名其妙地被人“暗杀”。


例子代码：
<pre class="prettyprint" id="c">
static void test_cleanup_module(void)
{
       if(test_task)
{
       kthread_stop(test_task);
       test_task = NULL;
}
}

module_init(test_init_module);
module_exit(test_cleanup_module);

</pre>
 

关于创建的新线程的线程函数，该函数是要来完成所需要的业务逻辑工作的，其一般框架如下：
<pre class="prettyprint" id="c">
int threadfunc(void *data)

{

       ……..

       while(1)

{

       set_current_state(TASK_UNINTERRUPTIBLE);

       //wait_event_interruptible(wq,condition)，或wait_for-completion(c)也可以

       if(kthread_should_stop())

              break;

       if(condition)//真

       {

              ………//业务处理

}

else

{

       //让出CPU，并在指定的时间内重新调度

       schedule_timeout(HZ);//延时1秒，重新调度执行

}
}
}
</pre>

另外，有该函数创建的内核线程可以指定在不同的CPU核上运行（如果是多核的话），这个可以通过下面的函数实现：

void kthread_bind(struct task_struct *k, unsigned int cpu);

k：创建的内核线程的进程描述符

cpu：CPU编号


内核线程会出现在系统进程列表中，但在ps的输出中由方括号包围，以便与普通进程区分。
 
	wolfgang@meitner> ps fax   
	PID TTY STAT TIME COMMAND  
	    2?  S<  0:00 [kthreadd]  
	    3?  S<  0:00 _ [migration/0]  
	    4?  S<  0:00 _ [ksoftirqd/0]  
	    5?  S<  0:00 _ [migration/1]  
	    6?  S<  0:00 _ [ksoftirqd/1]  
	...  
	    52?     S<  0:00 _ [kblockd/3]  
	    55?     S<  0:00 _ [kacpid]  
	    56?     S<  0:00 _ [kacpi_notify]  
	... 
 
 如果内核线程绑定到特定的CPU，CPU的编号在斜线后给出。
 
 

