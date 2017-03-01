---
layout: post
title: onSaveInstanceState 一些知识
date: 2016-11-03
categories: blog
tags: [编程]
description: 无
---

# onSaveInstanceState 一些知识

先来看下官方文档对于onSaveInstanceState的解释

> the method `onSaveInstanceState(Bundle)` is called before placing the activity in such a background state, allowing you to save away any dynamic instance state in your activity into the given Bundle, to be later received in `onCreate(Bundle)` if the activity needs to be re-created.

大概意思就是如果 内存不够了，那么系统就会强制杀死这个Activity，但是会调用onSaveInstanceState方法来保存Activity的状态。

例如，如果activity B启用后位于activity A的前端，在某个时刻activity A因为系统回收资源的问题要被杀掉，A通过onSaveInstanceState将有机会保存其用户界面状态，使得将来用户返回到activity A时能通过onCreate(Bundle)或者onRestoreInstanceState(Bundle)恢复界面的状态。 

 默认的实现负责了大部分UI实例状态(的保存)，比如文本信息，采用的方式是调用UI层上每个拥有id的view的onSaveInstanceState() ，并且保存当前获得焦点的view的id(所有保存的状态信息都会在默认的onRestoreInstanceState(Bundle)实现中恢复)。 

有这么几种情况，会调用`onSaveInstanceState(Bundle)`： 

1、当用户按下HOME键时。这是显而易见的，系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activity A是否会被销毁，故系统会调用onSaveInstanceState，让用户有机会保存某些非永久性的数据。

2、选择运行其他的程序时。 

3、按下电源按键（关闭屏幕显示）时。 

4、从activity A中启动一个新的activity时。 

5、屏幕方向切换时，例如从竖屏切换到横屏时。

6、键盘弹出（这个不确定）。

 可如果用户主动去销毁一个Activity（back），那么，说明了用户不需要保存这个Activity的状态，那么那么`onSaveInstanceState(Bundle)`不会被调用，所以`onSaveInstanceState(Bundle)`只适合保存一些临时的状态，比如说文本输入。所以最好还是在`OnPause`中去保存永久性的一些数据。

但

> Note that it is important to save persistent data in `onPause()` instead of `onSaveInstanceState(Bundle)` because the latter is not part of the lifecycle callbacks, so will not be called in every situation as described in its documentation.

官方建议是需要持久化数据的操作应该放在onPause里面，因为`onSaveInstanceState(Bundle)`并不是生命周期的一部分，有时候可能没有被调用。



写这篇文章的原因是因为遇到了这样的一个问题：

```java
java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
```

上网查了以后，好像是在切换fragment的时候，系统因为未知原因把这个呈现fragment的Activity给销毁了。

如果在保存完状态后再给Activity添加Fragment就会出错，一种解决方法是把commit换成commitAllowingStateLoss。

在stackoverflow上，有个回答：

Please check my answer [here](http://stackoverflow.com/a/10261438/542091). Basically I just had to :

```
@Override
protected void onSaveInstanceState(Bundle outState) {
    //No call for super(). Bug on API Level > 11.
}
```

don't make the call to `super()` on the `saveInstanceState` method. This was messing things up...

**EDIT:** after some more research, this is a know [bug](http://code.google.com/p/android/issues/detail?id=19917) in the support package.

If you need to save the instance, and add something to your `outState` `Bundle` you can use the following :

```
@Override
protected void onSaveInstanceState(Bundle outState) {
    outState.putString("WORKAROUND_FOR_BUG_19917_KEY", "WORKAROUND_FOR_BUG_19917_VALUE");
    super.onSaveInstanceState(outState);
}
```

**EDIT2:** in the end the proper solution was (as seen in the comments) to use :

```
transaction.commitAllowingStateLoss();
```

when adding or performing the `FragmentTransaction` that was causing the `Exception`.











