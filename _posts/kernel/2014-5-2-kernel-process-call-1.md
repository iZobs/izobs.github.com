---
layout: post
title: "Linux 进程管理和调度——进程的表示"
date: 2014-5-2 9:34
category: "学习"
tags: ["Kernel"]

---

###进程优先级

进程可以分为实时进程和非实时进程：

- 硬实时进程有严格的时间限制，任务在指定的时间内完成。硬实时进程的关键特征时，它们必须在可保证的时间内得到处理。请注意，这并不意味所要求的时间特别短，而是系统给了固定长度的时间。linux不支持硬实时处理，至少在主流的内核中不支持。但有一修改版本如RTlinux，Xenomai，RATI则可以。
- 软实时进程是硬实时进程的一种弱化。尽管仍然需要快速处理，但会稍晚一点。软实时的一个例子是对CD的写入，CD的写入进程的数据必须保持某一速率，但如果系统负荷太高，数据流可能会中断，这可能导致CD不能用。
- 大多数进程是没有特定时间约束的普通进程，但仍然可以根据重要性来分配来分配优先级。

###进程的生命周期
进程可能有下面的几种状态：

- 运行：该进程此刻正在执行
- 等待：进程能够运行，但没有得到许可，因为CPU分配给另一个进程。调度器可以在下一次任务切换时选择该进程。
- 睡眠：进程正在睡眠无法进行，因为它在等待一个外部事件。调度器无法在下一次任务切换时选择该进程

![process.png](/picture/process.png "process.png")

1.对于一个排队的可运行进程，他已经准备好了，但没有运行，因为CPU分配给另一个进程。该进程的状态为“等待”。在调度器授予CPU时间之前，进程会一直保持该状态。在分配CPU时间后，其状态改变为“运行”（路径4）
2.在调度器决定回收资源时，过程状态从“运行”改变为“等待”（路径2），循环重新开始。实际上根据是否可以被信号中断，有两种睡眠状态。现在这种差别还不重要，但在更仔细地考察具体实现时，其差别就相对重要了。
3.如果进程必须等待时间，则其状态“运行”改变为“睡眠”（路径1）.但进程状态无法直接从“睡眠”转到“运行”，所以要先转到“等待”(路径3)，然后再回到循环中。
    
	      僵尸进程：这样的进程已经killed，不过仍然已某种方式活着。实际上，说这个进程死了，是因为其资源已经释放，因此他们无法也决不会再次运行。说他们活着是因为进程中仍然有对应的表项。                            
	      产生的原因：其原因在于UNIX操作系统下进程创建和销毁方式。在这两种事件发生时，程序将终止运行。第一，程序必须由另一个进程或一个用户杀死；进程的父进程终止时必须调用或已经调用wait4系统调用。这相当于向内核证实父进程已经确认子进程已经终结。只有在第一个条件发生（程序终止）而第二个条件不成立的情况下(wait4)，才会出现“僵尸”状态。在进程终止后，其数据尚未从进程表删除前，进程是暂时处于“僵尸”状态。从ps或top的输出可以看到僵尸进程。因为残留在内核中占的空间很少，所以这几乎不成问题。
	  
	      
	      
	      
###进程的表示
linux内核涉及进程和程序的所有算法都围绕task_struct(inlude/sched.h)。task_struct包含很多成员，将进程与各个内核子系统联系起来。

<pre class="prettyprint" id="c">

struct task_struct {
	volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	void *stack;
	atomic_t usage;
	unsigned int flags;	/* per process flags, defined below */
	unsigned int ptrace;

#ifdef CONFIG_SMP
	struct llist_node wake_entry;
	int on_cpu;
#endif
	int on_rq;

	int prio, static_prio, normal_prio;
	unsigned int rt_priority;
	const struct sched_class *sched_class;
	struct sched_entity se;
	struct sched_rt_entity rt;
#ifdef CONFIG_CGROUP_SCHED
	struct task_group *sched_task_group;
#endif

#ifdef CONFIG_PREEMPT_NOTIFIERS
	/* list of struct preempt_notifier: */
	struct hlist_head preempt_notifiers;
#endif

	/*
	 * fpu_counter contains the number of consecutive context switches
	 * that the FPU is used. If this is over a threshold, the lazy fpu
	 * saving becomes unlazy to save the trap. This is an unsigned char
	 * so that after 256 times the counter wraps and the behavior turns
	 * lazy again; this to deal with bursty apps that only use FPU for
	 * a short time
	 */
	unsigned char fpu_counter;
#ifdef CONFIG_BLK_DEV_IO_TRACE
	unsigned int btrace_seq;
#endif

	unsigned int policy;
	int nr_cpus_allowed;
	cpumask_t cpus_allowed;

#ifdef CONFIG_PREEMPT_RCU
	int rcu_read_lock_nesting;
	char rcu_read_unlock_special;
	struct list_head rcu_node_entry;
#endif /* #ifdef CONFIG_PREEMPT_RCU */
#ifdef CONFIG_TREE_PREEMPT_RCU
	struct rcu_node *rcu_blocked_node;
#endif /* #ifdef CONFIG_TREE_PREEMPT_RCU */
#ifdef CONFIG_RCU_BOOST
	struct rt_mutex *rcu_boost_mutex;
#endif /* #ifdef CONFIG_RCU_BOOST */

#if defined(CONFIG_SCHEDSTATS) || defined(CONFIG_TASK_DELAY_ACCT)
	struct sched_info sched_info;
#endif

	struct list_head tasks;
#ifdef CONFIG_SMP
	struct plist_node pushable_tasks;
#endif

	struct mm_struct *mm, *active_mm;
#ifdef CONFIG_COMPAT_BRK
	unsigned brk_randomized:1;
#endif
#if defined(SPLIT_RSS_COUNTING)
	struct task_rss_stat	rss_stat;
#endif
/* task state */
	int exit_state;
	int exit_code, exit_signal;
	int pdeath_signal;  /*  The signal sent when the parent dies  */
	unsigned int jobctl;	/* JOBCTL_*, siglock protected */
	/* ??? */
	unsigned int personality;
	unsigned did_exec:1;
	unsigned in_execve:1;	/* Tell the LSMs that the process is doing an
				 * execve */
	unsigned in_iowait:1;

	/* task may not gain privileges */
	unsigned no_new_privs:1;

	/* Revert to default priority/policy when forking */
	unsigned sched_reset_on_fork:1;
	unsigned sched_contributes_to_load:1;

	pid_t pid;
	pid_t tgid;

#ifdef CONFIG_CC_STACKPROTECTOR
	/* Canary value for the -fstack-protector gcc feature */
	unsigned long stack_canary;
#endif
	/*
	 * pointers to (original) parent process, youngest child, younger sibling,
	 * older sibling, respectively.  (p->father can be replaced with
	 * p->real_parent->pid)
	 */
	struct task_struct __rcu *real_parent; /* real parent process */
	struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */
	/*
	 * children/sibling forms the list of my natural children
	 */
	struct list_head children;	/* list of my children */
	struct list_head sibling;	/* linkage in my parent's children list */
	struct task_struct *group_leader;	/* threadgroup leader */

	/*
	 * ptraced is the list of tasks this task is using ptrace on.
	 * This includes both natural children and PTRACE_ATTACH targets.
	 * p->ptrace_entry is p's link on the p->parent->ptraced list.
	 */
	struct list_head ptraced;
	struct list_head ptrace_entry;

	/* PID/PID hash table linkage. */
	struct pid_link pids[PIDTYPE_MAX];
	struct list_head thread_group;

	struct completion *vfork_done;		/* for vfork() */
	int __user *set_child_tid;		/* CLONE_CHILD_SETTID */
	int __user *clear_child_tid;		/* CLONE_CHILD_CLEARTID */

	cputime_t utime, stime, utimescaled, stimescaled;
	cputime_t gtime;
#ifndef CONFIG_VIRT_CPU_ACCOUNTING
	struct cputime prev_cputime;
#endif
	unsigned long nvcsw, nivcsw; /* context switch counts */
	struct timespec start_time; 		/* monotonic time */
	struct timespec real_start_time;	/* boot based time */
/* mm fault and swap info: this can arguably be seen as either mm-specific or thread-specific */
	unsigned long min_flt, maj_flt;

	struct task_cputime cputime_expires;
	struct list_head cpu_timers[3];

/* process credentials */
	const struct cred __rcu *real_cred; /* objective and real subjective task
					 * credentials (COW) */
	const struct cred __rcu *cred;	/* effective (overridable) subjective task
					 * credentials (COW) */
	char comm[TASK_COMM_LEN]; /* executable name excluding path
				     - access with [gs]et_task_comm (which lock
				       it with task_lock())
				     - initialized normally by setup_new_exec */
/* file system info */
	int link_count, total_link_count;
#ifdef CONFIG_SYSVIPC
/* ipc stuff */
	struct sysv_sem sysvsem;
#endif
#ifdef CONFIG_DETECT_HUNG_TASK
/* hung task detection */
	unsigned long last_switch_count;
#endif
/* CPU-specific state of this task */
	struct thread_struct thread;
/* filesystem information */
	struct fs_struct *fs;
/* open file information */
	struct files_struct *files;
/* namespaces */
	struct nsproxy *nsproxy;
/* signal handlers */
	struct signal_struct *signal;
	struct sighand_struct *sighand;

	sigset_t blocked, real_blocked;
	sigset_t saved_sigmask;	/* restored if set_restore_sigmask() was used */
	struct sigpending pending;

	unsigned long sas_ss_sp;
	size_t sas_ss_size;
	int (*notifier)(void *priv);
	void *notifier_data;
	sigset_t *notifier_mask;
	struct callback_head *task_works;

	struct audit_context *audit_context;
#ifdef CONFIG_AUDITSYSCALL
	kuid_t loginuid;
	unsigned int sessionid;
#endif
	struct seccomp seccomp;

/* Thread group tracking */
   	u32 parent_exec_id;
   	u32 self_exec_id;
/* Protection of (de-)allocation: mm, files, fs, tty, keyrings, mems_allowed,
 * mempolicy */
	spinlock_t alloc_lock;

	/* Protection of the PI data structures: */
	raw_spinlock_t pi_lock;

#ifdef CONFIG_RT_MUTEXES
	/* PI waiters blocked on a rt_mutex held by this task */
	struct plist_head pi_waiters;
	/* Deadlock detection and priority inheritance handling */
	struct rt_mutex_waiter *pi_blocked_on;
#endif

#ifdef CONFIG_DEBUG_MUTEXES
	/* mutex deadlock detection */
	struct mutex_waiter *blocked_on;
#endif
#ifdef CONFIG_TRACE_IRQFLAGS
	unsigned int irq_events;
	unsigned long hardirq_enable_ip;
	unsigned long hardirq_disable_ip;
	unsigned int hardirq_enable_event;
	unsigned int hardirq_disable_event;
	int hardirqs_enabled;
	int hardirq_context;
	unsigned long softirq_disable_ip;
	unsigned long softirq_enable_ip;
	unsigned int softirq_disable_event;
	unsigned int softirq_enable_event;
	int softirqs_enabled;
	int softirq_context;
#endif
#ifdef CONFIG_LOCKDEP
# define MAX_LOCK_DEPTH 48UL
	u64 curr_chain_key;
	int lockdep_depth;
	unsigned int lockdep_recursion;
	struct held_lock held_locks[MAX_LOCK_DEPTH];
	gfp_t lockdep_reclaim_gfp;
#endif

/* journalling filesystem info */
	void *journal_info;

/* stacked block device info */
	struct bio_list *bio_list;

#ifdef CONFIG_BLOCK
/* stack plugging */
	struct blk_plug *plug;
#endif

/* VM state */
	struct reclaim_state *reclaim_state;

	struct backing_dev_info *backing_dev_info;

	struct io_context *io_context;

	unsigned long ptrace_message;
	siginfo_t *last_siginfo; /* For ptrace use.  */
	struct task_io_accounting ioac;
#if defined(CONFIG_TASK_XACCT)
	u64 acct_rss_mem1;	/* accumulated rss usage */
	u64 acct_vm_mem1;	/* accumulated virtual memory usage */
	cputime_t acct_timexpd;	/* stime + utime since last update */
#endif
#ifdef CONFIG_CPUSETS
	nodemask_t mems_allowed;	/* Protected by alloc_lock */
	seqcount_t mems_allowed_seq;	/* Seqence no to catch updates */
	int cpuset_mem_spread_rotor;
	int cpuset_slab_spread_rotor;
#endif
#ifdef CONFIG_CGROUPS
	/* Control Group info protected by css_set_lock */
	struct css_set __rcu *cgroups;
	/* cg_list protected by css_set_lock and tsk->alloc_lock */
	struct list_head cg_list;
#endif
#ifdef CONFIG_FUTEX
	struct robust_list_head __user *robust_list;
#ifdef CONFIG_COMPAT
	struct compat_robust_list_head __user *compat_robust_list;
#endif
	struct list_head pi_state_list;
	struct futex_pi_state *pi_state_cache;
#endif
#ifdef CONFIG_PERF_EVENTS
	struct perf_event_context *perf_event_ctxp[perf_nr_task_contexts];
	struct mutex perf_event_mutex;
	struct list_head perf_event_list;
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *mempolicy;	/* Protected by alloc_lock */
	short il_next;
	short pref_node_fork;
#endif
#ifdef CONFIG_NUMA_BALANCING
	int numa_scan_seq;
	int numa_migrate_seq;
	unsigned int numa_scan_period;
	u64 node_stamp;			/* migration stamp  */
	struct callback_head numa_work;
#endif /* CONFIG_NUMA_BALANCING */

	struct rcu_head rcu;

	/*
	 * cache last used pipe for splice
	 */
	struct pipe_inode_info *splice_pipe;

	struct page_frag task_frag;

#ifdef	CONFIG_TASK_DELAY_ACCT
	struct task_delay_info *delays;
#endif
#ifdef CONFIG_FAULT_INJECTION
	int make_it_fail;
#endif
	/*
	 * when (nr_dirtied >= nr_dirtied_pause), it's time to call
	 * balance_dirty_pages() for some dirty throttling pause
	 */
	int nr_dirtied;
	int nr_dirtied_pause;
	unsigned long dirty_paused_when; /* start of a write-and-pause period */

#ifdef CONFIG_LATENCYTOP
	int latency_record_count;
	struct latency_record latency_record[LT_SAVECOUNT];
#endif
	/*
	 * time slack values; these are used to round up poll() and
	 * select() etc timeout values. These are in nanoseconds.
	 */
	unsigned long timer_slack_ns;
	unsigned long default_timer_slack_ns;

#ifdef CONFIG_FUNCTION_GRAPH_TRACER
	/* Index of current stored address in ret_stack */
	int curr_ret_stack;
	/* Stack of return addresses for return function tracing */
	struct ftrace_ret_stack	*ret_stack;
	/* time stamp for last schedule */
	unsigned long long ftrace_timestamp;
	/*
	 * Number of functions that haven't been traced
	 * because of depth overrun.
	 */
	atomic_t trace_overrun;
	/* Pause for the tracing */
	atomic_t tracing_graph_pause;
#endif
#ifdef CONFIG_TRACING
	/* state flags for use by tracers */
	unsigned long trace;
	/* bitmask and counter of trace recursion */
	unsigned long trace_recursion;
#endif /* CONFIG_TRACING */
#ifdef CONFIG_MEMCG /* memcg uses this to do batch job */
	struct memcg_batch_info {
		int do_batch;	/* incremented when batch uncharge started */
		struct mem_cgroup *memcg; /* target memcg of uncharge */
		unsigned long nr_pages;	/* uncharged usage */
		unsigned long memsw_nr_pages; /* uncharged mem+swap usage */
	} memcg_batch;
	unsigned int memcg_kmem_skip_account;
#endif
#ifdef CONFIG_HAVE_HW_BREAKPOINT
	atomic_t ptrace_bp_refcnt;
#endif
#ifdef CONFIG_UPROBES
	struct uprobe_task *utask;
#endif
};

</pre>


要弄清楚该结构体的信息数量很困难。但该结构体可以分为下面各个部分：

- 状态和执行信息,如待解决信号，使用的二进制格式，进程ID号（pid），到父进程及其他有关进程的指针，优先级和程序执行有关的时间信息(CPU时间)
- 有关已经分配的虚拟内存的信息
- 进程身份凭证，如用户ID，组ID以及其权限
- 使用的文件包含程序代码的二进制文件，以及进程所处理的所有文件系统信息，这些都必须保存下来
- 线程信息记录该进程特定于CPU的运行时间数据
- 在与其他应用程序协作时所需的进程间通信有关的信息
- 该进程所用的信号处理程序，用于响应到来的信号



###task_struct中重要的成员

####1.进程状态(volatile long state)

state指定了进程的当前状态，可使用下列的值，定义于<sched.h>

<pre class="prettyprint" id="c">

/*
 * Task state bitmask. NOTE! These bits are also
 * encoded in fs/proc/array.c: get_task_state().
 *
 * We have two separate sets of flags: task->state
 * is about runnability, while task->exit_state are
 * about the task exiting. Confusing, but this way
 * modifying one set can't modify the other one by
 * mistake.
 */
#define TASK_RUNNING		0
#define TASK_INTERRUPTIBLE	1
#define TASK_UNINTERRUPTIBLE	2
#define __TASK_STOPPED		4
#define __TASK_TRACED		8
/* in tsk->exit_state */
#define EXIT_ZOMBIE		16
#define EXIT_DEAD		32
/* in tsk->state again */
#define TASK_DEAD		64
#define TASK_WAKEKILL		128
#define TASK_WAKING		256
#define TASK_STATE_MAX		512

#define TASK_STATE_TO_CHAR_STR "RSDTtZXxKW"

extern char ___assert_task_state[1 - 2*!!(
		sizeof(TASK_STATE_TO_CHAR_STR)-1 != ilog2(TASK_STATE_MAX)+1)];

/* Convenience macros for the sake of set_task_state */
#define TASK_KILLABLE		(TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
#define TASK_STOPPED		(TASK_WAKEKILL | __TASK_STOPPED)
#define TASK_TRACED		(TASK_WAKEKILL | __TASK_TRACED)

/* Convenience macros for the sake of wake_up */
#define TASK_NORMAL		(TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)
#define TASK_ALL		(TASK_NORMAL | __TASK_STOPPED | __TASK_TRACED)

/* get_task_state() */
#define TASK_REPORT		(TASK_RUNNING | TASK_INTERRUPTIBLE | \
				 TASK_UNINTERRUPTIBLE | __TASK_STOPPED | \
				 __TASK_TRACED)


</pre>

+ TASK_RUNNING:意味着进程处于可运行状态。这并不意味着已经实际分配CPU。进程可能会一直等待到调度器选中它。该状态确保进程可以立即运行，而无需等待外部事件。
+ TASK_INTERRUPTIBLE：是针对等待某事件或其他资源的睡眠进程设置的。在内核发送信号给该进程表明事件已经发生时，进程状态改为TASK_RUNNING,它只要调度器选中该进程即可恢复执行。
+ TASK_UNINTERRUPTIBLE：用于因内核指示而停用的睡眠进程。它们不能由外部信号唤醒，只能由内核亲自唤醒。
+ TASK_STOPPED：进程停止运行。                          
+ TASK_TRACED：本来不是进程状态，用于从停止的进程中，将当前被调用的那些（ptrace机制）与常规的进程区分。

下面的宏可以用于tack_struct或exit_state.
+ EXIT_ZOMBIE：僵尸进程。qil

+ EXIT_DEAD:状态则指wait系统调用已经发出，而进程完全从系统删除之前的状态。只有多个线程对同一个进程发出wait调用时，该状态才由意义。


####2.命名空间                
 
 __(1)概述__                                    
 在计算机的世界中，“程序”是一个名词，是一堆代码的集合，如果它只是静静的躺在磁盘上，即使代码堆积如山也毫无意义，运行起来的程序才能如愿以偿，由此也给它起一个新的名字-进程。不管进程呆在内存，还是正在CPU上运行，标识它唯一身份的就是进程身份号PID（Process IDentifier）。                              
 命名空间提供了虚拟化的一种轻量级形式，使得我们可以从不同的方面来查看运行系统的全局属性。该机制类似于solaris中zone或FreeBSD中的jail。在虚拟化的系统中，一台物理计算机可以运行多个内核，可能是并行的多个的不同操作系统。而命名空间则只使用一个内核在一台物理计算机上运行，前述的所有全局资源都通过命名空间抽象起来。这使得可以将一组进程放置到容器中，各个容器彼此隔离。隔离可以使容器的成员与其他容器完全没有关系。但也可以通过允许容器进行一定的共享，来降低容器之间的分隔。目前Linux系统实现的命名空间子系统主要有UTS、IPC、MNT、PID以及NET网络子模块。                        
 __(2)实现__                                                        
命名空间的实现需要两个部分：每个子系统的命名空间结构，将此前所有的全局组件包装到命名空间中；将给定进程关联到所属各个命名空间的机制。子系统此前的全局属性现在封装在命名空间中，每个进程关联到一个选定的命名空间。每个可以感知命名空间的内核系统都必须提供一个数据结构，将所有通过命名空间形式提供的对象集中起来。这个结构体就是struct nsproxy.                        
 
`nsproxy.h`
<pre class="prettyprint" id="c">

struct mnt_namespace;
struct uts_namespace;
struct ipc_namespace;
struct pid_namespace;
struct fs_struct;

/*
 * A structure to contain pointers to all per-process
 * namespaces - fs (mount), uts, network, sysvipc, etc.
 *
 * 'count' is the number of tasks holding a reference.
 * The count for each namespace, then, will be the number
 * of nsproxies pointing to it, not the number of tasks.
 *
 * The nsproxy is shared by tasks which share all namespaces.
 * As soon as a single namespace is cloned or unshared, the
 * nsproxy is copied.
 */
struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;
	struct ipc_namespace *ipc_ns;
	struct mnt_namespace *mnt_ns;
	struct pid_namespace *pid_ns;
	struct net 	     *net_ns;
};

</pre>
 ![namespace.png](/picture/namespace.png "namespace.png")

这里只介绍两种namespace：UTS namespace和PID namespace，是因为这两种namespace比较有代表性。UTS namespace很简单，也没有树形或者很复杂的结构。

__UTS命名空间__                                
UTS命名空间几乎不需要特别的处理，因为它只需要简单量，没有层次组织。所有相关信息都汇集到下列结构的一个实例中
struct uts_namespace结构如下：                    

	struct uts_namespace {
		struct kref kref;
		struct new_utsname name;
		struct user_namespace *user_ns;
		unsigned int proc_inum;
	};
kref是一个嵌入的引用计数器，可用于跟踪内核中有多少地方使用了struct uts_namespace的实例（回想第1章，其中讲述了更多有关处理引用计数的一般框架信息）。uts_namespace所提供的属性信息本身包含在struct new_utsname中                          

	<utsname.h> 
	struct new_utsname {  
		char sysname[65];  
		char nodename[65];  
		char release[65];  
		char version[65];  
		char machine[65];  
		char domainname[65];  
	}; 
	
各个字符串分别存储了系统的名称（Linux...）、内核发布版本、机器名，等等。使用uname工具可以取得这些属性的当前值，也可以在/proc/sys/kernel/中看到.                    

初始设置保存在init_uts_ns中:                 

	init/version.c  
	struct uts_namespace init_uts_ns = {  
	...  
		.name = {  
			.sysname = UTS_SYSNAME,  
			.nodename = UTS_NODENAME,  
			.release = UTS_RELEASE,  
			.version = UTS_VERSION,  
			.machine = UTS_MACHINE,  
			.domainname = UTS_DOMAINNAME,  
		},  
	}; 

内核如何创建一个新的UTS命名空间呢？这属于copy_utsname函数的职责。在某个进程调用fork并通过CLONE_NEWUTS标志指定创建新的UTS命名空间时，则调用该函数。在这种情况下，会生成先前的uts_namespace实例的一份副本，当前进程的nsproxy实例内部的指针会指向新的副本。如此而已!由于在读取或设置UTS属性值时，内核会保证总是操作特定于当前进程的uts_namespace实例，在当前进程修改UTS属性不会反映到父进程，而父进程的修改也不会传播到子进程.                      



