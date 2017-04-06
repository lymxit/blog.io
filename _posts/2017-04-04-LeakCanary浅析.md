---
layout: post
title: LeakCanary浅析
date: 2017-04-04
categories: blog
tags: [编程]
description: 无
---

# LeakCanary浅析

由于Java语言的垃圾回收机制，很多时候我们以为垃圾回收的内存管理可以像灵丹妙药一样管。

但是如果碰到糟糕的代码和数据结构，Java中的内存泄露也会十分严重，那么首先我们得理解一下Java的垃圾回收机制（基于深入理解Java虚拟机）。

### Java垃圾回收

#### JVM管理的内存

大概可以分为两类：

- 堆：运行时数据区域。所有类示例和数组内存在这里分配。
- 非堆：包含方法去、JVM内部处理的内存、每个类结构以及类方法的代码。

我们主要关注的就是堆，一般GC也就是发生在这篇区域。

#### 堆内存的分类

为了方便理解，可以把一个对象的分配看成是一个婴儿。

Minor GC：发生在新生代的垃圾回收，每次回收的对象多，比较频繁，比较快。

Major GC/Full GC：发生在老年代的GC，一般由Minor GC触发，每次回收的对象少，比较慢。

- 新生代
  - Eden区：比较大，一般所有的宝宝都在这出生。
  - Survivor 1：如果宝宝熬过第一次Minor GC，就会被送到这来。
  - Survivor 2：和Survivor 1作用一致，只不过两者交替使用。
- 老年代：如果宝宝在Survivor区熬过15次 Minor GC（默认情况），就会被移动过来。
- 永久代：一般不用管。

用图来先看一下正常情况：

![mark](http://oicc5e0b7.bkt.clouddn.com/blog/20170406/151420450.png)

那婴儿一定要15岁才能移动到老年代吗？但是虚拟机还有**动态对象年龄判断**，如果Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，那么年龄大于或等于该年龄的对象就直接进入老年代。

又或者由于在垃圾回收的时候，由于Survivor区内存不够（复制算法），有些对象会直接分配到老年代。

关于Minor GC的流程，书中大致如下：

![mark](http://oicc5e0b7.bkt.clouddn.com/blog/20170406/155558362.png)



#### 垃圾回收算法

##### 如何判断哪些对象需要回收呢？标记算法

- 引用计数法：对于每个对象，都有一个引用计数器，当有其他对象引用它就+1，取消引用就-1。那么0则是垃圾。
- 可达性分析算法（Java采用）：根据"GC Roots"作为起点，去看哪些对象是可达的。不可达的则是垃圾。

引用计数法实现很简单，但是问题在于很难解决循环引用的问题：

![](http://oicc5e0b7.bkt.clouddn.com/blog/20170406/152444032.png)

这样2者就僵持在这里，谁都不能被识别成垃圾，可以用弱引用（内存不足时会回收）去解决。

哪些能作为GC Roots呢：

- 虚拟机栈中引用的对象（也就是线程局部变量）
- 方法区中类静态属性引用对象
- 方法区中常量引用的对象
- JNI引用对象

如果一个对象第一次被可达性分析算法判断为不可达，它也不会立刻死亡。

要真正宣告一个对象死亡，还需要看看它有没有覆盖`finalize()`（设计用于释放Native内存），如果覆盖了而且没有执行过，那么就会去调用，在这个方法中，对象还可以去"自救"。

##### 垃圾回收算法

- 标记清除（Mark-Sweep）：直接将被标记为垃圾的对象回收，但是容易产生内存碎片。
- 标记整理（Mark-Compact）：如果想在回收过程中将小的内存碎片合并成规整的空间，那么这种算法就很适合。它在回收的过程中将所有的对象往一边移动，然后清除掉另外部分的内存。
- 复制算法（Copying）：首先将可用内存划分为大小相等的部分，每次只用其中一块。在回收过程中，将不是垃圾的对象复制到另外一块上，然后清除掉这一块的所有内存。

复制算法就很适合**新生代**的垃圾回收，所以这就是为什么要划分成Eden、Survivor1和Survivor2的原因了。

一般来说三者的比例是8：1：1，每次使用Eden区和其中一块Survivor区。但如果另外一块Survivor区不足，那么就需要依赖其他内存进行辅助（一般是老年代），那么有部分新生代的内存直接就进入了老年代。

而标记整理和标记清除比较适合**老年代**。

关于Android GC：[Android GC 那点事](https://zhuanlan.zhihu.com/p/20282779) 

### LeakCanary

#### 使用

从没有见过这么简单的库，只需要在Application中调用` LeakCanary.install(this); `即可。

#### 原理

首先给个整体的流程图：

![mark](http://oicc5e0b7.bkt.clouddn.com/blog/20170407/014833153.png)

然后从入口开始：

```java
 public static RefWatcher install(Application application, Class<? extends AbstractAnalysisResultService> listenerServiceClass, ExcludedRefs excludedRefs) {
        if(isInAnalyzerProcess(application)) {  //判断是否是AnalyzerProcess进程
            return RefWatcher.DISABLED;            
        } else {
            enableDisplayLeakActivity(application);
            ServiceHeapDumpListener heapDumpListener = new ServiceHeapDumpListener(application, listenerServiceClass);
            RefWatcher refWatcher = androidWatcher(application, heapDumpListener, excludedRefs);
            //在Activity的生命周期中去加入回调函数  
            ActivityRefWatcher.installOnIcsPlus(application, refWatcher);
            return refWatcher;
        }
    }
```

看到有这么一个进程的判断，就联想是不是LeakCanary新开了一个进程，在AndroidManifest.xml中看到注册了一个`HeapAnalyzerService`，它单独跑在一个分析进程中。

```java
 protected void onHandleIntent(Intent intent) {
        if(intent == null) {
            CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.", new Object[0]);
        } else {
          	//将来利用反射去获取展示的Service Name，因为用户可以自定义Service
            String listenerClassName = intent.getStringExtra("listener_class_extra");
            //获取Heap的情况
            HeapDump heapDump = (HeapDump)intent.getSerializableExtra("heapdump_extra");
            HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);
            AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey); //分析获取结果
            AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result); //启动分析结果的Service
        }
    }
```

结果展示是通过`DisplayLeakService`，它的作用就是告诉我们哪里泄露了，直接看看它的`onHandleIntent()`:

```java
 protected final void onHandleIntent(Intent intent) {
        HeapDump heapDump = (HeapDump)intent.getSerializableExtra("heap_dump_extra");
        AnalysisResult result = (AnalysisResult)intent.getSerializableExtra("result_extra");

        try {
            this.onHeapAnalyzed(heapDump, result); //对结果进行分析，拼接字符串，通知结果
        } finally {
            heapDump.heapDumpFile.delete(); //删除文件
        }

    }
```

这里先不去具体分析如何分析泄露的算法，先看看是在什么时机去启动了一个新进程的`HeapAnalyzerService`.

找了半天，最后在`ServiceHeapDumpListener#analyze()`找到了：

```java
public void analyze(HeapDump heapDump) {
        Preconditions.checkNotNull(heapDump, "heapDump");
  		//会将堆的垃圾回收情况发送给分析的进程Service
        HeapAnalyzerService.runAnalysis(this.context, heapDump, this.listenerServiceClass);
    }
```

那么这个`ServiceHeapDumpListener#analyze()`又是在哪里调用的呢？

在`RefWatcher#ensureGone()` 中，找到了结果：

```java
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
	
    //尝试从queue中移除被GC对象的弱引用
    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    //如果引用已经被垃圾回收了，就直接结束
    if (gone(reference)) {
      return DONE;
    }
    gcTrigger.runGc(); //尝试调用GC，如果没被回收成功，那么就会使内存泄露
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
      //GC之后这个引用还在，那么就说明发生了内存泄漏
      long startDumpHeap = System.nanoTime();
      //计算GC耗时
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
	  //获取GC之后堆的情况
      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      //启动IntentService开始分析情况，在新的进程进行
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
    }
    return DONE;
  }
```

通过源码的查看，可以发现`ensureGone()`是由`AndroidWatchExecutor`去调度的。

这个`AndroidWatchExecutor`会在主线程空闲的时候往自己的后台线程去调度Runnable，这个过程有点复杂而且显得有些奇怪，但我们能知道所有的`ensureGone()`都是通过**一个后台线程**去完成的。

![mark](http://oicc5e0b7.bkt.clouddn.com/blog/20170407/010211320.png)

那又是什么时候去调用`ensureGoneAsync()`呢？这就来到了重点，`RefWatcher#watch()`：

```java
public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {   //如果在分析进程，就不运行
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    //生成一个随机的UUID key
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    //为每一个监控对象建立一个弱引用，同时加入引用队列queue，这个队列就是检测内存泄露的关键
    //如果这个对象被正常回收了，那么queue会自动清除掉这个引用
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    ensureGoneAsync(watchStartNanoTime, reference); //异步调用
  }
```

这里大概能猜到，LeakCanary应该是对我们应用中的每个对象调用了watch()，这样才能知道哪些对象被正常回收而哪些没有。

而实际上，watch()的调用是发生在Activity的onDestroy()之后，通过`Application.registerActivityLifecycleCallbacks()`就能去对Activity进行监控，如果Activity没有正常释放，那么说明内部有强引用没有得到正常释放，也就是内存泄漏。

那么怎么去监控Fragment呢？只需要我们自己去在Fragment的onDestory()中调用`RefWatcher#watch()`即可。