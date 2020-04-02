---
title: python装饰器小结
category: Python
date: 2019-12-31 23:21:11
tags: Python
---

<!-- more -->

主要有以下几点：
* 装饰器定义与作用
* 函数装饰器
* 类装饰器
* 装饰器的嵌套
* 带参数的装饰器
* 装饰器是如何管理函数和类

##### 装饰器定义与作用

> 定义

* 在定义的函数或者类的方法或者类使用`@`的修饰符修饰,`@`后面可以是一个函数或者是一个类

> 作用

* 可以理解为对原函数对象或者类对象进行代理或者包装，增强原有的函数和类功能
* 能够管理函数对象和函数调用，也能够管理定义类对象的类本身和实例对象

##### 函数装饰器

函数装饰器：函数实际被调用的时候会直接返回一个由函数装饰器包装好的函数对象进行回调,可以修饰在函数或定义的类方法

> 编写基本的函数装饰器

```python
def decorator_fn(call_fn):
    if auth():
        u"""
        如果是授权成功则直接调用函数
        """
        return call_fn
    # 没有权限，不执行函数，返回一个包装None的可调用对象
    return callable(None)


@decorator_fn
def call_fn():
    print("call fn ...")
    
# 上述的call_fn对象等价于

call_fn = decorator_fn(call_fn)

# 之后再根据参数进行调用
call_fn()   # 此时该函数已增加授权校验功能
```

##### 类装饰器

类装饰器：类实际被调用的时候会直接返回一个由函数装饰器包装好的类进行回调,让该类具有某种属性或行为

```python
def decorator(aClass):
    print("intercept ....")
    return aClass


@decorator
class Person(object):
    pass

# 注意上述使用装饰器修饰的Person已经是调用装饰器函数并返回Person对象，
# 即定义类的时候已经拥有装饰器的功能，因此不论如何调用Person()创建实例，
# 上面仅会打印一次intercept

Person()分两步：
其一，Person = decorator(Person) ，# 执行包装的intercept然后返回原Person类
其二，利用装饰器返回的Person类再创建对象
```

##### 嵌套的装饰器

> 函数装饰器嵌套

```python
@fn1
@fn2
@fn3
def fn(*args, **kwargs):
    pass
    
# fn对象等价于

fn = fn1(fn2(fn3(fn)))
```

> 类装饰器嵌套

```python
@fn1
@fn2
@fn3
class Person(object):
    pass
    
# 上述定义的类Person已经是具有fn1,fn2,fn3的所有装饰器功能
Person = fn1(fn2(fn3(Person)))
```

##### `带参数`的装饰器

> 修饰带有参数的函数的装饰器,这时候装饰器的作用就是返回一个函数的代理

```python
def decorator(fn):
    def proxy(*args, **kwargs):
        u"""
        作为代理函数来调用原有的函数，并对原来的函数进行auth的校验
        :param args:
        :param kwargs:
        :return:
        """
        print("auth checking")
        return fn(*args, **kwargs)
    return proxy


@decorator
def fn(a=9, b=10):
    print(a+b)
    
# 上述的被装饰器修饰的fn等价于proxy，在函数proxy中已经保存了fn函数的对象
因此调用fn()就等价于调用proxy(),而proxy()函数增加了auth认证校验，即
fn(*args, **kwargs) =  decorator(fn)(*args, **kwargs),再进一步拆分，即
fn = decorator(fn)
```

> 修饰带有参数初始化类的装饰器

* 定义一个装饰器函数并传递类对象
* 在定义的装饰器函数内部定义一个代理函数对象，此代理函数对象与原函数传递的参数一致，并负责处理日志，认证等工作，最后返回一个类对象
* 此时使用装饰器修饰类必须重载运算符`__call__`保证可被动态调用（类似与反射创建类对象）

```python
def decorator(aClass):
    def proxy(*args, **kwargs):
        u"""
        作为代理函数来调用原有的函数，并对原来的函数进行auth的校验
        :param args:
        :param kwargs:
        :return:
        """
        print("auth checking")
        return aClass(*args, **kwargs)
    return proxy


@decorator
class Person(object):

    def __call__(self, *args, **kwargs):
        print("person ...")

# 注意体会与上述定义类装饰器的区别,这个时候定义的Person并非Person类，
而是一个定义在decorator函数内部的一个代理函数，这个代理函数保存了aClass这个类对象
Person(*args, **kwargs) = decorator(Person)(*args, **kwargs)，进一步拆分，即
Person = decorator(Person)
```

> 定义的装饰器函数传递参数

```python
def decorator(cache_time, strategy):
    def actual_decorator(fn):
        print("using cache_time[%s] and strategy[%s] do the first job" % (cache_time, strategy))
        def proxy(*args, **kwargs):
            print("before call must be do the second job..")
            fn(*args, **kwargs)
        return proxy
    return actual_decorator


@decorator(cache_time=10, strategy="")
def call_fn(a, b=10):
    if a < b:
        print("a < b")
    elif a > b:
        print("a > b")
    else:
        print("a = b")

# 使用上述的装饰器修饰的call_fn等价于
# call_fn = decorator(cache_time=10, strategy="")(call_fn) # 调用了两次函数,最终返回代理proxy

class Person(object):

    @decorator(cache_time=10, strategy="nio")
    def cal(self,a, b=10):
        if a < b:
            print("a < b")
        elif a > b:
            print("a > b")
        else:
            print("a = b")
            
# 同理，定义的cal函数等价于
cal = decorator(cache_time=10, strategy="nio")(cal)   # 调用了两次函数并返回代理函数
```

##### 利用装饰器有效地管理函数和类

使用装饰器要分清楚以下几点：

* 使用装饰器修饰的函数或者类主要应用场景是什么,是直接返回原函数(类)还是嵌套定义的代理函数对象
* 如果是直接返回园函数或者是类，那么可以保证修饰前后的数据属性是一致的，并且能够获取原数据的属性信息
* 如果返回的是一个包装原函数或者类的代理函数对象，那么此时数据属性便发生改变，这种情况下只适用于调用的方式而非使用元数据进行下一步业务操作


