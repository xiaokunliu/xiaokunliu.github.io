---
title: 基于AQS原理实现的锁
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->

###### 1. AQS概念及其组件
> AQS基础概念及其作用

- AQS基础概念
	- AQS: 即抽象队列同步器,AbstractQueuedSynchronizer
	- AQS之shared mode: 即共享锁/读锁,用于线程读取加锁,不能进行写操作,可以读读共享
	- AQS之exclusive mode: 即独占锁/排他锁/写锁,用于线程原子写操作时加锁,只能一个线程持有,其他线程处于等待状态
	- AQS中不同mode的线程共享相同的等待队列wait queue,也就是在同一个阻塞队列中,线程持有的mode可能会不同
	- state属性: 作为AQS的同步状态信息属性,state具备线程安全特性(valatile & CAS分别保证可见性和原子性)

- AQS主要作用
	- 提供一个基于FIFO等待队列的阻塞锁和相关同步器的模板框架,即AQS
	- 对于阻塞锁和同步器的实现子类,必须定义一个非对外访问的`helper class`来继承AQS,利用AQS中受保护的方法来为阻塞锁和同步器对外暴露的方法提供服务
	- 继承AQS的同步器子类将通过模板框架提供的CAS操作state方式来保证原子性,以及volatile修饰保证可见性,这样能够实时知道当前对象获取锁或者释放锁所处的状态信息
	- 一般情况下,子类只会实现上述两种mode之一,但是对于ReadWriteLock具备上述两种mode,这也就是ReadWriteLock具备读写锁的特征
	- AQS内部定义一个实现Condition接口的实现内部类ConditionObject,主要作用是**结合独占模式下的方法一同使用,也就是说在并发线程持有相同的独占锁情况下,独占资源下的方法可以结合Condition下的唤醒与挂起线程的方式完成线程之间的通信**(独占资源方法有, 比如isHeldExclusively判断当前线程是否持有独占锁, release释放独占锁, acquire获取独占锁)

> AQS包含的组件

- Node: 自定义双端链表,实现同步队列等待池,其包含的核心要素有
	- 指向前一个节点Node的prev
	- 指向后一个节点Node的next
	- 节点所处状态信息,即waiter status
	- 指向下一个处于condition的等待队列的节点nextWaiter
```java
// Node 部分代码,在AQS内部中定义
static final class Node{
	static final Node SHARED = new Node(); 	// 表示持有共享锁
	static final Node EXCLUSIVE = null;		// 持有独占锁
	// waiter status,即节点所处的阻塞状态列表如下
	static final int CANCELLED =  1;		// 被取消,意味着放弃竞争锁资源,移出阻塞队列
	static final int SIGNAL    = -1;		// 持有锁状态
	static final int CONDITION = -2;		// 线程处于条件等待队列中,也就是condition.await让线程挂起
	static final int PROPAGATE = -3;		// 释放锁并正在处于通知其他等待节点可以竞争锁资源的状态
	
	volatile int waitStatus;	// 当前节点的状态,初始化为0,不属于上述任何一种状态,属于非阻塞可竞争获取锁的状态
	
	// 实现双端链表
	volatile Node prev;
	volatile Node next;
	
	// 当前节点的线程
	volatile Thread thread;
	
	// 标志当前节点是共享锁还是独占锁,用节点指针引用指向对应的mode
	Node nextWaiter;
	
	// 独占锁:nextWaiter = null & thread = Thread.currentThread;
	// 共享锁:nextWaiter = SHARED & thread = Thread.currentThread;
	// 不具备上述条件属于正常对象,不持有锁状态
	 Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
     }

     Node(Thread thread, int waitStatus) { // Used by Condition
         this.waitStatus = waitStatus;
         this.thread = thread;
     }
	
	// 判断是否为共享锁的方法
	boolean isShared(){
		// ...
	}
	// 获取上一个节点Node
	Node predecessor(){
		// ...
	}
}
```
- ConditionObject: 自定义接口Condition的实现,业务线程可以创建对应的锁条件来完成线程之间的通信.即线程唤醒与等待,其核心要素如下:
	- 自定义条件队列,即使拥有属性firstWaiter与lastWaiter
	- 实现接口await/signal/signalAll等方法
	- ConditionObject内部实现的核心方法
```java
// ConditionObject部分代码,在AQS内部中定义
public class ConditionObject implements Condition,Serializable {
	// 定义队列的首尾节点,内部自定义队列实现wait queue的操作
    private transient Node firstWaiter;
    private transient Node lastWaiter;
	
    private static final int REINTERRUPT =  1;		//  调用Thread.interrupt()中断当前线程
    private static final int THROW_IE    = -1;		// 	直接向当前线程抛出中断异常
	
	// implements Condition methods ...
	// 调用await,当前线程挂起
	await(){}	
	// 调用signal,当前线程被唤醒	
	signal(){}		
	signalAll(){}
	// 挂起/唤醒超时方法 ...
	
	// 内部核心方法
	// 添加一个新的节点到当前条件阻塞队列中
	private Node addConditionWaiter() {
		// ...
	}
	
	// 对于已经cancelled的waiter移出队列
	private void unlinkCancelledWaiters() {
		// ...
	}

	// 唤醒持有相同的Condition的线程去争抢资源锁,获取到锁并通过CAS设置对应节点的waiter status为SIGNAL
	private void doSignalAll(Node first) {
		// ...
	}
}
```

- AQS属性及其依赖
	- state,即同步状态,在对应的实现类中表达含义不同,AQS中称为一个同步状态值信息
	- exclusiveOwnerThread,即当前线程持有独占锁,独占线程
	- Unsafe,即借助CAS技术来完成同步状态以及等待池队列节点的更新操作等
	- LockSupport工具类,借助LockSupport来完成线程的加锁与解锁操作
```java
// AQS.java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
	
	// 自定义实现双向FIFO的队列, 定义队列的head & tail
	private transient volatile Node head;
	private transient volatile Node tail;
	private volatile int state;				// 当前AQS同步状态信息,具体实现类含义可能不一样
	// CAS操作上述的state,head&tail, waitstuts
	private static final Unsafe unsafe = Unsafe.getUnsafe();	
	
	// 除了获取/释放锁之外,内部定义核心方法
	private Node enq(final Node node) {}	// 	将Node插入队列
	private Node addWaiter(Node mode) {}	//  创建想要持有mode的锁方式的Node并将其添加到阻塞队列中
	
	// 内部实现的独占锁方式
	final boolean acquireQueued(final Node node, int arg) {}	// 获取独占锁,失败则挂起
	private void unparkSuccessor(Node node) {}	// 唤醒当前Node的下一个节点的线程
	
	// 内部实现共享锁
	private void doReleaseShared() {}			// 唤醒下一个节点的线程并通知其他节点的线程已经释放锁
	private void doAcquireShared(int arg) {}	// 如果获取锁失败则挂起线程,成功则持有锁
}
```

- AQS的类图明细
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200204152205143.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- AQS的使用,也就是子类继承AQS之后,根据需要重写AQS中定义的`protected`方法,从上述类图可以看出:
	- tryAcquire: 获取独占锁
	- tryRelease: 释放独占锁
	- isHeldExclusively: 判断当前线程是否持有独占锁
	- tryAcquireShared: 获取共享锁
	- tryReleaseShared: 释放共享锁
	- 操作和获取属性state的getState()以及CAS修改state的方法
	- 操作线程独占方法get&set方法
	- 如果锁需要Condition,则可以直接使用ConditionObject来实现对应的分类锁逻辑实现线程通信(可选)

###### 2.AQS 工作原理
> AQS 核心组成

- 内部核心主体
	- state: AQS中表达为一个同步状态信息,具体实现类含义不一样,比如CountDownLatch的state表示计数器
	- exclusiveOwnerThread:独占线程对象
	- 双向队列等待池(Node head & Node tail): 基于内部类属性成员的链表head以及链表tail
- 内部核心接口
	- 独占资源接口,设置为独占锁
		- 模板方法: acquire/release
		- 具体实现方法: tryAcquire/tryRelease, 
	- 共享资源接口,设置为共享锁
		- 模板方法: acquireShared/releaseShared, 
		- 具体实现方法: tryAcquireShared/tryReleaseShared
	-  acquire/acquireShared定义资源争用逻辑,没有拿到则加入队列等待池中等待
	-  tryAcquire/tryAcquireShared实际执行占用资源的操作,交由具体实现的AQS完成
	- release/releaseShared定义释放资源的逻辑,释放之后通知等待池中的节点进行争抢
	- tryRelease/tryReleaseShared实际释放资源的操作,由具体实现的AQS完成

- AQS 加解锁核心代码

```java
// 独占锁
 public final void acquire(int arg) {
     if (!tryAcquire(arg) &&
          acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
          selfInterrupt();
 }

 public final boolean release(int arg) {
     if (tryRelease(arg)) {
         Node h = head;
         if (h != null && h.waitStatus != 0)
             unparkSuccessor(h);
         return true;
     }
     return false;
 }

// 共享锁
 public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
 }

 public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

// 初始化阻塞队列方法
private Node enq(final Node node) {
   for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize, 可以看到初始化的head不持有锁,属于正常状态
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

// 加入阻塞队列
private Node addWaiter(Node mode) {
   Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

> AQS 加锁与解锁的流程

- 共享锁加锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200204222436498.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 共享锁解锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200204211918409.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 独占锁加锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200204215201519.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 独占锁解锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200204220132343.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 根据上述流程总结如下
	- AQS中加锁以及解锁的过程中是根据wait status来判断是否进行加锁和释放锁,wait status可理解为AQS中的“锁”,通过CAS更新wait status的状态来实现加锁和解锁,同时为了防止并发多线程再次修改,针对已经恢复正常的节点信息,通过`PROPAGATE`来控制,也就是拥有这个状态的wait status说明已经释放锁并唤醒其他线程争抢锁
	- AQS中通过链表的方式实现双向队列的操作,同时为了避免并发造成存储在链表中的节点顺序错乱,于是通过CAS的方式来设置链表的head和tail,保证修改的安全性
	- AQS中区分mode在于存储节点的nextWaiter的指向,在AQS中,共享锁和独占锁在加锁过程中,共享锁加锁后需要唤醒下一个mode为SHARED的阻塞节点线程,而独占锁不需要;在共享锁和独占锁解锁过程中,共享锁为了避免并发线程多次更新,于是通过`PROPAGATE`来控制,告知节点已经释放锁并通知其他线程
	- 可以看到,加锁和解锁可以通过走“捷径”的方式来完成,也就是对应的AQS实现相应的加锁和解锁处理逻辑
	- 总结,**加锁流程:当前节点先加入阻塞队列,CAS自旋获取锁,成功则删除当前节点(重置为head);解锁流程就是从阻塞队列中获取下一个阻塞节点,并唤醒当前节点的线程**

> AQS 原理总结

- AQS的核心组成部分
	- 具备线程安全的双向阻塞队列
	- 具备线程安全的独占线程
	- 具备线程安全的状态state属性

- 基于上述的组成部分自定义AQS伪代码实现

```java
// DefineAQS.java
public abstract class DefineAQS {
	final static class AQSNode{
        final static int SHARED = 9999;
        final static int EXCLUSIVE = -9999;

        private int mode;
        private volatile Thread thread;

        public AQSNode(int mode){
            this.thread = Thread.currentThread();
            this.mode = mode;
        }

        public Thread getThread() {
            return thread;
        }

        public int getMode() {
            return mode;
        }
    }

    private AtomicInteger state = null;
    private AtomicReference<Thread> exclusiveOwnerThread = new AtomicReference<>();
    private LinkedBlockingQueue<AQSNode> waiters = new LinkedBlockingQueue<>();


    public AtomicInteger getState() {
        return state;
    }

    public void setState(int state) {
        this.state = new AtomicInteger(state);
    }

    public void compareAndSetState(int expect, int update){
        this.state.compareAndSet(expect, update);
    }

    public void acquire(int arg){
        // 加入阻塞队列中
        AQSNode node = new AQSNode(AQSNode.EXCLUSIVE);
        waiters.offer(node);
        while (!tryAcquire(arg)){
            LockSupport.park(node.getThread());
        }
        // 当前线程已经获取到锁，移出阻塞队列，通知后续节点
        waiters.remove(node);
    }

    public void release(int arg){
        if (tryRelease(arg)) {
            while (true){
                AQSNode node = waiters.peek();
                if (node.getMode() == AQSNode.EXCLUSIVE){
                    LockSupport.unpark(node.getThread());
                    break;
                }
            }
        }
    }

    public void acquireShared(int arg){
        AQSNode node = new AQSNode(AQSNode.SHARED);
        waiters.offer(node);
        while (tryAcquireShared(arg) < 0){
            LockSupport.park(node.getThread());
        }
        waiters.remove(node);
    }

    public void releaseShared(int arg){
        if (tryReleaseShared(arg) > 0){
            while (true){
                AQSNode node = waiters.peek();
                if (node.getMode() == AQSNode.SHARED){
                    LockSupport.unpark(node.getThread());
                    break;
                }
            }
        }
    }
    // abstract method for tryXXX 
}
```




