---
title: Java并发线程之Lock应用
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->


##### 1. ReentrantLock的基本使用
> lock使用以及注意事项
```java
// task.java
public class Task {
    private int count;

    public void read(){
        System.out.println(Thread.currentThread().getName()+"读取count数据：" +  count);
    }

    public void write(){
        System.out.println(Thread.currentThread().getName()+"开始写count数据。。。");
        count++;
        System.out.println(Thread.currentThread().getName() +"结束写count数据。。。");
    }
}
	// main.java
	final ReentrantLock lock = new ReentrantLock();
	Task task = new Task();
	 for (int index = 0; index < 4; index++){
	     new Thread(){
	         @Override
	         public void run() {
	             lock.lock();
	             try{
	//                        dirtyCode();
	                 task.write();
	             }finally {
	                 lock.unlock();
	             }
	         }
	     }.start();
	 }
```
- 执行结果如下
```text
Thread-0开始写count数据。。。
Thread-0结束写count数据。。。
Thread-3开始写count数据。。。
Thread-3结束写count数据。。。
Thread-2开始写count数据。。。
Thread-2结束写count数据。。。
Thread-1开始写count数据。。。
Thread-1结束写count数据。。。
```

- 现在将代码变更如下
```java
// Task.java
private volatile boolean flag = false;
public void write(){
    System.out.println(Thread.currentThread().getName()+"开始写count数据。。。");
    count++;
    if (count == 3){
        flag = true;
    }
    System.out.println(Thread.currentThread().getName() +"结束写count数据。。。");
}

public boolean isFlag() {
    return flag;
}
//main.java
private static void dirtyCode(boolean flag){
    if (flag){
        int sum = 2 / 0;
    }
}
// 修改线程中的run方法
public void run() {
   try{
   		// lock.lock();  // 这里与上述代码加锁是一致的道理,这里为了演示效果
        dirtyCode(task.isFlag());
        lock.lock();		// 	可以看到我们是要缩小粒度加锁的优化
        task.write();
    } finally {
        lock.unlock();
    }
}
```
- 此时的执行结果
```text
Thread-0开始写count数据。。。
Thread-0结束写count数据。。。
Thread-1开始写count数据。。。
Thread-1结束写count数据。。。
Thread-2开始写count数据。。。
Thread-2结束写count数据。。。
Exception in thread "Thread-3" Exception in thread "Thread-4" java.lang.IllegalMonitorStateException
```
- 分析
	- 上述代码会抛出异常但是不是我们预期的算法异常,而是非法监视器状态异常
	- 从代码分析看,执行`dirtyCode`之后不会再执行加锁操作,但是一定会执行unlock操作,于是我们可以推测是调用unlock的时候出错
	- 查看源码
	```java
	if (Thread.currentThread() != getExclusiveOwnerThread())
	                throw new IllegalMonitorStateException();
	```
	- 可以看到原因就是我们当前并没有加锁,当前线程并没有持有独占锁,因此调用unlock会报错
	- 于是代码可以变更为
	```java
	run(){
		dirtyCode(task.isFlag());
		lock.lock();
		try{
			task.write();
		}finally{
			lock.unlock();
		}
	}
	```
	- `lock.lock()`比较好的做法是放在try块的外面而不是在里面

##### 2. 读写锁
- 读锁: 意味着允许并发多线程进行读取操作,但是不能执行写操作
- 写锁: 意味着仅能一个线程执行写操作,其他线程必须等待
- 代码示例
```java
// task.java
// 读写方法分别增加时间点输出并添加1s的等待操作
// main.java
 	ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    final ReentrantReadWriteLock.ReadLock readLock =  readWriteLock.readLock();
    final ReentrantReadWriteLock.WriteLock writeLock =  readWriteLock.writeLock();
    Task task = new Task();

    for (int index = 0; index < 10; index++){
        new Thread(){
            @Override
            public void run() {
                readLock.lock();
                try{
                    task.read();
                }finally {
                    readLock.unlock();
                }
            }
        }.start();

        new Thread(){
            @Override
            public void run() {
                writeLock.lock();
                try {
                    task.write();
                }finally {
                    writeLock.unlock();
                }
            }
        }.start();
    }
```
- 注释掉写操作的线程,执行读操作的线程,执行结果如下
```java
Thread-0开始读取count数据：0, 时间点：1582106214921
Thread-9开始读取count数据：0, 时间点：1582106214923
Thread-8开始读取count数据：0, 时间点：1582106214922
Thread-7开始读取count数据：0, 时间点：1582106214922
Thread-6开始读取count数据：0, 时间点：1582106214922
Thread-5开始读取count数据：0, 时间点：1582106214922
Thread-4开始读取count数据：0, 时间点：1582106214922
Thread-3开始读取count数据：0, 时间点：1582106214921
Thread-2开始读取count数据：0, 时间点：1582106214921
Thread-1开始读取count数据：0, 时间点：1582106214921
Thread-8完成读取count数据：0, 时间点：1582106215942
Thread-7完成读取count数据：0, 时间点：1582106215942
Thread-6完成读取count数据：0, 时间点：1582106215942
Thread-9完成读取count数据：0, 时间点：1582106215942
Thread-5完成读取count数据：0, 时间点：1582106215942
Thread-0完成读取count数据：0, 时间点：1582106215942
Thread-4完成读取count数据：0, 时间点：1582106215942
Thread-1完成读取count数据：0, 时间点：1582106215945
Thread-2完成读取count数据：0, 时间点：1582106215945
Thread-3完成读取count数据：0, 时间点：1582106215945
```
- 可以看出,上述读锁可以保证同一个时间点多个线程都持有读锁进行读操作
- 注释掉写操作的线程,查看输出结果
```java
Thread-0开始写count数, 时间点：1582106322255
Thread-0完成写count数, 时间点：1582106323279
Thread-1开始写count数, 时间点：1582106323279
Thread-1完成写count数, 时间点：1582106324281
Thread-2开始写count数, 时间点：1582106324282
Thread-2完成写count数, 时间点：1582106325287
Thread-3开始写count数, 时间点：1582106325288
Thread-3完成写count数, 时间点：1582106326290
```
- 上面可以看出写锁必须是等待当前一个线程执行完成才能释放锁给下一个线程,写锁只能一个线程持有

##### 3. 锁中断响应操作
> lock()与lockInterruptibly()方法比较
```java
private static void testInterrupt(Lock lock) throws InterruptedException {
    lock.lockInterruptibly();
//        lock.lock();
     try{
         System.out.println(Thread.currentThread().getName()+"获取到锁执行了。。。。");
         TimeUnit.SECONDS.sleep(5);
     }finally {
         lock.unlock();
         System.out.println(Thread.currentThread().getName()+"释放锁。。。。");
     }
 }
// main.java
	final ReentrantLock lock = new ReentrantLock();
    Thread t1 = new Thread("线程1"){
        @Override
        public void run() {
            try {
                testInterrupt(lock);
                System.out.println(Thread.currentThread().getName()+"执行其他操作。。。。");
            }catch (InterruptedException e){
                System.out.println(Thread.currentThread().getName() + "被中断，获取锁失败。。。");
            }
        }
    };

    Thread t2 = new Thread("线程2"){
        @Override
        public void run() {
            try {
                testInterrupt(lock);
                System.out.println("执行其他操作。。。。");
            }catch (InterruptedException e){
                System.out.println(Thread.currentThread().getName() + "被中断，获取锁失败。。。");
            }
        }
    };

    t1.start();
    TimeUnit.MICROSECONDS.sleep(1000);
    t2.start();

    TimeUnit.MICROSECONDS.sleep(3000);
    t2.interrupt();
```
- 使用lock()执行结果
```text
线程1获取到锁执行了。。。。
线程1释放锁。。。。
线程2获取到锁执行了。。。。
线程1执行其他操作。。。。
线程2释放锁。。。。
线程2被中断，获取锁失败。。。
```

- 使用lockInterruptibly()执行结果
```text
线程1获取到锁执行了。。。。
线程2被中断，获取锁失败。。。
线程1释放锁。。。。
线程1执行其他操作。。。。
```
- 分析,使用lock()的时候lock代码块仍然会执行,而主线程已经调用中断,说明没有响应到锁中断
- 对于使用lockInterruptibly()一旦收到主线程的中断操作,立马响应中断不会再继续往下执行

##### 4. 条件锁
使用典型的生产者-消费者模型
```java
	// 基于Condition的条件锁
   final ReentrantLock lock = new ReentrantLock();
   final Condition full = lock.newCondition();
   final Condition empty = lock.newCondition();
   final int size = 5;
   final List<String> list = new ArrayList<>();

   // 生产者
   new Thread(){
       @Override
       public void run() {
           while (true){
               lock.lock();
               try{
                   while (list.size() == size){
                       // 说明已经满了
                       System.out.println("数据已经满了。。。");
                       full.await();
                   }
                   list.add("aaa");
                   System.out.println("生产数据。。。。");
                   TimeUnit.SECONDS.sleep(1);
                   empty.signalAll();// 通知消费可以消费数据
               }catch (Exception e){

               } finally {
                   lock.unlock();
               }
           }
       }
   }.start();

   new Thread(){
       @Override
       public void run() {
           while (true){
               lock.lock();
               try{
                   while (list.size() == 0){
                       System.out.println("数据已经空了");
                       empty.await();//当前已经是空数据
                   }
                   list.remove("aaa");
                   System.out.println("消费数据。。。");
                   TimeUnit.SECONDS.sleep(1);
                   full.signalAll();   // 通知生产者生产数据
               }catch (Exception e){

               } finally {
                   lock.unlock();
               }
           }
       }
   }.start();
```
- 执行结果
```text
....
数据已经空了
生产数据。。。。
生产数据。。。。
生产数据。。。。
生产数据。。。。
生产数据。。。。
数据已经满了。。。
消费数据。。。
消费数据。。。
消费数据。。。
消费数据。。。
消费数据。。。
....
```