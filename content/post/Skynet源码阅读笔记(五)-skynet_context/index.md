---
title: Skynet源码阅读笔记(五)-skynet_context
description: 
slug: 
date: 2024-01-31 00:00:00+0000
image: skynet.png
categories:
    - skynet
---

# Skynet源码阅读笔记-skynet_context

## skynet_context
在Skynet中，服务是由skynetContext来定义的，服务拥有自己的module，自己的messageQueue以及自己的其他数据结构。

module、messageQueue、handle的概念在之前的文章中已经分析过了，现在就可以开始分析skynet_context了

```
struct skynet_context {
	void * instance;  // mod 实例
	struct skynet_module * mod;  // module
	void * cb_ud;  // 回调函数的data
	skynet_cb cb; // 回调函数
	struct message_queue *queue;  // 消息队列
	ATOM_POINTER logfile;  // 日志文件的指针（原子指针）
	uint64_t cpu_cost;	// in microsec CPU运行时间
	uint64_t cpu_start;	// in microsec 
	char result[32];  // 用来保存执行CMD命令的结果
	uint32_t handle;  // skynet_context 在全局映射的handle
	int session_id;  // 
	ATOM_INT ref;   // 引用计数 
	int message_count;  // 受到过的消息总数
	bool init;     // 是否初始化
	bool endless; // 是否死循环
	bool profile;  // 是否开启了profile 

	CHECKCALLING_DECL
};
```
在分析消息队列的时候，有说到全局的消息队列会不断dispatch消息到 skynet_context 的私有消息队列里。而work线程会从 skynet_context 的私有队列里不断消费数据。这边可以更详细的分析一下。

## 初始化skynetContext 

初始化 skynet_context_new 是通过 skynet_context_new 来进行的，这个函数在分析module、message_queue的时候已经有分析过了，这边完整的看一下

```
struct skynet_context * 
skynet_context_new(const char * name, const char *param) {

	// 加载module
	struct skynet_module * mod = skynet_module_query(name);
	if (mod == NULL)
		return NULL;

	// 创建module实例
	void *inst = skynet_module_instance_create(mod);
	if (inst == NULL)
		return NULL;
	struct skynet_context * ctx = skynet_malloc(sizeof(*ctx));
	CHECKCALLING_INIT(ctx)

	// 给 skynet_context  的所有变量赋初始值
	ctx->mod = mod;
	ctx->instance = inst;
	// 将 skynet_context 的引用计数设置为2
	ATOM_INIT(&ctx->ref , 2);
	ctx->cb = NULL;
	ctx->cb_ud = NULL;
	ctx->session_id = 0;
	// 初始化logfile 原子指针
	ATOM_INIT(&ctx->logfile, (uintptr_t)NULL);

	ctx->init = false;
	ctx->endless = false;

	ctx->cpu_cost = 0;
	ctx->cpu_start = 0;
	ctx->message_count = 0;
	ctx->profile = G_NODE.profile;

	// 初始化handle 和 message
	ctx->handle = 0;	
	ctx->handle = skynet_handle_register(ctx);
	struct message_queue * queue = ctx->queue = skynet_mq_create(ctx->handle);

	context_inc();

	// 执行module的初始化
	CHECKCALLING_BEGIN(ctx)
	int r = skynet_module_instance_init(mod, inst, ctx, param);
	CHECKCALLING_END(ctx)
	if (r == 0) {
		// 初始化成功的处理
		// 这边会减少初始化的引用计数
		struct skynet_context * ret = skynet_context_release(ctx);
		if (ret) {
			ctx->init = true;
		}
		// 将当前queue 塞入到global mq中，等待work线程处理
		skynet_globalmq_push(queue);
		if (ret) {
			skynet_error(ret, "LAUNCH %s %s", name, param ? param : "");
		}
		return ret;
	} else {
		// 初始化失败就释放
		skynet_error(ctx, "FAILED launch %s", name);
		uint32_t handle = ctx->handle;
		skynet_context_release(ctx);
		skynet_handle_retire(handle);
		struct drop_t d = { handle };
		skynet_mq_release(queue, drop_message, &d);
		return NULL;
	}
}
```
初始化流程中主要就是加载并实例化module，并创建message_queue提供给work进行处理。

在初始化的时候，可以看到引用计数被设置成2，这边初始化为2的原因我猜测是因为
1. 初始化流程算引用一次，所以初始化结束会减少一次 可以看到调用了skynet_context_release；
2. 另一次是因为注册到handle中， 可以看到skynet_handle_register并不会增加引用结束，但在执行 handle_exit的时候，会调用到 skynet_handle_retire，这里面会对ctx的引用技术减少一。

减少引用计数的调用如下
```
struct skynet_context * 
skynet_context_release(struct skynet_context *ctx) {
	if (ATOM_FDEC(&ctx->ref) == 1) {
		delete_context(ctx);
		return NULL;
	}
	return ctx;
}

```
ATOM_FDEC 会将ref减少1， 并返回调用前的值。所以当引用计数为1进来的时候，就会删掉对应的 skynet_context

## 回调函数的注册与回调时机

每个服务都会有自己的回调函数，这个回调函数的作用就是在收到message的时候，调用回来函数来进行处理。

### 注册时机
```
void 
skynet_callback(struct skynet_context * context, void *ud, skynet_cb cb) {
	context->cb = cb;
	context->cb_ud = ud;
}
```
每个服务通过 skynet_callback 来注册自己的回调，调用这个的时机都是在每个服务内部自己决定。
在skynet中的自带的几个服务，都是在init的时候调用 skynet_callback 来注册回调

### 回调时机
```
struct message_queue * 
skynet_context_message_dispatch(struct skynet_monitor *sm, struct message_queue *q, int weight) {
	...
	for (i=0;i<n;i++) {
		// 如果没有回调函数，则直接释放消息
		if (ctx->cb == NULL) {
			skynet_free(msg.data);
		} else {
			dispatch_message(ctx, &msg);
		}
	}
}

```
skynet_context_message_dispatch 这个函数是在work线程执行中回调用的函数，该函数会在ctx存在要处理的消息时调用

dispatch_message 具体函数如下
```
static void
dispatch_message(struct skynet_context *ctx, struct skynet_message *msg) {
	assert(ctx->init);
	CHECKCALLING_BEGIN(ctx)
	// 设置当前线程正在跑的handle
	pthread_setspecific(G_NODE.handle_key, (void *)(uintptr_t)(ctx->handle));
	int type = msg->sz >> MESSAGE_TYPE_SHIFT;
	size_t sz = msg->sz & MESSAGE_TYPE_MASK;

	// 打个收到消息的日志
	FILE *f = (FILE *)ATOM_LOAD(&ctx->logfile);
	if (f) {
		skynet_log_output(f, msg->source, type, msg->session, msg->data, sz);
	}
	++ctx->message_count;
	int reserve_msg;

	if (ctx->profile) {
		// 如果开启了profile，则会记录调用前后的CPU时间差
		ctx->cpu_start = skynet_thread_time();
		reserve_msg = ctx->cb(ctx, ctx->cb_ud, type, msg->session, msg->source, msg->data, sz);
		uint64_t cost_time = skynet_thread_time() - ctx->cpu_start;
		ctx->cpu_cost += cost_time;
	} else {
		reserve_msg = ctx->cb(ctx, ctx->cb_ud, type, msg->session, msg->source, msg->data, sz);
	}
	if (!reserve_msg) {
		skynet_free(msg->data);
	}
	CHECKCALLING_END(ctx)
}
```
dispatch_message 函数实际上就是调用了ctx函数的回调函数，只是在调用前后额外设置了环境相关的变量。

## 小结
skynet_context 的重要结构到这边就分析完了，skynet_context的作用主要就是给modules提供一个运行环境，拥有自己独立的数据结构，并提供接口给module让其能够用handle或者名字的形式给其他skynet_context 发送消息。