---
layout: post
title: ListView 和RecyclerView
date: 2017-04-07
categories: blog
tags: [编程]
description: 无
---

# ListView 和RecyclerView

### ListView缓存机制

ListView一共有两种缓存：

- mActiveViews:显示在屏幕上View的缓存，只在`layoutChildren()`中用，防止重复数据。
- mScrapViews: 保存废弃的View缓存。

当你的列表在滑动的时候，离屏的Item会被回收到缓存中，即将入屏的Item则会优先从缓存中去取。

但是ListView是有viewType的概念的，也就是说，如果viewType不一样，会放到不一样的缓存中。

这样设计的优点在于：因为你屏幕的大小是固定的，那么**一般来说**用户看到的Item数量是固定的（考虑到可能出现最上面的Item出现一半，最下面的Item出现一半），那么Adapter只需要生成固定数量个View，然后不断去替换刷新这些View的数据就可以了。

![img](https://i.stack.imgur.com/VLG9g.jpg)

在`getView(final int position, View convertView, ViewGroup parent)`中：

如果代码是这样，借用第一行代码：

```java
public View getView(final int position, View convertView, ViewGroup parent){
  	 Fruit fruit = list.get(position); //获取当前位置的水果
  	 View view;
  	 ViewHolder viewHolder;
  	 if(convertView == null){ //分支1
  	   view = LayoutInflater.from(mContext).inflate(resId,null); 
  	   viewHolder = new ViewHolder();
  	   viewHolder.fruitImage = (ImageView) view.findViewById(R.id.image);
  	   viewHolder.fruitName = (TextView) view.findViewById(R.id.name);
  	   view.setTag(viewHolder); //将viewHolder存储在View中
  	 }else{                  //分支2
  	   view = convertView; 
  	   viewHolder = (ViewHolder) view.getTag(); //重新获取ViewHolder
  	 }
  	 //刷新数据
  	 viewHolder.fruitImage.setImageResource(fruit.getImageId());
  	 viewHolder.fruitName.setText(fruit.getName());
  	 return view;
}
//省略ViewHolder
```

首先分析Item 1—Item 7显示在屏幕上的时候，所有的`convertView`都为null，所以进入分支1，会创建7个View.

然后，当Item1被移除屏幕，Item8被移入屏幕。根据最上面那副图的原理可以看出，如果Item 1移除屏幕外之后会进入一个缓存池，当Item 8要使用的时候，会调用`getView()`，这时候的`convertView`其实是缓存池中的Item 1一开始创建的View，所以进入分支2，只需要从`convertView`中获取viewHolder就可以了。

当没有`ViewHolder`的时候：

```java
@Override  
public View getView(int position, View convertView, ViewGroup parent) {  
    Fruit fruit = getItem(position);  
    View view;  
    if (convertView == null) {  
        view = LayoutInflater.from(getContext()).inflate(resourceId, null);  
    } else {  
        view = convertView;  
    } 
    //每次还需要重新findViewById
    ImageView fruitImage = (ImageView) view.findViewById(R.id.fruit_image);  
    TextView fruitName = (TextView) view.findViewById(R.id.fruit_name);  
    //刷新数据
    fruitImage.setImageResource(fruit.getImageId());  
    fruitName.setText(fruit.getName());  
    return view;  
}  
```

> 不要将ListView的 layout_height和layout_width设置成**wrap_content**，因为这样会在`getView()`中强制去测量子项的高度，可能引发一些问题。

#### 图片乱序:[图片乱序](http://blog.csdn.net/guolin_blog/article/details/45586553)  [trinea](http://www.trinea.cn/android/android-listview-display-error-image-when-scroll/)

由于之前`getView()`中去重用了`convertView`，所以会出现图片乱序的行为。

有这些情况：

- 显示错误图片

  新出现的Item却显示了屏幕外Item的图片，这是因为新出现的Item重用了之前Item的View，而**图片却来不及加载**（一般图片加载是异步的）。比如说Item 1图片是梨子，而Item 8本来应该显示苹果，却显示成了梨子。

- 显示图片闪烁

  在上一种情况的基础上，Item 8很快将梨子替换成苹果。

详细解决方案可以看贴上的2个链接。

#### 优化ListView:[流畅滑动](https://developer.android.com/training/improving-layouts/smooth-scrolling.html)

在`getView()`中不要做任何耗时操作，而且需要使用ViewHolder。

如果只是局部更新，不要傻乎乎的直接adapter全部刷新，可以自己去写局部刷新的操作。

> 流畅滑动的关键之处就在于不要在主线程中做复杂操作。这些复杂操作包括：I/O，网络交互，数据库查询。

在用户滚动列表的时候并不要去加载图片，可以当用户停止滚动列表的时候才去加载图片。

### RecyclerView 缓存 [RecyclerView](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286405&idx=1&sn=414e2d2eb577884ccee5c9076e8b8357&chksm=8334c387b4434a9124f5acd93f331968a44256b8374eeafb4b1857671072b3b6364e5ec38485&mpshare=1&scene=1&srcid=1021WRiw6K6yEuMcvwRJvTI4#rd)

RecyclerView 一共有四种缓存：

- mAttachedScrap:显示在屏幕上View的缓存，只在`layoutChildren()`中用，防止重复数据。
- mCachedViews : 保存屏幕外的View缓存。
- mViewCacheExtension：留给开发人员自定义。
- mRecyclerPool:所有的RecyclerView都可以共用的缓存。

可以将RecyclerView.ViewHolder 理解为View + ViewHolder(避免每次createView时调用findViewById) + flag(标识状态)。

与ListView不同的是，mCachedViews在重用时，是通过匹配pos，并不需要像`ListView#getView()`一样。

而RecyclerView在局部刷新上是优于ListView的。

### 结论

在数据不会频繁变动的情况下，ListView和RecyclerView的性能差异并不明显，而RecyclerView的优势在于配置的多可选择性（Item分隔，layoutManager），且动画效果更佳。

ListView原生是不支持局部刷新的，而RecyclerView支持。

但也可以自己去写个局部刷新（单个），范围刷新可以类似：

```java
private void updateItem(int index) {
    int visiblePosition = listView.getFirstVisiblePosition();
    if (index - visiblePosition >= 0) {
        //得到要更新的item的view
        View view = listView.getChildAt(index - visiblePosition);
        // 更新界面（示例参考）
        // TextView nameView = ViewLess.$(view, R.id.name);
        // nameView.setText("update " + index);
        // 更新列表数据（示例参考）
        // list.get(index).setName("Update " + index);
    }
}
```

还有要注意的是关于Item的点击方法，ListView是提供了Item的点击方法的，但是RecyclerView时没有提供这个方法：  [知乎讨论](https://www.zhihu.com/question/30336190)。

大概原因就是因为Rv引入了动画，而且可能在布局时会出现几个Item重叠的情况。

但是在RecyclerView中你可以在ViewHolder创建时去为每个子View去绑定点击方法。

如果要解决的话也有方法 [RecyclerView onClick](http://stackoverflow.com/questions/24471109/recyclerview-onclick/26196831#26196831)

大概思路就是实现`RecyclerView$OnItemTouchListener()`，然后在拦截方法`onInterceptTouchEvent()`中先获取被点击的Item，然后通过`GestureDetector`去监听点击和长点击事件。

### RecyclerView解析：[解析](https://blog.saymagic.tech/2016/10/21/understand-recycler.html)