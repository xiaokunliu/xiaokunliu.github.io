---
title: synchronized的工作原理(三)
category: 并发编程
date: 2020-03-06 20:47:28
tags: 并发编程
---

<!-- more -->


###### 1. synchronized的锁存储以及锁分类
> synchronized的存储位置: 对象MarkWork

* JVM的ObjectHeader信息
    * MarkWord: hashcode(哈希code) + age(分代年龄age) + biased_lock(偏向锁标志) + lock (锁标志)
    * Class Metadata Address(类元信息地址)
    * Array Length: 如果对象是一个数组类型,则存储数组长度

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200114151521308.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)


* MarkWord信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200114151647395.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

* JVM中synchronized使用的锁
	* 无锁: 严格意义上应该说是正常对象,包含hashcode + 分代年龄age + 无偏向锁标志 + 锁状态标志
	* 轻量级锁: 栈记录的地址 + 锁状态
	* 监视器锁: 对象/监视器地址 + 锁状态
	* GC标志: GC链接地址等 + 锁状态
	* 偏向锁(JVM提供的): 当前执行的线程ID +支持偏向锁标记的epoch + 分代年龄age + 偏向锁标志 + 锁状态

* JVM源码关于MarkWord的说明
```c++
// markWord.hpp
//  32 bits,32bit的MarkWord存储信息如下:
//  hash:25bit --------------->| age:4bit       biased_lock:1bit lock:2bit (normal object)
//  JavaThread*:23bit epoch:2bit age:4bit       biased_lock:1bit lock:2bit (biased object)

//  64 bits, 64bit的MarkWord存储的信息信息如下
//  unused:25bit hash:31bit -->| unused_gap:1   age:4bit    biased_lock:1bit lock:2bit (normal object)
//  JavaThread*:54bit epoch:2bit unused_gap:1   age:4bit    biased_lock:1bit lock:2bit (biased object)

// JVM底层代码定义的状态锁的值
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
//                                               not valid at any other time

// 当前线程持有偏向锁
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread 

// 匿名偏向锁,说明当前线程不持有偏向锁,但是对象已被其他线程设置为偏向锁
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased 
```
> jvm底层使用的锁
```c++
// 源码定义的锁
class BasicLock			// 轻量级锁
class BasicObjectLock   // 对象or监视器锁
```

> synchronized的锁细节问题说明

* 偏向锁: JVM创建对象如果没有发生竞争,则默认开启偏向锁,即偏向锁标志+无锁状态,如果创建对象多次(JVM经过统计)之后发现处于竞争状态将会关闭偏向锁,这是在JVM底层实现的,在JVM源码中显示为BiasedLock
* 轻量级锁:JVM启动并创建线程的时候,会在栈帧中为线程分配内存栈空间信息,此时线程栈会开辟一个锁记录(Lock Record)空间,并且会将锁定对象的mark word复制到锁记录空间中,jvm称锁记录的mark word为displaced_mark_word,线程将通过CAS完成对象的mark word复制操作,成功则获得锁,失败表示有其他锁竞争,将会通过自旋锁的方式获取(不断CAS循环获取)
* 重量级锁:如果上述的轻量级锁自旋一定次数之后仍然获取锁失败,便会升级称为重量级锁,使用重量级锁的时候锁将无法降级,这时候jvm提供使用快速获取锁的方式来实现,比如quick_enter/reenter/complete_exit,直接绕过slow-path的路径,jvm称重量级锁为heavy weight monitor

###### 2. synchronized加锁原理

> synchronized的enter加锁源码

```c++
// synchronizer.hpp
// This is the "slow path" version of monitor enter and exit.
// "slow path": 理解为缓慢路径,也就是jvm会通过当前方法检测对象是否处于竞争状态来确定锁的升级,以便于加快程序的性能(体现在响应时间和吞吐量)
static void enter(Handle obj, BasicLock* lock, TRAPS);

// 加锁具体实现:synchronizer.cpp
// 校验是否开启偏向锁,默认是开启偏向锁的设置
if (UseBiasedLocking) {
    // 撤销偏向锁操作
    if (!SafepointSynchronize::is_at_safepoint()) {
    // Java线程执行存在竞争,将当前的obj的markword设置为非偏向锁状态
      BiasedLocking::revoke(obj, THREAD);
    } else {
    // is_at_safepoint: 程序的所有Java用户线程都处于停止或者阻塞状态,除了在VM线程和本地执行的Java线程可执行
    // 阻塞所有Java线程,安全撤销偏向锁
      BiasedLocking::revoke_at_safepoint(obj);
    }
  }
 
  markWord mark = obj->mark();
  assert(!mark.has_bias_pattern(), "should not see bias pattern here");
  
  if (mark.is_neutral()) {
    // 设置为轻量级锁
    lock->set_displaced_header(mark);
    // CAS自旋锁,如果失败将升级为重量级别锁
    if (mark == obj()->cas_set_mark(markWord::from_pointer(lock), mark)) {
      return;
    }
  } else if (mark.has_locker() &&
             THREAD->is_lock_owned((address)mark.locker())) {
    // 锁升级失败,也就是不需要升级锁,表示当前线程已经获取锁了
    // 不需要再获取相同的锁,相当于重入锁/锁消除,直接设置对应的线程栈帧的displaced_mark_word为null
    assert(lock != mark.locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark().value(), "don't relock with same BasicLock");
    lock->set_displaced_header(markWord::from_pointer(NULL));
    return;
  }

  // unused_mark : it is only used to be stored into BasicLock as the indicator that the lock is using heavyweight monitor
  // 升级为重量级锁
  lock->set_displaced_header(markWord::unused_mark());
  // 锁升级的原因为: inflate_cause_monitor_enter,线程发生竞争争抢锁
  inflate(THREAD, obj(), inflate_cause_monitor_enter)->enter(THREAD);
```

> synchronized偏向锁的加锁与撤销流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200115103837344.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

> synchronized锁升级流程(轻量级锁升级到重量级锁过程)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200115102655297.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
 
###### 3.synchronized解锁过程

> synchronized的exit源码

```c++
// synchronizer.hpp
static void exit(oop obj, BasicLock* lock, Thread* THREAD);

// 具体实现:synchronized.cpp
void ObjectSynchronizer::exit(oop object, BasicLock* lock, TRAPS) {
  // 对于偏向锁的处理,撤销操作是加锁的过程,这里仅仅是对偏向锁存在的验证
  // 省略对应代码 .... 
  
  // 对markword的状态进行诊断判读,没有任何其他工作,
  // 省略代码 .....

	// 释放轻量级锁,本质上就是一个逆向恢复markword的过程
  if (mark == markWord::from_pointer(lock)) {
    assert(dhw.is_neutral(), "invariant");
    if (object->cas_set_mark(dhw, mark) == mark) {
      return;
    }
  }
  
  // 释放重量级锁,需要通过slow path的方式进行释放锁
  // We have to take the slow-path of possible inflation and then exit.
  inflate(THREAD, object, inflate_cause_vm_internal)->exit(true, THREAD);
}
```

> synchronized解锁的流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200115105113844.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

###### 4. 小结:JVM对synchronized的优化策略
> 锁优化目的

* 提升响应时间
* 增加程序的吞吐量
* 基于上述代码中,jvm底层代码存在缓慢路径加锁,说明也存在更快速地加锁方式,避免更多的加锁处理流程,提供锁升级的方式来走“**捷径**”调用加锁方法

> 优化手段

* 使用偏向锁,如果使用资源没有存在竞争状态,那么将开启偏向锁的方式进行加锁,通过上述可以看到偏向锁的流程,并无需消耗过多的资源,仅操作使用资源的markword信息,适用于单线程或者是并发量不多的场景下
* 轻量级锁:如果使用资源存在竞争状态(有一定的并发基础),那么jvm底层就会关闭偏向锁的设置,开启使用轻量级锁,通过将在线程已经开启Lock Record记录使用资源对象的markword信息,同时将markword的bitfields设置为栈帧引用地址,但是轻量级锁会出现CAS自旋消耗CPU资源,使用轻量级锁可以在并发条件下提升响应速度
* 重量级锁:线程不自旋转,不消耗CPU资源,jvm底层同时在锁升级之后会直接使用重量级锁对应的加锁和解锁方式,也就是走“捷径”方式调用,但是会阻塞线程
* 自旋锁: 可以看到这个是在使用CAS对使用资源进行循环compare and set的操作,主要是为了防止线程在操作系统底层产生阻塞,采用消耗CPU的方式来不断对使用资源进行CAS操作,但是会存在一些问题,一个是什么时候获取锁是一个未知数,在jvm底层会通过计数器来设置上限,达到上限之后会采用重量级锁的方式进行加锁;另一个就是ABA的问题(就是线程无法知道使用资源是否被变更,无法辨别前后的使用资源是否一致)
* 锁消除:在VM进行JIT编译时,会通过上下文的扫描来消除不存在共享资源竞争的锁,在上述的锁升级失败代码中,就是不允许对相同的锁进行加锁操作