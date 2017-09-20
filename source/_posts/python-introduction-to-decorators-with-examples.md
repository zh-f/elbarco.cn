---
title: Python中的装饰器——初识篇
date: 2017-09-20 14:53:15
tags: [Python, decorator]
---

## 认识装饰器 

装饰器简单来讲就是一个接收被装饰的函数作为固定参数的函数<!--more-->（或称为callable，比如一个对象具有__call__方法），并且对被装饰的函数做些处理。举个例子：
```python
def decorated_by(func):
    func.__doc__ += '\nDecorated by decorated_by.'
    return func


def add(x, y):
    """Return the sum of x and y."""
    return x + y


if __name__ == '__main__':
    add = decorated_by(add)
    print add.__doc__

# output 
Return the sum of x and y.
Decorated by decorated_by.

```
大多数情况下，我们只关心最终被装饰的函数，而持有装饰器函数的引用基本上是没必要的。所以，在Python 2.5中引入了对装饰器的特殊语法——装饰器的应用可以通过在被装饰的函数上面一行添加@字符，其后紧跟装饰器函数名的这种方式：比如针对上面的例子，我们就可以写作：
```python
@decorated_by
def add(x, y):
    """Return the sum of x and y."""
    return x + y


if __name__ == '__main__':
    # add = decorated_by(add)
    add(1, 2)
    print add.__doc__

# output
Return the sum of x and y.
Decorated by decorated_by.
```
添加@+装饰器名的这种方式就等价于 add = decorated_by(add)。这种方式看起来更简洁明了。

## 装饰器应用的顺序

当@语法被使用时，装饰器会在被装饰的callable被创建后立即调用（即装饰器的代码是在应用到被装饰的函数上时执行，而不是被装饰的函数调用时执行）。就上面的例子而言，就是add函数被创建后，紧接着decorated_by函数被应用。那么如果对于单个callable使用@语法应用多个装饰器时（Python中是支持这种场景的），装饰器的应用顺序有事怎样的？答案就是：从下往上，按顺序执行。举个例子：
我们有另外的一个函数叫做also_decorated_by，也是在func.__doc__后面添加一段话，然后对add函数应用该装饰器：
```python
def also_decorated_by(func):
    func.__doc__ += '\nAlso decorated by also_decorated_by.'
    return func


@also_decorated_by
@decorated_by
def add(x, y):
    """Return the sum of x and y."""
    return x + y

```
按照从下往上的顺序，我们知道当我们调用add后，执行decorated_by相当于add = decorated_by(add) ，然后对此时的add应用also_decorated_by，就相当于add = also_decorated_by(decorated_by(add))。最终的结果正是：
```python
if __name__ == '__main__':
    # add = decorated_by(add)
    add(1, 2)
    print add.__doc__

Return the sum of x and y.
Decorated by decorated_by.
Also decorated by also_decorated_by.
```
## 装饰器的应用场景

标准库中的很多模块都包含有装饰器，许多常见的工具和框架都将其用于常见的功能。例如，如果要在类上创建一个方法，调用该方法时不需要该类的实例，则可以使用@classmethod或@staticmethod装饰器，这是标准库中的一个简单例子。
常用的工具中也是用装饰器，比如常见的Python Web框架Django中，使用@login_required作为装饰器允许开发者指定用户在访问特定页面时必须要登录。另外一个Web框架Flask中使用@app.route来注册指定的URI被访问到时要执行的函数。再比如，Celery中使用一个复杂的@task装饰器来标识一个函数为一个异步任务，这个装饰器的返回实际上是一个Task类的实例，展示出了如何用装饰器来制作方便快捷的API。

## 为什么要使用装饰器

 有了装饰器，你就可以做到在某些特定的地方使用具体的、可复用的功能——如果代码写得好，装饰器就是模块化的、明确的。正是由于装饰器的模块化，使得它们非常适合避免重复的模版设置和拆解代码，同时由于装饰器仅与被装饰的函数本身有交互，所以非常擅长在其他地方注册功能。
在Python应用程序和模块中编写装饰器有几个非常好的用例——
* 附加功能 - 在被装饰的方法前后执行额外的附加功能
* 数据清洗或添加 - 对传入被装饰的函数的参数做一下清晰，确保参数类型一致性或者使参数值村从一定的模式，比如@requires_ints
* 功能注册
 
## 动手写几个装饰器

纸上得来终觉浅，绝知此事要躬行。写下来就要动手写几个装饰器的例子。

### 功能注册

先上代码打个样：

```python
class Registry(object):
    def __init__(self):
        self._functions = []

    def register(self, decorated):
        self._functions.append(decorated)
        return decorated

    def run_all(self, *args, **kwargs):
        return_values = []
        for func in self._functions:
            return_values.append(func(*args, **kwargs))
        return return_values


a = Registry()
b = Registry()


@a.register
def foo(x=3):
    return x


@b.register
def bar(x=5):
    return x


@a.register
@b.register
def baz(x=7):
    return x

a_r = a.run_all() 	# [3, 7]
b_r = b.run_all() 	# [5, 7]
a_r = a.run_all(x=4) 	# [4, 4]

```

### 运行时封装代码（wrap code）

上面装饰器的例子比较简单，因为被装饰的函数作为参数传递过去时并没有被修改。然而，有些时候当我们运行被装饰的方法时，希望做些额外的功能，那么我们可以通过返回不同的可调用函数（callable）来添加相应的功能，并且（通常）在执行过程中调用装饰方法。举几个例子：

#### 类型检查

```python
def requires_int(decorated):
    def inner(*args, **kwargs):
        """ Get any values that may have been sent as keyword arguments.
        """
        kwarg_values = [i for i in kwargs.values()]
        for arg in list(args) + kwarg_values:
            if not isinstance(arg, int):
                raise TypeError('%s only accepts integers as arguments.' %
                                decorated.__name__)
        return decorated(*args, **kwargs)

    return inner

@requires_int
def foo(x,y):
    """ Return the sum of x and y"""
    return x+y

if __name__ == '__main__':
    print foo(1,y='a')

# output 
Traceback (most recent call last):
  File "F:/Test/PyTest/decorators/typecheck.py", line 17, in <module>
    print foo(1,y='a')
  File "F:/Test/PyTest/decorators/typecheck.py", line 7, in inner
    decorated.__name__)
TypeError: foo only accepts integers as arguments.

```
上面的例子中，我们使用装饰器@requires_int对foo的参数进行检查，如果入参中有任何不是int类型的参数，就抛出一个异常。

如果此时我们执行help(foo)会发现，得到的结果是：
```python
Help on function inner in module __main__:

inner(*args, **kwargs)
    Get any values that may have been sent as keyword arguments.
```
很奇怪吧，为什么显示的是inner的函数名及doc而不是foo呢？因为此时inner函数已经被赋值给了foo，如果我们执行foo，其实就是执行inner——首先执行了类型检查，然后运行被装饰的方法，因为inner通过return decorated(*args, **kwargs)调用了被装饰的函数。如果没有return这个调用，被装饰的方法就会被忽略。

#### 保留原函数的帮助信息

经过上面的例子，我们会自然而然的思考一问题，如果当我们运行help()去查看函数的帮助信息时，希望看到的是被装饰的原函数的文档，而不是装饰器的文档，该怎么做呢？这里，我们的解决方案就是使用装饰器@functools.wraps ，它可以将一个函数重要的内部元素拷贝到另一个函数中：
```python
import functools

def requires_int(decorated):
    @functools.wraps(decorated)
    def inner(*args, **kwargs):
        """ Get any values that may have been sent as keyword arguments.
        """
        kwarg_values = [i for i in kwargs.values()]
...

```
那么此时查看help(foo)，我们得到的输出结果是：
```python
Help on function foo in module __main__:

foo(*args, **kwargs)
    Return the sum of x and y
```
> 注：我这里运行的是Python2.7，如果实在Python3下，看到的结果应该是`foo(x, y)`。

#### 用户校验

```python
import functools


class User(object):
    def __init__(self, username, email):
        self.username = username
        self.email = email


class AnonymousUser(User):
    def __init__(self):
        self.username = None
        self.email = None

    def __nonzero__(self):
        return False


def require_user(func):
    @functools.wraps(func)
    def inner(user, *args, **kwargs):
        if user and isinstance(user, User):
            return func(user, *args, **kwargs)
        else:
            raise ValueError('A valid user is required to run this.')
    return inner

@require_user
def foo(user):
    if user.username and user.email:
        print user.username + ',' + user.email
    else:
        print 'None'

if __name__ == '__main__':
    user = User('Tom','tom.smith@tom.com')
    a_user = AnonymousUser()
    foo(user)
    foo(a_user)

# output
Tom,tom.smith@tom.com
Traceback (most recent call last):
  File "F:/Test/PyTest/decorators/user.py", line 39, in <module>
    foo(a_user)
  File "F:/Test/PyTest/decorators/user.py", line 25, in inner
    raise ValueError('A valid user is required to run this.')
ValueError: A valid user is required to run this.
```
我们定义了一个User类，一个AnonymousUser类集成User类，注意，在AnonymousUser中，我们使用了__nonzero__方法，这个方法定义了对类的实例调用bool()时的行为，即在require_user的inner方法中，if user and isinstance(user, User)时的if user，相当于if bool(user)，因为在AnonymousUser中我们定义了__nonzero__返回False，所以这里就没有办法通过if检查，从而rasie了异常。

#### 格式化输出

除了为函数的输入清理数据，装饰器的另一个作用就是清理/格式化输出数据。比如我们希望函数的输出以JSON格式，在每个相关的函数的最后手动去格式化输出为JSON显得特别冗余和笨重。此时，装饰器就站出来了。

```python
import functools
import json


class JSONOutputError(Exception):
    def __init__(self, message):
        self._message = message

    def __str__(self):
        return self._message


def json_output(decorated):
    """Run the decorated function, serialize the result of that function
    to JSON, and return the JSON string.
    """
    @functools.wraps(decorated)
    def inner(*args, **kwargs):
        try:
            result = decorated(*args, **kwargs)
        except JSONOutputError as ex:
            result = {
                'status': 'error',
                'message': str(ex),
            }
        return json.dumps(result)
    return inner


@json_output
def do_nothing():
    return {'status': 'done'}


@json_output
def error():
    raise JSONOutputError('This function is erratic')


@json_output
def other_error():
    raise ValueError('The grass is always greener..')

if __name__ == '__main__':
    # print do_nothing()
    print error()

# output
{"status": "error", "message": "This function is erratic"}
```

#### 打印日志

最后一个例子，打印日志，记录调用了哪个方法、什么时间调用、方法执行时长以及方法的返回值：
```python
import functools
import logging
import time

logging.basicConfig(evel=logging.DEBUG)
LOG = logging.getLogger(__name__)


def logged(method):
    @functools.wraps(method)
    def inner(*args, **kwargs):
        start = time.time()
        return_value = method(*args, **kwargs)
        end = time.time()
        delta = end - start
        LOG.warn('Called method %s at %.02f; execution time %.02f '
                 'seconds; result %r.' %
                 (method.__name__, start, delta, return_value))
        return return_value

    return inner


@logged
def sleep_and_return(return_value):
    time.sleep(2)
    return return_value


if __name__ == '__main__':
    print sleep_and_return(27)

# output 
WARNING:__main__:Called method sleep_and_return at 1505889774.61; execution time 2.00 seconds; result 27.
27

```

## 小节结语

本篇文章的重点是简单的认识一下装饰器，了解一下装饰器的简单应用。

目前上面的例子，装饰器除了被装饰的函数作为参数之外，都不接收其他的参数，但是很多情况下，装饰器本身接收其他参数是很有必要的。我们后面再展开。
