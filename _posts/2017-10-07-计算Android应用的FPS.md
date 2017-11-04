---
layout: post
title: 计算Android应用的FPS
date: 2017-10-07
categories: blog
tags: [编程]
description: 无
---

# 如何计算 Android 应用的 FPS

貌似某次面试的时候被问到如何计算 RecyclerView 的帧率，当时没回答出来，然后最近比较闲就找了一下。

在 Google I/O 2012 的时候，宣布了一个黄油计划，也就是垂直同步的引入，而且还介绍了`Choreographer`。

那么这个`Choreographer`又是什么呢？

首先中文翻译过来：`舞蹈指导`，然后看一下它的第一行注释：

> Coordinates the timing of animations, input and drawing.

这个类的主要作用就是协调动画、输入和绘制这三个过程（看来这个名字起得还是很不错的！）

> The choreographer receives timing pulses (such as vertical synchronization)
>
> from the display subsystem then schedules work to occur as part of rendering
>
> the next display frame.

这个类接收定时电子脉冲（比如说垂直同步），然后调度下一帧的渲染工作。

然后就是 [TinyDancer](https://github.com/friendlyrobotnyc/TinyDancer) 这个库，使用这个库直接可以计算出帧率。

那么它是什么原理呢？

首先，需要看一个接口 `Choreographer.FrameCallback`：

它有一个方法 `doFrame(long frameTimeNanos)`，看一下它的注释：

> Called when a new display frame is being rendered.

当新的一帧正在渲染时调用，参数是该帧渲染开始的时间。

所以可以看到 `TinyDancer`这个库有一个对象：`FPSFrameCallback`，听到这个名字很知道这个类是用来获取帧率的，来看下它的 `doFrame`是怎么写的吧。

```java
	@Override
    public void doFrame(long frameTimeNanos)
    {
        // 初始化，记录第一帧来的时间
        if (startSampleTimeInNs == 0){
            startSampleTimeInNs = frameTimeNanos;
        }
        // 如果用户自定义了callback，那么就告诉它
        // 默认这个callback为空
        else if (fpsConfig.frameDataCallback != null)
        {
            // 获取记录的上一帧的渲染开始时间
            long start = dataSet.get(dataSet.size()-1);
            // 缺失的帧数
            int droppedCount = Calculation.droppedCount(start, frameTimeNanos, fpsConfig.deviceRefreshRateInMs);
            // 回调
            fpsConfig.frameDataCallback.doFrame(start, frameTimeNanos, droppedCount);
        }

        // 如果记录的时间超过某个值，我们就应该收集帧率发送给UI了
        if (isFinishedWithSample(frameTimeNanos))
        {
            collectSampleAndSend(frameTimeNanos);
        }

        // 记录一下当前帧的渲染起始时间
        dataSet.add(frameTimeNanos);

        // 注册到Choreographer中
        Choreographer.getInstance().postFrameCallback(this);
    }
```

其实整体流程很简单，而且还能知道整体丢帧情况，如果在实际场景中发现丢帧太高还能做出相应调整（例如减少UI 工作等等），来让帧率暂时提升起来。

