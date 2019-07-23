---
layout: post
title: "Linux惊群效应之Nginx解决方案"
date: 2019-07-23
description: "Linux惊群效应之Nginx解决方案"
tag: 网络
---

### 前言

项目涉及到Nginx一些公共模块的使用，而且也想对惊群效应有个深入的了解，在整理了网上资料以及实践后，记录成文章以便复习巩固。

### 目录

* [结论](#chapter1)
* [惊群效应是什么](#chapter2)
* [惊群效应消耗了什么](#chapter3)
* [Linux解决方案之Accept](#chapter4)
* [Linux解决方案之Epoll](#chapter5)
* [Nginx解决方案之锁的设计](#chapter6)
* [Nginx解决方案之惊群效应](#chapter7)

### <a name="chapter1"></a>结论

* 不管是多进程还是多线程，都存在惊群效应，本篇文章使用多进程分析。
* 在Linux2.6版本之后，已经解决了系统调用accept的惊群效应（前提是没有使用select、poll、epoll等事件通知机制）。
* 目前Linux已经部分解决了epoll的惊群效应（epoll在fork之前），Linux2.6是没有解决的。
* epoll在fork之后创建仍然存在惊群效应，Nginx使用自己实现的互斥锁解决惊群效应。

### <a name="chapter2"></a>惊群效应是什么

`惊群效应`（thundering herd）是指多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的这个事件发生，那么他就会唤醒等待的所有进程（或者线程），但是最终却只能有一个进程（线程）获得这个时间的“控制权”，对该事件进行处理，而其他进程（线程）获取“控制权”失败，只能重新进入休眠状态，这种现象和性能浪费就叫做惊群效应。

### <a name="chapter3"></a>惊群效应消耗了什么

* Linux内核对用户进程（线程）频繁地做无效的调度、上下文切换等使系统性能大打折扣。上下文切换（context switch）过高会导致cpu像个搬运工，频繁地在寄存器和运行队列之间奔波，更多的时间花在了进程（线程）切换，而不是在真正工作的进程（线程）上面。直接的消耗包括cpu寄存器要保存和加载（例如程序计数器）、系统调度器的代码需要执行。间接的消耗在于多核cache之间的共享数据。
* 为了确保只有一个进程（线程）得到资源，需要对资源操作进行加锁保护，加大了系统的开销。目前一些常见的服务器软件有的是通过`锁机制`解决的，比如Nginx（它的锁机制是默认开启的，可以关闭）；还有些认为惊群对系统性能影响不大，没有去处理，比如Lighttpd。

### <a name="chapter4"></a>Linux解决方案之Accept

* Linux 2.6版本之前，监听同一个socket的进程会挂在同一个等待队列上，当请求到来时，会唤醒所有等待的进程。 
* Linux 2.6版本之后，通过引入一个标记位`WQ_FLAG_EXCLUSIVE`，解决掉了Accept惊群效应。

**具体分析会在代码注释里面，accept代码实现片段如下：**

    // 当accept的时候，如果没有连接则会一直阻塞（没有设置非阻塞）
    // 其阻塞函数就是：inet_csk_accept（accept的原型函数）  
	struct sock *inet_csk_accept(struct sock *sk, int flags, int *err)
	{
    	...  
    	// 等待连接 
    	error = inet_csk_wait_for_connect(sk, timeo); 
    	...  
	}

	static int inet_csk_wait_for_connect(struct sock *sk, long timeo)
	{
    	...
    	for (;;) {  
        	// 只有一个进程会被唤醒。
        	// 非exclusive的元素会加在等待队列前头，exclusive的元素会加在所有非exclusive元素的后头。
        	prepare_to_wait_exclusive(sk_sleep(sk), &wait,TASK_INTERRUPTIBLE);  
    	}  
    	...
	}

	void prepare_to_wait_exclusive(wait_queue_head_t *q, wait_queue_t *wait, int state)  
	{
    	unsigned long flags;  
    	// 设置等待队列的flag为EXCLUSIVE，设置这个就是表示一次只会有一个进程被唤醒，我们等会就会看到这个标记的作用。  
    	// 注意这个标志，唤醒的阶段会使用这个标志。
    	wait->flags |= WQ_FLAG_EXCLUSIVE;  
    	spin_lock_irqsave(&q->lock, flags);  
    	if (list_empty(&wait->task_list))  
    	// 加入等待队列  
		__add_wait_queue_tail(q, wait);  
    	set_current_state(state);  
    	spin_unlock_irqrestore(&q->lock, flags);  
	}

**唤醒阻塞的accept代码片段如下：**

	// 当有tcp连接完成，就会从半连接队列拷贝socket到连接队列，这个时候我们就可以唤醒阻塞的accept了。
	int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
	{
    	...
    	// 关注此函数
    	if (tcp_child_process(sk, nsk, skb)) { 
        	rsk = nsk;  
        	goto reset;  
    	}
    	...
	}

	int tcp_child_process(struct sock *parent, struct sock *child, struct sk_buff *skb)
	{
    	...
    	// Wakeup parent, send SIGIO 唤醒父进程
    	if (state == TCP_SYN_RECV && child->sk_state != state)  
        	// 调用sk_data_ready通知父进程
        	// 查阅资料我们知道tcp中这个函数对应是sock_def_readable
        	// 而sock_def_readable会调用wake_up_interruptible_sync_poll来唤醒队列
        	parent->sk_data_ready(parent, 0);  
    	}
    	...
	}

	void __wake_up_sync_key(wait_queue_head_t *q, unsigned int mode, int nr_exclusive, void *key)  
	{  
    	...  
    	// 关注此函数
    	__wake_up_common(q, mode, nr_exclusive, wake_flags, key);  
    	spin_unlock_irqrestore(&q->lock, flags);  
    	...  
	} 

	static void __wake_up_common(wait_queue_head_t *q, unsigned int mode, int nr_exclusive, int wake_flags, void *key)
	{
    	...
    	// 传进来的nr_exclusive是1
    	// 所以flags & WQ_FLAG_EXCLUSIVE为真的时候，执行一次，就会跳出循环
    	// 我们记得accept的时候，加到等待队列的元素就是WQ_FLAG_EXCLUSIVE的
    	list_for_each_entry_safe(curr, next, &q->task_list, task_list) {  
        	unsigned flags = curr->flags;  
        	if (curr->func(curr, mode, wake_flags, key) 
        	&& (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
        	break; 
    	}
    	...
	}

### <a name="chapter5"></a>Linux解决方案之Epoll

在使用select、poll、epoll、kqueue等IO复用时，多进程（线程）处理链接更加复杂。

因此在讨论epoll的惊群效应时候，需要分为两种情况：

	epoll_create在fork之前创建
	epoll_create在fork之后创建

#### epoll_create在fork之前创建

与accept惊群的原因类似，当有事件发生时，等待同一个文件描述符的所有进程（线程）都将被唤醒，而且解决思路和accept一致。

为什么需要全部唤醒？因为内核不知道，你是否在等待文件描述符来调用accept()函数，还是做其他事情（信号处理，定时事件）。

`此种情况惊群效应已经被解决。`

#### epoll_create在fork之后创建

epoll_create在fork之前创建的话，所有进程共享一个epoll红黑数。 

如果我们只需要处理accept事件的话，貌似世界一片美好了。但是epoll并不是只处理accept事件，accept后续的读写事件都需要处理，还有定时或者信号事件。

当连接到来时，我们需要选择一个进程来accept，这个时候，任何一个accept都是可以的。当连接建立以后，后续的读写事件，却与进程有了关联。一个请求与a进程建立连接后，后续的读写也应该由a进程来做。

当读写事件发生时，应该通知哪个进程呢？epoll并不知道，因此，事件有可能错误通知另一个进程，这是不对的。所以一般在每个进程（线程）里面会再次创建一个epoll事件循环机制，每个进程的读写事件只注册在自己进程的epoll种。

我们知道`epoll对惊群效应的修复，是建立在共享在同一个epoll结构上的`。epoll_create在fork之后执行，每个进程有单独的epoll红黑树，等待队列，ready事件列表。因此，惊群效应再次出现了。有时候唤醒所有进程，有时候唤醒部分进程，可能是因为事件已经被某些进程处理掉了，因此不用在通知另外还未通知到的进程了。

### <a name="chapter6"></a>Nginx解决方案之锁的设计

首先我们要知道在用户空间进程间锁实现的原理，起始原理很简单，就是能弄一个让所有进程共享的东西，比如mmap的内存，比如文件，然后通过这个东西来控制进程的互斥。

Nginx中使用的锁是自己来实现的，这里锁的实现分为两种情况，一种是支持原子操作的情况，也就是由`NGX_HAVE_ATOMIC_OPS`这个宏来进行控制的，一种是不支持原子操作，这是是使用文件锁来实现。

#### 锁结构体

如果支持原子操作，则我们可以直接使用mmap，然后lock就保存mmap的内存区域的地址。

如果不支持原子操作，则我们使用文件锁来实现，这里fd表示进程间共享的文件句柄，name表示文件名。

	typedef struct {  
		#if (NGX_HAVE_ATOMIC_OPS)  
	   		ngx_atomic_t  *lock;  
		#else  
	    	ngx_fd_t       fd;  
	    	u_char        *name;  
		#endif  
	} ngx_shmtx_t;

#### 原子锁创建

	// 如果支持原子操作的话，非常简单，就是将共享内存的地址付给loc这个域
	ngx_int_t ngx_shmtx_create(ngx_shmtx_t *mtx, void *addr, u_char *name)  
	{  
		mtx->lock = addr;
		return NGX_OK;  
	} 

#### 原子锁获取

`trylock`它是非阻塞的，也就是说它会尝试的获得锁，如果没有获得的话，它会直接返回错误。


`lock`它也会尝试获得锁，而当没有获得他不会立即返回，而是开始进入循环然后不停的去获得锁，知道获得。不过Nginx这里还有用到一个技巧，就是每次都会让当前的进程放到cpu的运行队列的最后一位，也就是自动放弃cpu。

#### 原子锁实现

如果系统库支持的情况，此时直接调用`OSAtomicCompareAndSwap32Barrier`即CAS。
	
	#define ngx_atomic_cmp_set(lock, old, new)                                   
	    OSAtomicCompareAndSwap32Barrier(old, new, (int32_t *) lock)
	
如果系统库不支持这个指令的话，Nginx自己还用汇编实现了一个。
	
	static ngx_inline ngx_atomic_uint_t ngx_atomic_cmp_set(ngx_atomic_t *lock, ngx_atomic_uint_t old,  
	    ngx_atomic_uint_t set)  
	{  
		u_char  res;  
	
	    __asm__ volatile (  
	
	         NGX_SMP_LOCK  
	    "    cmpxchgl  %3, %1;   "  
	    "    sete      %0;       "  
	
	    : "=a" (res) : "m" (*lock), "a" (old), "r" (set) : "cc", "memory");  
	
	    return res;  
	}

#### 原子锁释放

unlock比较简单，和当前进程id比较，如果相等，就把lock改为0,说明放弃这个锁。
	
	#define ngx_shmtx_unlock(mtx) 
		(void) ngx_atomic_cmp_set((mtx)->lock, ngx_pid, 0)  

### <a name="chapter7"></a>Nginx解决方案之惊群效应

#### 变量分析

	 // 如果使用了master worker，并且worker个数大于1
	 // 同时配置文件里面有设置使用accept_mutex.的话，设置ngx_use_accept_mutex  
	 if (ccf->master && ccf->worker_processes > 1 && ecf->accept_mutex) 
	 { 
	 	ngx_use_accept_mutex = 1;  
		// 下面这两个变量后面会解释。  
		ngx_accept_mutex_held = 0;  
		ngx_accept_mutex_delay = ecf->accept_mutex_delay;  
	 } else {  
	   	ngx_use_accept_mutex = 0;  
	 }
	
	ngx_use_accept_mutex:
	这个变量，如果有这个变量，说明nginx有必要使用accept互斥体，这个变量的初始化在ngx_event_process_init中。
	
	ngx_accept_mutex_held:
	表示当前是否已经持有锁。
	
	ngx_accept_mutex_delay:
	表示当获得锁失败后，再次去请求锁的间隔时间，这个时间可以在配置文件中设置的。
	
	ngx_accept_disabled = ngx_cycle->connection_n / 8  
	                              - ngx_cycle->free_connection_n;
	ngx_accept_disabled:
	这个变量是一个阈值，如果大于0,说明当前的进程处理的连接过多。

#### 是否使用锁

	// 如果有使用mutex，则才会进行处理。  
	if (ngx_use_accept_mutex) 
	{  
	    // 如果大于0,则跳过下面的锁的处理，并减一。  
	    if (ngx_accept_disabled > 0) {  
	        ngx_accept_disabled--; 
	    } else {  
	        // 试着获得锁，如果出错则返回。  
	        if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {  
	            return;  
	        }  
	        // 如果ngx_accept_mutex_held为1,则说明已经获得锁，此时设置flag，这个flag后面会解释。
	        if (ngx_accept_mutex_held) {  
	            flags |= NGX_POST_EVENTS;  
	        } else {  
	            // 否则，设置timer，也就是定时器。接下来会解释这段。  
	            if (timer == NGX_TIMER_INFINITE  
	                 || timer > ngx_accept_mutex_delay) {  
	                timer = ngx_accept_mutex_delay;  
	            }  
	        }  
	    }  
	}

	// 如果ngx_posted_accept_events不为NULL，则说明有accept event需要nginx处理。  
	if (ngx_posted_accept_events) {  
	        ngx_event_process_posted(cycle, &ngx_posted_accept_events);  
	}
	
`NGX_POST_EVENTS`标记，设置了这个标记就说明当socket有数据被唤醒时，我们并不会马上accept或者说读取，而是将这个事件保存起来，然后当我们释放锁之后，才会进行accept或者读取这个句柄。

如果没有设置`NGX_POST_EVENTS`标记的话，nginx会立即accept或者读取句柄。
	
`定时器`这里如果nginx没有获得锁，并不会马上再去获得锁，而是设置定时器，然后在epoll休眠(如果没有其他的东西唤醒).此时如果有连接到达，当前休眠进程会被提前唤醒，然后立即accept。否则，休眠 `ngx_accept_mutex_delay`时间，然后继续try lock。

#### 获取锁来解决惊群

	ngx_int_t ngx_trylock_accept_mutex(ngx_cycle_t *cycle)  
	{ 
	    // 尝试获得锁  
	    if (ngx_shmtx_trylock(&ngx_accept_mutex)) {  
	        // 如果本来已经获得锁，则直接返回Ok  
	        if (ngx_accept_mutex_held  
	            && ngx_accept_events == 0  
	            && !(ngx_event_flags & NGX_USE_RTSIG_EVENT))  
	        {  
	            return NGX_OK;  
	        }  
	
	        // 到达这里，说明重新获得锁成功，因此需要打开被关闭的listening句柄。  
	        if (ngx_enable_accept_events(cycle) == NGX_ERROR) {  
	            ngx_shmtx_unlock(&ngx_accept_mutex);  
	            return NGX_ERROR;  
	        }  
	
	        ngx_accept_events = 0;  
	        // 设置获得锁的标记。  
	        ngx_accept_mutex_held = 1;  
	
	        return NGX_OK;  
	    } 
	
	    // 如果我们前面已经获得了锁，然后这次获得锁失败
	    // 则说明当前的listen句柄已经被其他的进程锁监听
	    // 因此此时需要从epoll中移出调已经注册的listen句柄
	    // 这样就很好的控制了子进程的负载均衡  
	    if (ngx_accept_mutex_held) {  
	        if (ngx_disable_accept_events(cycle) == NGX_ERROR) {  
	            return NGX_ERROR;  
	        } 
	        // 设置锁的持有为0.  
	        ngx_accept_mutex_held = 0;  
	    }  
	
	    return NGX_OK;  
	} 

如上代码，当一个连接来的时候，此时每个进程的epoll事件列表里面都是有该fd的。抢到该连接的进程先释放锁，在accept。没有抢到的进程把该fd从事件列表里面移除，不必再调用accept，造成资源浪费。 
同时由于锁的控制(以及获得锁的定时器)，每个进程都能相对公平的accept句柄，也就是比较好的解决了子进程负载均衡。

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [Linux惊群效应之Nginx解决方案](https://shimingyah.github.io/2019/07/Linux%E6%83%8A%E7%BE%A4%E6%95%88%E5%BA%94%E4%B9%8BNginx%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/)