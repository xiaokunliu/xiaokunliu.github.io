---
title: IO事件驱动设计实现
category: 网络IO编程
date: 2020-04-23 17:56:44
tags: 网络IO编程
---

<!-- more -->

### 事件驱动架构EDA
#### EDA组件
- 事件源/发起器(event emitters): 负责轮询检测事件状态的变化
- 解复用器(Demultiplexer): 等待从事件源上获取就绪事件的集合,并将就绪事件通过转发器分发给响应就绪事件的处理器进行回调处理
- 事件处理引擎(event handlers): 响应就绪事件发生的处理程序,由开发人员在应用程序上进行定义并针对就绪事件发生的状态进行注册绑定
- 事件队列(event queue): 或者称为事件通道,可以理解为注册绑定对应的事件存储的位置,一旦就绪事件发生,解复用器就会从事件队列中检测并返回对应的就绪事件

#### EDA组件运作与设计
##### 简要流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121142540.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

##### AWT完整事件流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121157805.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
对于AWT事件的设计需要有客户端,事件源(发生器),事件通道,事件处理器以及事件对象组件一起配合完成完整的点击事件流程,基于监听者模式的设计思路如下:

- 客户端

```java
// 需要注册和绑定处理器
class Client{
 
  public static void main(String[] args){
    // 获取事件源
   	Button button = new Button();
    // 绑定事件源
    button.bind("click", new ClickHandler());
    // 执行点击事件
    button.click();
  }
}
```

- 事件处理器定义

```java
interface ActionHandler{
    void handler(ActionEvent e);
}

class ClickListener implements ActionHandler {
  	// 类似于上述的handler,用于处理点击事件的响应
    @Override
    public void handler(ActionEvent e) {
      
    }
}
```

- 事件源定义

```java
// 这个时候Button只是一个普通class,我们知道事件源需要具备检测监听的行为,对此继承监听者的功能
class Button extends ActionListener {
  
  // 这个时候click只是一个普通方法  
  void click(){
    // 在上述事件流程可知,要让click事件被传播,需要借助事件通道进行传播执行回调,此时触发监听传播
  	 this.trigger("click");
  }
  
  // 事件源还需要具备绑定具体的动作行为
  void bind(String type, ActionHandler handler){
    // 在上述的事件流程中,将其绑定到事件通道中
    // 存储哪个事件源 哪个事件行为类型, 对应的处理动作
    this.store(this, type, handler);
  }
}
```

- 事件通道组件

```java
// 作为通道组件
class ActionListener {
  // 定义map存储事件类型以及对应的事件,作为存储事件的通道
  private Map<String, ActionEvent> events = new HashMap<>();
  
  public void store(Object source, String type, ActionHandler hander){
    events.put(type, new ActionEvent(source, handler, type));
  }
  
  public void trigger(String type){
    // 从事件通道中搜索事件,并回调执行事件
    ActionEvent event = map.get(type);
    Object target = event.getTarget();
    Method callback = target.getClass(),getDeclaredMethod("handler", ActionEvent.class);
    callback.invoke(target, event);
  }
}
```

- 事件组件

```java
// 通过上述的事件通道组件可知
class ActionEvent{
   // 定义事件源
  private Object source;
  // 定义处理事件的目标处理器
  private Object target;
  // 定义事件的类型
  private String eventType;
  // ..others such as status, timestamp, id etc....
}
```

最后,关于编程设计的一个思考,就是在推导设计的时候,可以尝试借用TDD的方式进行编程设计,先预先定义自己想要实现的效果,一步步从最简单的目标效果思考逼近最终的设计。
好了言归正传,通过上述的一个设计思路,我们接下来要思考如何实现一个IO事件驱动设计呢?对此,先从简单的网络NIO事件处理流程开始.

##### 网络NIO事件处理流程

对于web服务设计,主要处理服务端监听连接并接收客户端连接事件以及客户端发起服务端读取事件,这里主要以服务端的设计为准.

- 服务端监听连接事件流程 -- Accept事件流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121227737.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

上述的Accept监听是对服务端的ServerSocket进行连接事件的监听.

- 服务端读取事件流程 -- 响应IO事件流程

在先前的Unix的IO模型中,真正进行IO操作的是调用`recvfrom`方法产生阻塞,对于非阻塞IO是当内核真正接收到可操作的IO事件时候才发起`recvfrom`方法,对此这里的事件是指对客户端socket的读取事件进行监听,在上述建立连接监听的基础上,事件读取流程如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121253498.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

上述是一个完整的IO事件连接与读取流程,可以看出,最左边的一个是事件处理器负责处理事件状态发生变化的一个响应,而右边的一侧则是属于处理网络IO事件的监听,此时所有的资源都阻塞该非阻塞IO的API调用,通过接收到就绪事件的通知由内核发起唤醒回调并返回就绪事件集合,然后传输给响应事件的处理器,于是也就有了Reactor反应器设计.

#### Reactor设计原理
##### Reactor运作流程

通过上述的NIO事件流程可知,对于web服务端而言,一个是我们需要关注IO轮询就绪事件以及获取就绪事件集合的操作,另一个是关注响应IO就绪事件的处理,主要包含连接的响应处理以及读取请求处理的响应处理,可以从宏观上,引入中间组件分别处理上述事件的轮询监听以及事件响应操作,在Reactor设计中,Reactor组件负责实现事件的轮询监听操作,Handler负责就绪事件的响应操作,对此,一个Reactor模式的简要事件流程如下:
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020041012131717.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

对此,一个Reactor模式的web服务设计实现需要两个核心组件,即:Handler以及Reactor:
- Handler则需要拆分伪`RequestHandler`以及`Acceptor`两个处理器,
- Reactor组件中,参与的工作有注册绑定操作,IO事件的监测以及就绪事件的转发操作,同时也可以看到Reactor与系统内核之间都通过socket事件源来感知到事件状态的变化,是系统内核与Reactor之间通信的一个重要渠道,即网络设备接收到连接或者请求操作唤醒socket然后异步回调让Reactor获取CPU执行权,这个时候Reactor获取到socket事件为就绪事件.

##### Reactor组件具体实现

现理清Reactor整个事件流程之后,接下来要思考如何实现,先从一个服务端入口程序开始一步步往后推导.

- 服务端入口程序(也可以称为客户端)

```java
// 这里使用java实现一个简易版本的Reactor模式
class NIOReactorServer {
   public static void main(String[] args){
     	// ServerSocketChannel 类似Button
      ServerSocketChannel server = ServerSocketChannel.open();
      server.bind(new InetSocketAddress(port), MaxBackLogs);
      server.configurable(false);
     // 类似于button.bind操作
      Reactor reactor = new Reactor();
      Handler request = new RequestHandler();
      reactor.register(ACCEPT, server, new Acceptor(request));
     // 类似于button.click();只不过这里是处于阻塞等待事件
      reactor.handle_events();
   }
}
```

- 定义事件处理器

```java
abstract class Handler {
    public void acceptorHandler(SelectableChannel server){
       // do nothing
    }
  
    public void requestHandler(SelectableChannel client){
       // do nothing
    }
}

// 服务端处理Acceptor服务
class Acceptor extends Handler{
  
  private Handler handler;
  
  public Acceptor(Hanlder handler){
    this.handler = handler;
  }
 
   public void acceptorHandler(SelectionKey key){
       // 需要从server中获取客户端的socket
       Socket client = key.channel().accept();
       // 重新注册到reactor上
       reactor.register(READ, client, handler);
    }
}

// 服务端处理Request请求操作
class RequestHandler extends Handler {
   
    public void requestHandler(Request req,Response resp){
      // decode 
      // process
      // encode 
    }
}
```

- Reactor组件

```java
// 为了保证实现的Reactor是通用的,这里不使用java的NIO实现,仅用java伪代码实现
class Reactor {
  // 事件通道,在Java中是使用SelectionKey保存每个socket事件
  private Map<SelectionKey, Invoker> acceptMap = new ConcurrentHashMap<>();
  private Map<SelectionKey, Invoker> readMap = new ConcurrentHashMap<>();
  private Selector demultiplexer;
  
  public Reactor(){
    this.demultiplexer = Selector.open();
  }
  
  public void register(String type,SelectableChannel socket, Handler handler){
      // 向系统内核注册socket事件并投递到事件等待队列中
      if (ACCEPT.equals(type)){
        socket.register(demultiplexer, SelectionKey.ACCEPT);
        map.put(socket, new Invoker(ACCEPT,handler));
      }else if(READ.equals(type)){
        socket.register(demultiplexer, SelectionKey.READ, new Request());
        map.put(socket, new Invoker(READ,handler));
      }
  }
  
  public void handle_events(){
     while(true){
        Set<SelectionKey> readySet = demultiplexer.select();
        Iterator<SelectionKey> it = readySet.iterator();
        while(it.hasNext()){
          SelectionKey key = it.next();
          it.remove();
            if (acceptMap.keys().contains(key)){
               dispatch(ACCEPT, key);
            }else if(readMap.keys().contains(key)){
                dispatch(READ, key);
            }
        }
       readySet.clear();
     }
  }
  
  public void dispatch(String type,Selection key){
    Invoker invoker = null;
    Method method = null;
      if (ACCEPT == type){
         invoker = acceptMap.get(key);
         method = invoker.getCallbackMethod();
        // method callback
      }else if(READ == type){
         invoker = readMap.get(key);
         method = invoker.getCallbackMethod();
         // read data ...
         Request req = (Request)key.attachement();
         req.setData(readData);
        
         // using method callback (pass req and resp)
        
         // write data ...
      }
  }
}

// 可以理解为事件通道
class Invoker {

   private Handler handler;
   private Method method;
   
  public Invoker(String type,Handler handler){
    this.handler = handler;
    if(ACCEPT == type){
      this.method = this.handler.getClass().getDeclaredMethod("acceptHandler", SelectionKey.class);
    }else if(READ == type){
      this.method = this.handler.getClass().getDeclaredMethod("requestHandler", Request.class, Response.class);
    } 
  }
  
   public Handler getHandler(){
     return this.handler;
   }
  
   public Method getCallbackMethod(){
     return this.method;
   }
}
```

- 通过上述部分伪代码的设计实现,一个通用的NIO设计组件结构如下所示:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121338567.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 关于Reactor使用Java的NIO实现,后面讲述netty的时候会更为详细,这里主要是说明Reactor设计的实现思路,最后通过实现Reactor时序展示运作流程,以epoll/kqueue为准,如果为select,那么图中的第2步和下面的事件轮询都是合并在同一步操作中

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121352233.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

接下来我们可以来了解下IO事件驱动设计的异步实现原理,即Proactor模式实现

#### Proactor设计原理
##### AIO模型以及API
在IO事件驱动设计实现,还有另一种实现模式,即Proactor模式,以网络AIO模型为基础,面向IO事件编程的一种模式

- AIO使用的API

```c++
// 将处理请求入队进行异步读取
int aio_read(struct aiocb *aiocbp)
// 返回aio的处理结果
ssize_t aio_return(struct aiocb *aiocbp);
// 将处理请求入队进行异步写出
aio_write()
// 对一个IO同步操作入队并通过异步的方式执行
aio_fsync()
// 返回入队异步执行请求的错误
aio_error()
// 挂起调用者直到有一个或者多个就绪事件发生  
aio_suspend()
// 尝试取消IO操作
aio_cancel()
// 使用单个函数调用已入队的多个IO请求
lio_listio()

// 携带的结构体
struct aiocb {
    int             aio_fildes;     /* 文件描述符 */
    off_t           aio_offset;     /* IO操作执行的文件位置 */
    volatile void  *aio_buf;        /* 实现数据交换的buffer */
    size_t          aio_nbytes;     /* buffer大小 */
    int             aio_reqprio;    /* 执行请求的优先级 */
    struct sigevent aio_sigevent;   /* IO操作完成的时候进行异步回调通知的方法 */
    int             aio_lio_opcode; /* 操作类型,仅用于lio_listio*/
  };
```

- AIO模型

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121416891.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

在Unix网络IO模型中,AIO的工作原理是由应用进程定义好一个异步操作并通过`aio_read`方法的调用告知内核启动某个操作(异步操作)并在整个操作(等待数据+数据copy)完成之后通知应用进程,同时需要向内核传递文件描述符,缓冲区引用和其大小以及文件的偏移offset,并告知内核完成操作之后如何通知应用进程.

##### Proactor运作流程

通过上述的AIO模型分析,我们可以类比Proactor与Reactor实现模式,对于Proactor模式而言,只是使用的IO策略不同,因而在设计的实现细节也会有所不同,可以通过Reactor事件流程,我们可以推测Proactor模式的事件流程如下:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121433152.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

通过上述可以粗略看到Proactor模式与Reactor模式在设计思路上是基本一致:
- 1)都是基于事件驱动设计实现,同时将Handler与关注的IO事件操作分离,
- 2)开发者可以更加集中于Handler的业务实现逻辑,重要的区分在于Reactor依赖的是同步IO的复用器
- 3)Proactor依赖的是异步IO的复用器实现.同时Proactor的核心操作主要有注册异步操作以及业务处理的Handler
- 4)异步接收完成操作的通知以及获取就绪事件和对应的完成结果的集合,而Handler与Reactor模式基本一致.

##### Proactor组件具体实现

> Proactor组件运作流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121448179.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

> Proactor组件参与者

- Handle: 可以理解为事件源,在这里表示网络socket对象
- Completion Handler: 定义一系列接口模板方法用于处理异步操作完成的结果处理逻辑
- Proactor: 提供应用程序的事件循环，将完成事件分解为相关的完成处理程序，并分派抽象模板方法来处理结果
- Asynchronous Event Demultiplexer: 异步多路复用器,阻塞等待添加到完成队列中的完成事件,并将它们返回给调用者
- Completion Event Queue: 对等待多路复用器的完成事件进行缓冲,以便于完成事件的处理Handler能够从队列缓冲中获取相应的completion event进行处理.
- 异步操作: 主要用于处理程序中长时间持续操作
- 异步处理器: 绑定在Handle上,负责对监听到Handle执行进行回调唤醒对应的异步操作,生成对应的CompletionEvent并添加到事件的缓冲队列中
- Initiator: 本地应用程序服务入口,初始化一个异步操作并注册一个完成处理程序和一个带有异步操作处理器的Proactor,当操作完成时通知它

> 具体实现

通过上述组件的协作流程以及Proactor的组件说明,对此,我们可以从主程序入口开始推导Proactor的实现流程.

- 主程序入口Initiator

```java
// 用java伪代码模拟
class NIOServer {
  public static void main(String[] args){
    // 保存异步操作产生的CompletionEvent
    final Queue<CompletionEvent<Object>> completedEventQueue = new ConcurrentLinkedQueue<>();
    
     // 初始化一个异步操作,提交给内核接收到事件变化执行异步操作
    AsyncOperactionProcessor<Object> processor = new AsyncOperactionProcessor<>(){
    		public void asyncOprAccept(AsyncChannel channel, Handler<V,A> handler, Object attach){
           // 操作完成之后创建CompletionEvent
           CompletionEvent event = new CompletionEvent(channel, handler, attach);
           // 新的连接
           event.setResult(channel);
           //添加到队列中以便于处理CompletionHandler能够进行处理
           completedEventQueue.add(event); 
        }
      
        public void asyncRead(AsyncChannel channel, Handler<V,A> handler, ByteBuffer result, Object attach){
          // 操作完成之后创建CompletionEvent
          CompletionEvent event = new CompletionEvent(channel, handler, attach);
          // 请求操作
          // 发起读取操作
          channel.read(result);
          event.setResult(read.size());
        }
      
        // write opr ...
    };
    
    // 创建server
    AsyncServerSocketChannel server = AsyncServerSocketChannel.open().bind(9999);
   	 // 创建Proactor并注册一个处理Accept的完成事件的Handler
    Proactor proactor = new Proactor(server, processor, new Acceptor(server,processor));
    // 监听socket事件并处理socket事件
    proactor.handle_events();
  }
}
```

- 异步操作处理器

```java
// 定义异步处理器接口
abstract class AsyncOperactionProcessor<A> {
    public abstract void asyncAccept(AsyncChannel channel, Handler<V,A> handler, A attach);
  
    public abstract void asyncRead(AsyncChannel channel, Handler<V,A> handler, ByteBuffer result, A attach);
  
    // write opr ...  
}
```

- Proactor组件

```java
// 伪代码实现Proactor组件
class Proactor {
   private AsyncOperactionProcessor processor; 
   private Handler handler;
   private AsyncServerSocketChannel server;
   
   public Proactor(AsyncServerSocketChannel server, AsyncOperactionProcessor processor, Handler handler){
       this.server = server;
       this.processor = processor;
       this.handler = handler;
   }
  
   public void handle_events(){
     while(true){
       	 // 接收客户端新的请求
         Future<Queue<CompletionEvent>> result = this.server.accept(this.processor, this.handler);
         Queue<CompletionEvent> queue = result.get();
         CompletionEvent event = null;
         while((event = queue.pop()) != null){ 
             dispatch(event);
         }
     }
   }
  
  public void dispatch(CompletionEvent event){
      Handler handler = event.getHandler();
      ParameterizedType type = event.getHandler().getClass().getGenericInterfaces();
      Class<?> resultClass = (Class<?>) type.getActualTypeArguments()[0];
      Class<?> attachClass = (Class<?>) type.getActualTypeArguments()[1];
      Method method = event.getClass().getDeclaredMethod("completed", event.getResult(), attachClass);
      method.invoke(handler, event.getResult(), event.getAttachment());
  }
}
```

- CompletionEvent组件

```java
class CompletionEvent<A> {
   private AsyncChannel channel;
   private Handle<V, A> handler;
   private A attach;
   
   public CompletionEvent(AsyncChannel channel, Handle<V, A> handler, A attach){
      //....
   }
}
```

- Handler组件

```java
interface Handler<V, A> {
    void completed(V result, A attach);
}

// Acceptor类实现Handler,处理对应的业务,即实现客户端socket的注册流程
class Acceptor<AsyncSocketChannel, Object> implements AcceptorHandler {
   private AsyncServerChannel server;
   private AsyncOperactionProcessor processor;
     
   public Acceptor(AsyncServerChannel server, AsyncOperactionProcessor processor){
     this.server = server;
     this.processor = processor;
   }
  
  	void completed(AsyncSocketChannel socketChannel, Object attch){
      	ByteBuffer buffer = ByteBuffer.allocate(MAX_SIZEs);
        Handler reqHandler = new ReqHandler(this.processor, client, read);
        // val根据业务自定义,可以定义为存储Session信息
        // 注册读取事件并执行异步的读取操作
        socketChannel.read(this.processor, reqHandler, read, attch);
    }
}

// RequestHandler类实现Handler,处理的对应业务,即处理decode-process-encode操作,同时注册写操作
class ReadHandler<Integer, Object> extends Handler{
   private AsyncSocketChannel client;
   private ByteBuffer read; 
   private AsyncOperactionProcessor<CompletionEvent> processor;
  
    void completed(Integer buffSize, Object attch){
      byte[] buffer = new byte[buffSize];
        read.flip();
         // Rewind the input buffer to read from the beginning
        read.get(buffer);
        // deocde 
        // process
        // encode
        
        // 注册写出事件,并以异步的方式执行写出操作
        Handler writeHandler = new WriteHandler(client);
        ByteBuffer output = ByteBuffer.wrap(buffer);
        client.write(this.processor, writeHandler, output, attch);
    }
}
```

至此，Proactor核心组件都用java模拟出来,主要目的是为了能够更好地去理解Proactor模式,同时对于一个异步处理器以及其中的异步操作,可以将其绑定在对应的AsyncChannel上,由AsyncChannel去实现相应的异步操作就可以将上述的设计变得更为简单,不需要再传递对应的异步操作处理器processor,绑定在channel能够直接传递到系统内核中,当有事件就绪的时候内核直接触发异步操作然后唤醒到应用程序执行操作后的结果处理Handler.在Java的AIO使用的API是`CompletionHandler`以及`AsynchronousChannel`之间的协作.最后,基于上述的代码实现,对一个通用的Proactor模式组件设计类图如下:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121514146.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

#### Reactor&Proactor小结

##### Reactor模式与Proactor模式对比

> 相同点

- 均是基于事件驱动设计模式的解决方案来设计支持并发连接的web服务,指示如何在网络IO环境中发起,接收就绪事件,解复用事件,分发以及执行不同类型的事件.
- 提供可重用以及可配置的解决方案和应用程序组件,通过组件分离不同事件的关注点,有助于针对相应的关注点进行调试和优化

> 不同点

- Reactor模式是基于同步多路复用器,使用的非阻塞同步IO的API协作完成,Proactor模式是基于异步多路复用器,使用的是异步IO的API协作完成,整个执行过程都是异步化.
- 对于异步读取数据(从内核数据复制到用户缓存区)是持续不间断执行,因此会对内存空间的缓存区域造成很大的压力,存储的数据会越来越多,不知道数据什么时候能够被消费完成释放空间,而Reactor模式属于同步读取,不存在对缓存空间的内存压力.
- Reactor模式本质上是属于同步操作,而Proactor是属于异步操作,在先前的高性能IO中表述到,同步存在以下几个问题,一个是同步在资源竞争环境下性能会比异步更差些,二是存在可伸缩性问题,Reactor模式是在原有的连接线程架构分离关注点优化,但是在处理有业务逻辑的相关处理时候仍然存在同步的移植以及伸缩问题,也就是对于并发连接的优化上去了,但是对于复杂的QPS仍然会是一个瓶颈.对于Proactor模式的异步操作,其运作效率依赖于内核执行效率,和操作系统有关,无法控制被调度的异步操作以及难以对程序进行调试排错.
- Reactor模式是等待就绪事件发生然后依次顺序处理就绪事件,Proactor模式是等待就绪事件完成处理完成之后的

##### Reactor&Proactor使用库

- ACE框架: 提供Reactor以及Proactor模式实现,可以了解下UniPi项目,一个并行环境使用ACE的Reactor模式实现并发通信的分布式程序.
- Boost.Asio库: 基于Proactor模式提供同步与异步操作提供并行支持
- TProactor: 模拟Proactor

> 一个关于TProactor的性能分析对比如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410121706901.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

> 最后关于Java相关NIO的API

```text
https://docs.oracle.com/javase/7/docs/api/java/nio/package-summary.html
https://www.ibm.com/developerworks/java/library/j-nio2-1/index.html
https://www.javacodegeeks.com/2012/08/io-demystified.html
```
