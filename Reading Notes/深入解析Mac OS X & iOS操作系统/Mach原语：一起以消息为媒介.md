# Mach原语：一起以消息为媒介
## 1. Mach概述
### 1.1 Mach设计原则
- 在Mach中所有东西（Task、线程、虚拟内存等））都是对象。
- 对象与对象之间通信**只能**通过端口收发消息。

### 1.2 Mach设计目标
内核为了保持极简，只做如下的事情：

- “控制点”或执行单元的管理。
- 线程或线程组（Task）的资源分配。
- 虚拟内存的分配和管理。
- 底层物理资源--即CPU、内存和任何物理设备的分配。

## 2. Mach消息
### 2.1 简单消息
最基本的包含两部分：消息头、消息体。可以选择性的添加消息尾。结构如下：

```
typedef	struct 
{
  mach_msg_bits_t	msgh_bits;//标志位
  mach_msg_size_t	msgh_size;//大小
  mach_port_t		msgh_remote_port;//目标端口（发送：接受方，接收：发送方）
  mach_port_t		msgh_local_port; //源端口（发送：发送方，接收：接收方）
  mach_port_name_t	msgh_voucher_port;
  mach_msg_id_t		msgh_id;
} mach_msg_header_t; //消息头

typedef struct
{
        mach_msg_size_t msgh_descriptor_count;
} mach_msg_body_t;//消息体

typedef struct
{
        mach_msg_header_t       header;
        mach_msg_body_t         body;
} mach_msg_base_t; //基本消息

typedef	unsigned int mach_msg_trailer_type_t;//消息尾的类型

typedef struct 
{
  mach_msg_trailer_type_t	msgh_trailer_type;
  mach_msg_trailer_size_t	msgh_trailer_size;
} mach_msg_trailer_t; //消息尾

```

### 2.2 复杂消息
将消息头的标志位`mach_msg_bits_t`设置为`MACH_MSGH_BITS_COMPLEX`，就表示复杂消息。此时消息体里面指定了描述符的个数，接下来就是一个接着一个的描述符：

```
typedef struct
{
  uint64_t			address;//数据的大小
  boolean_t     		deallocate: 8;//发送之后是否接触分配
  mach_msg_copy_options_t       copy: 8;//复制指令
  unsigned int     		pad1: 8;
  mach_msg_descriptor_type_t    type: 8;
  mach_msg_size_t       	size;//数据的大小
} mach_msg_ool_descriptor64_t;

```

### 2.3 消息收发
消息的收发在用户态都是通过如下方法进行的：

```
extern mach_msg_return_t	mach_msg(
					mach_msg_header_t *msg,
					mach_msg_option_t option,//可以设置为收消息还是发消息等类型
					mach_msg_size_t send_size,
					mach_msg_size_t rcv_size,
					mach_port_name_t rcv_name,
					mach_msg_timeout_t timeout,
					mach_port_name_t notify);					
```

### 2.4 端口
端口实际上就是一个整型的标识符，是如下结构（在osfmk/ipc/ipc_port.h中定义）的一个句柄：

```
struct ipc_port {

	/*
	 * Initial sub-structure in common with ipc_pset
	 * First element is an ipc_object second is a
	 * message queue
	 */
	struct ipc_object ip_object;
	struct ipc_mqueue ip_messages;

	natural_t ip_sprequests:1,	/* send-possible requests outstanding */
		  ip_spimportant:1,	/* ... at least one is importance donating */
		  ip_impdonation:1,	/* port supports importance donation */
		  ip_tempowner:1,	/* dont give donations to current receiver */
		  ip_guarded:1,         /* port guarded (use context value as guard) */
		  ip_strict_guard:1,	/* Strict guarding; Prevents user manipulation of context values directly */
		  ip_reserved:2,
		  ip_impcount:24;	/* number of importance donations in nested queue */

	union {
		struct ipc_space *receiver;
		struct ipc_port *destination;
		ipc_port_timestamp_t timestamp;
	} data;

	union {
		ipc_kobject_t kobject;
		ipc_importance_task_t imp_task;
		uintptr_t alias;
	} kdata;
		
	struct ipc_port *ip_nsrequest;
	struct ipc_port *ip_pdrequest;
	struct ipc_port_request *ip_requests;
	struct ipc_kmsg *ip_premsg;

	mach_vm_address_t ip_context;

	mach_port_mscount_t ip_mscount;
	mach_port_rights_t ip_srights;
	mach_port_rights_t ip_sorights;

#if	MACH_ASSERT
#define	IP_NSPARES		4
#define	IP_CALLSTACK_MAX	16
/*	queue_chain_t	ip_port_links;*//* all allocated ports */
	thread_t	ip_thread;	/* who made me?  thread context */
	unsigned long	ip_timetrack;	/* give an idea of "when" created */
	uintptr_t	ip_callstack[IP_CALLSTACK_MAX]; /* stack trace */
	unsigned long	ip_spares[IP_NSPARES]; /* for debugging */
#endif	/* MACH_ASSERT */
} __attribute__((__packed__));

```

### 2.5 Mach接口生成器（MIG）
Mach消息传递模型是远程调用（Remote Procedure Call，RPC）的一种现实(类似Thrift)。在/usr/include/mach目录下可以看到一些`.defs`文件，这些文件包含了Mach子系统（一组操作）的定义。操作类型如下：
![IG_opt.png][1]

## 3. 深入IPC
- Mach的每个Task都包含一个指针，这个指针指向一个IPC命名空间，这个IPC命名空间了包含了Task的端口，当然Task还可以获取系统范围的端口，例如：主机端口、特权端口（可以重启机器等）等。
- 在用户态下，消息传递都是通过`mach_msg()`函数实现的，这个函数会触发一个mach陷阱`mach_msg_trap()`，接下来`mach_msg_trap()`又会调用`mach_msg_overwrite_trap()`，它会通过`MACH_SEND_MSG`和`MACH_RCV_MSG`来判断是发送操作，还是接收操作。
- 期中内核态中还可以通过`mach_msg_receive()`和`mach_msg_send()`来收发数据。

## 4. 同步原语
### 4.1 锁的实现方式
- **阻塞**：如果锁对象被其他线程所持有，那么请求访问的线程就会被加入到等待队列中，因而被阻塞。这就意味着被阻塞的线程放弃了时间片，调度器会将CPU让给下一个执行的的线程。**当锁可用的时候**，调度器会得到通知，然后根据情况将线程从等待队列取出来，并重新调度。
- **忙等**：线程不放弃CPU时间片，而是继续重复的尝试访问所对，直到锁可用。
- **阻塞与忙等的对比**：当锁只持续短短几个周期的时候，阻塞会带来性能问题，因为至少会消耗两次上下文切换的时间。因此如果锁的持续时间短，应该采用忙等形式的锁对象，反之就采用阻塞形式的锁对象。

### 4.2 互斥体(lck_mtx_t)（阻塞）
- 互斥体其实就是一个内核中的一个不同的变量，通常是机器字节大小的整数，但是必须要求硬件能对这些变量进行原子操作。
- 原子的意思就是：对互斥体的操作不能打断，即使是硬件中断也不能打断。

### 4.3 信号量(semaphore_t)（阻塞）
信号量在初始化的时候可以设置一个大于零的初始值。信号量包含两个操作：一个是+1操作，一个是-1操作，当值大于0表示锁可用，当值小于等于0的时候表示锁不可用。互斥体可以看做是初始值为1的信号量。

### 4.4 自旋锁(hw_lock_t)（忙等）
一种采用忙等形式的锁。
### 4.5 读写锁(hw_lock_t)（阻塞）
当多个线程对资源只做只读的操作，这种情况下这些线程并不会相互影响。为了提高效率，读写锁就应运而生了。读写锁能够区分是读访问还是写访问，对个读者可以同时持有锁，但一次只能一个写者持有锁。

### 4.6 锁集(lock_set_t)
锁集就是锁的一个数组。

## 5. 机器原语
### 5.1 主机对象（Host）
主机就是一组“特殊”端口的集合，以及一组异常处理程序的集合，同时定义了一个锁用于保护异常处理的并发访问。结构如下：

```
struct	host {
	decl_lck_mtx_data(,lock)		/* lock to protect exceptions */
	ipc_port_t special[HOST_MAX_SPECIAL_PORT + 1];
	struct exception_action exc_actions[EXC_TYPES_COUNT];
};
```

### 5.2 时钟对象（Clock）
Mach内核提供了一个简单的“时钟”对象（在osfmk/kern/clock.h中定义）的抽象，这个对象用于计时和闹铃，期中最重要的内部API是`clock_deadline_for_periodic_event（）`，调度器通过它设置了一个重复发生的通知--从而保证了多任务引擎的运转。

### 5.3 处理器对象（Processer）
在多核架构中每一个核心都可以看做是一个CPU，处理器被分配给**处理器集**，处理器是CPU的简单抽象，被Mach用于一些基本的操作，比如：启动和关闭一个CPU，向CPU分发要执行的线程。结构的定义（在osfmk/kern/processor.h）如下：

```
struct processor {
	queue_chain_t		processor_queue;/* idle/active queue link,
										 * MUST remain the first element */
	int					state;			/* See below */
	boolean_t		is_SMT;
	boolean_t		is_recommended;
	struct thread
						*active_thread,	/* thread running on processor */
						*next_thread,	/* next thread when dispatched */
						*idle_thread;	/* this processor's idle thread. */

	processor_set_t		processor_set;	/* assigned set */

	int					current_pri;	/* priority of current thread */
	sched_mode_t		current_thmode;	/* sched mode of current thread */
	sfi_class_id_t		current_sfi_class;	/* SFI class of current thread */
	int					cpu_id;			/* platform numeric id */

	timer_call_data_t	quantum_timer;	/* timer for quantum expiration */
	uint64_t			quantum_end;	/* time when current quantum ends */
	uint64_t			last_dispatch;	/* time of last dispatch */

	uint64_t			deadline;		/* current deadline */
	boolean_t               first_timeslice;                /* has the quantum expired since context switch */

#if defined(CONFIG_SCHED_TRADITIONAL) || defined(CONFIG_SCHED_MULTIQ)
	struct run_queue	runq;			/* runq for this processor */
#endif

#if defined(CONFIG_SCHED_TRADITIONAL)
	int					runq_bound_count; /* # of threads bound to this processor */
#endif
#if defined(CONFIG_SCHED_GRRR)
	struct grrr_run_queue	grrr_runq;      /* Group Ratio Round-Robin runq */
#endif

	processor_t			processor_primary;	/* pointer to primary processor for
											 * secondary SMT processors, or a pointer
											 * to ourselves for primaries or non-SMT */
	processor_t		processor_secondary;
	struct ipc_port *	processor_self;	/* port for operations */

	processor_t			processor_list;	/* all existing processors */
	processor_data_t	processor_data;	/* per-processor data */
};
```
其中最重要的是runq，这是分发到这个处理器的线程队列。

### 5.3 处理器集
处理器集就是一个或多个processor_t的分组，也被称为pset。pset通常维护三个队列：

- `active_queue`：用于保存当前正在执行线程的CPU。
- `idle_queue`：用于保存当前空闲的CPU（例如：正在执行`idle_thread`）。
- `pset_runq`：保存了在这个集合中的所有CPU上执行的线程。

`processor_set`的定义如下：

```
struct processor_set {
	queue_head_t		active_queue;	/* active processors */
	queue_head_t		idle_queue;		/* idle processors */
	queue_head_t		idle_secondary_queue;		/* idle secondary processors */

	int					online_processor_count;

	int					cpu_set_low, cpu_set_hi;
	int					cpu_set_count;

#if __SMP__
	decl_simple_lock_data(,sched_lock)	/* lock for above */
#endif

#if defined(CONFIG_SCHED_TRADITIONAL) || defined(CONFIG_SCHED_MULTIQ)
	struct run_queue	pset_runq;      /* runq for this processor set */
#endif

#if defined(CONFIG_SCHED_TRADITIONAL)
	int					pset_runq_bound_count;
		/* # of threads in runq bound to any processor in pset */
#endif

	/* CPUs that have been sent an unacknowledged remote AST for scheduling purposes */
	uint64_t			pending_AST_cpu_mask;
#if defined(CONFIG_SCHED_DEFERRED_AST)
	/*
	 * A seperate mask, for ASTs that we may be able to cancel.  This is dependent on
	 * some level of support for requesting an AST on a processor, and then quashing
	 * that request later.
	 *
	 * The purpose of this field (and the associated codepaths) is to infer when we
	 * no longer need a processor that is DISPATCHING to come up, and to prevent it
	 * from coming out of IDLE if possible.  This should serve to decrease the number
	 * of spurious ASTs in the system, and let processors spend longer periods in
	 * IDLE.
	 */
	uint64_t			pending_deferred_AST_cpu_mask;
#endif

	struct ipc_port	*	pset_self;		/* port for operations */
	struct ipc_port *	pset_name_self;	/* port for information */

	processor_set_t		pset_list;		/* chain of associated psets */
	pset_node_t			node;
};
```

---
[1]: https://github.com/Easence/EADocuments/blob/master/Reading%20Notes/深入解析Mac%20OS%20X%20&%20iOS操作系统/Resources/Images/MIG_opt.png?raw=true