---
title: volatile工作原理
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->


在前文中已经讲解到volatile的使用以及原子性的问题,volatile修饰的变量可以实现线程对变量的写操作能够让其他线程“看到”当前最新的变量数据值,从内存语义上而言,相当于告诉线程读取当前变量要从主内存中读取,对此,现将继续前文的volatile继续深入原理分析.
###### 1. volatile作用与使用场景
> volatile规则与作用

- 基于Happen-Before的原则,对volatile变量执行写操作happen-before该变量读操作的后续每个动作
- 另外,基于内存语义,我们可以知道volatile读取的数据变量可以立马“看到”volatile在另一个线程写入的最新值,同时遵循先写后读的顺序规则
- 对于声明volatile变量的单步的操作具有原子性
- 还有一点就是volatile底层处理器是**使用内存屏障的机制来强制工作内存失效,从而消除处理器的重排序**

> 使用volatile的场景

- 当定义的数据变量需要与其他CPU寄存器需要进行数据交互的时候,即在多核CPU的机器下需要声明为volatile,从而保证数据是可见的
- 当数据存储在主内存(JMM中定义的共享内存)时,由于主内存中并没有存在锁的保护,并且依赖于内存的访问顺序,使用volatile的开销会比lock更“廉价”

###### 2. volatile内存语义
> 部分源码

```java
// shared.java
volatile boolean finished = false;
producer(){
    TimeUnit.SECONDS.sleep(1);
    finised = true;
    System.out.println("have finished product done ....");
}
consumer(){
   while (!finised){
    // nothing
  }
  System.out.println("have consume product done " + full);
}
// producer.java
// 代码仅做演示,忽略其他因素
run(){
	producer();
}
// consumer.java
run(){
	consumer();
}
```

> 基于上述代码内存演示图

- 代码初始化过程,将数据复制到工作内存中
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200123110749810.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 生产者开始对数据进行写操作,基于volatile语义可以看到写操作之后是刷新到主内存中
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200123112223315.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 运行结果
	- 使用volatile修饰的消费者线程退出循环完成程序的正常执行
	- 不使用volatile修饰的消费者线程由于读取工作内存的数据将会处于不断循环中,没有退出程序

- 基于上述的内存分析,我们也许会存在一个问题
	- 消费者线程在读取的时候看到finshed是volatile修饰为什么工作缓存会失效?
	- 也就是接下来要说明的内存屏障,内存语义的实现机制

###### 3. volatile内存屏障实现
内存屏障是处理器层面进行的,因此这里直接查阅jvm下的cpu架构源码对volaitle的内存屏障进行说明

> 关于ARM(指令集机器指令)参考说明

- dmb: Data Memory Barrier
- ish:  DMB operation only applies to the inner shareable domain.
- ishld: DMB operation that waits only for loads to complete, and only applies to the inner shareable domain.
- 参考文档: http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0802b/CIHGHHIE.html

> JVM中的aarch64架构处理器对volatile的内存屏障说明

- `ldar<x>` 表示volatile的读指令, `stlr<x>` 表示volatile的写指令
```c++
// AArch64 has ldar<x> and stlr<x> instructions which we can safely
// use to implement volatile reads and writes. For a volatile read
// we simply need
//
//   ldar<x>
//
// and for a volatile write we need
//
//   stlr<x>
```

- 读写屏障流程
```c++
// 读屏障
// Alternatively, we can implement them by pairing a normal
// load/store with a memory barrier. For a volatile read we need
//
//   ldr<x>			// 读取volatile数据
//   dmb ishld		// 内存屏障
	
// 写屏障	
// for a volatile write
//
//   dmb ish		// 内存屏障
//   str<x>		    // 写volatile数据
//   dmb ish		// 内存屏障
```

- 读写屏障伪代码实现
```java
// demo.java
volatile int j = 0;
//threadA write 
run(){
	// ...code start ...
	j = 9;
	// ... code end ...
}

//threadB read
run(){
	// ... code start ...
	a = j;
	a ++;
	// ... code end ...
}
```
- 对上述的读写转换为aarch伪指令如下
```c++
// 写屏障
threadA run(){
	//... code start ...
	dmb ish			// 防止上面代码与写volatile代码重排
	str<i>			// j = 9,由于是写屏障,所以会使缓存失效,并更新到主内存中
	dmb ish			// 防止下面代码与写volatile代码重排
	//... code end ....
}

// 读屏障
threadB run(){
	// ... code start ...
	ldr<x>			// 读取j的数据
	dmb ishld		// 内存屏障,防止与下面的a++重排序
	a++;
	// ... code end ...
}
```

- 结果分析
	- 对于重排序目的无非就是要将一些读操作优先处理或者说是本地CPU能够不依赖于主内存处理的任务先处理
	- 因此,对于volatile的写操作,增加内存屏障一个是防止重排序,为什么可以防止重排序,主要是内存屏障强制CPU将当前的缓存失效,直接从主内存中读取数据,CPU就无需因为性能的问题再去重排序
	- 同理,对于volatile的读操作,增加内存屏障,也就是需要从主内存中获取数据进行下一步的操作,也无需再进行重排序

###### 4. volatile工作原理小结
> 问题思考

就是前文说明有一个-server服务端模式,编译器会对代码进行重排序,而这里说的volatile因为内存屏障而没有重排序不会矛盾?
重排序有编译器重排序,也有CPU处理器重排序,内存屏障是在CPU层面进行的,也就是说使用volatile修饰的变量仍然会在编译阶段导致代码重排,但是字节码转换为机器码的时候会识别有带volatile的修饰,因此会增加内存屏障来防止处理器进行重排序,两者并不矛盾

> 总结

- volatile变量的写操作将会使得当前本地缓存(线程/CPU等工作内存)失效,直接刷新数据到主内存(堆内存)中
- volatile变量的读操作将会使得当前本地缓存失效并从主内存中重新加载读取数据
- volatile的内存语义是基于内存屏障的机制实现,因此读取的数据可以保证写入的数据是最新的