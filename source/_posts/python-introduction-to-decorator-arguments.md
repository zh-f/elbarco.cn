---
title: Python中的装饰器——装饰器参数篇
date: 2017-09-20 16:52:14
tags: [Python, decorator]
---

## 先举个例子

在[上一篇文章](https://elbarco.cn/2017/09/20/python-introduction-to-decorators-with-examples/)中，我们提到的装饰器的例子有一个共同的特点，就是<!--more-->只接收被装饰的方法作为参数。但是在很多时候，装饰器本身接收更多的参数是非常有用的。但是如何做到呢？我们先回想一下基本的装饰器，它在内部声明一个方法，然后将这个内部方法返回，即被装饰器返回的callable。如果要使装饰器接收更多的参数，我们就要再包装一层——即接收参数的"装饰器"并不是真正的装饰器，而是一个返回装饰器的函数，而真正的装饰器负责接收被装饰的方法作为参数，然后装饰方法，返回一个callable。 还是用前面`json_output`的例子，简单改造一下：
```python
import functools
import json


class JSONOutputError(Exception):
    def __init__(self, message):
        self._message = message

    def __str__(self):
        return self._message


def json_output(indent=None, sort_keys=False):
    """Run the decorated function, serialize the result of that function
    to JSON, and return the JSON string.
    """

    def actual_decorator(decorated):
        @functools.wraps(decorated)
        def inner(*args, **kwargs):
            try:
                result = decorated(*args, **kwargs)
            except JSONOutputError as ex:
                result = {
                    'state': 'error',
                    'message': str(ex),
                }
            return json.dumps(result, indent=indent, sort_keys=sort_keys)

        return inner

    return actual_decorator
```

可以看到，在`json_output`中，传入了两个参数`indent`和`sort_keys`, 返回的是装饰器`actual_decorator`，而在`inner`中用到了传入的两个参数，用于JSON格式化时的缩进和Key的排序展示，来看这个装饰器的效果：
```python
@json_output(indent=4, sort_keys=True)
def do_nothing():
    return {'status': 'done','a': '1'}

# output 
{
    "a": "1", 
    "status": "done"
}
```

## 还有这种操作？

通过上面的例子，可以看到，装饰器是`actual_decorator`而不是`json_output`，那么问题来了，如果`json_output`不是装饰器而只是一个返回装饰器的函数，为什么可以像装饰器一样使用？

问题的关键在于操作的顺序。具体来说，`json_output(indent=4,sort_keys=True)`的调用在前，@操作符应用在后，那么这个函数的结果就会被当作装饰器使用。即先调用`json_output`，其中定义了装饰器`actual_decorator`，并且由`json_output`返回，则此时再应用@操作符就等价于：
```python
@actual_decorator
def do_nothing():
    return {'status': 'done','a': '1'}
```
这不就相当于在一个函数上应用装饰器嘛！

重要的一点是要意识到，当我们引入新的`json_output`函数时，实际上引入了一个后向不兼容的修改。

为什么这么说？因为现在这里有一个预期的额外函数调用。如果这里我们不想给`json_output`传递参数，那么我们依然要**调用**这个函数，即程序必须这么写：
```python
@json_output()
def do_nothing():
    return {'status': 'done','a': '1'}
```
>号外：一定要注意**圆括号**！因为这表示函数是被调用，然后函数结果被应用@。

上面的代码，不等同于，注意，是不等同于下面的写法：
```python
@json_output
def do_nothing():
    return {'status': 'done','a': '1'}

# print do_nothing() output
TypeError: actual_decorator() takes exactly 1 argument (0 given)
```
这里有两个问题：其一是比较让人疑惑，因为一旦习惯于见到没有括号的装饰器，见到类似`json_output`这种就会觉得反常；其二是如果旧的装饰器已经应用了其他很多地方，如果修改了这个装饰器类似上面的例子，那么其他应用的地方要一并修改，因为这是一个后向不兼容的更改。

理想情况下，我们希望装饰器在程序中对于下列三种应用方式都能兼容：
* @json_output
* @json_output()
* @json_output(indent=4, sort_keys=True)

实时证明，这是可行的，只需要让装饰器根据参数来改变其行为即可。下面，我们改写一下`json_output`：
```python
def json_output(decorated_=None, indent=None, sort_keys=False):
    """Run the decorated function, serialize the result of that function
    to JSON, and return the JSON string.
    """
    if decorated_ and (indent or sort_keys):
        raise RuntimeError('Unexpected arguments.')

    def actual_decorator(func):
        @functools.wraps(func)
        def inner(*args, **kwargs):
            try:
                result = func(*args, **kwargs)
            except JSONOutputError as ex:
                result = {
                    'state': 'error',
                    'message': str(ex),
                }
            return json.dumps(result, indent=indent, sort_keys=sort_keys)

        return inner
    if decorated_:
        return actual_decorator(decorated_)
    else:
        return actual_decorator
```
注意，我们为`json_output`增加了一个入参，`decorated_=None`，因为我们不希望同时传递被装饰的方法和关键字参数，所以在`json_output`中先做检查，来确保我们要么只传递被装饰的方法作为参数，要么只传递关键字参数。接着，`actual_decorator`是实际的装饰器。最终，如果设置了`decorated_`，即采用`@json_output`这种方式去修饰do_nothing方法（等价于`do_nothing = json_output(decorated_=do_nothing)`），则此时的`json_output`就是一个装饰器，那么我们应该返回的是`actual_decorator(decorated_)`；如果没有设置`decorated_`，则这就是一个方法，即可以通过`@json_output()`或`@json_output(indent=4)`的方式应用到`do_nothing()`上去——
```python
# 1
def do_nothing():
    return {'status': 'done','a': '1'}

do_nothing = json_output(decorated_=do_nothing)
print do_nothing()
# 1 - output
{"status": "done", "a": "1"}

# 2
@json_output
def do_nothing():
    return {'status': 'done','a': '1'}

print do_nothing()
# 2 - output
{"status": "done", "a": "1"}

# 3
@json_output()
def do_nothing():
    return {'status': 'done','a': '1'}

print do_nothing()
# 3 - output
{"status": "done", "a": "1"}

# 4
@json_output(indent=4, sort_keys=True)
def do_nothing():
    return {'status': 'done','a': '1'}

print do_nothing()
# 4 - output
{
    "a": "1", 
    "status": "done"
}

# 5
def do_nothing():
    return {'status': 'done','a': '1'}

do_nothing = json_output(decorated_=do_nothing, indent=4)
print do_nothing()
# 5 - output
Traceback (most recent call last):
  File "/decorators/output.py", line 77, in <module>
    do_nothing = json_output(decorated_=do_nothing, indent=4)
  File "/decorators/output.py", line 18, in json_output
    raise RuntimeError('Unexpected arguments.')
RuntimeError: Unexpected arguments.
```

1、2等价，json_output就是一个实实在在的装饰器；3、4中的json_output则是一个带参数的装饰器。

## 结语

本篇中，我们知道了如何给"装饰器"传递更多的参数，同时也学到了如何让"装饰器"更灵活，以适应不同的应用形式。










