---
title: python类编写细节
category: Python
date: 2018-04-11 23:21:11
tags: Python
---

<!-- more -->


##### 类编写细节
1.class 语句

> class语句细节

* python的class语句是属于OOP的一种工具(即定义变量名的工具，将数据和逻辑暴露给客户端)，而不是声明式的
* class语句是对象的创建者，类似于对象工厂
* class语句是一种隐含的赋值运算，即执行class语句时，会产生类对象并且将其引用存储到定义的类名称上
* class语句与def一样，都是可执行语句，即python还没有执行到class语句时，类是不存在的
* class是复合语句，所有种类语句都可以位于其主体内,如print, 赋值语句, if, def...

> class语句如何得到命名空间

* 首先，执行类语句的时候，会从头至尾执行其主体内的所有语句
* 其次，是在执行过程中的赋值运算会在这个类作用域中创建变量名，从而成为对应的类对象属性
* 与函数相比,可以把class语句看成一个本地作用域,在class语句下定义的变量就属于这个本地作用域


* 与模块相比,定义的变量名是可以共享的并且成为当前类的对象属性


> class语句一般形式 

```python
## 根据上述所言，className是类对象的一个引用
class className(superclass1,superclass2,...):          
    '''
        定义类属性,属于所有实例的共享数据,通过类语句下进行定义和创建
    '''
    class_attr = value 
    
    
    '''
        定义实例方法以及实例属性
    '''
    def method(self,data):      ## 定义实例方法
        self.attr = data        ## 设置实例属性,通过带有self的方法来分配属性信息
```

2.方法

> 实例方法对象调用等价于类方法函数调用

```python
## python自动将实例方法的调用自动转成类方法函数,并传递实例对象作为第一个参数传递
class Person:
    def study(self,name):
        print("%s study method in for %s" % (name,self.__class__.__name__)

>>> p = Person()
>>> p.study("keithl")
keithl study method in for Person

>>> Person.study(p,"keithl")
keithl study method in for Person

## instance.method(arg1,arg2,...) == class.method(instance,arg1,arg2,...)
```

> 调用超类的构造函数`__init__`方法

```python
class Person:
	def __init__(self):
		print("call person init ....")

class Student(Person):
	pass
		
>>> s = Student()               ## 创建子类时会调用父类构造函数,原因是子类没有定义自己的构造函数
call person init ....

## 为子类增加构造函数
class Student(Person):
	def __init__(self):
		print("call student init ....")

>>> s = Student()               ## 只输出子类的__init__方法,并没有调用父类方法,原因在于python是根据命名空间来执行调用方法
call student init ....

## 若要调用父类构造方法则必须显示进行调用
class Student(Person):
    """
        必须在子类构造函数中显式调用父类的构造函数,并传递子类的self引用
    """
	def __init__(self):
		print("call student init start....")
		Person.__init__(self)              
		print("call student init end....")

>>> s = Student()               
call student init start....
call person init ....
call student init end....
```

> 静态方法

* 使用场景：
    * 目标：为所有类实例提供数据共享的类属性
    * 执行：通过类名称访问类属性
    * 优化：其一是使用OOP思想封装类属性而对外提供方法,其二是考虑扩展性,通过继承来定制
    * 落地：使用静态方法或者类方法,即不需要传递类对象self实例参数的方法

```python
## person.py
class Person:
	num = 1
    """
        定义一个没有带参数的普通方法
    """
	def printNum():
		Person.num += 1
		print("the number is %s" % Person.num)
	
	printNum = staticmethod(printNum)                   ## 声明为静态方法
    
    """
        定义一个带参数的普通方法,此参数为类对象参数
    """
	def clsPrintNum(cls):
		Person.num += 1
		print("the number is %s" % Person.num)         

	clsPrintNum = classmethod(clsPrintNum)              ## 声明为类方法

>>> Person.printNum()
the number is 2

>>> Person.clsPrintNum()
the number is 3

## person.py 使用装饰器来声明静态或类方法
class Person:
	num = 1
	
	@staticmethod
	def printNum():
		Person.num += 1
		print("the number is %s" % Person.num)

	@classmethod
	def clsPrintNum(cls):
		Person.num += 1
		print("the number is %s" % Person.num)
```

> 静态方法、类方法与实例方法

* 类中带有实例对象self的参数传递的方法称为实例方法
* 类中带有类对象cls的参数传递的方法并通过函数classmethod或者装饰器@classmethod声明的方法称为类方法
* 类中没有实例对象self和类对象cls参数传递的方法，且通过staticmethod或装饰器@staticmethod什么的方法称为静态方法

```python
class Person:

	@staticmethod
	def static_method():
		print("static method ...")

	@classmethod
	def class_method(cls):
		print("class method ....")

	def instance_method(self):
		print("instance method ...")

	'''
		python3.x可以调用下面的函数，可以说是静态方法,但严格意义上是属于类的一个行为方法，但是python2.x无法该方法
	'''
	def fn():
		print("just a fn,if py3.x,it is static method")

## 总结:
1)在类中定义方法一定要规范化,明确是静态方法还是类方法抑或是实例方法
2)避免使用最后一种方式在类中定义方法
```

3.命名空间与作用域

* 命名空间：用于记录变量的轨迹,key是变量名称,value是变量值,作用就是根据变量名称搜索变量
    * 使用无点号运算的变量名称(X),将根据LEGB(local/enclosing/global/builtin)作用域查找法则来搜索变量
    * 使用点号的属性名称(object.x)使用的是对象命名空间来搜索变量（对象：类的实例对象和类对象）
    * 有些作用域会对对象的命名空间进行初始化(模块和类)

> 无点号运算的变量名称

* 赋值语句:在当前作用域创建或更改变量X,除非声明为全局变量

```python
X = "global X"
def enclosing_fn():
    ## global X             
    X = "enclosing fn"      ## 创建当前enclosing_fn的本地变量X如果没有声明为全局变量的话
```

* 引用:根据LEGB作用域法则来搜索变量

```python
X = "global X"
def enclosing_fn():
    X = "enclosing fn"      ## 如果注释此行,将打印全局的变量X
    print(X)
    def local_x()
        x = "local x"       ## 如果仅注释此行,将会打印嵌套的变量X
        print(x)
    local_x()
```

> 点号的属性变量名称

* 赋值语句:在对应的对象命名空间中创建或修改属性名称X,即object.X = value

```python
>>> p = Person()

## 在对象实例的命名空间创建或更改属性名称name
p.name = "keithl"       ## 并无进行变量名称的搜索

## 在类的命名空间中创建或更改属性名称name
Person.name = "keithl"  ## 并无进行变量名称的搜索
```

* 引用
    * 基于类的对象引用：会在对象内搜索属性名称X,若没有找到则根据继承搜索来查找
    * 基于模块对象的引用：先导入模块,再从模块中读取X

```python
>>> p = Person()
>>> p.name          ## 从对象命名空间开始按照继承树来搜索
>>> Person.name     ## 从类的命名空间开始按照继承树来搜索
```

> 命名空间字典

* 模块的命名空间是以字典的形式实现的，并且可以由属性`__dict__`来显示
* 类和对象可以看成一个带有链接的字典,属性点号就是字典索引运算,属性继承就是搜索链接的字典
    * 实例与类通过`__class__`属性链接
    * 类与超类通过`__bases__`属性链接,可以通过递归往上遍历超类
* 都可以通过`__dict__`查看模块、类或者对象的属性信息

> 类与模块的关系总结

* 类
    * 调用类会创建新的对象
    * 由class来创建类对象
    * 通过调用来使用
    * 属于模块的一部分

* 模块
    * 是数据和逻辑包
    * 通过py抑或其他语言来扩展
    * 必须导入才能使用
   
