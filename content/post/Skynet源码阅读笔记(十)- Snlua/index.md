---
title: Skynet源码阅读笔记(十)- Snlua 服务
description: 
slug: 
date: 2024-08-10 00:00:00+0000
image: skynet.png
categories:
    - skynet
---

# Skynet源码阅读笔记-Snlua服务


snlua 服务是 skynet 中重要的服务之一，其主要功能是为了创建一个执行lua代码的服务, skynet在默认情况下执行的第一个服务就是 snlua bootstrap

先看看其主要的的数据结构

```
struct snlua {
	lua_State * L;   // lua 服务的主线程
	struct skynet_context * ctx;  // skynet服务的上下文
	size_t mem;      //  目前的内存使用
	size_t mem_report;   // 内存警报阈值
	size_t mem_limit;    // 内存使用上限 
	lua_State * activeL; // 当前的活跃的lua 线程
 	ATOM_INT trap;    // 标记设置signal_hook
};
```

之前在skynet_module中曾经提到过，创建一个新的服务时，对应的服务需要实现4个函数，用于初始化和释放
```
struct skynet_module {
	const char * name; // 名字
	void * module;  // module的地址
	skynet_dl_create create;  // 创建module实例执行的函数 
	skynet_dl_init init;  // 初始化module执行的函数
	skynet_dl_release release; // 释放module执行函数
	skynet_dl_signal signal;  //  module信号执行函数
};
```

当skynet在初始化的时候，执行的第一个snlua的服务是snlua bootstrap, 以这个为例子看看snlua中对应的函数是如何实现的。

## snlua_create

```
struct snlua *
snlua_create(void) {
	struct snlua * l = skynet_malloc(sizeof(*l));
	memset(l,0,sizeof(*l));
	l->mem_report = MEMORY_WARNING_REPORT;
	l->mem_limit = 0;
	l->L = lua_newstate(lalloc, l);
	l->activeL = NULL;
	ATOM_INIT(&l->trap , 0);
	return l;
}

```
snlua的创建函数，可以只是对上述属性进行一个简单的初始化操作，可以注意到在lua_newstate的时候传入了一个lalloc, 在这边对lua内存分配做了一个自定义的行为。

### lalloc
```
static void *
lalloc(void * ud, void *ptr, size_t osize, size_t nsize) {
	struct snlua *l = ud;
	size_t mem = l->mem;
	l->mem += nsize;
	if (ptr)
		l->mem -= osize;
	if (l->mem_limit != 0 && l->mem > l->mem_limit) {
		if (ptr == NULL || nsize > osize) {
			l->mem = mem;
			return NULL;
		}
	}
	if (l->mem > l->mem_report) {
		l->mem_report *= 2;
		skynet_error(l->ctx, "Memory warning %.2f M", (float)l->mem / (1024 * 1024));
	}
	return skynet_lalloc(ptr, osize, nsize);
}
```

lalloc 中主要干了两件事情
1. 对内存的分配进行了监控，超过一定限制会输出日志报警
2. 调用skynet_lalloc来实际分配内存，这边先不展开，不过skynet_lalloc中使用的是jemalloc来对内存进行分配。

## snlua_init

```
int
snlua_init(struct snlua *l, struct skynet_context *ctx, const char * args) {

    // 先把参数拷贝出来
	int sz = strlen(args);
	char * tmp = skynet_malloc(sz);
	memcpy(tmp, args, sz);  // args 这边是bootstrap

	skynet_callback(ctx, l, launch_cb); // 设置消息回调的接口和userData，这边设置进行去的ud是 l
    // 调用REG 来获取handle_id
	const char * self = skynet_command(ctx, "REG", NULL);
	uint32_t handle_id = strtoul(self+1, NULL, 16);

	// it must be first message 
	skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz);  // 通过给自己发送一条消息的方式来触发后续的初始化操作
	return 0;
}

```

init流程看似也很简单，但实际上是为了调用skynet_send来给自己发送一条消息，用消息回调的方式来触发剩余的初始化操作。
不过为啥要用回调的方式以及为啥这个消息必须是第一条，我目前没有理解。



###  launch_cb

消息回调的时候首先调用的是launch_cb，在init的时候已经通过skynet_callback将ud和launch_cb设置到消息回调中了

```
static int
launch_cb(struct skynet_context * context, void *ud, int type, int session, uint32_t source , const void * msg, size_t sz) {
	assert(type == 0 && session == 0);
	struct snlua *l = ud;
	skynet_callback(context, NULL, NULL);  // 这边又将回调的接口给重置了
	int err = init_cb(l, context, msg, sz);  // 真正的初始化位置
	if (err) {
		skynet_command(context, "EXIT", NULL);
	}

	return 0;
}
```

launch_cb 也只是用来包装调用init_cb，自身只是处理了如果init_cb出错了，就调用EXIT指令卸载掉当前的服务

### init_cb

init_cb 中就是初始化snlua服务的主要流程了，主要是 hook一些协程接口、处理路径相关、加载执行lua文件
```
static int
init_cb(struct snlua *l, struct skynet_context *ctx, const char * args, size_t sz) {
	lua_State *L = l->L;
	l->ctx = ctx;
	lua_gc(L, LUA_GCSTOP, 0); // GC STOP
	lua_pushboolean(L, 1);  /* signal for libraries to ignore env. vars. */
	lua_setfield(L, LUA_REGISTRYINDEX, "LUA_NOENV"); // 跳过LUA_PATH和LUA_CPATH
	luaL_openlibs(L);
	luaL_requiref(L, "skynet.profile", init_profile, 0);  // require skynet profile

    // hook coroutine相关接口, 相当于下面的lua代码
	// replace coroutine.resume / coroutine.wrap
	// coroutine[resume] = profile_lib[resume]
	// coroutine[wrap] = profile_lib[wrap]
    ... 

	// 相当于 LUA_REGISTRYINDEX[skynet_context] = ctx 
	lua_getglobal(L, "coroutine");
	lua_getfield(L, profile_lib, "resume");
	lua_setfield(L, -2, "resume");
	lua_getfield(L, profile_lib, "wrap");
	lua_setfield(L, -2, "wrap");
    ...

	// 设置路径，如果配置有提供lua_path、lua_cpath、luaservice 则使用配置的， 否则使用默认的
	const char *path = optstring(ctx, "lua_path","./lualib/?.lua;./lualib/?/init.lua");
    ....

	// 加载lua loader， 
	lua_pushcfunction(L, traceback);
	assert(lua_gettop(L) == 1);
	
	const char * loader = optstring(ctx, "lualoader", "./lualib/loader.lua");
	int r = luaL_loadfile(L,loader);
	if (r != LUA_OK) {
		skynet_error(ctx, "Can't load %s : %s", loader, lua_tostring(L, -1));
		report_launcher_error(ctx);
		return 1;
	}

	// pcall 调用 如果报错了，则把错误信息打印出来
	// 这边一开始的args应该是 bootstrap
	lua_pushlstring(L, args, sz);
	r = lua_pcall(L,1,0,1);
	if (r != LUA_OK) {
		skynet_error(ctx, "lua loader error : %s", lua_tostring(L, -1));
		report_launcher_error(ctx);
		return 1;
	}
	
	// 如果 LUA_REGISTRYINDEX[memlimit] 有被设置的话， 则更新mem_limit
    if (lua_getfield(L, LUA_REGISTRYINDEX, "memlimit") == LUA_TNUMBER) {
        ...
    }

	lua_gc(L, LUA_GCRESTART, 0);

	return 0;
}


```

init_cb 里面主要是使用了很多Lua的C API来执行了一系列操作，分为一下几个步骤
1. 替换掉系统的coroutine相关的操作
2. 将ctx设置到全局变量中，以便C 和 lua更好的交互
3. 设置require的路径
4. 加载 loader.lua 
5. 调用 loader.lua args 。这边的args 内容最开始是 bootstrap.lua
6. 设置 memlimit 

这边目前更关心 loader.lua bootstap 这个调用的过程


```
-- skynet.loader.lua

-- 其他更多是在处理配置路径相关的事情
-- main = load(bootstap.lua) snlua初始化的时候执行的是这个语句
... 

-- 如果定义了LUA_PRELOAD，那么就先加载对应文件
if LUA_PRELOAD then
	local f = assert(loadfile(LUA_PRELOAD))
	f(table.unpack(args))
	LUA_PRELOAD = nil
end

_G.require = (require "skynet.require").require  -- skynet.require 后续新开一个文档研究，这边可以认为是对于协程并行的一些处理，如果没有使用协程，那么这边就是普通的require

main(select(2, table.unpack(args)))  -- 这边就是执行服务的函数，main 对应的函数就是loadfile进来的

```

loader.lua 中大部分都是处理服务路径相关的事情，需要注意的就是如果定义了LUA_PRELOAD，那么就会提前加载对应的模块。
顺便再提一嘴，main函数是通过 loadfile(targeServiceFile) 的结果。

这边args再初始化加载的时候应该只有{bootstap}，所以main执行的时候是没有参数的。

init的流程先到此为止，这边知道skynet snlua服务会在init_cb的时候通过loadfile的方式将lua服务加载进来就行了。

之后具体的bootstrap流程还会具体在分析。这边先告一段落。

后续看看snlua的另外几个操作

## snlua_signal
singla接口主要是为了能让其他线程给snlua服务在跑的过程发送一些信号，以达到一些自定义的需求。

```
void
snlua_signal(struct snlua *l, int signal) {
	skynet_error(l->ctx, "recv a signal %d", signal);
	if (signal == 0) {
		if (ATOM_LOAD(&l->trap) == 0) {
			// only one thread can set trap ( l->trap 0->1 )
			if (!ATOM_CAS(&l->trap, 0, 1))
				return;
			lua_sethook (l->activeL, signal_hook, LUA_MASKCOUNT, 1);  -- 相当于debug.sethook(signal_hook, "count", 1)
			// finish set ( l->trap 1 -> -1 )
			ATOM_CAS(&l->trap, 1, -1);  -- 设置成功了就将trap 设置成-1
		}
	} else if (signal == 1) {
		skynet_error(l->ctx, "Current Memory %.3fK", (float)l->mem / 1024);
	}
}
```

snlua_signal目前只处理了两个信号事件
- 当信号为1的时候，snlua会输出当前的内存使用
- 当新号为0的时候，会执行 相当于 相当于debug.sethook(signal_hook, "count", 1)的 语句，
```
static void
signal_hook(lua_State *L, lua_Debug *ar) {
	void *ud = NULL;
	lua_getallocf(L, &ud);
	struct snlua *l = (struct snlua *)ud;

	lua_sethook (L, NULL, 0, 0);
	if (ATOM_LOAD(&l->trap)) {
		ATOM_STORE(&l->trap , 0); -- 走到这边说明必定触发，则先将trap设置会0
		luaL_error(L, "signal 0");  -- 抛出异常
	}
}
```
这个signal_hook在执行的时候就会将hook解除，并立即抛出异常。

还记得在分析线程作用的一章里，有谈到过关于死循环检查的事情吗，当在日志中发现存在死循环后，可以通过给snlua服务发送信号0的方式来打断它的执行，从而跳出死循环。

## snlua_release
```
void
snlua_release(struct snlua *l) {
	lua_close(l->L);
	skynet_free(l);
}
```
snlua_release 这边就很简单了，关闭lua的虚拟机、释放对应的内存即可


## 小结

这边主要过了一下snlua服务中4个主要函数的大致实现。其中比较重要的就是初始化的过程和信号处理的过程。唯一还没理解的地方就是为啥snlua服务需要通过给自己发信息的方式来触发初始化过程。
snlua在init_cb的时候，代码就来到了lua层，后面就准备分析一些bootstrap.lua中做了什么事情。
其实snlua这边还有很大篇幅是有关协程调度的（只是hook了调度函数，用来做profile和信号打断的），在源码中定义在init_profile函数里，有机会在分析这块内容。




