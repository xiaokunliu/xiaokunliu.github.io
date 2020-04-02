---
title: python3.x之super()用法小结
category: Python
date: 2019-12-31 23:21:11
tags: Python
---

<!-- more -->

##### 传统调用父类的方式

> 传统父类方法调用

```python
# super.py
class SuperClass(object):
	def act(self):
		print("Super Class act method ...")


class SubClass(SuperClass):
	def act(self):
		SuperClass.act(self)        # 显式地调用父类方法
		print("SubClass act method ...")

		
if __name__ == '__main__':
    sub = SubClass()
    sub.act()

>>> python super.py  
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA0ODA2NTc2?x-oss-process=image/format,png)

##### super基本使用

> super()方法只能在py3.x中使用，py2.x必须要指明类名

```python
# super.py
class SuperClass(object):
	def act(self):
		print("Super Class act method ...")


class SubClass(SuperClass):
	def act(self):
		super().act()
		# super(SubClass,self).act()     # py2.x 调用父类，过于复杂，这个可以正常执行
		print("SubClass act method ...")

		
if __name__ == '__main__':
    sub = SubClass()
    sub.act()

>>> python super.py     
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA0ODE4OTg4?x-oss-process=image/format,png)

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA0ODMwNzgw?x-oss-process=image/format,png)
> super在多继承中的使用

```python
# super.py
class A(object):
	def act(self):
		print("call A act method ....")


class B(object):
	def act(self):
		print("call B act method ....")


class C(B,A):       # 搜索的顺序会从B开始搜索，B存在act方法则调用并返回，不再继续搜索
	def act(self):
		super().act()

if __name__ == '__main__':
    c = C()
	c.act()

>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA0ODQ5MjAw?x-oss-process=image/format,png)

> super不支持运算符操作，如super()[index]

```python
# super.py
class E(object):
	def __getitem__(self, item):
		print("call E __getitem__ method ...")


class F(E):
	def __getitem__(self, item):
		print('call F __getitem__ method ..')
		E.__getitem__(self, item)
		super().__getitem__(item)
		super()[item]       # 不支持运算符操作，'super' object is not subscriptable

def test_opr():
	f = F()
	f[2]
	
if __name__ == '__main__':
    test_opr()

>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA0OTA1OTI5?x-oss-process=image/format,png)

##### super的作用

> 在运行时可以改变类的继承树

```python
# super.py
class X(object):
	def m1(self):
		print("call X method")

class Y(object):
	def m1(self):
		print("call Y method")

class Z(X):
	def m1(self):
		super().m1()

def test_z():
	z = Z()
	z.m1()
	print("change Z class base for Y ....")
	Z.__bases__=(Y,)
	z.m1()

if __name__ == '__main__':
	test_z()
	
>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA0OTE5Nzc1?x-oss-process=image/format,png)

> 能够在多继承中按照广度优先的继承树搜索关系链来有序地调用与父类相同名称的方法，且在每个子类拥有super的都会执行父类方法

```python
# super.py
class IA(object):
	def __init__(self):
		print("call IA init ...")   # 基类IA没有super()，没有意义


class IB(IA):
	def __init__(self):
		print("call IB init ....")
		super().__init__()


class IC(IA):
	def __init__(self):
		print("call IC init ....")
		super().__init__()


class ID(IC,IB):
	def __init__(self):
		print("call ID init ...")
		super().__init__()


def test_ID():
	d = ID()
	print(ID.mro())

if __name__ == '__main__':
    test_ID()

>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA0OTM4MDI0?x-oss-process=image/format,png)

**纠正一下，上面不是深度优先而是广度优先**

##### super()多继承方法协作的使用注意事项

> 在父类没有使用super()情况下，在子类中将使用类的继承树搜索方式来匹配父类方法，找到并返回调用，不再继续往下搜索

```python
# super.py
class P1(object):
	def __init__(self):
		print("call P1 init ...")


class P2(object):
	def __init__(self):
		print("call P2 init ...")


# 以下两种方式等价
class S(P2,P1):
	pass    # 默认调用父类的构造方法,如果有定义的话并且子类没有调用super()将不会调用父类方法


class S(P2,P1):
    def __init__(self):
        super().__init__() 

if __name__ == '__main__':
    S()

>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA0OTUzNDkz?x-oss-process=image/format,png)

> 子类的继承树最近的父类如果没有使用super(),则调用链停止,如果使用super()将以广度优先的方式进行遍历,通过mro()查看调用链

```python
# super.py 用上述的IA,IB,IC,ID做示例
...

class IC(IA):
	def __init__(self):
		print("call IC init ....")
		# super().__init__()    # 调用链停止

...
>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA1MDA3Mzgy?x-oss-process=image/format,png)

> 子类多继承中,按上述所说的,如果父类存在super()调用，将以广度优先地处理方式进行搜索,比如

```python
# super.py
class P1(object):
	def __init__(self):
		print("call P1 init ...")
		super().__init__() 


class P2(object):
	def __init__(self):
		print("call P2 init ...")
		super().__init__()

class S(P2,P1):
    # 最先搜索到的父类是P2,如果P2有使用super()来调用,那么就会以MRO的方式来搜索并继续往下调用,否则将停止调用
	pass


def test_s():
	s = S()
	S.mro()
    
    
if __name__ == '__main__':
    test_s()
    
>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA1MDIzMTk5?x-oss-process=image/format,png)

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA1MDMzOTA0?x-oss-process=image/format,png)
> 在多继承中,最后的一个父类的顶层基类对应的super()将断开关系链,因为MRO的继承树最右边是object对象,不会继续往下调用

* 如上述的P1,P2,P1是属于仅次于object的顶层基类,调用super()不会指向其他的基类
* P2和P1是两个没有任何关系的基类,但是在S类的MRO中,P2的super()便是P1(),因此在P2中使用super()会调用到P1的方法
* 对于P1没有super(),在实际编码中应当删除调用super()的语句,避免混淆

> super()在多继承多个方法中的使用

```python
# super.py
class MixSuperA(object):
	def m1(self):
		print("call mix super class A m1 method ...")


class MixSuperB(object):
	def m2(self):
		print("call mix super class B m2 method ...")


class SubClassA(MixSuperA):
	def m2(self):
		print("call sub class A m2 method ...")


class SubClassD(MixSuperA,MixSuperB):
	def m1(self):
		print("call sub class D m1 method ...")
		super().m2()    # 以MRO的搜索方式遍历
		super().m1()    # 以MRO的搜索方式遍历

if __name__ == '__main__':
    d = SubClassD()
    d.m1()  

>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA1MDQ5Mjkw?x-oss-process=image/format,png)

> super调用的参数约束

* super调用的方法必须存在

```python
# super.py
class Person(object):
	def __init__(self,name,age):
	    print("call person init ...")
		self.__name = name
		self.__age = age

class Student(Person):
	def __init__(self,age):
	    print("call Student init ...")
		super().__init__("keithl",age)      # 这里的super()使用MRO方式进行搜索，对应是Son,而Son只提供一个参数age的传递

class Son(Person):
	def __init__(self,age):
		print("call son init ...")
		super().__init__("keithl",age)
		
class Me(Student,Son):
	pass

if __name__ == '__main__':
    Me(27)

>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA1MTAzMzg1?x-oss-process=image/format,png)

* 为避免复杂化,定义规范:在每一个被super调用的方法参数签名必须与基类的定义的方法一致

```python
# super.py
...
class Student(Person):
	def __init__(self,name,age):
    	print("call Student init ...")
		super().__init__(name,age)  

class Son(Person):
	def __init__(self,name,age):
		print("call son init ...")
		super().__init__(name,age)
...
>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA1MTE1Nzcw?x-oss-process=image/format,png)

* 为避免复杂化以及多继承中的方法混淆,最好是在MRO中最后一个调用的方法链中使用super()去覆盖基类的方法,其他的不要使用super()覆盖

```python
...
class Student(Person):
    pass    # Student不做定义

class Son(Person):
	def __init__(self,name,age):
		print("call son init ...")
		super().__init__(name,age)
...
>>> python super.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjA1MTI5Njkw?x-oss-process=image/format,png)
##### super()小结

* super()方法在py3.x中可用,py2.x是使用super关键字并传入父类名称和子类self对象
* super()调用方法将以MRO的搜索方式进行关系链的调用
* 在子类中如果没有使用super()将停止关系链的调用
* 在MRO最右边的顶层基类中不要声明super()语句调用
* 混合方法使用super()仍然以上述规则来查询
