---
title: 线程核心方法简介
category: 并发编程
date: 2020-03-16 20:47:28
tags: 并发编程
---

<!-- more -->
##### 1. 线程终止
> 基于可见性的volatile实现

```java
// 定义任务线程
class VolatileTask {

    private volatile boolean flag = false;

    public void read() {
        while (!flag){
            System.out.println("query data ....");
        }
        System.out.println("query data done ....");
    }

    public void write(){
        flag = true;
        System.out.println("writing data done ....");
    }
}
// 执行main方法
 	final VolatileTask task = new VolatileTask();
    new Thread(new Runnable() {
         @Override
         public void run() {
             task.read();
         }
     }).start();

     TimeUnit.SECONDS.sleep(2);

     new Thread(new Runnable() {
         @Override
         public void run() {
             task.write();
         }
     }).start();
```
- 执行结果:
```text
// ...
query data ....
query data ....
query data ....
writing data done ....
query data done ....
```
- 分析
	- 上述执行的结果属于线程执行正常结束
	- 上述是在client模式下执行,没有执行重排序操作,在JMM规范中,volatile保持可见性,在字节码层面,volatile带有修饰符ACC_VOLATILE,在JVM规范中有声明为没有缓存,也就是直接从主内存读取数据,因此在另一个线程可以读取到flag的最新值

> stop方法以及存在的问题
```java
class Task implements Runnable {

    private int i = 0;
    private int j = 0;

    @Override
    public void run() {
        synchronized (this){
            ++i;
            try{
                TimeUnit.SECONDS.sleep(3);
            }catch (InterruptedException e){
                e.printStackTrace();
            }
            ++j;
        }
    }

    public void output(){
        System.out.printf("time=%s,i=%d, j=%d%n", System.currentTimeMillis(), i, j);
    }
}
// main方法
	Task task = new Task();
    Thread t1 = new Thread(task);
    t1.start();

     TimeUnit.SECONDS.sleep(1);
     t1.stop();

     task.output();
     while (t1.isAlive()){
     }
     
     // 保证线程已经终止
     task.output();
```
- 执行结果
```text
// 线程未终止
time=1582099251750,i=1, j=0
// 线程已终止
time=1582099251769,i=1, j=0
```
- 分析
	- 根据上述执行的结果,执行stop方法无法保证数据达到我们预期值,即j=1
	- 其次也可以看出stop方法调用执行之后对线程是无法预测的,也就是说线程在执行的过程中突然收到停止的方法“蒙圈”了,无法捕获到异常信息
	- 最后一点就是在同步代码中结束的时候,stop方法直接退出并没有将同步代码完全执行完

> 使用interrupt方法中断线程

```java
// task 代码不变
// main方法
	Task task = new Task();
    Thread t1 = new Thread(task);
     t1.start();

     TimeUnit.SECONDS.sleep(1);
     t1.interrupt();

     task.output();

     while (t1.isAlive()){

     }

     // 保证线程已经终止
     task.output();
```
- 执行结果
```text
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.xiaokunliu.homework.thread.base.methods.Task.run(ThreadStop.java:87)
	at java.lang.Thread.run(Thread.java:748)
time=1582099560454,i=1, j=0
time=1582099560475,i=1, j=1
```
- 分析
	- 可以看出,上述执行interrupt方法之后,会在线程中抛出中断异常,线程可以进行捕获处理
	- 其次中断操作能够保证同步操作中的代码能够被执行,从代码中也可以知道是因为可以处理中断异常,因此在实际开发过程中线程可以业务策略进行处理保证数据是正常可预期的

###### 2. 线程join方法
> join的场景

假设现在有一个业务场景是查询直播信息,而查询直播信息需要几个步骤完成,一个是获取用户的频道信息,二是查询是否在推荐直播位置中,三是该直播是否正在直播,上述三个接口分别是在基础组,应用组和直播组开发团队进行维护,这个时候为了完成需求,我需要通过接口的方式进行分别调用然后汇聚再返回,此时我们可以开三个线程,一个是处理用户频道查询,一个是查询是否在推荐位中,一个是查询是否正在直播的状态,于是有以下代码(伪代码)
```java
// main方法
Thread t1 = new Thread(){
	public void run(){
		// get user channel 
	}
};

Thread t2 = new Thread(){
	public void run(){
		// get user recommand lives
	}
};

Thread t3 = new Thread(){
	public void run(){
		// get user get live status
	}
};

t1.start();
t2.start();
t3.start();

// 这里是处理汇总的数据方式,因此必须等待上述执行完成
t1.join();
t2.join();
t3.join();
```

- 分析
	- join方法就是在当前线程调用join方法的线程中的任务执行完成之后再进行下一步的操作
	- 其次基于上述的应用场景,还可以使用CountDownLatch或者是Future/Callable抑或是join/fork框架实现

##### 3. 线程的yeild方法
> yeild定义与理解

yield是属于一个由static native 修饰的底层实现机制,它的作用是一个“不完全让出CPU资源的权利”来调度线程
**现在有一个应用场景: 假设执行一个耗时且不重要的A任务需要10s,而执行一个紧急不耗时的B任务需要1s,那么这个时候当启动A线程的时候,在A线程中调用yield()方法来告诉CPU说我当前执行的任务不是很重要但是比较耗时,可以适当让出CPU资源优先给其他紧急处理任务的线程执行**
**类比于一个生活场景就是去看医生,医生可能会在会诊一个普通病人的时候突然遇到重症病人,需要紧急优先处理的场景**

> yeild示例代码
```java
	Thread t1 = new Thread("线程1"){
            @Override
            public void run() {
                try{
                    // 执行完成需要2s
                    TimeUnit.SECONDS.sleep(2);
                    System.out.println(Thread.currentThread().getName() + "执行重要的事情");
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
        };


      Thread t2 = new Thread("线程2"){
           @Override
           public void run() {
               try{
                   // 让给执行重要的线程优先执行，当前为不重要且耗时操作
                   Thread.yield();
                   TimeUnit.SECONDS.sleep(12L);
                   System.out.println(Thread.currentThread().getName()+"执行不重要的事情。。。");
               }catch (Exception e){
                   e.printStackTrace();
               }
           }
       };

	  // 保证t2 先执行
       t2.start();
       TimeUnit.MILLISECONDS.sleep(500);
       t1.start();

       t2.join();
       t1.join();
       System.out.println("执行完成。。。");
```
- 执行结果如下
```text
线程1执行重要的事情
线程2执行不重要的事情。。。
执行完成。。。
```