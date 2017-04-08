---
layout: post
title: SurfaceView
date: 2017-04-07
categories: blog
tags: [编程]
description: 无
---

# SurfaceView

### SurfaceView是什么

它继承自View，所以本质上它也是一个View，但与普通View不同，普通的View只能在UI线程中进行更新，而SurfaceView是在一个子线程中去更新自己，对于经常需要刷新View的场景，SurfaceView很有优势。

### 原理     [老罗参考](http://blog.csdn.net/luoshengyang/article/details/8661317)

为什么普通的View需要在UI线程才能更新呢？因为普通的view以及派生类都是共享同一个Surface的，但SurfaceView有自己的Surface。

这个Surface在Z轴方向上是在持有Surface的window后面的，但SurfaceView在window上挖了一个洞（透明化）以便于显示出SurfaceView。

![img](http://img.my.csdn.net/uploads/201303/11/1363016714_1787.jpg)

#### Surface类

> Handle onto a raw buffer that is being managed by the screen compositor.

操作一块Raw buffer（缓存区），这个缓存区由Sreen compositor（SurfaceFlinger）管理。

[Android 显示原理简介](http://djt.qq.com/article/view/987)

> ​    首先，用一句话来概括一下Android应用程序显示的过程：**Android应用程序调用SurfaceFlinger服务把经过测量、布局和绘制后的Surface渲染到显示屏幕上。**

Surface中有这些成员，可以看到最重要的属性应该就是这个`Canvas`：

```java
    // Guarded state.
    final Object mLock = new Object(); // protects the native state
    private String mName;
    long mNativeObject; // package scope only for SurfaceControl access
    private long mLockedObject;
    private int mGenerationId; // incremented each time mNativeObject changes
    private final Canvas mCanvas = new CompatibleCanvas(); //画布对象

    // A matrix to scale the matrix set by application. This is set to null for
    // non compatibility mode.
    private Matrix mCompatibleMatrix;

    private HwuiContext mHwuiContext;
```

根据网络上的资料大概可以了解到，Surface类是用来管理数据的，而且一个Surface对应一个Window。

![image](http://wiki.jikexueyuan.com/project/deep-android-v1/images/chapter8/image009.png)

再往底层去探究就没意义了，我们的主要的目的还是要明白SurfaceView的优点和原理。

#### SurfaceView和Window的绑定

```java
if (mWindow == null) {
                    Display display = getDisplay();
                    mWindow = new MyWindow(this);  //绑定window
                    mLayout.type = mWindowType;
                    mLayout.gravity = Gravity.START|Gravity.TOP;
                    mSession.addToDisplayWithoutInputChannel(mWindow, mWindow.mSeq, mLayout,
                            mVisible ? VISIBLE : GONE, display.getDisplayId(), mContentInsets,
                            mStableInsets);
                }
```

由于Surface和Window是一一对应的关系，所以SurfaceView也就是内嵌了一个Surface，因为从前面我们知道Surface只能画画，而SurfaceView可以说就是规定要画哪些东西（比如button和TextView）和东西位置。

#### SurfaceHolder

我们在SurfaceView中可以看到一个函数`getHolder()`，会返回一个`SurfaceHolder`【接口】，它的作用更像是一个Surface的监听器，比如说`SurfaceHolder$Callback#surfaceCreated()`能够感知Surface创建成功。同时它还提供了`lockCanvas()`和`unlockCanvasAndPost()`这样的接口（和Surface是一样的）。

> 总结：从架构的高度来看，Surface、SurfaceView和SurfaceHolder实质上就是广为人知的MVC，即Model-View-Controller。Model就是模型的意思，或者说是数据模型，或者更简单地说就是数据，也就是这里的Surface；View即视图，代表用户交互界面，也就是这里的SurfaceView；SurfaceHolder很明显可以理解为MVC中的Controller（控制器）。

#### 双缓存机制

SurfaceView在更新视图时用到了两张Canvas，一张frontCanvas和一张backCanvas，每次实际显示的是frontCanvas，backCanvas存储的是上一次更改前的视图。

当使用lockCanvas（）获取画布时，得到的实际上是backCanvas而不是正在显示的frontCanvas。

之后你在获取到的backCanvas上绘制新视图，再unlockCanvasAndPost（canvas）此视图，那么上传的这张canvas将替换原来的frontCanvas作为新的frontCanvas，原来的frontCanvas将切换到后台作为backCanvas。

例如，如果你已经先后两次绘制了视图A和B，那么你再调用lockCanvas（）获取视图，获得的将是A而不是正在显示的B，之后你将重绘的C视图上传，那么C将取代B作为新的frontCanvas显示在SurfaceView上，原来的B则转换为backCanvas。