---
title: python类设计浅谈
category: Python
date: 2018-04-01 23:21:11
tags: Python
---

<!-- more -->


##### 类设计浅谈

1.抽象类设计

* 抽象方法：在编写class语句中，若有存在未定义或是未实现的或是@abstractmethod装饰器修饰的
* 抽象类：class中定义有抽象方法的类

```python
## abs.py

## 第一种形式,伪抽象类,
class Super(object):
    def delegate(self):
        self.action()               ## action具体实现方法由子类定义z
    
    def action(self):
		raise NotImplementedError('action must be defined!')

class Sub(Super):
    def action(self):
        print("call sub action ...")

## 第二种形式
## py3.x
from abc import abstractmethod,ABCMeta

class Super(metaclass=ABCMeta):     ## 定义为抽象类
    def delegate(self):
        self.action()
    
    @abstractmethod
    def action(self):
        pass

class Sub(Super):
    def action(self):
        print("call sub action ...")
        

## py2.x
from abc import abstractmethod,ABCMeta

class Super:                
    __metaclass__ = ABCMeta   ## 通过类属性声明
    def delegate(self):
        self.action()
    
    @abstractmethod
    def action(self):
        pass

class Sub(Super):
    def action(self):
        print("call sub action ...")
        
## 第二种形式的Super不能够进行实例化,第一种形式可以进行实例化
```

2.类的继承与组合设计

* 继承是属于"is a"的强依赖关系,拥有父类的一系列属性和方法
* 组合是属于"has a"的弱依赖关系,也称为聚合,由各个组件组合而成共同完成一项任务目标

```python
## task.py 
## 继承
class Employee:
	def __init__(self, name, age):
		self.__name = name  ## 定义私有属性,严格意义上是属于伪属性
		self.__age = age

	def get_name(self):
		return self.__name

	def get_age(self):
		return self.__age

	def work(self):
		print("employ work ...")


class Engineer(Employee):
	def work(self):
		print("engineer coding ...")


class ProductManager(Employee):
	def work(self):
		print("product manager desgin ...")


## 定义组合
class Customer:
	def __init__(self, name):
		self.__name = name

	def post_requirement(self):
		print("the %s post requirement ..." % self.__name)
		
class ProjectGroup:
	def __init__(self, engineer, manager):
		self.__engineer = engineer
		self.__manager = manager

	def build_dtrees(self, customer_name):
		customer = Customer(customer_name)
		customer.post_requirement()
		self.__manager.work()
		self.__engineer.work()

>>> engineer = Engineer("keithl",27)
>>> manager = ProductManager("keithl",27)
>>> pg = ProjectGroup(engineer,manager)
>>> pg.build_dtrees("xiaoming")

the xiaoming post requirement ...
product manager desgin ...
engineer coding ...
```


3.类的委托(代理)

* 对原有对象的方法进行增强，即在原有的方法基础上，丰富方法的行为
* 重载`__getattr__`内置方法并使用`getattr`方法来回调被包装类对象的属性方法

```python
## wrapper.py
class Wrapper:
    def __init__(self,object):
        self.__wrapper = object     ## 传递被包装的对象
    
    def __getattr__(self,attrname):
        print("call wrapper for attr[%s] ...." % attrname)
        return getattr(self.__wrapper,attrname)
    
    def __str__(self):
		return str(self.__wrapper)
        
>>> engineer = Engineer("keithl",27)
>>> we = Wrapper(engineer)
>>> we.work("dtrees")
call wrapper for attr[work] ....
engineer review and coding ...
```

4.伪属性

* 保证定义的类中的属性(类属性和实例属性)名称唯一,即使在同名属性的情况下也能够区分是属于哪一个类中定义的属性
* 在属性的名称前面添加`__`双下划线,后面不添加下划线,py会将这类属性转换为`_className__attrName`
* 可以看成是私有属性,即对外暴露的属性名称不再是定义的属性名,而是`_className__attrName`
* 使用伪属性是保证唯一性,防止在多继承过程中不同的子类命名相同而冲突

> 定义Person类私有的name属性

```python
## private.py
class Person:
	__template_name = "person instance template name"

	def __init__(self,name):    
		self.__name = name      ## __name 属于Person类,

	def get_name(self):
		return self.__name

	@staticmethod
	def get_template_name():
		return Person.__template_name

>>> p = Person("keithl")
>>> print(p.get_name())
keithl

>>> print(p._Person__name)
keithl

>>> print(p.__name)
AttributeError: 'Person' object has no attribute '__name'

>>> print(dir(p))
```
截图

> 定义Person的子类各自的私有属性

```
## sub.py
class Manager(Person):
    def __init__(self,name):
		self.__name = name      ## __name 属于Manager类
	

class Engineer(Person):
    def __init__(self,name):
		self.__name = name      ## __name 属于Engineer类

>>> m = Manager("manager")
>>> e = Engineer("engineer")

## 打印的结果如下图所示
>>> print(dir(m)) 
>>> print(dir(e))
```
截图

5.方法的绑定与未绑定

* 未绑定对象的方法：不带有self参数的方法,通过定义的类来调用函数并返回一个没有绑定self的方法
* 绑定对象的方法：带有self参数的方法,即实例方法，通过实例对象调用函数并返回一个绑定self参数的方法

```python
## fn.py
class BoundClass:
    def action(self):
        print("bound class action ....")
    
    """
        python3.x未绑定self对象的方法均是类定义的函数,注意这个还不是属于静态方法和类方法
    """
    def unbound(num):               ## work in py3.x,fail in py2.x
        print("the number is %d" % num)
    
>>> t = BoundClass()
>>> t.action()              ## action方法绑定实例对象self,可以直接通过点号运算调用  

>>> m = BoundClass.action   ## m是没有绑定实例对象self的方法
>>> m(t)                    ## 调用需要传递实例对象t
```

6.对象工厂方法

* 使用场景
    * 将代码与动态配置对象构造器的实现细节分离 
    * 可以通过配置文件的内容在运行时动态创建实例对象
    * 工厂方法允许我们使用未定义的类来避免硬编码的方式进行类的导入和传递

```python
## fa.py
def factory(aClass,*pargs,**kwargs):    ## aClass 具体是什么类不清楚，只有在运行的时候才知道
    return aClass(pargs,kwargs)

## run-fa.py
"""
现在有一个配置文件setting.py,主要是用作配置数据库的一系列参数,其中需要指定一个类是表明使用哪种数据库,如MySQL、Oracle...
假设在一个db.py定义了实现不同的数据库对应的类模板
"""

import setting.py                  ## 读取配置文件

classargs = setting_db_config      ## 获取实例化连接对象的数据库配置参数,uri,port,username,password

classname = setting_db_class       ## 获取类名称的字符串

aClass = getattr(db,classname)     ## 从模块中获取类对象

connection = factory(aClass,classargs)  ## 传递参数创建类实例对象
```