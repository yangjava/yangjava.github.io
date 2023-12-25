---
layout: post
categories: [Linux]
description: none
keywords: Linux
---
# 趣谈网络协议栈(五)，Epoll从用户态到内核态过程分析

## poll,select基本过程
将fd信息从用户态copy到内核态(copy_from_user)
注册回调函数 __pollwait
遍历所有fd，调用对应设备的poll方法(sock_poll->tcp_poll, sock_poll->udp_poll)
对sock设备来说，接下来就是调用前面注册的回调函数 __pollwait
将用户态进程挂到设备的等待队列中，不同的设备有不同的等待队列；
设备的poll方法会返回一个描述读写操作是否就绪的mask掩码，如果有非0掩码，就会返回给用户态
如果遍历完所有fd，还没有一个可读写的mask掩码，则调用 poll_schedule_timeout 使用户态进程进入睡眠状态；并注册定时器，定时用户设置的超时时间
设备收到消息(网络设备)或填充完文件数据(磁盘设备)后，会唤醒设备等待队列上睡眠的进程；如果定时器到时也会唤醒进程
将各个设备的返回fd_set从内核空间拷贝到用户空间

## poll,select区别
对应的内核主要功能函数一致
poll使用 struct pollfd 代表一个fd；select 使用 struct fd_set代表所有fd一种事件(读/写/异常)，一个bite位代表一个fd，置1地位，表示该fd该类事件需要监听

## select缺点
不管是poll、select都需要把关心的fd集合从用户态copy到内核态，fd较多时开销较大；特别是poll
每次调用poll、select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大
每次poll、select都要把当前进程放入各个文件描述符的等待队列；在调用结束返回用户态时又要把进程从各个等待队列中删除
select内核默认只支持1024个

## epoll优点
传统的select/poll的一个致命弱点是部分活跃socket，每一次扫描都会线性扫描整个socket集合，导致IO效率随fd数量线性下降
epoll只会经检测活跃的socket，idle状态的socket则不会；相当于实现了一个伪AIO
如果所有socket基本都是活跃的，epoll并不比select/poll效率高，相反由于使用epoll_ctl效率反而下降
epoll使用mmap避免用户态、内核态对fd相关数据的来回copy

## 代码分析
eventpoll_init
```
/*
 * 初始化epoll系统，主要是初始化epitem, epoll_entry 缓冲区
 */
static int __init eventpoll_init(void)
{
	struct sysinfo si;
	/* 获取系统内存相关系统 */
	si_meminfo(&si);
	/* 最大允许4%的低端内存用于epoll */
	max_user_watches = (((si.totalram - si.totalhigh) / 25) << PAGE_SHIFT) /
		EP_ITEM_COST;
	BUG_ON(max_user_watches < 0);

	/* 初始化唤醒结构体？ */
	ep_nested_calls_init(&poll_loop_ncalls);

	/* Initialize the structure used to perform safe poll wait head wake ups */
	ep_nested_calls_init(&poll_safewake_ncalls);

	/* Initialize the structure used to perform file's f_op->poll() calls */
	ep_nested_calls_init(&poll_readywalk_ncalls);

	/*
	 * We can have many thousands of epitems, so prevent this from
	 * using an extra cache line on 64-bit (and smaller) CPUs
	 */
	BUILD_BUG_ON(sizeof(void *) <= 8 && sizeof(struct epitem) > 128);

	/* 申请epitem缓冲区 */
	epi_cache = kmem_cache_create("eventpoll_epi", sizeof(struct epitem),
			0, SLAB_HWCACHE_ALIGN | SLAB_PANIC, NULL);

	/* 申请epoll_entry缓冲区 */
	pwq_cache = kmem_cache_create("eventpoll_pwq",
			sizeof(struct eppoll_entry), 0, SLAB_PANIC, NULL);

	return 0;
}
```
epoll_createl
int epoll_create(int size);
```
/*
 * Open an eventpoll file descriptor.
 * 建立一个 eventpoll 文件描述符，并将其和inode、fd关联
 */
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
	int error, fd;
	struct eventpoll *ep = NULL;
	struct file *file;

	/* Check the EPOLL_* constant for consistency.  */
	BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);
	/* 校验传入参数flags， 目前仅支持 EPOLL_CLOEXEC 一种，如果是其他的，立即返回失败 */
	if (flags & ~EPOLL_CLOEXEC)
		return -EINVAL;
	/* 分配一个 eventpoll 结构体*/
	error = ep_alloc(&ep);
	if (error < 0)
		return error;
	/*
	 * Creates all the items needed to setup an eventpoll file. That is,
	 * a file structure and a free file descriptor.
	 * 从用户态进程管理的文件描述符中获取一个空闲的文件描述符
	 */
	fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
	if (fd < 0) {
		error = fd;
		goto out_free_ep;
	}
	/* 创建一个匿名inode的 struct file，其中会使用 file->private_data = priv
	 * 将第二步创建的 eventpoll 对象赋值给 struct file 的 private_data 成员变量
	 * 匿名inde：可以简单理解为没有对应的dentry，在目录下ls看不到这类文件，close后会自动删除；
	 */
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	if (IS_ERR(file)) {
		error = PTR_ERR(file);
		goto out_free_fd;
	}
	/* 将ep和文件系统的file关联起来 */
	ep->file = file;
	fd_install(fd, file);
	return fd;/* 返回文件描述符 */

out_free_fd:
	put_unused_fd(fd);
out_free_ep:
	ep_free(ep);
	return error;
}
```
epoll_ctl
int epoll_ctl(int epfd, int op, int fd, struct epoll_event)
入参：epfd: epoll_create创建的文件描述符； op: 对新fd的操作码； fd, epoll_event: 要操作的fd，及相应的事件类型
```
/*
 * 将一个fd添加/删除到一个eventpoll中，如果此fd已经在eventpoll中，可以更改其监控事件
 * 1. 将 epoll_event 从用户态copy到内核态(只需copy一次)
 * 2. 先由传入的 epoll fd 和被监听的 socket fd 获取到其对应的文件句柄 struct file， 针对文件句柄和传入的flags做边界条件检测
 * 3. 针对epoll嵌套用法，检测是否有环形epoll监听
 * 4. 针对 EPOLL_CTL_ADD, EPOLL_CTL_DEL, EPOLL_CTL_MOD 分别做处理
 */
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	int error;
	int full_check = 0;
	struct fd f, tf;
	struct eventpoll *ep;
	struct epitem *epi;
	struct epoll_event epds;
	struct eventpoll *tep = NULL;

	error = -EFAULT;
	/* ep_op_has_event() 其实就是判断当前的op不是 EPOLL_CTL_DEL操作
	 * 如果不是删除fd操作，则从用户态拷贝epoll_event 
	 */
	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;

	error = -EBADF;
	/* 检测用户设置的epoll文件是否存在 */
	f = fdget(epfd);
	if (!f.file)
		goto error_return;

	/* 检测要操作的fd文件是否存在 */
	tf = fdget(fd);
	if (!tf.file)
		goto error_fput;

	/* The target file descriptor must support poll 
	 * 被添加的fd必须支持poll方法
	 */
	error = -EPERM;
	if (!tf.file->f_op->poll)
		goto error_tgt_fput;

	/* linux提供了autosleep的电源管理功能
	 * 如果当前系统支持autosleep功能，支持休眠；则允许用户传入 EPOLLWAKEUP 标志；
	 * 否则，从用户传入的数据中去除掉 EPOLLWAKEUP 标志
	 */
	if (ep_op_has_event(op))
		ep_take_care_of_epollwakeup(&epds);

	/* epoll不能监控自己： 如果epoll对应的file结构体和目标fd对应的file是同一个，则异常 */
	error = -EINVAL;
	if (f.file == tf.file || !is_file_epoll(f.file))
		goto error_tgt_fput;

	/* 获取struct eventpoll */
	ep = f.file->private_data;

	/*
	 * When we insert an epoll file descriptor, inside another epoll file
	 * descriptor, there is the change of creating closed loops, which are
	 * better be handled here, than in more critical paths. While we are
	 * checking for loops we also determine the list of files reachable
	 * and hang them on the tfile_check_list, so we can check that we
	 * haven't created too many possible wakeup paths.
	 *
	 * We do not need to take the global 'epumutex' on EPOLL_CTL_ADD when
	 * the epoll file descriptor is attaching directly to a wakeup source,
	 * unless the epoll file descriptor is nested. The purpose of taking the
	 * 'epmutex' on add is to prevent complex toplogies such as loops and
	 * deep wakeup paths from forming in parallel through multiple
	 * EPOLL_CTL_ADD operations.
	 */
	mutex_lock_nested(&ep->mtx, 0);
	if (op == EPOLL_CTL_ADD) {
		if (!list_empty(&f.file->f_ep_links) ||
						is_file_epoll(tf.file)) {
			full_check = 1;
			mutex_unlock(&ep->mtx);
			mutex_lock(&epmutex);
			if (is_file_epoll(tf.file)) {
				error = -ELOOP;
				if (ep_loop_check(ep, tf.file) != 0) {
					clear_tfile_check_list();
					goto error_tgt_fput;
				}
			} else
				list_add(&tf.file->f_tfile_llink,
							&tfile_check_list);
			mutex_lock_nested(&ep->mtx, 0);
			if (is_file_epoll(tf.file)) {
				tep = tf.file->private_data;
				mutex_lock_nested(&tep->mtx, 1);
			}
		}
	}

	/* 寻找fd是否已经在 eventpoll->rbr 树上，如果是则返回 epi!=NULL */
	epi = ep_find(ep, tf.file, fd);

	error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= POLLERR | POLLHUP;
			/* 将当前fd加入红黑树，重点下面讲 */
			error = ep_insert(ep, &epds, tf.file, fd, full_check);
		} else
			error = -EEXIST;
		if (full_check)
			clear_tfile_check_list();
		break;
	case EPOLL_CTL_DEL:
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		if (epi) {
			epds.events |= POLLERR | POLLHUP;
			error = ep_modify(ep, epi, &epds);
		} else
			error = -ENOENT;
		break;
	}
	if (tep != NULL)
		mutex_unlock(&tep->mtx);
	mutex_unlock(&ep->mtx);

error_tgt_fput:
	if (full_check)
		mutex_unlock(&epmutex);

	fdput(tf);
error_fput:
	fdput(f);
error_return:

	return error;
}
```
ep_insert
```
/*
 * Must be called with "mtx" held.
 * init_poll_funcptr：真正将待监听的fd加入到epoll中去
 * 1. 对传入的fd，在内核会创建一个对应的 struct epitem, 并将item插入到 eventpoll.rbr 红黑树当中
 * 2. 在设备驱动tcp_poll中调用poll_table.ep_ptable_queue_proc回调函数；生成一个对应的eppoll_entry，并将 eppoll_entry.wait塞到对应 sock.sk_wq 队列中
 */
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
		     struct file *tfile, int fd, int full_check)
{
	int error, revents, pwake = 0;
	unsigned long flags;
	long user_watches;
	struct epitem *epi;
	struct ep_pqueue epq;
	/* 做MAX_USER_WATCHES检测；内核在 epoll_init() 函数中对所有使用epoll监听fd消耗的内存作了限制 */
	user_watches = atomic_long_read(&ep->user->epoll_watches);
	if (unlikely(user_watches >= max_user_watches))
		return -ENOSPC;
	/* 从epitem缓冲区中申请一个 epitem 管理结构体 */
	if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
		return -ENOMEM;

	/* 初始化 epitem 管理结构体 */
	INIT_LIST_HEAD(&epi->rdllink);
	INIT_LIST_HEAD(&epi->fllink);
	INIT_LIST_HEAD(&epi->pwqlist);
	
	epi->ep = ep;
	ep_set_ffd(&epi->ffd, tfile, fd);
	epi->event = *event;
	epi->nwait = 0;
	epi->next = EP_UNACTIVE_PTR;
	if (epi->event.events & EPOLLWAKEUP) {
	/* 如果events中设置了 EPOLLWAKEUP, 还需要为autosleep创建一个唤醒源 */
		error = ep_create_wakeup_source(epi);
		if (error)
			goto error_create_wakeup_source;
	} else {
		RCU_INIT_POINTER(epi->ws, NULL);
	}

	/* Initialize the poll table using the queue callback
	 * 设置 poll_table._qproc=ep_ptable_queue_proc 回调函数，该回调类似 __pollwait: 
	 * 1. 获取当前被监听的fd上是否有感兴趣的事件发生，同时生成新的 eppoll_entry 对象
	 * 2. 在 eppoll_entry 中设置事件发生回调函数 ep_poll_callback
	 * 3. 将 eppoll_entry.llink 挂到 epitem.pwqlist
	 * 4. 将 eppoll_entry.wait 添加到被监听的socket fd的等待队列中
	 */
	epq.epi = epi;
	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

	/* 检查目标 fd 是否有感兴趣事件发生
	 * revents：事件掩码；在后面会进行处理
	 */
	revents = ep_item_poll(epi, &epq.pt);

	/*
	 * We have to check if something went wrong during the poll wait queue
	 * install process. Namely an allocation for a wait queue failed due
	 * high memory pressure.
	 */
	error = -ENOMEM;
	if (epi->nwait < 0)
		goto error_unregister;

	/* Add the current item to the list of active epoll hook for this file */
	spin_lock(&tfile->f_lock);
	list_add_tail_rcu(&epi->fllink, &tfile->f_ep_links);
	spin_unlock(&tfile->f_lock);

	/* 通过 epitem.rbn ,将epitem链入 eventpoll.rbr 中 */
	ep_rbtree_insert(ep, epi);

	/* now check if we've created too many backpaths */
	error = -EINVAL;
	if (full_check && reverse_path_check())
		goto error_remove_epi;

	/* We have to drop the new item inside our item list to keep track of it */
	spin_lock_irqsave(&ep->lock, flags);

	/* If the file is already "ready" we drop it inside the ready list
	 * 如果要检测的 fd 已经有感兴趣事件发生，则作环形操作
	 * 1. 将 epi 加入到 eventpoll.rdllist 中
	 * 2. 如果当前eventpoll处于wait状态，则唤醒
	 * 3. 如果当前的eventpoll被嵌套地加入到了另外的poll中，且处于wait状态，则唤醒上层poll
	 */
	if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
		list_add_tail(&epi->rdllink, &ep->rdllist);
		ep_pm_stay_awake(epi);

		/* Notify waiting tasks that events are available */
		if (waitqueue_active(&ep->wq))
			wake_up_locked(&ep->wq);
		if (waitqueue_active(&ep->poll_wait))//如果ep->poll_wait非空，则说明本eventpoll被嵌套poll
			pwake++;
	}

	spin_unlock_irqrestore(&ep->lock, flags);
	/* 检视的 fd 数量 +1 */
	atomic_long_inc(&ep->user->epoll_watches);

	/* We have to call this outside the lock */
	if (pwake)
		ep_poll_safewake(&ep->poll_wait);

	return 0;

error_remove_epi:
	spin_lock(&tfile->f_lock);
	list_del_rcu(&epi->fllink);
	spin_unlock(&tfile->f_lock);

	rb_erase(&epi->rbn, &ep->rbr);

error_unregister:
	ep_unregister_pollwait(ep, epi);

	/*
	 * We need to do this because an event could have been arrived on some
	 * allocated wait queue. Note that we don't care about the ep->ovflist
	 * list, since that is used/cleaned only inside a section bound by "mtx".
	 * And ep_insert() is called with "mtx" held.
	 */
	spin_lock_irqsave(&ep->lock, flags);
	if (ep_is_linked(&epi->rdllink))
		list_del_init(&epi->rdllink);
	spin_unlock_irqrestore(&ep->lock, flags);

	wakeup_source_unregister(ep_wakeup_source(epi));

error_create_wakeup_source:
	kmem_cache_free(epi_cache, epi);

	return error;
}
```
epoll_wait
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
/*
 * epoll事件收割机；
 * 对用户传入的参数作校验后，调用ep_poll对 eventpoll 监控
 */
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
		int, maxevents, int, timeout)
{
	int error;
	struct fd f;
	struct eventpoll *ep;

	/* The maximum number of event must be greater than zero */
	if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
		return -EINVAL;

	/* Verify that the area passed by the user is writeable */
	if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event)))
		return -EFAULT;

	/* Get the "struct file *" for the eventpoll file */
	f = fdget(epfd);
	if (!f.file)
		return -EBADF;

	/* 判断传入的 fd 是否为 eventpoll 文件类型*/
	error = -EINVAL;
	if (!is_file_epoll(f.file))
		goto error_fput;

	/* 获取 eventpoll 结构体 */
	ep = f.file->private_data;

	/* 类似select/poll 的 do_poll 函数
	 * error!=0 则为收到的异常信号，见具体代码
	 */
	error = ep_poll(ep, events, maxevents, timeout);

error_fput:
	fdput(f);
	return error;
}
```
ep_poll
```
/**
 * ep_poll - Retrieves ready events, and delivers them to the caller supplied
 *           event buffer.
 *
 * @ep: Pointer to the eventpoll context.被监测的 eventpoll 文件
 * @events: Pointer to the userspace buffer where the ready events should be
 *          stored.用户空间传下来的buffer，回传数据直接赋值在指针指向的地址即可
 * @maxevents: Size (in terms of number of events) of the caller event buffer.
 * 一次最多处理的事件数量
 * @timeout: Maximum timeout for the ready events fetch operation, in
 *           milliseconds. If the @timeout is zero, the function will not block,
 *           while if the @timeout is less than zero, the function will block
 *           until at least one event has been retrieved (or an error
 *           occurred).
 *
 * Returns: Returns the number of ready events which have been fetched, or an
 *          error code, in case of error.ready events数量/错误码
 * 类似 poll/select 的 do_poll 函数；优势在于：
 * epoll不需要逐个遍历所有fd检查是否有事件发生，只需要检查 eventpoll.rdllist 中是否有元素
 * 但是 epoll 支持被多个进程监控，需要将本进程封装进一个 wait_queue_t 中，然后挂到 eventpoll.wq 中，等待被有事件到时被惊醒
 * 1. 根据用户传入超时时间，计算deadline
 * 2. for循环检测 eventpoll;满足3个跳出条件之一，跳出循环
 * 3. ep_send_events：如果是因为事件发生终止循环，则将事件copy到指定地址
 */
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
{
	int res = 0, eavail, timed_out = 0;
	unsigned long flags;
	long slack = 0;
	wait_queue_t wait;
	ktime_t expires, *to = NULL;
	/* 根据用户传入超时时间，计算deadline
	 * 1. 如果用户设置了超时时间，作相应的初始化
	 * 2. 如果timeout=0，表示此次调用立即返回
	 */
	if (timeout > 0) {
		struct timespec end_time = ep_set_mstimeout(timeout);

		slack = select_estimate_accuracy(&end_time);
		to = &expires;
		*to = timespec_to_ktime(end_time);
	} else if (timeout == 0) {
		/*
		 * Avoid the unnecessary trip to the wait queue loop, if the
		 * caller specified a non blocking operation.
		 */
		timed_out = 1;
		spin_lock_irqsave(&ep->lock, flags);
		goto check_events;
	}

fetch_events:
	spin_lock_irqsave(&ep->lock, flags);
	
	if (!ep_events_available(ep)) {
		/*
		 * We don't have any available event to return to the caller.
		 * We need to sleep here, and we will be wake up by
		 * ep_poll_callback() when events will become available.
		 * 如果现在本 eventpoll 中没有时间发生，则用户进程睡眠等待
		 * 等待事件发生后被 ep_poll_callback 唤醒
		 */
		init_waitqueue_entry(&wait, current);
		/* 针对同一个 eventpoll, 可能在不同的进程(线程)调用 epoll_wait
		 * 此时 eventpoll 的等待队列中将会有多个task，为避免惊群，每次只唤醒一个 task */
		__add_wait_queue_exclusive(&ep->wq, &wait);
		/* 该过程类似 select/poll 的for循环；不同的是
		 * select/poll 每一次for循环要遍历一遍所有的fd，然后再sleep睡眠等待超时
		 * epoll 只需要检查 eventpoll.rdllist 中是否有元素即可
		 */
		for (;;) {
			/*
			 * We don't want to sleep if the ep_poll_callback() sends us
			 * a wakeup in between. That's why we set the task state
			 * to TASK_INTERRUPTIBLE before doing the checks.
			 */
			set_current_state(TASK_INTERRUPTIBLE);
			/* 1. 有事件发生； 2. 超时 */
			if (ep_events_available(ep) || timed_out)
				break;
			/* 3. 有signal发生，被中断后退出 */
			if (signal_pending(current)) {
				res = -EINTR;
				break;
			}

			spin_unlock_irqrestore(&ep->lock, flags);
			/* 当前task被调度走；如果到达deadline、没有别的紧急任务又会切换回来 */
			if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
				timed_out = 1;

			spin_lock_irqsave(&ep->lock, flags);
		}
		/* 唤醒后将本进程从 eventpoll task等待队列中删除 */
		__remove_wait_queue(&ep->wq, &wait);
		/* 用户态进程被激活后，修改状态标志 */
		__set_current_state(TASK_RUNNING);
	}
check_events:
	/* 检查是否有事件发生 */
	eavail = ep_events_available(ep);

	spin_unlock_irqrestore(&ep->lock, flags);

	/*
	 * 如果不是由于信号中断退出，且有事件发生；尝试将事件copy到用户态内存空间
	 * ep_send_events->ep_scan_ready_list：将ready事件传到用户态；其中关键回调函数： ep_send_events_proc
	 */
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
		goto fetch_events;/* 如果既不是信号中断，也没有事件发生，也没有超时，则继续监控 */

	return res;
}
```
ep_scan_ready_list
```
/**
 * ep_scan_ready_list - Scans the ready list in a way that makes possible for
 *                      the scan code, to call f_op->poll(). Also allows for
 *                      O(NumReady) performance.
 * 扫描 eventpoll.rdllist 并使用传入的扫描函数，对链表中每个元素进行处理
 * @ep: Pointer to the epoll private data structure.
 * @sproc: Pointer to the scan callback. 扫描函数
 * @priv: Private opaque data passed to the @sproc callback.传给扫描函数的入参；用户透传的epoll_event空间地址、大小
 * @depth: The current depth of recursive f_op->poll calls. 对 eventpoll 循环嵌套情况最多扫描层数？
 * @ep_locked: caller already holds ep->mtx
 * 
 * 1. 将 rdllist 中元素转移到临时链表txlist中，便于回调函数逐个处理
 * 2. 使用回调函数处理事件
 * 3. 将挂在ovlist上回调处理过程中发生的事件再转移到 rdllist上
 * 4. 将未能写到用户空间的 txlist 合并回 rdllist
 * 5. 唤醒用户态线程
 */
static int ep_scan_ready_list(struct eventpoll *ep,
			      int (*sproc)(struct eventpoll *,
					   struct list_head *, void *),
			      void *priv, int depth, bool ep_locked)
{
	int error, pwake = 0;
	unsigned long flags;
	struct epitem *epi, *nepi;
	LIST_HEAD(txlist);

	/*
	 * We need to lock this because we could be hit by
	 * eventpoll_release_file() and epoll_ctl().
	 * 如果上层调用没有加锁，则函数内需要加锁互斥
	 */
	if (!ep_locked)
		mutex_lock_nested(&ep->mtx, depth);

	/*
	 * Steal the ready list, and re-init the original one to the
	 * empty list. Also, set ep->ovflist to NULL so that events
	 * happening while looping w/out locks, are not lost. We cannot
	 * have the poll callback to queue directly on ep->rdllist,
	 * because we want the "sproc" callback to be able to do it
	 * in a lockless way.
	 */
	spin_lock_irqsave(&ep->lock, flags);
	/* 将 eventpoll.rdllist 中元素转移到 txlist 链表上，并初始化 rdllist
	 * 原因注释说明很清楚了，在回调函数中，将会不加锁处理链表上元素
	 */
	list_splice_init(&ep->rdllist, &txlist);
	/* 去除 ovflist 链表的 EP_UNACTIVE_PTR 状态，用于在回调期间，如果有事件上来，暂时保存 */
	ep->ovflist = NULL;
	spin_unlock_irqrestore(&ep->lock, flags);

	/* 调用回调函数，对 txlist的每个epi进行poll检测状态
	 * 如果满足状态
	 * a. 发送到用户空间
	 * b. 将非EPOLLET的epi重新链入 rdllist(ET,LT关键区别点在这)
	 * 对于LT模式的epi，下次还得通过poll进行筛选，及时也许你已经将文件的读缓冲读完了
	 */
	error = (*sproc)(ep, &txlist, priv);

	spin_lock_irqsave(&ep->lock, flags);
	/*
	 * During the time we spent inside the "sproc" callback, some
	 * other events might have been queued by the poll callback.
	 * We re-insert them inside the main ready-list here.
	 * 将 ovflist 上的元素转移到 rdllist 上
	 */
	for (nepi = ep->ovflist; (epi = nepi) != NULL;
	     nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {
	/* 把sproc执行期间产生的事件加入到 rdllist 中；但有可能这些新诞生的事件到目前位置还在 txlist 中
	 * 1. 去重，通过 ep_is_linked() 判断是否已经在txlist中，如果在 epi.rdllikn 非空
	 * 2. 如果新发生的事件没有在 txlist 中，则放入 rdllist 中；并保证这些事件在新诞生的事件前面
		if (!ep_is_linked(&epi->rdllink)) {
			list_add_tail(&epi->rdllink, &ep->rdllist);
			ep_pm_stay_awake(epi);
		}
	}
	/*
	 * We need to set back ep->ovflist to EP_UNACTIVE_PTR, so that after
	 * releasing the lock, events will be queued in the normal way inside
	 * ep->rdllist.
	 * ovflist被置上 EP_UNACTIVE_PTR 后，不再接收新元素
	 */
	ep->ovflist = EP_UNACTIVE_PTR;

	/*
	 * Quickly re-inject items left on "txlist".
	 * 将未能处理的元素重新挂回rdllist
	 */
	list_splice(&txlist, &ep->rdllist);
	__pm_relax(ep->ws);

	if (!list_empty(&ep->rdllist)) {
		/*
		 * Wake up (if active) both the eventpoll wait list and
		 * the ->poll() wait list (delayed after we release the lock).
		 */
		if (waitqueue_active(&ep->wq))
		/* 唤醒 eventpoll.wq 中的等待线程；该函数是防惊群唤醒
		 * wake_up_all 则是唤醒所有的线程
		 */
			wake_up_locked(&ep->wq);
		if (waitqueue_active(&ep->poll_wait))
			pwake++;
	}
	spin_unlock_irqrestore(&ep->lock, flags);

	if (!ep_locked)
		mutex_unlock(&ep->mtx);

	/* We have to call this outside the lock */
	if (pwake)
		ep_poll_safewake(&ep->poll_wait);

	return error;
}
```
ep_send_events_proc
```
/* 循环获取监听项的事件数据，使用 ep_item_poll 获取监听到的目标文件事件，将发生的事件events复制到用户空间
 * priv：结构体指针，其中包含用户透传的epoll_event空间地址、大小
 */
static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
			       void *priv)
{
	struct ep_send_events_data *esed = priv;
	int eventcnt;
	unsigned int revents;
	struct epitem *epi;
	struct epoll_event __user *uevent;
	struct wakeup_source *ws;
	poll_table pt;

	init_poll_funcptr(&pt, NULL);

	/* 遍历 rdllist 元素； maxevents是用户传入的一次最多处理事件个数 */
	for (eventcnt = 0, uevent = esed->events;
	     !list_empty(head) && eventcnt < esed->maxevents;) {
		epi = list_first_entry(head, struct epitem, rdllink);

		/*
		 * Activate ep->ws before deactivating epi->ws to prevent
		 * triggering auto-suspend here (in case we reactive epi->ws
		 * below).
		 *
		 * This could be rearranged to delay the deactivation of epi->ws
		 * instead, but then epi->ws would temporarily be out of sync
		 * with ep_is_linked().
		 */
		ws = ep_wakeup_source(epi);
		if (ws) {
			if (ws->active)
				__pm_stay_awake(ep->ws);
			__pm_relax(ws);
		}
		/* 将要被处理的 epitem 从 txlist中移除
		 * 1. 因为并不是所有 txlist 上的元素都会被处理，剩下的没处理的会放回 rdllist 中
		 */
		list_del_init(&epi->rdllink);
		/* 获取发生的事件; 
		 * 通过驱动注册的poll函数(sock_poll->tcp_poll)查询发生的事件，并和用户感兴趣的事件对比 */
		revents = ep_item_poll(epi, &pt);

		/*
		 * If the event mask intersect the caller-requested one,
		 * deliver the event to userspace. Again, ep_scan_ready_list()
		 * is holding "mtx", so no operations coming from userspace
		 * can change the item.
		 */
		if (revents) {
			if (__put_user(revents, &uevent->events) ||
			    __put_user(epi->event.data, &uevent->data)) {
				list_add(&epi->rdllink, head);
				ep_pm_stay_awake(epi);
				return eventcnt ? eventcnt : -EFAULT;
			}
			eventcnt++;
			uevent++;
			if (epi->event.events & EPOLLONESHOT)
				epi->event.events &= EP_PRIVATE_BITS;
			else if (!(epi->event.events & EPOLLET)) {
				/* ET,LT的唯一区别；处理过的 epitem 是否要塞回 rdlist; 下一次继续轮询 
				 * 那对于LT什么时候会将事件剔除出rdllist：
				 * 当用户态代码一直读取这个fd上的数据直到 EGAIN, 下次epoll_wait时仍然会从rdllist中碰到该事件；但此时 tcp_poll 不会返回该事件可读了，即会被剔除
				 */
				list_add_tail(&epi->rdllink, &ep->rdllist);
				ep_pm_stay_awake(epi);
			}
		}
	}

	return eventcnt;
}
```
eventpoll
```
/*
 * This structure is stored inside the "private_data" member of the file
 * structure and represents the main data structure for the eventpoll
 * interface.
 */
struct eventpoll {
	/* Protect the access to this structure */
	spinlock_t lock;
	/* 用于锁定这个eventpoll数据结构
	 * 在用户空间多线程操作这个poll结构，比如调用epoll_ctl作add,mod,del时，用户空间不需要加锁
	 */
	struct mutex mtx;

	/* sys_epoll_wait()使用的等待队列
	 * 在 epoll_wait 时如果当前没有拿到有效事件，则将当前task加入这个等待队列后作进程切换，等待被唤醒
	 */
	wait_queue_head_t wq;

	/* file->poll() 使用的等待队列
	 * eventpoll对象在使用时都会对应一个 struct file 对象，赋值到 private_data
	 * 其本身也可以被poll，那也就需要一个 wait queue
	 * 有了这个变量；则eventpoll对应的 struct file 也可以被 poll；则可以将这个 epoll fd加入到另一个 epoll fd中，实现epoll嵌套
	 */
	wait_queue_head_t poll_wait;

	/* 所有有事件触发的被监控的fd都会加入到这个链表 */
	struct list_head rdllist;

	/* 用于管理所有被监控fd的红黑树 */
	struct rb_root rbr;

	/* 当将ready的fd复制到用户进程中，会使用上面的lock锁锁定rdllist
	 * 此时如果有新的ready状态fd，则临时加入到 ovflist 表示的单链表中
	 */
	struct epitem *ovflist;

	/* 会autosleep准备的唤醒源 */
	struct wakeup_source *ws;

	/* 保存了一些用户变量，比如fd监听数量的最大值等 */
	struct user_struct *user;
	/* 一切皆文件，epoll实例被创建时，同时会创建一个file，file的private_data指向当前这个eventpoll结构 */
	struct file *file;

	/* used to optimize loop detection check */
	int visited;
	struct list_head visited_list_link;
}
```
epitem
```
/*
 * 每一个被 epoll 监控的句柄都会保存在 eventpoll 内部的红黑树上(eventpoll->rbr)
 * ready状态的句柄也会保存在 eventpoll 内部的一个链表上 eventpoll->rdllist
 * 实现时会将每个句柄封装在一个 struct epitem 中；即一个 epitem 对应一个 被监视fd
 */
struct epitem {
	union {
		/* 该成员会链入 eventpoll 的红黑树中 */
		struct rb_node rbn;
		/* Used to free the struct epitem */
		struct rcu_head rcu;
	};
	/* 链表节点，所有已经ready的epitem都会被链到 eventpoll->rdllist 中 */
	struct list_head rdllink;
	/* 用于将当前 epitem 链接到 eventpoll->ovflist 单链中 */
	struct epitem *next;
	/* 对应被监听的文件描述符信息 */
	struct epoll_filefd ffd;
	/* poll操作中事件的个数 */
	int nwait;
	/* 双向链表，保存被监视文件的等待队列，类似于 select/poll 中的 poll_table */
	struct list_head pwqlist;
	/* 指向归属的 eventpoll */
	struct eventpoll *ep;
	/* 双向链表，用来链接被监视文件描述符对应的struct file ; 因为file里有 f_ep_links , 用来保存所有监视这个文件的epoll节点 */
	struct list_head fllink;
	/* wakeup_source used when EPOLLWAKEUP is set */
	struct wakeup_source __rcu *ws;
	/* 用户态传下来的数据，保存文件fd编号以及感兴趣的事件 */
	struct epoll_event event;
}
```
epoll_event
```
/*
 * epoll_ctl 从用户态传给内核态参数，告诉内核需要监控哪些事件
 */
struct epoll_event {
	__u32 events;
	__u64 data;
} EPOLL_PACKED
```
eppoll_entry
```
/* Wait structure used by the poll hooks 
 * 类似select/poll的 poll_table_entry;作为挂到驱动等待队列中的钩子元素
 */
struct eppoll_entry {
	/* List header used to link this structure to the "struct epitem" */
	struct list_head llink;

	/* The "base" pointer is set to the container "struct epitem" */
	struct epitem *base;

	/*
	 * Wait queue item that will be linked to the target file wait
	 * queue head.
	 */
	wait_queue_t wait;

	/* The wait queue head that linked the "wait" wait queue item */
	wait_queue_head_t *whead;
};
```