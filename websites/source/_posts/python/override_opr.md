---
title: python类重载运算符 
category: Python
date: 2020-04-01 23:21:11
tags: Python
---

<!-- more -->


#### python类重载运算符

1.成员操作符:`__contains__`

> 判断member是否存在**可迭代的对象**中

```python
# membership.py
# 定义的对象必须是可迭代的
class Iters:
"""
   将所有可迭代的方法列出来 
"""
	def __init__(self, value):
		self.data = value

	def __getitem__(self, i):    # 迭代、索引、分片操作的时候会被调用
		print('get[%s]:' % i, end='')
		return self.data[i]

	def __iter__(self):          # 进行迭代的时候会被调用
		print('iter=> ', end='')
		self.ix = 0
		return self

	def __next__(self):
		print('next:', end='')
		if self.ix == len(self.data):raise StopIteration
		item = self.data[self.ix]
		self.ix += 1
		return item

	def __contains__(self, x):      # 成员变量的判断，使用in
		print('contains: ', end='')
		return x in self.data

	next = __next__

def test_iter():
    X = Iters([1,2,3,4])
    print(3 in X)   # contains: True
    for i in X:
		print(i, end=' | ')

if __name__ == '__main__':
    test_iter()

>>> python membership.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxNjAyMDg4?x-oss-process=image/format,png)

2.属性管理

> 属性访问`__getattr__`

* obj.attr_name的形式访问对象属性
* 自定义类管理属性信息,对于访问不存在的属性可以返回错误信息提示

```python
# attr.py
class AttrObject:
	def __getattr__(self, attr_name):
		print("Attr Object get attr method ....")
		if attr_name == 'age':
			return 40
		else:
			raise AttributeError("could not access the attr[%s]" % attr_name)

# for test
def test_empty():
	e = AttrObject()
	print(e.age)        # 调用__getattr__方法,访问age属性,匹配并返回40
	print(e.hobby)      # AttributeError: could not access the attr[hobby]

if __name__ == '__main__':
    test_empty()

>>> python attr.py      # 2.x & 3.x表示结果一样,以下都会以此标记
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxNjE5NTM5?x-oss-process=image/format,png)

> 属性设置`__setattr__`

* 定义类对象管理属性并将属性名以及值存储在一个实例对象中
* self.attrname 以及 instance.attrname 将会调用类的内置方法`__setattr__`方法

```python
# attr.py
class AcessControl:
	def __init__(self):
		self.hobby = "basketball"               # 会调用下面的__setattr__方法

	def __setattr__(self, key, value):
		# self.name = "xxxx"                    # 不能在__setattr__上使用self.attr,会导致递归应用循环
		print("access control set attr ...")
		if key == 'age':
			self.__dict__[key] = value + 10     # 通过内建字典来保存属性数据
		else:
			self.__dict__[key] = value

	def __delattr__(self, item):
		print("del item[%s]" % item)

	def __getattr__(self, item):
		print("get item[%s]" % item)

def test_access_control():
	ac = AcessControl()
	ac.age = 10         # 调用__setattr__
	print(ac.age)       # 直接输入值，没有调用__getattr__
	print(ac.hobby)     # 当属性有值时，也就是非None是不会调用__getattr__方法的，如果没有值，即None就会调用__getattr__方法
	del ac.age          # 调用__delattr__
	print(ac.name)      # 调用__getattr__,调用未定义的属性时候就会回调这个函数并且返回None

if __name__ == '__main__':
    test_access_control()

>>> python attr.py      # 2.x & 3.x
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxNjM2Mzc0?x-oss-process=image/format,png)

> 属性设置与访问`__getattribute__`

* 这个方法将会拦截所有获取的属性的操作，包括未定义的属性
* 属性内置函数允许我们将方法与特定类属性的获取和集合操作相关联
* 属性描述符为特定的类的属性提供了一组`__get__`和`__set__`方法访问协议
* 实例属性在类中声明但是在每一个类的对象实例中隐式存储

```python
class AttrManager:
	def __getattribute__(self, item):
		"""
		:param item:
		:return:
		避免产生递归循环引用,不要使用self.attr_name对属性进行访问或者设置,属性访问会经过这个方法
		这里没有保存数据，因此返回的时候是None
		"""
		print("call  __getattribute__ for item[%s] " % item)

def test_attr_manager():
	manager = AttrManager()
	manager.name = "attr manager name"
	print(manager.name)
	print(manager.age)

if __name__ == '__main__':
    test_attr_manager()

>>> python attr.py  # 2.x与3.x的不同输出
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxNjU4MTg0?x-oss-process=image/format,png)
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxNzE0NDY1?x-oss-process=image/format,png)
> 小结

* `__setattr__`和`__getattr__`能够在python不同的版本下对未定义的属性进行访问和赋值
* `__getattribute__`只能在3.x下对未定义的属性名进行访问
* 为了更好的兼容2.x和3.x，应当重载两个方法,即`__getattr__`以及`__getattribute__`
* python3.x中的执行的优先级，`__getattribute__` > `__getattr__`
* python2.x中的执行的优先级，`__getattr__` > `__getattribute__`
* 如果类中没有定义`__getattribute__`或者`__getattr__`,则在访问未定义的属性时候会报没有属性的错误

3.字符串打印:`__str__`,`__repr__`

* `__str__`主要应用于print函数以及字符串函数str的转换操作,
* `__repr__`应用于所有输出操作,如果有print以及str操作并定义`__str__`,则会以`__str__`为准
* `__repr__`与`__str__`均未定义的时候，默认打印的是输出对象地址信息

```python
# str.py
class DisplayClass:
	"""
	__repr__ is used everywhere, except by print and str when a __str__ is defined.
	__str__ to support print and str exclusively
	"""
	def __repr__(self):
		return "display __repr__ class"

	def __str__(self):
		return "display __str__ class"


# 使用命令行的形式打印输出  2.x & 3.x 输出效果一致,以2.x作为截图
>>> d = DisplayClass()
>>> d                   repr
>>> print(d)            str
>>> print(repr(d))      repr
>>> print(str(d))       str
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxNzMxMjk1?x-oss-process=image/format,png)
4.运算操作符:`__add__`,`__radd__`,`__iadd__`

> 变量+常量,如X + 1024,调用`__add__`方法

```python
# addition.py
class Counter:
	def __init__(self, value = 0):
		self.data = value

	def __add__(self, other):
		print("counter data[%s] add [%d]..." % (self.data,other))
		return self.data + other

def test_add():
	r = Counter()
	q = r + 1024       # call add, instance +
	print("q is %d" % q)

test_add()
>>> python addition.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxNzQ2MDU0?x-oss-process=image/format,png)
> 常量+变量,如1024 + X,调用`__radd__`方法

```python
# addition.py
class Counter:
    ...     # 省略上述代码
    ...
    def __radd__(self, other):
		print("counter data[%s] radd [%d]..." % (self.data,other))
		return self.data + other

def test_radd():
 	p = 7 + r       # call radd, + instance
	print("the p is %d" % p)
	
test_radd()
>>> python addition.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxODExNTYw?x-oss-process=image/format,png)
> 变量+变量,如X + Y

```python
# addition.py
# 执行X+Y的时候,会先执行__add__方法,再执行__radd__方法,此时代码需要稍微调整下
class Counter:
    def __radd__(self, other):
		if isinstance(other,Counter):
			print("counter data[%s] radd [%d]..." % (self.data,other.data))
		elif isinstance(other,int):
			print("counter data[%s] radd [%d]..." % (self.data, other))
		return self.data + other

	def __add__(self, other):
		if isinstance(other,Counter):
			print("counter data[%s] add [%d]..." % (self.data,other.data))
		elif isinstance(other,int):
			print("counter data[%s] add [%d]..." % (self.data, other))
		return self.data + other

def test_addition():
    X = Counter(2)
	Y = Counter(4)
	sum = X + Y     # X + Y ==> 2 + Y ==> 4 + 2
	print(sum)

test_addition()
>>> python addition.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxODI1MzM3?x-oss-process=image/format,png)

5.回调:`__call__`

> 函数回调设计

```python
# others.py
# 使用函数进行回调
def callback_fn(color):
	def oncall():
		print("select the color[%s]" % color)
	return oncall

def test_callback_fn():
	fn = callback_fn("green")
	fn()

test_callback_fn()
>>> python others.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxODM4ODU0?x-oss-process=image/format,png)

> 匿名函数回调

```python
# others.py
def test_lamaba_callback():
	cs4 = (lambda color='red': print('turn ' + color))
	cs4()

test_lamaba_callback()
>>> python others.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxODUyNDE3?x-oss-process=image/format,png)

> 类回调设计

```python
# others.py
class CallBack:
	def __init__(self,color):
		self.color = color

	def __call__(self):
		print("call callback color[%s]" % self.color)

class EventCall:
	def __init__(self,callback = None):
		self.c = callback

	def press(self):
		self.c()

def test_call_back():
	c1 = CallBack("red")
	c2 = CallBack("green")
	e1 = EventCall(callback = c1)
	e2 = EventCall(callback = c2)

	e1.press()      # when press then trigger callback
	e2.press()

test_call_back()
>>> python others.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxOTA3MDUx?x-oss-process=image/format,png)

6.比较运算符

* py3.x不存在`__cmp__`以及cmp方法,也就是py3.x不能再使用`__cmp__`以及cmp方法,py2.x仍然保留
* 为兼容性,统一在py2.x与py3.x使用`__lt__`和`__gt__`方法

```python
# others.py
class Comp:
	"""
	方式一
	"""
	data = 'spam'

	def __gt__(self, other): return self.data > other

	def __lt__(self, other): return self.data < other

def test_comp():
	c = C()
	print(c > 'ham')
	print(c < 'ham')

test_comp()
>>> python others.py 
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxOTIxNTI4?x-oss-process=image/format,png)

> py2.x可用而py3.x不可用情况

```python
# others.py
class S:
	data = "spam"
	"""
	python3.x将不可用
	"""
	def __cmp__(self, other):
		return cmp(self.data,other)

class D:
	"""
	方式三,改变cmp的方法,3.x仍然报错
	"""
	data = "d"

	def __cmp__(self, other):
		return (self.data > other) - (self.data < other)
		
def test_S():
	s = S()
	print(s > 'ham')

def test_d():
	d = D()
	print(d > 'ham')
	print(d < 'ham')

test_S()
test_d()
>>> python others.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxOTM1MDYw?x-oss-process=image/format,png)
7.布尔测试:`__bool__`,`__len__`,`__nozero__`

* py3.x:如果没有`__bool__`就会执行`__len__`,也就是两个都存在的时候，`__bool__`的优先级比`__len__`高
* py2.x如果没有`__bool__`就会执行`__len__`,也就是两个都存在的时候，`__len__`的优先级比`__bool__`高

```python
# others.py
class Truth:
	def __bool__(self):
		print("call bool method ...")
		return True
    
    def __len__(self):
		print("call len method ....")
		return 0
		
def test_truth():
	X = Truth()
	if X:   
		print("i'm truth ....")

test_truth()
>>> python others.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAxOTUwMzcz?x-oss-process=image/format,png)
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAyMDAyNjUx?x-oss-process=image/format,png)
> `__nonzero__`:仅在python2.x使用，用于返回一个布尔值,并且优先级大于上面的`__len__`方法

```python
# others.py
class Truth:
    # 定义了__bool__方法以及__len__方法
    ....
    
	def __nonzero__(self):
		print "call zero for python2.x"
		return False
		
... # 同样的测试代码

>>> python others.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAyMDE2MzIy?x-oss-process=image/format,png)

8.析构函数:`__del__`

> __del__使用笔记

* Need:即是否必需,python会自动回收内存空间，析构器并非是空间管理的一项必需工具
* Predictability:即是否可预测,python退出解释器的时候，并不能保证会调用仍然存在的对象的析构函数来进行回收操作
* Exceptions:由于存在异常情况并且不知道异常发生的时刻，析构器很难确定什么时候调用来进行回收操作
* Cycles:对象的循环引用可以防止垃圾回收器发生一些如我们所期望的错误

> 代码测试

```python
# others.py
class Life:
	def __init__(self, name='unknown'):
		print('Hello ' + name)
		self.name = name

	def live(self):
		print(self.name)

	def __del__(self):
		print('Goodbye ' + self.name)

def test_del():
	l = Life("keithl")
	l.live()

if __name__ == '__main__':
    test_del()

>>> python others.py        # 2.x & 3.x
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMTIyMjAyMjEwMjM1?x-oss-process=image/format,png)

