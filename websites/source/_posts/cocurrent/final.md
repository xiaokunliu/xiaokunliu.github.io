---
title: final线程安全与工作原理
category: 并发编程
date: 2020-02-15 20:47:28
tags: 并发编程
---

<!-- more -->

###### 1.final语义与使用
> final的语义
 
 - 编译器做的处理
	- 编译器可以跨同步屏障移动对final修饰的字段值进行读取和调用任意或未知的方法
	- 对于final与non-final修饰的字段,允许编译器保存一份final的数据缓存放在寄存器中,对比必须要加载non-final数据的情况下,它不需要从主内存中加载就可以获取​

- 并发线程下是安全的
	- 对于final修饰的字段在所有线程中是属于不可变(基本类型值不可变,引用类型是引用地址不可变),也就是对于程序员而言,在线程中重新对final修饰的字段赋值将会编译不通过
	- 只有在对象完全初始化之后，线程才能看到对该对象的引用，这样就可以保证看到该对象的final字段的正确初始化值
	```text
	基于Happen-Before原则,程序任何对象的初始化happen-before于程序中任何其他的动作操作行为
	因此能够保证不会被重排序,也就是说final修饰的字段在线程读取已经先在构造器中执行写操作
	因而所有线程看到final修饰的变量均为最终最新的版本
	```

- final的使用模型
	- 在对象的构造函数中为对象设置final字段;在对象的构造函数完成之前，不允许在其他线程可以看到的地方对正在构造的对象的引用执行写操作
	- 这样可以保证在线程看到该对象的时候,将始终看到该对象final字段的最终正确构造版本

> final的线程安全性

- 源代码

```java
// FinalClass.java
public class FinalClass {

    public final int i;
    public int j;
    public final DefineFinalObject defineFinalObject;

    static FinalClass finalClass;


    public FinalClass(){
//        int x = i;
//        DefineFinalObject var = defineFinalObject;
        i = 4;
        defineFinalObject = new DefineFinalObject();
        try{
            // code ... may throw exception
//            i = 4;
//            defineFinalObject = new DefineFinalObject();
            j = 9;
            // code ...
        }catch (Exception e){}
    }

    static void writer(){
        finalClass = new FinalClass();
        System.out.println("have init FinalClass");
    }

    static void reader(){
        if (finalClass != null){
            int x = finalClass.i;
            int y = finalClass.j;
            System.out.printf("get x = %d, and y = %d", x, y);
        }
    }

    public static void main(String[] args) {
        FinalClass finalClass = new FinalClass();
        System.out.println(finalClass.defineFinalObject);
        finalClass.defineFinalObject.setAge(10);
        System.out.println(finalClass.defineFinalObject);
    }
}
```

- 代码分析
	- 在构造器中,对于final修饰的基本类型/引用类型变量编译器不允许在try中对`i=4`进行写操作,会出现编译报错,对于没有使用final修饰的变量j进行写操作的`j=9`则没有出现编译报错
	- 其次,在对`i=4`执行的写操作之前,编译器不允许对final修饰的基本/引用变量进行读操作,否则编译报错
	- 基于上述编译器的规则,最终保证final的基本类型/引用变量是在其他线程是最终最新的版本,也就是`i=4`以及`defineFinalObject = new DefineFinalObject()`创建对象并引用对应的对象地址
	- 在main的线程方法中,可以对不可变的defineFinalObject的属性信息进行修改,说明引用类型不可变是指对应的对象内存地址,即使无法再通过`defineFinalObject = new DefineFinalObject()`的方式重新指向一个新的引用地址
	- 最后一点就是,final必须是在构造器中完成初始化,同时根据Happen-Before原则,线程访问final的数据一定是在完成初始化后的最终数据且无法再进行修改(引用类型是可以修改其属性信息),从而保证了线程对final修饰的变量是属于线程安全的共享数据

> final与static使用分析

- 源代码
```java
// FinalSharedClass.java
public class FinalSharedClass {

    public final static int num;
    public static int x;
    public final static DefineFinalObject defineFinalObject;

    static {
        //    静态代码块，保证在加载类信息的时候也完成上述静态数据变量的初始化赋值的操作
        num = 10;
        defineFinalObject = new DefineFinalObject();
        System.out.printf("have finished static code for num=%d and obj=%s...\n", num, defineFinalObject);
    }

	{
		// 默认代码块,等同于对象构造器,编译器报错,要么是声明被分配过要么是上述定义的static报没有被分配
//        num = 10;
//        defineFinalObject = new DefineFinalObject();
	}
	
    static void writer(){
        // final 修饰的无法修改，将会编译报错提示已经分配值操作
//        num = 20;
//        defineFinalObject = new DefineFinalObject();
        x = 10;
        defineFinalObject.setAge(10);
    }

    static void reader(){
        // must be same with the end of static code
        System.out.printf("read final static num: %d \n", num);
        System.out.printf("read final static defineFinalObject: %s \n", defineFinalObject);

        // may be 0 or 10
        System.out.printf("read static x: %d \n", x);
    }
}
```

- 分析
	- 静态代码中的final初始化过程必须是在静态代码块中,也就是加载类信息的时候同时完成对final的数据赋予值的操作
	- 在方法writer()中可以看到无法对num以及defineFinalObject再次进行写操作,从而保证线程对于final修饰的数据只能读取,因此并不存在线程安全问题

- 小结
	- final且非静态的对象变量,final将在对象构造器中完成初始化赋值操作,且不能在构造器之外执行写操作,只能被读取,因而不存在线程安全性问题
	- final且为静态的类对象变量时,final将会在类的静态代码块中完成初始化(优先于对象构造器执行),且不能在静态代码之外完成初始化操作,由于JVM加载类的信息的时候是优先于创建线程的,因此当线程访问的时候final的static数据已经完成初始化赋值操作,因此也不存在线程安全问题

###### 2. final的内存语义与实现

> final的遵循的规则

- 对于final领域修饰的非static变量,对象的final领域变量的写操作优先于该对象构造器完成初始化之后的引用赋值操作,即`i=4`优先于`finalClass = new FinalClass();`,也就是两个操作不能重排序,final修饰的为引用类型也是一样遵循这个规则
- 对于final领域修饰的非static变量,对象的final领域变量在构造器初始化的读操作优先于所有线程对该对象的final数据的读操作,也就是构造器执行默认值`i 为默认值 0`的操作优先于其他线程对`i 为 4`的读操作,也就是两者不能重排序,同理final修饰的引用变量也是遵循这个规则
- 另外,对于final修饰且为static的变量,在java程序中静态代码只执行一次,且静态代码完成final领域的数据变量初始化操作优先于所有线程对该变量的读操作,相当于“写一次读多次”,并且写一次是在JVM第一次创建该对象实例的时候加载的,且优先于所有线程的其他行为动作,对此是保证写在前读在后的一个逻辑顺序

> final的内存语义是如何实现的

- aarch架构内存屏障指令
```C++
// A more convenient access to dmb for our purposes
  enum Membar_mask_bits {
    StoreStore = ISHST,
    LoadStore  = ISHLD,
    LoadLoad   = ISHLD,
    StoreLoad  = ISH,
    AnyAny     = ISH
 };
```

- final语义也是基于内存屏障实现(aarch架构)
```c++
// templateTable_aarch64.cpp
// Issue a StoreStore barrier after all stores but before return
// from any constructor for any class with a final field.  We don't
// know if this is a finalizer, so we always do so.
if (_desc->bytecode() == Bytecodes::_return)
  __ membar(MacroAssembler::StoreStore);
```

- 分析
	- 根据上述可知,jvm在实现中由于不清楚对象什么时候会调用finalizer方法进行回收,因此会在任何对象的构造器返回前插入内存屏障对final修饰的变量执行写操作
	- 其次,可以看到final插入的内存屏障为StoreStore类型,也就是在构造器返回之前插入StoreStore的内存屏障,也就是说final对变量的写操作的可利用结果在内存屏障之前的代码是不可用的,也就是对`final x = 9`的写操作之前是看不到`x=9`的结果

- volatile与final写操作的内存屏障实现区分
```c++
// templateTable_aarch64.cpp
// According to the new Java Memory Model (JMM):
// (1) All volatiles are serialized wrt to each other.  ALSO reads &
//     writes act as aquire & release, so:
// (2) A read cannot let unrelated NON-volatile memory refs that
//     happen after the read float up to before the read.  It's OK for
//     non-volatile memory refs that happen before the volatile read to
//     float down below it.
// (3) Similar a volatile write cannot let unrelated NON-volatile
//     memory refs that happen BEFORE the write float down to after the
//     write.  It's OK for non-volatile memory refs that happen after the
//     volatile write to float up before it.
// We only put in barriers around volatile refs (they are expensive),
// not _between_ memory refs (that would require us to track the
// flavor of the previous memory refs).  Requirements (2) and (3)
// require some barriers before volatile stores and after volatile
// loads.  These nearly cover requirement (1) but miss the
// volatile-store-volatile-load case.  This final case is placed after
// volatile-stores although it could just as well go before
// volatile-loads.

// templateTable_arm.cpp
// StoreLoad barrier after volatile field write
  volatile_barrier(MacroAssembler::StoreLoad, Rtemp);
  __ b(skipMembar);

  // StoreStore barrier after final field write
  __ bind(notVolatile2);
  volatile_barrier(MacroAssembler::StoreStore, Rtemp);
```

- 分析
	- 从上面可以看到volatile的写操作内存屏障是使用StoreLoad方式,final使用的内存屏障是StoreStore方式
	- 在aarch64处理器架构中,final也可以使用与volatile相同的内存屏障

- volatile与final内存屏障伪代码

```c++
// 针对写操作
// Store为写屏障,作用就是防止重排序,同时让数据刷新到主内存
// Load为读屏障,作用就是使得当前工作线程的缓存失效,直接读取主内存数据,保证数据一致性
// for a volatile write
//	
//   dmb ish		// store内存屏障		-- 防止重排序
//   str<x>		    // 写volatile数据
//   dmb ish		// load内存屏障		-- 保证数据一致性(目的就是要看见最新的数据)
//   other codes...

// for a final write 
//
//   dmb ishst		// store内存屏障		-- 防止重排序
//   final x = 9;	    // 写final数据
//   dmb ishst		// store内存屏障		-- 防止重排序
// other codes ...
```

- 结果
	- 可以看到上述描述中使用内存屏障的技术是非常昂贵的,为了适应对应的使用场景,在java中对于volatile与final不能同时存在,同时volatile的使用场景是读写多,而final是一次性写多次读的场景,对此使用的内存屏障技术也会有所不同
	- final建议使用为StoreStore而不使用与volatile相同的StoreLoad内存屏障是根据使用场景来的,final实现写一次,那么在创建线程的时候工作内存会copy一份相同的数据作为缓存,不需要读取主内存的数据,同时final的写是在构造器中完成,也就是在构造器中添加内存屏障,也保证了在对象构造器之外不能再对final的数据进行修改的操作
	- 同理,对于static的final数据,是在static代码块实现StoreStore内存屏障,作用和对象构造器类似

###### 3. final规范小结

> Java语言规范

- final在构造器中执行赋予值的写操作,因此当线程访问的时候会看到当前final修饰的变量为最新版本的数据
- 如果在构造器函数中执行final变量的读操作在写操作之后,那么会看到final分配给变量的最新数据,不存在缓存读取
- 读取共享变量里的final数据,则必须先要访问这个共享变量的引用对象然后再读取final数据
- 通常static final表示为常量,然而System.in/System.err/System.out也是属于static final,是属于遗留的原因,可通过 System.setIn, System.setOut, and System.setErr来完成赋值操作,java规范中称之为“写保护”
- 对应部分源码说明如下

```c++
//ciField.cpp
// 源码中注释说明
// Is this field a constant?
//
// Clarification: A field is considered constant if:
//   1. The field is both static and final
//   2. The field is not one of the special static/final
//      non-constant fields.  These are java.lang.System.in
//      and java.lang.System.out.  Abomination.
```

> JMM规范

使用final修饰的数据在字节码中显示带有`ACC_FINAL`的访问标识符,对应访问标示符号的值为`0x1000`
- 使用`final class XX`表明该Class不能被继承,说明该Class没有子类
- 类的属性字段被声明为final,表明该字段在对象构造器之外不能被分配值操作
- JVM规范中,volatile的访问标识与final的访问标识不能同时出现,也就是说在程序代码中不能同时使用final和volatile修饰同一个变量
- 方法声明为final,表示该方法不能被覆盖重写
- 内部类使用final修饰的时候,表示在源代码中标记或隐式结束,也就是final修饰的内部类在生成字节码的时候内部类的标识没有被分配，默认值为0，一般情况下在jvm实现中没有检查内部类属性与类文件的一致性
