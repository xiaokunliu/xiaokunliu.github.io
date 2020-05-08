---
title: Java线程synchronized使用
category: 并发编程
date: 2020-03-12 20:47:28
tags: 并发编程
---

<!-- more -->


###### 1. synchronized 同步方法
* 作用在实例化方法上,监视器锁对象为当前实例对象this
* 作用在静态方法上,监视器锁对象为当前Class对象
* 不论线程执行同步代码正常完成抑或是异常退出的时候,将会自动释放监视器锁
* 同步实例方法产生的效果:
```text
1. 控制当前方法只能有一个线程执行，其他线程只能处于阻塞状态
2. 换言之,每个使用synchronized关键字声明的方法都是处于一个临界区，而Java只允许执行对象的一个临界区
```
* 同步静态方法产生的效果
```text
1. 静态方法同步仅保证声明为static且使用的监视器锁为当前类对象时只有一个线程执行,其他线程处于阻塞状态
2. 需要注意的一点是,对于共享资源,如果同时存在于同一个类声明的static和非static的方法,将无法保证共享数据的安全性,因为实例方法和静态方法的监视器锁对象不同,无法达到预期效果
```
* 实例方法示例(主要演示同步方法)
```java
// Account.java
public class Account {
	private double balance;

    public double getBalance() {
        return balance;
    }

    public void setBalance(double balance) {
        this.balance = balance;
    }
    
    public synchronized void addAmount(double amount) {
        double tmp = balance;
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        tmp+=amount;
        balance=tmp;
    }

    public synchronized void subtractAmount(double amount) {
        double tmp=balance;
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        tmp-=amount;
        balance=tmp;
    }
}

// Bank.java
// 实现Runnable接口,run方法负责循环100次调用subtractAmount方法

// Company.java
// 实现Runnable接口,run方法负责循环100次调用addAmount方法

// main.java
public static void main(String[] args) {
        Account account = new Account();
        account.setBalance(1000);

        Company company = new Company(account);
        Thread companyThread = new Thread(company);

        Bank bank = new Bank(account);
        Thread bankThread = new Thread(bank);

        System.out.printf("Account : Initial Balance: %f\n",account.getBalance());
//        Start the threads
        companyThread.start();
        bankThread.start();
        try {
            // 开始进行转账操作，使用join，也就是等待join的线程执行完成之后才进行下一步操作
            companyThread.join();
            bankThread.join();
            // 结束转账操作可以进行下一步的操作，
            System.out.printf("Account : Final Balance: %f\n",account.getBalance());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```
* 执行结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200107100012147.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
* 扩展
```java
// 上述的同步代码也可以替换为
public void addAmount(){
	// not shared data operation
	synchronized(this){
		// balance operation
	}
	// not shared data operation
}
```

###### 2. synchronized同步代码块
* 作用在同步代码块上,做到尽可能在方法中仅针对共享变量进行同步加锁的操作
* 同步块的锁监视器对象可以为this/当前Class的对象/类实例属性/类对象属性等等,只需要保证在同步代码块的监视器是唯一的即可
* 不论程序正常执行还是异常退出将会自动释放监视器锁
* 代码示例(主要演示同步块)
```java
// Cinema.java
public class Cinema {

    private long vacanciesCinema1;
    private long vacanciesCinema2;

    // 通过锁类实例的变量
    private final Object controlCinema1, controlCinema2;

    public Cinema(){
        this.controlCinema1 = new Object();
        this.controlCinema2 = new Object();
        this.vacanciesCinema1 = 20L;
        this.vacanciesCinema2 = 20L;
    }

    public boolean sellTickets1(int number){
        synchronized (controlCinema1){
            System.out.println(Thread.currentThread().getName() + " get controlCinema1 lock, execute sellTickets1 vacanciesCinema1 number " + number);
            if (number < this.vacanciesCinema1){
                this.vacanciesCinema1 -= number;
                return true;
            }
            return false;
        }
    }

    public boolean sellTickets2(int number){
        synchronized (controlCinema2){
            System.out.println(Thread.currentThread().getName() + " get controlCinema2 lock, execute sellTickets2 vacanciesCinema2 number " + number);
            if (number < this.vacanciesCinema2){
                this.vacanciesCinema2 -= number;
                return true;
            }
            return false;
        }
    }

	// 仅对应sellTickets1有同步效果
    public void addTickets1(int number){
        synchronized (controlCinema1){
            System.out.println(Thread.currentThread().getName() + " get controlCinema1 lock, add number is " + number);
            this.vacanciesCinema1 += number;
        }
    }

	// 仅对应sellTickets2有同步效果
    public void addTickets2(int number){
        synchronized (controlCinema2){
            System.out.println(Thread.currentThread().getName() + " get controlCinema2 lock, add number is " + number);
            this.vacanciesCinema2 += number;
        }
    }

    // get方法....
}

// 声明TicketOffice实现Runnable接口,run方法随机调用上述带有synchronized关键字的方法
// 代码省略 ...

// 测试main方法
public class TicketMain {

    public static void main(String[] args) throws Exception{

        Cinema cinema = new Cinema();

        // 由于操作的锁分别只针对vacanciesCinema1, vacanciesCinema2
        // 获取controlCinema1 lock 只能操作vacanciesCinema1
        // 获取controlCinema2 lock 只能操作vacanciesCinema2
        TicketOffice ticketOffice1 = new TicketOffice(cinema);
        Thread thread1 = new Thread(ticketOffice1);

        TicketOffice2 ticketOffice2 = new TicketOffice2(cinema);
        Thread thread2 = new Thread(ticketOffice2);

        thread1.start();
        thread2.start();

        // 等待线程1和线程2 执行完成，因此需要加入join方法
        thread1.join();
        thread2.join();

        TimeUnit.SECONDS.sleep(2);
        System.out.println("room1 number is " + cinema.getVacanciesCinema1());
        System.out.println("room2 number is " + cinema.getVacanciesCinema2());
    }
}
```

###### 3. 线程通信

> 与线程wait()/notify()/notifyAll()的使用

* q1: 为什么需要使用监视器锁来完成线程通信?
* q2: 代码中使用while而不使用if的原因是什么?
* q3: wait与notify在代码中分别起到的作用是什么?
* 代码示例
```java
// EventStorage.java
ublic class EventStorage {

    // TODO write your logic code
    private int maxSize;

    private LinkedList<Date> storage;

    public EventStorage(){
        this.maxSize = 10;
        storage = new LinkedList<>();
    }

    public synchronized void set(){
    	// 思考:使用while而不使用if的原因是什么?
        while (storage.size() == maxSize){
            try {
                wait();
            }catch (InterruptedException e){

            }
        }
        storage.offer(new Date());
        System.out.printf("Set: %d \n", storage.size());
        notifyAll();
    }

    public synchronized void get(){
    	//// 思考:使用while而不使用if的原因是什么?
        while (storage.size() == 0){
            try {
                wait();
            }catch (InterruptedException e){

            }
        }
        System.out.printf("Get size: %d, get value: %s \n", storage.size() , storage.poll());
        notifyAll();
    }

// Consumer.java
// 负责循环100次调用EventStorage的get方法

// Producer.java
// 负责循环100次调用EventStorage的set方法

// main.java
public class SyncMain {

    // TODO write your logic code

    public static void main(String[] args) {
        EventStorage eventStorage = new EventStorage();

        Thread producerThread = new Thread(new Producer(eventStorage));
        Thread consumerThread = new Thread(new Consumer(eventStorage));


        producerThread.start();
        consumerThread.start();

        producerThread.join();
        consumerThread.join();
        System.out.println("done !!! ");
    }
}
```
> **正常执行结果**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200107102203945.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

> **q1: 需要使用监视器锁完成通信原因**

```java
// 将上述的代码的synchronized去掉
// 执行main方法
// 执行结果如下
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108105104581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
**结论: 调用wait方法之前需要通过synchronized来添加监视器锁完成线程通信,否则会抛出异常:Exception in thread "Thread-1" java.lang.IllegalMonitorStateException**

> **q2: 代码中使用while而不使用if的原因是什么?**

```java
// 将while更改为if
// 执行main方法
// 执行结果如下
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108111809398.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200108111843366.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
* 原因分析
	* 从代码中我们想要的效果是当队列满了之后才通知消费者进行消费,队列为空的时候才让生产者进行生产数据,但是上述结果并没有等待队列全满就开始进行消费
	* 从执行结果看,使用if一次性条件判断无法保证队列完全为空或者为满的情况才进行消费,因此需要在调用wait方法的时候需在while循环中不断检查队列的情况

> q3: 针对上述情况对wait以及notify进行小结

* wait方法
	* 当前线程是通过共享变量调用wait方法,并且调用该线程的时候将会被挂起阻塞
	* 其他线程调用该共享变量的notify/notifyAll方法时将有机会重新获取锁
	* 同时如果其他线程执行过程中调用当前线程的中断方法也会退出有机会重新获取锁
	* wait方法调用之前必须先获得监视器进行加锁,否则会抛出异常错误
* notify方法
	* notify调用必须是在wait方法之后调用,因为只有当线程获取到监视器锁之后才可以调用notify进行唤醒
	* 调用notify唤醒的线程需要和其他线程进行竞争获取监视器锁,之后才能返回调用的wait方法并继续往下执行
