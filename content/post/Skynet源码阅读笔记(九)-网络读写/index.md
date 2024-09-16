---
title: Skynet源码阅读笔记(二)-SkynetModule
description: 
slug: 
date: 2023-11-21 00:00:00+0000
image: skynet.jpg
categories:
    - skynet
---

# Skynet源码阅读笔记-SkynetModule

skynet的服务可以认为是 SkynetModule 的实例，skynet本身自带了4个类型的服务，gate、logger、snlua、 harbor。 自带的这几个服务位于 service-src下。

承接上文，logger 通过 skynet_context_new 进行创建，在 skynet_context_new 中可以看到对 skynet-module 的引用。
```
struct skynet_context * 
skynet_context_new(const char * name, const char *param) {

    // 根据名字来获取一个module 
	struct skynet_module * mod = skynet_module_query(name);
	printf("in skynet_context_new %s %s\n", name, param);
	if (mod == NULL)
		return NULL;

    // 创建出module的实例
	void *inst = skynet_module_instance_create(mod);
	if (inst == NULL)
		return NULL;

    // 创建Ctx
	struct skynet_context * ctx = skynet_malloc(sizeof(*ctx));
	CHECKCALLING_INIT(ctx)

	ctx->mod = mod;
	ctx->instance = inst;
    //  设置其他cxt的变量初始值
    .......
	CHECKCALLING_BEGIN(ctx)
    // mod 初始化
	int r = skynet_module_instance_init(mod, inst, ctx, param);
	CHECKCALLING_END(ctx)
	...
}
```
skynet_context_new 本身是为了创建 skynet_context， 但可以看到 skynet_context 中加载对应 skynet_module , 并创建了对应的实例， 下面就具体看看怎么实现的。


## skynet_module 基本机构
先看看skynet_module的结构
```
typedef void * (*skynet_dl_create)(void);
typedef int (*skynet_dl_init)(void * inst, struct skynet_context *, const char * parm);
typedef void (*skynet_dl_release)(void * inst);
typedef void (*skynet_dl_signal)(void * inst, int signal);

struct skynet_module {
	const char * name; // 名字
	void * module;  // module的地址
	skynet_dl_create create;  // 创建module实例执行的函数 
	skynet_dl_init init;  // 初始化module执行的函数
	skynet_dl_release release; // 释放module执行函数
	skynet_dl_signal signal;  //  module信号执行函数
};
```
对于skynet来说，一个module相当于有一个名字以及对应的4个需要执行的函数

在skynet_context_new中有一个skynet_module_query，这个函数会将module导入进来

```
struct modules {
	int count;
	struct spinlock lock;
	const char * path;
	struct skynet_module m[MAX_MODULE_TYPE];
};
```
导入进来的 skynet_module 会被保存在 modules 的数组中，MAX_MODULE_TYPE 默认是32.

### skynet_module_query 导入过程

```

// 查找一下当前的M中释放已经存在了对应的module
static struct skynet_module * 
_query(const char * name) {
	int i;
	for (i=0;i<M->count;i++) {
		if (strcmp(M->m[i].name,name)==0) {
			return &M->m[i];
		}
	}
	return NULL;
}

struct skynet_module * 
skynet_module_query(const char * name) {
    //查找一下当前的M中释放已经存在了对应的module
	struct skynet_module * result = _query(name);
	if (result)
		return result;

    //没有的话就需要加锁导入
	SPIN_LOCK(M)

	result = _query(name); // double check

	if (result == NULL && M->count < MAX_MODULE_TYPE) {
		int index = M->count;
		void * dl = _try_open(M,name);  // 尝试导入对应名字的so
		if (dl) {
			M->m[index].name = name;
			M->m[index].module = dl;

			if (open_sym(&M->m[index]) == 0) { // 将对应的4个函数进行赋值
				M->m[index].name = skynet_strdup(name);
				M->count ++;
				result = &M->m[index];  //完成后返回导入的module
			}
		}
	}

	SPIN_UNLOCK(M)

	return result;
}
```
skynet_module_query 执行的行为很简单, 如果当前缓存内不存在，就加锁导入进行导入，导入时通过_try_open进行的，导入完成后需要对skynet_module内的所有变量进行赋值

open_sym 是对传入的skynet_module的4个函数进行赋值，里面是使用字符串拼接的方式将module名字和函数名字拼在一起后通过dlsym的方式找到对应的函数

_try_open 实际上是dlopen的包装，在给定的搜索路径内搜索对应的so文件，然后尝试用dlopen打开。

skynet_module.c 内的其他接口就是对结构体内的保存的4个函数地址的封装


## 小结

skynet_module 这个结构本身就是对so的一个包装，如果一个so需要通过这种方式加载到skynet内当作服务的话，则需要实现对应的4个方法。
