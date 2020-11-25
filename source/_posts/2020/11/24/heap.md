---
title: go(23)--golang标准库container
date: 2020-11-24 10:50:38
tags:
---

`golang`标准库`container`实现了三个复杂的数据结构：堆，链表，环。在一些特定的场景下，这三个数据结构可以非常方便的帮我们解决问题。


<!-- more -->

## 堆

堆是一个非常实用的数据结构，可以非常方便的实现优先队列。

从定义上看，它在`sort.Interface`基础上增加了`Push`和`Pop`方法。

```golang
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}
```

构建一个堆

```golang
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
	res := (*h)[len(*h)-1]
	*h = (*h)[:len(*h)-1]
	return res
}
```

