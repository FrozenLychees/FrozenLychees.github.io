---
title: Lua 源码阅读笔记-chunk 加载相关
description: 
slug: 
date: 2023-11-21 00:00:00+0000
image: cover.jpg
categories:
    - Lua
---

# Lua 源码阅读笔记-加载chunk
lua版本基于5.4.6， 文章更多是记录自己在阅读的是思绪，而非科普。
如果内容有问题或者不正确的地方，欢迎留言讨论。

## 大致流程
在Python中，当python虚拟机执行py脚本的时候，会先将py脚本编译成pyc（实际上就是字节码的集合），然后虚拟机加载对应的字节码开始工作。这个流程在lua中也是一样的，lua解释器会将lua源码编译成所谓的chunk，然后加载这个chunk进行解释工作，只是一般情况下，chunk文件没有保存在本地。lua提供luac的方式手动将源码编译成二进制的chunk文件并保存。

## Chunk
Chunk貌似就是类似pyc一样的文件，包含了运行的字节码、版本信息等

## load chunk
从加载chunk开始 稍微往下跟一下，加载chunk的函数是lua_load
```
LUA_API int lua_load (lua_State *L, lua_Reader reader, void *data,
                      const char *chunkname, const char *mode) {
  ...
  if (!chunkname) chunkname = "?";
  luaZ_init(L, &z, reader, data);
  status = luaD_protectedparser(L, &z, chunkname, mode);
  if (status == LUA_OK) {  /* no errors? */
    ...
  }
  lua_unlock(L);
  return status;
}

int luaD_protectedparser (lua_State *L, ZIO *z, const char *name,
                                        const char *mode) {
  struct SParser p;
  int status;
  ...
  luaZ_initbuffer(L, &p.buff);
  status = luaD_pcall(L, f_parser, &p, savestack(L, L->top.p), L->errfunc);
  ...
  return status;
}
```
可以看到缩减之后，函数本质是调用了luaD_protectedparser这个函数，而luaD_protectedparser中通过luaD_pcall进行了一系列的操作。

luaD_pcall这个函数在源码中的定义如下
```
** Call the C function 'func' in protected mode, restoring basic
** thread information ('allowhook', etc.) and in particular
** its stack level in case of errors.

int luaD_pcall (lua_State *L, Pfunc func, void *u, ptrdiff_t old_top, ptrdiff_t ef)
```
看起来就是通过 lua 虚拟机的操作来 “安全的” 调用 一个Pfunc，保证出现异常的时候不会直接挂掉. 这边先不深入这个函数，后面应该还会遇到它很多次

回到luaD_protectedparser，可以看到调用的函数是f_parser。

## f_parser

```
static void f_parser (lua_State *L, void *ud) {
  LClosure *cl;
  struct SParser *p = cast(struct SParser *, ud);
  int c = zgetc(p->z);  /* read first character */
  if (c == LUA_SIGNATURE[0]) {
    checkmode(L, p->mode, "binary");
    cl = luaU_undump(L, p->z, p->name);
  }
  else {
    checkmode(L, p->mode, "text");
    cl = luaY_parser(L, p->z, &p->buff, &p->dyd, p->name, c);
  }
  lua_assert(cl->nupvalues == cl->p->sizeupvalues);
  luaF_initupvals(L, cl);
}
```

这个函数很短，这个cast宏可以看作是一个构造函数，构造出Sparse这个对象，实际上这边传入的是luaD_protectedparser 函数的那个栈变量p；

通过这个p的checkmode函数来检查加载的文件和加载的方式是否匹配，并调用不同的函数构造出LClosure，设置到对应的虚拟机中，这边构造出的LClosure应该是一个最外层的Lua闭包。

具体的parse部分目前先不深究了，对于语法、词法分析相关的基础知识还差太多了，等以后有空再补。

闭包这个概念在Lua中应该是需要花一整个篇幅来阅读的，这边写不展开看了。

## 小结
从load chunk开始一层层往下，基本上可以确定Lua从给定的名字和打开方式中解析出一个闭包，然后设置到对应的虚拟机对象中，这样后面Lua虚拟机就可以根据这闭包执行对应的字节码了。

## 引用

https://yuerer.com/Lua5.3-%E8%AE%BE%E8%AE%A1%E5%AE%9E%E7%8E%B0(%E4%B8%80)-Lua%E6%98%AF%E6%80%8E%E4%B9%88%E8%B7%91%E8%B5%B7%E6%9D%A5%E7%9A%84/
https://yuerer.com/Lua5.3-%E8%AE%BE%E8%AE%A1%E5%AE%9E%E7%8E%B0(%E5%9B%9B)-Closure%E4%B8%8EUpvalues/
https://zhuanlan.zhihu.com/p/358423900

1. 《Lua设计与实现》
2. https://blog.codingnow.com/2011/03/lua_gc_1.html 《云风-Lua GC 的源码剖析》
