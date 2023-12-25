---
layout: post
categories: [Linux]
description: none
keywords: Linux
---
# 趣谈网络协议栈(三)，Socket编程常用函数的原理及代码实现
每个arp表项都会注册一个timer，用于检测表项对应的网络设备是否还存活
arp整体会注册一个timer，用于定期更新arp缓存表，保证表项内容有效

## socket陷入内核
在start_kernel()中会调用trap_init()注册系统中断及不同中断对应的handler
在触发0x80中断陷入内核后，根据trap_init()中定义，会触发system_call()汇编函数
在system_call中根据中断号，在_sys_call_table中寻找对应的handle，socket相关的中断号为101,对应sys_socketcall函数

## 收包数据处理
全局backlog队列：top half后，驱动会调用netif_rx将接收到的数据包缓存于该队列中
struck sock->back_log队列；网络层在tcp_rcv函数中将接收到的数据包缓存与该队列
struct sock->receive_queue队列：在该队列中的数据包方可被正式处理，可作为tcp_read函数原材料
1–>2：网络设备软中断触发net_bh函数，转移数据包
2–>3：release_sock->tcp_rcv, 先通过skb_dequeue从back_log中取一个包，再通过tcp_rcv将包挂到receive_queue队列；且在tcp_rcv中sk_buff被转换成sock

## socket函数
sys_socketcall->sock_socket(int family, int type, int protocol)，对应用户态socket(AF_INET, SOCK_STREAM, 0)函数，参数刚好对应sock_socket
socket创建一般只执行到会话层，调用传输层注册的init函数(tcp,udp都为NULL)后，完成套接字资源创建

sock_socket
```
/*
 * 创建一个socket套接字：
 * 1. 找到域对应的操作接口结构体 struct propto_ops
 * 2. sock_alloc：申请socket管理结构体；实际是一种特殊的inode
 * 3. 调用 inet_proto_ops->create 进入会话层
 */
static int sock_socket(int family, int type, int protocol)
{
	int i, fd;
	struct socket *sock;
	struct proto_ops *ops;

	/* Locate the correct protocol family.
	 * 通过域协议号，找到对于的表示层->会话层的操作接口结构体
	 * AF_INET->inet_proto_ops
	 */
	for (i = 0; i < NPROTO; ++i) 
	{
		if (pops[i] == NULL) continue;
		if (pops[i]->family == family) 
			break;
	}

	if (i == NPROTO) 
	{
  		return -EINVAL;
	}
	ops = pops[i];

	/* 判断要创建的socket类型是否正确 */
	if ((type != SOCK_STREAM && type != SOCK_DGRAM &&
		type != SOCK_SEQPACKET && type != SOCK_RAW &&
		type != SOCK_PACKET) || protocol < 0)
			return(-EINVAL);

	/*
	 * 申请socket管理结构体；实际是通过申请 struct inode 管理结构体；
	 * inode中包含了struct socket; 所以socket是一种特殊的inode
	 */
	if (!(sock = sock_alloc())) 
	{
		printk("NET: sock_socket: no more sockets\n");
		return(-ENOSR);	/* Was: EAGAIN, but we are out of
				   system resources! */
	}

	sock->type = type;
	sock->ops = ops;
	/* 调用会话层创建接口函数 */
	if ((i = sock->ops->create(sock, protocol)) < 0) 
	{
		sock_release(sock);
		return(i);
	}
	/* 创建文件包裹inode，并最终将文件描述符返回给用户态
	 * 且会设置inode->f_op = &socket_file_ops，为socket操作函数
	 */
	if ((fd = get_fd(SOCK_INODE(sock))) < 0) 
	{
		sock_release(sock);
		return(-EINVAL);
	}

	return(fd);
}
```
inet_create
```
/*
 * 完成socket资源创建
 * 1. 申请sock管理结构体
 * 2. 根据socket类型，找到对应的会话层-->传输层操作接口结构体 struct proto
 */
static int inet_create(struct socket *sock, int protocol)
{
	struct sock *sk;
	struct proto *prot;
	int err;
	/* 申请会话层对应的sock管理结构体 */
	sk = (struct sock *) kmalloc(sizeof(*sk), GFP_KERNEL);
	if (sk == NULL) 
		return(-ENOBUFS);
	sk->num = 0;
	sk->reuse = 0;
	switch(sock->type) 
	{
	/* 根据socket类型，找到对应的会话层-->传输层操作接口结构体 struct proto */
		case SOCK_STREAM:
		case SOCK_SEQPACKET:
			if (protocol && protocol != IPPROTO_TCP) 
			{
				kfree_s((void *)sk, sizeof(*sk));
				return(-EPROTONOSUPPORT);
			}
			protocol = IPPROTO_TCP;
			sk->no_check = TCP_NO_CHECK;
			prot = &tcp_prot;
			break;
		.......
	}
	sk->socket = sock;
#ifdef CONFIG_TCP_NAGLE_OFF
	sk->nonagle = 1;
#else    
	sk->nonagle = 0;
#endif  
	sk->type = sock->type;
	
	......
  	
	sk->state_change = def_callback1;
	sk->data_ready = def_callback2;
	sk->write_space = def_callback3;
	sk->error_report = def_callback1;

	if (sk->num) 
	{
		/* 将具有确定端口号一个新sock 结构加入到sock_array 数组表示的sock 结构链表群中 */
		put_sock(sk->num, sk);
		sk->dummy_th.source = ntohs(sk->num);/* 用该端口号初始化TCP头 */
	}
	/* 调用传输层初始化函数 */
	if (sk->prot->init) 
	{
		err = sk->prot->init(sk);
		if (err != 0) 
		{
			destroy_sock(sk);
			return(err);
		}
	}
	return(0);
}
```
get_fd
```
/*
 * 创建一个struct file结构体wrap inode，并将该结构体填入current->files->fd中
 * file, inode结构体都是可以在用户态使用，必须使用get_empty_filp()函数来获取内存
 */
static int get_fd(struct inode *inode)
{
	int fd;
	struct file *file;

	/*
	 *	Find a file descriptor suitable for return to the user. 
	 * 由于file管理结构体用户态也要使用，因此必须从伙伴系统中分配内存
	 */
	file = get_empty_filp();
	if (!file) 
		return(-1);
	/* 找到本进程下file结构体数组的空位置 */
	for (fd = 0; fd < NR_OPEN; ++fd)
		if (!current->files->fd[fd]) 
			break;
	if (fd == NR_OPEN) 
	{
		file->f_count = 0;
		return(-1);
	}
	/* 用来将一个给定的文件描述符加入集合之中 */
	FD_CLR(fd, &current->files->close_on_exec);
		current->files->fd[fd] = file;/* 将file结构体填入 */
	file->f_op = &socket_file_ops;/* 将socket的操作函数赋给file->f_op */
	file->f_mode = 3;
	file->f_flags = O_RDWR;
	file->f_count = 1;
	file->f_inode = inode;
	if (inode) 
		inode->i_count++;
	file->f_pos = 0;
	return(fd);
}
```
bind函数
用户态函数bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen)对应sock_bind

sock_bind
```
/*
 * 1. 找到sockfd对应的socket结构体
 * 2. 将用户态的地址信息结构体cp到内核态===》表示层完成
 * 3. 调用表示层-->会话层接口函数，进入会话层
 */
static int sock_bind(int fd, struct sockaddr *umyaddr, int addrlen)
{
	struct socket *sock;
	int i;
	char address[MAX_SOCK_ADDR];
	int err;

	if (fd < 0 || fd >= NR_OPEN || current->files->fd[fd] == NULL)
		return(-EBADF);
	/* 找到fd对应的socket */
	if (!(sock = sockfd_lookup(fd, NULL))) 
		return(-ENOTSOCK);
	/* 将用户态的地址信息结构体copy到内核态 */
	if((err=move_addr_to_kernel(umyaddr,addrlen,address))<0)
	  	return err;
	/* 调用表示层-->会话层接口函数，进入会话层  inet_bind */
	if ((i = sock->ops->bind(sock, (struct sockaddr *)address, addrlen)) < 0) 
	{
		return(i);
	}
	return(0);
}
```
inet_bind
```
/* 将sock绑定到对应的传输层协议sock_array中
 * 1. 判断sock状态，在没有bind之前，一定是close的
 * 2. 如果端口号为0，则分配一个
 * 3. 检查source addr
 * 4. 检查source addr & port num是否符合复用条件限制；如果不允许复用，又再次申请，则报错
 * 5. put_sock：将sock注册到sock_array中
 */
static int inet_bind(struct socket *sock, struct sockaddr *uaddr,
	       int addr_len)
{
	struct sockaddr_in *addr=(struct sockaddr_in *)uaddr;
	struct sock *sk=(struct sock *)sock->data, *sk2;
	unsigned short snum = 0 /* Stoopid compiler.. this IS ok */;
	int chk_addr_ret;

	/* 在进行地址绑定时，该套接字应该处理关闭状态 */
	if (sk->state != TCP_CLOSE)
		return(-EIO);
	if(addr_len<sizeof(struct sockaddr_in))
		return -EINVAL;
		
	if(sock->type != SOCK_RAW)
	{
		if (sk->num != 0) 
			return(-EINVAL);

		snum = ntohs(addr->sin_port);

		/* 用户没有传port下来，则内核自己找一个空闲的port使用 */
		if (snum == 0) 
		{
			snum = get_new_socknum(sk->prot, 0);
		}
		/* 0-1023端口号仅限super user使用 */
		if (snum < PROT_SOCK && !suser()) 
			return(-EACCES);
	}
	
	chk_addr_ret = ip_chk_addr(addr->sin_addr.s_addr);
	/* 如果指定的地址不是本地地址，并且也不是一个多播地址，则错误返回：地址不可用 */
	if (addr->sin_addr.s_addr != 0 && chk_addr_ret != IS_MYADDR && chk_addr_ret != IS_MULTICAST)
		return(-EADDRNOTAVAIL);	
	/* 没有指定地址，则系统自动分配一个本地地址 */
	if (chk_addr_ret || addr->sin_addr.s_addr == 0)
		sk->saddr = addr->sin_addr.s_addr;
	
	if(sock->type != SOCK_RAW)
	{
		/* 对于非SOCK_RAW,需要将分配了端口号的sock插入到对应传输层协议的sock_array hash 链表中. */
		cli();
		/* sock_array也是一个hash链表，先找到对应的socket hash桶；再在bulk中寻找空闲的sock. */
		for(sk2 = sk->prot->sock_array[snum & (SOCK_ARRAY_SIZE -1)];
					sk2 != NULL; sk2 = sk2->next) 
		{
		/* 检查新分配的snum端口号是否有冲突；如果有冲突则要进一步检查 */
			if (sk2->num != snum) 
				continue;
			if (!sk->reuse)/* 如果二者端口号相同，检查sock是否允许地址复用；没有则返回错误 */
			{
				sti();
				return(-EADDRINUSE);
			}
			
			if (sk2->num != snum) 
				continue;		/* more than one */
			if (sk2->saddr != sk->saddr) /* 端口号可以复用的情况下检查source addr是否相同 */
				continue;	/* socket per slot ! -FB */
			 /* 如果sock结构为TCP_LISTEN,表示该sock为服务端，服务端不可使用地址复用选项 */
			if (!sk2->reuse || sk2->state==TCP_LISTEN) 
			{
				sti();
				return(-EADDRINUSE);
			}
		}
		sti();
		/* 保证sk不在sock_array队列中 */
		remove_sock(sk);
		/* 根据新分配的snum将sock插入到sock_array合适位置，相当于正式在传输层注册了sock */
		put_sock(snum, sk);
		/* 初始化tcp header->source_addr */
		sk->dummy_th.source = ntohs(sk->num);
		sk->daddr = 0;
		sk->dummy_th.dest = 0;
	}
	return(0);
}
```
listen函数
用户态函数listen(int sockfd, int backlog)对应sock_listen

sock_listen
```
/*
 * listen函数BSD层对应的函数，为server端接受client端连接做准备
 */
static int sock_listen(int fd, int backlog)
{
	struct socket *sock;
	
	if (fd < 0 || fd >= NR_OPEN || current->files->fd[fd] == NULL)
		return(-EBADF);
	/* 找到fd对应的socket */
	if (!(sock = sockfd_lookup(fd, NULL))) 
		return(-ENOTSOCK);
	/*  */
	if (sock->state != SS_UNCONNECTED) 
	{
		return(-EINVAL);
	}
	/* 调用会话层接口，进入会话层 */
	if (sock->ops && sock->ops->listen)
		sock->ops->listen(sock, backlog);
	sock->flags |= SO_ACCEPTCON;/* 调用完成后，设置标志位 */
	return(0);
}
```

inet_listen
```
/*
 *	listen函数会话层对应的函数：将sock转为监听状态
 * 1. 检查backlog的长度大小
 * 2. 检查状态并重置sk的状态
 */
static int inet_listen(struct socket *sock, int backlog)
{
	struct sock *sk = (struct sock *) sock->data;

	if(inet_autobind(sk)!=0)
		return -EAGAIN;

	/* We might as well re use these. */ 
	/*
	 * note that the backlog is "unsigned char", so truncate it
	 * somewhere. We might as well truncate it to what everybody
	 * else does..
	 */
	/* 如果设置的队列长度大于128则设置为128 */
	if ((unsigned) backlog > 128)
		backlog = 128;
	sk->max_ack_backlog = backlog;
	if (sk->state != TCP_LISTEN)
	{
		/* 设置sock为Listen状态 */
		sk->ack_backlog = 0;
		sk->state = TCP_LISTEN;
	}
	return(0);
}
```
accept函数
用户态函数accept(int sockfd, const struct sockaddr *addr, socklen_t addrlen)对应sock_accept

sock_accept
```
/*
 * accept函数的BSD层对应函数，用于server端接受一个client的连接
 */
static int sock_accept(int fd, struct sockaddr *upeer_sockaddr, int *upeer_addrlen)
{
	struct file *file;
	struct socket *sock, *newsock;
	int i;
	char address[MAX_SOCK_ADDR];
	int len;

	if (fd < 0 || fd >= NR_OPEN || ((file = current->files->fd[fd]) == NULL))
		return(-EBADF);
	/* 找到fd对应的socket */
  	if (!(sock = sockfd_lookup(fd, &file))) 
		return(-ENOTSOCK);
	/* 检查socket状态 */
	if (sock->state != SS_UNCONNECTED) 
	{
		return(-EINVAL);
	}
	/* 检查sock状态 */
	if (!(sock->flags & SO_ACCEPTCON)) 
	{
		return(-EINVAL);
	}
	/* 新申请一个socket，同时也申请一个新inode */
	if (!(newsock = sock_alloc())) 
	{
		printk("NET: sock_accept: no more sockets\n");
		return(-ENOSR);	/* Was: EAGAIN, but we are out of system
				   resources! */
	}
	newsock->type = sock->type;
	newsock->ops = sock->ops;
	/* 调用inet_create，创建一个新的socket */
	if ((i = sock->ops->dup(newsock, sock)) < 0) 
	{
		sock_release(newsock);
		return(i);
	}
	/* 使用新的socket，调用会话层accpet函数 */
	i = newsock->ops->accept(sock, newsock, file->f_flags);
	if ( i < 0) 
	{
		sock_release(newsock);
		return(i);
	}
	/* 将新的socket封装在struct file中 */
	if ((fd = get_fd(SOCK_INODE(newsock))) < 0) 
	{
		sock_release(newsock);
		return(-EINVAL);
	}

	if (upeer_sockaddr)
	{
		newsock->ops->getname(newsock, (struct sockaddr *)address, &len, 1);
		move_addr_to_user(address,len, upeer_sockaddr, upeer_addrlen);
	}
	return(fd);
}
```

inet_accept
```
/*
 * 会话层accept：
 * 1. 创建一个本地新套接字用于通信，原监听套接字仍然用于监听
 * 2. 调用传输层accept，发送应答报文，并返回一个sock资源
 * 3. 将新创建sock结构挂接到监听sock结构的接收队列(sock->receive_queue)中
 */
static int inet_accept(struct socket *sock, struct socket *newsock, int flags)
{
	struct sock *sk1, *sk2;
	int err;
	/* 从老的socket中获取sock */
	sk1 = (struct sock *) sock->data;

	/* 检测newsock中是否已经存在套接字，如果是则要释放 */
	if (newsock->data)
	{
	  	struct sock *sk=(struct sock *)newsock->data;
	  	newsock->data=NULL;
	  	sk->dead = 1;
	  	destroy_sock(sk);
	}
	/* 检测会话层-->传输层接口函数指针是否存在，不存在则异常 */
	if (sk1->prot->accept == NULL) 
		return(-EOPNOTSUPP);

	/* Restore the state if we have been interrupted, and then returned. */
	if (sk1->pair != NULL ) 
	{
	/* 检查监听sock中是否挂有被中断的sock，有的话优先恢复被中断的 */
		sk2 = sk1->pair;
		sk1->pair = NULL;
	} 
	else
	{
		/* 老的socket经过accept后，会生成一个新的sock；以后就用这个新的sock进行通信管理
		 * 在传输层的accept会发送应答数据包；
		 * 传入的是侦听sock
		*/
		sk2 = sk1->prot->accept(sk1,flags);
		if (sk2 == NULL) 
		{
			if (sk1->err <= 0)
				printk("Warning sock.c:sk1->err <= 0.  Returning non-error.\n");
			err=sk1->err;
			sk1->err=0;
			return(-err);
		}
	}
	/* 将代表本次通话的sock赋给新的socket */
	newsock->data = (void *)sk2;
	sk2->sleep = newsock->wait;
	sk2->socket = newsock;
	newsock->conn = NULL;
	/*  */
	if (flags & O_NONBLOCK) 
		return(0);

	cli(); /* avoid the race. */
	while(sk2->state == TCP_SYN_RECV) 
	{
		interruptible_sleep_on(sk2->sleep);
		if (current->signal & ~current->blocked) 
		{
			sti();
			/* 如果socket在等待过程中被中断，则监听套接字sock结构之pair字段将被
			 * 初始化为该被中断套接字对应的sock结构；
			 * 则在下一次调用accept函数时，会优先处理这个之前被中断的套接字
			 */
			sk1->pair = sk2;
			sk2->sleep = NULL;
			sk2->socket=NULL;
			newsock->data = NULL;
			return(-ERESTARTSYS);
		}
	}
	sti();
	/* 如果while正常结束，则sock应该进入TCP_ESTABLISHED状态 */
	if (sk2->state != TCP_ESTABLISHED && sk2->err > 0) 
	{
		err = -sk2->err;
		sk2->err=0;
		sk2->dead=1;	/* ANK */
		destroy_sock(sk2);
		newsock->data = NULL;
		return(err);
	}
	/* 新创建的套接字处于连接状态，原套接字依旧处理监听状态 */
	newsock->state = SS_CONNECTED;
	return(0);
}
```

tcp_accept
```
/*
 *	传输层协议tcp，对应accept系统调用的实现
 * 1. 检查监听sock的报文接收队列，查看是否存在已经完成连接的数据包
 * 2. 如果没有则继续循环检查/阻塞等待
 * 3. 将找到的已完成连接sock返回上层
 */
static struct sock *tcp_accept(struct sock *sk, int flags)
{
	struct sock *newsk;
	struct sk_buff *skb;
  
  /* 传入的不是侦听sock，则异常 */
	if (sk->state != TCP_LISTEN) 
	{
		sk->err = EINVAL;
		return(NULL); 
	}

	/* Avoid the race. */
	cli();
	sk->inuse = 1;
	/* 检查监听sock->receive_queue队列，查看是否存在已经完成连接的数据包 */
	while((skb = tcp_dequeue_established(sk)) == NULL) 
	{
	/* 如果没有连接完成状态的数据包 */
		if (flags & O_NONBLOCK) 
		{
			/* 如果设置了非阻塞模式，则返回 */
			sti();
			/* 返回前将sock->back_log中缓存的数据包尽可能转移到sock->receive_queue */
			release_sock(sk);
			sk->err = EAGAIN;
			return(NULL);
		}
		
		release_sock(sk);
		/* 否则等待睡眠，用户态阻塞在accept函数 */
		interruptible_sleep_on(sk->sleep);
		if (current->signal & ~current->blocked) 
		{
			sti();
			sk->err = ERESTARTSYS;
			return(NULL);
		}
		sk->inuse = 1;
  	}
	sti();

	/*
	 *	Now all we need to do is return skb->sk. 
	 */

	newsk = skb->sk;

	kfree_skb(skb, FREE_READ);
	/* 缓存的未应答数据包个数-1 */
	sk->ack_backlog--;
	release_sock(sk);
	return(newsk);
}
```
tcp_dequeue_established
```
/*
 * 1. 通过tcp_find_established函数检查监听sock中是否有已经建立连接的数据包
 * 2. 有则将该包从监听sock->receive_queue队列中去除，向上层返回该包
 */
static struct sk_buff *tcp_dequeue_established(struct sock *s)
{
	struct sk_buff *skb;
	unsigned long flags;
	save_flags(flags);
	cli(); 
	skb=tcp_find_established(s);
	if(skb!=NULL)
		skb_unlink(skb);	/* Take it off the queue */
	restore_flags(flags);
	return skb;
}
```
tcp_find_established
```
/*
 * 对于监听sock，其sock->receive_queue中的数据包都是连接请求、连接完成数据包，即syn包
 * 数据处理是会话sock负责
 * 1. 从监听sock缓冲队列中检查是否存在已经完成连接的数据包
 * 2. 如果有则直接返回sock
 */ 
static struct sk_buff *tcp_find_established(struct sock *s)
{
	struct sk_buff *p=skb_peek(&s->receive_queue);
	if(p==NULL)
		return NULL;
	do
	{
		if(p->sk->state == TCP_ESTABLISHED || p->sk->state >= TCP_FIN_WAIT1)
			return p;
		p=p->next;
	}
	/* receive_queue 为环形队列，如果p指针回到起始点，则遍历已完成 */
	while(p!=(struct sk_buff *)&s->receive_queue);
	return NULL;
}
```
connect
sock_connect
connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen)
表示层、会话层的调用不需要多说 sock_connect->inet_connect()->tcp_connect()，在tcp_connect()中

tcp_connect
```
/*
 *	This will initiate an outgoing connection. 
 */
static int tcp_connect(struct sock *sk, struct sockaddr_in *usin, int addr_len)
{
	struct sk_buff *buff;
	struct device *dev=NULL;
	unsigned char *ptr;
	int tmp;
	int atype;
	struct tcphdr *t1;
	struct rtable *rt;

	if (sk->state != TCP_CLOSE) 
	{
		return(-EISCONN);
	}
	
	if (addr_len < 8) 
		return(-EINVAL);

	if (usin->sin_family && usin->sin_family != AF_INET) 
		return(-EAFNOSUPPORT);

  	/* 如果source addr为空，则填入本机IP */
  	if(usin->sin_addr.s_addr==INADDR_ANY)
		usin->sin_addr.s_addr=ip_my_addr();
		 
	if ((atype=ip_chk_addr(usin->sin_addr.s_addr)) == IS_BROADCAST || atype==IS_MULTICAST) 
		return -ENETUNREACH;
  
	sk->inuse = 1;
	sk->daddr = usin->sin_addr.s_addr;
	sk->write_seq = tcp_init_seq();/* 使用时间产生序列号 */
	sk->window_seq = sk->write_seq;
	sk->rcv_ack_seq = sk->write_seq -1;
	sk->err = 0;
	sk->dummy_th.dest = usin->sin_port;
	release_sock(sk);

	buff = sk->prot->wmalloc(sk,MAX_SYN_SIZE,0, GFP_KERNEL);
	if (buff == NULL) 
	{
		return(-ENOMEM);
	}
	sk->inuse = 1;
	buff->len = 24;
	buff->sk = sk;
	buff->free = 0;
	buff->localroute = sk->localroute;
	
	t1 = (struct tcphdr *) buff->data;

	/* 查询路由表获取可发送该数据包的路由表项；如果没有找到，则无法发送 */
	rt=ip_rt_route(sk->daddr, NULL, NULL);
	

	/* 创建IP，MAC首部，并返回两个首部的总长度
	 * skb:待创建首部的数据包；saddr: 数据包发送的最初源地址，被转发的数据包无需进行IP首部创建
	 * daddr:最终远端接收端IP地址
	 * dev:根据daddr查询路由表，将返回的表项中绑定的接口赋值给dev参数，并传出
	 * type:上层协议类型，tcp，udp
	 * tos,ttl：对应IP首部中的tos(服务类型),ttl(跳数);
	 */
	tmp = sk->prot->build_header(buff, sk->saddr, sk->daddr, &dev,
					IPPROTO_TCP, NULL, MAX_SYN_SIZE,sk->ip_tos,sk->ip_ttl);
	if (tmp < 0) 
	{
		sk->prot->wfree(sk, buff->mem_addr, buff->mem_len);
		release_sock(sk);
		return(-ENETUNREACH);
	}

	buff->len += tmp;
	t1 = (struct tcphdr *)((char *)t1 +tmp);

	memcpy(t1,(void *)&(sk->dummy_th), sizeof(*t1));
	t1->seq = ntohl(sk->write_seq++);
	sk->sent_seq = sk->write_seq;
	buff->h.seq = sk->write_seq;
	t1->ack = 0;
	t1->window = 2;
	t1->res1=0;
	t1->res2=0;
	t1->rst = 0;
	t1->urg = 0;
	t1->psh = 0;
	t1->syn = 1;
	t1->urg_ptr = 0;
	t1->doff = 6;
	/* use 512 or whatever user asked for */
	
	if(rt!=NULL && (rt->rt_flags&RTF_WINDOW))
		sk->window_clamp=rt->rt_window;
	else
		sk->window_clamp=0;

	if (sk->user_mss)
		sk->mtu = sk->user_mss;
	else if(rt!=NULL && (rt->rt_flags&RTF_MTU))
		sk->mtu = rt->rt_mss;
	else 
	{
#ifdef CONFIG_INET_SNARL
		if ((sk->saddr ^ sk->daddr) & default_mask(sk->saddr))
#else
		if ((sk->saddr ^ sk->daddr) & dev->pa_mask)
#endif
			sk->mtu = 576 - HEADER_SIZE;
		else
			sk->mtu = MAX_WINDOW;
	}
	if(sk->mtu <32)
		sk->mtu = 32;	/* Sanity limit */
		
	sk->mtu = min(sk->mtu, dev->mtu - HEADER_SIZE);

	ptr = (unsigned char *)(t1+1);
	ptr[0] = 2;
	ptr[1] = 4;
	ptr[2] = (sk->mtu) >> 8;
	ptr[3] = (sk->mtu) & 0xff;
	/* 计算tcp checksum */
	tcp_send_check(t1, sk->saddr, sk->daddr,
		  sizeof(struct tcphdr) + 4, sk);
	/* 修改TCP sock状态为 TCP_SYN_SENT */
	tcp_set_state(sk,TCP_SYN_SENT);
	sk->rto = TCP_TIMEOUT_INIT;
#if 0 /* we already did this */
	init_timer(&sk->retransmit_timer); 
#endif
	sk->retransmit_timer.function=&retransmit_timer;/* 注册超时重发函数 */
	sk->retransmit_timer.data = (unsigned long)sk;
	reset_xmit_timer(sk, TIME_WRITE, sk->rto);	/* 重置超时重发定时器 */
	sk->retransmits = TCP_SYN_RETRIES;

	sk->prot->queue_xmit(sk, dev, buff, 0);  /* 发送tcp报文 */
	reset_xmit_timer(sk, TIME_WRITE, sk->rto);
	tcp_statistics.TcpActiveOpens++;
	tcp_statistics.TcpOutSegs++;
  
	release_sock(sk);
	return(0);
}
```
tcp_rcv
在软中断函数(net_bh)中，通过从全局backlog函数中收包，再通过ip_packet_type->ip_rcv()处理后，数据包会进入各个sock的back_log队列
在ip_rcv中，会检查网络防火墙设置，如果不满足网络防火墙条件，则会给对端发一个ICMP报文(ICMP_DEST/PORT_UNREACH)
检查该IP包目的地址；如果dest addr不是本设备，而且也不是广播地址，则转发(ip_forward)
检查是否为分片数据包，如果是则要在网络层重组(ip_defrag)后，在将重组后的数据包送往传输层进行处理
通过ipheader->protocol确定传输层协议，传输层的所有协议都聚集在一个全局结构体struct inet_protocol中；然后调用传输层协议收包函数，将报文向上层传递；tcp->tcp_protocol;
```
/* 
 * 1. 通过get_sock，从传输层的所有sock中找到请求对应的服务端sock服务节点TCB(目的端口号)；如果找不到目标TCB，则返回TCP_reset
 * 2. 在skb_queue_tail中将报文发送到sock->back_log的队尾
 * 3. 检查sock->state, 如果在TCP_LISTEN状态，则使用 tcp_conn_request 进行ACK;如果在 TCP_SYN_RECV 状态，则调用 tcp_ack() 进行响应
		a. 在 tcp_conn_request 中，会新建sock，并将状态置为 TCP_SYN_RECV ,并响应ack报文
		b. 当再次收到client端的ACK报文时，会进入 tcp_ack() 函数，首先检查ACK报文是否合法
 */
```
tcp_conn_request
侦听sock接收队列receive_queue中缓存的均是请求连接数据包，不包含普通数据包
accept系统调用从侦听sock的 receive_queue 中取数据包，获得该数据包对应的sock；如果状态为TCP_ESTABLISHED，则系统调用成功返回
在 tcp_conn_request 中实现新创建对话sock，并将状态转为 TCP_SYN_RECV;
在 tcp_ack 函数中，专门负责对方发送的ACK书包；当检测到某个ACK数据包是三路握手连接过程中完成连接建立的应答数据包时，会将sock->state置为 TCP_ESTABLISHED
在 tcp_ack 函数中完成三路握手后，调用sock->state_change的回调函数，唤醒sock睡眠队列中的进程，从而继续accept(tcp_accept)函数的处理
```
/* 
 * 专门用于处理连接请求：
 * 1. 创建一个sock，并对该sock结构进行初始化
 * 2. 发送一个应答报文，并将新创建的sock结构体状态置为TCP_SYN_RECV
 * 3. 最后将新建的sock结构与请求连接数据包绑定并挂接到监听sock的receive_queue队列中
 */
```


































