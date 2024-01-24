---
title: Skynet源码阅读笔记(三)-SkynetetHandle
description: 
slug: 
date: 2023-11-21 00:00:00+0000
image: skynet.jpg
categories:
    - skynet
---

# Skynet源码阅读笔记-SkynetetHandle

在一个skyent进程中，handle的作用是用来通过它查找对应注册的ctx。

在 skynet_context_new 创建一个新的ctx的时候， 会通过 skynet_handle_register 来获取 ctx 对应的 handle。下面看看具体是怎么玩的。

## handle_storage
先看一下 handle_storage 这个结构
```

struct handle_name {
	char * name;
	uint32_t handle;
};

struct handle_storage {
	struct rwlock lock; // 读写锁

	uint32_t harbor;  // 本skynet节点的harbor

	uint32_t handle_index;  // 下一个handle 插入时查找的位置
	int slot_size;  // slot 的长度
	struct skynet_context ** slot; // skynet_context 数组 
	
	int name_cap;  // 当前数组的长度
	int name_count;  // 当前使用个数
	struct handle_name *name;   // 名字映射handle的数组
};

static struct handle_storage *H = NULL;
```
handle_storage结构很简单，就是一个 skynet_context 的数组，只是因为它是一个全局的对象，所以需要一个读写锁来控制访问。

## skynet_handle_register

```
uint32_t
skynet_handle_register(struct skynet_context *ctx) {
	struct handle_storage *s = H;

	rwlock_wlock(&s->lock);
	
	for (;;) {
		int i;
		uint32_t handle = s->handle_index;

		// 从handle_index 位置开始找一个可以插入的位置
		for (i=0;i<s->slot_size;i++,handle++) { 
			if (handle > HANDLE_MASK) {
				// 0 is reserved
				handle = 1;
			}
			int hash = handle & (s->slot_size-1);
			if (s->slot[hash] == NULL) {
				// 找到空位就插入返回
				s->slot[hash] = ctx;
				s->handle_index = handle + 1;

				rwlock_wunlock(&s->lock);

				handle |= s->harbor;
				return handle;
			}
		}
		assert((s->slot_size*2 - 1) <= HANDLE_MASK);
		// 需要扩容了
		struct skynet_context ** new_slot = skynet_malloc(s->slot_size * 2 * sizeof(struct skynet_context *));
		memset(new_slot, 0, s->slot_size * 2 * sizeof(struct skynet_context *));
		for (i=0;i<s->slot_size;i++) {
			if (s->slot[i]) {
				int hash = skynet_context_handle(s->slot[i]) & (s->slot_size * 2 - 1);
				assert(new_slot[hash] == NULL);
				new_slot[hash] = s->slot[i];
			}
		}
		skynet_free(s->slot);
		s->slot = new_slot;
		s->slot_size *= 2;
		// 扩容完成后再插入一遍
	}
}

```

skynet_handle_register 需要做的事情就是将 ctx 插入到一个当前空着的 slot 中。

因为是修改数组，所以进入函数的时候就获取写锁，这边从 handle_index 开始遍历数组，转一圈如果没找到空位就将数组扩充成原来的2倍，然后再执行一个插入。找到空位后将数组下标返回，这个数组下标就是 skynet_context 的handle。

稍微值得注意的是，因为返回的是下标，所以扩容的时候原来位置对应的元素还是应该一样的。

## skynet_handle_namehandle 
这个接口是用名字和handle进行把绑定
```
static const char *
_insert_name(struct handle_storage *s, const char * name, uint32_t handle) {
	int begin = 0;
	int end = s->name_count - 1;
	// 二分查找对应插入点
	while (begin<=end) {
		int mid = (begin+end)/2;
		struct handle_name *n = &s->name[mid];
		int c = strcmp(n->name, name);
		if (c==0) {
			return NULL;
		}
		if (c<0) {
			begin = mid + 1;
		} else {
			end = mid - 1;
		}
	}
	char * result = skynet_strdup(name);

	_insert_name_before(s, result, handle, begin); // 真正插入的行为执行点

	return result;
}

const char * 
skynet_handle_namehandle(uint32_t handle, const char *name) {
	rwlock_wlock(&H->lock);

	const char * ret = _insert_name(H, name, handle);

	rwlock_wunlock(&H->lock);

	return ret;
}

```
在 _insert_name 中可以很明显的看出来， handle_storage 的 name 数组是一个有序的数组，插入的时候通过二分来找到插入的位置，然后调用 _insert_name_before 完成插入， _insert_name_before 中包含了对该数组的扩容以及元素移动等行为。

## skynet_handle_retire 
看一下删除操作
```
int
skynet_handle_retire(uint32_t handle) {
	int ret = 0;
	struct handle_storage *s = H;

	rwlock_wlock(&s->lock);

	uint32_t hash = handle & (s->slot_size-1);
	struct skynet_context * ctx = s->slot[hash];

	if (ctx != NULL && skynet_context_handle(ctx) == handle) {
		s->slot[hash] = NULL;
		ret = 1;
		int i;
		int j=0, n=s->name_count;
		// 开始删除name中的对应关系
		for (i=0; i<n; ++i) {
			if (s->name[i].handle == handle) {
				skynet_free(s->name[i].name);
				continue;
			} else if (i!=j) {
				s->name[j] = s->name[i];
			}
			++j;
		}
		s->name_count = j;
	} else {
		ctx = NULL;
	}

	rwlock_wunlock(&s->lock);

	if (ctx) {
		// release ctx may call skynet_handle_* , so wunlock first.
		skynet_context_release(ctx);
	}

	return ret;
}
```

因为handle 本身就是数组下标，所以对于删除 slot 来说很简单，相对麻烦的事情是删name里面的对应，需要遍历和移动整个数组。


## skynet_handle_init

```
void 
skynet_handle_init(int harbor) {
	assert(H==NULL);
	struct handle_storage * s = skynet_malloc(sizeof(*H));
	s->slot_size = DEFAULT_SLOT_SIZE;
	s->slot = skynet_malloc(s->slot_size * sizeof(struct skynet_context *));
	memset(s->slot, 0, s->slot_size * sizeof(struct skynet_context *));

	rwlock_init(&s->lock);
	// reserve 0 for system
	s->harbor = (uint32_t) (harbor & 0xff) << HANDLE_REMOTE_SHIFT;
	s->handle_index = 1;
	s->name_cap = 2;
	s->name_count = 0;
	s->name = skynet_malloc(s->name_cap * sizeof(struct handle_name));

	H = s;

	// Don't need to free H
}
```

整个结构初始化的行为 在 skynet_start 的时候调用 skynet_handle_init，这边简单的为几个数组长度进行了初始化。稍微有点点疑惑的是 特别注释了一个 不用free H。 那H的内存什么时候释放呢？
