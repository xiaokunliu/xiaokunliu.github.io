---
title: 深入Netty事件流程分析(下)
category: 网络IO编程
date: 2020-04-24 09:39:59
tags: 网络IO编程
---

<!-- more -->

![title2](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/title2.jpg)

继上篇Netty事件分析,本文主要讲述Netty的责任链流程,Channel与Handler生命周期以及网络IO事件分析流程,最后对Netty事件流程进行一个总结梳理.

#### pipeline责任链流程分析

##### 责任链创建流程

- 入口程序

```java
// 责任链的创建是在Channel的初始化的时候进行的
// AbstractChannel.java
protected AbstractChannel(Channel parent) {
  // 如果当前为服务端的channel，则parent=null
  this.parent = parent;
  // 创建channelId
  id = newId();
  // 使用NioServerSocketChannel父类的AbstractNioMessageChannel下的NioMessageUnsafe
  // 使用NioSocketChannel父类的AbstractNioByteChannel下的AbstractNioUnsafe
  unsafe = newUnsafe();
  // 创建channel的责任链,DefaultChannelPipeline
  pipeline = newChannelPipeline();
}

// 创建默认的责任链实例对象
 protected DefaultChannelPipeline newChannelPipeline() {
   return new DefaultChannelPipeline(this);
 }
```

- Channel的类关系图

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/channel_class.jpg)

通过源码与类图可知,不论是服务端Channel的创建还是客户端Channel的创建,其默认的pipeline均通过AbstractChannel的构造方法来初始化一个责任链实例,且默认为`DefaultChannelPipeline`,接下来我们来看下责任链创建的流程

- 创建源码分析

```java
// DefaultChannelPipeline.java
protected DefaultChannelPipeline(Channel channel) {
  // 当前的责任链保存对应的channel信息
  this.channel = ObjectUtil.checkNotNull(channel, "channel");
  
  // channel在整个责任链处理正常返回的成功结果对象Future
  succeededFuture = new SucceededChannelFuture(channel, null);
  
  // 对channel在整个责任链处理添加监听,负责异常的捕获
  voidPromise =  new VoidChannelPromise(channel, true);

 // 创建上下文对象，每个上下文对象都包含当前pipeline实例对象
  tail = new TailContext(this);
  head = new HeadContext(this);

  // 在逻辑结构上通过双端链表的方式存储上文对象
  head.next = tail;
  tail.prev = head;
}


// 对于HeadContext与TailContext特殊上下文的创建
// 上下文创建
// AbstractChannelHandlerContext.java
HeadContext(DefaultChannelPipeline pipeline) {
  super(pipeline, null, HEAD_NAME, HeadContext.class);
  
  // 根据channel的类型来区分,服务端为NioMessageUnsafe
  // 客户端为NioSocketChannelUnsafe
  unsafe = pipeline.channel().unsafe();

  // 保证handler调用方法的顺序，可以理解为handler执行的生命周期，通过状态机来控制生命周期
  setAddComplete();
}

 TailContext(DefaultChannelPipeline pipeline) {
   super(pipeline, null, TAIL_NAME, TailContext.class);
   setAddComplete();
 }

// 对于Netty责任链使用的EventLoop是属于有序的执行器,为了保证handlerAdd与handlerRemove的执行存在先后关系,通过以下的状态机来控制,即handler方法执行的生命周期保证,如果EventLoop不保证有序的话,只需要通过ADD_COMPLETE或者REMOVE_COMPLETE来告知方法是否被调用即可
// 初始化状态,创建责任链的时候上下文的handlerState默认为初始化,表示handlerAdd/handlerRemove均没有被调用
private static final int INIT = 0;
// handlerAdded即将被调用(实际还没有调用,准备就绪,可以被调用),一般是在不需要保证有序的情况下
private static final int ADD_PENDING = 1;
// handlerAdd已经被调用
private static final int ADD_COMPLETE = 2;
// handlerRemoved已经被调用
private static final int REMOVE_COMPLETE = 3;
```

相比其他的handler的上下文创建,HeadContext与TailContext在构建方法中多了一个动作`setAddComplete()`,主要目的是由于双端链表的head与tail都是在初始化channel的时候构建而不是通过addLast或者是addFirst的方式构建,为了保证handler方法执行的有序性,于是在构建上下文的时候多添加一个步骤,接下来我们可以看到普通的handler添加方式,会在addLast中也调用上述`setAddComplete()`相应的方法执行.

- 创建流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/pipeline_init.jpg)

##### 添加handler流程

- 程序入口代码

```java
// 获取当前的责任链pipeline
Pipeline pipeline = channel.pipeline();
// 添加handler,这里以特殊的initHandler添加为准来说明,摘录启动类的init方法
pipeline.addLast(new ChannelInitializer<Channel>() {
  // ChannelInitializer是一个特殊的入站事件，添加到channel中的pipeline中
  //  一旦channel已经注册到EventLoop中就会触发执行
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
```

- `addLast()`源代码

```java
// DefaultChannelPipeline.java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
  final AbstractChannelHandlerContext newCtx;
  synchronized (this) {
    checkMultiplicity(handler);
		// 创建一个上下文对象,默认为DefaultChannelHandlerContext
    newCtx = newContext(group, filterName(name, handler), handler);
		
    // 将上下文对象添加到责任链尾部
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
      // 当前线程持有的eventloop非独占,需要将其添加到任务队列中
      callHandlerAddedInEventLoop(newCtx, executor);
      return this;
    }
  }
  // 核心方法
  callHandlerAdded0(newCtx);
  return this;
}

// callHandlerAdded0下会执行方法
void callHandlerAdded0(){
  ctx.callHandlerAdded();
}

// AbstractChannelContext.java
final void callHandlerAdded() throws Exception {
  // 可以看到在执行handlerAdd方法之前会调用setAddComplete方法
  if (setAddComplete()) {
    handler().handlerAdded(this);
  }
}

// 最终会当channel完成注册的时候会调用handlerAdd方法,而ChannelInitial的handlerAdd方法如下:
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
  if (ctx.channel().isRegistered()) {
    // 调用initChannel方法
    if (initChannel(ctx)) {
      // 完成channel的初始化链会将当前实例移除
      removeState(ctx);
    }
  }
}

private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
  if (initMap.add(ctx)) { // Guard against re-entrance.
    try {
      // 模板(钩子hook)方法,也就是我们上述的入口程序添加initHandler重载的initChannel方法
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

- 添加handler的流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/pipeline_add_flow.jpg)

通过上述流程图可知,初始化initHandler在channel注册之后责任链pipeline会将initHandler从中移除并将用户添加的handler添加到责)

##### 责任链销毁流程

在查看责任链销毁的源代码之前,我们是否应该先思考是什么样的动作行为会销毁pipeline责任链,如果想不出来,我们换另一个思路,责任链是什么时候创建的,根据上述的分析,责任链pipeline是创建channel的时候创建的,那么我们是否可以推测销毁时机是不是在channel销毁的时候对应的pipeline也将会销毁呢?那么channel是在什么时候销毁的呢,我们可以考虑一个已熟知的数据库连接connection创建与销毁,什么时候需要销毁connection,自然是调用`close()`方法的时候,这个时候connection要么等待被JVM回收要么就是存放到回收资源池中,对此关于责任链的销毁分析如下

- 入口程序

```java
// 服务端channel销毁,也就是服务端channel调用close()关闭服务
// 对于客户端channel,自然是断开与服务端的连接
// channel的关闭是属于事件触发,于是我们直接定位到事件轮询器下的方法processSelectedKey,该方法负责处理就绪事件
// 对于NIO的api,每个socket的就绪事件都存储在SelectionKey中,如果channel销毁,当前的SelectionKey也将会在销毁之前取消事件监听
// NioEventLoop.javas
void processSelectedKey(){
  if (!k.isValid()) {
    // ...
    unsafe.close(unsafe.voidPromise());
  }
  try{
    
  }catch (CancelledKeyException ignored) {
    unsafe.close(unsafe.voidPromise());
  }
}
```

- 源码分析

```java
// 通过代码定位最终会执行以下代码块
// AbstractChannel.java
try {
  // 调用java饿的socket进行关闭
  doClose0(promise);
} finally {
  // Call invokeLater so closeAndDeregister is executed in the EventLoop again!
  invokeLater(new Runnable() {
    @Override
    public void run() {
      if (outboundBuffer != null) {
        // Fail all the queued messages
        outboundBuffer.failFlushed(cause, notify);
        outboundBuffer.close(closeCause);
      }
      // 将channel事件传播到责任链中
      fireChannelInactiveAndDeregister(wasActive);
    }
  });
}
```

- 销毁流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/pipeline_destroy.jpg)

#### IO事件流程分析

##### 监听连接事件

基于上一篇的Netty事件流程分析中的事件轮询说明可知,服务端监听的事件变化以及事件转发处理都在EventLoop.run方法中,对此,我们详细看下Netty的服务端是如何接收并处理客户端的连接事件以及对应的事件流程是如何的.

- Netty框架下的核心事件轮询run方法源代码

```java
// 仅贴出部分核心代码
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

- ServerBootstrapAcceptor监听连接的源码

在上一篇的事件分析中,我们对服务端的一个绑定事件进行了分析(包括服务端channel的创建/初始化与注册,客户端的channel基本与服务端一致,这里也不再详细说明),最终的监听连接的事件将会调用`unsafe.read()`方法并且会将事件通过责任链pipeline传播到channelRead方法下,对此,我们关注Acceptor处理连接可以通过查看handler实现的`channelRead()`方法即可.

```java
// ServerBootstrapAcceptor.java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
  final Channel child = (Channel) msg;
  // 将处理客户端channel的handler添加到责任链pipeline中
  child.pipeline().addLast(childHandler);

  setChannelOptions(child, childOptions, logger);
  setAttributes(child, childAttrs);

  try {
    // 客户端channel注册到EventLoop,注册流程与之前服务端注册流程基本一致,这里不再详述
    childGroup.register(child).addListener(new ChannelFutureListener() {
      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
        if (!future.isSuccess()) {
          forceClose(child, future.cause());
        }
      }
    });
  } catch (Throwable t) {
    forceClose(child, t);
  }
}
```

- 监听连接的事件流程

通过上述源码以及责任链添加handler的流程可知,当服务端channel完成之后,pipeline链将会是`head -> handlers -> ServerBootstrapAcceptor -> tail`,因而我们根据现有的线索以及上述源码,对监听连接事件流程绘制如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/channel_listen_flow.jpg)

##### 请求读取事件

- 入口程序

```java
// NioEventLoop.run()方法
// 在这里关注的读写事件是NioSocketChannel
void processSelectedKey(){
  if ((readyOps & SelectionKey.OP_WRITE) != 0) {
     ch.unsafe().forceFlush();
  }

  // 对于处理请求,我们需要关注NioSocketChannel处理读取事件流程
  if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
     unsafe.read();
  }
}
```

- NioSocketChannel的类图组件

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/nio_socket_channel_class.jpg)

- 读取事件实现

```java
// 根据上述代码以及类图可知,NioSocketChannel使用初始化safe实现类为NioSocketChannelUnsafe
// 于是查看其的源码实现,但是方法NioSocketChannelUnsafe并没有read方法,而是在NioByteUnsafe类中,因而找到对应的read方法,摘录部分核心代码如下:

void read() {
  try{
      do{
      // 从socket中读取数据
      doReadBytes(byteBuff);

      // 传播读取事件到责任链中
      pipeline.fireChannelRead(byteBuf);
    }while(continueReading())
      
      // 传播读取完成事件到责任链中
    	pipeline.fireChannelReadComplete();
    	if(close){
      	closeOnRead(pipeline);
    	}
  }catch (Throwable t) {
    //传播事件异常以及userEventTriggered
    handleReadException(pipeline, byteBuf, t, close, allocHandle);
  } finally {
    if (!readPending && !config.isAutoRead()) {
      // 取消读取操作
      removeReadOp();
    }
  }
}

```

- 请求读取流程示意图

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/channel_read.jpg)

##### 数据写出事件

- 入口程序代码

对于写出操作,主要在开发者实现Channel的channelRead或者channelReadCompleted方法下手动调用方法执行写出操作,即入口程序代码如下:

```java
// 写出操作的触发点是在某个handler下的channelRead方法下手动执行write或者writeAndFlush方法
void handlerRead(AbstractHandlerContext ctx, Object msg){
  // 执行写操作的流程说明责任链执行当前入站事件handler已经是最后一个,从当前handler的上下文对象开始执行出站事件
  ctx.writeAndFlush(msg);
}
```

可以看到,写出操作是调用上下文对象的写出操作,基于这个线索,先来查看上下文对象的类图设计关系.

- 上下文对象类设计图

在添加handler的时候,我们通过源码看到addLast方法内部的实现中,默认上下文对象为DefaultChannelHandlerContext,于是对应的类图关系如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/context_class.jpg)

通过上述的类图可知,上下文对象有一个父类`AbstractChannelHandlerContext`来实现通用的方法,同时上下文对象具备出入站事件,因此我们可以在handler中对接收到的上下文对象ctx手动处理出站或入站事件的传播,对此当我们调用`ctx.writeAndFlush()`方法的时候也将会触发对应的一个handler触发事件(通过源码分析是属于出站事件)

- 源码分析

```java
// AbstractChannelHandlerContext.java
void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
  if (invokeHandler()) {
    // 负责将写出的数据存储到OutboundBuffer缓冲区
    invokeWrite0(msg, promise);
    // 执行刷新操作
    invokeFlush0();
  } else {
    writeAndFlush(msg, promise);
  }
}

// 通过责任链传播写出事件,
private void invokeWrite0(Object msg, ChannelPromise promise) {
  try {
    ((ChannelOutboundHandler) handler()).write(this, msg, promise);
  } catch (Throwable t) {
    notifyOutboundHandlerException(t, promise);
  }
}

// 通过责任链传播刷新事件
private void invokeFlush0() {
  try {
    ((ChannelOutboundHandler) handler()).flush(this);
  } catch (Throwable t) {
    notifyHandlerException(t);
  }
}

// 根据责任链执行流程可知,最终会执行headContext下write以及flush的方法
// HeadContext.java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
  unsafe.write(msg, promise);
}

@Override
public void flush(ChannelHandlerContext ctx) {
  unsafe.flush();
}

// 最后会调用AbstractUnsafe的write以及flush方法(这里就不贴出代码.直接查看流程图)
```

- 写出事件执行流程图

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/channel_write.jpg)

#### Channel与Handler生命周期小结

##### Channel的生命周期

结合上一篇的事件流程分析,channel主要经历创建 - 初始化 - 注册 - 事件接收处理 - 销毁过程,其经历的流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/channel_flow.jpg)

- Channel生命周期

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/channel_lifetime.jpg)

##### Handler的生命周期

- handler类设计图

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/handler_class.jpg)

- Handler生命周期

根据上述类图可知,ChannelHandler定义了handlerAdd以及handlerRemoved并且结合上述责任链的流程分析可知,handler是通过上下文对象来传播事件并回调方法,并且上下文对象通过handlerState以及channel事件流程来保证上述方法执行的先后顺序,从而保证handler的执行生命周期

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/handler_lifetime.jpg)

##### Handler方法回调与生命周期联系

最后,根据前面分析的事件流程以及上述的Channel生命周期,对Channel与handler执行回调方法作一个小结,如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/netty/event2/channle_life.jpg)

