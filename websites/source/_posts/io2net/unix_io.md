---
title: Unix网络编程之5种IO模型
category: 网络IO编程
date: 2020-04-02 20:47:28
tags: 网络IO编程
---

<!-- more -->


##### 1. Unix/Linux操作系统简述
> Unix操作与Linux系统结构图解 (引用计算机操作系统书籍)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020022311314023.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200223113215694.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
> 概要说明
- 用户空间: 姑且将上述用户级别或者是unix编程的应用程序的部分称为用户空间,我们可以通过启动进程来调用内核来完成从硬件设备读取或写入等操作
- 系统内核: 是直接与计算机硬件打交道的应用程序级别,在计算机相关的书籍中也称为操作系统,可以通过操作系统级别提供的一些组件来帮助用户进程与计算机硬件完成通信交互的操作,可以称为一个中转站

> 文件描述符（File descriptor）

在linux/unix系统中,文件进程存储着一份文件描述表(数组存储),数组存储file结构指针类型,文件进程通过数组索引来完成文件的操作,这里的数组索引也就是文件描述符,简称为fd

至此,有了上述的认知之后,我们再来看下Unix的IO模型是如何进行网络数据的传输

##### 2. Unix之IO模型
首先,讲述IO模型之前,需要先明确一个概念,比如我们现在需要从网络读取数据,那么流程将是先从用户进程中向系统内核发起读取数据的操作,然后等待内核接收到网络传输过来的数据报,有了数据报之后,根据上述操作系统的结构可知,我们需要将数据报通过系统内核复制到用户空间上,这个时候用户进程才能够获取到数据进行数据报的读取,由此可知,一个输入操作需要两个阶段:

- 等待数据报可达,也就是系统内核必须要有接收到数据报
- 有了数据报之后,需要将数据报从内核复制到用户空间

另外
- unix用recvfrom函数表示应用进程向系统内核发起读取操作的系统调用

至此,网络IO读取操作有了上述的认知,接下来再看IO模式,注意默认情况下所有的套接字,即socket都是阻塞式的,以网络读取数据为例展开,为了简化概念,将TCP的低套接字等信息隐藏不做说明,只说明IO模型,或者可以简单理解为UDP传输方式

> 阻塞式IO模型
- 自应用进程发起recvfrom系统调用,在此期间一直处于被阻塞,因为这个时候需要等待内核获取数据报信息并将数据报复制到用户空间中,除非被中断异常返回,否则将一直处于阻塞状态
- 以下是时序图展示阻塞式IO
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200223115104499.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
> 非阻塞式IO模型
- 非阻塞式主要体现在用户进程发起recvfrom系统调用的时候,这个时候系统内核还没有接收到数据报,直接返回错误给用户进程,告诉“当前还没有数据报可达,晚点再来”
- 用户进程接收到信息,但是用户进程不知道什么时候数据报可达,于是就开始不断轮询(polling)向系统内核发起recvfrom的系统调用“询问数据来了没”,如果没有则继续返回错误
- 用户进程轮询发起recvfrom系统调用直至数据报可达,这个时候需要等待系统内核复制数据报到用户进程的缓冲区,复制完成之后将返回成功提示
- 以下时序图展示NIO的方式
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200223115840214.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
> IO复用模型
- IO复用模式是使用select或者poll函数向系统内核发起调用,阻塞在这两个系统函数调用,而不是真正阻塞于实际的IO操作(recvfrom调用才是实际阻塞IO操作的系统调用)
- 阻塞于select函数的调用,等待数据报套接字变为可读状态
- 当select套接字返回可读状态的时候,就可以发起recvfrom调用把数据报复制到用户空间的缓冲区
- IO复用模式时序图如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020022312032477.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
> 信号驱动式IO模型
- 用户进程可以使用信号方式,当系统内核描述符就绪时将会发送SIGNO给到用户空间,这个时候再发起recvfrom的系统调用等待返回成功提示,流程如下
- 先开启套接字的信号IO驱动功能,并通过一个内置安装信号处理函数的signaction系统调用,当发起调用之后将会直接返回
- 其次,等待内核从网络中接收数据报之后,向用户空间发送当前数据可达的信号给到信号处理函数
- 信号处理函数接收到信息就发起recvfrom系统调用等待内核复制数据报到用户空间的缓冲区
- 接收到复制完成的返回成功提示之后,应用进程就可以开始从网络中读取数据
- 时序图如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020022312355190.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
> 异步IO模型
- 由POSIX规范定义,告知系统内核启动某个操作,并让内核在整个操作包括数据等待以及数据复制过程的完成之后通知用户进程数据已经准备完成,可以进行读取数据
- 与上述的信号IO模型区分在于异步是通知我们何时IO操作完成,而信号IO是通知我们何时可以启动一个IO操作
- 时序图如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200223131143774.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 现代计算机服务器操作系统大部分都是基于linxu实现,为处理高并发而采取NIO的模型,对于支持异步IO模型的系统持有不确定因素

##### 3. 同步与异步和阻塞与非阻塞
> 同步与异步的定义

- 同步:发起一个fn的调用,需要等待调用结果返回,该调用结果要么是期望的结果要么是异常抛出的结果,可以说是原子性操作(要么成功要么失败返回)
- 异步: 发起一个fn调用,无需等待结果就直接返回,只有当被调用者执行处理程序之后通过“唤醒”手段通知调用方获取结果(唤醒的方式有回调,事件通知等)
- **小结: 同步和异步关注的是程序之间的通信**

> 阻塞与非阻塞的定义

- 阻塞: 类比线程阻塞来说明,在并发多线程争抢资源的竞态条件下,如果有一个线程已持有锁,那么当前线程将无法获取锁而被挂起,处于等待状态
- 非阻塞: 一旦线程释放锁,其他线程将会进入就绪状态,具备争抢锁的资格
- **小结: 阻塞与非阻塞更关注是程序等待结果的状态**
- 由此可知,同步异步与阻塞非阻塞之间不存在关联,关注的目标是不一样的

> 同步IO与异步IO(基于POSIX规范)
- 同步IO: 表示应用进程发起真实的IO操作请求(recvfrom)导致进程一直处于等待状态,这时候进程被阻塞,直到IO操作完成返回成功提示
- 异步IO: 表示应用进程发起真实的IO操作请求(recvfrom)导致进程将直接返回一个错误信息,**“相当于告诉进程还没有处理好,好了会通知你”**
- 阻塞IO: 主要是体现发起IO操作请求通知内核并且内核接收到信号之后如果让进程等待,那么就是阻塞
- 非阻塞IO: 发起IO操作请求的时候不论结果直接告诉进程“**不用等待,晚点再来**”,那就是非阻塞

> IO模型对比

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200223152931555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 根基上述的同步与异步IO定义并结合上述的模型可知,只有异步IO模型符合POSIX规范的异步IO,其他IO模型都存在recvfrom系统调用被内核阻塞,属于同步IO操作
- 由此可知,阻塞IO与非阻塞IO可总结如下:

- 也就是说,要么称为同步与异步IO,要么称为上述5种模型的IO说法,注意上述的同步与异步的概念
- 大部分操作系统都是基于同步IO的方式实现,对于支持异步IO模型的操作系统还不确定,在实际工作我们经常会说Blocking-IO(阻塞IO)和Non-Blocking-IO(非阻塞IO),极少称同步IO与异步IO
- **小结: 同步与异步针对通信机制,阻塞与非阻塞针对程序调用等待结果的状态**
