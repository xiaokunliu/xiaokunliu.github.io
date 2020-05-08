---
title: python之OOP编程
category: Python
date: 2018-04-12 23:21:11
tags: Python
---

<!-- more -->


##### 面向对象编程OOP
> python类

* 继承：继承顶层类的通用属性，并且在通用情况下实现一次，目的是提高代码重用性
* 组合：由多个组件对象组合而成，通过多个对象来协作完成相应指令，每个组件都可以写成类来定义自己的属性和行为
* 与模块的区别：内存中模块只有一个实例，只能通过重载以获取其最新的代码，而类有多个实例

> python类具体特征

* 多重实例
    * 类是产生对象实例的工厂
    * 每调用一次类，便会产生独立命名空间的新对象
    * 每个对象存储的数据是不一样
* 通过继承进行定制
    * 在类的外部重新定义其属性
    * 建立命名空间的层次结构来定义类创建的对象所使用的变量名称
* 运算符重载
    * 根据提供特定的协议方法，可以定义对象来响应在内置类型上的几种运算

> python多重实例

* s1:定义类模板和创建实例对象

```python
## person.py
## 定义类模板
class Person:
	## python函数中在默认第一个有参数之后的任何参数都必须拥有默认值
	def __init__(self,name,job=None,pay=0):
		self.name = name
		self.job = job
		self.pay = pay
		
	def __str__(self):
	    return "the name[%s] - the job[%s] - the pay[%d]" % (self.name,self.job,self.pay)

## 创建实例并测试
tom = Person("tom")
jack = Person("jack",job="market",pay=20)
print(tom)
print(jack)

>>> python person.py
the name[tom] - the job[None] - the pay[0]
the name[jack] - the job[market] - the pay[20]

## 小结：
1）创建实例的时候使用了默认值参数以及关键字参数
2）tom和jack从技术的角度而言是属于不同的对象命名空间，都拥有类创建的独立副本
3）不足在要导入其他的python文件也会把print的信息也输出

## 更改存放测试代码的位置，在文件底部添加如下代码
if __name__ == '__main__':
    tom = Person("tom")
    jack = Person("jack",job="market",pay=20)
    print(toml)
    print(jackl)

>>> python person.py
the name[toml] - the job[None] - the pay[0]
the name[jackl] - the job[market] - the pay[20]

>>> import person
## 没有任何输出
```

* s2:添加行为方法

```python
## python OOP编程也应当遵循封装的特性，即细节隐藏，外部访问方法

## 非OOP的方式获取last name
print(jack.name.split()[-1])

## OOP的准则为添加方法
class Person:
    ....
    
    def getLastName(self):
        return self.name.split()[-1]

## 外部调用
print(jack.getLastName()
```

* s3:运算符重载

```python
## 上述测试实例对象每次都需要去调用相应的属性名称打印显示出来，重写__str__方法来显示一个对象属性的信息
class Person:
    ...
    def __str__(self):
	    list = []
		for k,v in self.__dict__.items():       ## 通过动态获取实例对象的属性信息,而不是直接将属性名以及值直接硬编码
			list.append("%s -- %s" % (k,v))
		str =  ",".join(list)
		return "Person[%s]" % str
        
## 上述打印出所有的对象属性信息出来
>>> print(tom)
Person[pay -- 0,name -- tom,job -- None]
```

* s4:通过子类定制行为

```python
## 扩展父类的方法，而不是去重写或者修改父类方法
class Person:
    ....
    def giveRaise(self,percent):
        self.pay = self.pay * (1 + percent)

class Manager(Person):
     def giveRaise(self,percent,bouns=.10):
        ## 调用类的方法并传递self参数,这里不能用self,self本身指当前类实例对象本身，会导致循环引用而内存耗尽
        Person.giveRaise(self,percent + bouns)      
    
## 对于python调用父类方法，一般是通过类.方法(self,parameters)来显示调用，像其他编程语言，如java调用父类是通过super.方法来显示调用，这是区别

## 多态
p1 = Person("p1",job="dev",pay=11000)
p2 = Manager("p2",job="dev",pay=11000)
>>> print(p1.giveRaise(.10))        
>>> print(p2.giveRaise(.10))
Person[name -- p1,job -- dev,pay -- 12100.000000000002]
Person[name -- p2,job -- dev,pay -- 13200.0]

## 小结：
1）python的多态和其他编程语言略有差异，只要方法名称一样便会覆盖顶层类相应的方法，不管参数个数还是参数类型
2）原因在于python的属性以及行为是根据搜索树来遍历获取最近的属性或者方法
3）python参数类型是在赋值的时候才知道的，没有预先定义的类型
```

* s5:定制构造函数

```python
## 定制Manager类，使其特殊化
class Manager(Person):
    def __init__(self,name,pay = 0):
	    Person.__init__(self,name,job = "manager",pay = pay)

>>> tom = Manager("tomk",pay=11000)
>>> print(tom)
Person[pay -- 13200.0,name -- tomk,job -- manager]
```

* s6:使用内省工具(类似其他编程语言的"反射")

    * 问题描述1：Manager打印显示的信息是Person，然后实际中应当是显示Manager的信息
    * 问题描述2：Manager如果在__init__增加属性，如果是硬编码的话则通过打印出来的信息就无法显示新的属性信息

```python
## 问题1的解决方案：
实例对象到创建它的类的链接通过instance.__class__属性
反过来,可以通过一个__name__或者是__bases__序列提供超类的访问    

p1 = Person("p1",job="dev",pay=11000)
p2 = Manager("p2",pay=11000)
>>> print(p1.__class__.__name__)
Person
>>> print(p1.__class__.__bases__)
(<class 'object'>,)

>>> print(p2.__class__.__name__)
Manager
>>> print(p2.__class__.__bases__)
(<class '__main__.Person'>,)


## 问题2的解决方案：
通过内置的object.__dict__字典来显示实例对象的所有属性信息

## 在上述person已经使用__dict__属性来显示实例属性信息
>>> print(p1)
Person[job -- dev,name -- p1,pay -- 11000]

## 新增加对象属性
p1.age = 10            
p1.hobby = "maths"      
>>> print(p1)
Person[pay -- 11000,hobby -- maths,age -- 10,job -- dev,name -- p1]

## 将上述Person改造成通用的工具显示
class Person:
	def get_all_attrs(self):
		attrs = []
		for key,value in self.__dict__.items():
			attrs.append("%s==%s" % (key,value))
		return ",".join(attrs)

	def __str__(self):
		return "%s[%s]" % (self.__class__.__name__,self.get_all_attrs())

>>> print(p1)
Person[hobby==maths,name==p1,job==dev,pay==11000,age==10]
>>> print(p2)
Manager[name==p2,pay==11000,job==manager]

## 显示类和实例对象的所有属性使用dir方法

## 一份完整的OOP
https://github.com/xiaokunliu/python-code/tree/master/01base/OOP
```

* s7:对象持久化
    * pickle:任意python对象和字节串之间的序列化
    * dbm:实现一个可通过键访问的文件系统，以存储字符串
    * shelve:使用上述两个模块把python对象存储到一个文件中，即按键存储pickle处理后的对象并存储在dbm的文件中

```test.py
## pickle
## 将对象序列化到文件
f1 = open("pickle.db","wb+")
pickle.dump(p1,f1)  ## 这里不能一步到位，即open("pickle.db","wb+")，会导致pickle在读取的时候抛出EOFError: Ran out of input 
f1.close()

## 将对象序列化为字符串
string = pickle.dumps(p1)

## 从文件读取
f = open("pickle.db","rb")
p = pickle.load(f)

## 从字符串读取
p_obj = pickle.loads(string)

## dbm
## 存储
db = dbm.open("dbm","c")
db[k1] = v1
db.close()

## 读取
db = dbm.open("dbm","c")
for key in db.keys():
    print("key[%s] -- %s" % (key,db[key]))

## shelve
import shelve
db = shelve.open("persondb")    ## filename
for object in [p1,p2]:
	db[object.name] = object
db.close()  ## 必须关闭

## 从db文件中读取
db = shelve.open("persondb")        ## db拥有和字典相同的方法，区别在于shelve需要打开和关闭操作
for key in db.keys():
	print("from db[%s]" % db[key])
```

> python重载运算符
    
* 双下划线命名的方法(`__X__`)是特殊的钩子
* 当实例出现在内置运算时，这类方法会自动调用
* 类可覆盖多数内置类型的运算
* 运算符覆盖方法没有默认，而且也不需要
* 运算符可以让类与Python的对象模型项集成

```python
## 重载类的内置方法，一般是应用于数学类对象的计算才需要重载运算符
class OverrideClass(ClassObject):
	def __init__(self,data):
		self.data = data

	def __add__(self, other):
		return OverrideClass(self.data + other)

	def __str__(self):
		return "[OverrideClass.data[%s]]" % self.data

	def mul(self,other):
		self.data *= other
		
## 调用
t = OverrideClass("9090")     ## 调用__init__方法
t2 = t+"234"                  ## 调用__add__方法，这个产生了新的对象
print t2                      ## 调用__str__方法

t2.mul(3)
print(t2.data)
```

> python属性继承搜索,object.attribute

* 找出attribute首次出现的实例对象
* 然后是该对象之上的所有类带有`__init__`方法中定义的属性查找attribute，由下至上，由左至右，属于继承搜索树
* 最后是定义该对象的类属性,查找方式也是由下至上，由左至右的遍历搜索

> 编写类树

* 每个class语句生成一个新的类对象
* 每次类调用，就会生成一个新的实例对象
* 实例对象自动连接到创建该实例对象的类
* 类连接至超类的方式，将超类列在类的头部括号中，其从左至右的顺序会决定树中的次序

```python
class M1:
	def __init__(self):     ## 相当于构造器
		self.name = "m1 name"
		print("M1 class")

class M2:
	def __init__(self):
		self.name = "m2 name"
		print("M2 class")

class M3(M1,M2):
	pass

class M4(M2,M1):
	pass

#搜索树：M3 M1 M2,在多重继承中，以括号从左到右的次序会决定超类搜索的顺序
>>> a = M3()
>>> print(a.name) 
M1 class
m1 name

#搜索树：M4 M2 M1
>>> b = M4()
>>> print(b.name)
M2 class
m2 name

## a.name属性的查找
1.先查找当前实例对象的属性,定义对象属性是在一个特殊方法__init__定义,
1.1因此会从M3 -> M1 -> M2的搜索树中查找最近定义的__init__方法
1.2 如果__init__方法中有定义属性name则返回,否则进行下一步查找
2.如果在对象属性中没有找到,则会从类的搜索树中查找属性值，即M3 -> M1 -> M2中查找
2.1若找不到则抛出异常
```

> python OOP总结

* 实例创建 -- 填充实例的属性
* 行为方法 -- 在类方法中封装逻辑
* 运算符重载 -- 为外部调用的程序提供自身内置方法的定制化行为
* 定制行为  -- 重新定义子类方法以使其特殊化
* 定制构造函数 --  为子类添加构造逻辑特殊化

