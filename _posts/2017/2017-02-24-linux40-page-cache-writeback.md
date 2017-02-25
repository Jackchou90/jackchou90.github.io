---
title: linux4.0页缓存回写机制
layout: post
---

* content
{:toc}



# 综述

linux的写在page cache层面分为直接IO(O_DIRECT)和带缓存IO。
直接IO比较好理解，这里只讲带缓存IO。
带缓存IO的写在内核中分为两部分:

1）page cache 之前:  
sys_write => file->f_op->writer_iter => inode->i_fop->write_iter => 映射file为page cache中的pages，然后调用iov_iter接口
将用户空间的data拷贝到映射好的pages中去，标记dirty脏页，返回用户空间。

2) 内核中bdi writeback机制在以下时机将1)中的脏页flush到具体的存储media中。  
   a) page cache中总dirty page超过了一定的阈值.  
   b) page cache中的dirty page存活时间超过了一定的阈值。  
   c) 用户主动调用fsync sync等系统调用，手动flush page cache。  

在linux 4.0系统中，内核中的page cache回写机制本质上是由工作队列实现的，也即flusher线程实际上是工作队列对应的worker_pool中的worker对应的内核线程，具体到```ps -eLf```的表现形式是kworker:ux:y内核线程。


# 欲知页回写，先晓4.0工作队列机制

4.0内核中工作队列机制主要由 工作池(work_pool), 工作者(worker)，工作池中的具体工作项(work_struct), 工作队列(workqueue_struct)构成。

**工作池(work_pool):** 系统中每个逻辑CPU可能对应多个工作池，每个工作池是包含多个工作项的队列，并且该池中的每个工作项都来自于该池对应的workqueue的添加操作产生, 而该池对应的workqueue本身并不保存工作项，只是上层在对workqueue增加工作项时，底层做了转化，将工作项添加到workqueue对应的工作池的队列中。工作池和workqueue的关联关系由pool_workqueue结构体保存。所以，workqueue实际上是不保存work的，每次调用类似queue_work(workqueue, work);实际上内核先通过遍历系统中每个worker指向的pool_workqueue结构体中的workqueue,从而匹配到queue_work函数参数中workqueue关联的那个worker，从而得到该worker的pool_workqueue，进而得到该pool_workqueue中的worker_pool,最终，将函数queue_work中参数work指向的工作项加入到刚才找到的worker_pool中。所以，由此可以看出，在`***4.0内核中,工作池和工作者，工作项才是最核心的，而工作队列实际上是为了保持原有接口的兼容而引入的，在4.0中通过再引入pool_waitqueue结构体，完成工作队列到工作池的转化关系，对上层用户透明，用户对工作队列的添加操作最终会通过查询pool_workqueue进而变成对对工作池的添加操作.***    

**工作队列(workqueue_struct):** 为了保证对原有接口的兼容而引入，和工作池是对应关系。  

**工作者(worker):** 每个工作池(work_pool)对应至少一个工作者，是工作池中工作项的消费者，由内核线程(kworker)封装而成，用于执行具体work_struct中的钩子函数.  

**工作池中的具体工作项(work_struct):** 具体要推后执行的工作项，会被加入workqueue对应的工作池中的队列中，随后由工作池对应的worker执行其中的钩子函数.  


## 工作队列关键数据结构

### work_func_t 工作项中的钩子函数  



>include/linux/workqueue.h

work_func_t 是工作队列中每个工作项(work_struct结构体)的钩子函数，最终被工作项所属的工作队列(workqueue_struct)关联的woker_pool对应的woker封装的线程(kworker线程)调用并执行。

```c
typedef void (*work_func_t)(struct work_struct *work);
```
### work_struct 工作项结构体   

>include/linux/workqueue.h

work_struct 结构体是工作项的具体表现形式，通过queue_work实现将work_struct添加到工作队列(workqueue_struct)对应的工作池中的队列中，等待工作者线程消费掉:

```c
struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;  //定义钩子函数
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};
```

    
    queue_work  
     => queue_work_on  
      => __queue_work  
       => insert_work(pwq, work, worklist, work_flags);  //pwq为找到的保存工作池和工作队列对应关系的pool_workqueue结构体指针  
        => list_add_tail(&work->entry, head); //将工作项work添加到工作池队列中，供该池对应的工作者线程消费。  
    

### delayed_work

>include/linux/workqueue.h

delayed_work结构体基于work_struct，同时增加延时(timer)功能, 等delay超时候后将work item加入到工作队列中.
delayed_work类似work_struct结构体，主要不同之处在于其是等待delay延时后再通过timer的超时函数钩子将其中的work_struct结构体加入到工作队列中.

```c
struct delayed_work {
	struct work_struct work;  //包含work_struct结构体
	struct timer_list timer;  //保存延时

	/* target workqueue and CPU ->timer uses to queue ->work */
	struct workqueue_struct *wq;  //保存对应的工作队列指针
	int cpu;
};
```

添加延时工作项到工作队列函数接口:

```c
/**
 * queue_delayed_work - queue work on a workqueue after delay
 * @wq: workqueue to use
 * @dwork: delayable work to queue
 * @delay: number of jiffies to wait before queueing
 *
 * Equivalent to queue_delayed_work_on() but tries to use the local CPU.
 */
static inline bool queue_delayed_work(struct workqueue_struct *wq,
				      struct delayed_work *dwork,
				      unsigned long delay)
{
	return queue_delayed_work_on(WORK_CPU_UNBOUND, wq, dwork, delay);
}

```

#### workqueue_struct 工作队列结构体

>kernel/workqueue.c

该结构体存在的意义主要是为了保证现有机制对内核中原有接口的兼容和透明，对workqueue的操作最终通过查询pool_workqueue转化为对工作池的操作.

```c

/*
 * The externally visible workqueue.  It relays the issued work items to
 * the appropriate worker_pool through its pool_workqueues.
 */

struct workqueue_struct {
	struct list_head	pwqs;		/* WR: all pwqs of this wq */
	struct list_head	list;		/* PR: list of all workqueues */

	struct mutex		mutex;		/* protects this wq */
	int			work_color;	/* WQ: current work color */
	int			flush_color;	/* WQ: current flush color */
	atomic_t		nr_pwqs_to_flush; /* flush in progress */
	struct wq_flusher	*first_flusher;	/* WQ: first flusher */
	struct list_head	flusher_queue;	/* WQ: flush waiters */
	struct list_head	flusher_overflow; /* WQ: flush overflow list */

	struct list_head	maydays;	/* MD: pwqs requesting rescue */
	struct worker		*rescuer;	/* I: rescue worker */

	int			nr_drainers;	/* WQ: drain in progress */
	int			saved_max_active; /* WQ: saved pwq max_active */

	struct workqueue_attrs	*unbound_attrs;	/* PW: only for unbound wqs */
	struct pool_workqueue	*dfl_pwq;	/* PW: only for unbound wqs */

#ifdef CONFIG_SYSFS
	struct wq_device	*wq_dev;	/* I: for sysfs interface */
#endif
#ifdef CONFIG_LOCKDEP
	struct lockdep_map	lockdep_map;
#endif
	char			name[WQ_NAME_LEN]; /* I: workqueue name */

	/*
	 * Destruction of workqueue_struct is sched-RCU protected to allow
	 * walking the workqueues list without grabbing wq_pool_mutex.
	 * This is used to dump all workqueues from sysrq.
	 */
	struct rcu_head		rcu;

	/* hot fields used during command issue, aligned to cacheline */
	unsigned int		flags ____cacheline_aligned; /* WQ: WQ_* flags */
	struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
	struct pool_workqueue __rcu *numa_pwq_tbl[]; /* PWR: unbound pwqs indexed by node */
};

```


### worker_pool 工作池结构体

>kernel/workqueue.c

```c
/*
 * Structure fields follow one of the following exclusion rules.
 *
 * I: Modifiable by initialization/destruction paths and read-only for
 *    everyone else.
 *
 * P: Preemption protected.  Disabling preemption is enough and should
 *    only be modified and accessed from the local cpu.
 *
 * L: pool->lock protected.  Access with pool->lock held.
 *
 * X: During normal operation, modification requires pool->lock and should
 *    be done only from local cpu.  Either disabling preemption on local
 *    cpu or grabbing pool->lock is enough for read access.  If
 *    POOL_DISASSOCIATED is set, it's identical to L.
 *
 * A: pool->attach_mutex protected.
 *
 * PL: wq_pool_mutex protected.
 *
 * PR: wq_pool_mutex protected for writes.  Sched-RCU protected for reads.
 *
 * PW: wq_pool_mutex and wq->mutex protected for writes.  Either for reads.
 *
 * PWR: wq_pool_mutex and wq->mutex protected for writes.  Either or
 *      sched-RCU for reads.
 *
 * WQ: wq->mutex protected.
 *
 * WR: wq->mutex protected for writes.  Sched-RCU protected for reads.
 *
 * MD: wq_mayday_lock protected.
 */

/* struct worker is defined in workqueue_internal.h */

struct worker_pool {
	spinlock_t		lock;		/* the pool lock */
	int			cpu;		/* I: the associated cpu */
	int			node;		/* I: the associated node ID */
	int			id;		/* I: pool ID */
	unsigned int		flags;		/* X: flags */

	struct list_head	worklist;	/* L: list of pending works */ //工作项队列头
	int			nr_workers;	/* L: total number of workers */

	/* nr_idle includes the ones off idle_list for rebinding */
	int			nr_idle;	/* L: currently idle ones */

	struct list_head	idle_list;	/* X: list of idle workers */
	struct timer_list	idle_timer;	/* L: worker idle timeout */
	struct timer_list	mayday_timer;	/* L: SOS timer for workers */

	/* a workers is either on busy_hash or idle_list, or the manager */
	DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);
						/* L: hash of busy workers */

	/* see manage_workers() for details on the two manager mutexes */
	struct mutex		manager_arb;	/* manager arbitration */
	struct worker		*manager;	/* L: purely informational */
	struct mutex		attach_mutex;	/* attach/detach exclusion */
	struct list_head	workers;	/* A: attached workers */      //工作者队列头
	struct completion	*detach_completion; /* all workers detached */

	struct ida		worker_ida;	/* worker IDs for task name */

	struct workqueue_attrs	*attrs;		/* I: worker attributes */
	struct hlist_node	hash_node;	/* PL: unbound_pool_hash node */
	int			refcnt;		/* PL: refcnt for unbound pools */

	/*
	 * The current concurrency level.  As it's likely to be accessed
	 * from other CPUs during try_to_wake_up(), put it in a separate
	 * cacheline.
	 */
	atomic_t		nr_running ____cacheline_aligned_in_smp;

	/*
	 * Destruction of pool is sched-RCU protected to allow dereferences
	 * from get_work_pool().
	 */
	struct rcu_head		rcu;
} ____cacheline_aligned_in_smp;

```

### pool_workqueue 

>kernel/workqueue.c

用于保存工作队列和工作池的转化(关联)关系。

```c
/*
 * The per-pool workqueue.  While queued, the lower WORK_STRUCT_FLAG_BITS
 * of work_struct->data are used for flags and the remaining high bits
 * point to the pwq; thus, pwqs need to be aligned at two's power of the
 * number of flag bits.
 */
struct pool_workqueue {
	struct worker_pool	*pool;		/* I: the associated pool */
	struct workqueue_struct *wq;		/* I: the owning workqueue */
	int			work_color;	/* L: current color */
	int			flush_color;	/* L: flushing color */
	int			refcnt;		/* L: reference count */
	int			nr_in_flight[WORK_NR_COLORS];
						/* L: nr of in_flight works */
	int			nr_active;	/* L: nr of active works */
	int			max_active;	/* L: max active works */
	struct list_head	delayed_works;	/* L: delayed works */
	struct list_head	pwqs_node;	/* WR: node on wq->pwqs */
	struct list_head	mayday_node;	/* MD: node on wq->maydays */

	/*
	 * Release of unbound pwq is punted to system_wq.  See put_pwq()
	 * and pwq_unbound_release_workfn() for details.  pool_workqueue
	 * itself is also sched-RCU protected so that the first pwq can be
	 * determined without grabbing wq->mutex.
	 */
	struct work_struct	unbound_release_work;
	struct rcu_head		rcu;
} __aligned(1 << WORK_STRUCT_FLAG_BITS);

```


### worker 工作者(由kworker内核线程封装)

>kernel/workqueue_internal.h

是内核工作者线程封装后的结构体，主要保存工作线程信息，对应的工作池信息等。

```c
/*
 * The poor guys doing the actual heavy lifting.  All on-duty workers are
 * either serving the manager role, on idle list or on busy hash.  For
 * details on the locking annotation (L, I, X...), refer to workqueue.c.
 *
 * Only to be used in workqueue and async.
 */
struct worker {
	/* on idle list while idle, on busy hash table while busy */
	union {
		struct list_head	entry;	/* L: while idle */
		struct hlist_node	hentry;	/* L: while busy */
	};

	struct work_struct	*current_work;	/* L: work being processed */
	work_func_t		current_func;	/* L: current_work's fn */
	struct pool_workqueue	*current_pwq; /* L: current_work's pwq */
	bool			desc_valid;	/* ->desc is valid */
	struct list_head	scheduled;	/* L: scheduled works */

	/* 64 bytes boundary on 64bit, 32 on 32bit */

	struct task_struct	*task;		/* I: worker task */     //指向内核线程kworker的task_struct结构体指针
	struct worker_pool	*pool;		/* I: the associated pool */
						/* L: for rescuers */
	struct list_head	node;		/* A: anchored at pool->workers */
						/* A: runs through worker->node */

	unsigned long		last_active;	/* L: last active timestamp */
	unsigned int		flags;		/* X: flags */
	int			id;		/* I: worker id */

	/*
	 * Opaque string set with work_set_desc().  Printed out with task
	 * dump for debugging - WARN, BUG, panic or sysrq.
	 */
	char			desc[WORKER_DESC_LEN];

	/* used only by rescuers to point to the target workqueue */
	struct workqueue_struct	*rescue_wq;	/* I: the workqueue to rescue */
};
```

   

## 页回写机制，关键函数及流程分析


### bdi_wq 

>mm/backing-dev.c

bdi_wq是内核页回写机制(不同于fadvise64系统调用NO_NEED, 该系统调用直接使用mapping->a_ops->writepage刷脏页,而不通过bdi_wq来退后执行)的工作项对应的工作队列指针，内核页回写机制会以(延时)工作项的形式添加到该工作队列(对应的工作池中的队列)中.

#### 定义全局bdi_wq指针。

/* bdi_wq serves all asynchronous writeback tasks */
struct workqueue_struct *bdi_wq;

#### 初始化bdi_wq工作队列指针，其中最重要的函数是alloc_workqueue。

```c
static int __init default_bdi_init(void)
{
	int err;

	bdi_wq = alloc_workqueue("writeback", WQ_MEM_RECLAIM | WQ_FREEZABLE |
					      WQ_UNBOUND | WQ_SYSFS, 0);
	if (!bdi_wq)
		return -ENOMEM;

	err = bdi_init(&noop_backing_dev_info);

	return err;
}

```
all_workqueue函数主要创建worker, worker_pool, workqueue, 并通过结构体pool_workqueue关联worker_pool和workqueue, 最终返回workqueue结构体指针，流程如下:

    alloc_workqueue
     => __alloc_workqueue_key
      => alloc_and_link_pwqs  //将workqueue和和worker_pool结构体关联，形成转换关系
       => apply_workqueue_attrs
        => apply_workqueue_attrs_locked
         => apply_wqattrs_prepare
          => alloc_unbound_pwq
           => get_unbound_pool
            => create_worker
             => worker->task = kthread_create_on_node(worker_thread, worker, pool->node, "kworker/%s", id_buf);  //创建worker中封装的内核线程。





### wb_init(初始化用于页缓存回写的通用工作项及通用工作项钩子函数为wb_workfn)

>mm/backing-dev.c

```c
static int wb_init(struct bdi_writeback *wb, struct backing_dev_info *bdi,
		   int blkcg_id, gfp_t gfp)
{
	int i, err;

	memset(wb, 0, sizeof(*wb));

	wb->bdi = bdi;
	wb->last_old_flush = jiffies;
	INIT_LIST_HEAD(&wb->b_dirty);
	INIT_LIST_HEAD(&wb->b_io);
	INIT_LIST_HEAD(&wb->b_more_io);
	INIT_LIST_HEAD(&wb->b_dirty_time);
	spin_lock_init(&wb->list_lock);

	wb->bw_time_stamp = jiffies;
	wb->balanced_dirty_ratelimit = INIT_BW;
	wb->dirty_ratelimit = INIT_BW;
	wb->write_bandwidth = INIT_BW;
	wb->avg_write_bandwidth = INIT_BW;

	spin_lock_init(&wb->work_lock);
	INIT_LIST_HEAD(&wb->work_list);
	INIT_DELAYED_WORK(&wb->dwork, wb_workfn);   //此处是重点，初始化dwork中的work_struct的钩子函数
	                                            //为wb_workfn函数,同时设置定时器超时回调钩子函数为
	                                            //delayed_work_timer_fn, 该超时函数会将dwork中的工作项
	                                            //添加到bdi_wq工作队列中(在后面调用queue_delayed_work时,
	                                            //激活该定时器, 超时后，delayed_work_timer_fn钩子函数将
	                                            //queue_delay_work函数参数中的dwork加入bdi_wq队列中.)
	                                            

	

	wb->congested = wb_congested_get_create(bdi, blkcg_id, gfp);
	if (!wb->congested)
		return -ENOMEM;

	err = fprop_local_init_percpu(&wb->completions, gfp);
	if (err)
		goto out_put_cong;

	for (i = 0; i < NR_WB_STAT_ITEMS; i++) {
		err = percpu_counter_init(&wb->stat[i], 0, gfp);
		if (err)
			goto out_destroy_stat;
	}

	return 0;

out_destroy_stat:
	while (--i)
		percpu_counter_destroy(&wb->stat[i]);
	fprop_local_destroy_percpu(&wb->completions);
out_put_cong:
	wb_congested_put(wb->congested);
	return err;
}
```


### wb_workfn函数 

>fs/fs-writeback.c

是writeback具体的工作项钩子函数，该函数最终由工作者对应的线程(kworker)执行。

```c
/*
 * Handle writeback of dirty data for the device backed by this bdi. Also
 * reschedules periodically and does kupdated style flushing.
 */
void wb_workfn(struct work_struct *work)
{
	struct bdi_writeback *wb = container_of(to_delayed_work(work),
						struct bdi_writeback, dwork);
	long pages_written;

	set_worker_desc("flush-%s", dev_name(wb->bdi->dev));
	current->flags |= PF_SWAPWRITE;

	if (likely(!current_is_workqueue_rescuer() ||
		   !test_bit(WB_registered, &wb->state))) {
		/*
		 * The normal path.  Keep writing back @wb until its
		 * work_list is empty.  Note that this path is also taken
		 * if @wb is shutting down even when we're running off the
		 * rescuer as work_list needs to be drained.
		 */
		do {
			pages_written = wb_do_writeback(wb);      // 实施页缓存写操作函数
			trace_writeback_pages_written(pages_written);
		} while (!list_empty(&wb->work_list));
	} else {
		/*
		 * bdi_wq can't get enough workers and we're running off
		 * the emergency worker.  Don't hog it.  Hopefully, 1024 is
		 * enough for efficient IO.
		 */
		pages_written = writeback_inodes_wb(wb, 1024,
						    WB_REASON_FORKER_THREAD);
		trace_writeback_pages_written(pages_written);
	}

	if (!list_empty(&wb->work_list))
		mod_delayed_work(bdi_wq, &wb->dwork, 0);   //如果wb->work_list非空，重新将wb->dwork中的work_struct工作项加入bdi_wq队列中。
	else if (wb_has_dirty_io(wb) && dirty_writeback_interval)  //页缓存中有脏页同时dirty_writeback_interva为真
		wb_wakeup_delayed(wb);    //以dirty_writeback_interva为超时时间，重新将wb->dwork
		                          //中的work_struct加入bdi_wq队列中，这样再delay了
		                          //dirty_writeback_interva时间后重新将wb->dwork->work加入bdi_wq队列
		                          //最终重新执行wb_workfn函数。

	current->flags &= ~PF_SWAPWRITE;
}
```

### wb_do_writeback 

>fs/fs-writeback.c

该函数是wb_workfn中真正实施页缓存写操作的函数，最终调用address_space->a_ops->writepage将页缓存中的page提交给block层最后到达存储介质.
    
    wb_workfn
     => wb_do_writeback
      => wb_writeback
       => writeback_sb_inodes
        => __writeback_single_inode
         => do_writepages
          => generic_writepages
           => __writepage
            => int ret = mapping->a_ops->writepage(page, wbc);  //调用具体文件系统address_space->a_ops->writepage函数，最终提交到存储介质.

**由此可见，address_space->a_ops->writepage是页缓存回写机制最终调用的函数接口，该接口由具体的文件系统实现并初始化**

### wb_wakeup_delayed

>mm/backing-dev.c

该函数用于推迟 dirty_writeback_interval 时间后将预先定义好的wb->dwork->work工作项加入到bdi_wq工作队列中。在上层写入页缓存后首次标记脏页的时候，会调用该函数，唤醒内核页回写机制在推迟 dirty_writeback_interval 时间后将预先定义好的wb->dwork->work工作项加入到bdi_wq工作队列中，最终bdi_wq关联的worker_pool对应的worker工作者封装的kworker线程会消费该工作项(wb->dwork->work)。

```c
/*
 * This function is used when the first inode for this wb is marked dirty. It
 * wakes-up the corresponding bdi thread which should then take care of the
 * periodic background write-out of dirty inodes. Since the write-out would
 * starts only 'dirty_writeback_interval' centisecs from now anyway, we just
 * set up a timer which wakes the bdi thread up later.
 *
 * Note, we wouldn't bother setting up the timer, but this function is on the
 * fast-path (used by '__mark_inode_dirty()'), so we save few context switches
 * by delaying the wake-up.
 *
 * We have to be careful not to postpone flush work if it is scheduled for
 * earlier. Thus we use queue_delayed_work().
 */
void wb_wakeup_delayed(struct bdi_writeback *wb)
{
	unsigned long timeout;

	timeout = msecs_to_jiffies(dirty_writeback_interval * 10);
	spin_lock_bh(&wb->work_lock);
	if (test_bit(WB_registered, &wb->state))
		queue_delayed_work(bdi_wq, &wb->dwork, timeout);
	spin_unlock_bh(&wb->work_lock);
}
```