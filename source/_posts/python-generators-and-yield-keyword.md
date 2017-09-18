---
title: Python中的生成器和yield关键字
date: 2017-09-18 16:49:46
tags: [Python, yield]
---
## 前言

我们都知道`yield`语句用于定义生成器，替代函数的`return`语句来向其调用者提供结果，并且不破坏局部变量。<!--more-->与函数不同的是，每次调用时，生成器会以新的变量集开始，继续执行它被关闭的执行。

## 关于Python生成器

Python中的生成器的目的是能够即时的按照我们的要求逐个计算一系列结果。举个最简单的例子，生成器可以用作列表，列表中的每个元素会在用到的时候的方式被计算（lazily）：
```python
>>> # 定义列表
>>> the_list = [2**i for i in range(5)]
>>> # 类型检查，确实是一个列表
>>> type(the_list)
<type 'list'>
>>> for j in the_list:
...     print j
... 
1
2
4
8
16
>>> # 列表长度为5
>>> len(the_list)
5
>>> # 定义一个生成器，注意是'()'而不是'[]'
>>> the_gen = (x+x for x in range(5))
>>> # 类型检查，确实是一个生成器
>>> type(the_gen)
<type 'generator'>
>>> # 遍历生成器中的元素，并打印
>>> for j in the_gen:
...     print j
... 
0
2
4
6
8
>>> # 看起来好像跟列表似的，那如果我们来检查一下长度……
>>> len(the_gen)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: object of type 'generator' has no len()
>>> 

```
从上面的例子，可以看出，遍历列表和遍历生成器是一样的。不过，尽管生成器可遍历的，但是却不是一个数据集合，因此没有长度的属性。数据集合（比如列表、元组、集合等）将数据存储在内存中，所以我们需要时就可以获得；生成器即时的计算结果，然后下一次迭代时就把上一次结果“忘掉了”，所以生成器没有对自己结果集的任何概述。

正因为生成器有这个特性——不需要同时在内存中保留数据集合中的全部元素——所以非常适合内存敏感的任务。当我们不需要完整的结果时，逐个的计算结果值的做法就显得十分有用，对调用者即时的返回中间结果，直到满足一些要求然后停止处理。

## 使用Python的`yield`关键字

这里我们有一个很好的例子，就是当我们在搜索时，我们不需要等所有的结果都被查找出来。比如在文件系统中搜索时，用户更希望能即时的看到结果，而不是等搜索工具遍历每个文件，然后返回所有的结果。再比如，用Google搜索的用户会一直翻到最后一页吗？

这里我们就可以使用`yield`关键字/语句来定义一个生成器。`yield`指令应当放在生成器立即返回结果给调用者并且等待下次调用发生的地方。举个例子，我们先定义一个用于在大文件中逐行搜索关键字的生成器：
```python
def search(keyword, filename):
    print 'Generator started'
    f = open(filename,'r')
    for line in f:
        if keyword in line:
            yield line
    f.close()

# 在data.txt中搜索yield关键字
the_gen = search('yield', 'data.txt')
# 检查the_gen的类型
print type(the_gen)
# 也可以用the_gen.next()或next(the_gen)遍历
for i in the_gen:
    print i

```
最终，我们得到的the_gen的类型是`<type 'generator'>`，遍历the_gen得到`data.txt`中包含`yield`关键字的每一行，输出结果为：
```
<type 'generator'>
Generator started
Using the Python "yield" keyword

The yield instruction should be put into a place... 

Since the yield keyword is only used with generators...
```
## 更多的例子

生成器的应用有很多，比如扮演传送带的角色，一个比较好的例子即缓冲区：获取大量的数据并将其以小数据块进行处理：
```python
def buffered_read():
    while True:
        buffer = fetch_big_chunk()
        for small_chunk in buffer:
            yield small_chunk
```

最后我们再看一个经典例子——给定数字N，使用生成器给出前N个斐波那契序列（Fibonacci）数字：
```python
def fibonacci(n):
    curr = 1
    prev = 0
    counter = 0
    while counter < n:
        yield curr
        prev, curr = curr, prev + curr
        counter += 1

f = fibonacci(6)
    for i in f:
        print i
1
1
2
3
5
8
```
直到`counter = n`，停止while循环。

## 参考

[1].[Python generators and the yield keyword](http://pythoncentral.io/python-generators-and-yield-keyword/)
[2].[What does the “yield” keyword do?](https://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do)
