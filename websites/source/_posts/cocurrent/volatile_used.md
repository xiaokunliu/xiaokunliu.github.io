---
title: 关键字volatile的使用与原子性问题
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->


###### 1. volatile的使用
> java源代码

```java
public class VolatileUsedClass {

    private static int sharedVar = 10;

    public static void main(String[] args) throws Exception {

        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    // modify the sharedVar,write first
                    TimeUnit.MICROSECONDS.sleep(500L);
                    sharedVar = 20;
                    System.out.printf("%s modify the shared var to %s  ...\n", "thread-1", sharedVar);
                } catch (Exception e) {
                    System.out.println(e);
                }
            }
        });


        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    // read the value,read for the last
                    TimeUnit.MICROSECONDS.sleep(505L);
                    System.out.printf("%s read the shared var %s \n", "thread-2", sharedVar);
                } catch (Exception e) {
                    System.out.println(e);
                }
            }
        });

        thread2.start();
        thread1.start();

        thread1.join();
        thread2.join();

        System.out.println("finish the thread task...");
    }
}
```
> 客户端模式-client

- 不加volatile的执行结果(多次执行)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020012210205840.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200122102554675.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 加volatile的执行结果(多次)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200122102337750.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)![在这里插入图片描述](https://img-blog.csdnimg.cn/20200122102504544.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 结果分析
	- 在源代码中是添加时间休眠主要是保证先写后读的逻辑
	- 从运行结果可以看出,虽然时间片很短,读线程的数据仍然是本地缓存的数据,并没有从主内存中读取值
	- 添加volatile关键字之后,可以看到读线程的数据正是写线程之后的数据,也就是写读数据是一致的

> 服务端模式,-server

- 不带volatile执行结果(多次)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200122103655136.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020012210402446.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 带volatile执行结果(多次)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200122104120365.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 结果分析
	- 不带volatile修饰与客户端执行效果一致
	- 但是使用server模式带有volatile的方式却出现了数据不一致的情况,为什么?
	- 原因在于-server模式会在编译成字节码的时候进行代码重排序导致的,主要用于优化程序提升执行效率

> 小结

- clinet模式是jdk执行的默认配置,可用于测试环境或者本地开发
- server模式一般用于生产环境,目的是开启server模式的时候编译器会针对OS系统情况做一些优化操作
- 参考: [JVM server vs Client Mode](https://javapapers.com/core-java/jvm-server-vs-client-mode/)

###### 2. 原子性问题
**说明: 以下运行环境是使用-client模式进行,排除重排序的干扰**
> Java中的原子性

- 参考文档: https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html
- 文档对于原子性的说明如下:
	- 除了long和double类型之外,引用变量与大多数的原始数据类型都具备读写操作的原子性
	- 所有使用volatile修饰的变量都具备读写操作的原子性
- 分析
	- 针对64bit的数据类型,主要与处理器(32bit/64bit)有关,在32bit处理器上,JVM会将64bit的long/double划分为两个32bit的写操作,并不具备原子性(数据的读写主要是通过处理器总线与主内存进行传递)
	- 基于Happen-Before原则,对于volatile的变量读取总是可以“看到”任何一个线程对该volatile变量的最后写入,因此在临界区代码的执行是具备原子性,即使是long或是double类型

> volatile修饰单个变量的自增或自减是不具备原子性

- 代码
```java
// 部分代码,在上述的写线程进行修改
Thread t1 = new Thread(){
            @Override
            public void run() {
                try {
                    // modify the sharedVar
                    TimeUnit.MICROSECONDS.sleep(500L);
                    sharedVar ++;
                    System.out.printf("%s modify the shared var with atomic %s  ...\n", "thread-1", sharedVar);
                } catch (Exception e) {
                    System.out.println(e);
                }
            }
        };
```
- 运行结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200122115720973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 字节码显示
	- sharedVar = 20L显示的字节码如下:
	
	```text
	 L3
     LINENUMBER 102 L3
     BIPUSH 20		// 压入线程的操作数栈
     INVOKESTATIC com/xiaokunliu/blogs/thread/volatile2code/VolatileUsedClass.access$002 (I)I    //实例化并加载sharedVar并压入操作数栈,说明完成赋值操作
     POP   // 弹出数据
     L4
	```
	- sharedVar ++ 显示的字节码如下:
	
	```text
	 L3
     LINENUMBER 27 L3
     INVOKESTATIC com/xiaokunliu/blogs/thread/volatile2code/VolatileUsedClass.access$000 ()I  // 实例化并加载sharedVar并压入操作数栈
     ICONST_1    // 将常量 1 压入线程操作数栈总
     IADD            // 执行sharedVar+1
     INVOKESTATIC com/xiaokunliu/blogs/thread/volatile2code/VolatileUsedClass.access$002 (I)I  // 重新实力化加载sharedVar,说明完成赋值操作
     POP
    L4
	```
- 运行分析: 
	- 通过字节码可知,sharedVar ++;相比单纯赋值操作增加了一个添加的动作
	- 也就整体代码块存在在并发多线程下交替执行的两个操作,不具备原子性,volatile在这里是保证代码刷新到主内存,对于sharedVar = const 是具备原子性的

> 使用volatile小结

- 对变量进行写操作的时候可以通过volatile来实现对其他线程的可见,同样在单步指令操作中是具备原子性,针对long或者double也起到具备原子性的作用
- 对于需要用volatile修饰的变量来完成一系列的非单步操作运算是无法保证原子性,必须借助lock的方式来实现代码块的原子性