---
title: golang系列(24)--逃逸分析
date: 2020-12-09 13:40:26
tags:
- golang
categories:
- golang
---

我们都知道`golang`是一门有`GC`的语言，并且频繁的`GC`会带来很大的性能开销。是不是所有生成对象都会产生`GC`呢？这个问题我们需要了解`golang`的逃逸分析。

<!-- more -->

## 堆栈

首先需要了解一下什么是`堆栈`。它数据结构里的`heap`和`stack`是一个东西吗？

这里说的`堆栈`是操作系统中的概念

> 栈：由编译器自动分配释放,存放函数的参数值，局部变量的值等。其操作方式类似于数据结构中的栈，栈使用的是一级缓存,他们通常都是被调用时处于存储空间中，调用完毕立即释放。我们可以理解为在函数调用分配的内存上的变量会随着调用函数一起被操作系统回收。

> 堆： 一般由程序员分配释放,若程序员不释放,程序结束时可能由系统回收，分配方式倒是类似于链表。堆则是存放在二级缓存中，生命周期由虚拟机的垃圾回收算法来决定（并不是一旦成为孤儿对象就能被回收）。所以调用这些对象的速度要相对来得低一些。

* 变量分配在栈上的好处：存取速度比堆要快，随函数调用结束释放无需`GC`


## 逃逸分析是什么

> 逃逸分析（Escape analysis）是指由编译器决定内存分配的位置，不需要程序员指定

在函数中创建一个新的对象：

* 如果分配在`栈`中，则函数执行结束可自动回收内存；
* 如果分配在`堆`中，则函数执行结束可交给`GC`处理;

不同于`JVM`语言，`golang`的逃逸分析也是编译期完成的。

可以在编译时添加`-gcflags=-m`开启逃逸分析的`debug`。

```shell
go build -gcflags=-m main.go
go build -gcflags="-m -l" main.go // -l 禁止内联编译
```

## 逃逸场景

### 1.返回指针

将指针作为返回值，一定会触发逃逸，很明显返回了指针，编译器会认为这个变量还会被使用。

```golang
package main

type (
	A struct {
		name string
	}
)

func T() *A {
	return &A{}
}

func main() {
	T()
}

// $ go build -gcflags="-m -l"  main.go
// # command-line-arguments
// ./m.go:10:9: &A literal escapes to heap
```

### 2.栈空间占用过大

我们创建一个`slice`的时候，编译器会根据栈空间判断是否在`栈`上分配。

```golang
package main

func S1() {
	s := make([]int, 1000)

	for index := range s {
		s[index] = index
	}
}

func S2() {
	s := make([]int, 10000)

	for index := range s {
		s[index] = index
	}
}

func main() {
	S1()
	S2()
}

// $ go build -gcflags="-m -l"  main.go
// # command-line-arguments
// ./m.go:4:11: make([]int, 1000) does not escape
// ./m.go:13:11: make([]int, 10000) escapes to heap
```

由于逃逸分析发生在编译期，如果我们指定一个变量作为`slice`的长度，它就一定会发生逃逸。

```golang
package main

func S(n int) {
	s := make([]int, n)

	for index := range s {
		s[index] = index
	}
}

func main() {
	S(0)
}

// $ go build -gcflags="-m -l"  main.go
// # command-line-arguments
// ./main.go:4:11: make([]int, n) escapes to heap
```

### 3.对象被引用

我们对一个对象取地址了，很多情况都会导致这个对象在函数外被访问到，所以这个对象会逃逸到堆上。

```golang
package main

type (
	A struct {
		a *int
	}
)

func F() A {
	var b int
	a := A{a: &b}
	return a
}

func main() {
	F()
}

// $ go build -gcflags="-m -l"  main.go
// # command-line-arguments
// ./main.go:10:6: moved to heap: b
```


### 4.闭包引用对象逃逸

虽然没有直接对对象取地址进行引用，但是闭包本质上就是允许函数内部变量被外部函数访问，所以它也必须分配到堆上。

```golang
package main

import "fmt"

func Fibonacci() func() int {
	a, b := 0, 1
	return func() int {
		a, b = b, a+b
		return a
	}
}

func main() {
	f := Fibonacci()
	for i := 0; i < 10; i++ {
		fmt.Printf("Fibonacci: %d\n", f())
	}
}

// $ go build -gcflags="-m -l"  main.go
// # command-line-arguments
// ./main.go:6:2: moved to heap: a
// ./main.go:6:5: moved to heap: b
// ./main.go:7:9: func literal escapes to heap
// ./main.go:17:13: ... argument does not escape
// ./main.go:17:34: f() escapes to heap
```

## 官网说明

关于分配`golang`的官方说明如下：

> From a correctness standpoint, you don’t need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.
> 
> The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function’s stack frame.
> 
> However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
> 
> In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.