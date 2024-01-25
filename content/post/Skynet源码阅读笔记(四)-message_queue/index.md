---
title: Skynet源码阅读笔记(四)-message_queue
description: 
slug: 
date: 2024-1-24 00:00:00+0000
image: skynet.jpg
categories:
    - skynet
---

# Skynet源码阅读笔记-message

## skynet_message
在skynet中，两个服务之间是通过message传递消息来触发事件的

skynet_message 结构如下
```
struct skynet_message {
	uint32_t source; // 消息源的handle
	int session; // 消息的session Id
	void * data; // 消息的内容
	size_t sz; // 消息的长度 和 类型，
};

// type is encoding in skynet_message.sz high 8bit
#define MESSAGE_TYPE_MASK (SIZE_MAX >> 8)    
#define MESSAGE_TYPE_SHIFT ((sizeof(size_t)-1) * 8)

```

## message_queue
而保存 skynet_message 的结构则是 message_queue
```
struct message_queue {
	struct spinlock lock; // 自旋锁
	uint32_t handle;  // 对应的handle
	int cap;  // 当前消息队列容量
	int head; // 消息队列头
	int tail; // 消息队列尾
	int release;   //是否被释放
	int in_global;  // 标记是否在全局队列中
	int overload; // 是否超载
	int overload_threshold;  //超载阈值
	struct skynet_message *queue; // 实际的消息对了
	struct message_queue *next; // 下一个消息队列
}
```

message_queue 是一个简单的结构并不复杂的消息队列，本质上就是 由一个 循环数组 + 自旋锁构成。

操作 message_queue 的代码都在 skyenet_mq.c 中。本质上和操作普通队列没什么特别大区别，只是每次操作前都通过自旋锁来锁住操作。当消息通过 skynet_mq_push 塞进队列的时候，如果队列满了的话就会变成原来的2倍。

message_queue 中的handle 说明了这个 message_queue 绑定上的对应的 skynet_context。在skynet_context_new中可以看到对应的代码
```
struct skynet_context * 
skynet_context_new(const char * name, const char *param) {

    ...
    ctx->handle = 0;	
	ctx->handle = skynet_handle_register(ctx);  // 这一步就是笔记三中的部分
	struct message_queue * queue = ctx->queue = skynet_mq_create(ctx->handle); // 获取handle后，就创建 message_queue， 并绑定上对应的handle
    ...
}
```

## global_queue
mesaage_queue 在接收到消息后, 会将自己放入到global_queue 这个结构中
```
struct global_queue {
	struct message_queue *head; // mesaage_queue 的链表头
	struct message_queue *tail; // mesaage_queue 的链表尾
	struct spinlock lock; // 自旋锁
};

static struct global_queue *Q = NULL;
```


## 数据的消费以及产生
这个结构是一个全局变量，它管理着目前有消息的消息队列。在 skynet_context_message_dispatch 函数中，message_queue的message 会被消费

```
struct message_queue * 
skynet_context_message_dispatch(struct skynet_monitor *sm, struct message_queue *q, int weight) {
    if (q == NULL) { 
		q = skynet_globalmq_pop();  // 获取全局消息队列的头部
		if (q==NULL)
			return NULL;
	}

	uint32_t handle = skynet_mq_handle(q);
    struct skynet_context * ctx = skynet_handle_grab(handle); // 找到对应的 skynet_context
    .....

    int i,n=1;
	struct skynet_message msg;

	for (i=0;i<n;i++) {
        //从队列头部获取消息
		if (skynet_mq_pop(q,&msg)) { 
			...
		} 
        .....
		
		if (ctx->cb == NULL) {
			skynet_free(msg.data);
		} else {
            // 触发消息事件
			dispatch_message(ctx, &msg);
		}
        ...
	}

}

```

可以看到skynet_context_message_dispatch 函数就是从全局队列拿到队列的消息列表后，在从消息列表中获取到对应消息来消费。

这个函数skynet_context_message_dispatch 实际上是被包装在thread_worker，在初始化的时候由多个线程一起调用，所以需要加锁。


而对于一个消息，它会通过下面两个接口塞入到消息队列中
```

int
skynet_context_push(uint32_t handle, struct skynet_message *message) {
	struct skynet_context * ctx = skynet_handle_grab(handle);
	if (ctx == NULL) {
		return -1;
	}
	skynet_mq_push(ctx->queue, message);
	skynet_context_release(ctx);

	return 0;
}

void
skynet_context_send(struct skynet_context * ctx, void * msg, size_t sz, uint32_t source, int type, int session) {
	struct skynet_message smsg;
	smsg.source = source;
	smsg.session = session;
	smsg.data = msg;
	smsg.sz = sz | (size_t)type << MESSAGE_TYPE_SHIFT;

	skynet_mq_push(ctx->queue, &smsg);
}

```
skynet.send 最后也是调用的 skynet_context_push， messsage会在外面包装好，然后直接丢到对应的 skynet_context 的 message_queue中。

skynet_harbor_send 则是调用的 skynet_context_send 来把消息塞入到对应的message中。