---
title: synchronized基于JVM规范的工作原理(一)
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->

###### 1. synchronized同步方法
> synchronized同步方法的字节码还原
* java声明的方法在jvm中的结构格式method_info
```text
method_info {
       u2             access_flags;
       u2             name_index;
       u2             descriptor_index;
       u2             attributes_count;
       attribute_info attributes[attributes_count];
}
```
* 其中关注method_info的access_flags取值
	* ACC_SYNCHRONIZED,对应标示值为0x0020,作用是声明为同步方法,在jvm内部包装一个监视器锁被调用
	* ACC_PUBLIC,对应标示值为0x0001,作用声明方法为public,运行被包外的类公开访问
* 带synchronized的部分java代码
```java
// Account.java
public synchronized void setBalance(double balance) {
        this.balance = balance;
}
```
* 对应的java代码的字节码
```text
## 上述代码的字节码如下
public synchronized void setBalance(double);
    descriptor: (D)V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=3, args_size=2
         0: aload_0
         1: dload_1
         2: putfield      #2                  // Field balance:D
         5: return
      LineNumberTable:
        line 20: 0
        line 21: 5
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0  this   Lcom/xiaokunliu/concurrency/sync/syncmethod/Account;
            0       6     1 balance   D
 ## 从字节码中可以看出在方法添加synchronized关键字,会在编译阶段多产生一个flag,即ACC_SYNCHRONIZED
```
> synchronized同步方法的本质
* 摘录jvm规范中的原文
```text
1) Method-level synchronization is performed implicitly, as part of method invocation and return.
2) A synchronized method is distinguished in the run-time constant pool's method_info structure by the ACC_SYNCHRONIZED flag, which is checked by the method invocation instructions. 
3) When invoking a method for which ACC_SYNCHRONIZED is set, the executing thread enters a monitor, invokes the method itself, and exits the monitor whether the method invocation completes normally or abruptly. 
4) During the time the executing thread owns the monitor, no other thread may enter it. If an exception is thrown during invocation of the synchronized method and the synchronized method does not handle the exception, the monitor for the method is automatically exited before the exception is rethrown out of the synchronized method
```
* 简言概括
```text
1) 方法级同步在JVM中是隐式执行的，是作为方法调用和返回的一部分
2) 同步方法在运行时常量池的method_info结构中由ACC_SYNCHRONIZED标志加以区分，该标志由方法调用指令进行检查
3) 执行线程识别到方法中含有ACC_SYNCHRONIZED标志将会获取一个监视器对象然后调用方法,最后而且不论当线程正常执行或是异常退出时将会释放监视器对象
4) 在执行期间,执行线程持有监视器对象,而其他执行线程将无法获取监视器对象,如果方法抛出异常将会释放监视器对象
```

###### 2. synchronized同步代码块
> synchronized同步代码块字节码还原
* java同步代码块
```java
// Cinema.java
  public void addTickets1(int number){
        synchronized (controlCinema1){
            System.out.println(Thread.currentThread().getName() + " get controlCinema1 lock, add number is " + number);
            this.vacanciesCinema1 += number;
        }
    }
```
* 同步代码块对应的字节码
```text
public void addTickets1(int);
    descriptor: (I)V
    flags: ACC_PUBLIC
    Code:
      stack=5, locals=4, args_size=2
         0: aload_0
         1: getfield      #3                  // Field controlCinema1:Ljava/lang/Object;
         4: dup
         5: astore_2
        // ========================= //
         6: monitorenter
        // ========================= //
         7: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
        10: new           #10                 // class java/lang/StringBuilder
        13: dup
        14: invokespecial #11                 // Method java/lang/StringBuilder."<init>":()V
        17: invokestatic  #12                 // Method java/lang/Thread.currentThread:()Ljava/lang/Thread;
        20: invokevirtual #13                 // Method java/lang/Thread.getName:()Ljava/lang/String;
        23: invokevirtual #14                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        26: ldc           #20                 // String  get controlCinema1 lock, add number is
        28: invokevirtual #14                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        31: iload_1
        32: invokevirtual #16                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        35: invokevirtual #17                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        38: invokevirtual #18                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        41: aload_0
        42: dup
        43: getfield      #7                  // Field vacanciesCinema1:J
        46: iload_1
        47: i2l
        48: ladd
        49: putfield      #7                  // Field vacanciesCinema1:J
        52: aload_2
        // ========================= //
        53: monitorexit
        // ========================= //
        54: goto          62
        57: astore_3
        58: aload_2
        // ========================= //
        59: monitorexit
        // ========================= //
        60: aload_3
        61: athrow
        62: return
      Exception table:
         from    to  target type
             7    54    57   any
            57    60    57   any
      LineNumberTable:
        line 42: 0
        line 43: 7
        line 44: 41
        line 45: 52
        line 46: 62
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      63     0  this   Lcom/xiaokunliu/concurrency/sync/syncAttribute/Cinema;
            0      63     1 number   I
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 57
          locals = [ class com/xiaokunliu/concurrency/sync/syncAttribute/Cinema, int, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
```
* 字节码解读
	* jvm通过monitorenter来完成加锁的操作
	* 53行的monitorexit之后是go语句到62行,属于程序正常退出释放锁的操作
	* 59行的monitorexit之后是athrow的字节码指令,表示当程序异常的时候释放锁的操作
	* monitorenter和monitorexit支持编译指令来实现同步指令

> synchronized同步代码块的工作原理
* 引入jvm规范原语
```text
1) Synchronization of sequences of instructions is typically used to encode the synchronized block of the Java programming language. 
2) The Java Virtual Machine supplies the monitorenter and monitorexit instructions to support such language constructs. 
3) Proper implementation of synchronized blocks requires cooperation from a compiler targeting the Java Virtual Machine
```
* 简言概括之
```text
1) 指令序列的同步通常用于java程序中的同步代码块
2) jvm支持在编译阶段执行同步指令
3) 同步块的实现需要java编译器的支持
```
###### 3. synchronized工作原理小结
> 结构化锁
```text
结构化锁定是这样一种情况:在方法调用期间，给定监视器上的每个出口与该监视器上的前一个入口匹配。
由于不能保证提交给Java虚拟机的所有代码都将执行结构化锁定，所以允许Java虚拟机的实现，
```
* jvm通过以下规则保证结构化锁定:
	* 不论方法是正常还是异常退出,jvm必须保证线程对监视器入口(monitorenter)的执行次数与对监视器出口(monitorexit)的执行次数相等
	* 在方法调用期间,线程对监视器执行的出口次数(monitorexit)不可能超过对监视器入口的执行次数(monitorenter)

> 工作原理本质
* synchronized的实现是通过jvm的监视器的入口和出口来实现的
* synchronized同步方法是隐式实现(编译阶段仅看到同步标志)
* synchronized同步代码块是显示实现(编译阶段可见)

> 注意点
* jvm通过一个单一的同步结构:监视器来支持方法和方法中的指令序列的同步
* 使用synchronized同步代码块不论程序是正常完成还是异常退出都会自动释放锁



