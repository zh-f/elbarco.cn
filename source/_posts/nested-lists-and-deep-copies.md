---
title: 从Python中内嵌列表复制说起
date: 2017-03-29 10:00:31
tags: [Python, Deep Copy, Shallow Copy]
---

## 写在前面

在学习Python3时，看到了列表的拷贝，于是把这个小课题整理在这里，以作记录。
<!--more-->
## 内嵌列表和列表拷贝中的问题

Python中的列表是可以嵌入列表的，这个特性的应用场景之一便是用于表示二维矩阵。如下所示：
```python
>>> m = [[0, 1, 2], [10, 11, 12], [20, 21, 22]]
>>> m[0]
[0, 1, 2]
>>> m[0][1]
1
>>> m[2]
[20, 21, 22]
>>> m[2][2]
22
```
当然，这一特性可以用于按照我们自己的方式扩展到更多维的矩阵。

大部分情况下，内嵌列表如果只是这样使用，我们需要关心的也就到此为止了。但是因为有变量引用对象，而对象本身又是可被修改的情况，比如列表中内嵌列表，而内嵌列表本身是可被修改的，我们还会遇到下面提到的问题，我们通过例子来演示。


创建一个含有内嵌列表的列表`original`：
```python
>>> nested = [0]
>>> original = [nested, 1]
>>> original
[[0], 1]
```
列表`original`的第一个元素指向了列表`nested`，如图所示：
![](http://elbarco.eos.eayun.com/imgs/nested-list-01.png)

列表`nested`的修改可以通过直接修改其本身，也可以通过修改列表`original`来实现，即：
```python
>>> nested[0] = 'zero'
>>> original
[['zero'], 1]
>>> original[0][0] = 0
>>> nested
[0]
>>> original
[[0], 1]
```

如果我们将`nested`赋值为其他列表，则`nested`和`original`之间的连接就会断掉，即：
```python
>>> nested = [2]
>>> original
[[0], 1]
```
如图所示：
![](http://elbarco.eos.eayun.com/imgs/nested-list-02.png)

除了上面提到的直接赋值的方式，列表的拷贝我们还可以使用——
* 全切片(full slice)
```python
>>> x = [0, 1, 2]
>>> y = x[:]
>>> y
[0, 1, 2]
```
* `+`或`*`运算符
```python
>>> x = [0, 1, 2]
>>> y = x + []
>>> y
[0, 1, 2]
>>> z = x * 1
>>> z
[0, 1, 2]
```

但是无论上面哪种复制方式，只要列表中存在嵌入列表，就会存在这种问题，我们把这种复制称之为“浅拷贝”，即*shallow copy*，与之相对的，是“深拷贝”，即*deep copy*。

## 对列表的深拷贝

对于含有内嵌列表的列表来讲，如果我们需要把内嵌列表也一并拷贝，则需要使用`copy`模块的`deepcopy`功能。

```python
>>> original = [[0], 1]
>>> shallow = original[:]
>>> import copy
>>> deep = copy.deepcopy(original)
```
复制后两个列表的构成其实如下图所示：
![](http://elbarco.eos.eayun.com/imgs/nested-list-03.png)

在得到列表`shallow`和列表`deep`后，我们去尝试修改列表中的值和内嵌列表的值，并看看效果如何：
```python
>>> shallow[1]=2 #更改列表中非内嵌列表的值，原列表值不变
>>> shallow
[[0], 2]
>>> original
[[0], 1]
>>> shallow[0][0]='zero' #更改内嵌列表的值，原列表内嵌列表值改变
>>> shallow
[['zero'], 2]
>>> original
[['zero'], 1]
>>> 
>>> 
>>> deep[0][0]=5 #对于deep copy的列表，即使修改内嵌列表的值也不会影响原列表
>>> deep
[[5], 1]
>>> original
[['zero'], 1]
>>> 
```

此外，对于Python来讲，任何列表中嵌入的对象是可修改的，如字典，都会存在这样的问题。

## 总结和引申思考

首先，明确一点，*deep copy*和*shallow copy*并不是Python中特有的概念，而是一个与复制对象时对象的成员是否被复制有关的通用的概念。

根据维基百科中的[Object copying](https://en.wikipedia.org/wiki/Object_copying)中的描述，我们总结如下：

![](http://elbarco.eos.eayun.com/imgs/shallow-copy.png)

我们有变量A和变量B指向不同的内存地址，当B被赋值为A时，两个变量指向了同样的内存地址，之后无论是修改A还是B的内容，都会在另一个变量中立即体现出来，因为两者共享内容。

![](http://elbarco.eos.eayun.com/imgs/deep-copy.png)

我们有变量A和B指向了不同的内容地址，当B被赋值为A时，指向A内存地址的内容被复制到B的内存中，之后无论是修改A还是B的内容，A和B都是保持独立的，因为两者不共享内存，即不共享内容。

其他语言，如Java，可以参见StackOverFlow上的这一个讨论：[How do I copy an object in Java](http://stackoverflow.com/questions/869033/how-do-i-copy-an-object-in-java)，后面有机会再详细的梳理一下。



## 参考

1.[What's the difference between a deep copy and a shallow copy](http://stackoverflow.com/questions/184710/what-is-the-difference-between-a-deep-copy-and-a-shallow-copy)
2.[Object copying](https://en.wikipedia.org/wiki/Object_copying)
3.[Shallow and deep copy](http://www.python-course.eu/deep_copy.php)
4.[The Quick Python Book, 2nd Edition. Chapter 5.6](https://item.jd.com/19176803.html)







