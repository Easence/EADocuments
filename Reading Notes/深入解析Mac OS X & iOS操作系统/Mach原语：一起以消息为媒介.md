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
Mach消息传递模型是远程调用（Remote Procedure Call，RPC）的一种现实。在/usr/include/mach目录下可以看到一些`.defs`文件，这些文件包含了Mach子系统（一组操作）的定义。操作类型如下：

## 3. 深入IPC
## 4. 同步原语
## 5. 机器原语