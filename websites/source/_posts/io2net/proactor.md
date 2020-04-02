---
title: proactor模式
category: 网络IO编程
date: 2020-04-02 20:47:28
tags: 网络IO编程
---

<!-- more -->
```text
## 参考链接
https://www.alexeyshmalko.com/2014/proactor-pattern-release-the-power-of-asynchronous-operations/
https://www.boost.org/doc/libs/1_72_0/doc/html/boost_asio/overview/core/async.html
https://medium.com/@beeleeong/the-proactor-pattern-bbfe7c33a43c
```

###### Proactor 模式
> 定义

Proactor是一种用于事件处理的软件设计模式，其中长时间运行的活动在异步部分中运行。在异步部分终止后调用完成处理程序。proactor模式可以被认为是同步反应器模式的异步变体

> 三个特殊操作

- Proactive通过异步操作处理器发起异步IO操作,并定义完成处理程序的处理器
- 完成处理程序的处理器需要等待异步处理器结束异步的IO操作之后发起调用执行
- 整个过程是异步操作

> 两个标准流程

- 在整个过程中通过异步操作处理器来控制整个过程的异步操作
- 调度处理器执行需要根据执行的环境进行调度执行

> 流程示意图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200322155750624.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
