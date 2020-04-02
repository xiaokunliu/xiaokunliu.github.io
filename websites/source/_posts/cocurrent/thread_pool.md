---
title: 并发编程之线程池原理
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->



###### 1. 线程池作用
> 使用背景

- 在并发大量异步任务处理程序中,每执行一个任务就需要创建一个线程,同时任务执行完毕之后需要将线程销毁.我们知道JVM创建线程的时候需要为其分配线程栈空间以及一些初始化操作,同时销毁的过程需要回收线程栈空间并由gc释放资源,期间都需要耗费一定的时间,因此一个任务的最终执行时间=创建线程newTime + 程序执行excuteTime + 线程销毁的gcTime,如果期间newTime+gcTime >  excuteTime,那么这时候执行任务创建线程在程序中就显得十分不划算
- 线程执行需要JVM分配线程栈空间,需要从系统内存申请,如果创建的线程过多,那么就容易导致内存空间不够用导致内存溢出,默认的线程栈空间大小是1M
- 线程是由操作系统的CPU进行调度,因此并发多线程执行时CPU需要分配时间片并发执行线程,也就是线程并发执行是需要来回切换CPU的context,严重影响性能
- 并发环境下,如果创建的线程很多,增加对线程的维护和管理的困难

> 作用

- 运用资源重复利用的思维,我们建立一个“池”的概念,多任务异步执行通过线程池实现线程复用,利用池化技术来分配和管理线程的使用,避免线程频繁创建和销毁消耗更多的时间,**提高并发执行效率**
- 其次,**通过线程池我们可以控制线程的数量,可以根据指定的策略来管理线程**,比如任务过多,线程处理不过来,可以分配新的线程,当线程数量达到上限时,可以自定义策略管理任务,要么是放入阻塞队列中等待线程消费完成再继续,要么直接丢弃任务,相比单个线程处理方式,灵活性更大,也容易管理
- **最后,由于池可回收线程资源,能够降低资源消耗**

###### 2. 线程池相关API
> 线程池接口API

- 线程池核心接口与实现类类图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200205220103135.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- Executor & ExecutorService接口核心方法
```java
// Executor.java
// 从线程池中分配一个线程执行任务
void execute(Runnable command);		

// ExecutorService.java
// 关闭线程池,会将已提交的任务执行完毕再关闭线程池,不接受新的提交任务
void shutdown();					
// 立即关闭,不论是否有任务正在执行还是已提交未执行,都立即退出jvm不再执行,返回已提交未执行的任务
List<Runnable> shutdownNow();
// 线程池是否关闭
boolean isShutdown();
// 线程池调用shutdown()后,所有任务是否已经执行完成
boolean isTerminated();
// 不论是线程中断,超时还是线程池shutdown()发生,将会阻塞知道所有任务执行完成
boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
// 提交Callable任务返回Future,用于获取Callable执行结果
<T> Future<T> submit(Callable<T> task);
// 提交Runable任务,并返回Future对象,同时将执行结果传入result
<T> Future<T> submit(Runnable task, T result);
// 提交提交Runable任务,并返回Future对象,执行结果为null
Future<?> submit(Runnable task)
// 执行一系列的任务,并返回对应的Future集合对象,同时Future包含task的执行结果
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
// 执行一系列任务,其中任意一个执行成功将返回Future对象并取消其他任务的执行,Future包含成功执行task的结果
<T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
```

- 接口ScheduledExecutorService核心方法(可选择)

```java
// 主要用于处理定时任务
//ScheduledExecutorService.java
// 创建并执行一个一次性定时任务,指定在 delay&unit(时长+时间单位) 时间点将会执行
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay, TimeUnit unit);
public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                   long delay, TimeUnit unit)

// 下面两个方法主要是创建并执行一个周期性任务,如果发生异常会退出程序
// 区别在于
// scheduleAtFixedRate执行周期任务之后,如果任务超过指定的周期时间,下一个任务会立刻执行不会等待
// scheduleWithFixedDelay执行任务超过周期时间,仍然会等待delay时间再进行下一个任务的执行
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);  
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                   long initialDelay,
                                                   long delay,
                                                   TimeUnit unit);                                                                               
```

> 线程池工具类API

- 线程池与线程池工具类类图关系
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200206105559185.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 根据类图可以将线程池划分为定时和非定时
	- 非定时线程池实现类: ThreadPoolExecutor / ForkJoinPool /  FinalizableDelegatedExecutorService
	- 定时线程池实现类:  ScheduledThreadPoolExecutor /  DelegatedScheduledExecutorService

- 线程池工具类Executors创建线程池的核心方法

```java
// Executors.java
// 创建一个基于无界的链表阻塞队列(LinkedBlockingQueue)且为固定线程数为nThreads的线程池
// 无界: 队列没有指定容器大小,可以不断添加元素
public static ExecutorService newFixedThreadPool(int nThreads) {}
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {}

// 创建一个基于无界的同步队列(SynchronousQueue)且线程数无限制(0 - Integer.MAX_VALUE)的线程池
// 1.该线程池线程空闲时间为60s,超时将销毁线程,适用于耗时较小的异步任务
// 2.不存储数据的同步队列(最多只有一个元素的队列),仅作为缓存作用,也就是等待内部创建工作线程完成之后就立即交由线程进行消费
// 3.适用于处理任务但是不确定线程个数的场景,同时为了防止不断创建线程造成CPU资源消耗过多,一般会添加对应的线程最大数而不是Integer.MAX_VALUE
public static ExecutorService newCachedThreadPool() {}
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {}

// 创建一个基于延迟的无界任务队列且能够执行定时任务的线程池
// 线程池的数量corePoolSize - Integer.MAX_VALUE
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize){}
public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory)

// 创建一个基于延迟的无界任务队列,能够执行定时任务且只有一个线程的线程池
// 当该线程池中的线程被中断或者异常退出的时候,线程池会新创建一个线程继续执行后续的任务
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {}
public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory){}
```

- Executors包含的组件
	- **接口ThreadFactory实现类**,借助内部定义的默认工厂类创建线程,并为当前线程池中的线程命名比较规范的线程名称,有默认线程工厂以及带有ACC控制权限的线程工厂
	- **代理线程池类**,负责创建线程池,即将实现ExecutorService的线程池包装起来代理其完成相应的方法实现,具体实现类FinalizableDelegatedExecutorService是重写父类方法finalize以便gc调用回收线程池,DelegatedScheduledExecutorService是实现定时功能
	- **实现Callable接口的任务类**,RunnableAdapter主要实现处理任务task的执行结果返回给主线程,PrivilegedCallable以及PrivilegedCallableUsingCurrentClassLoader增加ACC控制权限设置来执行任务

###### 3. 线程池原理
> 线程池核心类ThreadPoolExecutor

- ThreadPoolExecutor类图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200205230637844.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- ThreadPoolExecutor类下核心组件
	- 工作线程Worker:包装一个线程以及任务的工作线程,在没有任务的时候处于等待,可以循环任务,并且实现AQS接口,从类图中可以看出具备独占锁的功能
	- ThreadFactory:用于创建线程的工厂类,并且为线程池中的每个线程分配更有意义的名称
	- 任务接口Runnable: 每个任务必须实现的接口,以便于工作线程调度任务的执行
	- 任务队列BlockingQueue: 用于存放没有处理的任务,相当于缓冲作用
	- 拒绝处理策略RejectedExecutionHandler: 对不满足线程池执行条件的任务采取指定的处理策略

- RejectedExecutionHandler策略
	- CallerRunsPolicy: 只用获取当前拒绝的任务的线程且线程是活跃状态的时候来执行任务,如果当前线程池已经被关闭将不会执行
	- AbortPolicy: 直接抛出异常, 告知当前的任务已经无法加入到阻塞队列中
	- DiscardPolicy: 不做任何处理,空方法,直接将任务丢弃,不执行
	- DiscardOldestPolicy: 移除队列中最靠前的一个任务,并执行当前拒绝的任务

- ThreadPoolExecutor 构造方法代码
```java
 public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            ThreadFactory threadFactory,
                            RejectedExecutionHandler handler) {
         
}
```

- ThreadPoolExecutor构造方法参数说明
	- corePoolSize: 线程池保持活跃的线程数
	- maximumPoolSize: 线程池可创建最大的线程数,**如果队列为无界队列,则maximumPoolSize没有效果**
	- keepAliveTime: 线程池中的线程保持活跃的时长, **如果超过时长, 并且线程池中的线程个数比核心线程数多,那么空闲的线程将会被回收**
	- unit: keepAliveTime的时间单位
	- threadFactory: 借助线程工厂类创建线程
	- handler: 当线程池中的队列已经满&线程数达到最大值,此时添加的新任务将采取handler策略进行处理

> 线程池执行流程

- ThreadPoolExecutor核心方法代码
```java
public void execute(Runnable command) {
     if (command == null)
         throw new NullPointerException();
    // ctl 高三位表示线程池状态,低(Integer.SIZE - 3)位表示线程个数,也就是ctl = 线程状态 + 线程个数
	// 获取当前池的状态和线程个数的组合值
     int c = ctl.get();
	
	// 当前线程的个数是否小于核心线程数
     if (workerCountOf(c) < corePoolSize) {
     	 // 创建新的线程执行任务,会通过加锁的方式保存工作线程,如果成功则执行任务
         if (addWorker(command, true))
             return;
         c = ctl.get();
     }
     // 判断当前线程池的状态是否为Running,是则将任务加入阻塞队列中,说明上述的corePool不够用
     if (isRunning(c) && workQueue.offer(command)) {
         int recheck = ctl.get();
         if (! isRunning(recheck) && remove(command))
         	 // 线程池已经shutdown并且不接受当前的任务,交由指定的策略处理
             reject(command);
         else if (workerCountOf(recheck) == 0)
         	// 线程池已经没有线程,由于任务已经加入阻塞队列,这里只需要新创建一个线程
             addWorker(null, false);
     }
     // 队列满,从线程池中创建新线程后处理任务,如果创建失败则执行策略
     else if (!addWorker(command, false))
         reject(command);
 }
```
- ThreadPoolExecutor内部执行示意图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200206122007466.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- ThreadPoolExecutor示意图说明
	- 主线程提交任务给线程池之后,线程池的corePool未满则创建线程执行任务(1,2)
	- 若corePool已经满了,则将任务提交到阻塞队列中(3)
	- 任务添加到阻塞队列之后将会进行双重检测,如果线程池关闭,则移除当前任务并执行拒绝策略处理任务返回主线程中,若线程池未关闭,再次检测corePool的线程个数,如果没有线程则创建新的线程但不执行任务(4.1&4.2)
	- 若队列已经满了,在创建新线程过程中会检测队列容量,corePool和maxPool的数量,如果满足< max就会创建新线程并执行任务,否则说明已经达到maxPool,创建失败并将任务移除到拒绝策略执行返回给主线程(5,6)
	- 对于非corePool下的线程,若存在空闲线程超过单位未unit的keepalive的时间,将销毁线程
	- 同时对于corePool下的空闲线程,将会从阻塞队列中获取任务并执行任务
	- 从线程创建流程知道,添加的工作线程将由一个基于HashSet的集合来维护,只有添加到set成功之后才会执行任务,如果检测到工作线程已经是启动过了,将不会再次添加到set集合中

- execute核心执行流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200206113502986.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- execute核心流程小结
	- 如果当前线程池中的线程数num < corePoolSize, 则创建新线程执行任务,需要获取全局锁
	- 如果当前线程池中的线程数num >= corePoolSize, 则将任务加入BlockingQueue中
	- 如果BlockingQueue已经满且线程数num < maximumPoolSize, 那么创建新的线程来执行任务,需要获取全局锁
	- 如果BlockingQueue已经满且线程数num >= maximumPoolSize,新加入的任务将交由指定handler策略处理,默认采取的策略是被拒绝

> 线程池状态

- 状态代码
```java
// 存储线程池状态以及工作线程个数(RUNNING, 0)
// rs表示状态,wc表示工作线程个数
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
// 状态存储在对应数值的高三位
// 接收新任务并能处理队列里的任务
private static final int RUNNING    = -1 << COUNT_BITS;
// 不再接收新任务但能处理队列里的任务
private static final int SHUTDOWN   =  0 << COUNT_BITS;	
// 将会在处理任务过程被中断,不再接收和处理队列的任务
private static final int STOP       =  1 << COUNT_BITS;	
// 所有任务已经执行完毕,工作线程个数为0且在调用terminated()之前的状态
private static final int TIDYING    =  2 << COUNT_BITS;
// terminated()执行之后的状态
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
// 获取当前线程池状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 获取当前线程池的工作线程个数
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

- 状态变化图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200206142711916.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

###### 4. 线程池监控
- getLargestPoolSize(): 返回核心线程池中当前最大的线程数,与workers的集合大小一致,表示当前线程池中的线程数量
```java
// 核心代码
addWorker(Runnable firstTask, boolean core){
	// ..
	// 自旋检测线程个数与队列容量,corePool,maxPool的比较
	mainLock.lock();
	int s = workers.size();
    if (s > largestPoolSize)
        largestPoolSize = s;
   // ...
   mainLock.unlock();
}
public int getLargestPoolSize() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        return largestPoolSize;
    } finally {
        mainLock.unlock();
    }
}
```

- getPoolSize(): 返回当前线程池状态不为TERMINATED(>=TIDYING)时的线程数量
```java
public int getPoolSize() {
     final ReentrantLock mainLock = this.mainLock;
     mainLock.lock();
     try {
         // Remove rare and surprising possibility of
         // isTerminated() && getPoolSize() > 0
         return runStateAtLeast(ctl.get(), TIDYING) ? 0
             : workers.size();
     } finally {
         mainLock.unlock();
     }
 }
```

- getActiveCount(): 获取当前正在处理任务的线程数量
```java
public int getActiveCount() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        int n = 0;
        for (Worker w : workers)
        	// worker是基于AQS实现的,内部启动执行任务的时候会加锁
            if (w.isLocked())
                ++n;
        return n;
    } finally {
        mainLock.unlock();
    }
}
```

- getTaskCount(): 获取线程池中所有的任务,包括已经执行完成的任务+正在执行的任务+在阻塞队列中未执行的任务个数
```java
public long getTaskCount() {
     final ReentrantLock mainLock = this.mainLock;
     mainLock.lock();
     try {
         long n = completedTaskCount;	// 线程池执行完成的任务个数
         for (Worker w : workers) {
             n += w.completedTasks;   // 当前工作线程执行完成的任务个数
             if (w.isLocked())
                 ++n;
         }
         return n + workQueue.size();
     } finally {
         mainLock.unlock();
     }
 }
```

- getCompletedTaskCount(): 获取当前线程池中已经完成的任务个数(线程池完成的任务 + 工作线程完成的任务)
```java
public long getCompletedTaskCount() {
     final ReentrantLock mainLock = this.mainLock;
     mainLock.lock();
     try {
         long n = completedTaskCount;
         for (Worker w : workers)
             n += w.completedTasks;
         return n;
     } finally {
         mainLock.unlock();
     }
 }
```
