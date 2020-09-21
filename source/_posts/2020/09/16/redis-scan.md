---
title: 源码分析redis scan
date: 2020-09-16 08:27:25
tags:
---

Redis的SCAN是一个基于游标的迭代器，Redis的底层数据结构是`Hash`，那么它是如何保证发生`rehash`的情况下也能完整的遍历。

<!-- more -->

## scan用法

由于`redis`的单线程的特性，`keys`会导致`redis`长时间的堵塞，所以生产环境是严禁使用`keys`命令的。

但是很多时候，我们确实需要筛选出符合条件的`key`，`scan`命令可以很好的解决这个问题。

```shell
scan cursor [MATCH pattern] [COUNT count]

eg: scan 0 match * count 100
```


## scan的遍历顺序

`Redis`的`key`也是用的`Hash`表作为底层实现，也就是数组+链表的数据结构。

`scan`其实就是对这个数组的遍历。

```
# 总共4个key
127.0.0.1:6379> keys *
1) "abc"
2) "qwe"
3) "jhi"
4) "def"

# 每次scan一个,返回的第一个参数为数组index
127.0.0.1:6379> scan 0 match * count 1
1) "2"
2) 1) "abc"
127.0.0.1:6379> scan 2 match * count 1
1) "1"
2) 1) "jhi"
127.0.0.1:6379> scan 1 match * count 1
1) "3"
2) 1) "qwe"
127.0.0.1:6379> scan 3 match * count 1
1) "0"
2) 1) "def"
```

它的遍历顺序是 `0->2->1->3`

顺序看起来很奇怪，看下源码的实现。

```C
// dick.c
unsigned long dictScan(dict *d,
                       unsigned long v,
                       dictScanFunction *fn,
                       dictScanBucketFunction* bucketfn,
                       void *privdata)
      ...
        m0 = t0->sizemask;
        // 将游标的umask位的bit都置为1
        v |= ~m0;
        /* Increment the reverse cursor */
        // 反转游标
        v = rev(v);
        // 反转后+1，达到高位加1的效果
        v++;
        // 再次反转复位
        v = rev(v);
      ...  
```

发现每次这个序列是高位加1的。普通二进制的加法，是从右往左相加、进位。而这个序列是从左往右相加、进位的。

对应二进制顺序其实是 `00->10->01->11`

我们看下`redis`这种高位进位的遍历方式遇到扩容或者缩容会如何处理。假设加入原始数组有4个元素，也就是索引有2位，这时候扩容会把它变成3位，并进行Rehash。

![scan](https://pic.hupai.pro/img/scan.png)

原来索引为`xx`的所有元素被重新分配到`0xx`和`1xx`索引上。

此时的遍历顺序为，`0->4->2->6->1->5->3->7`，也就是`000->100->010->110->001->101->011->111`

参考上图，当我们即将遍历`10`时，`dict`进行了扩容，这时，`scan`命令会从`010`开始遍历，而`000`和`100`（原`00`下挂接的元素）不会再被重复遍历,未被遍历的元素`01`和`11`也不会遗漏。

从原理上说，这种遍历方式保证了在扩容的时候，原来`dict`上的4个元素在新的`dict`上保持了原来的遍历顺序，因此不会重复并且不会遗漏。

再来看看缩容的情况。假设`dict`从3位缩容到2位，当即将遍历`110`时，`dict`发生了缩容，这时`scan`会遍历`10`。这时`010`上的元素会被重复遍历，但`010`之前的元素都不会被重复遍历了。所以，缩容时还是可能会有些重复元素出现的。

总结来说，`redis`里`rehash`扩容时，`scan`命令不会重复也不会遗漏。而缩容时，有可能会造成重复但不会遗漏。

