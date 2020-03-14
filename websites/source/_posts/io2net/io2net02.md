---
title: epoll技术扩展
category: 网络IO
date: 2020-03-13 16:44:43
tags: 网络IO编程
---

<!-- more -->
##### Epoll技术回顾
基于上一篇的epoll技术存在一些问题，对此将纠正部分要点
###### epoll使用技术

> 使用锁的技术

- 读写锁:内核在操作对象进行轮询的时候加读锁,而通过加写锁为了保证唤醒只执行一次,即在网络socket数据报可达,通过中断上下文调用`wake_up()`方法来触发回调`callback`方法的执行,通过callback方法将执行的task添加到CPU就绪队列️中，方便CPU进行调度执行
- 使用epoll空间的内置的锁`mtx`:当事件就绪的时候,内核需要将就绪的socket拷贝到用户空间,为了保证期间能够被拷贝而能够进行休眠,休眠的过程需要进行加锁
- 全局锁:通过加全局锁释放epoll容器下的资源,避免产生死锁

###### epoll技术使用SLAB的方式进行内存管理
epoll技术基于SLAB的方式管理内存,通过SLAB的方式来创建epoll空间以及epitem结构体对象

> epoll对象

```c
// eventpoll.c
ep = kzalloc(sizeof(*ep), GFP_KERNEL);

// slab.h
static inline void *kzalloc(size_t size, gfp_t gfp)
{
     return kmalloc(size, gfp | __GFP_ZERO);
}
```

> epitem结构体

```c
/* Slab cache used to allocate "struct epitem" */
static struct kmem_cache *epi_cache __read_mostly;

/* Slab cache used to allocate "struct eppoll_entry" */
static struct kmem_cache *pwq_cache __read_mostly;
```

> SLAB内存管理特点

- 使用连续的内存地址空间来存储epitem/epoll,避免内存碎片(多个epoll的产生是在多核下多进程)
- 使用的epitem/epoll释放存放在"对象池"中进行重复利用，同时减少创建和销毁epitem带来的性能开销(内存申请和释放的开销),可以理解为高速缓存
- 内存分配原理如下:
![slab_principle9dde9930a78a6378.jpg](http://s1.wailian.download/2020/03/14/slab_principle9dde9930a78a6378.jpg)

###### epoll技术设计

> epoll空间以及epitem部分源代码

```c
struct eventpoll {
    /* Wait queue used by sys_epoll_wait() */
    wait_queue_head_t wq;
    
    /* Wait queue used by file->poll() */
    wait_queue_head_t poll_wait;
    
    /* List of ready file descriptors */
    struct list_head rdllist;
    
    /* Lock which protects rdllist and ovflist */
    rwlock_t lock;  // 读写锁
    
    /* RB tree root used to store monitored fd structs */
    struct rb_root_cached rbr;  // 红黑树根节点
    
    /*
     * This is a single linked list that chains all the "struct epitem" that
     * happened while transferring ready events to userspace w/out
     * holding ->lock.
     */
    struct epitem *ovflist;
}

struct epitem {
	union {
		/* RB tree node links this structure to the eventpoll RB tree */
		struct rb_node rbn;     //连接红黑树结构的节点
		/* Used to free the struct epitem */
		struct rcu_head rcu;
	};
    
    /* List header used to link this structure to the eventpoll ready list */
    struct list_head rdllink;   // 就绪队列header节点，与epoll空间的ready list连接
    
    /* The file descriptor information this item refers to */
    struct epoll_filefd ffd;    // 存储注册的socket的fd以及对应的file
    
	/* The "container" of this item */
	struct eventpoll *ep;   // 指向epoll容器
	
	/* wakeup_source used when EPOLLWAKEUP is set */
    struct wakeup_source __rcu *ws;
    
    /* The structure that describe the interested events and the source fd */
    struct epoll_event event;   // 监听的事件变化结构
}
```

> epoll设计分析

- epoll技术在逻辑设计上,epoll空间作为epitem的容器,同时将注册的socket绑定到epitem中,并且epoll空间与epitem在逻辑设计存储上用红黑树结构进行存储，方便通过socket的fd快速定义到对应的epitem
- epoll空间通过ovflist将所有就绪的epitem以单链表的结构连接起来
- epitem包含wake_up唤醒函数以及对应的socket描述符信息，同时在注册新的socket的时候绑定对应的一个epitem,而每个epitem都将添加到对应的epoll_entry节点上,并将epoll_entry节点添加到epoll空间下的等待队列中,在系统调用`epoll_wait`的时候进行轮询
- `ep_wait`通过epoll空间的轮询队列wait queue进行事件轮询遍历检测当前是否有就绪事件,如果有会将就绪事件拷贝到`ready list`队列中
- 对此基于上述进行小结,对于wait_queue - poll_wait_queue - ovflist - ready_list,其流程如下：
![epoll_logic_struct.jpg](http://s1.wailian.download/2020/03/14/epoll_logic_struct.jpg)


###### Epoll给予的思考小结

> 使用epitem&epoll技术的思考

- epoll技术通过使用epitem中间层的方式来完成对每个注册的socket进行监控,通过对等待队列上事件节点的轮询将就绪节点以链表的方式连接起来,避免了数组遍历与查找
- 借助epoll容器来对每个epitem进行管理,由内核对epoll空间进行事件监控,即将所有的事情委托给容器来进行操作,epoll容器将最终的结果返回告知唤醒处理逻辑
- 从epoll技术的思考中，我们也可以想到在实际业务工作领域中，可以借鉴epoll思想引入中间层来解决原有结构存在的不足,通过中间层增强功能来弥补不足 

> 采用分散冗余思想

- epoll技术为了解决大内存数据拷贝问题,将注册和等待进行拆分,分别针对对应的细小功能模块进行优化和改进,也就是相比select/poll设计上更为细粒度且专业化
- 为了提升性能,在内部使用SLAB内存管理方式会预先申请连续内存存储对应的对象,也就是在某一个时刻上存在空间的冗余
- 对此,在我们大型系统中,为了缓解高并发流量的压力,也会采取集群分布式的方式来分担系统的流量负载,也会存在分散冗余的设计


##### 其他高级轮询技术
###### /dev/poll

是Solaris操作系统上名为`/dev/poll`的特殊文件提供了可扩展的轮询大量描述,相比select技术,其轮询技术可以预先设置好待查询的文件描述符列表,然后进入一个循环等待事件发生,每次循环回来之后不需要再设置该列表,其流程如下:

- 打开`/dev/poll`文件，然后初始化一个pollfd结构数组(poll使用的结构)
- 调用`write`方法往`/dev/poll`写这个结构数组并传递给内核
- 执行`io_ctl`的`DP_POLL`阻塞自身以等待事件的发生
- 调用结构如下:

```c
struct dvpoll{
    struct pollfd* dp_fds;      // 链表的形式,指向一个缓冲区,提供给ioctl返回的时候存储一个链表的数组
    int            dp_nfds;     // 缓冲区成员大小
    int            timeout; 
}
```

- 解决大内存拷贝问题,但是对于`/dev/poll`的实现需要查看对应的solaris系统细节,存在兼容性问题

###### kqueue技术

- FreeBSD4.1引入kqueue技术,允许进程向内核注册描述所关注的kqueue事件的事件过滤器(event filter),其定义如下:

```c
// 返回一个新的kqueue描述符,用户后续的kevent调用
int kqueue(void);

// 用于注册事件也用于确定是否有事件发生
int kevent(int kq,                                                                  // kqueue的注册事件fd
           const struct kevent *changelist, int nchanges,                           // 给出关注事件作出的更改,无更改为NULL & 0
           struct kevent *eventlist, int nevents,                                   // kevent结构数组
           const struct timespc *timeout);                                          // 超时

// 更新事件函数
void EV_SET(struct kevent *kev, uintptr_t ident, short filter, u_short flags, u_int fflags, intptr_r data, void *udata);

// kevent结构体
struct kevent {
    uintptr_t   ident;
    short filter;
    u_short flags;
    u_int fflags;
    intptr_r data;
    void *udata;
}
```

- 从上述的结构可以推测出与`epoll`的实现原理类似,只不过相比epoll实现,增加更多事件的监听(异步IO/文件修改通知/进程跟踪/信号处理等)
- 但是和`/dev/poll`一样存在的兼容性问题,目前是在FreeBSD系统中
- 对应不同的事件以及事件的过滤器，如下：
![kqueue_filters.jpg](http://s1.wailian.download/2020/03/14/kqueue_filters.jpg)
![kqueue_flags.jpg](http://s1.wailian.download/2020/03/14/kqueue_flags.jpg)


##### C10K
###### 什么是C10K问题
> 对于C10K问题,给出以下的参考链接

```text
https://en.wikipedia.org/wiki/C10k_problem
```

> C10K的理解

摘录wiki百科,在互联网应用中,服务端进程要处理大量的客户端的socket,C10K命名是为了同时处理1w个连接的缩写名称,是属于一个优化问题.

###### 现有解决C10K成熟方案

- Nginx是为了解决C10K设计的高并发连接处理服务器
- IO框架
- 事件驱动设计
- Reactor设计模式

###### C10K与C10M相关文章

```text
http://www.kegel.com/c10k.html
http://highscalability.com/blog/2013/5/13/the-secret-to-10-million-concurrent-connections-the-kernel-i.html
```
