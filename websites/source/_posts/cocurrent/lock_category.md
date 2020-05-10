---
title: 并发原子性技术之加锁方式
category: 并发编程
date: 2020-02-26 20:47:28
tags: 并发编程
---

<!-- more -->
###### 1. 锁的分类
> 乐观锁与悲观锁

- 乐观锁:在并发环境下,一般情况认为是属于读多写少的情况,没有数据冲突,当对共享资源发生写操作的时候,会先检测当前版本的数据与先前版本数据是否一致,如果不一致说明有其他线程已经发生写操作,需要重复进行读取然后检测再尝试修改,比如CAS机制,zk最优先通过最新主版本的策略来选举master,数据库根据版本更新数据等
- 悲观锁:在并发环境下,认为竞争非常激烈,于是在对共享资源发生写操作的时候先加上锁,然后执行临界区代码完成操作再释放锁,其他线程获取相同的锁需要进行等待,处于阻塞状态
- 乐观锁与悲观锁示例伪代码:

```java
// 使用java以及数据库的操作方式演示乐观锁与悲观锁
// optimistic.java
AtomicInteger count = new AtomicInteger();
run(){
	// cas instance
	int expected = 10;
	int updated = 12;
	count.compareAndSet(expected, updated);
	// db instance
	Object row = selectById();
	// update bussiness data except row version
	row.setXX();
	// update data 
	updateRowByIdWithVersion(row);
	// db.execute("update tb_entity set x1=?,x2=?,version=#{version}+1 where id=#{id} and version=#{version}", row);
}

// pessimistic.java
run(){
	// mutex lock
	synchronized(this){
		// code ..
	}
	// db lock
	// start transaction
	db.execute("select * from tb_entity where id=#{id} for update");	// lock
	db.execute("update tb_entity set x1=?,... where id=#{id}");			// update
	// commit trasaction;
}
```

> 可重入锁与递归锁

- 可重入锁: 同一个线程获取到锁对象,同时能够执行拥有该锁的其他同步代码,即递归锁/外部方法调用内部方法,两个方法持有同一把锁
- 递归锁: 本质也是可重入锁,也就是线程执行当前递归的方法时,由于是同一把锁,因此不会再次获取锁,而是持有锁进行执行方法的递归操作
- java实现可重入锁的技术
	- ReentrantLock
	- synchronized
```java
// source.java
final static ReentrantLock reentrantLock = new ReentrantLock();
run(){
	reentrantLock.lock();
	// code ...
    {
        reentrantLock.lock();
        // code ...
        reentrantLock.unlock();
    }
    // code ...
    reentrantLock.unlock();
}
// order.java
synchronized void createOutBoundOrder(){
	//...
}

run(){
	//Order sharedOrder = new Order();
	synchronized(sharedOrder){
		// create order item
		// get logistics price and caculate the best optimisic algorithm
		createOutBoundOrder(orders);
		// dispatch order towards sys
	}
}
```

> 公平锁与非公平锁

- 公平锁: 在并发环境下多线程争抢锁的顺序,如果是按照队列的先来后到的顺序,则视为公平
- 非公平锁: 在并发多线程环境下不按照先来后到的顺序,而是强行“插队”的方式获取锁,则视为不公平
- 场景分析: 如果线程A已经持有锁,这时候线程B获取失败并被挂起,处于阻塞状态,与此同时线程C也来争夺锁也将被挂起阻塞,当线程A执行完同步方法之后释放锁的时候,如果锁是公平的,那么我们期望就是线程B获取锁而线程C仍然处于阻塞状态,如果是非公平锁,那么将线程B/C获取锁的顺序是随机不确定的
- java实现公平锁的技术方案

```java
 // 公平锁,会消耗性能
 ReentrantLock pairLock = new ReentrantLock(true);

 // 非公平锁, 默认为不公平锁,获取锁的方式是随机,不按照线程争抢锁的顺序
 ReentrantLock unpairLock = new ReentrantLock(false);
```

> 共享锁(读)与独占锁(写)

- 共享锁: 也就是所谓的读锁,即给共享资源加上读锁,线程获取该锁的时候只能读取不能进行写操作,同理也可以被其他线程获取,但不能写,共享锁可以被多个线程共同持有
- 独占锁: 也就是写锁,或者称为排它锁,也就是只能有一个线程拥有该锁,当前给共享资源加上写锁时,当前线程可以进行写操作,但是其他线程要获取锁只能处于等待
- **简言之,共享锁能为多个线程所持有并只能进行读操作,独占锁只能被单个线程所持有并只能进行单写操作**
- java实现的读写锁技术

```java
// read_write_lock.java
private static void readWriteLock() {
    final Lock read = reentrantReadWriteLock.readLock();
    final Lock write = reentrantReadWriteLock.writeLock();

    for (int index = 0; index < 10; index++){
    	// 读取操作交替执行,所有读线程都能够获取读锁
        new Thread(){
            @Override
            public void run() {
                try{
                    read.lock();
                    read();
                }finally {
                    read.unlock();
                }
            }
        }.start();
		
		// 写操作按照获取锁的方式顺序执行,只能在单线程中实现写操作
        new Thread(){
            @Override
            public void run() {
                try{
                    write.lock();
                    write();
                }finally {
                    write.unlock();
                }
            }
        }.start();
    }
}
```

> 自旋锁 = CAS无锁 + 循环

- 自旋锁: 在并发环境下,当一个线程获取锁的时候发现锁已经被其他线程所占有,这时候并没有将该线程挂起而是在可用的CPU资源情况下,不断尝试获取锁的方式,如果成功则获取锁,如果失败则进行尝试,在进行尝试一定次数之后会对当前的并发环境进行分析并采取其他的策略获取锁
- 在Java中,默认尝试此时为10, 可以通过`-XX:PreBlockSpinsh`来设置对应的自旋失败次数
- 不足:消耗CPU资源,容易引起CPU占用资源过高导致机器卡顿甚至处理效率变低
- java技术实现的自旋锁方式

```java
// 使用CAS无锁 + 循环的方式 -- 自旋锁
	private static Unsafe unsafe;
    private static long valueOffsetValue = 0;
    private volatile int count = 0;

    static {
        try {
            Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
            theUnsafe.setAccessible(true);
            unsafe = (Unsafe) theUnsafe.get(null);

            // CAS 硬件原语 ---java语言 无法直接改内存。 曲线通过对象及属性的定位方式
            valueOffsetValue = unsafe.objectFieldOffset(LockCASDemo.class.getDeclaredField("count"));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
	private void countAdder(){
     	// 自增加
     	int current;
     	int value;
     	do{
         	current = unsafe.getIntVolatile(this, valueOffsetValue); // 读取当前值
         	value = current + 1; // 计算
     	}while (unsafe.compareAndSwapInt(this, valueOffsetValue, current , value));
 	}
```

> 分片加锁

- 分片加锁其实将锁的粒度缩小范围,尤其是在并发情况下,需要对存储容器不断进行读写操作,如果每次都对容器进行加锁,那么锁范围十分大,会大大降低程序执行的效率,因此分片加锁就是针对容器进行划分为若干等分(可以片段)进行加锁,也就是说针对容器某一个等分进行读写操作的时候只针对该部分进行加锁操作,其他部分仍保持无锁的状态,可以大大提升程序CPU利用率,加快程序的执行
- 分片加锁技术
	- java中使用ConcurrentHashMap针对Segment片段进行加锁,每一份Segment存储key-value,通过对Segment进行加锁方式(进一步优化可以处理为读写锁方式)来保证线程安全
	- 在数据库中,可以将一系列的update/delete操作进行行级别加锁,相比表级别的加锁方式,可以提升并发执行效率

> jdk锁的细节

- 锁响应中断: 在并发环境下,当前线程想要获取锁却发现锁已经被其他线程所持有,这时候当前线程就处于阻塞状态,如果当前线程在主线程中被中断,那么此时阻塞的线程收到中断的通知将会结束线程工作,如果不做锁的响应中断处理,那么当前线程仍然会获取锁并执行锁中的同步代码之后释放锁才中断

```java
// - java技术伪代码
// business.java
private Lock lock = new ReentrantLock();
void doBusiness(){
	// code ...
	// 锁响应中断,如果主线程对当前线程进行中断操作,那么当前线程将会直接退出线程
	lock.lockInterruptibly();
	// 如果主线程对当前线程进行中断操作,那么当前线程会继续获取锁之后再进行中断操作
	// lock.lock();
	try{
		// execute core should spent 10s 
		TimeUnit.SECONDS.sleep(10L);
	}catch(Exception e){
		LOG.error(e);	
	}finally{
		lock.unlock();
	}
}
// main.java
// 伪代码,为了简化代码的编写
Thread t1,t2 = new Thread(){
	public void run(){
		try{
			doBusiness();
			doOthers();
		}catch(InterruptedException e){
			LOG.error(e);
		}
	}
};

t1.start();
TimeUnit.SECONDS.sleep(1L);
t2.start();
TimeUnit.SECONDS.sleep(2L);
// 线程2中断
t2.interrupt();	
```

- 分类锁Condition: 用于替代wait/notify方法,wait和notify方法是结合synchronized一起使用,可以令线程等待和唤醒等待线程的集合,而Condition是结合Lock一起使用,能够提供多个等待集合,更精准地控制线程的等待和唤醒(底层使用LockSupport的park和unpark机制)

```java
	// 使用线程通信方式
  final Condition full = reentrantLock.newCondition();
  final Condition empty = reentrantLock.newCondition();
  final int size = 10;
  final ArrayDeque<String> container = new ArrayDeque<>();

  Thread producer = new Thread(){
      @Override
      public void run() {
          reentrantLock.lock();
         try{
             // 不断生产数据
             while (true){
                 // 数据已经满了
                 while (size == container.size()){
                     // 不再进行生产数据
                     full.await();
                 }
                 // 生产数据
                 container.push(new String("xxx"));
                 System.out.println(Thread.currentThread().getName() + " produce ...");
                 empty.signalAll();// 通知消费进行消费
             }
         } catch (Exception e) {
             e.printStackTrace();
         } finally {
             reentrantLock.unlock();
         }
      }
  };

  Thread consumer = new Thread(){
      @Override
      public void run() {
          reentrantLock.lock();
          try{
              while (true){
                  // 判断数据是否为空
                  while(0 == container.size()){
                      empty.await();
                  }
                  String str = container.pop();
                  System.out.println(Thread.currentThread().getName() + " consumer data for :" + str);
                  full.signalAll();  // 通知生产者生产数据
              }
          } catch (Exception e) {
              e.printStackTrace();
          } finally {
              reentrantLock.unlock();
          }
      }
  };

  producer.start();

  TimeUnit.SECONDS.sleep(2L);
  consumer.start();
```

###### 2. 加锁原理
> jdk中加锁的工作原理

- 基于UnSafe实现的CAS机制,其原理可以参考先前的[CAS机制](https://blog.csdn.net/wind_602/article/details/104099524)
- 基于内存屏障实现的synchronized加锁方式,其原理可以参考[原理一](https://blog.csdn.net/wind_602/article/details/103873050),[原理二](https://blog.csdn.net/wind_602/article/details/103936767)以及[原理三](https://blog.csdn.net/wind_602/article/details/103966182)
- 基于java并发包下的[AbstractQueuedSynchronizer(AQS)实现的锁方式](https://blog.csdn.net/wind_602/article/details/104161960)

###### 3. JVM锁优化技术手段
> synchronized加锁方式的优化

- 无锁: 正常的java对象,属于默认的初始化的锁状态
- 偏向锁: 对象头markword中的hashcode通过CAS替换为当前的ThreadId,同时对象的偏向锁标志为1
- 轻量级锁: 达到一定的并发量,JVM撤销偏向锁,锁升级为轻量级锁,即在对象头markword中的bitfields通过CAS替换为当前的执行线程栈帧的引用地址,同时栈帧中存储对象头的markword信息
- 重量级锁(mutex lock): 并发竞争激烈,升级为重量级锁,通过将对象头markword中的bitfields通过CAS替换为当前对象的引用地址,通过JVM优化会通过走“捷径”方式来进行加锁,锁无法降级
- 参考[synchronized工作原理(三)](https://blog.csdn.net/wind_602/article/details/103966182)

> 减少锁持有时间

- 仅在程序出现线程安全的情况下进行加锁
- 代码演示减少锁持有时间

```java
run(){
	// query from user
	// query from order
	try{
		// 保证一致性, 思考: 这里用事务也可以实现同样的效果,事务与锁有什么关系? 后面有时间再写
		lock.lock();
		// create order items
		// create orders
		// create outbound orders
	}finally{
		lock.unlock();
	}
}
```
- 关于事务与锁的联系与区别的参考文档
```text
参考mysql文档: https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-transaction-model.html
参考美团技术文档: https://tech.meituan.com/2014/08/20/innodb-lock.html
```

> 锁粗化

- 在并发条件下,程序获取锁的成本十分昂贵,如果在一块代码中对同一个锁不断地获取和释放,容易导致CPU系统资源被消耗殆尽,严重影响程序在操作系统中的执行效率,也就是说为了保证在并发多线程环境下,要求每个线程尽可能持有锁的时间片段尽可能少,同时能够在完成同步代码之后释放共享资源以便其他线程能够获取锁进行相应的操作

```java
// 锁粗化伪代码示例
increase(){
	synchronized(this){
		this.productNum -- ;
	}
	// send message to mq notice product num have been decreased
	synchronized(this){
		this.orderItem ++ ;
	}
	// shopping cart have add new one product one
}

take(){
	for(int index=0; index < LIMIT_SIZE; index++){
		synchronized(queue){
			queue.pop();
		}
	}
}
// 从上述看到,其实锁粗化出现的情况很多时候是我们是在指定的业务顺序逻辑进行编写代码造成的,一般情况下对代码的优化我们也需要考虑到两个同步代码中间的其他代码是否不影响执行程序代码的执行结果来进行优化
increase(){
	// 减少锁的获取
	synchronized(this){
		this.productNum -- ;
		this.orderItem ++ ;
	}
	// send message to mq notice product num have been decreased
	// shopping cart have add new one product one
}
take(){
	synchronized(queue){
		for(...){
			// ...
		}
	}
}
```
- JIT编译器优化:如果发现调用的次数过多,JIT会在编译的时候会进行锁粗化的优化
```java
int i = 0;
test1(){
    synchronized(this){
        i++;
    }
    synchronized(this){
        i++;
    }
}
// main函数
for (int i = 0; i < 1000000; i++) {
    new LockOptmistic().test1()
}
// 分别截图JIT编译 i < 10000 & i < 100000  & i < 1000000 对应的效果
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200211153628607.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200211153446382.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200211153124831.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

> 锁消除

- 也就是在JIT在编译阶段发生对于非共享资源在程序代码进行加锁的处理,那么这个时候JIT识别是非共享资源,这个时候将直接跳过加锁的方式运行程序代码

```java
// 锁消除的伪代码
execute(){
	// 线程封闭,属于线程私有的变量,此时非共享资源,加锁没有意义,JIT编译器会自动将锁进行消除
	Object object = new Object();
	int count = 0;	
	synchronized(object){
		count++;
	}
}
// 出现上述的原因很多时候是由于工程师本身在编译不规范造成的,这种并非业务性逻辑错误,可以避免的
```

- JIT Watch查看结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200211143531328.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

> 缩小锁的粒度(分片锁)

- 将一个存储同类数据的对象的容器拆分为更小的“容器”或者是“片段”, 在并发情况下针对小片段进行加锁,提升并发执行效率,降低锁的竞争, 比如ConcurrentHashMap(分片)与HashTable(整块)
- 注意: 并非分片锁执行效率一定比整块锁的方式执行效率快,只是分片能够降低cpu并发争抢锁的压力,要根据具体业务场景而定

> 锁分离

- 可以按照读写功能进行划分为读写锁,即写写互斥,读写互斥,读读共享,也可以按照指定的业务场景来对相应的程序代码设置对应的加锁方式,有效地提升并发执行的处理能力,降低锁之间的竞争,比如读写锁ReadWriteLock

