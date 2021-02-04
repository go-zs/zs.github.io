---
title: golang系列(23)--golang标准库container
date: 2020-11-24 10:50:38
tags:
- golang
categories:
- golang
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

堆的基本操作，`Pop`和`Push`，`Pop`是弹出当前堆顶元素，`Push`是添加元素。操

```golang
// Push pushes the element x onto the heap.
// The complexity is O(log n) where n = h.Len().
func Push(h Interface, x interface{}) {
	h.Push(x)
	up(h, h.Len()-1)
}

// Pop removes and returns the minimum element (according to Less) from the heap.
// The complexity is O(log n) where n = h.Len().
// Pop is equivalent to Remove(h, 0).
func Pop(h Interface) interface{} {
	n := h.Len() - 1
	h.Swap(0, n)
	down(h, 0, n)
	return h.Pop()
}
```

通过使用堆构建优先队列，完成N个链表的合并，源码如下。

```golang
package main

import (
	"container/heap"
	"fmt"
)

type ListNode struct {
	Val  int
	Next *ListNode
}

type HeapInt []Node

type Node struct {
	Val   int
	Index int
}

func (h *HeapInt) Push(x interface{}) {
	*h = append(*h, x.(Node))
}

func (h *HeapInt) Pop() interface{} {
	r := (*h)[len(*h)-1]
	*h = (*h)[:len(*h)-1]
	return r
}

func (h HeapInt) Less(i, j int) bool { return h[i].Val < h[j].Val }
func (h HeapInt) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h HeapInt) Len() int           { return len(h) }

func mergeKLists(lists []*ListNode) *ListNode {
	r := &ListNode{}
	tmp := r
	h := &HeapInt{}

	for i, l := range lists {
		if l != nil {
			*h = append(*h, Node{Val: l.Val, Index: i})
		}
	}
	heap.Init(h)
	for h.Len() > 0 {
		v := heap.Pop(h)
		node := v.(Node)
		tmp.Next = lists[node.Index]
		tmp = tmp.Next
		lists[node.Index] = lists[node.Index].Next
		if lists[node.Index] != nil {
			heap.Push(h, Node{Val: lists[node.Index].Val, Index: node.Index})
		}
	}

	return r.Next
}

func main() {
	a1 := &ListNode{Val: 1}
	a2 := &ListNode{Val: 4}
	a3 := &ListNode{Val: 5}
	a1.Next = a2
	a2.Next = a3

	b1 := &ListNode{Val: 1}
	b2 := &ListNode{Val: 3}
	b3 := &ListNode{Val: 4}
	b1.Next = b2
	b2.Next = b3

	c1 := &ListNode{Val: 2}
	c2 := &ListNode{Val: 6}
	c1.Next = c2

	r := mergeKLists([]*ListNode{a1, b1, c1})

	for r != nil {
		fmt.Println(r.Val)
		r = r.Next
	}
}

```


## 双向链表

`golang`的`slice`非常好用，但是它底层是数组，数组的添加、删除元素的开销非常大，频繁增减元素的场景下，双向链表是更适合的数据结构。

`golang`中`container/list`的数据结构源码中，每个节点记录了前后元素指针及归属的`list`对象。

```golang
type Element struct {
	// Next and previous pointers in the doubly-linked list of elements.
	// To simplify the implementation, internally a list l is implemented
	// as a ring, such that &l.root is both the next element of the last
	// list element (l.Back()) and the previous element of the first list
	// element (l.Front()).
	next, prev *Element

	// The list to which this element belongs.
	list *List

	// The value stored with this element.
	Value interface{}
}
```


`list`使用也非常简单，示例如下。

```golang
package main

import (
	"container/list"
	"fmt"
)

func main() {
	// Create a new list and put some numbers in it.
	l := list.New()
	e4 := l.PushBack(4)
	e1 := l.PushFront(1)
	l.InsertBefore(3, e4)
	l.InsertAfter(2, e1)

	// Iterate through list and print its contents.
	for e := l.Front(); e != nil; e = e.Next() {
		fmt.Println(e.Value)
	}

}

```

通常实现一个`LRU`缓存，都需要用到`list`。

## 环

环也就是循环链表，它使用场景不多，但是也有很多独特的优势：

1) 任何节点都可以做为头节点。 可以从任何节点开始进行链表的遍历。只要当第一个节点被重复访问时，则意味着遍历结束。
2) 用于实现队列数据结构是很有帮组的。 如果使用循环链表，则不需要为了队列而维护两个指针(`front`以及`rear`)。只需要维护尾节点一个指针即可，因为尾节点的后向节点就是`front`了。

`golang`中`container/ring`也是通过双向链表实现,源码如下：

```golang
type Ring struct {
	next, prev *Ring
	Value      interface{} // for use by client; untouched by this library
}
```

* 约瑟夫问题

  **约瑟夫环（约瑟夫问题）是一个数学的应用问题**：已知n 个人（以编号1，2，3…n分别表示）围坐在一张圆桌周围。 从编号为k 的人开始报数，数到m 的那个人出圈；他的下一个人又从1 开始报数，数到m 的那个人又出圈；依此规律重复下去，直到剩余最后一个胜利者。 例如：有10个人围成一圈进行此游戏，每个人编号为1-10 。

```golang
package main

import (
	"container/ring"
	"fmt"
)

func solve(n, start, m int) {
	r := ring.New(n)
	for i := 0; i < n; i++ {
		r.Value = i + 1
		r = r.Next()
	}
	for i := 0; i < start-2; i++ {
		r = r.Next()
	}
	for {
		r = r.Move(m - 1)
		fmt.Println(r.Next().Value)
		if r.Len() == 1 {
			break
		}
		r.Unlink(1)
	}
}

func main() {
	solve(10, 1, 3)
}

```