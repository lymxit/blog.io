---
layout: post
title: 后台计划任务神器：Android-Job使用及解析
date: 2017-3-27
categories: blog
tags: [编程]
description: 无
---

# 后台计划任务神器：Android-Job使用及解析

[作者的PDF](https://speakerd.s3.amazonaws.com/presentations/c9e9289df7df48bab2bf52db0a087248/Schedule_background_jobs_at_the_right_time_-_MTC.pdf)

由于Android版本的割裂化，所以造成很多API被混着用：`JobScheduler`, `GcmNetworkManager` , `AlarmManager`

- `JobScheduler`：API21之后才加入，部分特性必须API24，[官方文档](https://developer.android.com/reference/android/app/job/JobScheduler.html)
- `GcmNetworkManager`：国内不支持
- `AlarmManager`：不同API版本的行为不太一样，模板代码太多

关于三者的使用情况，如下：

![img](https://lh3.googleusercontent.com/-T1FVw3kXxgE/VfHM5bk5Q7I/AAAAAAAAHAY/SRRp4V3AmOU/w530-h316-p-rw/Pro-tip%2BAlarms%2BFlowchart%2Bv2.png)

这里有一个关于`AlarmManager`的示例：

```java
public static void schedule(Context context, Reminder reminder) {
 // …
 if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
   //API 23-25
   manager.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, when, pendingIntent);
 } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
 manager.setExact(AlarmManager.RTC_WAKEUP, when, pendingIntent); //API 19-22
 } else {
 manager.set(AlarmManager.RTC_WAKEUP, when, pendingIntent);  //API14-18
 }
}
```

可以说Android-Job就是封装了`JobScheduler`, `GcmNetworkManager` , `AlarmManager`:

- 准时的工作      ——> AlarmManager
- API > 21           ——> JobScheduler
- 安装了谷歌服务 ——> GcmNetworkManager
- 其他情况           ——> AlarmManager

在上一篇文章[IntentService + Timer VS AlarmManager](http://jackieming.com/blog/2017/01/03/IntentService-+-Timer-VS-AlarmManager/)，提过如果重启之后，所有AlarmManager提出的警报全部会被清除，所以在**Android-Job**中自动帮你处理了这些事。

### 引入Android-Job

#### Gradle

```groovy
dependencies {
    compile 'com.evernote:android-job:1.1.8'  //878个方法，100kb的jar
}
```

Gradle会自动将permissions和services加入到[AndroidManifest](https://github.com/evernote/android-job/blob/master/library/src/main/AndroidManifest.xml).

#### 使用

所有的`Job`都被全局的一个`JobManager`所管理，所以首先需要在Application创建这个全局单例：

```java
public class App extends Application {
  	public static JobManager jobmanager;
    @Override
    public void onCreate() { //建议在onCreate方法创建，但也可以用其他方式
        super.onCreate();
        jobmanager = JobManager.create(this); //提供一个context
        //需要创建器
        jobmanager.addJobCreator(new DemoJobCreator()); 
        //也可以加入多个JobCreatorCreate
      //jobmanager.addJobCreator(new AnotherJobCretor()); 
    }
}
public class DemoJobCreator implements JobCreator {
    //使用工厂模式创建Job，这样就不需要使用反射啦
    @Override
    public Job create(String tag) {
        switch (tag) {
            case DemoSyncJob.TAG:
                return new DemoSyncJob();
            default:
                return null;
        }
    }
}
```

就像EventBus一样，我们需要继承`Job`类，例如`DemoSyncJob`：

```java
public class DemoSyncJob extends Job {

    public static final String TAG = "job_demo_tag";

    @Override
    @NonNull
    protected Result onRunJob(Params params) {
        //会在后台线程运行，写业务逻辑，比如说同步数据到服务器...
        return Result.SUCCESS; //也可以返回FAILURE、RESCHEDULE
    }
  
	//方便外部类使用
    public static void scheduleJob() {
        new JobRequest.Builder(DemoSyncJob.TAG)
                .setExecutionWindow(30_000L, 40_000L)  //java1.7写法，30000ms开始，40000ms结束
                .build()
                .schedule(); //每次调度其实都是一次JobRequest
    }
}
```

当一个`JobRequest#schedule()`执行之后，那么会调用`JobManager#schedule(JobRequest request)`：

```java
public void schedule(@NonNull JobRequest request) {
  		//必须要有JobCreator，这个mJobCreatorHolder中保存了一个List<JobCreator>
        if (mJobCreatorHolder.isEmpty()) {
            CAT.w("you haven't registered a JobCreator with addJobCreator(), it's likely that your job never will be executed");
        }
		//这个request是否已经存在，存在就覆盖之前的Job
        if (request.isUpdateCurrent()) {
            cancelAllForTag(request.getTag());
        }

        JobProxy.Common.cleanUpOrphanedJob(mContext, request.getJobId());
		//获取本机支持的API
        JobApi jobApi = request.getJobApi();
  		//是否是周期性任务
        boolean periodic = request.isPeriodic();  
        boolean flexSupport = periodic && jobApi.isFlexSupport() && request.getFlexMs() < request.getIntervalMs();

        if (jobApi == JobApi.GCM && !mConfig.isGcmApiEnabled()) {
            // shouldn't happen
            CAT.w("GCM API disabled, but used nonetheless");
        }

        request.setScheduledAt(System.currentTimeMillis());
        request.setFlexSupport(flexSupport);
        //将这个request存起来
        mJobStorage.put(request);
		//根据不同API版本，选择合适的JobProxy
        JobProxy proxy = getJobProxy(jobApi);
        if (periodic) {
            if (flexSupport) {
                proxy.plantPeriodicFlexSupport(request);
            } else {
                proxy.plantPeriodic(request);
            }
        } else {
            proxy.plantOneOff(request); //只调用一次
        }
    }
```

接下来就看看具体是怎么进行调用的吧，`JobProxy#plantOneOff(Request request)`：

```java
//看看API14的情况,JobProxy14
@Override
    public void plantOneOff(JobRequest request) {
        //这个Intent会启动PlatformAlarmReceiver
        PendingIntent pendingIntent = getPendingIntent(request, false);
		//使用AlarmManager
        AlarmManager alarmManager = getAlarmManager();
        if (alarmManager == null) {
            return;
        }
        try {
            if (request.isExact()) {
                plantOneOffExact(request, alarmManager, pendingIntent);
            } else {
                //不精确调用，使用set方法
                plantOneOffInexact(request, alarmManager, pendingIntent);
            }
        } catch (Exception e) {
            mCat.e(e);
        }
    }

protected void plantOneOffExact(JobRequest request, AlarmManager alarmManager, PendingIntent pendingIntent) {
  		//获取触发时间
        long triggerAtMillis = getTriggerAtMillis(request);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            alarmManager.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerAtMillis, pendingIntent);
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            alarmManager.setExact(AlarmManager.RTC_WAKEUP, triggerAtMillis, pendingIntent);
        } else {
           //API版本不支持精确调用
            alarmManager.set(AlarmManager.RTC_WAKEUP, triggerAtMillis, pendingIntent);
        }
        logScheduled(request); //日志记录
    }

protected long getTriggerAtMillis(JobRequest request) {
  	    //系统当前时间+（startMs+(endMs - startMs)/2)
        return System.currentTimeMillis() + Common.getAverageDelayMs(request);
    }
```

在API14的情况下，`AlarmManager`最后会启动`PlatformAlarmReceiver`，它会启动`PlatformAlarmService`，这个`PlatformAlarmService`里面有个线程池,在onStartCommand中调度`PlatformAlarmService#runJob()`:

```java
 private void runJob(Intent intent) {
        if (intent == null) {
            Cat.i("Delivered intent is null");
            return;
        }
		//从Intent中获得之前存的jobId，每个jobId都不一样
        int jobId = intent.getIntExtra(PlatformAlarmReceiver.EXTRA_JOB_ID, -1);
        final JobProxy.Common common = new JobProxy.Common(this, jobId);

        //根据jobId获取对应的request
        final JobRequest request = common.getPendingRequest(true);
        if (request != null) {
            common.executeJobRequest(request); //真正执行request的方法
        }
    }
```

OK，最后看看`JobProxy$Common#executeJobRequest(JobRequest request)`：

```java
JobExecutor jobExecutor = mJobManager.getJobExecutor();
            Job job = null;

            try {
                //通过TAG来获取之前的Job，尽量不要让不同Creator的Tag case一样
                job = mJobManager.getJobCreatorHolder().createJob(request.getTag());

                if (!request.isPeriodic()) {
                    request.setTransient(true);
                }
				//使用线程池去获取执行结果
                Future<Job.Result> future = jobExecutor.execute(mContext, request, job);
                if (future == null) {
                    return Job.Result.FAILURE;
                }

                // wait until done
                Job.Result result = future.get();
                mCat.d("Finished job, %s %s", request, result);
                return result;

            } catch (InterruptedException | ExecutionException e) {
                mCat.e(e);

                if (job != null) {
                    job.cancel();
                    mCat.e("Canceled %s", request);
                }

                return Job.Result.FAILURE;

            } finally {
                if (!request.isPeriodic()) {
                    mJobManager.getJobStorage().remove(request);

                } else if (request.isFlexSupport()) {
                    mJobManager.getJobStorage().remove(request); // remove, we store the new job in JobManager.schedule()
                    request.reschedule(false, false);
                }
            }
```

### 总结

Android-Job这个框架很好地去解决了不同版本之间的API割裂化的问题，是个很好的轮子。

同时使用了工厂模式去替代了反射，这在性能上也是个不错的做法。

