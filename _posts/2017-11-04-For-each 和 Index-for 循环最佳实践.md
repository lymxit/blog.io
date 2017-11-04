---
layout: post
title: For-each 和 Index-for 循环最佳实践
date: 2017-11-04
categories: blog
tags: [编程]
description: 无
---

# For-each 和 Index-for 循环最佳实践

> ~~for-each 循环优先于传统的for循环~~  -- Joshua Bloch
>
> a hand-written counted loop is better than for the enhanced loop. -- jackiemliu

首先，相信大家对于这两种循环都很熟悉：

```java
// foreach
for (Box box : boxList)
	box.id++;

// index for（记住保存len，否则每次都需要调用boxList.size()）
for (int index = 0,len = boxList.size(); index < len; index++)
	boxList.get(index).id++;
```

但两者何时使用，性能和效率怎么权衡，又成为了新的问题！

因此这篇文章就主要是针对这两种循环方式和 Android 平台上的取舍做一些简单的分析。

### 最佳实践

先给出最佳实践，原因后面进行分析：

1、优先使用 index-for 模式（Android Framework 推荐）;

2、如果想在遍历过程中暴露出其它线程正在修改（*ConcurrentModificationException*）的问题，请使用 for-each 模式.

### 深入分析

#### Iterate 模式

for each这种写法其实是一个语法糖，其实

`for(Box box:boxList)` 等同于 `for(Iterator var1 = boxList.iterator(); var1.hasNext();var1.next())`

可以看到这种模式会去额外生成一个 Iterator 对象，所以相较于 Index 模式而言，它会额外使用一些内存。

> 在 Android 平台，内存资源是极为有限的，如果只是单层的循环还算 OK，但是如果是多层循环，
>
> 或者是隐式的多层循环中使用 Iterate 模式，可想而知内存会临时分配很多个变量。

例如：

```java
// 显式多层循环
for (Box box : boxList)
  for (InnerBox innerBox : box)
    ; // do something

// 隐式的多层循环（View 的 onDraw 方法）
void onDraw(Canvas c){
  for (Box box : boxList)
    ; // do something
}
```

以上这两种情况可能来说就会分配过多的临时对象，导致内存不足进行 GC ，从而影响 App 流畅度。

除此之外，在迭代的过程中，会去调用 Iterator 的  `next()`，这里我以 *ArrayList* 为例：

```java
public E next() {
    checkForComodification();  // 检查是否在遍历过程中有人修改了列表
     int i = cursor;
     if (i >= size)    // 检查下标是否合法
         throw new NoSuchElementException();
     Object[] elementData = ArrayList.this.elementData;
     if (i >= elementData.length) // 检查下标是否合法
         throw new ConcurrentModificationException();
     cursor = i + 1;
     return (E) elementData[lastRet = i];  // 返回对象
}
```

相比与Index 模式，它增加了很多检查，所以会带来一定的开销，但也是一种特性（检查遍历中是否有人修改）。

#### Index 模式

大家最常见的可能是：

```java
for(int index = 0; index < boxList.size();index++)
	; // do something
```

但这种方式其实并不够理想，因为每次循环时，都会去调用 List.size()。

所以我们可以将其保存下来：

```java
for(int index = 0,len = boxList.size(); index < len;index++)
	; // do something
```

相对而言，保存 len 的方法更快，尤其在 Effective Java 中的范例：

```java
for(int index = 0,n = expensiveComputation(); index < n;index++)
	; // do something
```

如果一个方法耗时较多且结果不会改变，那么可以用一个临时变量充当缓存。

#### 性能比较

##### 速度

一开始我还自己在电脑上做了个小 demo，然而发现 Android Team 已经做了一个更具代表性的（多次取平均值），这里直接借用吧：

![mark](http://oicc5e0b7.bkt.clouddn.com/blog/20171104/220244603.png)

这里的案例是 400,000 随机的 Integer，每种模式都跑 10 次，去掉最大最小值后取平均结果。

可以很明显看出 Index 模式的耗时完美胜出，Yeah |ω・）

##### 内存占用

由于之前介绍了 Iterate 模式会初始化一个 Iterator 对象，所以它的内存占用肯定多于 Index 模式。

### 总结

0、语法糖可能很甜，但也可能有隐藏的性能损耗（e.g lambda 表达式会增加运行时开销）

1、从语法上而言，foreach 这种更简洁，也更加地隐藏了细节（语法糖），但也缺失了某些特性。

简单来说，尽可能将 foreach 看成是一种 **只读向后遍历**。

在 Effective Java 中，介绍了无法使用的三种场景：

- 过滤——如果需要在遍历过程中删除元素。如果在 foreach 中删除，会引发并发修改的异常。
- 转换——如果需要在遍历过程中对元素进行替换。如果在 foreach 中替换，例如 o = new Object(); 其实并没有改变列表中的值。
- 平行迭代——如果需要灵活更改遍历的顺序时。

2、开发 Android 的时候，尽可能放弃使用 foreach ，减少内存压力。

3、如果嫌弃 Index 模式的模板代码太麻烦，可以试试 Live Templates 中自带的 itli ，一键生成循环代码哦~

4、在写 Index 模式的时候，尽可能去保存 list.size()，虽然 JIT 有可能会进行优化，但这种方式可以更加保险。

5、性能优化不只是整体架构或者类库的优化，也要从平时点滴做起。

最后，推荐大家看看： [Performance Tips](https://developer.android.com/training/articles/perf-tips.htm)

### 参考：

[To Index or Iterate?](https://www.youtube.com/watch?v=MZOf3pOAM6A)

[Use Enhanced For Loop Syntax](https://developer.android.com/training/articles/perf-tips.html#Loops)