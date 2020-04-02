---
title: 并发编程之伪共享
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->


###### 1. CPU高速缓存与伪共享
> CPU缓存与一致性

在[CPU高速缓存与内存屏障](https://blog.csdn.net/wind_602/article/details/103914263)的介绍中,CPU在对数据进行读取的时候遵循缓存一致性来解决高速缓存的数据不一致问题,现简述如下:

- CPU高速缓存包含L1-L3 Cache缓存,每个缓存Cache都是分段(line)存储的,也就是缓存段(Cache line)
- 根据缓存的一致性,多核CPU处理器情况下,当其中一个CPU对其所在的Cache进行写操作并通知其他CPU,这时候其他CPU便会令该缓存失效从而去读取主内存上的数据并将连续的变量地址的数据copy到缓存段中

> 伪共享定义以及产生原因

- 伪共享
	- 前提:在多核CPU处理器中,每个CPU都会有对应的缓存数据,并且缓存段缓存连续存储的数据,假设是在多核处理器上有L1和L2缓存,那么此时在L1和L2的缓存段上将会从主内存中拷贝一份连续内存地址的变量数据的一个副本,实现一次读主内存,后续多次读取缓存段的数据,**即CPU高速缓存是针对内存地址连续的数据变量实现一次写,多次读的效果**
	- 伪共享情景:当其中一个CPU对一个内存地址不连续的变量数据进行写操作的时候,由于CPU遵循缓存的一致性,那么此时另外的CPU需要对变量数据进行读操作,**由于变量数据内存地址不连续导致读取缓存段中不存在该数据而从主内存中加载,也就是说对数据变量写入缓存段的时候还没有来得及读取就失效了**
	- 产生的影响: 伪共享产生的结果就是CPU高速缓存不生效,没有命中缓存,同时缓存的一致性破坏了读取CPU一级缓存的原则,原因是在于并发线程进行写操作的时候会令CPU缓存失效,也会造成伪共享
	- 上述就是一个伪共享的现象,即在CPU多写的情况下,CPU高速缓存对内存地址不连续的数据变量并没有真正起到缓存的作用

- 读取数据伪共享代码演示

```java
// FalseShared.java
public class FalseShared {

    // 存储连续数据，同一行
    private final static int[][] arr2continuous = new int[1024][1024];
    // 存储不连续数据，同一列
    private final static int[][] arr2notcontinuos = new int[1024][1024];

    static {
        for (int index = 0; index < 1024; index ++){
            arr2continuous[0][index] = index * 2 + 1;
            arr2notcontinuos[index][0] = index * 2 + 1;
        }
    }

    private static void readByContinuous(){
        long start = System.currentTimeMillis();
        int len = arr2continuous.length;
        for (int row = 0; row < len; row ++){
            for (int col = 0; col < len;  col ++){
            	// 读取1数据的时候会先从内存加载并把与1连续的数据也一起加载到缓存,下次读取3的时候是从缓存读取
                long temp = arr2continuous[row][col];
            }
        }
        long end = System.currentTimeMillis();
        System.out.println("read arr by continuous with time : "+ (end - start));
    }


    private static void readByNotContinuous(){
        long start = System.currentTimeMillis();
        int len = arr2notcontinuos.length;
        for (int row = 0; row < len; row ++){
            for (int col = 0; col < len;  col ++){
            	// 读取1的时候并没有发现有连续数据的,因此只会copy数据1到缓存,也就是下次读取3的时候还要从
            	// 主内存中读取
                long temp = arr2notcontinuos[col][row];
            }
        }
        long end = System.currentTimeMillis();
        System.out.println("read arr by not continuous with time : "+ (end - start));
    }


    public static void main(String[] args) throws Exception {
        new Thread(){
            @Override
            public void run() {
                readByContinuous();
            }
        }.start();

        new Thread(){
            @Override
            public void run() {
                readByNotContinuous();
            }
        }.start();
    }
}
```

- 代码结果

```text
## case 1
read arr by continuous with time : 12
read arr by not continuous with time : 24

## case 2
read arr by continuous with time : 12
read arr by not continuous with time : 19
// ...
```

- 代码结果分析
	- 在上述演示的代码中,数组存储的数据都一样,区别在于数据分别存储在数据的同一行以及同一列
	- 在程序运行过程中,遍历获取同一列数组的数据执行的时间要慢于同一行的数组
	- 数组存储的数据在同一行上是数据连续的,同一列的数组数据不具备连续性
	- 存储连续性的数据有助于提升程序的执行效率,也就是响应时间
	- 不连续性的数据最终存储在不同的缓存段中

- 结果示意图演示(便于理解)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200129164400909.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 写数据产生伪共享

```java
// shared.java
int x = 10;
int y = 9;

// write thread
run(){
	x = 11;
}

// read thread
run(){
	TimeUnit.SECONDS.sleep(1L);
	int x1 = x;
	int temp = y;
}
```

- 分析
	- 根据缓存一致性原则,最后输出的x1必然与x=11的数据一致
	- 说明由于缓存一致性的原则,读取线程最终会去读取主内存更新后的x的数据,读取线程当前的缓存段缓存的x与y(数据内存连续性会缓存在同一个缓存段中)失效,说明这时候的y也是从主内存中读取数据,对于y而言,缓存没有命中,存在伪共享

###### 2. java解决伪共享的方案
> 使用`@sun.misc.Contended`解决伪共享的问题

- 修饰字段属性
```java
  // The following three initially uninitialized fields are exclusively
  // managed by class java.util.concurrent.ThreadLocalRandom. These
   // fields are used to build the high-performance PRNGs in the
   // concurrent code, and we can not risk accidental false sharing.
   // Hence, the fields are isolated with @Contended.

   /** The current seed for a ThreadLocalRandom */
   @sun.misc.Contended("tlr")
   long threadLocalRandomSeed;

   /** Probe hash value; nonzero if threadLocalRandomSeed initialized */
   @sun.misc.Contended("tlr")
   int threadLocalRandomProbe;

   /** Secondary seed isolated from public ThreadLocalRandom sequence */
   @sun.misc.Contended("tlr")
   int threadLocalRandomSecondarySeed;
```

- 修饰类

```java
@sun.misc.Contended
public class ForkJoinPool extends AbstractExecutorService {
	// 表示这个类下的属性内存地址在cache line都具备连续性
}
```

- 解决方案原理示意图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200129182306442.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 分析
	- 其实就是将不连续的数据变量通过注解的方式将它们存储到缓存段中连续紧密的区域中
	- 按照这个思路,也可以根据实际场景有针对性地对数据进行划分,比如热点数据以及冷数据,即可以根据数据的读写场景情况来分配变量在缓存段的存储位置,这样能够有效地提升CPU利用缓存的特点来提升响应速度

- 参考外链文档代码以及输出字段

```java
// source.java
 public static class ContendedTest1 {
    @Contended
      private Object contendedField1;
      private Object plainField1;
      private Object plainField2;
      private Object plainField3;
      private Object plainField4;
}
// 输出
  TestContended$ContendedTest1: field layout
     @ 12 --- instance fields start ---
     @ 12 "plainField1" Ljava.lang.Object;
     @ 16 "plainField2" Ljava.lang.Object;
     @ 20 "plainField3" Ljava.lang.Object;
     @ 24 "plainField4" Ljava.lang.Object;
     @156 "contendedField1" Ljava.lang.Object; (contended, group = 0)
     @288 --- instance fields end ---
     @288 --- instance ends ---
 
// 结果: 将contendedField1与其他字段分配的内存地址区分开,没有放在同一个cache line中(jdk默认cache line是128bit)
```

- 用户使用方式
	- 在对应类或者属性字段添加注解`@Contended`
	- 其次运行的时候需要设置VM参数为`-XX:-RestrictContended`
	- 如果需要设置cache line的宽度,可以通过`-XX:-RestrictContended -XX:ContendedPaddingWidth=256`设置生效

> 参考文档

```text
https://dzone.com/articles/what-false-sharing-is-and-how-jvm-prevents-it
http://mail.openjdk.java.net/pipermail/hotspot-dev/2012-November/007309.html
```
