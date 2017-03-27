---
layout: post
title: IntentService + Timer  VS AlarmManager
date: 2017-01-03
categories: blog
tags: [编程][Android]
description: 无
---

# IntentService + Timer  VS AlarmManager

参考资料：[[Repeating IntentService using Timers- is it advisable?]](http://stackoverflow.com/questions/24241804/repeating-intentservice-using-timers-is-it-advisable)

​		    [Scheduling Repeating Alarms](https://developer.android.com/training/scheduling/alarms.html)

​                    [Handler vs Timer : fixed-period execution and fixed-rate execution android development](http://androidtrainningcenter.blogspot.com/2013/12/handler-vs-timer-fixed-period-execution.html)

之前做了一个项目使用了IntentService来启动一个后台进程，然后在`IntentService#onHandleIntent()`创建一个Timer来进行一个定时任务调度。

如果想要对用户更友好化（节约内存和CPU），应该拒绝这种方式。

> 1、使用一个`Service`(`IntentService`也是)会提高你应用进程的优先级，但同时这意味着你的应用需要更多的内存去，所以会剥夺其他后台应用的内存，用户体验不佳。
>
> 2、由于太多开发人员让应用的`Service`跑很长时间，Android系统会自动结束`Service`（尤其国内定制ROM）。
>
> 3、由于`Service`中的`Timer`一直在后台跑（会占用CPU），这会调用`WakeLock`，这样就对电池的损耗很大了，用户在谷歌商店的评级就会很低。

**相反，如果使用`AlarmManager`**：

- `IntentService`可以得到正常的关闭(上一种情况中一直在跑，会浪费掉IntentService执行完任务关闭的特性)，只需要定时启动`IntentService`进行一次逻辑代码处理，然后`IntentService`会得到正常关闭。
- 同时，通过使用`WakefulBroadcastReceiver`来启动Service，可以更加节约用户的电量。

接下来，先看看使用`AlarmManager`的好处：

- 可以定时发送Intent
- 可以和`BroadcastReceiver`一起用来启动服务和进行其他操作
- 它可以在我们应用生命周期之外（APP被杀死）运行，因此即使应用没有运行或者设备处于休眠状态，可以用它来触发事件
- 可以在最大程度上减少应用资源的需求，我们可以不使用`IntentService`+`Timer`也可以运行定时任务。

> **Note**:对于保证在**应用程序生命周期中**发生的定时操作，优先选择结合了`Timer`和`Thread`的`Handler`类。

### 适用场景

一个常见的场景就是同步数据，我们可以在APP被关闭之后将客户端的数据和服务器的数据进行同步（如果可以使用GCM，优先使用GCM）。

> 不过可以试想这样一个场景，如果同步数据的操作是基于时间的（比如每天晚上11点），那么所有用户的同步操作都会发生在11点，这样对服务器是很不友好的，所以应该这样：
>
> - 尽量减少alarm的频率
> - 首先，当触发了alarm，可以先做一些"本地操作"，也就是不与服务器交互的操作；然后，在将来一个随机的时间调度alarm来执行"网络同步操作"。
> - 除非必要，不要唤醒设备（与**Alarm Type** 有关系 ）。
> - 可以使用`setInexactRepeating()`替代`setRepeating()`，前面这个方法会让系统在合适的时间去为所有的APP添加Alarm，可以更节约资源。

### 设置重复警报(Alarm)

一个重复Alarm有一下几点特性：

- 警报种类Alarm Type
  - ELAPSED_REALTIME——根据设备启动后的时间去发送待处理的`Intent`，但不唤醒设备，比如可以每30分钟发送一次警报；
  - ELAPSED_REALTIME_WAKEUP——与上面相对，会唤醒设备；
  - RTC——在某个确切的时间去发送`Intent`，不唤醒设备，比如下午2:00发送警报（用户可能会修改系统时间）；
  - RTC_WAKEUP——与上面相对，会唤醒设备。
- 触发时间
  - 对于绝大多数APP，`setInexactRepeating`就足够需求了。但如果你需要去**精确**去控制触发时间的话，还是使用`setRepeating`（这个会立即生效）。
- 警报之间的间隔
  - 必须使用间隔常量，比如`INTERVAL_FIFTEEN_MINUTES`, `INTERVAL_DAY`。
- 一个待处理的Intent

### 取消警报

```java
//通过AlarmManager取消待处理的Intent
if (alarmMgr!= null) {
    alarmMgr.cancel(alarmIntent); 
}
```

### 当设备启动的时候触发一个警报

默认情况下，当设备关机后，所有的警报都会被取消。所以当设备启动的时候，应该自动重启警报。（这里略去过程）

## Handler VS Timer

Handler可以更加灵活，比如说在有2个Runnable不同时间调度的场景下，但这两个Runnable其实还是在handler的进程里面跑（不一定能保证时间间隔），直接调用的是`Handler#handleCallback()`，里面直接调用run方法：

```java
 Handler mHandler;
  public void useHandler() {
    mHandler = new Handler();
    mHandler.postDelayed(mRunnable1, 1000);
    mHandler.post(mRunnable2);      
  }

  private Runnable mRunnable1 = new Runnable() {

    @Override
    public void run() {
      Log.e("Handlers", "mRunnable1");
      /** Do something **/
      mHandler.postDelayed(mRunnable1, 1000);
    }
  };
  private Runnable mRunnable2 = new Runnable() {

    @Override
    public void run() {
      Log.e("Handlers", "mRunnable2");
      /** Do something **/
      mHandler.postDelayed(mRunnable2, 3000);
    }
  };
```

Handler在内存泄露上会更胜一筹，两者调度的语法差别不大.

具体可以参见这篇文章 [Handler vs Timer : fixed-period execution and fixed-rate execution android development](http://androidtrainningcenter.blogspot.com/2013/12/handler-vs-timer-fixed-period-execution.html)

