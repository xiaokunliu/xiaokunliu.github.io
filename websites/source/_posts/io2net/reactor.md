---
title: 高性能IO编程设计
category: 网络IO
date: 2020-03-14 17:57:27
tags: 网络IO编程
---

<!-- more -->

```text
参考：
https://dzone.com/articles/understanding-reactor-pattern-thread-based-and-eve
https://en.wikipedia.org/wiki/Reactor_pattern#Structure
https://en.wikipedia.org/wiki/Event_loop
https://en.wikipedia.org/wiki/Proactor_pattern
https://en.wikipedia.org/wiki/Event-driven_architecture
```
![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/pattern_title.jpg)

首先,在讲述高性能IO编程设计的时候,我们先思考一下何为“高性能”呢,如果自己来设计一个web体系服务,选择BIO还是NIO的编程方式呢?其次,我们可以了解下构建一个web体系服务中,为了能够支撑更多的并发连接数,一般会有两种web架构设计方案,即线程架构以及事件驱动设计,在Java的IO设计演进文章已对线程架构设计方案进行详细的阐述,本文主要以事件驱动设计具体实现技术展开讨论.
#### web体系设计

##### 何为高性能
在epoll技术原理分析文章中讲到C10K的优化问题,需要从文件描述符限制,线程资源,内存资源,网络数据包大小传输等方面进行优化,目的是提升web服务的连接调度处理能力,支撑更多的客户端并发连接响应,因此高性能的IO设计意味着(实现IO高性能的目标)需要考虑以下几个方面,即:

- 可以实现并发连接的响应调度,那么web服务可能需要借助多线程技术
- 在上述基础上可以支撑更多的连接处理,那么web服务能够实现可伸缩
- 充分利用计算机资源并减少资源的空闲浪费,那么web服务需要尽可能地合理利用CPU/带宽/内存等资源

##### BIO与NIO性能区分
为了达到web服务的高性能设计目标,我们需要考虑技术落地方案的选择,现有方案有基于'one thread one connection'的BIO以及'one thread one server'的NIO技术实现方式,其次,在这里需要声明一点就是BIO视为单线程的同步操作,NIO视为单线程的异步操作,同时我们也需要关注两种不同IO的实现在性能测试中的结果是如何的,才能有效地帮助我们实现高性能的目标,以下是摘录《 Thousands of Threads and Blocking I/O》的性能测试结果数据,现分析如下:

> 异步web与同步web的吞吐量

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/io_straight.jpg)

通过上述可知,在相同的操作系统环境下,同步web的IO吞吐量更高,主要包含以下方面:
同步Web的IO模型吞吐量性能要比NIO高出25%-35%,即使使用多个selector的NIO实现方式也无法比基于Linux的NPLT实现同步操作的性能更快

其次,linux内核使用epoll的技术主要是解决poll本身性能以及可伸缩性问题,epoll在技术实现也将通过创建少量线程的方式来提升性能,增加吞吐量的处理能力

> 编程方式

- NIO编程伪代码

```java
while(true){
    // 调用select()
    int rs = select();
    
    //  如果rs没有对应的就绪事件个数，继续select()
    if (rs <= 0){
        continue;
    }
    
    // 获取可用的key
    Set<Keys> keySet = selectKeys();
    for(Key key: keySet){
        if(key.isAcceptable()){
            client = accept();
            // register and save the key
            client.register(...);
        }else if(key.isReadable()){
            // read();
            // decode();
            // process();
            // encode();
            // write();
        }
    }
}
```

- BIO编程伪代码

```java
while(true){
    client = accept();
    client.read();
    // decode();
    // process();
    // encode();
    // write();
}
```

通过上述可以看出,BIO是面向单连接处理的编程方式,调用accept以及read方法都需要进行等待就绪状态才能进行下一步操作,而NIO则是面向单线程处理多连接的编程方式(严格意义上是基于事件编程),通过轮询以及就绪事件的遍历来处理就绪事件,相比BIO在实现上会更为复杂些,然而对于实现高性能的IO设计,我们还需要借助多线程技术来实现,下面针对多线程的同步与异步方式进行对比与分析

> 多线程环境下同步与异步性能对比

linux在内核2.6版本之后使用NPTL的规范实现线程技术，空闲的线程成本接近为0，同时线程上下文能够实现更快切换以及尽可能地运行更多线程，如下图所示:

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/thread_context_switch.jpg)

通过上述可知,多线程环境下使用同一个类库进行测试的性能,1000个与1个线程执行的性能效率上相差不大,因此线程上下文切换的成本其实不高
然而对于多线程环境的同步操作如下图:

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/thread_context_switch2.jpg)

通过上述可知,syncHashMap与HashTable随着增加的线程数,其执行的性能耗时更高,因为同步操作的hashtable和syncHashMap是在线程级别加锁实现顺序的写操作,因此需要等待其他线程执行完成才能被唤醒执行,对于具备“异步”特性的类库则是通过多线程并发方式对容器实现写操作,即同一个时刻可以有多个线程对容器实现写操作.
多核环境下的同步与异步性能对比

> 单核环境

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/core1.jpg)

> 多核环境

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/core2.jpg)

通过上述可知,具备‘异步‘的并发类库不论是在单核还是多核环境下性能基本差不多,但是对于实现同步hashtable的性能在多核环境充分利用cpu核数提升性能,但是在上述我们注意到SyncHashMap执行的性能会更差,为什么?个人理解上述的map类库都是放在相同环境并发执行,而并发环境必然存在资源的竞争,因此对于在激烈的并发竞争环境中,同步操作的成本会更高.

> BIO与NIO分析小结

- BIO在吞吐量性能上比NIO的方式更好
- BIO编程相比NIO更为简单
- 对于同步与异步操作,无竞争的同步操作性能更好,而存在竞争的同步操作会降低执行的性能,此时进行同步操作成本更高
- 线程上下文切换的成本其实并不是特别高,但是在多线程的同步环境下性能损耗的成本更高
- 另外可以看到并发类库具备更好伸缩性,比如concurrenthashmap与hashtable执行内存IO的写操作,后者需要通过加锁实现线程同步,而前者同样是加锁,但却是分片加锁,使得线程可异步化执行,即同一个对象可以让不同线程进行写操作,这个时候性能上的提升并不依赖于线程资源.
- 同步操作能够充分利用多核cpu资源来提升性能

简而言之,高性能IO设计可以运用分散的思想并借助并发多线程技术以及充分利用计算机资源技术手段来达到目标,同时为了保证web服务可伸缩性,可以考虑引入中间层的思想来解决现有无法扩展的问题,接下来,我们开始进入web服务设计,为了能够支撑更多的并发连接数,一般会有两种web体系架构设计模式,一种是基于线程的架构,另一种是基于事件驱动架构设计.现针对上述两种架构展开分析.

##### 基于线程连接架构(TBA)
线程连接架构是基于"每个连接对应每个线程"的设计思想,这样设计主要有以下几个方面考虑：

- 它适用于那些为了与非线程安全的库兼容而需要避免线程化的站点,比如每个线程连接可以使用hashmap来处理当前线程的业务数据等操作,避免产生线程安全问题
- 使用多模块处理机制隔离每个请求,保证每个请求request之间是相互独立不干扰的

> 线程与连接1:1模式

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/1_1_thread.jpg)
![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/1_1_thread2.jpg)
![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/1_1_thread3.jpg)

- 上述每一个连接请求都需要创建相应的线程资源来处理对应的每个连接任务
- 如果需要支撑的连接成千上万,将会导致创建的线程资源个数达到瓶颈,无法满足每连接每线程的目标
- 创建与销毁线程产生的开销也将会影响性能,执行期间有可能会导致其他线程处于idle状态,浪费资源空间

> 线程与连接N:M模式

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/m_n_thread.jpg)
![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/m_n_thread2.jpg)
![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/m_n_thread3.jpg)

- 对于线程池技术,如果创建的线程无法来得及处理连接请求那么此时将会把还未处理的连接添加到阻塞队列中,如果是有界队列,那么超出的连接怎么处理,如果是无界队列,那么连接堆积,内存资源以及cpu资源都会成为瓶颈,这些都无法满足我们对于一个高性能web服务的要求,即阻塞队列要应该设置多大才合适?
- 如果线程池所有的线程处理的连接都保持"keep alive"却没有任何其他业务操作,这个时候也会造成线程空闲,也会导致阻塞队列上的连接一直没有被执行而处于等待状态,出现"假死"状态,即线程池调整的线程数量应当设置多大才能保证被充分利用?

基于上述线程架构的问题,引入事件驱动设计方案,接下来我们来看下事件驱动设计是什么,如何解决上述的问题.

##### 事件驱动架构(EDA)
在讲述事件驱动设计之前,可以先通过一个简单的示例展开.当我们在前端页面触发点击事件的时候,就会调用对应的一个触发函数来响应对应的点击事件,也就是说开发人员需要通过以下方式来完成一个点击事件的注册与绑定操作:

```js
// 获取button组件
var btn = document.getElementById("login");
// 绑定点击事件
login.click(function(){
   // login process
});
```

对此,我们先梳理下什么是事件,事件系统有哪些组成,最后再根据上述的点击事件将整个事件处理流程以时序图的方式展开.

> 事件定义与结构组成

- 什么是事件:在网络编程中,一个事件可以被定义为网络socket有新的连接,有数据可读,有数据可写等状态的变更,即socket从等待到就绪状态的变化过程,一个事件结构包含事件header以及事件body
- 事件header: 主要包含事件信息,比如事件名称,发生事件的时间戳以及事件类型
- 事件body: 提供检测到状态变更的详细信息
- 事件通知: 事件的变更可以让体系内的其他应用程序知道事件的状态变化,事件通知在事件产生,发布事件,传输事件,检测事件以及事件处理等流程进行传递,事件通知一般是以异步消息方式进行传递

> 事件流层

- 事件发射器(event emitters): 负责检测，收集以及传输事件
- 事件消费者/接收者(event consumer): 负责对产生的事件作出响应(对产生的事件进行处理并响应)或者反应(只负责对产生的事件进行过滤或者验证并传递事件到下一个活动接收者进行处理并响应)
- 事件通道(event channel): 负责事件传输的组件(事件发送器传输到接收者的管道),可以是TCP/IP连接通道,也可以是通过消息中间件进行传输,还可以是邮件或者是输入文件等形式

至此,我们对事件的定义有了基本认知之后,那么对于上述的一个完整的点击事件流程是如何进行运作的呢,现如下图所示:

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/login_event.jpg)

对于EDA的NIO而言,相比上述事件设计是运用相同的思路,但是具体实现的技术方案略有不同,EDA的NIO技术实现是基于Reactor模式,现展开NIO编程的Reactor模式进行分析.

#### 高性能之Reactor设计
##### Reactor模式
在一个通用的web服务中,一般具备以下的几方面的特征:
- web服务实现可扩展,需要借助分散设计的思想来实现
- 大部分web服务具备的通用逻辑有: 读取请求,对请求数据进行拆包,处理请求业务逻辑,结果返回的数据进行粘包,最后将数据发送到客户端.
- 对于web服务而言,不同协议在处理拆包-业务处理-粘包过程的实现方式以及成本都会有所不同

一般地,对于经典的TBA架构的web服务如下图:

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/classic.jpg)


在上述图中看到每个线程处理每个handler,且不讨论先前TBA存在的问题,就可扩展性而言就存在局限性,尤其是针对部分线程执行decode-compute-encode过程中出现耗时缓慢情况时,很难对其进行优化操作,甚至无法通过服务进行配置调优,没有达到高性能的可伸缩性要求.

##### 可伸缩web服务目标

- 一旦负载过多的时候,能够实现对客户端的降级操作
- 可以通过增加资源来改进或者完善现有的web服务性能,比如cpu/内存/网络带宽/磁盘IO读写能力等
- 还要满足低延迟,支撑高峰要求以及服务可用性
- 可伸缩实现的手段一般采用分而治之的设计思想来解决

##### IO事件驱动架构
对于一个高性能的IO事件驱动设计,主要包含有以下三个内容:

- 基于上述的事件驱动架构(EDA)原理
- 借助NIO中非阻塞的API
- 分而治之的设计思想实现web可伸缩性

对于IO事件驱动架构实现的技术主要是使用Reactor模式,现开始进入Reactor模式的分析

> Reactor定义

反应器设计模式是一个事件处理模式，用于处理一个或多个输入并发地传递给服务处理程序的服务请求。然后，服务处理程序对传入请求进行多路复用，并将它们同步分派给相关的请求处理程序.

- 反应器模式是事件驱动架构的一种实现技术.简而言之,它使用单线程事件循环对资源发出的事件进行阻塞,并将其分配给相应的处理程序和回调.
- 只要注册了事件的处理程序和回调来处理它们,就不需要阻塞IO.事件是指实例,例如新的传入连接,可以读取,可以写入等操作.这些处理程序或者回调函数可以在多核环境中利用线程池方式实现
- 这种模式将模块化应用程序级代码与可重复使用的反应堆实现解耦

> Reactor组成结构

- 请求资源:可以为系统提供输入的资源,可以是读取外部文件,接收的网络数据报,其他或当前系统输出资源都可以作为系统输入的资源,在网络编程中请求资源为发起网络请求的socket
- 同步事件多路复用器:所有的请求资源都阻塞于事件轮询,通过事件轮询检测请求资源是否处于就绪状态,一旦处于就绪状态,多路复用器就会启动资源同步操作,将就绪资源发送到调度程序中处理请求
- 请求转发器:负责接收多路复用器的就绪资源,并根据请求的资源进行注册或注销对应的请求处理器,交由对应的处理器负责处理请求
- 请求处理器:在应用程序中定义对应请求资源的请求处理器来完成相应的业务请求并给予请求响应

Reactor设计示意图如下

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/reactor_design.jpg)

通过上述示意图可知,Reactor模式在应用程序级别代码交由handler进行处理,而对于整个网络的复用操作交由多路复用器进行处理,实现反应堆的复用与应用程序业务逻辑的解耦,同时可以针对handler处理器进行调优处理以达到handler能够更快速地响应真正的IO事件并返回给客户端程序响应结果.

##### Reactor核心原理

> Reactor的事件轮询

通过上述可知,在事件轮询中包含以下三个步骤:

- 查找所有处于活动状态且未锁定的处理程序,或将其委托给dispatcher实现
- 依次执行这些处理程序直到完成或者到达它们被阻塞的点.完成的处理程序将会被停用并允许事件循环继续.
- 重复执行第一个步骤

> 两个核心参与者

- Reactor反应器:也可称为多路复用器,即在单独的线程中运行,它是通过将工作分派给适当的处理程序来响应IO事件.
- Handler处理器:处理程序执行与I/O事件有关的实际工作,反应堆通过分派适当的处理程序来响应I/O事件,即处理程序执行非阻塞操作.

> Reactor处理流程

- Java的Reactor反应器通过调用select()不断监听socket事件的变化,通过NIO的SelectionKey保存当前socket事件变化状态.
- 当创建服务端socket的时候会将服务端socket进行注册与端口绑定操作,实现端口的监听事件
- 当客户端与服务端建立连接的时候,服务端socket端口监听到事件变化,此时将客户端的socket注册并保存到SelectionKey中,即Acceptor操作
- 当客户端发起请求操作时,服务端保存的客户端socket监听到可读事件,将会在Reactor中添加对应的事件响应处理器Handler并由内部的转发器分发到对应的Handler进行处理
- Reactor相当于事件的发起器,SelectionKey相当于事件通道,用于保存和投递消息通知,Handler相当于事件消费者,也有称为事件处理引擎.

下游事件反应器为可选,主要用于处理返回的结果呈现,可以理解为前端结果展示的组件.

##### Reactor技术演进

在文章开头部分讲述到实现高性能的目标,通过对比NIO与BIO的编程设计分析,我们基本上都会基于NIO模式来设计一个高性能的web服务,而一般地,对于NIO服务设计具备高性能的目标,需要借助以下的技术手辅助段来达到目标.

> 实现高性能手段

- 线程池技术:需要关注线程池核数,线程池最大线程数,超时时间,阻塞队列存储的策略,连接负载过多处理策略
- NIO提供非阻塞技术:即保证accept以及read操作为非阻塞
- NIO提供的内存优化技术:以字节byte为单位使用byteBuffer缓存或发送数据
- 可以使用并发库技术:在上述中对比异步与同步的性能分析,可以使用并发库来实现多线程环境下的异步操作

> 一个单线程NIO服务通用设计

- 处理select的轮询调用
- 读取request数据
- 写出response数据
- 后台业务核心数据处理逻辑,即DB数据的读写/网络数据的读写/磁盘数据的读写/内存数据的读写

在上述过程,业务核心数据逻辑具备多样性,需要针对不同的场景来进行分析,因此影响性能的处理步骤往往在于最后一步,由此可通过Reactor与多线程技术来进一步提升web服务的处理性能

> Reactor技术演进

接下来以图解的方式来查看Reactor与多线程技术的演进过程.以下图解均来自《Scale IO in Java》以及github上的gnet库来演示.
单Reactor + 单线程模式

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/single_reactor.jpg)

单Reactor + 多线程模式

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/single_reactor_workers.jpg)

多Reactor + 多线程模式

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/multiple_reactors.jpg)

gnet库实现的一种Reactors模式

![](https://github.com/xiaokunliu/xiaokunliu.github.io/blob/feature/writing/websites/zimages/reactor/multiple_reactors2.jpg)


最后,我们单从Reactor技术变化来看,其设计的目的无非包含以下几个方面:
- 反应器体系结构模式允许事件驱动的应用程序对来自一个或多个客户机的服务请求进行多路复用和分派,即支持更多的客户端连接请求调度.
- 反应器模式是一种用于同步解复用和事件到达时的顺序的设计模式,通过轮询不断寻找就绪事件,并在事件触发时通知相应的事件处理程序来处理它,引入新的对象组件Reactor与Handler,实现程序业务逻辑与socket的IO复用事件处理逻辑解耦.
- 它接收来自多个并发客户机的消息、请求和连接，并使用事件处理程序顺序处理这些帖子.反应器设计模式的目的是避免为每个消息、请求和连接创建线程的常见问题
- 它从一组处理程序接收事件，并将它们按顺序分发到相应的事件处理程序,同时可以看到采用分而治之的设计思路来实现可web服务的伸缩性.
