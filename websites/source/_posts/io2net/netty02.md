---
title: 深入Netty事件流程分析(上)
category: 网络IO编程
date: 2020-04-19 11:14:54
tags: 网络IO编程
---

<!-- more -->
#### Netty框架核心内容

##### 丰富的Buffer数据结构

Netty在NIO的ByteBuffer基础上自定义一套自己的Buffer API,其实现的Buffer API具备以下特性:

- 如果需要可以自定义自己的buffer类型
- 内建composite buffer类型实现零拷贝机制(无需在内存实现数据复制)
- 可以支持动态扩容,类似于StringBuffer
- 不需要像NIO的Buffer需要每次调用flip(),Netty实现的Buffer通过readerIndex以及writeIndex避免flip()调用
- 通常会比ByteBuffer在性能上更好

零拷贝示例,通过分割与组合:

```java
// 假设buffer1以及buffer2都存储在堆外内存,堆内内存同理(只是在JVM中)
ByteBuf httpHeader = buffer1.silice(OFFSET_PAYLOAD, buffer1.readableBytes() - OFFSET_PAYLOAD);
ByteBuf httpBody = buffer2.silice(OFFSET_PAYLOAD, buffer2.readableBytes() - OFFSET_PAYLOAD);

ByteBuf http = ChannelBuffers.wrappedBuffer(httpHeader, httpBody);
```

上述的零拷贝示意图如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/zero_copy.jpg)

##### 使用异步IO的API

- 基于TCP/IP的NIO模式
- 基于TCP/IP的BIO模式
- 基于UDP/IP的BIO模式

##### 基于事件驱动设计的责任链设计

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/pipeline.jpg)

##### 支持特性组件,简化开发

- 支持SSL/TLS安全协议
- HTTP/WebSocket协议的实现
- 支持Google的Protocol协议
- 支持编解码器,Netty也有内置一些编解码器实现

#### Netty线程模型

讲述Netty线程模型之前,摘录Netty官网的描述:

```reStructuredText
The Netty project is an effort to provide an asynchronous event-driven network application framework and tooling for the rapid development of maintainable high-performance and high-scalability protocol servers and clients.
```

从上述官网可得到以下的信息:

- Netty是基于EDA事件驱动架构设计实现
- Netty采用异步方式来完成事件通知,完成事件之后会进行回调唤醒Handler,可以理解为Proactor模式,或者是多线程异步执行的Reactor模型.
- Netty是一个高性能高扩展性的web服务框架,可以在不影响性能情况快速实现web服务的开发

在Netty组件分析中,EventLoop是一个线程池执行器,同时对于长时间的耗时Handler操作,会额外分配一个线程池执行器来负责处理handler相关的业务逻辑,于是基于Reactor/Proactor模型可知,Netty的一个线程模型如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/thread_model.jpg)

#### Netty事件流程

基于上述的Netty线程模型的理解,现摘录官网的一个EchoServer例子来深入分析Netty事件流程,对应的EchoServer代码示例如下:

```java
// 仅摘录部分核心代码
// main方法
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
final EchoServerHandler serverHandler = new EchoServerHandler();
try {
  ServerBootstrap b = new ServerBootstrap();
  b.group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .option(ChannelOption.SO_BACKLOG, 100)
    .handler(new LoggingHandler(LogLevel.INFO))
    .childHandler(new ChannelInitializer<SocketChannel>() {
      @Override
      public void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline p = ch.pipeline();
        //p.addLast(new LoggingHandler(LogLevel.INFO));
        // 将处理读写事件的handler添加到责任链中
        p.addLast(serverHandler);
      }
    });

  // Start the server.
  ChannelFuture f = b.bind(PORT).sync();

  // Wait until the server socket is closed.
  f.channel().closeFuture().sync();
} finally {
  // Shut down all event loops to terminate all threads.
  bossGroup.shutdownGracefully();
  workerGroup.shutdownGracefully();
}

@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.write(msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```

根据上述的代码示例,其运作的核心流程如下:

- 初始化Boss以及Worker的事件轮询NioEventLoop线程,即Reactor线程
- 创建服务启动类ServerBootsrtap
- 将Boss以及Worker的Reactor线程添加到服务启动类中
- 服务启动类创建并注册服务端的Channel
- 服务启动类为服务端的Channel配置可选属性
- 服务启动类添加一个通用共享且日志级别为INFO的处理器并应用在整个Netty的pipeline
- 基于之前的Channel组件分析,我们知道childChannel也就是对应客户端的SocketChannel,这个时候是服务启动类为SocketChannel添加一个初始化的Handler,并在后续基于事件触发完成之后执行责任链下的handler回调
- 至此,一系列的初始化操作完成,这个时候服务启动类开始为ServerChannel绑定端口开始对客户端连接进行监听
- 关闭连接的监听以及销毁Reactor线程释放空间

对此,基于上述给出的完整服务端流程,现对上述流程结合源码进行分析与总结.

##### EventLoopGroup与EventLoop初始化流程

> EventLoopGroup初始化流程

- NioEventLoopGroup类图结构

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/eventloopgroup_class.jpg)

- EventLoopGroup初始化源码

```java
// NioEventLoopGroup构造器 
// NioEventLoopGroup.java
public NioEventLoopGroup(int nThreads) {
    this(nThreads, (Executor) null);
 }

public NioEventLoopGroup(int nThreads, Executor executor) {
  this(nThreads, executor, SelectorProvider.provider());
}

public NioEventLoopGroup(
  int nThreads, Executor executor, final SelectorProvider selectorProvider) {
  // 默认使用阻塞式轮询策略
  this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}

public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,final SelectStrategyFactory selectStrategyFactory) {
        super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());   // channel处理不过来的时候直接丢弃
}

// MultithreadEventLoopGroup.java
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
  // 默认线程数量为CPU*2 或者是通过 io.netty.eventLoopThreads进行配置
  super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}

// MultithreadEventExecutorGroup.java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
  // 创建默认事件执行选择器(从Group中选择一个EventLoop来处理Channel的策略选择器)
  this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}

// 初始化Group的核心方法
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
        EventExecutorChooserFactory chooserFactory, Object... args) {
}

// 上述可以看到初始化创建默认具备线程池的一些默认策略(线程大小/线程工厂/存储任务队列/丢弃策略)/创建默认的事件轮询选择器/默认的IO复用器提供者
```

- EventLoopGroup初始化的核心流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/eventloopGroup_init.jpg)

根据上述可以知道,EventLoopGroup初始化的操作主要是初始化一组EventLoop的执行器,并创建选举EventLoop的选择器,并为每个EventLoop在销毁的时候添加监听器以便于程序能够获取当前EventLoop销毁情况,同时每个EventLoop对外提供服务都是只读模式,也就是选举EventLoop都是处于只读的稳定版本.

> EventLoop的创建流程

EventLoop的创建流程包含在上述EventLoopGroup为每个执行器(EventLoop)进行初始化的过程,即在源代码中如下:

```java
// MultithreadEventExecutorGroup.java
// 初始化执行器
children[i] = newChild(executor, args);

// newChild的实现子类NioEventLoopGroup
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
  EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
  return new NioEventLoop(this, executor, (SelectorProvider) args[0],
                          ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}

// NioEventLoop
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
             SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
             EventLoopTaskQueueFactory queueFactory) {
  super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
        rejectedExecutionHandler);
  this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
  this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
  final SelectorTuple selectorTuple = openSelector();
  this.selector = selectorTuple.selector;
  this.unwrappedSelector = selectorTuple.unwrappedSelector;
}

```

EventLoop的初始化流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/eventloop_init.jpg)

基于上述的EventLoopGroup与EventLoop的认知,我们来总结下EventLoopGroup,EventLoop,EventExecutor以及Thread之间的关系,首先先从源码开始分析如下:

```java
// MultithreadEventExecutorGroup.java
// ThreadPerTaskExecutor看成线程池 - 对应默认的线程工厂类
executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());

// 有多少个线程就有多少个EventLoop
children = new EventExecutor[nThreads];

// 用线程池创建EventLoop
children[i] = newChild(executor, args);

// SingleThreadEventExecutor.java
// this为NioEventLoop
this.executor = ThreadExecutorMap.apply(executor, this);

// ThreadExecutorMap.java
// 创建新的执行器
public static Executor apply(final Executor executor, final EventExecutor eventExecutor) {
  // check not null ...
  return new Executor() {
    @Override
    public void execute(final Runnable command) {
      // ThreadPerTaskExecutor.execute -> apply
      // 启动一个线程执行任务并传递事件轮询器
      // FastThreadLocalThread.run()
      executor.execute(apply(command, eventExecutor));
    }
  };
}

// 新任务Task
public static Runnable apply(final Runnable command, final EventExecutor eventExecutor) {
  ObjectUtil.checkNotNull(command, "command");
  ObjectUtil.checkNotNull(eventExecutor, "eventExecutor");
  return new Runnable() {
    @Override
    public void run() {
      // 将EventLoop存储到FastThreadLocal(即保证FastThreadLocalThread独占持有自己的EventLoop)
      setCurrentEventExecutor(eventExecutor);
      try {
        // 执行任务
        command.run();
      } finally {
        // 任务执行完之后释放独占EventLoop的资源
        setCurrentEventExecutor(null);
      }
    }
  };
}

private static void setCurrentEventExecutor(EventExecutor executor) {
  // 使用FastThreadLocal来存储事件轮询器，保证每个事件轮询器都会有对应的一个线程来处理
  mappings.set(executor);
}
// 通过上述的流程可知,在每个EventLoop都含有一个新的Executor
// 而每一个Executor都通过默认的线程工厂创建一个FastThreadLocalThread线程来处理task任务
// 此时的Task任务为一个新的任务task
```

通过源码分析,可以得到以下简要的EventLoopGroup,Group下的线程池Executor,EventLoop与EventLoop下的Executor以及Thread之间的关系如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/eventloop_thread.jpg)

通过上述示意图可知,每个EventLoop处理任务时都会通过Group下的Executor来创建对应的线程来执行EventLoop的事件任务,并且为了保证并发安全问题,在每次处理任务之前,将会把当前的EventLoop与Thread进行绑定,也就是当前EventLoop为当前执行的线程Thread所独占持有,通过FastThreadLocal来维护两者之间的关系,一旦EventLoop事件任务处理完成之后,将解除两者的绑定.同时也可以看到处理一组事件任务的Thread将通过线程组的方式进行维护和管理.

> Netty线程模型细化

可以看到上述一个EventLoop绑定一个专有的线程,由专有的线程负责处理EventLoop的事件,且一个channel都会对应着一个EventLoop来负责处理channel相关的事件,同时一个EventLoop/Thread能够处理多个Channel需要依赖于AIO或者是NIO的API才能实现,AbstractBootstrap处理服务端Channel,ServerBootstrap处理客户端Channel,而对于BIO模型而言,只能一个EventLoop/Thread处理对应一个Channel,即摘录《Netty实战》关于NIO/OIO(old IO,BIO)模型如下:

- 基于NIO/AIO的线程模型

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/eventloop_thread_channel.jpg)

- 基于BIO的线程模型(OIO为old IO,即使用BIO的API)

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/group_thread_bio_channel.jpg)

- EventLoop启动任务的执行源码

```java
// 调用以下的方法时执行流程
// SingleThreadEventExecutor.java
eventLoop.execute(task);

// 这里的线程执行流程不弄清楚,后面的事件流程将很理解
// 根据类设计可知,execute为SingleThreadEventExecutor下的方法,结合上面的EventLoop初始化流程可知,每个EventLoop都拥有一个内置的Executor,而这个Executor用于创建FastThreadLocalThread线程来保证当前eventloop与当前线程之间的绑定关联,源码如下:

private void execute(Runnable task, boolean immediate) {
  // 判断当前执行的线程是否与eventloop对应(EventLoop - Thread绑定一起）
  boolean inEventLoop = inEventLoop();
  // 将任务添加到队列中,如果队列满则丢弃当前任务
  addTask(task);
  if (!inEventLoop) {
    // 启动一个线程,如果当前EventLoop持有的线程已经开启过则直接跳过,如果开启过线程,则执行doStartThread方法
    startThread();
    // ...
}

private void doStartThread(){
  // EventLoop持有的executor来创建一个FastThreadLocalThread线程,在该线程中保证当前事件轮询器与线程处于线程安全,通过FastThreadLocal将线程与EventLoop进行关联
  executor.execute(new Runnable() {
    	//....
  });               
}
```

结合EventLoop初始化对应的executor以及ThreadExecutorMap中的源码,现将一个不在当前线程的EventLoop提交任务时创建一个完整线程执行细节流程绘制如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/eventloop_thread_detail.jpg)

也就是说,最终处理任务task都在NioEventLoop执行的run方法中体现,或者更为严格意义上来取决于我们选择的EventLoop的IO操作模式,具体是交由EventLoop的IO操作模式的run方法通过队列中获取任务来进行处理,于是根据源码中提供的任务队列与拒绝策略,对于EventLoop处理任务的流程如下(摘录自《Netty实战》):

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/eventloop_task_nettyinaction.jpg)

- 与线程池不一样的是,EventLoop是与指定的线程绑定在一起,也就是一个线程处理一个EventLoop,并且在整个Web服务中EventLoop始终是由当前的专有线程负责事件的任务的处理
- 当添加任务到EventLoop执行的时候,需要校验当前的线程是不是持有之前分配好的EventLoop,如果不是那么就添加到任务队列进行等待EventLoop下一次处理事件时再执行,如果队列满了,那么此时就会触发拒绝策略丢弃任务,如果是之前分配好的EventLoop那么就会直接执行任务Task.

> Netty之NIO事件轮询流程

基于上述的线程任务流程分析之后,我们知道在EventLoop中最终会调用NioEventLoop下的run方法,对此,现该run方法执行的事件轮询操作流程进行分析.

- 事件轮询源码

```java
// NioEventLoop.run()核心代码
for(;;){
  	// 检测当前的EventLoop的队列中是否有任务
    if (!hasTasks()) {
      strategy = select(curDeadlineNanos);
    }
  // 根据服务器配置eventloop的IO处理能力比率
  if (ioRatio == 100) {
    // 如果IO处理比率高，则同时处理就绪事件以及当前轮询器队列中的所有任务
    // 不然就分开处理
    try {
      if (strategy > 0) {
        // 处理一系列的就绪事件
        processSelectedKeys();
      }
    } finally {
      // Ensure we always run tasks.
      // 执行所有的任务
      ranTasks = runAllTasks();
    }
  } else if (strategy > 0) {
    final long ioStartTime = System.nanoTime();
    try {
      // 处理就绪事件,处理ACCEPT/READ/WRITE事件
      processSelectedKeys();
    } finally {
      // 在一定事件内处理队列中的任务
      // Ensure we always run tasks.
      final long ioTime = System.nanoTime() - ioStartTime;
      ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }
  } else {
    // 处理任务
    ranTasks = runAllTasks(0); // This will run the minimum number of tasks
  }             
}

// NioEventLoop的unsafe为NioMessageUnsafe
processSelectedKeys(){
  int readyOps = k.readyOps();
  if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
    int ops = k.interestOps();
    ops &= ~SelectionKey.OP_CONNECT;
    k.interestOps(ops);

    unsafe.finishConnect();
  }

  if ((readyOps & SelectionKey.OP_WRITE) != 0) {
    ch.unsafe().forceFlush();
  }

  if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
  }
}

// runAllTasks
runAllTasks(){
  do {
    fetchedAll = fetchFromScheduledTaskQueue();
    if (runAllTasksFrom(taskQueue)) {
      ranAtLeastOne = true;
    }
  } while (!fetchedAll); 
}

protected final boolean runAllTasksFrom(Queue<Runnable> taskQueue) {
  Runnable task = pollTaskFrom(taskQueue);
  if (task == null) {
    return false;
  }
  for (;;) {
    // 在当前EventLoop所在的线程执行run方法
    // task.run();
    safeExecute(task);
    task = pollTaskFrom(taskQueue);
    if (task == null) {
      return true;
    }
  }
}
```

- 事件轮询流程图

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/eventloop_run.jpg)

##### SeverBootstrap初始化流程

> Netty组件初始化流程

在分析启动类初始化之前,我们先来查看下启动类的类图结构设计

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/boostrap_class.jpg)

根据类图结构,启动类比较简单,同时关于启动类的详细已经在先前组件分析过,现来看下启动类初始化Netty相关组件的源码

- EventLoopGroup添加到启动类

```java
// ServerBootsrtap.java
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
  super.group(parentGroup);
  if (this.childGroup != null) {
    throw new IllegalStateException("childGroup set already");
  }
  this.childGroup = ObjectUtil.checkNotNull(childGroup, "childGroup");
  return this;
}

// AbstractBootstrap.java
// AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel>>
public B group(EventLoopGroup group) {
  ObjectUtil.checkNotNull(group, "group");
  if (this.group != null) {
    throw new IllegalStateException("group set already");
  }
  this.group = group;
  return self();
}

// 通过上述可知并结合多Reactor模式可知
// ServerBootstrap持有childGroup,用于处理socketChannel的读写事件
// AbstractBootstrap持有parentGroup,用于处理serverChannel的accept事件
```

- 将服务端的Channel类对象以及服务端配置添加到启动类中

```java
// 入口函数
// 创建一个服务端的ServerChannel并指定其BACKLOG大小为100
bootstrap.channel(NioServerSocketChannel.class).option(ChannelOption.SO_BACKLOG, 100);

// 源代码
// AbstractBootstrap.java
// 通过传递的服务端Channel构造一个Channel创建工厂类,用于后续构建服务端的Channel
public B channel(Class<? extends C> channelClass) {
  return channelFactory(new ReflectiveChannelFactory<C>(
    ObjectUtil.checkNotNull(channelClass, "channelClass")
  ));
}

// 服务端Channel的配置存储到容器Map中
public <T> B option(ChannelOption<T> option, T value) {
  ObjectUtil.checkNotNull(option, "option");
  synchronized (options) {
    if (value == null) {
      options.remove(option);
    } else {
      options.put(option, value);
    }
  }
  return self();
}

```

- 将处理服务端的Handler添加到启动类中

```java
// 入口程序类
bootstrap.handler(new LoggingHandler(LogLevel.INFO));

// 源码
// AbstractBootstrap.java
// 当前的服务端channelHandler存在于AbstractBootstrap
public B handler(ChannelHandler handler) {
  this.handler = ObjectUtil.checkNotNull(handler, "handler");
  return self();
}
```

- 将处理客户端channelHandler添加到启动类

由于服务端程序启动服务端Channel之后会监听客户端SocketChannel的连接,一旦有连接进来这个时候就会注册绑定客户端的SocketChannel并监听读写事件,对此,对于使用Netty框架的服务端程序而言只需要关注处理读写事件的Handler即可,由于先前在组件源码分析中已经说明到,ServerChannel是作为客户端SocketChannel的语义层次上的父类,于是对于handler我们也可以理解childHandler是处理客户端读写事件的handler的处理器,其对应的源码如下:

```java
// 入口类程序
bootstrap.childHandler(new ChannelInitializer<SocketChannel>() { 
  // 保证每一个socket channel都会对应着一个自己的channel handler
  @Override
  public void initChannel(SocketChannel ch) throws Exception {
    ChannelPipeline p = ch.pipeline();
    if (sslCtx != null) {
      p.addLast(sslCtx.newHandler(ch.alloc()));
    }
    p.addLast(serverHandler);
  }
});

// 源码
// ServerBootstrap.java
// 将上述的childHandler绑定到ServerBootstrap,为ServerBootstrap所持有
public ServerBootstrap childHandler(ChannelHandler childHandler) {
  this.childHandler = ObjectUtil.checkNotNull(childHandler, "childHandler");
  return this;
}
```

- Bootstrap初始化

基于上述添加EventLoopGroup以及ChannelHandler,我们再来看下Bootsrtap启动类初始化的时候做了什么事情.

```java
// 入口程序
new ServerBootstrap();

// 在上述执行初始化流程中,会在内部完成以下组件的初始化
private final Map<ChannelOption<?>, Object> childOptions = new LinkedHashMap<ChannelOption<?>, Object>();
private final Map<AttributeKey<?>, Object> childAttrs = new ConcurrentHashMap<AttributeKey<?>, Object>();
private final ServerBootstrapConfig config = new ServerBootstrapConfig(this);
```

结合之前组件分析,我们知道Channel是存在语义上的层次关系,对此,我们关注ServerBootstrap与ServerBootstrapConfig, AbstractBootstrap与AbstractBootstrapConfig之间分别获取channel信息的区分,其类图组件如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/bootstrap_config_class.jpg)

通过上述的类图可以知道,ServerBootsrtap与SocketChannel进行关联,AbstractServerBootstrap与ServerSocketChannel进行关联,对于channel,ServerSocketChannel与SocketChannel是层次上的父子关系,对于Bootsrap类抑或是Config类,均通过子类获取与SocketChannel相关的信息,通过父类获取与ServerSocketChannel相关信息,层次划分明确,现将Bootstrap构造初始化操作事件流程绘制如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/bootstrap_flow.jpg)

我们知道在Netty框架在处理服务端与客户端的事件是划分层次的,在语义层次上,服务端属于“父类”,客户端属于“子类”,两者之间的事件所依赖的组件也在语义上划分层次,对此,结合上述对EventLoopGroup与EventLoop的源码分析,现将启动类Bootstrap,EventLoopGroup,EventLoop,Channel以及Thread之间的关联示意图绘制如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/bootstrap_group_thread_channel.jpg)

##### 启动类绑定端口事件流程

入口程序源码分析如下:

```java
// 入口程序
bootsrtap.bind(PORT);

// AbstractBootstrap.java
// 通过类名称可知是创建服务端的Channel并注册Channel事件实现对客户端Channel连接的监听
public ChannelFuture bind(int inetPort) {
  return bind(new InetSocketAddress(inetPort));
}

// bind包括: 创建channel -> 初始化channel -> 注册channel -> channel绑定端口操作
doBind(final SocketAddress localAddress){
  // 由于注册绑定流程复杂,这里将绑定注册流程划分出来,摘录核心方法,Netty框架中使用EventLoop来处理每个channel事件,存在多线程异步执行的情况.对于异步返回的结果ChannelFuture已在Netty组件源码分析说明到,这里不再详述
// 初始化并注册服务端的channel
initAndRegister();

//...
  
//如果注册成功,执行服务端channel的绑定操作
doBind0(regFuture, channel, localAddress, promise);
}
```

> 服务端channel初始化与注册事件

- 创建服务端channel的流程

```java
// 源代码
// 使用channelFactory创建NioServerSocketChannel实例
channel = channelFactory.newChannel();
```

`NioServerSocketChannel`类图结构设计如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/nioserversocketchannel_class.jpg)

这个时候再来看创建`NioServerSocketChannel`实例都做了哪些事情

```java
public NioServerSocketChannel() {
  // 创建java的nio下的ServerSocketChannel并传递到当前的NioServerSocketChannel构造器中
  this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}

public NioServerSocketChannel(ServerSocketChannel channel) {
  // 服务端监听Accept事件并保存,后续在进行注册的时候将会使用到OP_ACCEPT
  	// 1.设置channel的父类,如果当前为服务端的channel则为null
  	// 2.创建channelId
    // 3.创建Nio的Unsafe类
    // 4.创建channel的责任链pipeline,同时每个pipeline都会创建一个双端链表连接上下文对象
  super(null, channel, SelectionKey.OP_ACCEPT);
  // 1. 为当前的channel创建接收数据的ByteBuff分配器,即AdaptiveRecvByteBufAllocator,该分配器默认从1024kb开始创建缓冲区分配数据,最小为64kb,最多不超过65536kb
  // 2. 保存java对象的ServerSocketChannel
  config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

根据上述源码可知,创建Channel时会将与操作Channel相关的组件也一起完成初始化操作,即创建操作缓冲区数据的Unsafe以及对缓冲区数据进行读写存储的ByteBuff分配器.

- 服务端channel的初始化流程

```java
// 基于上述完成channel的创建,接下来对channel进行初始化操作
// 对应的源码
// ServerBootstrap.java
	// 1. 为当前的channel设置option以及attributes
  // 2. 获取当前channel的责任链,为当前的责任添加初始化handler处理器
init(channel);

// init下初始化handler核心代码
p.addLast(new ChannelInitializer<Channel>() {
  @Override
  public void initChannel(final Channel ch) {
    final ChannelPipeline pipeline = ch.pipeline();
    // 获取服务端channel的handler处理类
    ChannelHandler handler = config.handler();
    if (handler != null) {
      pipeline.addLast(handler);
    }
		// 在channel所在的eventloop创建一个线程来执行任务
    ch.eventLoop().execute(new Runnable() {
      @Override
      public void run() {
        // 在任务下为服务端的channel添加Acceptor处理器负责处理客户端channel连接进来的事件完成处理
        pipeline.addLast(new ServerBootstrapAcceptor(
          ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
      }
    });
  }
});
```

- 服务端channel的注册流程

```java
// 源码
// AbstractBootstrap.java
// 获取boss NioEventLoopGroup,将channel注册到当前的group下
ChannelFuture regFuture = config().group().register(channel);

// 根据NioEventLoopGroup的继承类图,可知register方法是在MultithreadEventLoopGroup下
// MultithreadEventLoopGroup.java
 public ChannelFuture register(Channel channel) {
   // 选举一个EventLoop来注册channel
   // 在初始化Group操作的时候已经完成选择器的初始化操作,这里调用选择器来选择一个EventLoop
   // 这里调用EventLoop的注册方法,在上述入口中使用NioEventLoop可知使用的register方法为SingleThreadEventLoop类下的方法,最终调用AbstractChannel下的register方法
   // 方法调用走向如下:
   // MultithreadEventLoopGroup.regitser() -> SingleThreadEventLoop.regitser() -> promise.channel().unsafe().register() -> unsafe(NioMessageUnsafe).regitser() -> AbstractNioUnsafe.regitser() -> AbstractUnsafe.register() -> AbstractUnsafe.register0()
   return next().register(channel);
 }

//AbstractChannel.java下的AbstractUnsafe
register0(promise);

// 上述注册方法的核心步骤:
// 1. 将channel注册到复用器selector上
// 2. 注册完成之后唤醒回调责任链下所有先前已加入的channelHandler类下的handlerAdd方法
// 3. 注册完成之后将结果设置在promise中
// 4. 将注册结果传递到责任链pipeline中,并执行回调channelHandler(ChannelInboundHandler)类下的channelRegistered方法,链式回调执行
// 5. 如果channel为active状态,则继续传播结果事件到channelHandler(ChannelInboundHandler)类下的channelActive方法,链式回调执行
// 6. 5步骤是在第一次进行注册的时候会执行(表示channel已经打开),如果已经注册过,那么校验会自动开始数据读取操作,客户端channel注册读取OP_READ操作, 对应服务端的Channel而言就是监听客户端socket的连接ACCEPT事件
```

在基于上述的线程执行任务细节基础之上,将服务端的初始化并注册流程示意图流程绘制如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/channel_create_init_register.jpg)

> 执行端口绑定与监听操作

- 端口绑定源码分析

```java
// 上述channel注册成功之后,这个时候在上面流程只会触发Active事件,这个时候没有绑定端口没有触发监听事件
// AbstractBootstrap.java
doBind0(regFuture, channel, localAddress, promise);

private static void doBind0(
  // channel所在的eventloop线程执行任务
  channel.eventLoop().execute(new Runnable() {
    @Override
    public void run() {
      // 注册成功将channel进行绑定操作
      if (regFuture.isSuccess()) {
        // 
        channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
      } else {
        promise.setFailure(regFuture.cause());
      }
    }
  });
}
  
  // AbstractChannel.java
  @Override
  public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
  }
  
  // DefaultChannelPipeline.java
  @Override
  public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    // 在链表尾部添加绑定操作
    return tail.bind(localAddress, promise);
  }
  
  // AbstractChannelHandlerContext.java
  @Override
  public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(localAddress, "localAddress");
    if (isNotValidPromise(promise, false)) {
      // cancelled
      return promise;
    }
// 搜索outboundContext上下文
    final AbstractChannelHandlerContext next = findContextOutbound(MASK_BIND);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
      // 执行责任链pipeline 出站事件,从链表尾部开始搜索,因而最后的context是headContext
      // 执行headContext下的invokeBind方法,该方法还是属于当前类,对此查看下文
      next.invokeBind(localAddress, promise);
    } else {
      safeExecute(executor, new Runnable() {
        @Override
        public void run() {
          next.invokeBind(localAddress, promise);
        }
      }, promise, null, false);
    }
    return promise;
  }
  
  // AbstractChannelHandlerContext.java
  private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
    if (invokeHandler()) {
      try {
        ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
      } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
      }
    } else {
      bind(localAddress, promise);
    }
  }
  
  // headContext的绑定方法
   @Override
  public void bind(
    ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
    unsafe.bind(localAddress, promise);
  }
  
  // unsafe为NioMessageUnsafe,执行该类下的bind方法(AbstractUnsafe.java中定义)
  // 最后再执行channel下的doBind(localAddress);方法,即NioServerSocketChannel下的方法
  @SuppressJava6Requirement(reason = "Usage guarded by java version check")
  @Override
  protected void doBind(SocketAddress localAddress) throws Exception {
    // 可以看到实现了端口的绑定操作
    if (PlatformDependent.javaVersion() >= 7) {
      javaChannel().bind(localAddress, config.getBacklog());
    } else {
      javaChannel().socket().bind(localAddress, config.getBacklog());
    }
  }
```

 对此,基于上述源码的分析,我们绘制服务端channel的端口绑定流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/channel_bind_flow.jpg)

##### 责任链创建执行流程

- 基于ChannelInitial来初始化pipeline的handler

```java
// 添加handler的入口程序
// 这里主要是捡重点说明
ChannelPipeline p = channel.pipeline();
p.addLast(new ChannelInitializer(){
  // 在源码中有如下说明:
  // 一旦channel注册将调用当前initChannel方法，方法执行完成之后将实例会从ChannelPipeline中移除
  void initChannel(C ch) throws Exception{
     final ChannelPipeline pipeline = ch.pipeline();
     pipeline.addLast(new ServerHandler());
     
     // 使用EventLoop添加
    ch.eventLoop().execute(new Runnable() {
      @Override
      public void run() {
        pipeline.addLast(new ServerBootstrapAcceptor(
          ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
      }
    });
  }
});

// DefaultChannelPipeline.java
 @Override
public final ChannelPipeline addLast(ChannelHandler... handlers) {
  return addLast(null, handlers);
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
  ObjectUtil.checkNotNull(handlers, "handlers");

  for (ChannelHandler h: handlers) {
    if (h == null) {
      break;
    }
    addLast(executor, null, h);
  }

  return this;
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
  final AbstractChannelHandlerContext newCtx;
  synchronized (this) {
    checkMultiplicity(handler);
		// 创建一个上下文对象,默认为DefaultChannelHandlerContext
    newCtx = newContext(group, filterName(name, handler), handler);
		
    // head -> handler1 --- handler - tail 
    addLast0(newCtx);

    if (!registered) {
      // channel还没有注册,设置当前handler处于等待状态
      newCtx.setAddPending();
      // 将其添加到等待链表的尾部中
      callHandlerCallbackLater(newCtx, true);
      return this;
    }

    EventExecutor executor = newCtx.executor();
    if (!executor.inEventLoop()) {
      callHandlerAddedInEventLoop(newCtx, executor);
      return this;
    }
  }
  callHandlerAdded0(newCtx);
  return this;
}

// 最终会当channel完成注册的时候会调用handlerAdd方法,而ChannelInitial的handlerAdd方法如下:
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
  if (ctx.channel().isRegistered()) {
    // 最终会调用initChannel方法
    if (initChannel(ctx)) {
      // 完成channel的初始化链会将当前实例移除
      removeState(ctx);
    }
  }
}

private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
  if (initMap.add(ctx)) { // Guard against re-entrance.
    try {
      initChannel((C) ctx.channel());
    } catch (Throwable cause) {
      exceptionCaught(ctx, cause);
    } finally {
      // 最后会在pipeline链中删除当前实例
      ChannelPipeline pipeline = ctx.pipeline();
      if (pipeline.context(this) != null) {
         pipeline.remove(this);
      }
    }
    return true;
  }
  return false;
}
```

- 责任链添加处理器handler流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/pipeline_add_flow.jpg)

- channel事件与责任链生命周期联系

通过上述可知,channel在注册前后过程中的pipeline存储的handler结构为:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/channel_pipeline_flow.jpg)

##### 服务端接收客户端连接的事件流程

根据EventLoop的事件轮询流程可知,服务端监听的事件变化以及事件转发处理都在EventLoop.run方法中,对此,我们详细看下Netty的服务端是如何接收并处理客户端的连接事件以及对应的事件流程是如何的.

- Netty框架下的核心事件轮询run方法源代码

```java
// 上述已经贴有源码,这里我们更关注细节问题,仅列出run方法的核心源码
// NioServerSocketChannel.java
run(){
  // 服务端不断轮询监听事件
  for (;;) {
    
    // 执行select操作
    if (!hasTasks()) {
      strategy = select(curDeadlineNanos);
    }
    
    // 处理就绪事件
    processSelectedKeys();
    
    // 处理任务队列中的任务
    ranTasks = runAllTasks();
  }
}

// 既然关注ACCEPT事件,这个时候我们需要知道服务端Channel在创建注册并绑定的时候初始化handler并将Acceptor添加到handler中,对此我们追踪下bind方法,最后查阅代码到init方法,其代码如下:
void init(Channel channel){
  // 在先前分析可知,这里已经完成了channel的创建,且此时channel为NioServerSocketChannel事件
  ChannelPipeline p = channel.pipeline();
  p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) {
      final ChannelPipeline pipeline = ch.pipeline();
      // 获取服务端channel的handler处理类
      ChannelHandler handler = config.handler();
      if (handler != null) {
        pipeline.addLast(handler);
      }

      ch.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
          pipeline.addLast(new ServerBootstrapAcceptor(
            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
        }
      });
    }
  });
}
```

根据上述源代码,服务端监听连接事件的流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/channel_listen_flow.jpg)

其中关于创建和注册流程基本和上述服务端channel一致,这里绘制的时候直接简化,没有详细绘制出来.

##### 服务端处理客户端channel的读写事件流程

- 读写事件源代码

```java
// 在这里关注的读写事件是NioSocketChannel
void processSelectedKey(){
  if ((readyOps & SelectionKey.OP_WRITE) != 0) {
     ch.unsafe().forceFlush();
  }

  if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
     unsafe.read();
  }
}

// NioSocketChannel使用NioByteUnsafe来实现读写
// 先读后写
// 读取流程
// AbstractNioByteChannel.NioByteUnsafe
read(){
  try{
      do{
      // 从socket中读取数据
      doReadBytes(byteBuff);

      // 传播数据
      pipeline.fireChannelRead(byteBuf);
    }while(continueReading())
    	pipeline.fireChannelReadComplete();
    	if(close){
      	closeOnRead(pipeline);
    	}
  }catch (Throwable t) {
    //传播userEventTriggered
    handleReadException(pipeline, byteBuf, t, close, allocHandle);
  } finally {
    if (!readPending && !config.isAutoRead()) {
      // 取消读取操作
      removeReadOp();
    }
  }
}

// 写出流程
// 写出操作的触发点是在某个handler下的channelRead方法下手动执行write或者writeAndFlush方法
// handler 通过addLast方法添加,默认上下文对象为DefaultChannelHandlerContext
void handlerRead(AbstractHandlerContext ctx, Object msg){
  // 执行写操作的流程说明责任链执行当前入站事件handler已经是最后一个,从当前handler的上下文对象开始执行出站事件
  ctx.writeAndFlush(msg);
}
// 执行链为:
// AbstractChannelHandlerContext.writeAndFlush() - invokeWriteAndFlush -> handler.write()  - head.write() - AbstractUnsafe.write() - filterOutboundMessage() - AbstractNioByteChannel.filterOutboundMessage()[创建一个堆外内存] - addMessage() -> incrementPendingOutboundBytes() -> [newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()] -> setUnwritable -> fireChannelWritabilityChanged -> head.fireChannelWritabilityChanged -> handler.fireChannelWritabilityChanged -> tail.fireChannelWritabilityChanged 
// -> invokeFlush0 -> handler.flush()  -> head.flush() -> AbstractUnsafe.flush() -> addFlush()  -> flush0() ->. NioSocketChannel.doWrite() -> ch.write(nioBuffers, 0, nioBufferCnt) -> incompleteWrite() -> setOpWrite()[注册写事件] -> 
```

- 读事件流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/channel_read.jpg)

- 写事件流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/channel_write.jpg)

##### ServerChannel/ChannelHandler/ChannelPipeline事件生命周期

结合之前组件的源码以及上述的事件流程分析,关于channel事件与pipeline责任链回调执行生命周期流程总结如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event/channel_pipeline_life.jpg)