---
title: Skynet源码阅读笔记(七)-skynet的线程类型（二）
description: 
slug: 
date: 2024-4-16 00:00:00+0000
image: skynet.jpg
categories:
    - skynet
---

# Skynet源码阅读笔记(七)-skynet的线程类型
skynet中的线程类型可以分为一下几种类型
- 主线程
- worker 线程
- timer 线程 
- monitor 线程
- socket 线程

这些线程的初始化都在start函数里，由主线程驱动

## tiemr线程

从start函数中进入，可以看到初始化timer线程的时候的使用了thread_timer。

```
#define CHECK_ABORT if (skynet_context_total()==0) break;

static void *
thread_timer(void *p) {
	struct monitor * m = p;
	skynet_initthread(THREAD_TIMER);  
	for (;;) {
		skynet_updatetime();    // 更新定时器相关
		skynet_socket_updatetime();  // 更新Socket 的时间，
		CHECK_ABORT
		wakeup(m,m->count-1);  // 唤醒睡眠线程
		usleep(2500);
		if (SIG) {  
			signal_hup();  // 如果接收到 SIGHUP 新号，则输出日志
			SIG = 0;
		}
	}
	// wakeup socket thread
	skynet_socket_exit();   // 唤醒socket 线程
	// wakeup all worker thread  //唤醒所有worker线程
	pthread_mutex_lock(&m->mutex);
  	m->quit = 1;  //work循环条件就是!quit
	pthread_cond_broadcast(&m->cond);
	pthread_mutex_unlock(&m->mutex);
	return NULL;
}
```

socket部分先跳过，主要是因为还没探索过，不确定里面再做啥。

从thread_timer 中可以看出，timer线程主要工作就是每2500微秒（2.5ms）定时调用 skynet_updatetime。当CHECK_ABORT检测执行到服务都退出了以后，timer才会跳出循环。
跳出循环后，还会唤醒socker和worker线程，让他们也执行退出流程。

## timer 相关数据结构

```
#define TIME_NEAR_SHIFT 8
#define TIME_NEAR (1 << TIME_NEAR_SHIFT)
#define TIME_LEVEL_SHIFT 6
#define TIME_LEVEL (1 << TIME_LEVEL_SHIFT)
#define TIME_NEAR_MASK (TIME_NEAR-1)
#define TIME_LEVEL_MASK (TIME_LEVEL-1)

struct timer_event {
	uint32_t handle;
	int session;
};

struct timer_node {
	struct timer_node *next;
	uint32_t expire;
};

struct link_list {
	struct timer_node head;
	struct timer_node *tail;
};

struct timer {
	struct link_list near[TIME_NEAR];  //  这边是 2 ^ 8 个，是最小的时间轮
	struct link_list t[4][TIME_LEVEL];  // 0 位置的每个都是 2^8的轮， 数组1上每个都是2^16, 依次类推
	struct spinlock lock;
	uint32_t time;
	uint32_t starttime;   //开服时间
	uint64_t current;   //当前时间
	uint64_t current_point;
};

static struct timer * TI = NULL;

static inline struct timer_node *
link_clear(struct link_list *list) {
	struct timer_node * ret = list->head.next;
	list->head.next = 0;
	list->tail = &(list->head);

	return ret;
}

```
skynet的定时器算法是时间轮，near是最近256个时间轮单位，每跑完256个单位才会更新一次时间轮。而t 可以认为是粒度更大的时间轮。
t0 上的每个元素的粒度都是2 ^ 8，一共TIME_LEVEL(2 ^ 6)个元素。
t1 上的每个元素的粒度都是2 ^ 14, 一共TIME_LEVEL(2 ^ 6)个元素。
t2 上的每个元素的粒度都是2 ^ 20, 一共TIME_LEVEL(2 ^ 6)个元素。
t3 上的每个元素的粒度都是2 ^ 26, 一共TIME_LEVEL(2 ^ 6)个元素。
所以t上刚好可表示2^32个元素。


## 初始化阶段
```
void 
skynet_timer_init(void) {
	TI = timer_create_timer();
	uint32_t current = 0;
	systime(&TI->starttime, &current);
	TI->current = current;
	TI->current_point = gettime();
}

// centisecond: 1/100 second
static void
systime(uint32_t *sec, uint32_t *cs) {
	struct timespec ti;
	clock_gettime(CLOCK_REALTIME, &ti);
	*sec = (uint32_t)ti.tv_sec;
	*cs = (uint32_t)(ti.tv_nsec / 10000000);
}

static uint64_t
gettime() {
	uint64_t t;
	struct timespec ti;
	clock_gettime(CLOCK_MONOTONIC, &ti);
	t = (uint64_t)ti.tv_sec * 100;
	t += ti.tv_nsec / 10000000;
	return t;
}

```
在初始化阶段，会将全局的TI需要的空间申请好，并将 current 和 current_point 设置好，这边需要注意的是，systime 和 gettime 的单位都是 1/100 秒，即10ms。

## 添加定时器
```
int
skynet_timeout(uint32_t handle, int time, int session) {
	if (time <= 0) {
		// 直接触发
		struct skynet_message message;
		message.source = 0;
		message.session = session;
		message.data = NULL;
		message.sz = (size_t)PTYPE_RESPONSE << MESSAGE_TYPE_SHIFT;

		if (skynet_context_push(handle, &message)) {
			return -1;
		}
	} else {
		// 定时触发
		struct timer_event event;
		event.handle = handle;
		event.session = session;
		timer_add(TI, &event, sizeof(event), time);
	}

	return session;
}


static void
timer_add(struct timer *T,void *arg,size_t sz,int time) {
	struct timer_node *node = (struct timer_node *)skynet_malloc(sizeof(*node)+sz); // 多申请了sz的内存
	memcpy(node+1,arg,sz);//拷贝事件数据

	SPIN_LOCK(T);

		node->expire=time+T->time; // 设置过期事件
		add_node(T,node); // 放到对应位置上

	SPIN_UNLOCK(T);
}

static void
add_node(struct timer *T,struct timer_node *node) {
	uint32_t time=node->expire;
	uint32_t current_time=T->time;
	
	if ((time|TIME_NEAR_MASK)==(current_time|TIME_NEAR_MASK)) { // 如果是near粒度下
		link(&T->near[time&TIME_NEAR_MASK],node);
	} else {
		int i;
		uint32_t mask=TIME_NEAR << TIME_LEVEL_SHIFT;
		for (i=0;i<3;i++) {
			if ((time|(mask-1))==(current_time|(mask-1))) { // 找到与timer相对应的粒度
				break;
			}
			mask <<= TIME_LEVEL_SHIFT;
		}

		link(&T->t[i][((time>>(TIME_NEAR_SHIFT + i*TIME_LEVEL_SHIFT)) & TIME_LEVEL_MASK)],node);	
	}
}

```
对外的主要接口是skynet_timeout，如果传入的时间小于0的话，则直接触发，产生对应的message丢入消息队列中。
而添加定时器通过timer_add来完成，这边在申请的内存的时候可以看到，定时器和对应事件的内存是一起申请的，node + 1 开始的内存存放的都是事件相关的数据。
add_node 则会根据过期事件来选择放到对应粒度的位置上


## 更新阶段
```
void
skynet_updatetime(void) {
	uint64_t cp = gettime();
	if(cp < TI->current_point) {
		skynet_error(NULL, "time diff error: change from %lld to %lld", cp, TI->current_point);
		TI->current_point = cp;
	} else if (cp != TI->current_point) {
		uint32_t diff = (uint32_t)(cp - TI->current_point);
		TI->current_point = cp;
		TI->current += diff;
		int i;
		for (i=0;i<diff;i++) {
			timer_update(TI);
		}
	}
}

static void 
timer_update(struct timer *T) {
	SPIN_LOCK(T);

	// try to dispatch timeout 0 (rare condition)
	timer_execute(T); // 先再执行一遍当前时间的timer_execute，让那些执行timeout(0)的也能理解触发

	// shift time first, and then dispatch timer message
	timer_shift(T); //偏移时间

	timer_execute(T); // 继续执行正常的timer_execute

	SPIN_UNLOCK(T);
}
```

在time线程中，每次睡眠后执行的update就是 skynet_updatetime， cp是当前的时间戳，不过单位是10ms。
如果cp出现小于之前记录的current_point，则是出现了系统时间偏移了，这边打个日志记录一下。
正常情况下，update 会计算跟上一次差了多少单位，然后每个单位都执行一下timer_update, timer_update 执行的主要函数就是下面的两个子函数

```
static inline void
timer_execute(struct timer *T) {
	int idx = T->time & TIME_NEAR_MASK;
	
	while (T->near[idx].head.next) {
		struct timer_node *current = link_clear(&T->near[idx]);
		SPIN_UNLOCK(T);
		// dispatch_list don't need lock T
		dispatch_list(current);   // 这个函数中会包装message，然后放到 消息队列中
		SPIN_LOCK(T);
	}
}

static void
timer_shift(struct timer *T) {
	int mask = TIME_NEAR;
	uint32_t ct = ++T->time;
	if (ct == 0) {
		move_list(T, 3, 0); // 最大的一轮跑完了，那就从最大粒度的0号元素重新开始
	} else {
		uint32_t time = ct >> TIME_NEAR_SHIFT;
		int i=0;

		// 找到下一个轮，如果为0则说明跑完当前的轮了，该去下一个更大的粒度的轮
		while ((ct & (mask-1))==0) {
			int idx=time & TIME_LEVEL_MASK;
			if (idx!=0) { 
				move_list(T, i, idx); 
				break;				
			}
			mask <<= TIME_LEVEL_SHIFT;
			time >>= TIME_LEVEL_SHIFT;
			++i; 
		}
	}
}

static void
move_list(struct timer *T, int level, int idx) {
	struct timer_node *current = link_clear(&T->t[level][idx]); //把对应粒度上面的事件分发到小粒度的轮上面
	while (current) {
		struct timer_node *temp=current->next;
		add_node(T,current);
		current=temp;
	}
}

```
   
timer_execute 的作用就是将near中对应的链表取出，然后将对应的事件抛到消息队列中去。
timer_shift 的作用就是更新当前的时间轮，如果当前粒度的时间轮跑完了，则会到更大到粒度的时间轮中把对应的事件更新到小粒度的轮上


## 小结
skynet的时间轮 就是将时间拆分成粒度的数组，加元素的时候，就直接到对应的粒度的位置上添加事件；更新的时候则把大粒度的事件轮拆分到小粒度上面等待执行。

不过这边我没看到取消定时器的函数，不知道是不是看漏了 还是说 实现上是再在触发后由业务判断是不是要回调。

后续在研究一下。

socket线程相关的有点复杂，后续新开一篇总结吧。