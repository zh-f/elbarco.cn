---
title: 简述StringBuilder和StringBuffer的区别
date: 2016-09-05 14:36:37
tags: [Java]
---

### 从StringBuilder和StringBuffer的不同说起

最近在搬砖的时候，发现在拼接字符串的时候，有人习惯使用StringBuffer，有人习惯使用StringBuilder，于是想到了之前在知乎上看到的这个讨论：[国内Java面试总是问StringBuffer，StringBuilder区别是啥？档次为什么这么低](https://www.zhihu.com/question/50211894)，果然这在面试中只是一道预热筛选题嘛<!-- more -->，不过一下子让我答，却并不能立刻回答上来区别，于是顺手Google了一下，在[StringBuilder](http://docs.oracle.com/javase/7/docs/api/java/lang/StringBuilder.html)和[StringBuffer](http://docs.oracle.com/javase/7/docs/api/java/lang/StringBuffer.html)的API(JDK1.7)里找到了答案。下面就做一下简述——

首先，`StringBuffer`和`StringBuilder`都是可变字符串，但是前者是线程安全的，因为在调用StringBuffer的操作时是同步的，在源代码中看到的就是方法上加了`synchronized`关键字：
```java
...
public synchronized StringBuffer append(String str) {
        super.append(str);
        return this;
}
...
```

而在StringBuilder的源码中，我们看到的是这样的：
```java
public StringBuilder append(String str) {
        super.append(str);
        return this;
    }
```

上面仅截取部分代码，更多的代码大家可自行查看。

在`StringBuffer`的API说明中，提到，在JDK5中，开始提供了功能相同，但是非线程安全、不使用`synchronized`、性能更好的类`StringBuilder`，在`StringBuilder`API说明中，有提到这么一句话：

>Instances of StringBuilder are not safe for use by multiple threads. If such synchronization is required then it is recommended that StringBuffer be used.

即只有在同步是必要的情况下，才建议使用`StringBuffer`。


### 再论拼接字符串的不同方法和效率

至此，区别就简述完了。什么，这就完了？摔……按照面试套路，理论上应该是进入下一话题了，不过这里我们还是要继续，现在就抛出一个非常基础常见的套路问题——

>问：常见的拼接字符串的方法有哪些？

答案是：String的`concat`方法、`+`操作符；`StringBuffer`和`StringBuilder`的`append`方法。

>再问：上面几种方法效率如何？

答案也很简单，当然是`StringBuilder>StringBuffer>concat或+操作符`。

回答完是什么之后，我们再问问为什么。首先，StringBuffer的每个append操作都是同步的，所以比StringBuilder要慢，那么为什么都比`concat`或者`+`效率搞呢？于是又Google一下，找到了这个[讨论](http://stackoverflow.com/questions/14927630/java-string-concat-vs-stringbuilder-optimised-so-what-should-i-do)（Google大法好！Stackoverflow大法好！Orz..），里面提到，在JDK1.6之后，使用"+"操作符时，编译器会自动使用StringBuilder将两个字符append到一起，比如我们代码里是这样写的：
```java
String one = "abc";
String two = "xyz";
String three = one + two;
```
在编译的时候，`String three`会被编译成：
```java
String three = new StringBuilder().append(one).append(two).toString();
```
乍一看，是效率了很多，但是如果在循环中这样干：
```java
String out = "";
for( int i = 0; i < 10000 ; i++ ) {
    out = out + i;
}
return out;
```
那么在编译时，可能得到的内容就是这样子的：
```java
String out = "";
for( int i = 0; i < 10000; i++ ) {
    out = new StringBuilder().append(out).append(i).toString();
}
return out;
```
此时，我们其实都知道应该这样写：
```java
StringBuilder out = new StringBuilder();
for( int i = 0 ; i < 10000; i++ ) {
    out.append(i);
}
return out.toString();
```
这也反映了，编译器一定程度上可以帮助我们优化，但是写出高效的代码，还需要我们自己。

### 另一个角度较真儿的验证

上面的代码是13年答主在JDK1.6中测试的结果，又有一位较真儿的朋友，在不同的JDK版本中进行了测试，全文见[Java StringBuilder myth debunked](https://www.javacodegeeks.com/2013/03/java-stringbuilder-myth-debunked.html)，最终得到了下面的图表：

* 使用`+`操作符

![](http://bop-to.top/catplus.png)

* 使用`StringBuilder`

![](http://bop-to.top/catsb.png)

* 使用`StringBuilder`的基准

![](http://bop-to.top/catsb2.png)

这位童鞋贴心的把测试用的代码托管在[Github](https://github.com/skuro/stringbuilder)上，有兴趣的可以去看一下。最终这篇文章得出的结论就是——通过对字节码的分析，我们得到了答案，显而易见的是，使用`StringBuilder`是可以提高性能的。文章开篇还提到这么一句话——

>Concatenating two Strings with the plus operator is the source of all evil — Anonymous Java dev

与大家共勉。










