---
layout: post
title: Why RecyclerView
date: 2017-09-07
categories: blog
tags: [编程]
description: 无
---

# Why RecyclerView?

### ListView 做错了什么？

- 设计 ListView 之时，还没有动画的出现，所以 ListView 很难支持动画
- ListView 不能很方便地改变布局，例如横着、Grid、瀑布流
- API 设计不好，不同版本有兼容性问题
- 没有标准的 ViewHolder，但是大家都会用

就像是重启电脑一样，既然维护一个老古董很麻烦，干脆就另起炉灶，重新搞一个 RecyclerView

### Recycler 的出现

优点：

- ViewHolder 的正式版本
- Decouple 。例如 `Recycler` ，我们可以使用自己的 view pool。或者是将`ListView#getView()`拆分为了`create`和`bind`两个阶段。

三大主要组件：

- Layout Manager
- Item Animator
- Adapter

同时它还遵循了开闭原则，比如说 `ItemTouchHelper`、`SnapHelper`、`DiffUtil`。还有最新的Page库。

#### 最佳实践

`View::requestLayout`，它和 RecyclerView 没有半毛钱的关系，它隶属于 Android View 系统中。

当我们改变了某个 View ， 它就会说：“ 你猜发生了什么？因为某些东西改变了所以我请求一次 layout."

这句话逐渐从它传给它双亲，最后传给 root，root 说："OK，绘制下一帧的时候，我会通知你。"任何请求layout的 View，他们能做的就是等待。

下一帧到来之际，the root 会补偿所有它的 children，"Measure 你们自己。是时候得到你新的 layouts 了。"当然，对于没有请求 layout 的 View， 他们会使用自己的缓存 measurements。（意味着请求 layout 的 View 会标记自己的缓存数据为脏数据）

所以这个 view hierarchy 还是没有变化，这对 RecyclerView 有什么意义呢？

```java
onBindViewHolder(ViewHolder vh, int position) {
    ....
    imageLoader.loadImage(vh.image, user.profileUrl,
R.drawable.placeHolder);
}
```

当 imageLoader 从网络上下载下来后，将图片转为 bitmap，然后通知 ImageView，设置 bitmap。

当这些发生之时，ImageView 会说："哎，我的数据被废弃了，快让我请求一次 layout。"当然，所有 item 的 parent 都是 RecyclerView，它负责管理所有的 item。

这将会让 RecyclerView 去重新定位所有的 item，这样的话代价太高了！

RecyclerView 中有一个 `setHasFiexSize`，它就是用来减少代价的，如果一个 RecyclerView 有固定大小，那么它将知道自己将不会因为 自己 children 的改变而改变大小，所以它就不会去调用 `requestLayout`。

但这仅仅是一个限制性条件下的优化，re-layout RecyclerView 的代价依旧很高。

所以官方修改了`ImageView`，当我们更新`Drawable`时，它会进行检查："我是否需要改变自己的大小呢？"如果不需要的话，它就不会调用`requestLayout`，它仅仅只会调用`invalidate`，相对来说性能更好。

但我们如果看到`TextView`时，它说道："我不知道发生了什么，让我请求一次 layout 吧。"即使我们设置了相同的文本，依旧会调用`requestLayout`，因为它内部的实现非常复杂。

**所以尽可能地，要避免在 RecyclerView 中去重新设置 TextView。**

Resizing 则是这个问题的升级版本，当我们使用一个图片的瀑布流时，可能会出现某个图片没有缓存，导致整个 RecyclerView  都需要调整，因此我们需要一个展位图，大小正好是将来放置图片的大小。

因此需要 我们的 API 告诉我们图片的高度和宽度，让我们在加载之前提前可以放好展位图。

也就是**metadata**：

```json
{
    "user" : {
        "name" : "Michael",
        "photoUrl" : {
            "width" : 300,
            "height" : 500,
            "url" : "https://...",
            "palette" : {}
        }
    }
}
```

### 资源管理

#### ViewHolder 的生命周期

onCreate -> onBindViewHolder -> onViewDetachedFromWindow（不可见） -> onRecycled

onViewAttachedToWindow ： 播放视频

onViewDetachedFromWindow： 停止视频

所以我们应该在`onRecycled`去释放昂贵的资源，比如很大的 bitmap。

### RecyclerView 是异步的

这里的异步意思是什么呢？ 并不是说它是多线程的，它意味着它处理事情是异步的。

![](http://oicc5e0b7.bkt.clouddn.com/blog/20170917/223515122.png)

我们请求`scrollToPosition`，它会在下一帧到来之际才发生改变，比如以下代码：

```java
recyclerView.scrollToPosition(15);
int x = layoutManager.getFirstVisibleItemPosition();
```

这边拿到的 x 不一定是 15。

```java
void onCreate(SavedInstanceState state) {
    ....
    mRecyclerView.scrollToPosition(selectedPosition);
    mRecyclerView.setAdapter(myAdapter);
}
```

这样的代码其实并不会报错，因为 这里的处理是"异步的"。

### ViewHolder ++

```java
class ViewHolder {
    ...
    public bindTo(Item item, ImageLoader imageLoader) {
        title.setText(item.getTitle());
        body.setText(item.getBody());
        imageLoader.loadImage(icon, item.IconUrl());
    }
}

void onBindViewHolder(ViewHolder vh, int position) {
    vh.bindTo(items.get(position), mImageLoader);
}
```

这样写可能会更好，这样我们就能复用ViewHolder了。

### View Types

平时我们可能是使用某些常数来充当 Type，但其实我们也可以使用 layout 本身来充当，这样反而更加不容易冲突。

### Adapter position vs layout position

AP stands for Adapter position。

LP stands for Layout Position。

正常情况下是同步的，但有时候会是不同步的，比如我们的 Adapter 自己移动了位置，在下一次的 `onLayout`之前，LP 不一定等于 AP。

最好就是使用 AP。

### 一些错误用法

这段代码就有问题：

```java
public void onBindViewHolder(ViewHolder vh,
final int position) {
  // 每次new 一个对象不好，所以应该在 create 的时候进行绑定
  // 可以使用 getAdapterPosition
    vh.likeButton.setOnClickListener = new
OnClickListener() {
        // 这里的position 可能会引发问题，应该使用getAdapterPosition()
        items[position].liked = true;
        notifyItemChanged(position);
    }
}
```

如果 ViewHolder 没有正确地进行循环，那么会调用`onFail`方法，看看为什么会重用失败。



`setData`和`notifyDataSetChanged`应该都要在主线程中被调用，不然可能会影响 LayoutManager 的工作。

