---
layout: post
title: LruCache和Collections.sort()分析
date: 2017-04-08
categories: blog
tags: [编程]
description: 无
---

# LruCache和Collections.sort()分析

这里分析的是support包下面的`LruCache`和sdk23情况下的`Collections.sort()`

### LruCache

关于Lru算法，这里这不再赘述。

#### 数据结构

```java
private final LinkedHashMap<K, V> map;  //用一个LinkedHashMap来保存
/** Size of this cache in units. Not necessarily the number of elements. */
private int size; //现在的大小
private int maxSize;  //最大的大小，这里注意和DiskLruCache的区别，这里是int
//底下就是操作过程中的一些计数
private int putCount;  
private int createCount;
private int evictionCount;
private int hitCount;
private int missCount;
```

#### 使用

初始化函数如下所示，一般还要覆盖`sizeOf()`：

```java
public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
//默认实现
protected int sizeOf(K key, V value) {
    return 1;
}
```

首先下一个结论，这是一个线程安全的类，同时会自动去运用Lru算法去控制缓存，不仅如此，通过`resize()`还能灵活更改最大容量。

#### 原理

分析一下`put()`和`get()`：

```java
public final V put(K key, V value) {
  		//不允许空key和空v插入
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }
		//可能存在覆盖的情况
        V previous;
  		//加锁
        synchronized (this) {
            putCount++; 
            size += safeSizeOf(key, value); //这个safe表示不会返回一个负数
            previous = map.put(key, value); //调用hashmap的put方法
            if (previous != null) {
              	//如果出现覆盖情况，那么减去之前缓存的大小
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            //这里的让用户去管理逻辑，false表示是被覆盖而不是被lru清除
            entryRemoved(false, key, previous, value);
        }
		//尝试调用Lru算法
        trimToSize(maxSize);
        return previous;
}
//------------------------------
public final V get(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }
	//要返回的Value
    V mapValue;
    //加锁
    synchronized (this) {
        mapValue = map.get(key);
        if (mapValue != null) { //命中缓存
            hitCount++; 
            return mapValue;
        }
        missCount++;
    }

    /*
     * Attempt to create a value. This may take a long time, and the map
     * may be different when create() returns. If a conflicting value was
     * added to the map while create() was working, we leave that value in
     * the map and release the created value.
     */
    //当缓存未命中时，用户可以尝试创建一个Value
    V createdValue = create(key); //默认返回null
    if (createdValue == null) {
        return null;
    }
	//加锁
    synchronized (this) {
        createCount++; 
        
        mapValue = map.put(key, createdValue);
      
		//如果不为空明显说明出问题了，因为之前是因为这个key对应没有value
        //现在有了，说明在create的过程中已经put进去了
        if (mapValue != null) { 
            // There was a conflict so undo that last put
            //如果发生了冲突就撤销创建的动作
            map.put(key, mapValue);
        } else {
            size += safeSizeOf(key, createdValue);
        }
    }
	
    if (mapValue != null) {
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        //尝试使用Lru算法
        trimToSize(maxSize);
        return createdValue;
    }
}
```

可以看到明显是保证了每个实例的线程安全的，那么接下来就要看重点Lru算法`trimeToSize()`的实现了：

```java
 public void trimToSize(int maxSize) {
    while (true) { //无限循环直到满足大小限制
        K key;
        V value;
        synchronized (this) {
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(getClass().getName()
                        + ".sizeOf() is reporting inconsistent results!");
            }
			
            if (size <= maxSize || map.isEmpty()) {
                break;
            }
			//获得当前队列的头指针，也就是最早入列的那个对象
            Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);  //从map中移除
            size -= safeSizeOf(key, value);
            evictionCount++;
        }
		//true表示是被LRU移除列表的
        entryRemoved(true, key, value, null);
    }
}
```

关于`remove()`其实逻辑也非常简单，可以看出LRU的实现从代码层面上市很简单的。

### Collections.sort()分析

平时在排序的时候，简简单单就Collections.sort(list)完事，但从来没有想过内部怎么实现的，对待ArrayList和LinkedList又有什么区别也没有考虑。

```java
//这里很好地使用了泛型，必须是实现了Comparable的对象才可以排序
public static <T extends Comparable<? super T>> void sort(List<T> list) {
        // Note that we can't use instanceof here since ArrayList isn't final and
        // subclasses might make arbitrary use of array and modCount.
        //意思是只有ArrayList才可以调用if，而它的子类不会这样调用，
        //因为不确定子类会怎么使用这个内部数据结构
        if (list.getClass() == ArrayList.class) {
            ArrayList<T> arrayList = (ArrayList<T>) list;
            Object[] array = arrayList.array;
            int end = arrayList.size();
            Arrays.sort(array, 0, end);  //调用Arrays.sort
            arrayList.modCount++; 
        } else {
            Object[] array = list.toArray(); //首先获得内部数据结构，有的可能是链表实现，所以统一
            Arrays.sort(array);      //和上面其实一样
            int i = 0;               
            //把排序结果重新赋值给原来的List
            ListIterator<T> it = list.listIterator();
            while (it.hasNext()) {
                it.next();
                it.set((T) array[i++]);
            }
        }
    }
```

可以看到其实大家都是统一去调用了Arrays.sort()：

```java
public static void sort(Object[] array, int start, int end) {
        ComparableTimSort.sort(array, start, end); //调用ComparableTimSort算法
}
```

在jdk中其实是有`ComparableTimSort`和`TimSort`的，排序是稳定的，两者几乎没有差别。

如果要排序的数组长度小于32（规定）时，就不会调用归并排序，否则就会调用TimSort（一种优化的归并排序）

直接分析一下TimSort吧：

```java
private ComparableTimSort(Object[] a) {
    this.a = a;

    // Allocate temp storage (which may be increased later if necessary)
    int len = a.length;
    @SuppressWarnings({"unchecked", "UnnecessaryLocalVariable"})
  	//建立一个临时数组，如果要排序的数组长度小于512，则新建大小为排序数组的一般
    //否则为最大值256
    Object[] newArray = new Object[len < 2 * INITIAL_TMP_STORAGE_LENGTH ?
                                   len >>> 1 : INITIAL_TMP_STORAGE_LENGTH];
    tmp = newArray;

    /*
     * Allocate runs-to-be-merged stack (which cannot be expanded).  The
     * stack length requirements are described in listsort.txt.  The C
     * version always uses the same stack length (85), but this was
     * measured to be too expensive when sorting "mid-sized" arrays (e.g.,
     * 100 elements) in Java.  Therefore, we use smaller (but sufficiently
     * large) stack lengths for smaller arrays.  The "magic numbers" in the
     * computation below must be changed if MIN_MERGE is decreased.  See
     * the MIN_MERGE declaration above for more information.
     */
  	//分配存储run-to-be-merged的栈
    int stackLen = (len <    120  ?  5 :
                    len <   1542  ? 10 :
                    len < 119151  ? 19 : 40);
    runBase = new int[stackLen];
    runLen = new int[stackLen];
} 
//-----------------------------------------分割线-----------------
Arrays.checkStartAndEnd(a.length, lo, hi); //检查是否是恶意输入
int nRemaining  = hi - lo;
if (nRemaining < 2)
    return;  // Arrays of size 0 and 1 are always sorted

// If array is small, do a "mini-TimSort" with no merges
// 数组长度小于32，则不用TimSort
if (nRemaining < MIN_MERGE) {
    int initRunLen = countRunAndMakeAscending(a, lo, hi);
    binarySort(a, lo, hi, lo + initRunLen);
    return;
}
/**
 * March over the array once, left to right, finding natural runs,
 * extending short natural runs to minRun elements, and merging runs
 * to maintain stack invariant.
 * 从左到右遍历一遍数组，找到自然排好序的序列，把短的自然排序序列扩展到minRun长度的elements
 * 然后合并升序序列维持栈
 */
ComparableTimSort ts = new ComparableTimSort(a); //根据array新建
int minRun = minRunLength(nRemaining); //如果数组长度为2^n，则返回16；否则返回一个16-32之间的数
do {
    // Identify next run
    //从低位开始，找到顺序排列的最大值（无论是升序还是降序），lo-runLen这一段其实就不用排序了
    //顺序就不会管，降序就直接逆转
    int runLen = countRunAndMakeAscending(a, lo, hi);

    // If run is short, extend to min(minRun, nRemaining)
    //如果这段自然升序长度小于minRun，把升序后min(minRun,nRemaining)的长度进行排序
    if (runLen < minRun) {
        int force = nRemaining <= minRun ? nRemaining : minRun;
        //排序数组长度为force，由于lo——lo+runLen已经排序好，从lo+runLen开始排序
        //调用的是优化版本的二分插入排序，非常优雅
        binarySort(a, lo, lo + force, lo + runLen);
        runLen = force;
    }
	
    // Push run onto pending-run stack, and maybe merge
    //把已排好序的序列压到栈中，可能发生归并
  	//lo相当于起始下标，runLen就是已排序好的数组长度
    ts.pushRun(lo, runLen);
    ts.mergeCollapse(); //如果有超过1个栈时，且不满足某些条件就合并

    // Advance to find next run
    // 由于lo——(lo+runLen)已经排序好了，指针移动
    lo += runLen;
    //待排序长度
    nRemaining -= runLen;
} while (nRemaining != 0);

// Merge all remaining runs to complete sort
if (DEBUG) assert lo == hi;
//强制合并所有没有合并的序列
ts.mergeForceCollapse(); 
if (DEBUG) assert ts.stackSize == 1;
```
这里看一下相应的时间复杂度：

![img](http://img.my.csdn.net/uploads/201211/15/1352947294_8034.jpg)

空间复杂度：![img](http://img.my.csdn.net/uploads/201211/15/1352947419_3920.jpg)