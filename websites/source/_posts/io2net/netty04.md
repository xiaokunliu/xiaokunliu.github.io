---
title: 深入分析Netty核心特性
category: 网络IO编程
date: 2020-05-01 17:47:36
tags: 网络IO编程
---

<!-- more -->

### 深入分析Netty高性能特性

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_title.jpg)

在讲述Netty的高性能特性之前,基于之前的epoll技术分析中谈到C10K问题与高性能IO设计文章的认知,对于C10K问题其实是属于一个优化问题,目的是为了能够在单台机器上支撑更多的并发连接调度所做的性能优化,为了达到上述目标,需要要求我们设计的web服务采用合理的IO模型,并在对应的IO模型基础上引入多线程与并发库技术的使用来支撑更多的连接调度,同时考虑到计算机资源的限制,我们需要在设计web服务的时候合理对资源进行分配优化,比如内存,网络带宽以及CPU核数的充分利用,也就是说我们还需要考虑到可伸缩性的问题,通过增加资源来使得我们的web服务能够得到线性提升效果.接下来我们就来结合源码分析Netty技术是如何体现高性能这一个特性.



#### C10K与C10M问题

##### C10K&C10M解决方案

> C10K问题

关于C10K的问题,在先前的epoll技术分析文章已经有讲述过,C10K是属于一个优化问题,即要让单个web服务支撑1w的并发连接,关于C10K的性能与可伸缩性问题,摘录C100M的博文并加入自己的理解:

- 采用线程连接架构TBA模型,也就是1个客户端连接对应1个线程,那么对于内核而言,假设这个时候需要10k个连接,那么也就意味着要10k个线程,此时内核需要从这个10k个线程中轮询遍历哪个线程是有数据流量进来的,对于服务器本身而言,不论线程数量多少,线程上下文切换的时间是恒定的,即使再多的连接分配给再多的线程,其性能也不会上去,线程调度仍然无法扩展,除了本身线程资源的瓶颈之外,我们可以看到的一个现场就是线程调度无法扩展.
- 相对地,采用选择/轮询来处理连接事件,也就是面向事件驱动设计EDA模式,我们在分析select/poll/epoll技术中讲到,它们都是对一个socket集合fds进行监听,每个数据包都会经过socket套接字,即使套接字增加,我们同样可以通过选择和轮询的方式来遍历socket数据流量进来的事件,这个时候单线程是可以完成一个选择和轮询就绪事件的操作,同时还可以实现连接的扩展性,随着IO技术的发展,现代服务器都会引入可扩展的epoll技术与异步IO Compeletion Port在指定时间内查询就绪的socket集合并返回给应用程序.

因此,优化一个C10K的问题可以从以下几个方面考虑:

- 选用的IO模型能够支持web实现可伸缩性
- 结合IO模型设计的线程模型,能够通过增加适当的线程数量来支撑web服务更多的并发连接
- 最后一个可以理解为性能问题,一个Web服务的性能可以参考以下几个因素: **数据复制拷贝问题/线程上下文切换问题/内存分配问题以及锁争用(无锁编程是一个我们理想的选择)**

> C10M问题

同理地,C10K问题的解决,随着互联网技术发展,又提出了一个C10M的优化问题,即如何让我们的单台机器支撑1000w的并发连接,这个时候Errata Security首席执行官Robert Graham从历史的角度出发讲述Unix最开始设计不是通用的服务器OS,而是作为电话网络的控制系统,实际上是电话网络在控制数据传输,因而控制平面与数据平面存在清晰的分隔,于是指出一个问题,即当前我们使用的Unix服务器是作为数据平面的一部分,这也是他所说的内核不是解决方案,而是问题所在,什么意思呢?**不要让内核承担所有繁重的工作**.将数据包处理,内存管理和处理器调度从内核中移出,并将其放入应用程序中,可以在其中高效地完成它.让Linux处理控制平面,让应用程序处理数据平面.对此,一个C10M关注的问题有以下几个方面:

- 1000w个并发连接
- 支撑一个持续时间约为10s的100w并发连接
- 1000万个数据包/秒-期望当前的服务器每秒处理5万个数据包，这将达到更高的水平。过去服务器每秒能够处理100K次中断，每个数据包都会引起中断。
- 10微秒延迟-可伸缩服务器可能会处理规模，但延迟会增加。
- 10微秒抖动-限制最大延迟
- 10个连贯的CPU内核-软件应扩展到更大数量的内核。通常，软件只能轻松扩展到四个内核。服务器可以扩展到更多的内核，因此需要重写软件以支持更大的内核计算机

基于上述的叙述,为了构建一个能够支撑1000w/s的并发连接系统,我们需要让数据平面系统能够处理1000w/s个数据包,而对于一个控制平面系统而言,持续10s的最多也就只能处理100w个并发连接,为了实现这个目标,我们借鉴C10K问题的解决方案,C10K问题主要是从构建一个可伸缩性的IO模型的web服务来达到支撑10K并发连接的目的,同时也引入线程模型与性能优化手段来配合实现达到目的,从这里我们也可以看到可伸缩性是我们设计的目标,同时为了支撑1000w的连接,我们不能将性能优化外包给操作系统,那么我们要编写一个可伸缩性的软件来达到上述的目标就需要解决以下的问题:

- 数据包可扩展: 编写一个自定义驱动程序以绕过TCP堆栈,直接将数据包发送到应用程序.如PF_RING，Netmap，Intel DPDK
- 多核可扩展: 多核编码并不是多线程编码,而是让我们的应用程序分布在每个CPU核心上,保证我们能够随着内核的增加以线性扩展我们应用程序的处理能力.即一个是保持每个cpu核数的数据结构,一个是每个cpu保证原子性操作,一个是使用无锁技术的数据结构,一个是使用线程模型完成流水工作,最后一个是利用处理器的亲和力,即保持运行在每个cpu核数上分配的线程是固定的,即每个cpu对应着专有的线程来完成工作.
- 内存可扩展: 一个是使用连续内存分配技术,增加数据的缓存命中率,一个是分页表运用高效的缓存数据结果并对数据压缩,一个是使用池化技术管理内存,一个是合理分配线程以降低内存访问延迟,最后一个是使用预分配的内存技术.

因此,我们可以借鉴C10K与C10M的优化思路来推导一个具备高并发,高性能且可伸缩性的web服务设计思路展开,高并发连接调度我们可以从IO模型以及线程模型思考,高性能的指标我们可以从计算机资源分配管理与优化方面思考(比如内存/无锁编程),而一个可伸缩性的web服务我们会从物理资源角度来考虑,通过增加相关的资源配置是否能够得到线性的性能提升.接接下来我们开始分析Netty是如何实现高并发,高性能以及如何体现可伸缩性的.

##### 高并发问题

> 高并发关注指标

- 响应时间(Response Time):发起一个request请求,执行这个request请求从开始到最后返回响应结果所花费的总体时间,也就是客户端发起请求到最后收到服务端返回响应结果的时间.比如http请求响应时间为200ms,200ms表示RT.
- 每秒并发连接(并发用户数): 每秒可支撑的连接调度/同时承载正常使用系统功能的用户数量,并发连接/用户数更关注的是能够处理调度连接而不在于处理速度.
- QPS/TPS(每秒查询量/每秒事物处理量): 比如现在客户端发起一个下单操作(用户鉴权/订单校验/下单操作三个步骤),这个下单操作形成一个TPS,而下单里的每个步骤形成一个QPS,也就是说TPS包含3个QPS操作,因而对于TPS理解是一个完整的事物请求的操作结果,而QPS是针对一个request请求的操作结果,对此TPS是衡量软件测试结果的度量单位,而QPS是特定的查询服务器在指定的时间段内处理流量度量标准的数量,域名服务器的机器性能通常用QPS来衡量,QPS与TPS更关注处理速度.
- 吞吐量(Throughput): 取决于我们关注系统的业务指标,比如我们关注的是软件测量结果相关的处理能力(处理速度),那么这个时候的吞吐量我们需要关注的是TPS,如果是关注机器性能的流量,那么我们关注的吞吐量是QPS,如果我们对接的是接入层的服务,那么我们可能需要关注的是并发连接的调度,此时关注的吞吐量是支撑的并发连接调度数据.

> 并发连接/QPS/TPS

基于上述的高并发指标的理解,现将并发连接/QPS/TPS的区分通过以下图解的方式展开:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/connections_qps_tps.jpg)

- 并发连接: 主要体现在服务端程序高效的连接调度机制上,也就是说服务端能够在一定的时间段内能够正确地响应给每个连接的请求即可,至于何时响应以及如何响应不是并发连接关注的事情.
- QPS/TPS: 主要体现在处理速度上,要求能够正常完成对请求响应的处理,不仅是要对请求结果正确响应,同时还要求处理能力能够尽可能快速.

> IO与线程模型实现高并发连接调度

- 基于先前的高性能IO编程设计并结合上述的C10K与C10M问题,实现一个支撑高并发连接调度的web服务需要借助具备可伸缩性的NIO或者AIO技术完成,通过监听socket的数据流量出入事件来响应给应用程序,并且轮询事件通过单线程的方式也能够处理,还能实现扩展,只要操作系统的fd资源配置足够大即可.
- 其次,为了支撑更多更快的响应连接调度处理,我们可以适当地加入多线程处理方式来扩展上述单线程处理连接事件的能力.同时也会看到在IO相关设计,基于事件的编程,为了简化应用开发者编写代码的复杂度以及具备更好的扩展性,引入了基于EDA的Reactor与Proactor的模式设计.

##### C10K与C10M提升性能优化因素

结合之前的高性能IO编程文章以及C10K与C10M问题,我们可以考虑设计一个高性能的Web服务可以从以下几个方面思考:

> 数据包的存储

- socket接收数据流量的时候我们要考虑如何将数据包直接传输到应用程序,尽量避免数据的拷贝问题.
- 应用程序接收数据包的时候能不能缓存起来,同时如果加入缓存的话,有没有办法提高命中率.
- 数据存储的区域能否重复利用,即使用池化技术进行管理分配,减少向计算机申请资源的性能.

> 应用程序的处理能力

对于处理处理能力,我们可以用一个词来说明,那就是吞吐量,既然想要提升吞吐量,那么我们的目标其实也是很明确的,即“快”.

- 充分利用CPU资源,避免CPU一直处于空闲假死状态(线程阻塞/空轮询/线程过多)
- 根据高性能IO设计的一文,我们可以在竞争环境下使用并发库,底层原子操作等手段有助于提升IO的吞吐量
- 同步环境下能够使用无锁来处理任务

#### Netty线程模型

在Netty技术中主要是采用NIO实现多连接的单线程复用机制以及借助多线程异步处理方式来提升支撑并发连接调度的处理能力,在C10M问题中已经指出,为了优化C10M问题,我们应该考虑在应用程序方面去设计数据平面系统来构建一个支撑1000W并发连接的调度处理机制.

##### 可伸缩的IO模型

- NIO多路复用技术具备可伸缩性,通过C10K问题的分析,我们知道单线程能够处理更多的socket就绪事件,也就是说单线程面向事件驱动设计的复用技术实现可扩展性且能支撑更多并发连接的请求调度处理,这里与线程连接不同的是我们关注的是事件而不是线程本身,因而不会受限于线程资源以及线程的调度分配问题.
- 其次Netty框架是基于Reactor模式进行演变,但于Reactor模式不同的是Netty是多线程异步处理,更像是Proactor模式,但异步处理是在应用程序通过回调的方式完成的,而Proactor是基于AIO的方式将异步操作传输到内核并在内核中进行回调返回.

##### Netty之Reactor模式

关于Netty框架的线程模式架构设计图如下所示:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_thread_model.jpg)

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_thread_model2.jpg)

现在我们基于宏观上对Netty的线程模型有一个基本认知之后,结合先前文章对Netty组件源码以及事件流程的分析可知,在Netty中存在EventLoopGroup,通过EventLoopGroup来分配EventLoop,而每一个EventLoop既具备线程池的功能又承担着事件轮询的工作,同时每个EventLoop都分配对应的一个FastThread专有线程来负责对处理当前EventLoop的pipeline的流水工作,由于每一个启动EventLoop都绑定专有的一个线程FastThread,那么对于EventLoop处理的一系列流水工作也将会在当前的线程执行,从而保证了单线程资源无竞争高效串行化流水任务的执行,简单点就是无锁流水工作,这个在我们上述讲到的C10M优化方案中体现的一个多核扩展问题,Netty框架很好地运用这一理念来提升我们web服务支撑高并发连接的调度处理.

关于Netty处理的单线程无锁串行化的流水工作流程示意图如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_safe_pipeline.jpg)

在了解上述的无锁串行化任务执行流程之后,我们还需要关注Netty另一个问题,即在多Reactor模式中,我们看到服务端channel其实只完成一次创建,初始化以及注册,相比客户端channel,提供给客户端的EventLoopGroup由于在客户端有新连接进来的时候就会在Acceptor进行注册,而我们也分析channel的注册流程,注册的时候会在EventLoopGroup根据选举策略分配一个EventLoop来完成channel到EventLoop的绑定,对此,我们知道对于客户端channel而言,EventLoopGroup的作用是类似于我们分布式的“集群”机器服务来对外提供服务的,分担高并发的连接压力,那么对于服务端channel而言呢,提供EventLoopGroup如果指定的线程数量大于1,这个时候EventLoopGroup又起到什么作用呢?其实对于服务端的channel,我们很多时候并不仅仅是处理连接的接收,还要在处理连接之前做一些鉴权校验抑或是风控等安全措施的处理,如果这些过程会比较耗时,那么就需要在我们处理的handler上添加从Group选举一个新的EventLoop事件轮询活动来缓解我们并发连接调度处理能力,其实说到底Group还是类似于分布式系统中的“集群”来缓解并发调度的压力.

于是基于上述的分析,我们对Netty支撑高并发采用的技术手段总结如下:

- 使用NIO模型实现多连接的可伸缩性扩展,同时引入Reactor模式以及责任链设计在原有的基础上使得Netty可伸缩性更为灵活,能够支撑更多的并发连接调度.
- 其次,Netty设计通过为每个执行的事件轮询EventLoop分配独有的线程,保证了每个事件轮询器之间处理的流水工作相互独立,同时也保证了在当前EventLoop下执行的所有流水工作都是专属于专有的线程,不存在资源竞争以及锁争用的情况,基于此,在多核环境下我们可以充分利用多核技术进一步去提升我们的并发连接调度处理能力.
- 最后一个就是Netty通过EventLoopGroup的“集群”手段来分担我们web服务的并发连接调度处理能力,有效缓解对单个线程处理并发连接的压力,提升并发连接调度的处理能力.

#### Netty高性能之ByteBuf

##### 堆外内存

对于linux操作系统读取数据块一般流程是:先从硬件设备将数据块加载数据到内核缓冲区,然后由内核将内核缓冲区的数据复制到用户空间的缓冲区,最后唤醒应用程序读取用户空间的缓冲区,对于Java程序而言,其无法直接操作OS系统内存区域,必须通过JVM堆申请内存区域来存放数据块,于是需要再从OS内存中的数据缓冲区将数据块复制到JVM堆中才能够进行操作数据,于是对于JVM操作socket的数据包,数据包拷贝的路径如下图示:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_direct_bytebuf.jpg)

网卡设备接收到数据包流量事件,内核将数据块加载到内核缓冲区中,并且通过socket传输数据到用户空间的缓冲区,最后JVM要操作socket缓冲区的数据,需要将其读取到JVM堆中存储,这个时候需要再JVM堆中申请一个内存区域用于存放数据包数据,而如果直接通过堆外内存读取数据,则可以减少一次数据的拷贝以及内存资源的损耗,如下图所示:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_direct_references.jpg)

Netty的堆外内存操作通过底层操作系统Unsfe的方式获取其内存位置来直接操作内存,相比使用堆内存分配更为高性能便利,同时也减少了数据拷贝,直接通过Unsafe指向的堆外内存引用来进行操作.

##### 零拷贝机制

```java
// 假设buffer1以及buffer2都存储在堆外内存,堆内内存同理(只是在JVM中)
ByteBuf httpHeader = buffer1.silice(OFFSET_PAYLOAD, buffer1.readableBytes() - OFFSET_PAYLOAD);
ByteBuf httpBody = buffer2.silice(OFFSET_PAYLOAD, buffer2.readableBytes() - OFFSET_PAYLOAD);
// 逻辑上的复制,header与body仍然存储在原有的内存区域中,http为JVM在堆中创建的对象,指向一个逻辑结构上的ByteBuf
ByteBuf http = ChannelBuffers.wrappedBuffer(httpHeader, httpBody);
```

上述的零拷贝机制示意图如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/zero_copy.jpg)

这个时候在应用程序中可以直接通过http的ByteBuf操作合并之后的header+body的ByteBuf缓冲区,http的byteBuf是属于逻辑上的合并,实际上并没有发生数据拷贝,只是在JVM中创建一个http的ByteBuf引用指向并操作合并之后的bytebuf.

##### 动态扩容

```java
@Override
public ByteBuf writeBytes(byte[] src) {
  writeBytes(src, 0, src.length);
  return this;
}

@Override
public ByteBuf writeBytes(byte[] src, int srcIndex, int length) {
  ensureWritable(length);
  setBytes(writerIndex, src, srcIndex, length);
  writerIndex += length;
  return this;
}

// minWritableBytes = byte.length
final void ensureWritable0(int minWritableBytes) {
  final int writerIndex = writerIndex();
  final int targetCapacity = writerIndex + minWritableBytes;
  if (targetCapacity <= capacity()) {
    ensureAccessible();
    return;
  }
  if (checkBounds && targetCapacity > maxCapacity) {
    ensureAccessible();
    throw new IndexOutOfBoundsException(String.format(
      "writerIndex(%d) + minWritableBytes(%d) exceeds maxCapacity(%d): %s",
      writerIndex, minWritableBytes, maxCapacity, this));
  }

  // Normalize the target capacity to the power of 2.
  final int fastWritable = maxFastWritableBytes();
  
  // 动态扩容
  // 需要在java中配置io.netty.buffer.checkBounds=false
  // 默认为4M
  // > 4M进行扩容, 
  // < 4M的话要进行扩容为3072byte
  int newCapacity = fastWritable >= minWritableBytes ? writerIndex + fastWritable
    : alloc().calculateNewCapacity(targetCapacity, maxCapacity);

  // Adjust to the new capacity.
  capacity(newCapacity);
}
```

##### 引用计数与资源管理

在ByteBuf添加引用计数能够计算当前对象持有的资源引用活动情况,通常以活动的引用计数为1作为开始,当引用计数大于0的时候,就能够保证对象不会被释放,当引用计数减少到0的时候说明当前对象实例就会被释放,将会被JVM的GC进行回收,对于池化技术而言则是存放到内存池中以便于重复利用.因此使用池化技术的`PooledByteBufAllocator`而言,使用引用计数能够降低内存分配的开销,有助于优化内存使用和性能的提升.

- ByteBuf的实现接口ReferenceCounted

```java
// ByteBuf.java
public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf>{
  boolean isAccessible() {
        return refCnt() != 0;
    }
}

//AbstractByteBuf.java
public abstract class AbstractByteBuf extends ByteBuf {
  // 对于引用计数为0的实例将无法访问,会抛出异常IllegalReferenceCountException
  protected final void ensureAccessible() {
        if (checkAccessible && !isAccessible()) {
            throw new IllegalReferenceCountException(0);
        }
    }
}

//ReferenceCounted.java
public interface ReferenceCounted {
		// 调用retain(increament) 将会增加引用计数increament
   // 调用release(increament)将会减少引用计数increament
}

```

- ChannelHandler的资源管理

```java
// 对于入站事件,如果当前消费入站数据并且没有事件进行传播的话,那么就需要手动释放资源
public void channelRead(ChannelHandlerContext ctx, Object msg){
  // ...
  // not call fireChannelRead,事件传播在当前handler终止,这个时候需要手动清除
  ReferenceCountUtil.release(msg);
  // SimpleChannelInboundHandler能够手动清除,但是一般入站事件我个人习惯用ChannelInboundHandlerAdapter并且自己手动管理,方法单一,处理简单,可以手动管理,同理出站事件也是用Adapter
}

// 对于出站事件,如果当前需要对非法消息采取丢弃操作,则也需要手动进行处理释放资源
public void channelWrite(ChannelHandlerContext ctx, Object msg, ChannelPromise promise){
  ReferenceCountUtil.release(msg);
  promise.setSuccess(); // 丢弃消息意味着不会将数据传输到出站事件的责任链上,这个时候FutureListener无法监听到消息处理情况,需要手动通知处理结果
}
```

- Netty的资源监控类ResourceLeakDetector

```bash
## 关于监控类的级别详细查看Netty类下的ResourceLeakDetector
## 通过java配置并执行可以查看资源泄漏情况以及输出报告
java -Dio.netty.leakDetection.level=ADVANCED
```

##### 内存分配算法

- 入口程序

首先,Netty处理读写事件默认分配的内存Allocator源码如下:

```java
// 创建Channel的时候会创建默认的AdaptiveRecvByteBufAllocator,不论是客户端还是服务端channel
// DefaultChannelConfig.java
public DefaultChannelConfig(Channel channel) {
  this(channel, new AdaptiveRecvByteBufAllocator());
}
```

上述的DefaultChannelConfig类图如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/config_class.jpg)

其次,我们关注socket的读写事件,也就是NioSocketChannel的相关事件,在NioEventLoop下的run方法定位到对应的unsafe的read方法,如下:

```java
// AbstractNioByteChannel.java
void read(){
  // 根据上述的config可以定位到是默认使用池化的内存分配器,默认为池化分配且为堆外内存分配
  final ByteBufAllocator allocator = config.getAllocator();
  // RecvByteBufAllocator默认为AdaptiveRecvByteBufAllocator
  // allocHandle为AdaptiveRecvByteBufAllocator下的HandleImp,而HandleImp继承MaxMessageHandle
  final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
  
  //我们关注byteBuf的分配,调用上述的handler的allocate方法
  byteBuf = allocHandle.allocate(allocator);
}

// MaxMessageHandle.java
@Override
public ByteBuf allocate(ByteBufAllocator alloc) {
  // 使用池化的分配器创建ioBuffer
  // 根据数据包的大小计算要申请的内存区域大小,每个区域大小都存储在一个table表中进行存储,每次计算都会通过二分查找来搜索适合当前数据包存储的数据
  return alloc.ioBuffer(guess());
}
```

接着,我们现在需要关注的是池化分配器的ioBuffer方法,其源码如下:

```java
// AbstractByteBufAllocator.java
@Override
public ByteBuf ioBuffer(int initialCapacity) {
  // 根据上述传递进来的数据包进行创建
  if (PlatformDependent.hasUnsafe() || isDirectBufferPooled()) {
    // 默认执行方式
    return directBuffer(initialCapacity);
  }
  return heapBuffer(initialCapacity);
}

// PooledByteBufAllocator.java
public class PooledByteBufAllocator extends AbstractByteBufAllocator implements ByteBufAllocatorMetricProvider {
  
 	// 默认使用堆外内存策略
    public static final PooledByteBufAllocator DEFAULT =
            new PooledByteBufAllocator(PlatformDependent.directBufferPreferred());
  
  // 分配bytebuf策略
   @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        // 使用线程缓存技术,类似于jemalloc的可伸缩的内存分配策略
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        final ByteBuf buf;
        if (directArena != null) {
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            buf = PlatformDependent.hasUnsafe() ?
                    UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                    new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }
}
```

最后我们可以看到,Netty采用的ByteBuf是使用池化且堆外内存分配的方式,如果OS支持Unsafe操作则默认为Unsafe操作,接下来我们来关注分配算法的核心代码

- 内存分配算法的核心代码

```java
//  buf = directArena.allocate(cache, initialCapacity, maxCapacity);
PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
  PooledByteBuf<T> buf = newByteBuf(maxCapacity);
  allocate(cache, buf, reqCapacity);
  return buf;
}

// 可以看到上述存在两个部分,一个是创建池化的ByteBuf,一个是从内存中申请资源存储数据
```

1) 创建池化的ByteBuf源码

```java
// PoolArena.java
@Override
protected PooledByteBuf<ByteBuffer> newByteBuf(int maxCapacity) {
  if (HAS_UNSAFE) {
    // 直接创建一个ByteBuf
    return PooledUnsafeDirectByteBuf.newInstance(maxCapacity);
  } else {
    return PooledDirectByteBuf.newInstance(maxCapacity);
  }
}

// PooledUnsafeDirectByteBuf.java
static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
  // 这里面的代码不深挖
  // 核心处理流程: 先从线程缓存获取栈，从栈获取buf，如果不存在则将创建ByteBuf并存储栈中,最后更新栈数据并一并更新到到线程的cache中
  PooledUnsafeDirectByteBuf buf = RECYCLER.get();
  // 重置buf的引用计数
  buf.reuse(maxCapacity);
  return buf;
}
```

2) 分配内存算法源码

```java
// PoolArena.java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
  // 计算合适的一个区域
  // 比如现在申请一个资源为19byte,则会为其创建一个2的临近整数方,这个时候会分配一个32byte的数据
  final int normCapacity = normalizeCapacity(reqCapacity);

  // 申请的容量小于8kb
  if (isTinyOrSmall(normCapacity)) { 
    int tableIdx;
    PoolSubpage<T>[] table;
    boolean tiny = isTiny(normCapacity);
    // 容量小于512byte
    if (tiny) {
      // 从缓存中获取
      if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
        return;
      }
      tableIdx = tinyIdx(normCapacity);
      table = tinySubpagePools;
    } else {
      // 分配的容量大于512byte小于8kb
      if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
        return;
      }
      tableIdx = smallIdx(normCapacity);
      table = smallSubpagePools;
    }

    final PoolSubpage<T> head = table[tableIdx];
    synchronized (head) {
      final PoolSubpage<T> s = head.next;
      if (s != head) {
        assert s.doNotDestroy && s.elemSize == normCapacity;
        long handle = s.allocate();
        assert handle >= 0;
        s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);
        incTinySmallAllocation(tiny);
        return;
      }
    }
    synchronized (this) {
      allocateNormal(buf, reqCapacity, normCapacity);
    }

    incTinySmallAllocation(tiny);
    return;
  }

  // 容量大于等于8kb小于16M
  if (normCapacity <= chunkSize) {
    if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
      return;
    }
    synchronized (this) {
      allocateNormal(buf, reqCapacity, normCapacity);
      ++allocationsNormal;
    }
  } else {
    // > 16M,直接从操作系统中申请资源并且不做缓存和池化处理,于是不会添加到arena中
    // Huge allocations are never served via the cache so just call allocateHuge
    allocateHuge(buf, reqCapacity);
  }
}
```

3) 摘录实际分配内存源码

```java
// PoolArena.java
 private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
   if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
       q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
       q075.allocate(buf, reqCapacity, normCapacity)) {
     return;
   }

   // Add a new chunk.
   // pageSize = 8kb
   // maxOrder = 11
   // pageShifts = 18
   // chunkSize = 16M
   PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
   boolean success = c.allocate(buf, reqCapacity, normCapacity);
   assert success;
   qInit.add(c);
 }

// PoolChunk.java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
  // 创建一个subpage
  final long handle;
  if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
    handle =  allocateRun(normCapacity);
  } else {
    // < 8kb
    handle = allocateSubpage(normCapacity);
  }

  if (handle < 0) {
    return false;
  }
  // 从缓存队列获取nioBuffer
  ByteBuffer nioBuffer = cachedNioBuffers != null ? cachedNioBuffers.pollLast() : null;
  
  // 填充实际数据
  initBuf(buf, nioBuffer, handle, reqCapacity);
  return true;
}

// 关注allocateSubpage以及allocateRun方法,最终会执行allocateNode,这里只分析allocateSubpage
private long allocateSubpage(int normCapacity) {
  // 从arena查询对应的区域类型的PoolSubPage
  PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);
  int d = maxOrder; 
  synchronized (head) {
    int id = allocateNode(d);
    if (id < 0) {
      return id;
    }

    // 存储subpage的池大小为8kb
    final PoolSubpage<T>[] subpages = this.subpages;
    final int pageSize = this.pageSize;
    freeBytes -= pageSize;
    int subpageIdx = subpageIdx(id);
    PoolSubpage<T> subpage = subpages[subpageIdx];
    if (subpage == null) {
      // 创建一个容量大小为normCapacity的subpage来存储数据
      subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
      subpages[subpageIdx] = subpage;
    } else {
      subpage.init(head, normCapacity);
    }
    // 完成分配并返回subpage的bitmap
    return subpage.allocate();
  }
}
```

4) 分配的数据存储到线程缓存的源码

```java
// 对于池化技术,有一点就是一旦数据释放的时候将会将资源进行回收重复利用.于是当调用byteBuf进行数据回收的时候,会执行以下动作
// 入口代码
ReferenceCountUtil.release(msg);

// 在上述我们已经知道默认使用PooledByteBuf,于是如果msg为PooledByteBuf,当进行资源回收的时候,就会执行以下的动作

// PooledByteBuf
@Override
protected final void deallocate() {
  if (handle >= 0) {
    final long handle = this.handle;
    this.handle = -1;
    memory = null;
    // 这里进行资源回收
    chunk.arena.free(chunk, tmpNioBuf, handle, maxLength, cache);
    tmpNioBuf = null;
    chunk = null;
    recycle();
  }
}

void free(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle, int normCapacity, PoolThreadCache cache) {
  if (chunk.unpooled) {
    int size = chunk.chunkSize();
    destroyChunk(chunk);
    activeBytesHuge.add(-size);
    deallocationsHuge.increment();
  } else {
    SizeClass sizeClass = sizeClass(normCapacity);
    // 可以看到这里将数据添加到线程缓存中
    if (cache != null && cache.add(this, chunk, nioBuffer, handle, normCapacity, sizeClass)) {
      // cached so not free it.
      return;
    }

    freeChunk(chunk, handle, sizeClass, nioBuffer, false);
  }
}
```

至此,Netty从分配到回收一个池化的ByteBuf工作流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_cache_flow.jpg)

Netty内存分配逻辑结构视图:

1) 从宏观上看,线程与Arena之间的关系:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_thread_arena.jpg)

2) 从微观上看每个arena存储数据过程,在上述源码中我们看到在没有使用线程缓存的时候,会创建一个PoolChunk对象,在这个PoolChunk中对于小于8kb的数据会通过维护着一个subpage类型的数组来组成一个page,我们可以认为把存储数据的buffer存放在一个chunk的一个page,并且每个page的容量都是2幂次方且单位为byte,在chunk为了便于搜索可用的page,于是在逻辑上将page以完全二叉树的数据结构进行存储,方便进行搜索查询,每个二叉树节点存储对应一个可分配的容量,根容量为16M,深度每增加1,容量就减半.如下图所示:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_chunk.jpg)

3) 最后我们看下线程缓存存储的逻辑结构(基于可伸缩性的jemalloc算法):

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_thread_cache.jpg)

上述的Tiny MemoryRegionCache对应于TinySubPageCache,Small MemoryRegionCache对应于SmallSubPageCache,而Normal MemoryRegionCache对应于NormalCache.

最后,我们根据源码将内存分配策略如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_cache_memory.jpg)

#### Netty高效处理机制

##### 解决空轮询的源码

```java
// NioEventLoop.java
// 仅摘录部分代码
static{
  // 可配置select的循环次数,当网络数据包一直不可达的时候,通过次数控制减少当前selector不断无结果的空轮询,一旦超过次数将会重建selector,将原有的selector关闭,避免cpu飙升.
    int selectorAutoRebuildThreshold = SystemPropertyUtil.getInt("io.netty.selectorAutoRebuildThreshold", 512);
    if (selectorAutoRebuildThreshold < MIN_PREMATURE_SELECTOR_RETURNS) {
      selectorAutoRebuildThreshold = 0;
    }

    SELECTOR_AUTO_REBUILD_THRESHOLD = selectorAutoRebuildThreshold;
}
void run(){
  for(;;){
     try{
       select();
     }catch(){}
     
     selectCnt++;
    
    // 处理就绪事件
    processSelectedKeys();
    // 执行任务
    ranTasks = runAllTasks();
    
    if (ranTasks || strategy > 0) {
      if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
       logger.debug(...);
      selectCnt = 0;
    } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
      selectCnt = 0;
    }
  } 
}

// unexpectedSelectorWakeup方法
private boolean unexpectedSelectorWakeup(int selectCnt) {
       
  if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
      selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
    // 超过一定的次数之后重建selector,如何重建这里不贴代码
    rebuildSelector();
    return true;
  }
  return false;
}
```

##### 使用责任链机制实现无锁串行化任务

基于事件轮询器的源码与线程模型可知,分配给每个EventLoop的专属线程都会负责处理select之后的就绪事件集合以及所有在阻塞队列中的任务,且线程与EventLoop通过FastThreadLocal进行绑定,也就是说所有事件的处理与任务的执行都是处于一个线程中,从而保证事件处理与任务处理都是保持在同一个线程中,同时与了保持一个channelHandler实例能够共享于多个pipeline中,需要通过注解@Shareble方式来保证线程安全.于是对于Netty处理的任务还是channelHandler下的完成事件处理都是能够得到线程安全的保证,于是对于无锁串行化的描述如下图:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_unlock_pipeline.jpg)

##### 使用并发库

在先前的高性能IO设计一文中有说到,在资源竞争的环境下,使用并发库甚至是无锁编程能够提升程序的性能,避免锁的争抢与等待.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_java_cocurrent.jpg)

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/feature/netty_java_cocurrent1.jpg)

