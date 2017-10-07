---
layout: post
title: onSaveInstanceState&&onRestoreInstanceState必知必会
date: 2017-08-29
categories: blog
tags: [编程]
description: 无
---

# onSaveInstanceState && onRestoreInstanceState必知必会

之前很少处理过Activity异常情况，最近解决bug的时候遇到了这些问题，特此记录下来，方便以后查阅，

如果有什么错误欢迎大家批评指正。

本文基于sdk25（7.1）进行分析，主要分析onSaveInstanceState、onCreate、onRestoreInstanceState。

**先来个总体概括吧，本文主要讲了以下内容：**

- `onSaveInstanceState`和`onPause`的区别
- `onSaveInstanceState`的几个注意事项
- `onCreate`和`onRestoreInstanceState`的区别及其使用

### onSaveInstanceState

首先看到这么长的javadoc你可能会很恐惧：

```java
/**
     * Called to retrieve per-instance state from an activity before being killed
     * so that the state can be restored in {@link #onCreate} or
     * {@link #onRestoreInstanceState} (the {@link Bundle} populated by this method
     * will be passed to both).
     *
     * <p>This method is called before an activity may be killed so that when it
     * comes back some time in the future it can restore its state.  For example,
     * if activity B is launched in front of activity A, and at some point activity
     * A is killed to reclaim resources, activity A will have a chance to save the
     * current state of its user interface via this method so that when the user
     * returns to activity A, the state of the user interface can be restored
     * via {@link #onCreate} or {@link #onRestoreInstanceState}.
     *
     * <p>Do not confuse this method with activity lifecycle callbacks such as
     * {@link #onPause}, which is always called when an activity is being placed
     * in the background or on its way to destruction, or {@link #onStop} which
     * is called before destruction.  One example of when {@link #onPause} and
     * {@link #onStop} is called and not this method is when a user navigates back
     * from activity B to activity A: there is no need to call {@link #onSaveInstanceState}
     * on B because that particular instance will never be restored, so the
     * system avoids calling it.  An example when {@link #onPause} is called and
     * not {@link #onSaveInstanceState} is when activity B is launched in front of activity A:
     * the system may avoid calling {@link #onSaveInstanceState} on activity A if it isn't
     * killed during the lifetime of B since the state of the user interface of
     * A will stay intact.
     *
     * <p>The default implementation takes care of most of the UI per-instance
     * state for you by calling {@link android.view.View#onSaveInstanceState()} on each
     * view in the hierarchy that has an id, and by saving the id of the currently
     * focused view (all of which is restored by the default implementation of
     * {@link #onRestoreInstanceState}).  If you override this method to save additional
     * information not captured by each individual view, you will likely want to
     * call through to the default implementation, otherwise be prepared to save
     * all of the state of each view yourself.
     *
     * <p>If called, this method will occur before {@link #onStop}.  There are
     * no guarantees about whether it will occur before or after {@link #onPause}.
     *
     * @param outState Bundle in which to place your saved state.
     *
     * @see #onCreate
     * @see #onRestoreInstanceState
     * @see #onPause
     */
```

我简要地总结一下吧：

1、该方法用来临时保存Activity的**UI状态**（注意和`onPause()`的区分，后面会讲）

2、保存的数据是保存在系统进程里面的（通过IPC传过来）

3、该方法是发生在`onPause()`到`onStop()`之间的

**接下来还需要针对以上三点进行仔细讲解：**

#### **1、保存UI状态**

**特别提醒：覆盖Activity的onSaveInstanceState时记得调用super.onSaveInstanceState()!**

​      代码其实很简单：

```java
protected void onSaveInstanceState(Bundle outState) {
        outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
        Parcelable p = mFragments.saveAllState();
        if (p != null) {
            outState.putParcelable(FRAGMENTS_TAG, p);
        }
  		//ActivityLifeCycler的回调，可以无视
        getApplication().dispatchActivitySaveInstanceState(this, outState);
}
```

可以看到上述代码中，保存了**当前UI树的状态**，还会通知内部的Fragments去保存状态。

 对于前者，有坑要注意：

1、UI控件需要有id，这样才会去调用View的`onSaveInstanceState`..

2、id不能有重复，即使**局部唯一**也不可以（虽然android中对于id的重复要求只要求**局部唯一**）

> **局部唯一是什么意思呢？**
>
> ​					activity1
>
> ​                                   /                     \
>
> ​                              layout1            layout2
>
> ​                             /                                \
>
> ​                        button                           button
>
> 这时候，我们通过layout1.findViewById(R.id.button)和layout2.finViewById(R.id.button)没问题，
>
> 这里不展开了。

其实这两点都可以通过源码`PhoneWindow#saveHierarchyState()`来理解：

```java
public Bundle saveHierarchyState() {
        Bundle outState = new Bundle();
        if (mContentParent == null) {
            return outState;
        }
		
        SparseArray<Parcelable> states = new SparseArray<Parcelable>();
        //这里的mContentParent就是DecorView，最终调用到View中去
        mContentParent.saveHierarchyState(states);
        outState.putSparseParcelableArray(VIEWS_TAG, states);
		
        // .... 一些其他处理

        return outState;
    }
```

然后会调用`View#dispatchSaveInstanceState(container)`：

```java
protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        // 如果没有id，就没有key，那么肯定保存不了
        if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
            mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
            Parcelable state = onSaveInstanceState();
            if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
                throw new IllegalStateException(
                        "Derived class did not call super.onSaveInstanceState()");
            }
            if (state != null) {
                // Log.i("View", "Freezing #" + Integer.toHexString(mID)
                // + ": " + state);
                container.put(mID, state);  //保存下来
            }
        }
    }
```

到这里就真相大白了，为什么没有id不能保存状态呢？为什么相同的id会有冲突呢（后一个会掩盖掉前一个）？

`SparseArray`是android独有的数据结构，其作用类似于`hashmap<Integer,Value>`。

没有id，那么肯定就不能保存下来状态；相同的id，则会产生掩盖效果，后一个会掩盖掉前一个。

#### 1.1、不要和onPause混淆

​       这里主要讲的是关于`onSaveInstanceState`的使用，什么时候我们应该使用它呢？

**只用它来保存与Activity UI有关的数据，不要保存太多的数据。**

比如说你有一个自定义的底端tab、有一个viewpager，一开始初始化的时候会在tab#0：

![mark](http://oicc5e0b7.bkt.clouddn.com/blog/20170829/211218587.png)

然后我们跳转到旅游页面**tab#1：**

![mark](http://oicc5e0b7.bkt.clouddn.com/blog/20170829/211345919.png)

假设这时候我们弹出去了，而且后台把我们的进程干掉了，再次回来时可能会出现：

![mark](http://oicc5e0b7.bkt.clouddn.com/blog/20170829/211546494.png)

因为系统提供的viewpager已经处理好了状态恢复，而我们使用的tab可能并没有处理好这个关系。

这时候有两种选择：1、修改tab的自定义控件，自己恢复状态 2、使用onSaveInstanceState()

其实都很简单就不说了。

**那么为什么要注意它和`onPause()`的区别呢？**

1、因为怕有人误认为它是lifecycle的必经一环，`onPause`在生命周期过程中，是**必然会被调用的**，而

`onSaveInstanceState`就不一定了，官方文档只说它发生在`onPause`和`onStop`之间。

2、它只处理Activity UI的状态存储（界面导航、界面状态），而`onPause()`可以做一些轻量级的存储。

这边举一个例子，假设你做一个视频播放，要保存当前的播放状态：

```java
public class PlayerActivity{
  private static final int INIT = 0;
  private static final int PLAYING = 1;
  private static final int PAUSE = -1;
  private static final int STOP = -2;
  
  private int mPlayerStatus = INIT; //播放状态
  private int mCurrentTime; //当前播放时间
  private int mMaxTime; //视频的最大长度
  
  
  private SeekBar mProgressBar; //进度条
  private ImageButton mPlayerButoon;  //播放按钮
  
  private int mVideoId; //播放的视频id
  
  protected void onCreate(@Nullable Bundle savedInstanceState){
    
    //从sp中获取播放的进度，没有的话就为0
    int time = SharedPreferenceUtil.getInt(mVideoId + "currentTime",0);
    mCurrentTime = time;
    if(savedInstanceState != null)
      mPlayerStatus = savedInstanceState.getInt("status",INIT);
    
    //balabala初始化View相关
  }
  
  protected void onPause(){
    SharedPreferenceUtil.putInt(mVideoId + "currentTime",mCurrentTime);
    mPlayerStatus = PAUSE; // 自动暂停
  }
  
  protected void onSaveInstanceState(Bundle outState){
    // 保存当前播放状态
    outState.putInt("status",mPlayerStatus);
  }
  
 
}
```

上面的代码还有很多不完善的地方，但想表达一个意思，关于播放位置（mCurrentTime）是持久化存储的，**即使用户退出app**，下次进来也可以恢复播放位置。

而播放状态（mPlayerStatus）只会存在内存中保存，比如说用户暂停视频播放，然后切出去，回来的时候还是暂停状态，**但重启app后进来就会正常播放**。

#### 2、数据保存在系统进程中

这里的话需要一些前置知识，比如说Activity的启动流程、Binder等。

这边简单分析一下，`ActivityThread#callCallActivityOnSaveInstanceState()`：

```java
private void callCallActivityOnSaveInstanceState(ActivityClientRecord r) {
        r.state = new Bundle();
        r.state.setAllowFds(false);
        if (r.isPersistable()) {
            r.persistentState = new PersistableBundle();
            mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state,
                    r.persistentState);
        } else {
            mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state);
        }
    }
```

可以看到`onSaveInstanceState(Bundle outState)`中的outState其实是保存到了`ActivityClientRecord`中，

`ActivityClientRecord `其实就是Activity的一种体现，`ActivityRecord`则是对应Server端的实现。

因此在内存紧张时，`AcitivityRecord`还在AMS中好好地保存着（除非AMS也挂掉），它有个成员变量：

`Bundle  icicle;         // last saved activity state`

> 但值得注意的是数据是通过binder传输，而binder传输数据是有大小限制的，所以最好不要存放过多数据。

#### 3、方法发生在onPause和onStop之间

因为API版本的不同，sdk底层实现方式也不一样，所以这点就不探究了，感兴趣的可以在ActivityThread中自己去看。

> 关于`onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState)`这个重载函数
>
> 其实很明显PersistableBundle可以存在硬盘上，但这个函数意义不大（你可以通过SqlLite或者其他的方式自己实现，还更加灵活），猜想是给新手使用（不需要接触数据库和持久层），单要设置activity的属性：android:persistableMode="persistAcrossReboots"

### onRestoreInstanceState和onCreate

由于前面`onSaveInstanceState`都讲的差不多了，这边主要讲一下`onRestoreInstanceState`和`onCreate`的区别。

看下两者的函数签名：

- `onCreate(@Nullable Bundle savedInstanceState)`
- `onRestoreInstanceState(Bundle savedInstanceState)`

`onCreate`的@Nullable表示这个参数可能为空，那么什么情况下不为空呢？

官方文档解释：

```java
/**
If the activity is being re-initialized after  previously being shut down then this Bundle contains the data it most recently supplied in {@link #onSaveInstanceState}.
**/
```

翻译一下就是说如果这个Activity不是初次启动时（被Lower Memory Killer回收了内存，或者屏幕旋转），bundle就不为空。

`onRestoreInstanceState`其实是另外一种可供选择，它如果被调用时，说明Activity发生了异常的情况（被Lower Memory Killer回收了内存，或者屏幕旋转），和前面`onCreate`的bundle不为空情况是一样的。

不过有几点需要注意：

- `onRestoreInstanceState`发生在`onStart`之后，那么如果你决定在`onRestoreInstanceState`恢复UI状态，那么可能你需要考虑用动画来呈现这个变化的效果（比如从tab#1跳到tab#3)
- `onCreate`的职责是非常重的，尤其是出现多层继承关系的时候，什么时候需要调用父类的`onCreate`是值得权衡的事情，而且一旦将恢复bundle的代码写在`onCreate`中，这就很纠结了（onCreate有Side Effect)。

而`onRestoreInstanceState`就很纯粹了，恢复的逻辑都在这里面，想用或者不想用都非常clean，所以整体而言，更加推荐使用`onRestoreInstanceState`。

### 参考文章

http://blog.sina.com.cn/s/blog_618199e60101g1k5.html

https://stackoverflow.com/questions/28586443/android-view-onsaveinstancestate-not-called

http://www.cnblogs.com/xiaoweiz/p/3813914.html

https://stackoverflow.com/questions/3542333/how-to-prevent-custom-views-from-losing-state-across-screen-orientation-changes

https://stackoverflow.com/questions/17040703/replace-image-on-imagebutton

https://stackoverflow.com/questions/12683779/are-oncreate-and-onrestoreinstancestate-mutually-exclusive