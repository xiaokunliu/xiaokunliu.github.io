---
title: synchronized基于内存语义的工作原理(二)
category: 并发编程
date: 2020-03-04 20:47:28
tags: 并发编程
---

<!-- more -->
基于[**工作原理一**](https://blog.csdn.net/wind_602/article/details/103873050)可知同步关键字底层是基于JVM操作监视器的同步指令原语monitorenter和monitorexit来实现,这次将会通过抽象的内存语义来说明侧面说明加锁和解锁的方式
###### 1. 工作内存与主内存
> 定义
- 主内存: 一般就是计算机操作系统上的物理内存,简言之,即使一般我们所说的计算机的内存含义
- 工作内存: 基于JMM(Java内存模型)规范规定,线程使用的变量将会把主内存的数据变量复制到自己线程栈的工作空间

> 线程工作内存与主内存的读写示意图

**前面已经有介绍到[CPU高速缓存的知识点](https://blog.csdn.net/wind_602/article/details/103914263),以下是CPU简单的架构图以及工作内存与主内存的读写流程**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200111152758566.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
从上述我们可以看到,CPU中包含L1-L3的Cache,线程每次读写都需要先经过CPU高速缓存,这样便会产生数据缓存的不一致,前面已经有讲到CPU厂商针对这类问题做了改进,运用缓存一致性来达到最终数据的一致性,那么此时如果有一个需求是强一致性,即使是很短的时间内,我也需要保证写数据之后立马看到写数据成功后的效果,这时候怎么办呢?在JMM规范中为了解决这类内存共享的数据在不同线程不可见的问题,就制定一种规范来强制java程序中的线程直接跳过CPU高速缓存数据去读取主内存的数据,这就是解决内存数据的不可见的一种手段.

###### 2. synchronized的代码演示
- 场景： 现在有一个共享变量sharedVar，thread-1执行写操作需要耗时500ms，而有一个线程thread-2由于网络原因延迟读操作耗时600ms，另一个线程thread-3正常读操作
- 期望的场景是希望写数据之后其他线程也知道数据已经发生改变了,需要读取最新的数据
```java
// Sync2memory.java
public class Sync2memory {

    private static Integer sharedVar = 10;

    public static void main(String[] args) throws Exception {
        testForReadWrite();
//        testForReadWriteWithSync();
 		TimeUnit.SECONDS.sleep(2L);
        System.out.printf("finish the thread task,the final sharedVar %s ....\n", sharedVar);
    }

    private static void testForReadWriteWithSync() throws Exception {
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                	// modify the sharedVar
                    TimeUnit.MICROSECONDS.sleep(500L);
                    synchronized (sharedVar){
                        System.out.printf("%s modify the shared var ...\n", "thread-1");
                        sharedVar = 20;
                    }
                }catch (Exception e){
                    System.out.println(e);
                }
            }
        });


        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                   // network delay
                   TimeUnit.MICROSECONDS.sleep(600L);
                   synchronized (sharedVar){
                       System.out.printf("%s read the shared var %s \n", "thread-2", sharedVar);
                   }
                }catch (Exception e){
                    System.out.println(e);
                }
            }
        });

        Thread thread3 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    synchronized (sharedVar){
                        System.out.printf("%s read the shared var %s \n",  "thread-3", sharedVar);
                    }
                }catch (Exception e){
                    System.out.println(e);
                }
            }
        });

        thread2.start();
        thread3.start();
        thread1.start();

        thread1.join();
        thread2.join();
        thread3.join();
    }

    private static void testForReadWrite() throws Exception {
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                	// modify the sharedVar
                    TimeUnit.MICROSECONDS.sleep(500L);
                    System.out.printf("%s modify the shared var ...\n", "thread-1");
                    sharedVar = 20;
                }catch (Exception e){
                    System.out.println(e);
                }
            }
        });


        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                	// network delay
                    TimeUnit.MICROSECONDS.sleep(600L);
                    System.out.printf("%s read the shared var %s \n", "thread-2", sharedVar);
                }catch (Exception e){
                    System.out.println(e);
                }
            }
        });

        Thread thread3 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.printf("%s read the shared var %s \n", "thread-3" , sharedVar);
                }catch (Exception e){
                    System.out.println(e);
                }
            }
        });

       //thread1-3 start and join ....
    }
}
```
- 没有加synchronized方式的执行产生的一种结果(多次运行)
```text
## 	执行结果如下
thread-3 read the shared var 10 
thread-1 modify the shared var to 20  ...
thread-2 read the shared var 10 
finish the thread task,the final sharedVar 20 ....

Process finished with exit code 0

## 分析
线程3正常执行,并且还没有在发生写操作之前就已经读取数据,属于正常输出
线程1执行写操作耗时500ms并将数据进行修改同步到主内存中
线程2由于网络延迟600ms,但是此时写操作已经完成,这时候读取出来的数据是属于脏数据,并不正确,因此线程2读取是其还没有被刷新的工作内存数据
最后看到执行的结果输出是写操作之后的数据,说明了CPU最终会保证缓存数据的一致性
最后的最后,这里仅仅是阐述上述问题,上述运行结果也可能发生thread-2会读取到正常的数据,只是在上述编码情况我们是无法保证线程2一定可以读取到正确的数据
```
- 添加synchronized方式的执行结果(多次执行)
```text
## 多次执行结果如下:
thread-3 read the shared var 10 
thread-1 modify the shared var ...
thread-2 read the shared var 20 
finish the thread task,the final sharedVar 20 ....

Process finished with exit code 0

## 分析
线程1执行写操作之后,我们可以看到线程2获取到的数据是线程1执行写操作之后的数据,现在程序可以保证线程2读取的数据是正常的
```

###### 3. synchronized内存语义的理解
> 内存语义小结
- 基于上述代码的执行结果可以看出,我们使用synchronized内使用的变量从线程的工作内存中清除或者称为失效,此时该变量就不会从工作内存中进行读取数据,而是直接从主内存中读取数据,从而保证缓存数据的强一致性
- 由此可知道,synchronized从内存语义上可以解决共享变量的内存可见性问题
- 从另一个角度而言,使用synchronized相当于jvm获取monitorenter的指令,此时会将该共享变量的缓存失效直接从主内存中加载数据到锁块的内存中,同时在进行monitorexit操作的指令时会将锁块的共享变量数据刷新到主内存中
> synchronized不足
- 使用monitor的方式是属于metux lock的方式(重量级锁),会降低程序的性能(响应时间可能会变慢,相当于利用性能来换取数据的强一致性问题)
- 另外一个就是线程是由CPU进行调度,来回切换线程会带来额外的调度开销




