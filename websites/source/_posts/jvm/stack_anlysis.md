---
title: java程序运行堆栈分析
category: 并发编程
date: 2020-04-02 20:47:28
tags: 并发编程
---

<!-- more -->


###### 1. java程序源代码与字节码
> 源代码

```java
public class StackHeapAnalysis {

    // java 运行堆栈分析

    public static void main(String[] args) {

        //define my wallet totel balance
        int balance = 500;

        // birthday cake
        int cakeVal = 99;
        int cakeNum = 1;

        // birthday flower
        int flower = 2;
        int flowerNum = 99;

        int cakeSpent = cakeNum * cakeVal;
        int flowerSpent = flower * flowerNum;

        int totalSpent = flowerSpent + cakeSpent;
        int currentBalance = balance - totalSpent;

        System.out.printf("in order to celebrate birthday for my gift friends, i have spent %d\n", totalSpent);
        System.out.printf("now my wallet balance is %d \n", currentBalance);
    }

```

> 字节码文件

```text
public class com.xiaokunliu.homework.thread.base.StackHeapAnalysis
  minor version: 0                          // 最低版本号
  major version: 52                         // 主版本号 52 = 0x34   意味着是jdk8版本
  flags: ACC_PUBLIC, ACC_SUPER              // 访问标识符
Constant pool:                              // 常量池
   #1 = Methodref          #4.#32         // java/lang/Object."<init>":()V
   #2 = Fieldref           #33.#34        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #35            // in order to celebrate birthday for my gift friends, i have spent %d\n
   #4 = Class              #36            // java/lang/Object
   #5 = Methodref          #37.#38        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #6 = Methodref          #39.#40        // java/io/PrintStream.printf:(Ljava/lang/String;[Ljava/lang/Object;)Ljava/io/PrintStream;
   #7 = String             #41            // now my wallet balance is %d \n
   #8 = Class              #42            // com/xiaokunliu/homework/thread/base/StackHeapAnalysis
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               LocalVariableTable
  #14 = Utf8               this
  #15 = Utf8               Lcom/xiaokunliu/homework/thread/base/StackHeapAnalysis;
  #16 = Utf8               main
  #17 = Utf8               ([Ljava/lang/String;)V
  #18 = Utf8               args
  // 常量池省略....

{
  public com.xiaokunliu.homework.thread.base.StackHeapAnalysis();		// 类的构造器
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 8: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/xiaokunliu/homework/thread/base/StackHeapAnalysis;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=6, locals=10, args_size=1							// 以下是main线程的字节码操作指令
         0: sipush        500
         3: istore_1
         4: bipush        99
         6: istore_2
         7: iconst_1
         8: istore_3
         9: iconst_2
        10: istore        4
        12: bipush        99
        14: istore        5
        16: iload_3
        17: iload_2
        18: imul
        19: istore        6
        21: iload         4
        23: iload         5
        25: imul
        26: istore        7
        28: iload         7
        30: iload         6
        32: iadd
        33: istore        8
        35: iload_1
        36: iload         8
        38: isub
        39: istore        9
        41: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        44: ldc           #3                  // String in order to celebrate birthday for my gift friends, i have spent %d\n
        46: iconst_1
        47: anewarray     #4                  // class java/lang/Object
        50: dup
        51: iconst_0
        52: iload         8
        54: invokestatic  #5                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        57: aastore
        58: invokevirtual #6                  // Method java/io/PrintStream.printf:(Ljava/lang/String;[Ljava/lang/Object;)Ljava/io/PrintStream;
        61: pop
        62: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        65: ldc           #7                  // String now my wallet balance is %d \n
        67: iconst_1
        68: anewarray     #4                  // class java/lang/Object
        71: dup
        72: iconst_0
        73: iload         9
        75: invokestatic  #5                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        78: aastore
        79: invokevirtual #6                  // Method java/io/PrintStream.printf:(Ljava/lang/String;[Ljava/lang/Object;)Ljava/io/PrintStream;
        82: pop
        83: return
      LineNumberTable:						// 字节码存储代码在代码表的数组中,LineNumberTable通过定位源文件的代码行数与代码表数据长度对应起来
        line 15: 0
          // ......
      LocalVariableTable:				// 线程本地局部变量表,存储main线程中的局部变量信息
        Start  Length  Slot  Name   Signature
            0      84     0  args   [Ljava/lang/String;
        // ....
} 
```

###### 2. java运行堆栈分析
> 类执行分析准备
- 定义的class文件编译为二进制.class的时候,产生幻数标识,访问标识
- 执行main的时候先执行当前类的init构造器完成初始化
- 线程执行包含程序计数器以及虚拟机栈(多个栈帧[操作数栈 + 局部变量表 + 动态链接 + 方法返回])

> JVM的int入栈操作指令参考
```text
## 参考链接
https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings
## int入栈指令
iconst_<i> 	操作常量 -1 - 5
ipush 			操作一个字节的int数据
sipush			操作两个字节的int数据
ldc				超过两个字节的int数据
```

> main方法执行分析
- `int balance = 500` 分析
```java
 // source.java
 int balance = 500;
 // 字节码文件
 0: sipush        500
 3: istore_1
// 其他定义变量的操作与上述一致
```
 **将500压入操作数栈中并存储在本地变量表,然后弹出操作数栈,同时程序计数器记录当前代码执行的位置**

- `int cakeNum = 1;`与`int flower = 2;`分析
```java
// source.java
 int cakeNum = 1;
 int flower = 2;

// 字节码文件
7: iconst_1		
8: istore_3
9: iconst_2
10: istore        4
```
**JVM将常量1和2压入操作数栈并存储在本地变量表中,然后弹出操作数栈,更新程序计数器当前执行的代码的位置**

- 进行乘法运算
```java
// source.java
int cakeSpent = cakeNum * cakeVal;

// 字节码
 16: iload_3		// 1
 17: iload_2		// 99
 18: imul
 19: istore        6
```
**将本地变量表中的数据1 和 数据99分别压入操作数栈中然后进行乘法运算最后将运算结果存储在本地变量表中**
其他的数学基本运算也是依次类推

- 打印输出
```java
//source.java
System.out.printf("in order to celebrate birthday for my gift friends, i have spent %d\n", totalSpent)

// 字节码文件
 41: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
 44: ldc           #3                  // String in order to celebrate birthday for my gift friends, i have spent %d\n
 46: iconst_1
 47: anewarray     #4                  // class java/lang/Object
 50: dup
 51: iconst_0
 52: iload         8
 54: invokestatic  #5                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
 57: aastore
 58: invokevirtual #6                  // Method java/io/PrintStream.printf:(Ljava/lang/String;[Ljava/lang/Object;)Ljava/io/PrintStream;
 61: pop
```
- 分析:
	- getstatic     #2 表示从常量池获取静态资源#2并将其加载到操作数栈中,同时创建新的栈帧,因为属于一个新的方法调用
	- ldc	将常量池中将字符串数据压入操作数栈中
	- anewarray     #4  从常量池中获取静态资源#4并将其创建一个新的对象数组引用
	- dup 上述的对象引用复制到操作数栈的顶部
	- iload         8 从本地变量表中加载totalSpent的数据
	- invokestatic  执行整数转换为字符串的方法,将totalSpent转换为字符串
	- aastore	将上述计算得到的结果存储到本地变量表中
	- invokevirtual 调用方法输出字符串到控制台中
	- main的线程中弹出getstatic的栈帧

###### 3. 小结
- java程序代码转换为.class字节码的时候是按照指定的字节码指令进行操作
- 线程栈中定义的局部变量数据将会压入操作栈中进行计算然后存储到本地变量表中,并且会将执行代码的行数记录到程序计数器中
- 线程调用方法的时候会新创建一个新的栈帧执行对应的方法中的代码,当方法执行完成之后栈帧将会从当前的线程中弹出



