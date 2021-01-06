## 进程控制块

进程状态	/include/kernel/sched.h

```c
#define TASK_RUNNING		0
#define TASK_INTERRUPTIBLE	1
#define TASK_UNINTERRUPTIBLE	2
#define TASK_ZOMBIE		4
#define TASK_STOPPED		8
```

内核栈  ／include/linux/sched.h

```c
union task_union {
	struct task_struct task;
	unsigned long stack[INIT_TASK_SIZE/sizeof(long)];
};
```

当前进程	/ linux/include/asm-i386／current.h

```c
static inline struct task_struct * get_current(void)
{
	struct task_struct *current;
	__asm__("andl %%esp,%0; ":"=r" (current) : "0" (~8191UL));
	return current;
 }
```

进程调度策略

```c 
/*
 * Scheduling policies
 */
#define SCHED_OTHER		0  //普通进程基于优先级的的时间片轮转调度
#define SCHED_FIFO		1  //实时进程使用的的先进先出策略，进程会一直占用cpu除非其自动放弃cpu
#define SCHED_RR		2  //实时进程的轮转策略，当分配个u进程的时间片用完后，进程会插入到原来优先级的队列中
/*
 * This is an additional bit set when we want to
 * yield the CPU for one re-schedule..
 */
#define SCHED_YIELD		0x10
```

进程的组织方式

1. 哈希表  sched.h

```c
static inline struct task_struct *find_task_by_pid(int pid)
{
	struct task_struct *p, **htable = &pidhash[pid_hashfn(pid)];

	for(p = *htable; p && p->pid != pid; p = p->pidhash_next)
		;

	return p;
}
```

2. 双向循环列表  sched.h

```c
#define REMOVE_LINKS(p) do { \
	(p)->next_task->prev_task = (p)->prev_task; \
	(p)->prev_task->next_task = (p)->next_task; \
	if ((p)->p_osptr) \
		(p)->p_osptr->p_ysptr = (p)->p_ysptr; \
	if ((p)->p_ysptr) \
		(p)->p_ysptr->p_osptr = (p)->p_osptr; \
	else \
		(p)->p_pptr->p_cptr = (p)->p_osptr; \
	} while (0)

#define SET_LINKS(p) do { \
	(p)->next_task = &init_task; \
	(p)->prev_task = init_task.prev_task; \
	init_task.prev_task->next_task = (p); \
	init_task.prev_task = (p); \
	(p)->p_ysptr = NULL; \
	if (((p)->p_osptr = (p)->p_pptr->p_cptr) != NULL) \
		(p)->p_osptr->p_ysptr = p; \
	(p)->p_pptr->p_cptr = p; \
	} while (0)

#define for_each_task(p) \
	for (p = &init_task ; (p = p->next_task) != &init_task ; )

/**
 * list_entry - get the struct for this entry
 * @ptr:	the &struct list_head pointer.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_struct within the struct.
 */
#define list_entry(ptr, type, member) \
	((type *)((char *)(ptr)-(unsigned long)(&((type *)0)->member)))

/**
 * list_for_each	-	iterate over a list
 * @pos:	the &struct list_head to use as a loop counter.
 * @head:	the head for your list.
 */
#define list_for_each(pos, head) \
	for (pos = (head)->next, prefetch(pos->next); pos != (head); \
        	pos = pos->next, prefetch(pos->next))
```

3. 运行队列 双向链表

```c
struct list_head {
	struct list_head *next, *prev;
};
/*
 * Careful!
 *
 * This has to add the process to the _beginning_ of the
 * run-queue, not the end. See the comment about "This is
 * subtle" in the scheduler proper..
 */
static inline void add_to_runqueue(struct task_struct * p)
{
	list_add(&p->run_list, &runqueue_head);
	nr_running++;
}

static inline void move_last_runqueue(struct task_struct * p)
{
	list_del(&p->run_list);
	list_add_tail(&p->run_list, &runqueue_head);
}

static inline void move_first_runqueue(struct task_struct * p)
{
	list_del(&p->run_list);
	list_add(&p->run_list, &runqueue_head);
}
```

4. 等待队列

对等待队列进行操作的函数在wait.h中



## 进程调度

**调度系统的表现关系到整个系统的性能**

**linux的时间系统**

RTC和OS时钟，linux通过读取RTC初始化OS时钟

时钟滴答，Linux 中用全局变量 jiffies 表示系统自启动以来的时钟滴答数目。

```c
unsigned long volatile jiffies
```

weight：调度程序以这个权值作为选择进程的唯一依据

```c
static inline int goodness(struct task_struct * p, int this_cpu, struct mm_struct *this_mm)
{
	int weight;

	/*
	 * select the current process after every other
	 * runnable process, but before the idle thread.
	 * Also, dont trigger a counter recalculation.
	 */
	weight = -1;
	if (p->policy & SCHED_YIELD)
		goto out;

	/*
	 * Non-RT process - normal case first.
	 */
	if (p->policy == SCHED_OTHER) {
		/*
		 * Give the process a first-approximation goodness value
		 * according to the number of clock-ticks it has left.
		 *
		 * Don't do any other calculations if the time slice is
		 * over..
		 */
		weight = p->counter;
		if (!weight)
			goto out;
			
#ifdef CONFIG_SMP
		/* Give a largish advantage to the same processor...   */
		/* (this is equivalent to penalizing other processors) */
		if (p->processor == this_cpu)
			weight += PROC_CHANGE_PENALTY;
#endif

		/* .. and a slight advantage to the current MM */
		if (p->mm == this_mm || !p->mm)
			weight += 1;
		weight += 20 - p->nice;
		goto out;
	}

	/*
	 * Realtime process, select the first one on the
	 * runqueue (taking priorities within processes
	 * into account).
	 */
	weight = 1000 + p->rt_priority;
out:
	return weight;
}
```

**idle进程**

> idle进程其pid=0，其前身是系统创建的第一个进程，也是唯一一个没有通过fork()产生的进程。在smp系统中，每个处理器单元有独立的一个运行队列，而每个运行队列上又有一个idle进程，即有多少处理器单元，就有多少idle进程。系统的空闲时间，其实就是指idle进程的”运行时间”。

**switch_to()**

```c
#define prepare_to_switch()	do { } while(0)
#define switch_to(prev,next,last) do {					\
	asm volatile("pushl %%esi\n\t"					\
		     "pushl %%edi\n\t"					\
		     "pushl %%ebp\n\t"					\
		     "movl %%esp,%0\n\t"	/* save ESP */		\
		     "movl %3,%%esp\n\t"	/* restore ESP */	\
		     "movl $1f,%1\n\t"		/* save EIP */		\
		     "pushl %4\n\t"		/* restore EIP */	\
		     "jmp __switch_to\n"				\
		     "1:\t"						\
		     "popl %%ebp\n\t"					\
		     "popl %%edi\n\t"					\
		     "popl %%esi\n\t"					\
		     :"=m" (prev->thread.esp),"=m" (prev->thread.eip),	\
		      "=b" (last)					\
		     :"m" (next->thread.esp),"m" (next->thread.eip),	\
		      "a" (prev), "d" (next),				\
		      "b" (prev));					\
} while (0)
```

它保存一些关键的寄存器，并在栈上设置好跳转到新进程的地址。


