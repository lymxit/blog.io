---
layout: post
title: retrofit + rxjava基本使用
date: 2017-04-01
categories: blog
tags: [编程]
description: 无
---

# retrofit + rxjava基本使用

### 基本使用

首先看一段代码，有个大概的概念.

```java
UserService service = retrofit.create(UserService.class);
service.getUser(userId)
    .observeOn(AndroidSchedulers.mainThread())  //在主线程回调
    .subscribe(new Consumer<User>() {
        @Override
        public void onNext(User user) {     //省略error handing
            userView.setUser(user);    //设置用户信息
        }
    });
```

### Retrofit基本原理

#### 动态代理

为什么那么复杂的网络操作只需要定义几个接口就能实现了呢？

记得以前在使用HttpClient的时候光是初始化和try catch就是占据了绝大多数的代码。那么Retrofit如何完成的？

因为Retrofit事先不知道开发者会定义哪些网络请求接口，而且又能让开发者很灵活的使用注解，所以采取了动态代理技术，所以我们所有的请求必须定义成接口的形式，因为jdk的动态代理只支持接口。

首先需要有些动态代理的知识：[公共技术点之 Java 动态代理](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)，大概有个概念就行了。

动态代理其实就相当于可以在运行的时候往你的函数里面去插入代码，如果有Spring的AOP基础其实就很好理解。

#### Retrofit的运用

那么看看动态代理怎么运用的呢？

`UserService service = retrofit.create(UserService.class);`

这行代码其实创建的对象是一个`UserService`的动态代理类，毕竟一个接口没有真正的代码逻辑怎么去运行呢。

在`Retrofit#create(Class)`中，利用动态代理和注解去帮我们处理了很多事情。

```java
public <T> T create(final Class<T> service) {
    一些判断...
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object[] args)
              throws Throwable {
            // 如果是从Object继承来的
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            // 估计是内部调试使用
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //获取我们接口中的Method,然后封装一下，同时获取注解信息（例如GET、POST、url）
            //这里使用了一个Map去缓存了方法，如果多次调用的话会更快
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            //根据每一个ServiceMethod都生成一个OkHttpCall
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //使用适配器模式将OkHttpCall转成Retrofit中的Call
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

#### 封装、再封装

上面可以看到Retrofit很巧妙地利用适配器模式将`OkHttpCall`转成了更方便使用的Retrofit的`Call`接口。

而在OkHttpCall中，又对OkHttp的Call（真正进行网络请求）进行了封装，关于`execute`和`enqueue`的同步异步处理就在`OkHttpCall`中，每一次网络调用其实都是一个`Request`，Call其实也只是一层封装。

### Json的转换

官方提供了一些转换，我这边使用了Gson库，所以看看`GsonConverterFactory`，感觉和Struts2的拦截器类似。

这个转换器类似于一个漏斗？丢进去一堆json数据，然后出来一个Gson转成的Item。

其实也很简单，`GsonResponseBodyConverter`类：

```java
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
  private final Gson gson;
  private final TypeAdapter<T> adapter;

  GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
    this.gson = gson;
    this.adapter = adapter;  //这个adapter是确定要转成的Object类型的
  }

  @Override public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      return adapter.read(jsonReader);
    } finally {
      value.close();
    }
  }
}
```

这样的话，直接就可以将网页的Json数据转成我们所需要的对象了。

### 能使用Rxjava的原理：[RxCallAdapter](http://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653577186&idx=1&sn=1a5f6369faeb22b4b68ea39f25020d28#rd)

#### Rxjava概念

基于观察者模式的定义，有两种对象：被观察者（Observable）和观察者（Observer）。

又或者可以看成：消息（Observable）和订阅者（Subscriber）。

网络请求就可以这么类比，比如做一个更新图片操作：

在非主线程进行了网络操作（io操作），那么这个操作可以得到一个图片结果（Observable），

在结果生成之后，订阅者（Subscriber）将获得结果生成的消息，根据结果的不同（成功或者失败）可以采取不同的策略，比如说成功就在主线程更新结果，而失败就在io线程记录日志。

这样的事情还能类比到其他地方上，比如说更新数据库，获取复杂计算的结果。

#### 基本使用：

```java
new Retrofit.Builder().baseUrl(baseUrl)
                .addCallAdapterFactory(RxJavaCallAdapterFactory.createAsync()) //rxjava
                .addConverterFactory(GsonConverterFactory.create())  //gson
                .client(httpClientBuilder.build()) //自定义的okhttp
                .build();
```

`RxJavaCallAdapterFactory`这个工厂类会将Retrofit的`Call`转成我们需要的`Observable`，具体的转换发生在`RxJavaCallAdapter#adapt(Call<R> call)`中。

`Create()`都是同步调用，所以如果有相关需求可以在创建`RxJavaCallAdapterFactory`时修改。

```java
 private RxJavaCallAdapterFactory(Scheduler scheduler, boolean isAsync) {
    this.scheduler = scheduler; //调度线程，默认为null
    this.isAsync = isAsync;    //是否异步调用，默认为false
  }
```

那么为什么之前都是使用Call<T>作为结果，现在可以使用Observable<T>了呢？

之前Retrofit的create方法，在动态代理中其实是可以返回任意类型的：

```java
new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //这里的callAdapter就是重点，使用的就是RxJavaCallAdapter
            //返回结果也就是Method的返回结果，也就是Observable
            return serviceMethod.callAdapter.adapt(okHttpCall); 
          }
```



