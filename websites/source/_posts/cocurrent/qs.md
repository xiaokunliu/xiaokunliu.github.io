---
title: 并发编程面临的问题小结
category: 并发编程
date: 2020-02-28 20:47:28
tags: 并发编程
---

<!-- more -->
###### 1. 多线程上下文切换
> 上下文切换

在单核CPU机器下,也可以支持并发多线程执行代码,这个时候CPU会为每一个线程分配对应的时间片,通过在指定的时间片内执行对应的线程程序代码,时间片一到,线程再继续争抢CPU资源重复上述动作

> 上下文切换对并发编程的影响

- 示例代码
```java
// cpu_test.java
// 定义业务方法
private static void meth(){
    long a = 0;
    long b = 100000000000000L;
    for(int index = 0; index < count; index ++){
        a += 2;
        b -= 4;
    }
}

// 当前mac机器配置: 4CPU
// 并发:创建6个线程分别执行上述方法一次
private static void cocurrent() throws Exception {
    long start = System.currentTimeMillis();
     // t1 - t5
      Thread t1 = new Thread("Thread-1"){
          @Override
          public void run() {
              meth();
          }
      };
       // ... 重复代码省略 ...
      meth();
      
      // t1 - t5 start
      t1.start();
     // ... 重复代码省略 ...
     
      // t1 - t5 join()
      t1.join();
      // ... 重复代码省略 ...

      long end = System.currentTimeMillis();
      System.out.println(Thread.currentThread() + " spend time : " + (end - start));
  }
// 串行:直接调用6次方法
 private static void serial(){
    // Thread[main,5,main] spend time : 4
     long start = System.currentTimeMillis();
     meth();
     meth();
     meth();
     meth();
     meth();
     meth();
     long end = System.currentTimeMillis();
     System.out.println(Thread.currentThread() + "  spend time : " + (end - start));
}
```

- 执行结果

| 次数(count) | 1w |  10w |  100w |  1000w  |  1 亿 |  
|--|--|--|--|--|--|
|串行耗时(ms)  | 2 | 5 | <10 | 53 |  466 |
|并发耗时(ms)  |  2|  10 |	10-12 |  25 |  166 |
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200205150407252.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
```text
查看上下文切换明细
- linux下使用vmstat/pidstat
- 使用工具Lmbench3
```
- 没有运行java程序前的cs变化
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200205150936630.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 结果分析
	- 在count数据不是特别多的情况,串行执行的效率比并发快,因为并发执行需要切换线程上下文
	- 随着次数的增加,串行执行的效率比并发执行效率低,原因是当前线程充分利用CPU核数的资源,利用多个线程在相应的CPU上执行,使得任务被对应的线程消费,在这种情况下,并发线程充分利用CPU空闲的资源完成任务的调度

> 上下文切换消耗资源的优化方案

- 设计适当的线程数, 根据CPU核数以及jmeter测试单进程单线程1s执行的并发效率来调整最优的并发QPS
- **使用线程池技术**
- 协程: 相当于代码段或者是函数式的程序代码,相比于程序代码而言,协程可以在当前线程中段转而执行其他代码片段,**在单线程中来回切换多任务的函数式代码块,不存在上下文切换,也不存在锁,简言之,就是“子程序就是协程的一种特例”**
- 协程伪代码
```java
// 生产者 - 消费者模型, 每生产一个数据就消费一次
void producer(Consumer consumer){
	while(true){
		int num = RandomUtils.nextInt(0, 100);
		log.info("produce num %d", num);
		Object result = caller.send(consumer, num);
	// 通知消费者进行消费,当前程序中断挂起,不再继续执行,caller为调度器,协程必须有一个调度器提供子程序切换执行
		log.info("conusmer num return %s", result);
	}
}

// 消费者
void consumer(){
	while(true){
		// get num by caller, 接收从调度器返回的数据,如果没有数据则中断并挂起当前程序
		int num = caller.receive();
		log.info("conusmer consuming the num %s", result);
		result = “consume OK”;
	}
}
```

###### 2. 线程安全之原子性
> 临界区与数据竞争

- 临界区: 在并发多线程中执行一系列对共享资源的修改操作的代码区域,在该区域下的操作的执行结果会对其他线程产生影响,称该代码区域为临界区
- 竞态条件: 表示并发多线程执行产生临界区的必要条件,也就是在临界区存在数据竞争,而数据竞争主要条件就是来源于多线程需要对共享资源执行读写操作,简言之就是多线程争夺共享资源的使用
- 代码示例
```java
// sahred.java
int num = 0;	// 在多线程中对于共享资源存在数据竞争,竞态条件

// mutil.java
run(){
	num ++;		// 临界区
	
}
```

> 原子性操作与线程安全

- 原子性操作
	- 对程序代码指令而言,表示一个步骤;对整体业务逻辑而言,表示一系列步骤并且这一系列步骤在整体业务逻辑中一个是不可被中断, 与其他的业务代码步骤是不可被重排序的
	- 核心特征就是相对整体业务逻辑而言,该一系列步骤要么全部成功,要么全部失败,在整体的业务逻辑保持操作结果的一致性
	- 需要原子性操作的原因,在并发多线程中存在竞态条件,临界区的执行结果会对其他线程产生影响,如果不能保证所有线程看到的操作都是一致的(要么成功然后处理成功的逻辑,要么失败然后处理失败的逻辑),这样才能够保证我们的实际应用业务是可控的,结果是可预期的
	
- 线程安全
	- 是属于一个概念性名词,主要体现线程在并发情况下可能会发生脏读或者是业务逻辑顺序被打乱出现的不可预期的现象

> JVM的资源

- 共享资源(存在线程安全问题)
	- 在JVM运行数据区中,方法区和堆内存均是属于共享资源数据,存在线程安全问题
- 线程封闭资源(不存在线程安全问题)
	- 在当前线程栈中的局部变量.方法参数,抛出异常的处理器对象,由于只在线程栈中自己使用,并没有共享给其他线程,因此这类数据是属于线程安全的,也就是不存在数据竞争的情况
	- ThreadLocal以及ThreadLocalRandom等存储的数据变量
	- 不可变的变量数据,即使用final修饰的变量数据

> 解决线程安全的技术手段

解决线程安全的问题手段,最主要就是要防止脏读,抑或是顺序被打乱的情况,也就是要保证在整体业务逻辑代码中操作的一致性,那么实现的技术手段 -- 通过加锁的方式实现共享资源的原子性问题
- java加锁方式
	- [基于底层系统的原子操作原语实现的CAS机制](https://blog.csdn.net/wind_602/article/details/104099524)
	- [基于AQS方式的加锁方式](https://blog.csdn.net/wind_602/article/details/104161960)
	- [基于JVM实现的监视器锁对象的同步关键字synchronized](https://blog.csdn.net/wind_602/article/details/103966182)

###### 3.死锁
> 产生原因

- 多线程相互争抢对方相互持有的资源,由于获取不到资源一直处于挂起状态而无法继续往下执行
- 伪代码

```java
// threadA.java
run(){
	synchronized(lockA){
		// ..
		synchronized(lockB){
			// ...
		}
	}
}
// threadB.java
run(){
	synchronized(lockB){
		// ..
		synchronized(lockA){
			// ...
		}
	}
}
```

> 解决方案

- 可以考虑将线程加入队列中,按照程序指定的逻辑先后执行
- 使用tryLock(timeout)的方式,一旦超时将自动释放锁资源
- 其他方案: 在业务代码中如果能够使用单锁解决问题则使用单锁的方式

###### 4. 机器资源限制
- 并发编程环境受限于机器资源的限制
	- 比如硬件方面有CPU核数以及CPU的处理读写能力, 网络带宽问题, 磁盘读写速度, 磁盘空间, 内存空间等因素;
	- 软件资源一般是并发线程池的数量,比如tomcat服务的并发线程数, 数据库连接池大小, 网络socket连接数等

- 资源限制导致的问题
	- 如果机器的CPU核数较少,比如只有一个的话,在机器启动jvm进程来创建多线程会容易导致线程切换频繁,再加上本身线程切换存在资源调度的性能消耗,容易降低程序执行效率
	- 内存空间不足也会导致创建并发线程个数受限,同时容易造成OOM的错误
	- 业务处理线程数多于数据库连接池数,如果数据库中的sql执行比较快的话,那么会导致程序很多业务进程处于阻塞等待状态,容易引起100%的CPU

- 资源限制解决方案
	- 水平扩展, 增加机器实现集群方案来分担机器的压力
	- 适当减少并发线程数量,尽量调整为CPU*2+1的线程数
	- 根据业务所处的场景,对文件并发读写频繁可以选择磁盘IO处理能力较强的机器,网络并发读写频繁可以选择带宽较好的机器等

###### 5. 可见性问题
参考[Java内存模型之可见性分析](https://blog.csdn.net/wind_602/article/details/104041188)
