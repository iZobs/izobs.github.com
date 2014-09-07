---
layout: post
title: "linux c-线程池"
date: 2014-5-20 21:10
category: "linux"
comments: false
tagline: "Supporting tagline"
tags: [linux,c]
---

##什么是线程池
什么是线程池？简单点说，线程池就是有一堆已经创建好了的线程，初始它们都处于空闲等待状态，当有新的任务需要处理的时候，就从这个池子里面取一个空闲等待的线程来处理该任务，当处理完成了就再次把该线程放回池中，以供后面的任务使用。当池子里的线程全都处理忙碌状态时，线程池中没有可用的空闲等待线程，此时，根据需要选择创建一个新的线程并置入池中，或者通知任务线程池忙,稍后再试。           

##为什么要使用线程池
我们说，线程的创建和销毁比之进程的创建和销毁是轻量级的，但是当我们的任务需要大量进行大量线程的创建和销毁操作时，这个消耗就会变成的相当大。比如，当你设计一个压力性能测试框架的时候，需要连续产生大量的并发操作，这个是时候，线程池就可以很好的帮上你的忙。线程池的好处就在于线程复用，一个任务处理完成后，当前线程可以直接处理下一个任务，而不是销毁后再创建，非常适用于连续产生大量并发任务的场合。

##线程池的原理
   线程池的任务就在于负责这些线程的创建，销毁和任务处理参数传递、唤醒和等待。          
- 创建若干线程，置入线程池
- 任务达到时，从线程池取空闲线程
- 取得了空闲线程，立即进行任务处理
- 否则新建一个线程，并置入线程池，执行3
- 如果创建失败或者线程池已满，根据设计策略选择返回错误或将任务置入处理队列，等待处理
- 销毁线程池

##线程池的设计
###数据结构的设计
__任务结构体__
{% highlight c++ %}

typedef struct tp_work_desc_s TpWorkDesc;  
typedef void (*process_job)(TpWorkDesc*job);  
struct tp_work_desc_s {  
         void *ret; //call in, that is arguments  
	     void *arg; //call out, that is return value  
}; 

{% endhighlight c++ %}
其中，TpWorkDesc是任务参数描述，arg是传递给任务的参数，ret则是任务处理完成后的返回值；

process_job函数是任务处理函数原型，每个任务处理函数都应该这样定义，然后将它作为参数传给线程池处理，线程池将会选择一个空闲线程通过调用该函数来进行任务处理；


**线程设计**

{% highlight c++ %}
typedef struct tp_thread_info_s TpThreadInfo;  
struct tp_thread_info_s {  
         pthread_t thread_id; //thread id num  
         TPBOOL is_busy; //thread status:true-busy;flase-idle  
         pthread_cond_t thread_cond;   //thread condition sem  
         pthread_mutex_t thread_lock;  
         process_job proc_fun;  
         TpWorkDesc* th_job;  
         TpThreadPool* tp_pool;  
};

{% endhighlight c++ %}

TpThreadInfo是对一个线程的描述。

thread_id是该线程的ID；

is_busy用于标识该线程是否正处理忙碌状态；

thread_cond用于任务处理时的唤醒和等待；

thread_lock，用于任务加锁，用于条件变量等待加锁；

proc_fun是当前任务的回调函数地址；

th_job是任务的参数信息；

tp_pool是所在线程池的指针；

**线程池的设计**
{% highlight c++ %}

struct tp_thread_pool_s {
	unsigned min_th_num; //min thread number in the pool
	unsigned cur_th_num; //current thread number in the pool
	unsigned max_th_num; //max thread number in the pool
	pthread_mutex_t tp_lock;
	pthread_cond_t tp_cond;
	pthread_mutex_t loop_lock;
	pthread_cond_t loop_cond;
	
	TpThreadInfo *thread_info;
	TSQueue *idle_q; //idle queue
	BOOL stop_flag; //whether stop the threading pool
	
	pthread_t manage_thread_id; //manage thread id num
	float busy_threshold; //
	unsigned manage_interval; //
};

{% endhighlight c++ %}
TpThreadPool是对线程池的描述。

min_th_num是线程池中至少存在的线程数，线程池初始化的过程中会创建min_th_num数量的线程；

cur_th_num是线程池当前存在的线程数量；

max_th_num则是线程池最多可以存在的线程数量；

tp_lock用于线程池管理时的互斥；

manage_thread_id是线程池的管理线程ID；

thread_info则是指向线程池数据，这里使用一个数组来存储线程池中线程的信息，该数组的大小为max_th_num；

idle_q是存储线程池空闲线程指针的队列，用于从线程池快速取得空闲线程；

stop_flag用于线程池的销毁，当stop_flag为FALSE时，表明当前线程池需要销毁，所有忙碌线程在处理完当前任务后会退出；

##算法设计
**线程创建**

{% highlight c++ %}

/**
 * user interface. creat thread pool.
 * para:
 * 	num: min thread number to be created in the pool
 * return:
 * 	thread pool struct instance be created successfully
 */
TpThreadPool *tp_create(unsigned min_num, unsigned max_num) {
	TpThreadPool *pTp;
	pTp = (TpThreadPool*) malloc(sizeof(TpThreadPool));

	memset(pTp, 0, sizeof(TpThreadPool));

	//init member var
	pTp->min_th_num = min_num;
	pTp->cur_th_num = min_num;
	pTp->max_th_num = max_num;
    /*初始化线程池里的一个mutex_lock，用于线程池管理的互斥*/
	pthread_mutex_init(&pTp->tp_lock, NULL);
	pthread_cond_init(&pTp->tp_cond, NULL);
    /*这个loop_lock用于*
	pthread_mutex_init(&pTp->loop_lock, NULL);
	pthread_cond_init(&pTp->loop_cond, NULL);

	//malloc mem for num thread info struct
	if (NULL != pTp->thread_info)
		free(pTp->thread_info);
	pTp->thread_info = (TpThreadInfo*) malloc(sizeof(TpThreadInfo) * pTp->max_th_num);
	memset(pTp->thread_info, 0, sizeof(TpThreadInfo) * pTp->max_th_num);

	return pTp;
}

{% endhighlight c++ %}
创建伊始，线程池线程容量大小上限为max_th_num，初始容量为min_th_num；
**线程的初始化**
{% highlight c++ %}
/**
 * member function reality. thread pool init function.
 * para:
 * 	pTp: thread pool struct instance ponter
 * return:
 * 	true: successful; false: failed
 */
int tp_init(TpThreadPool *pTp) {
	int i;
	int err;
	TpThreadInfo *pThi;

	//init_queue(&pTp->idle_q, NULL);
    /*创建一个队列，用于存放thread_info*/
	pTp->idle_q = ts_queue_create();
	pTp->stop_flag = FALSE;
	pTp->busy_threshold = BUSY_THRESHOLD;
	pTp->manage_interval = MANAGE_INTERVAL;

	//create work thread and init work thread info
	for (i = 0; i < pTp->min_th_num; i++) {
		pThi = pTp->thread_info + i; //一个线程对应于一个thread_info
		pThi->tp_pool = pTp;         //记录指针
		pThi->is_busy = FALSE;
		pthread_cond_init(&pThi->thread_cond, NULL);
		pthread_mutex_init(&pThi->thread_lock, NULL);
		pThi->proc_fun = NULL;
		pThi->arg = NULL;
		ts_queue_enq_data(pTp->idle_q, pThi);
        /*new一个线程，调用的函数为tp_work_thread,参数为pThi(thread_info)*/
		err = pthread_create(&pThi->thread_id, NULL, tp_work_thread, pThi);
		if (0 != err) {
			perror("tp_init: create work thread failed.");
			ts_queue_destroy(pTp->idle_q);
			return -1;
		}
	}

	//create manage thread
	err = pthread_create(&pTp->manage_thread_id, NULL, tp_manage_thread, pTp);
	if (0 != err) {//clear_queue(&pTp->idle_q);
		ts_queue_destroy(pTp->idle_q);
		fprintf(stderr, "tp_init: creat manage thread failed\n");
		return 0;
	}
	
	//wait for all threads are ready
	while(i++ < pTp->cur_th_num){
		pthread_mutex_lock(&pTp->tp_lock);
		pthread_cond_wait(&pTp->tp_cond, &pTp->tp_lock);
		pthread_mutex_unlock(&pTp->tp_lock);
	}
	DEBUG("All threads are ready now\n");
	return 0;
}


{% endhighlight c++ %}

初始线程池中线程数量为`min_th_num`，对这些线程一一进行初始化；
将这些初始化的空闲线程一一置入空闲队列；

创建管理线程，用于监控线程池的状态，并适当回收多余的线程资源；

**线程池的关闭和销毁**

{% highlight c++ %}
/**
 * member function reality. thread pool entirely close function.
 * para:
 * 	pTp: thread pool struct instance ponter
 * return:
 */
void tp_close(TpThreadPool *pTp, BOOL wait) {
	unsigned i;

	pTp->stop_flag = TRUE;
	if (wait) {
		DEBUG("current number of threads: %u", pTp->cur_th_num);
		for (i = 0; i < pTp->cur_th_num; i++) {
			pthread_cond_signal(&pTp->thread_info[i].thread_cond);
		}
		for (i = 0; i < pTp->cur_th_num; i++) {
			if(0 != pthread_join(pTp->thread_info[i].thread_id, NULL)){
				perror("pthread_join");
			}
			//DEBUG("join a thread success.\n");
			pthread_mutex_destroy(&pTp->thread_info[i].thread_lock);
			pthread_cond_destroy(&pTp->thread_info[i].thread_cond);
		}
	} else {
		//close work thread
		for (i = 0; i < pTp->cur_th_num; i++) {
			kill((pid_t)pTp->thread_info[i].thread_id, SIGKILL);
			pthread_mutex_destroy(&pTp->thread_info[i].thread_lock);
			pthread_cond_destroy(&pTp->thread_info[i].thread_cond);
		}
	}
	//close manage thread
	kill((pid_t)pTp->manage_thread_id, SIGKILL);
	pthread_mutex_destroy(&pTp->tp_lock);
	pthread_cond_destroy(&pTp->tp_cond);
	pthread_mutex_destroy(&pTp->loop_lock);
	pthread_cond_destroy(&pTp->loop_cond);

	//clear_queue(&pTp->idle_q);
	ts_queue_destroy(pTp->idle_q);
	//free thread struct
	free(pTp->thread_info);
	pTp->thread_info = NULL;
}

{% endhighlight c++ %}
线程池关闭的过程中，可以选择是否对正在处理的任务进行等待，如果是，则会唤醒所有任务，然后等待所有任务执行完成，然后返回；如果不是，则将立即杀死所有线程，然后返回，注意：这可能会导致任务的处理中断而产生错误！

{% highlight c++ %}
/**
 * member function reality. main interface opened.
 * after getting own worker and job, user may use the function to process the task.
 * para:
 * 	pTp: thread pool struct instance ponter
 *	worker: user task reality.
 *	job: user task para
 * return:
 */
int tp_process_job(TpThreadPool *pTp, process_job proc_fun, void *arg) {
	TpThreadInfo *pThi ;
	//fill pTp->thread_info's relative work key
    /*从队列中取出一个Thread_info*/
	pThi = (TpThreadInfo *) ts_queue_deq_data(pTp->idle_q);
	if(pThi){
		DEBUG("Fetch a thread from pool.\n");
		pThi->is_busy = TRUE;
		pThi->proc_fun = proc_fun;
		pThi->arg = arg;
		//let the thread to deal with this job
		DEBUG("wake up thread %u\n", pThi->thread_id);
		pthread_cond_signal(&pThi->thread_cond)}
	else{
		//if all current thread are busy, new thread is created here
		if(!(pThi = tp_add_thread(pTp, proc_fun, arg))){
			DEBUG("The thread pool is full, no more thread available.\n");
			return -1;
		}
		/* should I wait? */
		//pthread_mutex_lock(&pTp->tp_lock);
		//pthread_cond_wait(&pTp->tp_cond, &pTp->tp_lock);
		//pthread_mutex_unlock(&pTp->tp_lock);
		
		DEBUG("No more idle thread, a new thread is created.\n");
	}
	return 0;
}

{% endhighlight c++ %}
当一个新任务到达是，线程池首先会检查是否有可用的空闲线程，如果是，则采用才空闲线程进行任务处理并返回TRUE，如果不是，则尝试新建一个线程，并使用该线程对任务进行处理，如果失败则返回FALSE，说明线程池忙碌或者出错。

{% highlight c++ %}
/**
 * internal interface. real work thread.
 * @params:
 * 	arg: args for this method
 * @return:
 *	none
 */
static void *tp_work_thread(void *arg) {
	TpThreadInfo *pTinfo = (TpThreadInfo *) arg;
	TpThreadPool *pTp = pTinfo->tp_pool;

	//wake up waiting thread, notify it I am ready
	pthread_cond_signal(&pTp->tp_cond);
	while (!(pTp->stop_flag)) {
		//process
		if(pTinfo->proc_fun){
			DEBUG("thread %u is running\n", pTinfo->thread_id);
			pTinfo->proc_fun(pTinfo->arg);
			//thread state shoulde be set idle after work
			pTinfo->is_busy = FALSE;
			pTinfo->proc_fun = NULL;
			//I am idle now
			ts_queue_enq_data(pTp->idle_q, pTinfo);
		}
			
		//wait cond for processing real job.

		DEBUG("thread %u is waiting for a job\n", pTinfo->thread_id);
		pthread_mutex_lock(&pTinfo->thread_lock);
        /*等待process_job中发送一个send_signed来唤醒*/
		pthread_cond_wait(&pTinfo->thread_cond, &pTinfo->thread_lock);
		pthread_mutex_unlock(&pTinfo->thread_lock);
		DEBUG("thread %u end waiting for a job\n", pTinfo->thread_id);

		if(pTinfo->tp_pool->stop_flag){
			DEBUG("thread %u stop\n", pTinfo->thread_id);
			break;
		}
	}
	DEBUG("Job done, thread %u is idle now.\n", pTinfo->thread_id);
}


{% endhighlight c++ %}
上面这个函数是任务处理函数，该函数将始终处理等待唤醒状态，直到新任务到达或者线程销毁时被唤醒，然后调用任务处理回调函数对任务进行处理；当任务处理完成时，则将自己置入空闲队列中，以供下一个任务处理。

{% highlight c++ %}
/**
 * member function reality. add new thread into the pool and run immediately.
 * para:
 * 	pTp: thread pool struct instance ponter
 * 	proc_fun:
 * 	job:
 * return:
 * 	pointer of TpThreadInfo
 */
static TpThreadInfo *tp_add_thread(TpThreadPool *pTp, process_job proc_fun, void *arg) {
	int err;
	TpThreadInfo *new_thread;

	pthread_mutex_lock(&pTp->tp_lock);
	if (pTp->max_th_num <= pTp->cur_th_num){
		pthread_mutex_unlock(&pTp->tp_lock);
		return NULL;
	}

	//malloc new thread info struct
	new_thread = pTp->thread_info + pTp->cur_th_num; 
	pTp->cur_th_num++;
	pthread_mutex_unlock(&pTp->tp_lock);

	new_thread->tp_pool = pTp;
	//init new thread's cond & mutex
	pthread_cond_init(&new_thread->thread_cond, NULL);
	pthread_mutex_init(&new_thread->thread_lock, NULL);

	//init status is busy, only new process job will call this function
	new_thread->is_busy = TRUE;
	new_thread->proc_fun = proc_fun;
	new_thread->arg = arg;

	err = pthread_create(&new_thread->thread_id, NULL, tp_work_thread, new_thread);
	if (0 != err) {
		perror("tp_add_thread: pthread_create");
		free(new_thread);
		return NULL;
	}
	return new_thread;
}

{% endhighlight c++ %}
上面这个函数用于向线程池中添加新的线程，该函数将会在当线程池没有空闲线程可用时被调用。
函数将会新建一个线程，并设置自己的状态为busy（立即就要被用于执行任务）

##线程池的管理

线程池的管理主要是监控线程池的整体忙碌状态，当线程池大部分线程处于空闲状态时，管理线程将适当的销毁一定数量的空闲线程，以便减少线程池对系统资源的消耗。
这里设计认为，当空闲线程的数量超过线程池线程数量的1/2时，线程池总体处理空闲状态，可以适当销毁部分线程池的线程，以减少线程池对系统资源的开销。

**线程池状态计算**

{% highlight c++ %}
/**
 * member function reality. get current thread pool status:idle, normal, busy, .etc.
 * para:
 * 	pTp: thread pool struct instance ponter
 * return:
 * 	0: idle; 1: normal or busy(don't process)
 */
int tp_get_tp_status(TpThreadPool *pTp) {
	float busy_num = 0.0;
	int i;

	//get busy thread number
	busy_num = pTp->cur_th_num - ts_queue_count(pTp->idle_q);

	DEBUG("Current thread pool status, current num: %u, busy num: %u, idle num: %u\n", pTp->cur_th_num, (unsigned)busy_num, ts_queue_count(pTp->idle_q));
	if(busy_num / (pTp->cur_th_num) < pTp->busy_threshold)
		return 0;//idle status
	else
		return 1;//busy or normal status	
}

{% endhighlight c++ %}
**线程的销毁算法**
- 从空闲队列中dequeue一个空闲线程指针，该指针指向线程信息数组的某项，例如这里是p；
- 销毁该线程
- 把线程信息数组的最后一项拷贝至位置p
- 线程池数量减少一，即cur_th_num--

{% highlight c++ %}

/**
 * member function reality. delete idle thread in the pool.
 * only delete last idle thread in the pool.
 * para:
 * 	pTp: thread pool struct instance ponter
 * return:
 * 	true: successful; false: failed
 */
int tp_delete_thread(TpThreadPool *pTp) {
	unsigned idx;
	TpThreadInfo *pThi;
	TpThreadInfo tT;

	//current thread num can't < min thread num
	if (pTp->cur_th_num <= pTp->min_th_num)
		return -1;
	//all threads are busy
	pThi = (TpThreadInfo *) ts_queue_deq_data(pTp->idle_q);
	if(!pThi)
		return -1;
	
	//after deleting idle thread, current thread num -1
	pthread_mutex_lock(&pTp->tp_lock);
	pTp->cur_th_num--;
	/** swap this thread to the end, and free it! **/
	memcpy(&tT, pThi, sizeof(TpThreadInfo));
	memcpy(pThi, pTp->thread_info + pTp->cur_th_num, sizeof(TpThreadInfo));
	memcpy(pTp->thread_info + pTp->cur_th_num, &tT, sizeof(TpThreadInfo));
	pthread_mutex_unlock(&pTp->tp_lock);

	//kill the idle thread and free info struct
	kill((pid_t)tT.thread_id, SIGKILL);
	pthread_mutex_destroy(&tT.thread_lock);
	pthread_cond_destroy(&tT.thread_cond);

	return 0;
}

{% endhighlight c++ %}

##线程池的管理
线程池通过一个管理线程来进行监控，管理线程将会每隔一段时间对线程池的状态进行计算，根据线程池的状态适当的销毁部分线程，减少对系统资源的消耗。

{% highlight c++ %}
/**
 * internal interface. manage thread pool to delete idle thread.
 * para:
 * 	pthread: thread pool struct ponter
 * return:
 */
static void *tp_manage_thread(void *arg) {
	TpThreadPool *pTp = (TpThreadPool*) arg;//main thread pool struct instance

	//1?
	sleep(pTp->manage_interval);

	do {
		if (tp_get_tp_status(pTp) == 0) {
			do {
				if (!tp_delete_thread(pTp))
					break;
			} while (TRUE);
		}//end for if

		//1?
		sleep(pTp->manage_interval);
	} while (!pTp->stop_flag);
	return NULL;
}

{% endhighlight c++ %}

