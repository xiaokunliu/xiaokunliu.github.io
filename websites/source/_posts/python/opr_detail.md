---
title: python重载运算符详解 
category: Python
date: 2020-04-01 23:21:11
tags: Python
---

<!-- more -->


##### python重载运算符详解
1. 基础
    * 运算符重载能够让类拦截正常的python操作
    * 类能够重载所有的python表达式操作
    * 类也能够重载内置操作,如打印、函数调用抑或是属性访问
    * 重载能够让类的实例看上去更像是内置类型
    * 重载是实现类中提供的特殊方法名，如`__X__`

2. 索引和切片操作，`__getitem__` & `__setitem__`

* `__getitem__`方法被调用的应用场景

> 对象实例根据索引值获取索引值对应的数据

```python
# indexer.py
class Indexer:
    def __init__(self):
        print("call init ...")
        self.__data = [1,2,3]
        
    def __getitem__(self, index):           # index为索引值
        print("call getitem index...")
        return self.__data[index]


def test_index():
    # 取值
    for i in range(3):
    print(X[i])  # 等价于Indexer.__getitem__(X,i)


if __name__ == '__main__':
    test_index()

>>> python indexer.py   # 注：python如果没有说明表示都是3.x版本
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTMyNDMxMzMx?x-oss-process=image/format,png)
> 对象实例根据分片方式获取一组分片索引值对应的数据

```python
# slice.py
class SliceObject:
    def __init__(self):
        print("call slice object init ...")
        self.__data = [1,2,3,4,5,6    ]
        
    def __getitem__(self, index):   # index为索引值或者分片对象
        if isinstance(index, int):
            print('indexing', index)
        else:
            # 分片对象有start stop 以及step属性
            print('slicing', index.start, index.stop, index.step)
        return self.__data[index]

def test_slice():
    X = SliceObject()
    # 分片取值
    print(X[0:2])
    print(X[:-1])
    print(X[1:])

if __name__ == '__main__':
    test_slice()

>>> python slice.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTM1OTA5NDE1?x-oss-process=image/format,png)

> 迭代器调用并进行数据遍历

```python
# indexer.py 
def test_iterator():
    X = Indexer()
    for item in X:
        print(item)

if __name__ == '__main__':
    # test_index()
    test_iterator()

>>> python indexer.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTQwNjA4NzMw?x-oss-process=image/format,png)

> py2.x分片操作的内置方法,py3.x已经删除

```python
# slice.py 
class Slicer:
	"""
	py2.x 分片
	"""
    def __getitem__(self, index):
        print("call __getitem__ for index:%s" % str(index))

    def __getslice__(self, i, j):
        print("call __getslice__ for start[%d] to end[%d]" % (i,j))
    
    def __setslice__(self, i, j, seq):
        print("call __setslice__ for start[%d] to end[%d] with seq[%d]" % (i,j,seq))

def test_slicer():
    Slicer()[1]
    Slicer()[1:9]
    Slicer()[1:9:2]

if __name__ == '__main__':
    test_slicer()

>>> python slice.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTQwODA2MjQ2?x-oss-process=image/format,png)

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTQ0MDQ1NzY4?x-oss-process=image/format,png)
> py3.x新增`__index__`内置方法,该方法表示返回一个实例对象的整数型，与索引切片没关系

```python
# indexer.py
class Indexing:
	"""
py3.x 新增的内置方法，但是该操作不是表示索引操作，而是为一个对象实例返回一个整型数值,并且用于转换为数值字符串的内置操作
	"""
    def __index__(self):
        return 200

def test_indexing():
    X = Indexing()
    # 将对象转换为数值字符串，这里执行两个内置方法,__index__ 和 __str__
    print(hex(X))

if __name__ == '__main__':
    test_indexing()
	
>>> python indexer.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTQ0MzI4OTM2?x-oss-process=image/format,png)

* `__setitem__`对索引属性进行赋值操作

```python
# indexer.py
class Indexer:
	... 
	
    def __setitem__(self, index, value):    # index恒为索引下标,value为值
        print("call setitem ...")
        print("key[%s]--value[%s]" % (index,value))
        self.__data.insert(index,value)

def test_index():
	# 设置值
    X = Indexer()
    X[0] = 1  # 等价于Indexer.__setitem__(X,0,1)

if __name__ == '__main__':
    test_index()

>>> python indexer.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTQ0NDQxODQ5?x-oss-process=image/format,png)
3. 迭代器对象:`__iter__` & `__next__`

> 自定义迭代器

```python
# iterator.py
class Squares:
    def __init__(self, start, stop):
        self.value = start - 1
        self.stop = stop

    def __iter__(self):
        # 生成迭代器对象
        print("call iter ...")
        return self

    def __next__(self):
        if self.value == self.stop:
            raise StopIteration
        print("call next for value:", end='')
        self.value += 1
        return self.value ** 2

def test():
    X = Squares(1, 5)
    # 迭代器不支持使用索引
    X[1]    # TypeError: 'Squares' object does not support indexing

if __name__ == '__main__':
    test()
	
>>> python iterator.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTQ0NjQ3MDY2?x-oss-process=image/format,png)
> 使用循环迭代器

```python
# iterator.py

def test_for():
	# 循环使用迭代器
	for item in Squares(2, 9):  # 调用Squares.__iter__,生成迭代器对象
		print(item)             # 调用Squares.__next__
		
>>> python iterator.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTQ1MzUzNjcy?x-oss-process=image/format,png)
> 逐个进行迭代

```python
# iterator.py

def test_single():
	# 逐个调用迭代器
	X = Squares(2, 9)
	# 显示将X转换为迭代器对象，条件是该对象有实现__iter__，只需调用一次
	I = iter(X)             # Squares.__iter__
	print(next(I))          # 调用Squares.__next__
	print(next(I))          # 调用Squares.__next__

if __name__ == '__main__':
    test_single()

>>> python iterator.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTQ1NDU0MTk0?x-oss-process=image/format,png)

> 迭代器单次扫描

```python
# iterator.py
def single_scan():
    print("start single scan ....")
    X = Squares(1, 2)
    # 首次会执行__iter__对迭代器进行初始化，
    # 并且每次执行将调用__next__将数据返回并保存当前迭代器的状态，直至为不可迭代状态
    list1 = [n for n in X]
    print(list1)
	
    # 迭代器当前是不可迭代的状态，即list2是空列表
    list2 = [n for n in X]       # now list2 is empty
    print(list2)
    print("end single scan ....")

if __name__ == '__main__':
    single_scan()

>>> python iterator.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTU0NjQzNDk0?x-oss-process=image/format,png)

> 迭代器多次扫描

```python
# iterator.py
def mutil_scan():
    print("start mutil scan ... ")
    # 方式一：每次都是重新创建新的迭代器来进行迭代
    list1 = [n for n in Squares(1,2)]
    print(list1)
    list2 = [n for n in Squares(1, 2)]
    print(list2)

    # 方式二 转换为列表并用列表解析器来转换
    list_iter = list(Squares(1,2))
    list3 = [n for n in list_iter]
    print(list3)
    list4 = [n for n in list_iter]
    print(list4)
    print("end mutil scan ...")

if __name__ == '__main__':
    mutil_scan()

>>> python iterator.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTU0NzMyMzk1?x-oss-process=image/format,png)
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-N9x57QY5-1571413260940)(media/15078697725265/15081168372111.jpg)]

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTU1MDEzMjY0?x-oss-process=image/format,png)
> 设计可多次迭代的类对象

* 自定义类可多次迭代
    * 条件：在内置方法中`__iter__`中返回的迭代器应当新的迭代器对象，而不是self自身对象

```python
# iterator.py
class MutilIterator:
    def __init__(self, wrapper):
        self.__wrapper = wrapper

    def __iter__(self):
	    """
	     每次调用迭代器的时候就创建一个新的迭代器并返回
	    """
        return DefineIterator(self.__wrapper)


class DefineIterator:
    def __init__(self, wrapped):
        self.wrapped = wrapped
        self.offset = 0

    def __next__(self):
        if self.offset >= len(self.wrapped):
            raise StopIteration
        else:
            item = self.wrapped[self.offset]
        self.offset += 1
        return item

# 	py2.x 没有next方法,python2.x 和 python3.x 都可用
	next = __next__

def test_mutil_object_iter():
    S = "abcd"
    # 索引遍历操作
    M = MutilIterator(S)
    for x in M:
        for y in M:               # 第二次调用迭代器获取新的迭代器对象 
            print(x+y,end = ",")
    print("")

if __name__ == '__main__':
    test_mutil_object_iter()

>>> python iterator.py
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTU1MTAwNDU0?x-oss-process=image/format,png)

* 类的迭代器与生成器混搭

```python
# iterator.py
class MixIterator:
    def __init__(self,start,stop):
        self.__start = start
        self.__stop = stop

    def __iter__(self):
        for index in range(self.__start,self.__stop+1):
            yield index**2      # 返回一个生成器对象，通过调用next()来显示值


def test_mix_iterator():
    iterator = MixIterator(1,5)            # 同一个对象可以迭代多次
    list1 = [index for index in iterator]  
    list2 = [index for index in iterator]
    print(list1)
    print(list2)

if __name__ == '__main__':
    test_mix_iterator()
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTU1MTMwMTQz?x-oss-process=image/format,png)

* 重构上述设计类的迭代器

```python
# iterator.py
# 使用yield生成器可以保
class MutilIterator2:
    def __init__(self,wrapper):
        self.__wrapper = wrapper

    def __iter__(self):
        print("call iter ... ")
        offset = 0
        while offset < len(self.__wrapper):
            item = self.__wrapper[offset]
            offset+=1
            yield item

def test_mutil2_iterator():
    S = "abcd"
    M = MutilIterator2(S)
    for x in M:       # 执行__iter__方法
        for y in M:    # 执行__iter__方法，进入方法的时候offset又变为初始化0
            print(x+y,end = ",")
        print()

if __name__ == '__main__':
    test_mutil2_iterator()
```
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE2MTU1MjI5MzQ0?x-oss-process=image/format,png)

> 迭代器小结

* 迭代器是一个很强大的工具,能够让我们创建自定义迭代器对象来完成自己的业务需求,比如支持大型数据库在多个游标中进行数据查询时获取迭代
* 对象单个迭代器对象,即不能作为类似于上述的操作,python有map、zip、函数生成器以及生成器表达式
* 对象多个迭代器对象,即可以来操作上述的动作，python有list、string、range
* 使用类的迭代器，我们可以自定义迭代器是单例还是多例

