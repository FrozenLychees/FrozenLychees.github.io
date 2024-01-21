---
title: Skynet源码阅读笔记-启动流程
description: 
slug: 
date: 2023-11-21 00:00:00+0000
image: skynet.jpg
categories:
    - skynet
---

# Skynet源码阅读笔记-启动流程

## 主流程
main函数在skynet_main.c中, 主函数里的流程看起来不算很长，这边全贴出来，然后分块分析

```
int
main(int argc, char *argv[]) {


  // 1. 判断是否传入了配置，没有的话就退出了
	const char * config_file = NULL ;
	if (argc > 1) {
		config_file = argv[1];
	} else {
		fprintf(stderr, "Need a config file. Please read skynet wiki : https://github.com/cloudwu/skynet/wiki/Config\n"
			"usage: skynet configfilename\n");
		return 1;
	}

  // 2. 初始化skynet的主线程相关的变量
	skynet_globalinit();

  // 3. 初始化skynet的环境，其中包括一个lua环境和一个自旋锁
	skynet_env_init();

  // 4. 信号处理相关
	sigign();
  
	struct skynet_config config;

#ifdef LUA_CACHELIB
	// init the lock of code cache
	luaL_initcodecache();
#endif

  // 初始化一个lua虚拟机，用来读取配置
	struct lua_State *L = luaL_newstate();
	luaL_openlibs(L);	// link lua lib

  // 执行一段硬编码的lua代码，这个代码写在skynet_main.c文件里，就是load_config变量所对应的内容
  // load_config就是包装load进行的加载配置函数
  // 这边是把load_config变成lua函数然后执行加载
	int err =  luaL_loadbufferx(L, load_config, strlen(load_config), "=[skynet config]", "t");
	assert(err == LUA_OK);
	lua_pushstring(L, config_file);

	err = lua_pcall(L, 1, 1, 0);
	if (err) {
		fprintf(stderr,"%s\n",lua_tostring(L,-1));
		lua_close(L);
		return 1;
	}
  // 这个函数看起来是把刚刚读取的配置设置到lua env全局变量中
	_init_env(L);

  // 初始化skynet的基础配置
	config.thread =  optint("thread",8);
	config.module_path = optstring("cpath","./cservice/?.so");
	config.harbor = optint("harbor", 1);
	config.bootstrap = optstring("bootstrap","snlua bootstrap");
	config.daemon = optstring("daemon", NULL);
	config.logger = optstring("logger", NULL);
	config.logservice = optstring("logservice", "logger");
	config.profile = optboolean("profile", 1);

	lua_close(L);

  //skynet 框架启动
	skynet_start(&config);
	skynet_globalexit();

	return 0;
}
```


##skynet_start 
从上面的代码解析，可以清楚的知道，skynet启动的主要函数就是skynet_start，看看它做了什么
```
void 
skynet_start(struct skynet_config * config) {
	// register SIGHUP for log file reopen
  // 首先是注册了一个SIGHUP的信号执行函数
	struct sigaction sa;
	sa.sa_handler = &handle_hup;
	sa.sa_flags = SA_RESTART;
	sigfillset(&sa.sa_mask);
	sigaction(SIGHUP, &sa, NULL);

  // 看配置是否创建守护进程
	if (config->daemon) {
		if (daemon_init(config->daemon)) {
			exit(1);
		}
	}

  // 初始化 Harbor节点的数量
	skynet_harbor_init(config->harbor);
  // 初始化handle的存储
	skynet_handle_init(config->harbor);
  // 初始化全局消息队列
	skynet_mq_init();
  // 初始化模块的查找路径
	skynet_module_init(config->module_path);
  // 初始化定时器相关
	skynet_timer_init();
  // 初始化socket相关
	skynet_socket_init();
  // profile是否打开
	skynet_profile_enable(config->profile);

  // 初始化logger相关服务
	struct skynet_context *ctx = skynet_context_new(config->logservice, config->logger);
	if (ctx == NULL) {
		fprintf(stderr, "Can't launch %s service\n", config->logservice);
		exit(1);
	}
	skynet_handle_namehandle(skynet_context_handle(ctx), "logger");

  // 启动config->bootstrap 所配置的服务
	bootstrap(ctx, config->bootstrap);

  // 启动线程
	start(config->thread);

	// harbor_exit may call socket send, so it should exit before socket_free
	skynet_harbor_exit();
	skynet_socket_free();
	if (config->daemon) {
		daemon_exit(config->daemon);
	}
}
```

这段代码主要就是初始化skynet的组件，然后启动logger、定时器等相关服务，然后调用bootstrap 和 start

## bootstrap
```
static void
bootstrap(struct skynet_context * logger, const char * cmdline) {
	int sz = strlen(cmdline);
	char name[sz+1];
	char args[sz+1];
	int arg_pos;
	sscanf(cmdline, "%s", name);  
	arg_pos = strlen(name);
	if (arg_pos < sz) {
		while(cmdline[arg_pos] == ' ') {
			arg_pos++;
		}
		strncpy(args, cmdline + arg_pos, sz);
	} else {
		args[0] = '\0';
	}
	struct skynet_context *ctx = skynet_context_new(name, args);
	if (ctx == NULL) {
		skynet_error(NULL, "Bootstrap error : %s\n", cmdline);
		skynet_context_dispatchall(logger);
		exit(1);
	}
}

```
bootstrap 主要就是根据config->bootstrap配置来启动一个新的服务，在默认配置下一般都是snlua bootstrap

## start
start函数主要是启动所有对应个数的线程，具体代码如下
```
static void
start(int thread) {
	pthread_t pid[thread+3];  //额外多3个线程

	struct monitor *m = skynet_malloc(sizeof(*m));
	memset(m, 0, sizeof(*m));
	m->count = thread;
	m->sleep = 0;

  //给每个线程都分配一个skynet_monitor
	m->m = skynet_malloc(thread * sizeof(struct skynet_monitor *));
	int i;
	for (i=0;i<thread;i++) { // 这个循环稍微有一点疑问，要么应该是i<thread+3 要么应该i从3开始，或者是我没理解正确理解
		m->m[i] = skynet_monitor_new();
	}

  // 初始化monitor的锁和条件变量
	if (pthread_mutex_init(&m->mutex, NULL)) {
		fprintf(stderr, "Init mutex error");
		exit(1);
	}
	if (pthread_cond_init(&m->cond, NULL)) {
		fprintf(stderr, "Init cond error");
		exit(1);
	}

  // 额外的三个线程用于定时器、监控、socket
	create_thread(&pid[0], thread_monitor, m);
	create_thread(&pid[1], thread_timer, m);
	create_thread(&pid[2], thread_socket, m);

	static int weight[] = { 
		-1, -1, -1, -1, 0, 0, 0, 0,
		1, 1, 1, 1, 1, 1, 1, 1, 
		2, 2, 2, 2, 2, 2, 2, 2, 
		3, 3, 3, 3, 3, 3, 3, 3, };
	struct worker_parm wp[thread];

  // 给线程分配权重，并执行thread_worker
	for (i=0;i<thread;i++) {
		wp[i].m = m;
		wp[i].id = i;
		if (i < sizeof(weight)/sizeof(weight[0])) {
			wp[i].weight= weight[i];
		} else {
			wp[i].weight = 0;
		}
		create_thread(&pid[i+3], thread_worker, &wp[i]);
	}

  //主线程等待其他线程结束
	for (i=0;i<thread+3;i++) {
		pthread_join(pid[i], NULL); 
	}

	free_monitor(m);
}

```

主线程会额外启动三个线程，分别用于定时器、监控、socket， 在启动完毕所有线程之后，会阻塞等待所有线程的退出。对于每一个线程来说，还会再weight中有对应的权重，权重的作用先不探究。

