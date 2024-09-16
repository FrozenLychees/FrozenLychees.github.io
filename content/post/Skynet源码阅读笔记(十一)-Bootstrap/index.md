---
title: Skynet源码阅读笔记(十一)-Bootstrap 与 cmaster/cslave
description: 
slug: 
date: 2024-08-13 00:00:00+0000
image: skynet.png
categories:
    - skynet
---

# Bootstrap

bootstrap是skynet默认初始化第一个执行的lua文件，上一遍在梳理snlua的过程中，知道了snlua是如何在初始化的时候执行到lua层的代码的，那接下来就看看在初始化时bootstrap具体的过程。


## bootstrap.lua

```
skynet.start(function()
	local launcher = assert(skynet.launch("snlua","launcher"))
	skynet.name(".launcher", launcher)

	local standalone = skynet.getenv "standalone"
	local harbor_id = tonumber(skynet.getenv "harbor" or 0)
	if harbor_id == 0 then

		-- 这边是单节点模式启动
		assert(standalone ==  nil)
		standalone = true
		skynet.setenv("standalone", "true")

		-- 用cdummy 来代替cslave
		local ok, slave = pcall(skynet.newservice, "cdummy")
		if not ok then
			skynet.abort()
		end
		skynet.name(".cslave", slave)

	else

		-- 这边是多节点模式
		if standalone then
			-- 如果standalone有配置，则需要启动cmaster
			if not pcall(skynet.newservice,"cmaster") then
				skynet.abort()
			end
		end

		-- 无论什么类型，cslave一定都有
		local ok, slave = pcall(skynet.newservice, "cslave")
		if not ok then
			skynet.abort()
		end
		skynet.name(".cslave", slave)
	end

	-- 这边还需要给master节点启动datecenter
	if standalone then
		local datacenter = skynet.newservice "datacenterd"
		skynet.name("DATACENTER", datacenter)
	end
	skynet.newservice "service_mgr"

	....

	-- 启动用户自定义的脚本
	pcall(skynet.newservice,skynet.getenv "start" or "main")
	skynet.exit()
end)


```
1. 先启动一个launcher的服务
2. bootstrap 根据配置中是否配置了 harbor 来分成两个模式
	- 如果配置了没harbor，那么就一定是以单节点的方式启动，当前进程会额外启动一个cdummy的服务作为 cslave
	- 如果配置了harbor，那么还需要根据配置判断standlalone来判断是否启动 cmaster 服务 和 datecenter 服务；配置了harbor的节点一定会启动 cslave 服务
3. 无论什么模式，最后根据配置获取start 或者 main 脚本 来启动业务服务，然后本服务退出

## cmaster / cslave

cmaster / cslave 这边的源码就不具体分析了
### cmaster
cmaster 和 cslave 在 https://github.com/cloudwu/skynet/wiki/Cluster 中也有介绍，主要是用来局域网内的多节点之间的一个使用方式

master主要管理slave的节点信息，包括地址和名字等。主要的协议交互如下
```
--[[
	master manage data :
		1. all the slaves address : id -> ipaddr:port
		2. all the global names : name -> address

	master hold connections from slaves .

	protocol slave->master :
		package size 1 byte
		type 1 byte :
			'H' : HANDSHAKE, report slave id, and address.
			'R' : REGISTER name address
			'Q' : QUERY name


	protocol master->slave:
		package size 1 byte
		type 1 byte :
			'W' : WAIT n
			'C' : CONNECT slave_id slave_address
			'N' : NAME globalname address
			'D' : DISCONNECT slave_id
]]
```

master的启动过程如下
```
skynet.start(function()
	local master_addr = skynet.getenv "standalone"
	skynet.error("master listen socket " .. tostring(master_addr))
	local fd = socket.listen(master_addr)  -- 监听对应端口
	socket.start(fd , function(id, addr)  -- 在连接进来的时候，执行这个函数
		skynet.error("connect from " .. addr .. " " .. id)
		socket.start(id)
		local ok, slave, slave_addr = pcall(handshake, id)  -- 先进行handshake进行握手
		if ok then
			skynet.fork(monitor_slave, slave, slave_addr)  -- 成功了之后在创建协程
		else
			-- 否则关闭连接
			skynet.error(string.format("disconnect fd = %d, error = %s", id, slave))
			socket.close(id)
		end
	end)
end)
```
master在启动的时候会通过监听 standalone 配置的地址来等待slave的注册信息。

### handshake函数
```
local function handshake(fd)
	local t, slave_id, slave_addr = read_package(fd)
	assert(t=='H', "Invalid handshake type " .. t)
	assert(slave_id ~= 0 , "Invalid slave id 0")
	if slave_node[slave_id] then
		error(string.format("Slave %d already register on %s", slave_id, slave_node[slave_id].addr))
	end
	report_slave(fd, slave_id, slave_addr) -- 这边会广播给其他slave
	slave_node[slave_id] = {  -- 保存slave对应的数据
		fd = fd,
		id = slave_id,
		addr = slave_addr,
	}
	return slave_id , slave_addr
end

```
在slave连接上来之后，master首先会执行handshake与slave进行握手，并广播给当前的slave信息广播给其他slave节点。之后会创建一个单独的协程来执行 monitor_slave 与slave进行协议的交互。


### monitor_slave
```
local function monitor_slave(slave_id, slave_address)
	local fd = slave_node[slave_id].fd
	skynet.error(string.format("Harbor %d (fd=%d) report %s", slave_id, fd, slave_address))
	while pcall(dispatch_slave, fd) do end
	skynet.error("slave " ..slave_id .. " is down")
	local message = pack_package("D", slave_id)
	slave_node[slave_id].fd = 0
	for k,v in pairs(slave_node) do
		socket.write(v.fd, message)
	end
	socket.close(fd)
end
```
monitor_slave 就是协程不停的调用dispatch_slave 来处理slave的消息，知道slave关闭为止。 而dispatch_slave中就是具体的协议处理了，这边就不继续展开了。

### cslave
clave 就会稍微复杂一点了。
1. 在服务启动的时候，会通过配置先获取master的地址，并打开自己的端口进行的监听。
2. 注册自身的dispatch消息处理函数，这个函数会尝试在harbor这个table中查找对应的函数并执行。同时创建一个harbor的service。
3. 开始与master进行连接，连接上之后创建一个协程用来处理与master的数据交互
4. master连接之后会发送一个等待协议，让当前slave等待之前的N个slave与自己连接。当完成之后，创建slave的协程就退出。后面依赖第3步创建的协议与新的slave进行连接。

```
skynet.start(function()
	-- 在服务启动的时候，会通过配置先获取master的地址，并打开自己的端口进行的监听。
	local master_addr = skynet.getenv "master"
	local harbor_id = tonumber(skynet.getenv "harbor")
	local slave_address = assert(skynet.getenv "address")
	local slave_fd = socket.listen(slave_address)
	skynet.error("slave connect to master " .. tostring(master_addr))
	local master_fd = assert(socket.open(master_addr), "Can't connect to master")

	-- 注册自身的dispatch消息处理函数，这个函数会尝试在harbor这个table中查找对应的函数并执行。同时创建一个harbor的service。
	skynet.dispatch("lua", function (_,_,command,...)
		local f = assert(harbor[command])
		f(master_fd, ...)
	end)
	skynet.dispatch("text", monitor_harbor(master_fd))

	harbor_service = assert(skynet.launch("harbor", harbor_id, skynet.self()))

	-- 开始与master进行连接，连接上之后创建一个协程用来处理与master的数据交互
	local hs_message = pack_package("H", harbor_id, slave_address)
	socket.write(master_fd, hs_message)
	local t, n = read_package(master_fd)
	assert(t == "W" and type(n) == "number", "slave shakehand failed")
	skynet.error(string.format("Waiting for %d harbors", n))
	skynet.fork(monitor_master, master_fd)

	-- master连接之后会发送一个等待协议，让当前slave等待之前的N个slave与自己连接。
	if n > 0 then
		local co = coroutine.running()
		socket.start(slave_fd, function(fd, addr)
			skynet.error(string.format("New connection (fd = %d, %s)",fd, addr))
			socketdriver.nodelay(fd)
			if pcall(accept_slave,fd) then
				local s = 0
				for k,v in pairs(slaves) do
					s = s + 1
				end
				if s >= n then
					skynet.wakeup(co) -- 全部完成了之后唤醒创建slave的协程
				end
			end
		end)
		skynet.wait() --等待其他slave的连接
	end
	
	-- 当完成之后，创建slave的协程就退出。后面依赖第3步创建的协议与新的slave进行连接。
	socket.close(slave_fd)
	skynet.error("Shakehand ready")
	skynet.fork(ready)
end)
```

### harbor服务

这个服务主要就是用来转发数据到slave中
```
local skynet = require "skynet"

local harbor = {}

function harbor.globalname(name, handle)
	handle = handle or skynet.self()
	skynet.send(".cslave", "lua", "REGISTER", name, handle)
end

function harbor.queryname(name)
	return skynet.call(".cslave", "lua", "QUERYNAME", name)
end

function harbor.link(id)
	skynet.call(".cslave", "lua", "LINK", id)
end

function harbor.connect(id)
	skynet.call(".cslave", "lua", "CONNECT", id)
end

function harbor.linkmaster()
	skynet.call(".cslave", "lua", "LINKMASTER")
end

return harbor

```

可以看到就是包装了一层到slave的调用而已。


## 小结

这边稍微探究了一下skynet bootstrap的启动过程中具体是做了什么，以及对应master/slave模式具体是干了什么事情。不过在代码里面可以看到很多skynet.xxx的调用。接下来就要具体看看skynet.lua中具体是如何处理消息的了。