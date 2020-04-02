---
title: 并发原子性技术之CAS机制
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->

###### 1. CAS机制
> CAS定义

- 从wiki百科中,Compare-and-swap简称为“CAS”,是属于并发多线程中实现同步原子操作的指令,是依赖于硬件层次的原语发起的原子操作
- 从程序代码理解上,CAS包含`check then act`的两个动作,这两个动作在处硬件的处理器上是具备原子性,也就是在操作系统底层上已经实现对CAS算法的原子性保证

> CAS 使用条件

- 需要输入两个数值,一个是期望修改前的值(旧值),一个是需要被设置的新值(新值)
- 进行CAS操作需要进行对预期值的check操作

> CAS之java简易版本

- 通过CAS设置新值
```java
// 简单的check方式完成对象新值的设置
// cas.java
boolean cas(Object ref, int refOldVal, int refNewVal){
	if(ref.value != refOldVal) return false;
	ref.value = refNewVal;
	return true;
}
```

- CAS实现加法设置器
```java
// cas_adder.java
// import cas.java
int cas_adder(Object ref, int incr){
	while(!cas(ref, ref.value, ref.value+incr)){
	}
	// 表示修改成功
	return ref.value;
}
```

###### 2. Java之CAS实现原理UnSafe
> java原子操作类Atomic*实现

- 比如AtomicBoolean的实现源代码
```java
// AtomicBoolean.java 摘取核心代码
public class AtomicBoolean implements java.io.Serializable {
    private static final long serialVersionUID = 4654671469794556979L;
    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    // 使用volatile修饰,保证修改的可见性,令当前的线程缓存失效,读取主内存修改后的数据
	private volatile int value;		
	
    static {
        try {
        // 获取当前属性value的内存地址偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicBoolean.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    
     /**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(boolean expect, boolean update) {
        int e = expect ? 1 : 0;
        int u = update ? 1 : 0;
        // 使用unsafe的CAS方法完成修改操作
        return unsafe.compareAndSwapInt(this, valueOffset, e, u);
    }
}
```

- 基于UnSafe实现自定义的CAS操作(依样画葫芦)

```java
// AtomicDefineInt.java
public class AtomicDefineInt {

    private volatile int value;
    private static final Unsafe unsafe;
    // 定义内存偏移量
    private static final long iValueOffset;

    static {
        try {
        // 必须通过反射获取Unsafe,本身是属于不安全的一个操作,直接通过getUnsafe会抛出异常,
            Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
            theUnsafe.setAccessible(true);
            unsafe = (Unsafe) theUnsafe.get(null);
        // 从JNI中获取value属性在堆内存中的地址偏移量
            iValueOffset = unsafe.objectFieldOffset(AtomicDefineInt.class.getDeclaredField("value"));
        }catch (NoSuchFieldException e){
            throw new RuntimeException(e);
        }
    }
    
    // 借助UnSafe调用CAS方法完成操作
    public boolean compareAndSet(int expect, int update){
        return unsafe.compareAndSwapInt(this, iValueOffset, expect, update);
    }
}
```

> Unsafe源码以及实现
```java
// java代码
public final class Unsafe {
	// 字段属性theUnsafe
	private static final Unsafe theUnsafe;
	
	@CallerSensitive
    public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass();
        // 只允许jvm调用,java程序无法直接调用
        if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }
	
	// compareAndSwapInt的修改方法为native
	// 表示在线程的虚拟机栈中加载调用方法,由c++底层源码实现
	public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
}
// c++代码
// 核心方法
// 获取内存偏移量
UNSAFE_ENTRY(jlong, Unsafe_ObjectFieldOffset0(JNIEnv *env, jobject unsafe, jobject field)) {
  return find_field_offset(field, 0, THREAD);
} UNSAFE_END

// 完成CAS操作
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) {
  // JNI解析加载java object
  oop p = JNIHandles::resolve(obj);
  if (p == NULL) {
  	// 如果p不是java中定义的面向对象引用,直接从内存地址完成修改操作
    volatile jint* addr = (volatile jint*)index_oop_from_field_offset_long(p, offset);
    return RawAccess<>::atomic_cmpxchg(addr, e, x) == e;
  } else {
  	// 如果p是java对象,直接在堆内存中对p的属性数据进行修改操作
    assert_field_offset_sane(p, offset);
    return HeapAccess<>::atomic_cmpxchg_at(p, (ptrdiff_t)offset, e, x) == e;
  }
} UNSAFE_END
```

- 源码分析
	- 从上述可知,java底层CAS实现机制是通过JNI环境来调用c++的实现
	- 底层实现一类是根据是否为java对象类型来直接在堆内存中完成CAS操作,而另一种是针对非java对象类型则直接从内存地址中完成对应的CAS操作
- UnSafe的实现中可以看出,需要在java中使用UnSafe需要以下条件
	- 必须要有一个对象的属性在内存的偏移量valueOffset
	- 其次需要传递对应的java对象p
	- 同时还需要有修改前期望的数值以及要设置修改的值
	- 另外在java代码中使用volatile保证数据是刷新到内存的,因为JNI是调用c++实现是直接操作堆内存的,那么我们需要在并发多线程下保证读是可见的,写是最新的

###### 3. CAS存在的问题
> CAS问题

- 在上述的CAS实现代码中,对于做一些自增或是自减等数学运算操作时,会产生自旋判断,容易造成CPU性能下降
- 在CAS操作仅针对单个变量,如果涉及多个变量的原子操作,CAS是无法保证原子性
- 最后一个就是ABA问题

> ABA问题

- java源码示例
```java
// AtomicDefineObject
public final class AtomicDefineObject implements Serializable {

    private volatile String value;
    private static Unsafe unsafe;
    private static final long valueOffset;

    static {
        try {
            // 反射获取
            Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
            theUnsafe.setAccessible(true);
            unsafe = (Unsafe) theUnsafe.get(null);

            valueOffset = unsafe.objectFieldOffset(AtomicDefineObject.class.getDeclaredField("value"));
        }catch (NoSuchFieldException | IllegalAccessException e){
            throw new RuntimeException(e);
        }
    }

    private boolean compareAndSet(String expect, String update){
        return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
    }

    public void setValue(String value){
        this.compareAndSet(this.value, value);
    }

    public String getValue() {
        return value;
    }
}
```

- 对应执行的main方法
```java
main(){
		final AtomicDefineObject atomicDefineObject = new AtomicDefineObject();
        atomicDefineObject.setValue("A");
        System.out.println(Thread.currentThread() + " --- set value to A----");

        Thread t1 = new Thread("thread1"){
            @Override
            public void run() {
                try{
                    // 完成写操作需要1s
                    TimeUnit.SECONDS.sleep(1L);
                    atomicDefineObject.setValue("B");
                    System.out.println(Thread.currentThread() + " --- set value to B----");
                }catch (Exception e){}

            }
        };


        Thread t2 = new Thread("thread2"){
            @Override
            public void run() {
                try{
                    // 完成写操作需要2s
                    TimeUnit.SECONDS.sleep(2L);
                    atomicDefineObject.setValue("A");
                    System.out.println(Thread.currentThread() + " --- set value to A----");
                }catch (Exception e){}
            }
        };

        Thread t3 = new Thread("thread3"){
            @Override
            public void run() {
               try{
                   // 由于网路原因，读取操作延迟
                   TimeUnit.SECONDS.sleep(3L);
                   String val = atomicDefineObject.getValue();
                   System.out.println(Thread.currentThread() + " --- get value ----" + val);
               }catch (Exception e){}
            }
        };
        // start && join
}
```

- 执行结果
```text
Thread[main,5,main] --- set value to A----
Thread[thread1,5,main] --- set value to B----
Thread[thread2,5,main] --- set value to A----
Thread[thread3,5,main] --- get value ----A
finish task dome ...

## 最终读线程读取到的数据是A,但是不知道之前对象已经发生过修改操作,对当前读操作是一个透明
```

> ABA解决方案

- 为ABA问题增加版本号,版本号的值设置为long类型的自增加方式,这样程序就知道共享资源数据的变更情况

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200128210626898.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- java提供的一个支持类AtomicStampedReference,通过增加时间戳方式来记录修改的时候对应的时间戳,这样的方式便可以知道当前的数据最近修改的时间段

- ABA技术解决的意义
	- 通过知道数据对象变化的情况,我们可以利用版本或者时间戳的方式记录修改的变更日志,方便逻辑业务排查
	- 同时知道变更的时间或者是版本号,可以利用最新的一个数据值来作为一个起点修复过程,比如我们应用在某一个时间点down掉,如果此时更新应用的数据存在ABA问题,那么可以结合实际场景来进行恢复