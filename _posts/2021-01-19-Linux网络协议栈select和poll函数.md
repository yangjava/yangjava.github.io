---
layout: post
categories: [Linux]
description: none
keywords: Linux
---
# 趣谈网络协议栈(四)，学习select和poll函数的内核实现
学习select，poll函数的内核实现(本质来说，这两个函数的内核实现是一致的，select应该是poll的优化版本)；使用linux 4.1.10版本的代码分析; 主要内容如下：
注册监听；
将发生的事件返回给用户态；

## select缺点
每次调用select,都需要把fd集合从用户态copy到内核态，如果fd较多，则开销很大
每次调用select都需要在内核遍历传递进来的所有fd，fd多的情况下，遍历一轮时间较长
select默认支持的描述符1024个，相对可能不够
通过inet_init()注册了inet_family_ops到域协议族全局结构体中；其中关键函数是 inet_create

## poll注册监听流程
do_sys_poll
SYSCALL_DEFINE3(poll, struct pollfd __user *, ufds, unsigned int, nfds,int, timeout_msecs) —> do_sys_poll
```
/* 
 * 将用户态下传的所有 pollfd 拷贝到内核态，这些pollfd通过 poll_list 组织起来
 * poll_initwait：初始化 poll_wqueues,关键是注册__pollwait回调函数，以及将用户态进程赋予 poll_wqueues.polling_task
 * do_poll: 注册监听任务，原材料为 poll_list, poll_wqueues
 * 如果列表中fd有事件发生，或者超时则返回用户态，并将各个 pollfd->revents 拷贝回用户态
 */
static int do_sys_poll(struct pollfd __user *ufds, unsigned int nfds,
		struct timespec64 *end_time)
{
	/* 监听关键结构体，联系用户进程和驱动设备
	 * poll_wqueues.polling_task: 记录监听fd的用户进程
	 * poll_wqueues.inline_entries/table: 管理本poll关心的所有设备
	 */
	struct poll_wqueues table;
 	int err = -EFAULT, fdcount, len, size;
	/* 为了加快处理速度和提高系统性能，这里优先使用已经定好的一个栈空间，其大小为POLL_STACK_ALLOC */
	long stack_pps[POLL_STACK_ALLOC/sizeof(long)];
	/* stack_pps空间分为两部分，前半部分为一个 struct poll_list管理结构体; 后半部分存 struct pollfd */
	struct poll_list *const head = (struct poll_list *)stack_pps;
 	struct poll_list *walk = head;
 	unsigned long todo = nfds;

	if (nfds > rlimit(RLIMIT_NOFILE))
		return -EINVAL;
	/* stack_pps后半部分能存多少个 struct pollfd 
	 * N_STACK_PPS: 计算最多可以存多少个 struct pollfd
	 * nfds: 用户设置，代表目前ufds中有几个需要监控，即真实需要多少个
	 */
	len = min_t(unsigned int, nfds, N_STACK_PPS);
	/* 循环是为了解决预留的POLL_STACK_ALLOC空间不够储存所有的 pollfd */
	for (;;) {
		walk->next = NULL;
		walk->len = len;/* 将长度赋予poll_list.len */
		if (!len)
			break;
		/* 每次从用户态，cp len 个 pollfd到 walk->entries中 */
		if (copy_from_user(walk->entries, ufds + nfds-todo,
					sizeof(struct pollfd) * walk->len))
			goto out_fds;
		/* 将游标移动到下一组 pollfd */
		todo -= walk->len;
		if (!todo)
			break;/* 如果cp完了，则跳出循环 */
		/* 计算一页最多能存多少个 pollfd */
		len = min(todo, POLLFD_PER_PAGE);
		size = sizeof(struct poll_list) + sizeof(struct pollfd) * len;
		/* 通过链表链接 */
		walk = walk->next = kmalloc(size, GFP_KERNEL);
		if (!walk) {
			err = -ENOMEM;
			goto out_fds;
		}
	}
	/* 目前为止形成一个以 stack_pps 存储空间为头，然后一页一页分配的内存为接点的链表
	 * 这个链表上就存储了 poll 调用时传入的所有的socket描述符 */
	 
	/* 初始化 poll_wqueues,关键是注册__pollwait回调函数，以及将用户态进程赋予 poll_wqueues.polling_task
	 * 建立 poll_wqueues 结构体和用户态进程的映射
	 */
	poll_initwait(&table);
	/* 设置监听，并循环检测各个fd是否有事件发生；返回发生事件fd的个数 */
	fdcount = do_poll(head, &table, end_time);
	poll_freewait(&table);
	/* 发生事件、时间到点则将 pollfd.revents 拷贝回用户态，并返回 */
	for (walk = head; walk; walk = walk->next) {
		struct pollfd *fds = walk->entries;
		int j;
	
		for (j = 0; j < walk->len; j++, ufds++)
			if (__put_user(fds[j].revents, &ufds->revents))
				goto out_fds;
  	}

	err = fdcount;
out_fds:
	walk = head->next;
	while (walk) {
		struct poll_list *pos = walk;
		walk = walk->next;
		kfree(pos);
	}
	/* 返回用户态 */
	return err;
}
```
do_poll
```
/* 执行对 poll_list 中所有fd的监听
 * 1. do_pollfd： 对各个fd设置监听；返回fd对应的事件掩码，如果有关心的事件发生，则 mask!=0
 * 2. 完成对所有fd的挂载后；如果有fd返回的mask不为0、超时则返回用户态
 * 3. poll_schedule_timeout：用户态进程睡眠，等待超时或者被事件唤醒
 */
static int do_poll(struct poll_list *list, struct poll_wqueues *wait,
		   struct timespec64 *end_time)
{
	poll_table* pt = &wait->pt;
	ktime_t expire, *to = NULL;
	int timed_out = 0, count = 0;
	u64 slack = 0;
	__poll_t busy_flag = net_busy_loop_on() ? POLL_BUSY_LOOP : 0;
	unsigned long busy_start = 0;

	/* 如果等待时间为0，则直接超时返回 */
	if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {
		pt->_qproc = NULL;
		timed_out = 1;
	}
	
	if (end_time && !timed_out)/* 计时? */
		slack = select_estimate_accuracy(end_time);
	/* 一直循环到超时时间或者有相应的事件触发唤醒进程 */
	for (;;) {
	/* 循环检测所有的pollfd */
		struct poll_list *walk;
		bool can_busy_loop = false;
		/* 完成对所有fd对应的设备注册监听后，for循环会陷入等待超时 */
		for (walk = list; walk != NULL; walk = walk->next) {
		/* 对 poll_list 链表中所有 poll_list 进行遍历 */
			struct pollfd * pfd, * pfd_end;

			pfd = walk->entries;
			pfd_end = pfd + walk->len;
			for (; pfd != pfd_end; pfd++) {
			/* 对本 poll_list 中所有的 pollfd 遍历，给fd对应的设备注册监听 */
				if (do_pollfd(pfd, pt, &can_busy_loop,
					      busy_flag)) {
				/* 一旦 do_pollfd 函数返回不为0，说明该描述符有用户关心的事件发生 */
					count++; /* 计入count */
					pt->_qproc = NULL;
					/* found something, stop busy polling */
					busy_flag = 0;
					can_busy_loop = false;
				}
			}
		}
		/*
		 * All waiters have already been registered, so don't provide
		 * a poll_table->_qproc to them on the next loop iteration.
		 * 设置 poll_table 回调函数为NULL：由于将用户态进程挂到各个fd设备的等待队列，只需执行一次
		 * 因此在循环完所有fd后，将回调函数置NULL，下一次循环则不再进行挂载
		 */
		pt->_qproc = NULL;
		if (!count) {/* 如果count==0，检测用户态进程是否有信号要处理 */
			count = wait->error;
			if (signal_pending(current))
				count = -EINTR;
		}
		if (count || timed_out)
		/* 一旦count不为0，或者超时则跳出循环；对应用户态解除阻塞状态 */
			break;

		/* only if found POLL_BUSY_LOOP sockets && not out of time */
		if (can_busy_loop && !need_resched()) {
			if (!busy_start) {
				busy_start = busy_loop_current_time();
				continue;
			}
			if (!busy_loop_timeout(busy_start))
				continue;
		}
		busy_flag = 0;

		/*
		 * If this is the first loop and we have a timeout
		 * given, then we convert to ktime_t and set the to
		 * pointer to the expiry value.
		 */
		if (end_time && !to) {
			expire = timespec64_to_ktime(*end_time);
			to = &expire;
		}
		/* 用户态进程休眠，并设置定时器，如果超时前后没人来唤醒，则设置标志位timed_out
		 * poll_schedule_timeout->schedule_hrtimeout_range->schedule_hrtimeout_range_clock()
		 */
		if (!poll_schedule_timeout(wait, TASK_INTERRUPTIBLE, to, slack))
			timed_out = 1;
	}
	return count;/* 返回可操作描述符个数 */
}
```
do_pollfd
```
/*
 * 执行对fd对应设备的监听注册
 * 1. 通过驱动设备的poll函数注册监听(tcp: sock_poll->tcp_poll)
 * 2. 返回对文件监听的掩码
 */
static inline unsigned int do_pollfd(struct pollfd *pollfd, poll_table *pwait,
				     bool *can_busy_poll,
				     unsigned int busy_flag)
{
	unsigned int mask;
	int fd;

	mask = 0;
	fd = pollfd->fd;
	if (fd >= 0) {
		struct fd f = fdget(fd);
		mask = POLLNVAL;
		/* 根据fd找到对应的文件结构体 */
		if (f.file) {
			mask = DEFAULT_POLLMASK;
			if (f.file->f_op->poll) {
			/* 将用户对该fd要监听的事件，注册到 poll_table 中 */
				pwait->_key = pollfd->events|POLLERR|POLLHUP;
				pwait->_key |= busy_flag;
			/* 调用设备驱动的 poll 函数， sock_poll, 注册监听到设备驱动的等待队列中
			 * 在sock_poll中调用 tcp_poll 函数：
			 * 1. 设置监听
			 * 2. 检查各个fd的目前状态，并返回发生事件的掩码
			 */
				mask = f.file->f_op->poll(f.file, pwait);
				if (mask & busy_flag)
					*can_busy_poll = true;
			}
			/* Mask out unneeded events. 
			 * 该fd目前发生的事件是否是用户关心的？
			 */
			mask &= pollfd->events | POLLERR | POLLHUP;
			fdput(f);
		}
	}
	pollfd->revents = mask;/* 将fd发生的事件设置在结构体的返回事件上，用户态通过该数据感知 */

	return mask;
}
```
tcp_poll
```
/*
 * 1. 通过 poll_table->__pollwait 挂在监听到设备驱动事件等待队列中
 * 2. 检查本sock是否有事件需要处理；
 * 3. 如果有则返回事件掩码；否则返回0
 */
unsigned int tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
{
	unsigned int mask;
	struct sock *sk = sock->sk;
	const struct tcp_sock *tp = tcp_sk(sk);
	/* 在全局hash表rps_sock_flow_table中，记录当前流处理的CPU编号; */
	sock_rps_record_flow(sk);
	/* 
	 * 设置监听，只会执行一次
	 * 核心处理函数：sock_poll_wait->poll_wait->__pollwait()
	 * 申请一个 poll_wait_table_entry,并将entry.wait 结构体挂到驱动的事件等待队列(sk_sleep(sk))上，实现对硬件事件的监听
	 */
	sock_poll_wait(file, sk_sleep(sk), wait);
	/* 下面部分主要目的时返回一个描述读写操作是否就绪的mask掩码，根据这个掩码给fd_set赋值 */
	
	/* 经过listen系统调用后，sock会进入TCP_LISTEN状态 */
	if (sk->sk_state == TCP_LISTEN)
		/* icsk_accept_queue 在inet_csk_listen_start中被初始化；对于server端的连接控制fd，poll需要特殊处理
		 * 该队列用来保存正在建立连接和已经建立连接但未被accept的传输控制块；
		 * 该队列中有成员，则可能需要建立新的连接？accept系统调用即检查该队列中是否有满足条件的sock
		 */
		return inet_csk_listen_poll(sk);

	/* Socket is not locked. We are protected from async events
	 * by poll logic and correct handling of state changes
	 * made by other threads is impossible in any case.
	 */

	mask = 0;

	/* 如果sock->sk_shutdown有置位，意味着sock可能已经关闭或者正在走关闭挥手流程 */
	if (sk->sk_shutdown == SHUTDOWN_MASK || sk->sk_state == TCP_CLOSE)
		mask |= POLLHUP;
	if (sk->sk_shutdown & RCV_SHUTDOWN)
		mask |= POLLIN | POLLRDNORM | POLLRDHUP;

	/* Connected or passive Fast Open socket? */
	if (sk->sk_state != TCP_SYN_SENT &&
	    (sk->sk_state != TCP_SYN_RECV || tp->fastopen_rsk)) {
		int target = sock_rcvlowat(sk, 0, INT_MAX);

		if (tp->urg_seq == tp->copied_seq &&
		    !sock_flag(sk, SOCK_URGINLINE) &&
		    tp->urg_data)
			target++;

		/* Potential race condition. If read of tp below will
		 * escape above sk->sk_state, we can be illegally awaken
		 * in SYN_* states. */
		if (tp->rcv_nxt - tp->copied_seq >= target)
			mask |= POLLIN | POLLRDNORM;

		if (!(sk->sk_shutdown & SEND_SHUTDOWN)) {
			if (sk_stream_is_writeable(sk)) {
				mask |= POLLOUT | POLLWRNORM;
			} else {  /* send SIGIO later */
				set_bit(SOCK_ASYNC_NOSPACE,
					&sk->sk_socket->flags);
				set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);

				/* Race breaker. If space is freed after
				 * wspace test but before the flags are set,
				 * IO signal will be lost. Memory barrier
				 * pairs with the input side.
				 */
				smp_mb__after_atomic();
				if (sk_stream_is_writeable(sk))
					mask |= POLLOUT | POLLWRNORM;
			}
		} else
			mask |= POLLOUT | POLLWRNORM;

		if (tp->urg_data & TCP_URG_VALID)
			mask |= POLLPRI;
	}
	/* This barrier is coupled with smp_wmb() in tcp_reset() */
	smp_rmb();
	if (sk->sk_err || !skb_queue_empty(&sk->sk_error_queue))
		mask |= POLLERR;

	return mask;
}
```
__pollwait
```
/* 
 * 入参：filp: fd对应的file结构体指针；
 * wait_address: 特定fd对应的硬件驱动程序中的等待队列头指针；
 * p: poll/select 的应用进程中poll_wqueues结构体的 poll_table 项；
 * 作用：分配一个 poll_table_entry, 塞入到驱动的事件等待队列上，作为联系事件和实际用户态进程之间的桥梁
 * 对 poll_list 中每个fd调用 fop->poll()->...->__pollwait() 都要分配一个 poll_table_entry ;
 * 		1. 如果 poll_wqueues.inline_entries[] 中还有空闲的位置，则分配一个
 * 		2. 否则从 poll_wqueues.table 中找一个空闲位置；
 * 		3. 如果都没有空闲位；则申请一页内存；将内存头部强转为 poll_table_page 结构体，挂在 poll_wqueues.table 链表中，再从该页中分配entry
 * 对申请的entry初始化；并将 entry.wait 结构体挂到驱动的事件等待队列上
 */
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
				poll_table *p)
{
	struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);
	/* 分配 poll_table_entry 结构；如果剩余空间不足，则申请一页内存；内容为 struct poll_table_page + N* struct poll_table_entry */
	struct poll_table_entry *entry = poll_get_entry(pwq);
	if (!entry)
		return;
	entry->filp = get_file(filp);		/* 保存对应的file结构体 */
	entry->wait_address = wait_address; /* 保存本entry对应的设备驱动程序等待队列头 */
	entry->key = p->_key;				/* 保存对该fd关系的事件掩码 */
	init_waitqueue_func_entry(&entry->wait, pollwake);/* 初始化等待队列项，pollwake是唤醒该等待队列项时的回调函数*/
	entry->wait.private = pwq;			/* 将 poll_wqueues 作为该等待队列项的私有数据，poll_wqueues.polling_task 中记录对应的用户态进程 */
	add_wait_queue(wait_address, &entry->wait);/* 将 entry.wait 结构体挂到驱动的事件等待队列上 */
}
```
select注册监听流程
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp, fd_set __user *, exp, struct timeval __user *, tvp) —> core_sys_select()
本质来说select、poll监听的原理是一样的；都是通过传递关心的fd到内核态，将用户态进程挂到fd对应驱动设备事件等待队列中，等待被唤醒

core_sys_select
```
int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
			   fd_set __user *exp, struct timespec *end_time)
{
	fd_set_bits fds;
	void *bits;
	int ret, max_fds;
	unsigned int size;
	struct fdtable *fdt;
	/* Allocate small arguments on the stack to save memory and be faster */
	long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];

	ret = -EINVAL;
	if (n < 0)
		goto out_nofds;

	/* 对用户输入的fd做矫正，最大的fd肯定不能超过当前用户进程的max_fds */
	rcu_read_lock();
	fdt = files_fdtable(current->files);
	max_fds = fdt->max_fds;
	rcu_read_unlock();
	if (n > max_fds)
		n = max_fds;

	/*
	 * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
	 * since we used fdset we need to allocate memory in units of
	 * long-words.
	 */
	size = FDS_BYTES(n);
	bits = stack_fds;
	/* 对每个要监测的fd，要分配6bit；如果预分配的空间不足，则用kmalloc分配 */
	if (size > sizeof(stack_fds) / 6) {
		/* Not enough space in on-stack array; must use kmalloc */
		ret = -ENOMEM;
		bits = kmalloc(6 * size, GFP_KERNEL);
		if (!bits)
			goto out_nofds;
	}
	fds.in      = bits;
	fds.out     = bits +   size;
	fds.ex      = bits + 2*size;
	fds.res_in  = bits + 3*size;
	fds.res_out = bits + 4*size;
	fds.res_ex  = bits + 5*size;
	/* 将用户态的数据copy至内核态 */
	if ((ret = get_fd_set(n, inp, fds.in)) ||
	    (ret = get_fd_set(n, outp, fds.out)) ||
	    (ret = get_fd_set(n, exp, fds.ex)))
		goto out;
	/* 对所有fd的 res_* 比特位复位 */
	zero_fd_set(n, fds.res_in);
	zero_fd_set(n, fds.res_out);
	zero_fd_set(n, fds.res_ex);

	ret = do_select(n, &fds, end_time);

	if (ret < 0)
		goto out;
	if (!ret) {
		ret = -ERESTARTNOHAND;
		if (signal_pending(current))
			goto out;
		ret = 0;
	}

	if (set_fd_set(n, inp, fds.res_in) ||
	    set_fd_set(n, outp, fds.res_out) ||
	    set_fd_set(n, exp, fds.res_ex))
		ret = -EFAULT;

out:
	if (bits != stack_fds)
		kfree(bits);
out_nofds:
	return ret;
}
```
do_select
```
int do_select(int n, fd_set_bits *fds, struct timespec *end_time)
{
	ktime_t expire, *to = NULL;
	struct poll_wqueues table;
	poll_table *wait;
	int retval, i, timed_out = 0;
	unsigned long slack = 0;
	unsigned int busy_flag = net_busy_loop_on() ? POLL_BUSY_LOOP : 0;
	unsigned long busy_end = 0;

	rcu_read_lock();
	retval = max_select_fd(n, fds);
	rcu_read_unlock();

	if (retval < 0)
		return retval;
	n = retval;

	poll_initwait(&table);
	wait = &table.pt;
	if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {
		wait->_qproc = NULL;
		timed_out = 1;
	}

	if (end_time && !timed_out)
		slack = select_estimate_accuracy(end_time);
		
	/* 前半段几乎和 do_poll 一样 */
	
	
	/* 后半段基本思想和 poll 是一致的；
	 * 1. 对每个要监听的fd，通过 poll_table.__pollwait 注册监听到驱动的事件等待队列中
	 * 2. 无线for循环遍历所有fd，直到有事件发生或者时间超时
	 */
	retval = 0;
	for (;;) {
		unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;
		bool can_busy_loop = false;

		inp = fds->in; outp = fds->out; exp = fds->ex;
		rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;

		for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
			unsigned long in, out, ex, all_bits, bit = 1, mask, j;
			unsigned long res_in = 0, res_out = 0, res_ex = 0;

			in = *(inp++); out = *outp++; ex = *exp++;
			all_bits = in | out | ex;
			if (all_bits == 0) {
			/* 一次性检测32个fd，如果一个long 32bit fd都没有置位，则直接扫描下一个Long；加快扫描速度 */
				i += BITS_PER_LONG;
				continue;
			}
			/* 如果32bit内有关心的fd，则逐位遍历，找到关心的fd；通过__pollwait注册到fd对应的驱动等待队列中 */
			for (j = 0; j < BITS_PER_LONG; ++j, ++i, bit <<= 1) {
				struct fd f;
				if (i >= n)
					break;
				if (!(bit & all_bits))
				/* all_bits 未置位，表示不关心 */
					continue;
				f = fdget(i);
				if (f.file) {
					const struct file_operations *f_op;
					f_op = f.file->f_op;
					mask = DEFAULT_POLLMASK;
					if (f_op->poll) {
						wait_key_set(wait, in, out,
							     bit, busy_flag);
						mask = (*f_op->poll)(f.file, wait);
					}
					fdput(f);
					if ((mask & POLLIN_SET) && (in & bit)) {
						res_in |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					if ((mask & POLLOUT_SET) && (out & bit)) {
						res_out |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					if ((mask & POLLEX_SET) && (ex & bit)) {
						res_ex |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					/* retval!=0 意味着一定有事件发生 */
					if (retval) {
						can_busy_loop = false;
						busy_flag = 0;

					/*
					 * only remember a returned
					 * POLL_BUSY_LOOP if we asked for it
					 */
					} else if (busy_flag & mask)
						can_busy_loop = true;

				}
			}
			/* 将发生的事件拷贝出去；最终传给用户态使用 */
			if (res_in)
				*rinp = res_in;
			if (res_out)
				*routp = res_out;
			if (res_ex)
				*rexp = res_ex;
			cond_resched(); /* 完成一轮循环后，CPU放权，给紧急任务让渡CPU资源(有条件重调度) */
		}
		/* 和do_poll一样，在完成第一遍遍历所有fd后， 回调函数置空；因为只需挂载一次 */
		wait->_qproc = NULL;
		/* 如果有事件发生、超时、对应用户态进程收到挂起信号则停止循环检测 */
		if (retval || timed_out || signal_pending(current))
			break;
		if (table.error) {
		/* 如果挂监听出现异常也退出监听 */
			retval = table.error;
			break;
		}

		/* only if found POLL_BUSY_LOOP sockets && not out of time 
		 * need_resched: 检查是否设置了 TIF_NEED_RESCHED 标志
		 * 如果没有可以继续loop，且没有紧急任务需要执行，则继续循环
		 */
		if (can_busy_loop && !need_resched()) {
			if (!busy_end) {
				busy_end = busy_loop_end_time();
				continue;
			}
			if (!busy_loop_timeout(busy_end))
				continue;
		}
		busy_flag = 0;

		/*
		 * If this is the first loop and we have a timeout
		 * given, then we convert to ktime_t and set the to
		 * pointer to the expiry value.
		 */
		if (end_time && !to) {
			expire = timespec_to_ktime(*end_time);
			to = &expire;
		}
		/* 一整次扫描完成的最后，调用poll_schedule_timeout函数
		 * 如果还未超时，则进入睡眠，等待就绪的文件描述符唤醒 
		 */
		if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,
					   to, slack))
			timed_out = 1;
	}

	poll_freewait(&table);

	return retval;
}
```
名字解释
RSS(receive side scaling): 接收端
RPS(receive packet steering): 接收端包的控制；在收到数据包后提交给协议栈的时候(netif_receive_skb_internal)，根据/sys/class/net//queues/rx-/rps_cpus的设置,通过hash值，把这个接收队列收到的数据包发送到设置的CPU集合中
RFS(receive flow steering): 接收端流的控制；在SMP系统中，如果应用程序所在额CPU和RPS选择的CPU非同一个，会降低cache利用；RFS通过在一个全局的hash表(rps_sock_flow_table)上，用流的hash值，
ARFS(accelerated receive flow steering): 加速的接收端流的控制
XPS(transmit packet steering): 发送端包的控制

相关结构体
poll_wqueues
每一个poll调用只有一个poll_wqueues，对外接口是poll_table
```
struct poll_wqueues {
	poll_table pt;
	struct poll_table_page *table;//每一个page占1个PAGE_SIZE大小的区域
	//保存当前调用select的用户进程struct task_struct结构体
	struct task_struct *polling_task;
	int triggered;//当前用户进程被唤醒后置1，以免该进程接着进睡眠
	int error;    //错误码
	int inline_index;//数组inline_entries当前引用下标
	//内嵌的poll_table_entry数量有限，如果空间不够，后续会通过动态申请page以链表挂到poll_wqueues.table统一管理
	struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];
};
```
poll_table_entry
每一个被监控的fd对应一个poll_table_entry; 该poll监控的所有fd对应的poll_table_entry都在poll_wqueues.poll_table_page中保存；每一个poll_table_page结构体都代表一个page的内存页
```
struct poll_table_entry {
	struct file *filp;		//指向特定fd对应的file结构体
	unsigned long key;		//关心的硬件设备事件掩码，如POLLIN、POLLOUT、POLLERR
	wait_queue_t wait;		//桥梁：作为挂在设备等待队列中的元素
	wait_queue_head_t *wait_address;//本entry挂接的设备驱动程序的等待队列头
}
```
poll_table_page
用来管理poll_table_entry的内存页；申请的物理页都会被强制转换成该结构体，以便管理内存页中的entry
并不是每一个poll.poll_table_page都有数据，只有在预先分配的N_INLINE_POLL_ENTRIES个不够用时，才会动态申请新空间储存poll_table_entry
```
struct poll_table_page {
	struct poll_table_page * next;//指向下一个申请的物理页
	struct poll_table_entry * entry;//指向entries[]中首个待分配 poll_table_entry地址
	struct poll_table_entry entries[0];//该page页后面剩余的空间都是待分配的
}
```
poll_table_struct
```
/*
 * Do not touch the structure directly, use the access functions
 * poll_does_not_wait() and poll_requested_events() instead.
 */
typedef struct poll_table_struct {
	poll_queue_proc _qproc;
	unsigned long _key;
} poll_table
```
__wait_queue
唤醒结构体，该结构体挂在各个驱动的事件等待队列上，一旦该驱动有事件产生，会唤醒该等待队列中的所有 wait_queue.private 对应的 poll_wqueues
```
struct __wait_queue {
	unsigned int		flags;//prepare_to_wait 中对flags的操作，可以看出含义
	void			*private; //通常指向当前任务控制块
	wait_queue_func_t	func; //唤醒阻塞任务的函数，决定了唤醒的方式
	struct list_head	task_list;//阻塞任务链表
};
typedef struct __wait_queue wait_queue_t
```