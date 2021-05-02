---
title: Java之.class文件与字节码文件
category: 并发编程
date: 2020-02-02 20:47:28
tags: 并发编程
---

<!-- more -->

#### JVM运行数据区概述
> .class与字节码bytecode

- .class: 是指文件扩展名称为.class的文件,表示由java源程序经过java编译器编译而成且由JVM执行的二进制文件,因此可以通过拥有一份.class文件在不同的操作系统平台上的JVM执行,实现跨平台运行的特性
- 字节码bytecode: 简单说不是文件,而是JVM操作的指令格式,通常我们通过`javap -c -v xx.class`生成的文件称为字节码文件,是属于可阅读的字节码指令文件,能够让我们清楚地知道java文件编译成.class文件之后显示的执行指令,便于程序员理解jvm的相关的知识

> .class文件与字节码文件格式

- .class文件(16进制文件)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200124155250693.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 字节码文件(可阅读的指令文件)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200124155445198.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)


#### 2. .class文件结构与字节码

> .class文件结构

- 幻数
	- 类文件的四个字节表头0xCAFEBABE
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200124160239164.png)

- 类文件格式的版本
	- 类文件的次要和主要版本
	- jdk的主要版本如下
		```text
		Java SE 14 = 58 (0x3A hex),
		Java SE 13 = 57 (0x39 hex),
		Java SE 12 = 56 (0x38 hex),
		Java SE 11 = 55 (0x37 hex),
		Java SE 10 = 54 (0x36 hex),[3]
		Java SE 9 = 53 (0x35 hex),[4]
		Java SE 8 = 52 (0x34 hex),
		Java SE 7 = 51 (0x33 hex),
		Java SE 6.0 = 50 (0x32 hex),
		Java SE 5.0 = 49 (0x31 hex),
		```
		![在这里插入图片描述](https://img-blog.csdnimg.cn/20200124161328936.png)
表示0x0034当前使用jdk版本为jdk8

- 常量池: 类文件的常量池
- 访问标识: 类文件的访问标识,  abstract, static,等等
- 当前的类名称,class name
- 当前父类的名称, super class name
- 当前类的任何接口
- 当前类的字段信息
- 当前类的方法信息
- 当前类的属性信息

> 字节码指令

- 指令类别
	- 存储指令 （例如：aload_0, istore）
	- 算术与逻辑指令 （例如: ladd, fcmpl）
	- 类型转换指令 （例如：i2b, d2i）
	- 对象创建与操作指令 （例如：new, putfield）
	- 堆栈操作指令 （例如：swap, dup2）
	- 控制转移指令 （例如：ifeq, goto）
	- 方法调用与返回指令 （例如：invokespecial, areturn)

- 指令操作前后缀与数据类型
	- i		整数
	- l	 	长整数
	- s 	短整数
	- b	字节
	- c	字符
	- f		单精度浮点数
	- d	双精度浮点数
	- z	布尔值
	- a	引用

- 例子
```text
## iadd  表示两个整数相加
## dadd  表示两个double类型数据相加
....
```
- 参见jvm的规范
```text
## jvm指令码表
https://docs.oracle.com/javase/specs/jvms/se13/html/jvms-6.html#jvms-6.5

## jvm字节码文件格式
https://docs.oracle.com/javase/specs/jvms/se13/html/jvms-4.html#jvms-4.1
```


