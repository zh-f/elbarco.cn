---
title: Python中的装饰器——装饰器参数篇
date: 2017-09-20 16:52:14
tags: [Python, decorator]
---

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

可以看到，在`json_output`中，传入了两个参数`indent`和`sort_keys`, //todo 


