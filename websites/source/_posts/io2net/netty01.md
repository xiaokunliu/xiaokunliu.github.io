---
title: Netty组件源码分析
category: 网络IO编程
date: 2020-04-10 17:28:26
tags: 网络IO编程
---

<!-- more -->

###  深入Netty组件分析

讲述netty之前,先看下netty的一个整体结构如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_struct.jpg)

从上述可以看出,netty核心部分主要有基于可扩展性的事件驱动设计模型实现,通用的通信API(支持的网络协议比较丰富)以及基于ByteBuffer实现的零拷贝机制,同时从web的安全性考虑,netty支持SSL/TLS完整协议.

分析Netty的原理之前,我们先看看netty的核心组件有哪些.

#### Netty核心组件

> netty一个简单示例

```java
public class NettyServer {

    private static final String IP = "127.0.0.1";
    private static final int port = 8080;
    private static final int BIZGROUPSIZE = Runtime.getRuntime().availableProcessors() * 2;
    private static final int BIZTHREADSIZE = 100;

    private static final EventLoopGroup bossGroup = new NioEventLoopGroup(BIZGROUPSIZE);
    private static final EventLoopGroup workGroup = new NioEventLoopGroup(BIZTHREADSIZE);

    public static void main(String[] args) throws Exception {
        NettyServer.start();
    }
  
    public static void start() throws Exception {
        ServerBootstrap serverBootstrap = initServerBootstrap();
        ChannelFuture channelFuture = serverBootstrap.bind(IP, port).sync();
        channelFuture.channel().closeFuture().sync();
    }

    private static ServerBootstrap initServerBootstrap() {
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup, workGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel ch) {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast(new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0, 4, 0, 4));
                        pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8));
                        pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8));
                        pipeline.addLast(new TcpServerHandler());
                    }
                });
        return serverBootstrap;
    }
}

public class TcpServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("get new client connection ");
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
      // 使用之后需要进行释放回到内存池中方便回收,以便于下次使用的时候可重复利用
      // 如果不释放将会产生新的ByteBuff
       ((ByteBuf) msg).release();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
      ctx.close();
    }
}
```

通过上述的一个简单示例,可以看到Netty核心组件主要有:

- 启动类ServerBoostrap
- 事件轮询类EventLoop以及EventLoopGroup
- Netty自身实现的Channel,如上述的NioServerSocketChannel,作为数据传输的载体
- Netty自身实现异步操作的ChannelFuture
- 事件(入站与出站事件)与ChannelHandler
- 责任链ChannelPipeline与上下文ChannelHandlerContext
- ByteBuff组件

##### 启动类ServerBoostrap

通过上述应用程序可以看到,使用到启动类主要有以下方法:

```java
// ServerBoostrap类的定义
public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel>{
  // 添加boss事件轮询以及worker事件轮询
  ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup);

  // 添加事件源,监听客户端连接
  B channel(Class<? extends C> channelClass);

  // 添加监听事件变化后对事件响应的处理器
  ServerBootstrap childHandler(ChannelHandler childHandler);

  // 绑定服务IP以及端口
  ChannelFuture bind(String inetHost, int inetPort);
}
```

ServerBoostrap的类图结构为:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_boostrap.jpg)

根据上述可知,ServerBoostrap类依赖于Netty自身实现的Channel,需要借助Channel来实现数据传输,Channel可以是一个网络输入以及输出的载体通道.

##### Netty的事件轮询器EventLoop以及轮询器组EventLoopGroup

基于并发知识经验,有线程与线程组之分,因而可以推测到EvenntLoopGroup的作用主要为了在程序运行过程中可以管理和控制一组处理相同业务操作的EventLoop事件,对此先查看EventLoop的组成结构设计.

EventLoop定义如下:

```java
// EventLoop是具备有序性,是一个有序事件执行器
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup {
    @Override
    EventLoopGroup parent();
}

// EventLoopGroup 是一个事件执行器组,在原有的基础上增强事件功能方法
public interface EventLoopGroup extends EventExecutorGroup{}

// 对于OrderedEventExecutor,通过查看开发工具查看父类最终也是一个事件执行器组,在原有的基础上增强功能,即EventExecutor
// OrderedEventExecutor -> EventExecutor -> EventExecutorGroup

// 事件执行器具备两个功能:一个是迭代器功能,一个是计划任务的线程池
interface EventExecutorGroup extends ScheduledExecutorService, Iterable<EventExecutor>{}
```

通过上述可知,EventLoop以及EventLoopGroup具备线程池以及迭代器功能,其简要类图如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_eventloop.jpg)

结合Reactor模式以及上述的netty代码示例,我们从宏观上看`Channel`,`EventLoop`,`EventLoopGroup`以及`Thread`之间的关系如下(摘录Netty实战):

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_event_loop_thread.jpg)

对此,可以对`EventLoop`以及`EventLoopGroup`的理解如下:

- EventLoopGroup包含多个EventLoop,每个EventLoop的生命周期绑定一个线程,相当于`Thread`与`ThreadGroup`之间的关系,也就是不论是注册还是事件IO完成响应都将会在专有的线程上进行处理.
- 每个`Channel`进行注册操作都只能注册于一个`EventLoop`,但是由于一个`EventLoopGroup`存在多个`EventLoop`,因而一个`EventLoop`可以分配给一个或者多个`Channel`进行IO相关的事件处理.

对于Netty的实现是使用多线程方式来实现一个异步程序上的非阻塞式IO网络编程,同时将IO相关操作与Handler操作分离,真正实现IO操作与业务处理逻辑上的分离,并通过多线程异步的方式来回调唤醒执行IO事件完成操作的handler方法.

##### Netty自定义的Channel(重点)

Channel是Java的NIO的基本构造,Netty在原有的基础上进行方法增强,主要有:

- 对于用户而言具备以下三方面

1. 提供了channel的当前状态,即连接是open还是connected状态
2. channel相关的属性配置ChannelConfig,比如channel接收的buffer大小配置
3. channel支持读取,写出,连接以及绑定的IO操作,与ChannelPipeline一同协作完成所有和Channel相关的IO事件操作,其中ChannelPipeline以责任链的方式连接所有IO事件与Channel相关的请求事件的完成处理器.

- Channel是异步的

在netty中所有的IO操作都是以多线程的方式进行异步回调,是属于应用程序上的多线程异步操作,而本质上是使用非阻塞式IO的方式进行调用,在Reactor同步IO操作的基础上更改为异步完成处理操作的方式.类似于Proactor模式,但仍有不同,区分在于Netty的异步调用是在程序中进行回调将事件结果传递给响应的Handler,而Proactor模式是在内核中执行异步操作,异步茶走哦的实现需要借助`ChannelFuture`组件来通知异步操作的状态(成功/失败/取消).

- Channel是划分层次的

在Netty实现中,客户端`SocketChannel`是通过服务端的`ServerSocketChannel`创建的,因此对于SocketChannel的上一个层级父类是`ServerSocketChannel`,也就是说通过`ScoektChannel`的`parent()`方法能够获取到`ServerSocketChannel`,这个是属于语义上的层次划分

- 支持向下转换以便于于满足特殊传输协议的一些特有的方法

比如可以向下转换为`DatagramChannel`来调用对应的join或者leave方法

- 释放资源

很重要一点就是当socket在对应的channel通过完成读写操作时,需要释放所有的资源以便于能够被系统回收.

##### Netty自定义的ChannelFuture(重点)

在并发线程库中,Future类提供了异步操作的实现,通过调用方法返回Future,以异步的方式处理完某个操作之后通知到当前的程序执行位置,由于jdk实现需要手动检测,即`future.get()`,此时如果操作完成就会返回结果,如果未完成将会同步阻塞于线程等待完成,Netty为了解决这个繁琐的操作,自定义实现了ChannelFuture类.

```java
// 与ChannelFuture相关的
interface ChannelFuture extends Future<Void>{}

// 特殊的ChannelFuture,表示具备可写特征,即客户端获取当前类之后是具备可更改属性操作的.
interface Promise<V> extends Future<V>{}	// Future为java并发库下
interface ChannelPromise extends ChannelFuture, Promise<Void>{}
```

- ChannelFuture使用示例

```java
Channel channel = new NioSocketChannel();
// 以异步的方式与远程服务建立连接
ChannelFuture future = channel.connect(new InetSocketAddress("192.168.10.110", 8080));
future.addListener(new ChannelFutureListener(){
   void operationComplete(ChannelFuture future) throws Exception{
      if(future.future.isSuccess()){
         // 建立连接成功
        
      }else{
        //建立连接失败
        
      }
   }
});
```

- Channel的生命周期(摘录Netty实战)

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_channel_state.jpg)

```java
ChannelUnregistered: Channel已将被创建,但是还未被注册到EventLoop组件上
ChannelRegistered: Channel已经被注册到EventLoop组件上
ChannelActive: Channel已经与服务端建立连接状态,处于活跃状态,可以进行发送和写出数据
ChannelInActive: Channel与服务端没有建立连接状态
```

##### 事件与ChannelHandler(重点)

> Netty事件

Netty网络框架中的事件是按照网络数据流的相关性质来定义区分,主要有入站事件以及出站事件

- Netty入站事件: 主要是入站数据或者是相关状态发生更改而触发的事件,就是socket有新连接或者新请求
  - 已经建立连接或者连接失效/超时
  - 数据读取
  - 用户事件,即应用程序给予事件响应完成的处理程序
  - 异常错误事件

- Netty出站事件: 未来将会触发某个动作的结果,即程序主动向socket底层发起操作
  - 打开或者关闭socket远程节点的连接
  - 将数据写出或者刷新到socket缓冲区中

> ChannelHandler

在netty中对事件的响应最终会分发给ChannelHandler进行处理,每个ChannelHandler会通过一个pipeline链的方式连接起来,以下是展示ChannelHandler链式处理入站与出战事件的简要流程.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_channel_handler.jpg)

ChannelHandler在netty源码中主要有包含以下几个内容:

- 交由具体子接口定义出入站事件处理方法

```java
// 1. 交由具体子接口定义出入站事件处理方法
// ChannelHandler并没有提供很多的方法声明,同时通过上述的入站和出站事件处理,我们也很容易想到ChannelHandler存在处理入站事件的ChannelInboundHandler以及出站事件的ChannelOutboundHandler,同时在netty中默认有对应实现方式
// ChannelInboundHandlerAdapter实现入站事件的处理
// ChannelOutboundHandlerAdapter实现出站事件的处理
// ChannelDuplexHandler能够同时处理入站和出站事件
```

ChannelHandler类图组成结构如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_channelhandler_class.jpg)

- 上下文对象,ChannelHandlerContext

```java
// 2. 上下文对象,ChannelHandlerContext
// 要想在上述ChannelHandler的链式事件处理流程,就必须满足两个条件,一个是如何在每个单独ChannelHandler的处理器传递事件,二是每个ChannelHandler是如何通过链式绑定关联的
// ChannelHandler通过ChannelHandlerContext为每个对应的Handler传递事件,因此ChannelHandler必然存在一个上下文对象负责事件传递,类似于EDA的事件通道
// netty中事件触发就会创建响应事件的ChannelHandler,并添加到ChannelPipeline中,通过链表的数据结构来维护每个Handler之间关联,同时将ChannelPipeline存储在上下文中,可以通过上下文对象获取管道对象
```

ChannelHandler,ChannelHandlerContext以及ChannelPipeline之间的关联如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_channel_handler_dep.jpg)

- 能够存储并管理有状态的信息

如果在ChannelHandler定义专属于某一个连接的成员变量数据,即连接需要保持有状态数据,为了防止数据竞争产生数据不一致的问题,必须为每一个连接请求的处理操作创建一个新的handler对象去单独处理,保证业务的数据一致性.比如下面一个例子

```java
// 现定义一个需求:需要进行登录才能够获取数据,于是就有了以下的定义
interface Message{
  // methods
}

class DataServerHandler extends SimpleChannelInboundHandler<Message> {
  
  // 存储有状态数据
  private boolean isLogined;
  
  public void channelRead0(ChannelHandlerContext ctx, Message msg){
     if(message instanceof LoginMessage){
       auth((LoginMessage)message);
       isLogined = true;
  	}else if(message instanceof GetDataMessage){
       if(isLogined){
          ctx.writeAndFlush(fetchSecret((GetDataMessage) message));
       }else{
         fail();
       }
  	}
}

// 为了避免数据竞争产生数据不一致问题,避免上述需求中的非法登录用户获取到登录数据,必须为每个请求处理连接在提交给handler处理前保证handler是针对当前的连接是1:1的处理方式,即一个连接对应一个channel处理器,对此,需要在添加channel时通过以下的方式进行添加
serverBootstrap.childHandler(new ChannelInitializer<Channel>(){
  public void initChannel(Channel channel){
    channel.pipeline().addLast(new DataServerHandler());
    // 如果一个连接需要用到多个handler协同处理,则只需要调用addLast添加即可
    // 这样每次有事件发生的时候,对应的一个请求处理连接的事件响应处理都会重新创建handler,即保证每个请求连接的事件响应handler都是新创建的,避免了数据竞争产生的数据不一致问题
  }
}); 

// 如果要保证handler只创建一次,那么就只需要进行调用childHandler添加对应实现的具体Handler实例
```

- 使用AttributeKey来存储信息

尽管对于存储有状态的信息需要新创建channel去处理它,但是并不是所有情况都是需要创建一个新的handler去处理不同的连接,比如对于通用的共享数据,不存在于不同连接的状态变化,但是为了能够保证共享数据是安全的,为此可以使用AttribuiteKey存储这类数据信息,同时在每个handler中都会有一个上下文对象,而当前的AttributeKey能够通过上下文对象获取到,因此对于AttributeKey的获取在不同handler中可以通过上下文对象来获取,并且为对应的handler添加注解@Sharable能够保证线程是安全的,比如下面例子.

```java
interface Message{
  // methods
}

@Sharable
class DataServerHandler extends SimpleChannelInboundHandler<Message> {
  
  // 存储有状态数据
  private boolean sharedObject;
  public void channelRead0(ChannelHandlerContext ctx, Message msg){
    	// ...
  	}
}

serverBootstrap.childHandler(new ChannelInitializer<Channel>(){
  private static final DataServerHandler SHARED = new DataServerHandler();
  public void initChannel(Channel channel){
    // 这样保证在处理不同连接的处理链pipeline中存储的handler都是相同的,并且是线程安全的
    channel.pipeline().addLast(SHARED);
  }
}); 
```

- 注解@Sharable

根据上述的AttributeKey分析可知,对于注解为Sharable的说明如下:

1. 如果定义的Handler添加注解为@Sharable,则表示Handler在整个处理链pipeline中都是属于无竞争的环境,数据不存在线程安全问题,同时如果是无状态的数据可以通过只创建一次handler实例来完成整个事件处理链pipeline的handler处理.
2. 如果没有添加注解说明,那么为了保证每个handler的共享数据是属于线程安全的,就必须为每次亲戚连接的操作创建新的handler,也就是说不同的连接的事件处理链pipeline存储的handler都是连接特有的,属于连接与handler的1:1模式,这样才能保证数据是线程安全.

- ChannelHandler的方法回调机制

Netty设计实现是基于Reactor模式实现,在上文已经模拟Reactor模式的实现原理,可以知道在事件触发之后就会回调执行ChannelHandler实现类中的方法执行响应事件的处理逻辑,即主要回调有以下方法:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_channel_callback.jpg)

根据上述的类图结构,netty默认实现有出站与入站的HandlerAdaper实现,以及含有出站和入站实现的`ChannelDuplexHandler`,因此在实际使用中可以继承上述相关类来重写部分方法以实现目标业务逻辑的处理程序.

- ChannelHanlder的生命周期

```java
// handlerAdded: 当ChannelHandler添加到pipeline被调用
// handlerRemoved: 当ChannelHandler从pipeline中移除时被调用
// exceptionCaught: 在处理过程中ChannelPipeline发生异常时被调用
```

##### 责任链ChannelPipeline与处理器上下文ChannelHandlerContext(重点)

在讲述一个责任链与上下文对象前,先根据上述的Channel事件处理链pipeline演示一个责任链设计模式的代码实现原理.

- 伪代码实现

```java
/**
 * 责任链设计:
 *    channel1 -> channel2  -> channel3 -> ...
 */
// channel1处理完之后的结果将会传递给channel2进行处理,然后将channel2的处理结果传递给channel3再进行处理,那么什么时候结束呢?
//要么遍历到没有下一个channel节点为止就结束
//要么就是直接在当前节点中断不进行往下传播事件
// 根据netty的channel可知,pipeline需要依赖到一个上下文对象,通过上下文对象来实现责任链的数据传输,于是就有以下的定义.

class Main{
 	public static void main(String[] args){
     // 类比netty添加方式
     HandlerPipeline pipeline = new HandlerPipeline();
     pipeline.addLast(new HandlerTest());
     pipeline.addLast(new HandlerTest());
  }
}

// 基于链表结构存储handler
class HandlerPipeline {
  private HandlerContext head = new HandlerContext(new Handler(){
    	void doHandler(HandlerContext context, Object val){
        context.nextRun(val);
      }
  });
  
  public void addLast(Handler handler){
     HandlerContext ctx = head;
     while(ctx.next != null){
        ctx = ctx.next;
     }
    ctx.next = new HandlerContext(handler);
  }
}

// 通过上下文保存当前handler并传递数据到下一个context进行处理
class HandlerContext {
  private HandlerContext next;
  private Handler handler;
  
  HandlerContext(Handler handler){
    this.handler = handler;
  }
  
  void nextRun(Object val){
    if(this.next != null){
      this.next.handler(val);
    }
  }
  
  void handler(Object val){
    this.handler.doHandler(this, val);
  }
}

// handler处理器
interface Handler{
   void doHandler(HandlerContext context, Object val);
}

class HandlerTest implements Handler{
   public void doHandler(HandlerContext context, Object val){
       // 传播给下一个handler
       context.nextRun(val);
      // 如果不传播,在当前handler停止,就无需调用nextRun方法
   }
}
```

- Netty责任链运作流程

  > 责任链流程(摘录源码)

```java
 *  +---------------------------------------------------+---------------+
 *  |                           ChannelPipeline         |               |
 *  |                                                  \|/              |
 *  |    +---------------------+            +-----------+----------+    |
 *  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  .               |
 *  |               .                                   .               |
 *  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
 *  |        [ method call]                       [method call]         |
 *  |               .                                   .               |
 *  |               .                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  +---------------+-----------------------------------+---------------+
 *                  |                                  \|/
 *  +---------------+-----------------------------------+---------------+
 *  |               |                                   |               |
 *  |       [ Socket.read() ]                    [ Socket.write() ]     |
 *  |                                                                   |
 *  |  Netty Internal I/O Threads (Transport Implementation)            |
 *  +-------------------------------------------------------------------+
```

> 连接与事件处理链协作流程简要流程图

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_pipeline_context.jpg)

- Netty之ChannelPipeline源码分析

```java
// 1.pipeline的创建
// 每个Channel都有自己的pipeline,并且当一个新的channel被创建的时候会自动创建

// 2. 责任链流程 -- 见上述责任链流程

// 3. pipeline在上下文进行事件传播的方法
// 入站事件与出站事件
 /* <ul>
   * <li>Inbound event propagation methods:
   *     <ul>
   *     <li>{@link ChannelHandlerContext#fireChannelRegistered()}</li>
   *     <li>{@link ChannelHandlerContext#fireChannelActive()}</li>
   *     <li>{@link ChannelHandlerContext#fireChannelRead(Object)}</li>
   *     <li>{@link ChannelHandlerContext#fireChannelReadComplete()}</li>
   *     <li>{@link ChannelHandlerContext#fireExceptionCaught(Throwable)}</li>
   *     <li>{@link ChannelHandlerContext#fireUserEventTriggered(Object)}</li>
   *     <li>{@link ChannelHandlerContext#fireChannelWritabilityChanged()}</li>
   *     <li>{@link ChannelHandlerContext#fireChannelInactive()}</li>
   *     <li>{@link ChannelHandlerContext#fireChannelUnregistered()}</li>
   *     </ul>
   * </li>
   * <li>Outbound event propagation methods:
   *     <ul>
   *     <li>{@link ChannelHandlerContext#bind(SocketAddress, ChannelPromise)}</li>
   *     <li>{@link ChannelHandlerContext#connect(SocketAddress, SocketAddress, ChannelPromise)}</li>
   *     <li>{@link ChannelHandlerContext#write(Object, ChannelPromise)}</li>
   *     <li>{@link ChannelHandlerContext#flush()}</li>
   *     <li>{@link ChannelHandlerContext#read()}</li>
   *     <li>{@link ChannelHandlerContext#disconnect(ChannelPromise)}</li>
   *     <li>{@link ChannelHandlerContext#close(ChannelPromise)}</li>
   *     <li>{@link ChannelHandlerContext#deregister(ChannelPromise)}</li>
   *     </ul>
   * </li>
   * </ul>
 */

// 4. 构建一个ChannelPipeline
// 当我们的web服务为每个请求处理对应的decode-process-encode时,对于执行比较耗时的操作需要将线程隔离处理,也就是需要有针对Group对process进行处理,而其他线程仍然可以处理decode以及encode非耗时逻辑,可以通过以下的方式:

static final EventExecutorGroup group = new DefaultEventExecutorGroup();
pipeline = ch.pipeline();
pipeline.addLast("decoder", new MyProtocolDecoder());
pipeline.addLast("encoder", new MyProtocolEncoder());
// 这个时候的process处理会单独放在以下的group进行处理
pipeline.addLast(group, "handler", new MyBusinessLogicHandler());


// 5. ChannelPipeline是属于线程安全类
```

- Netty之ChannelHandlerContext源码分析

```java
// 1. 通过唤醒回调后在pipeline流程链中向不同的handler传递信息

// 2. 上下文存储的数据可以实现事件触发执行传递到不同的handler方法中,甚至可以是在不同线程中实现数据的共享,比如以下代码:

public class MyHandler extends ChannelDuplexHandler{
  	private ChannelHandlerContext ctx;
    
    public void beforeAdd(ChannelHandlerContext ctx){
       // 可以在添加到pipeline之前保存ctx信息
       this.ctx = ctx;
    }
  
   public void login(String username, password) {
     		// 将保存的ctx存储登录信息并将登录信息传递到责任链pipeline下后续的handler获取使用
        ctx.write(new LoginMessage(username, password));
     }
}

// 3. 存储有状态的信息,详细可以查看上述的ChannelHandler使用

// 4. 一个handler可以拥有多个context信息,因为一个handler可以添加到一个或者多个pipeline中,而每个pipeline都会对应着一个context,因而一个handler是可以拥有一个或者多个context,比如计算handler被添加到pipeline的次数

public class FactorialHandler extends ChannelInboundHandlerAdapter{
   private final AttributeKey<Integer> counter = AttributeKey.valueOf("counter");
   
   public void channelRead(ChannelHandlerContext ctx, Object msg){
      Integer a = ctx.attr(counter).get();
      if (a == null) {
        a = 1;
      }
     attr.set(a * (Integer)msg);
   }
}

// 下面将会进行4次计数器的计算,也就是一个handler实例添加到不同或者相同的active的pipeline中,其上下文对象是不一样的
ChannelPipeline p1 = channel.pipeline();
p1.addLast("f1", fh);
p1.addLast("f2", fh);

ChannelPipeline p2 = channel.pipeline();
p1.addLast("f3", fh);
p1.addLast("f4", fh);
```

##### ByteBuf组件(重点)

> 支持的API

- 可以被用户自定义的缓冲区扩展
- 通过内置的复合缓冲区实现透明的零拷贝
- 容量可以按需增长
- 拥有readerIndex与writeIndex,因而在读写之间不需要像NIO的ByteBuffer通过flip()方法进行切换
- 支持方法的链式调用
- 支持引用计数
- 使用池化技术

> 工作原理

存在两个索引,一个是读取索引,一个写入索引,当使用ByteBuf调用read方法的时候,readIndex将会向前移动,即readIndex+1,同样地,对于写入数据的时候,对应的writeIndex也会增加,当`readIndex==writeIndex`也就意味着读取数据达到数组的末尾,再次进行读取时会发生IndexOutOfBoundsException,而对于`writeIndex==Capacity`即ByteBuf的容量大小时也会发生下标越界异常,名称以read或者write将会推进对应的索引数值,而名称以set或者get的方法调用时将不会对读取或者写入索引进行递增操作.

> 字节级源码分析

- 随机访问: `ByteBuf`类似于一个字节数组,于数组索引具备相同的特征,即下标从0开始,以`capacity - 1`的下标为末尾索引,可以按照数组的方式对其通过下标随机访问,此时对于拥有readerIndex以及writeIndex值是不变,但可以通过调用`readerIndex(index)`或者`writeIndex(index)`来更改对应的索引值

```java
for(int i=0; i<buffer.capacity(); i++){
   byte b = buffer.getByte(i);
   logger.info("char s is " + (char)b);
}
```

- 顺序访问: 主要是通过`readerIndex`以及`writeIndex`两个索引值指针实现顺序访问,对于`ByteBuf`的顺序访问存在丢弃字节,可读字节以及可写字节的概念,对此摘录源码分析如下:

```java
/* 
 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      |                   |     (CONTENT)    |                  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 *.   
 */
// 可读字节区域: 代表ByteBuf还未读取到ByteBuf的数据区域,在netty的ByteBuf中以read或者skip开头的读取数据的方法都会在指针readerIndex实现计数的自增加操作,对于可读指针取值范围: 0<=readerIndex<=writeIndex

// 可写字节区域: 代表从writeIndex - capacity之间的区域为空闲区域,能够继续存储数据来填充区域,如果满足writeIndex < capacity代表该ByteBuf拥有足够的区域进行写入数据.

// 丢弃的字节区域: 表示已经被读取过的ByteBuf片段区域,初始化状态的时候,区域大小为0,如果数据一直在被读取,那么对应的区域大小会增加到writeIndex,也就是对于该区域满足: 0<= size <= writeIndex.

// 调用discardReadBytes()方法的区域变化:
/*
 *  BEFORE discardReadBytes()
 *
 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 *
 *
 *  AFTER discardReadBytes()
 *
 *      +------------------+--------------------------------------+
 *      |  readable bytes  |    writable bytes (got more space)   |
 *      +------------------+--------------------------------------+
 *      |                  |                                      |
 * readerIndex (0) <= writerIndex (decreased)        <=        capacity
 */
```

- 字节清除方法`clear`

```java
// 关于清除方法直接通过查看区域的变化即可
/*
 * <pre>
 *  BEFORE clear()
 *
 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 *
 *
 *  AFTER clear()
 *
 *      +---------------------------------------------------------+
 *      |             writable bytes (got more space)             |
 *      +---------------------------------------------------------+
 *      |                                                         |
 *      0 = readerIndex = writerIndex            <=            capacity
 * </pre>
 */
```

- 搜索操作

```java
// 对于一些普通字节的查询,可以调用indexOf(int, int, byte)或者是bytesBefore(int, int, byte)完成,bytesBefore这个方法尤其对于以一些特殊字符结尾的字符串尤其有用.

// 对于更为复杂的操作,比较存在不同系统的特殊符号查询,可以通过ByteProcessor接口指定查询特定的符号内容,调用forEachByte(int, int, ByteProcessor)方法完成字节的搜索
```

- Derived Buffers(派生缓冲区)

```java
// 可以通过以下的方法创建一个新的ByteBuf缓冲区,每个派生出来的缓冲区都拥有自己的readerIndex,writeIndex以及标记索引,与NIO的buffer议案具备数据共享,因而当派生的缓冲区数据发生变化的时候,对应的源缓冲区也会发生变化.

// 如果要让数据不具备共享,可以通过使用copy()方法来实现一个新的数据副本,这个时候与派生的数据缓冲区区分在于copy()拥有独立的数据副本信息,可以通过以下的图示来分析,假设现在申请的一块源bytebuf是使用堆外内存存储数据的方式(堆内内存也是同理),这个时候派生的缓冲与copy的缓冲内存区域分布如下:
```

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_byte_silce_copy.jpg)

通过上述可知,在JVM堆内存中创建ByteBuf对象,分别指向对应数据存储的区域,对于Java程序而言,派生缓冲区对象在JVM中创建ByteBuf对象指向原有存储数据的内存区域,因而对于派生的缓冲区的数据如果发生变化,对应的源缓冲区数据也会发生变化,相比使用copy()的申请数据一份新的数据副本方式,减少了数据内存的复制操作,避免更多的内存占用冗余的数据.

- Netty的ByteBuf转换为JDK存在的类型

```java
// 1. 转换为byte[]
if(byteBuff.hasArray()){
   byte[] byteArr = byteBuff.array();
}

// 2. 转换为NIO的Buffers
if(byteBuff.nioBufferCount() > 0){
   ByteBuffer nioByteBuffer = byteBuff.nioBuffer();
}

// 3. 转换为String
String str = byteBuff.toString(Charset.forName("utf8"));

// 4. 转换为IO的字节流
ByteBufInputStream in = new ByteBufInputStream(byteBuff);
ByteBufOutputStream out = new ByteBufOutputStream(byteBuff);
```

> ByteBuff使用模式

- 使用堆缓冲区

将数据存储在JVM堆内存中,也就是从JVM的内存中申请内存区域来存放ByteBuff数据,这种模式称为支撑数组(Backing Array)

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_byte_buff_heap.jpg)

通过上述可知,在Java程序进行网络IO数据传输的时候,对于存储在堆内存的缓冲数据,不论是读取还是写入数据,都需要经过JVM堆内缓冲,然后将数据复制到操作系统内存的一块区域中,最后在刷新到网卡设备的时候,需要将内存数据复制到socket缓冲区再进行数据刷新.

- 使用堆外缓冲区

直接从操作系统中申请内存区域来存储ByteBuff数据,但是分配和释放内存相对会更为昂贵,同时如果这个时候不知道时候用于数组相关的数据操作而使用堆外内存的缓冲数据,那么这个时候就需要进行一次数据内存的复制,相比堆内存操作更为复杂.

```java
// 现在有一数组数据
byte[] arr = [1,2,3,4,5];
// 这个时候要发送数据出去,可考虑使用堆外内存数据缓冲,避免数据缓冲多一次内存复制,将数据发送到网卡中
// 接收网卡数据的时候,如果这个时候知道接收的数据为数组且需要堆数组数据进行遍历,那么这个时候直接使用堆外内存存储,还需要再进行一次数据内容的复制,而如果使用堆内内存存储,直接可以通过直接获取到数据转换为数组进行操作
```

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_byte_buff_direct.jpg)

- 复合缓冲区(零拷贝机制)

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_compiste_byte_buff.jpg)

通过上述可知,复合缓冲区是将不同存储物理位置的缓冲区数据合并为单个缓冲区的虚拟表示,属于逻辑上述的缓冲区的数据合并,由此可知,如果程序中需要将一块有关联但存储物理位置不同的缓冲区数据进行一起操作的话,可以使用复合缓冲区方式将多个缓冲区数据进行合并,这个时候不存在数据的复制,即零拷贝机制,实现多个缓冲区数据合并为单个缓冲区,可以视为单个缓冲区进行操作.

> ByteBuff核心类图

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_byte_buff_class.jpg)

通过上述类图结构可知,ByteBuff划分为三个维度,即

- 存储方面: 使用堆内与堆外内存存储
- 资源利用技术: 使用非池化与池化技术,池化技术与内存管理将在讲netty高性能会详细说明
- 直接通过底层操作内存: Unsafe操作

> 使用ByteBufHolder接口

在一个web服务程序中,如果能够将一个http请求(请求头/请求体/状态码/cookie等信息)都封装一起以包的形式进行接收或者发送,那么对程序开发者而言将会带来很多的便利,对此,在netty框架提供了ByteBufHolder接口来存储ByteBuf之外的属性信息,同样提供了缓冲池化,底层访问数据以及引用计数的方法.对于网络编程中实现自定义的消息协议,可以采用ByteBufHolder接口来有效承载消息的信息.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_bufholder.jpg)

通过上述可知,ByteBuf具备引用计数(后续在高性能会写到)特性,能够在资源不被对象所持有时释放来优化内存使用和性能的技术.

> ByteBuf内存分配策略

- 按需分配:ByteBufAllocator接口

主要实现类有池化与非池化技术实现UnpooledByteBufAllocator以及PooledByteBufAllocator类,netty默认使用使用PooledByteBufAllocator实现池化的ByteBuf,但是对于默认也存在以下的规则:

```java
// 默认创建池化且为堆外内存存储的ByteBuf,如果当前环境支持Unsafe底层操作,那么默认就会使用Unsafe+堆外存储+池化技术方式来创建ByteBuf
// 如果显示使用PreferHeapByteBufAllocator方式进行分配,则会创建堆内数据存储的ByteBuf
```

ByteBufAllocator的类图如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_bytebuf_allocation.jpg)

- Unpooled分配ByteBuf工具类

对上述的ByteBufAllocator进一步封装,对外提供操作的工具类,关于核心API直接摘录《Netty实战》书籍:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/netty_unpooled.jpg)

- ByteBufUtil操作ByteBuf工具类

```java
// 核心方法
// 1. hexDump: 以16进制形式打印ByteBuf内容
// ByteBuf内容为: [0, 0, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
// 16进制打印出来的数据为: 00 00 00 01 02 03 04 05 06 07 08 09 0a 0b 
// (空格是手动加的,实际为: 0000000102030405060708090a0b)

// 2. 比较两个ByteBuf的存储的数据是否一致(内存 + 数据大小)
```
