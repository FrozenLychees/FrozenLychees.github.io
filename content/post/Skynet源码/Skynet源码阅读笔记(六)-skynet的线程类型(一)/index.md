---
title: Skynet源码阅读笔记(六)-skynet的线程类型（一）
description: 
slug: 
date: 2024-02-10 00:00:00+0000
image: skynet.png
categories:
    - skynet
---

# Skynet源码阅读笔记(六)-skynet的线程类型
skynet中的线程类型可以分为一下几种类型
- 主线程
- worker 线程
- timer 线程 
- monitor 线程
- socket 线程

这些线程的初始化都在start函数里，由主线程驱动

## 主线程
主线程只负责初始化工作，它主要是在start中创建了一些基础服务、创建出其他线程。执行完初始化后，主线程就会阻塞在最后等待其他结束。

```
static void
start(int thread) {
	pthread_t pid[thread+3];   // thread 是work 线程的数目，3是其他3个类型的线程

	struct monitor *m = skynet_malloc(sizeof(*m));
	memset(m, 0, sizeof(*m));
	m->count = thread;
	m->sleep = 0;

	m->m = skynet_malloc(thread * sizeof(struct skynet_monitor *));
	int i;
	for (i=0;i<thread;i++) {
		m->m[i] = skynet_monitor_new();
	}
	if (pthread_mutex_init(&m->mutex, NULL)) {
		fprintf(stderr, "Init mutex error");
		exit(1);
	}
	if (pthread_cond_init(&m->cond, NULL)) {
		fprintf(stderr, "Init cond error");
		exit(1);
	}

	create_thread(&pid[0], thread_monitor, m);  // 第一个线程用于 thread_monitor
	create_thread(&pid[1], thread_timer, m);    // 第二个线程用于 thread_timer 
	create_thread(&pid[2], thread_socket, m);   // 第三个线程用于 thread_socket

	static int weight[] = { 
		-1, -1, -1, -1, 0, 0, 0, 0,
		1, 1, 1, 1, 1, 1, 1, 1, 
		2, 2, 2, 2, 2, 2, 2, 2, 
		3, 3, 3, 3, 3, 3, 3, 3, };
	struct worker_parm wp[thread];
	for (i=0;i<thread;i++) {
		wp[i].m = m;
		wp[i].id = i;
		if (i < sizeof(weight)/sizeof(weight[0])) {
			wp[i].weight= weight[i];
		} else {
			wp[i].weight = 0;
		}
		create_thread(&pid[i+3], thread_worker, &wp[i]);  // 主线程创建其他work线程
	}

	for (i=0;i<thread+3;i++) { 
		pthread_join(pid[i], NULL);    // 然后阻塞在这里等待其他线程结束
	}

	free_monitor(m);
}
```

## monitor 线程

monitor线程执行的函数是thread_monitor
```
static void *
thread_monitor(void *p) {
	struct monitor * m = p;
	int i;
	int n = m->count;
	skynet_initthread(THREAD_MONITOR);
	for (;;) {
		CHECK_ABORT
		for (i=0;i<n;i++) {
			// 检测对应线程的monitor 状态
			skynet_monitor_check(m->m[i]);
		}
		for (i=0;i<5;i++) {
			CHECK_ABORT
			sleep(1);
		}
	}

	return NULL;
}

void 
skynet_monitor_check(struct skynet_monitor *sm) {
	if (sm->version == sm->check_version) {
		if (sm->destination) {
			skynet_context_endless(sm->destination);
			skynet_error(NULL, "A message from [ :%08x ] to [ :%08x ] maybe in an endless loop (version = %d)", sm->source , sm->destination, sm->version);
		}
	} else {
		sm->check_version = sm->version;
	}
}

```
该函数每5秒跑一下检测，判断对应线程的check_version 是否和其version相等，如果相等就说明可能存在死循环，调用 skynet_context_endless 来对ctx的 endless进行赋值。

monitor 这个结构如下
```
struct monitor {
	int count;
	struct skynet_monitor ** m; // skynet_monitor数组，每个work线程一个
	pthread_cond_t cond;  // 用于唤醒睡眠的worker线程
	pthread_mutex_t mutex; 
	int sleep; // 睡眠的work线程数量
	int quit;
};

struct skynet_monitor {
	ATOM_INT version;  // 执行message的版本
	int check_version; // 上一次检测时 执行message的版本
	uint32_t source;  // message 的source
	uint32_t destination;  //message 的destination
};

```
本质上monitor是维护了一个条件变量 + skynet_monitor数组，每个work线程都拥有一个自己的skynet_monitor。因为worker线程和monitor线程在使用skynet_monitor时，写的是skynet_monitor的不同字段，而读操作即使出现异步问题也不要紧，所以这边可以不需要加锁。

monitor中的sleep 表示由多少个work线程在睡眠，睡眠的worker线程会阻塞在对应条件变量上等待被唤醒。

## worker线程

worker 线程执行的是thread_worker 函数

```
struct worker_parm {
	struct monitor *m;  // monitor的引用
	int id;   // 线程ID
	int weight;  //权重
};


// work线程的执行的实际函数如下
static void *
thread_worker(void *p) {

	struct worker_parm *wp = p;
	int id = wp->id;
	int weight = wp->weight;
	struct monitor *m = wp->m;
	struct skynet_monitor *sm = m->m[id];
	skynet_initthread(THREAD_WORKER);
	struct message_queue * q = NULL;
	while (!m->quit) {
		q = skynet_context_message_dispatch(sm, q, weight);  // 主要执行这个函数
		if (q == NULL) {
			if (pthread_mutex_lock(&m->mutex) == 0) {  // 如果返回的q为空，则获取锁后标记自己为sleep
				++ m->sleep;
				// "spurious wakeup" is harmless,  
				// because skynet_context_message_dispatch() can be call at any time.
				if (!m->quit)
					pthread_cond_wait(&m->cond, &m->mutex);  // 等待被唤醒，pthread_cond_wait 可能存在虚假唤醒，但并不要紧
				-- m->sleep;
				if (pthread_mutex_unlock(&m->mutex)) {  // 释放掉对应的锁
					fprintf(stderr, "unlock mutex error"); 
					exit(1);
				}
			}
		}
	}
	return NULL;
}
```
thread_worker这边主要就是执行 skynet_context_message_dispatch 函数，如果这个函数的返回值是空，则等待下一次的执行


具体看一下 skynet_context_message_dispatch 的内容
```
struct message_queue * 
skynet_context_message_dispatch(struct skynet_monitor *sm, struct message_queue *q, int weight) {
	if (q == NULL) {
		q = skynet_globalmq_pop(); // 从全局的消息队列中获取一个消息队列
		if (q==NULL)
			return NULL; // 没有则返回
	}

	uint32_t handle = skynet_mq_handle(q);  //获取这个消息队列对应的服务实例
	struct skynet_context * ctx = skynet_handle_grab(handle);
	if (ctx == NULL) {
		struct drop_t d = { handle };
		skynet_mq_release(q, drop_message, &d);
		return skynet_globalmq_pop();
	}

	int i,n=1;
	struct skynet_message msg;
	for (i=0;i<n;i++) {   
		if (skynet_mq_pop(q,&msg)) {    // 开始获取消息队列中的消息，
			skynet_context_release(ctx); 
			return skynet_globalmq_pop();
		} else if (i==0 && weight >= 0) {
			// 这边可以看出，每次执行消息是队列的 1/(2^weight)
 			n = skynet_mq_length(q);
			n >>= weight;
		}
		int overload = skynet_mq_overload(q);  // 当消息超载了之后，打个日志报警一下
		if (overload) {
			skynet_error(ctx, "May overload, message queue length = %d", overload); 
		}

		skynet_monitor_trigger(sm, msg.source , handle);  // 告诉monitor当前执行的服务是哪个服务

		if (ctx->cb == NULL) {
			skynet_free(msg.data);
		} else {
			dispatch_message(ctx, &msg); // 执行真正的信息处理
		}
 
		skynet_monitor_trigger(sm, 0,0); 
	}

	assert(q == ctx->queue);
	struct message_queue *nq = skynet_globalmq_pop();
	if (nq) {
		// If global mq is not empty , push q back, and return next queue (nq)
		// Else global mq is empty or block, don't push q back, and return q again (for next dispatch)
		skynet_globalmq_push(q);
		q = nq;
	} 
	skynet_context_release(ctx);

	return q;
}

```
skynet_context_message_dispatch 就是从队列中消费消息，weight在这起到的作用就是每次执行队列总长度的1/(2^weight)

所以前8条线程应该都是执行完所有消息，8-16条每次执行1/2的消息长度的数据，一次类推。

还有一个值得注意的，skynet_monitor_trigger就是会在这往skynet_monitor写入当前的服务ID，以便monitor线程进行监控

执行消息的函数是dispatch_message，dispatch_message的内容也不长，完整的看一下

```
static void
dispatch_message(struct skynet_context *ctx, struct skynet_message *msg) {
	assert(ctx->init);
	CHECKCALLING_BEGIN(ctx)
	pthread_setspecific(G_NODE.handle_key, (void *)(uintptr_t)(ctx->handle));

	// 取出消息对应的类型和长度，然后打个日志
	int type = msg->sz >> MESSAGE_TYPE_SHIFT;
	size_t sz = msg->sz & MESSAGE_TYPE_MASK;
	FILE *f = (FILE *)ATOM_LOAD(&ctx->logfile);
	if (f) {
		skynet_log_output(f, msg->source, type, msg->session, msg->data, sz);
	}
	++ctx->message_count;
	int reserve_msg;

	// 如果开启了profile，则会在这边统计对应的服务执行的时间
	if (ctx->profile) {
		ctx->cpu_start = skynet_thread_time();
		reserve_msg = ctx->cb(ctx, ctx->cb_ud, type, msg->session, msg->source, msg->data, sz);
		uint64_t cost_time = skynet_thread_time() - ctx->cpu_start;
		ctx->cpu_cost += cost_time;
	} else {

		// 不然就直接调用
		reserve_msg = ctx->cb(ctx, ctx->cb_ud, type, msg->session, msg->source, msg->data, sz);
	}
	if (!reserve_msg) {
		skynet_free(msg->data);
	}
	CHECKCALLING_END(ctx)
}
```

dispatch_message 准确来说目的就是执行ctx->cb，但外部根据执行环境决定是否写入日志和统计函数执行时间。

## 小结

这边先看了一下主线程、monitor线程和worker线程是如何工作的，另外两个线程下一次再分析。

主线程主要是初始化环境以及各个其他线程，然后join等待其他线程结束
monitor线程可以认为就是每5秒判断一次是否当前执行的某个worker线程进入了死循环。
worker线程就是将消息队列中的消息取出，并调用对应服务的cb函数。
