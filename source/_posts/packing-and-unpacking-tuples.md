---
title: Python中的元组和Packing/Unpacking
date: 2017-03-30 11:02:31
tags: [Python, Tuple]
---

## 什么是元组

元组（Tuple）是与列表类似的数据结构，只可被创建，不可被修改，用一对圆括号`()`包起来，如：
<!--more-->
```python
>>> x = ('a','b','c') #具有三个元素的元组
>>> y = ('a',) #只有一个元素的元组，注意，必须有一个逗号来标识
>>> z = () #空元组
```
元组的操作跟列表非常类似，如`+`，`*`，切片等，在元组中同样适用：
```python
>>> a = (1,2,3)
>>> a[:2]
(1, 2)
>>> a * 1
(1, 2, 3)
>>> a + (4,5)
(1, 2, 3, 4, 5)
```

## 元组的Packing/Unpacking

Python中允许元组出现在赋值运算符的左侧，这样元组中的每个变量就可以被赋值为右侧对应位置的值，如：
```python
>>> (a,b,c,d) = (1,2,3,4)
>>> a
1
>>> c
3
```
上面的写法还可以简化为:
```python
>>> a,b,c,d = 1,2,3,4
```
这个用法还可以非常方便的完成交换两个变量的值：
```python
>>> var1, var2 = var2, var1
```

在Python3中，还提供了一个扩展的unpacking特性——使用`*`来标注元素来吸收与其他元素不匹配的任何数量的元素，举例如下：
```python
>>> x = (1,2,3,4)
>>> a, b, *c = x
>>> a, b, c
(1, 2, [3, 4])
>>> a, *b, c = x
>>> a, b, c
(1, [2, 3], 4)
>>> *a, b, c = x
>>> a, b, c
([1, 2], 3, 4)
>>> a, b, c, d, *e = x
>>> a, b, c, d, e
(1, 2, 3, 4, [])
```
被标星的元素接收多余的元素作为一个列表，如果没有多余的元素，则会接收一个空列表。

## Python中的Packing/Unpacking应用

Python中，我们可以使用`*`（对元组来说）和`**`（对字典来说）的Packing和Unpacking函数的参数。

### * for tuples
从下面这个例子说起，我们有一个函数`fun()`接收四个函数，并打印出来：
```python
def fun(a, b, c, d):
    print(a, b, c, d)
```

假设我们有一个list：
```
>>> my_list = [1, 2, 3, 4]
```
调用函数`fun()`：
```python
>>> my_list = [1,2,3,4]
>>> fun(my_list)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: fun() missing 3 required positional arguments: 'b', 'c', and 'd'
>>> 
```
会得到报错信息，函数认为我们的`my_list`是单独的一个参数，而函数还需要额外的三个参数。

这时候，我们可以使用`*`来<strong>解包（Unpacking）</strong>列表，使之作为四个参数：
```python
>>> fun(*my_list)
1 2 3 4
```
这里还有另一个例子，使用内置的`range()`函数，来演示解包列表的操作：
>注，这里使用的是Python3.x，range(3,7)不会直接打印区间的所有值

```python
>>> list(range(3,7))
[3, 4, 5, 6]
>>> args = [3,7]
>>> list(range(args))
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'list' object cannot be interpreted as an integer
>>> list(range(*args))
[3, 4, 5, 6]
```
当我们不知道究竟要传递多少参数给函数时，我们可以使用<strong>打包（Packing）</strong>把所有的参数打包到一个元组中，用下面的例子来演示打包操作。

我们有函数`iSum()`来做求和操作
```python
def iSum(*args):
    sum=0
    print(args) #把args打印出来，看是否被打包为一个元组
    for i in range(0, len(args)):
        sum = sum + args[i]
    return sum

```
测试时，传参个数不等，得到的输出如下：
```python
>>> print(iSum(1,2,3,4,5))
(1, 2, 3, 4, 5)
15
>>> print(iSum(10,20,30))
(10, 20, 30)
60
```

可以看到入参确实被打包成了一个元组，然后循环遍历元组求和。

如果打包后我们想修改参数，由于元组不可修改，所以需要先转换成列表。下面展示一个打包和解包混合使用的例子。
我们有函数`func1()`用于打印入参，`func2()`用户修改入参的值：
```python
>>> def func1(a,b,c):
...     print(a,b,c)
... 
>>> def func2(*args):
...     args = list(args)
...     args[0] = 'elbarco.cn'
...     args[1] = 'awesome'
...     func1(*args)
...
```
调用`func2()`，传递的三个参数，首先打包为一个元组，然后将元组转换为列表，并修改前两个元素的值，再解包为三个参数，打印出结果，如下所示：
```python
>>> func2('Hello','nice','visitors')
elbarco.cn awesome visitors
```
### ** for dictionaries

对于字典（Dictionary），Packing/Unpacking操作使用`**`。

还是使用上面的`func1()`，如果要打印字典的值，则需要使用`**`来解包：
```
>>> dict = {'a':1,'b':3,'c':5}
>>> func1(**dict)
1 3 5
```

下面来一个打包的例子：
```
>>> def func3(**elbarco):
...     print(type(elbarco))
...     for key in elbarco:
...         print("%s = %s" % (key, elbarco[key]))
... 
```
传几个参数，用`func3()`将入参打包为字典，然后在函数中把key和value输出出来，结果如下所示：
```python
>>> func3(name='elbarco', location='Beijing', language='Java/Python')
<class 'dict'>
name = elbarco
location = Beijing
language = Java/Python
```

以上。

## 参考

1.[The Quick Python Book 2nd Edition.Chaptor 5.7.3]()
2.[Packing and Unpacking Arguments in Python](http://www.geeksforgeeks.org/packing-and-unpacking-arguments-in-python/)
3.[Packing and Unpacking Arguments in Python](https://hangar.runway7.net/python/packing-unpacking-arguments)
4.[Python’s range() Function Explained](http://pythoncentral.io/pythons-range-function-explained/)
