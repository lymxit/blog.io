---
layout: post
title: Fragment中使用instantiate方法创建
date: 2016-10-12
categories: blog
tags: [编程]
description: 无
---

# Fragment中使用instantiate方法创建

最近在使用Github上的SmartTabLayout时，发现它提供了一个工具库util，里面有FragmentPagerItemAdapter可以直接使用，不需要自己去新写一个Adapter了。

但这样的话， 它用到了一个FragmentPagerItems的辅助类。而这个辅助类管理了几个FragmentPagerItem，我们如果使用的话，就是通过这个类去创建我们想要的Fragment。

```Java
FragmentPagerItems pages = new FragmentPagerItems(mContext);
//add fragments
pages.add(FragmentPagerItem.of(getString(R.string.demo_tab_no_title), Fragment_demo.class));
```

然后通过查看源码，我发现它是通过下面的代码来创建Fragment:

```Java
public Fragment instantiate(Context context, int position) {
  setPosition(args, position);
  return Fragment.instantiate(context, className, args);
}
```

其中，`Fragment.instantiate(context, className, args);`就是真正创建Fragment的代码，这个函数前两个参数不需要多提了，最后一个参数是一个Bundle。下面我们看一下 instantiate()：

```java
/**
 * Create a new instance of a Fragment with the given class name.  This is
 * the same as calling its empty constructor.
 *
 * @param context The calling context being used to instantiate the fragment.
 * This is currently just used to get its ClassLoader.
 * @param fname The class name of the fragment to instantiate.
 * @param args Bundle of arguments to supply to the fragment, which it
 * can retrieve with {@link #getArguments()}.  May be null.
 * @return Returns a new fragment instance.
 * @throws InstantiationException If there is a failure in instantiating
 * the given fragment class.  This is a runtime exception; it is not
 * normally expected to happen.
 */
```

调用这个函数，就相当于调用了空的构造器函数，第三个参数可以在OnCreateView()函数中通过getArguments()获得。

如何通过Bundle来传递对象或者对象数组可以在百度上查，通过

```Java
Fragment.instantiate(context, MyFragment.class.getName(), myBundle)
```

也可以创建一个Fragment，可能没什么优点。

相关Fragment的创建可以看看这个：

http://stackoverflow.com/questions/9245408/best-practice-for-instantiating-a-new-android-fragment

