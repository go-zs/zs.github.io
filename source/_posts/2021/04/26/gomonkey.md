---
title: gomonkey
date: 2021-04-26 11:00:50
tags:
- golang
- unittest
---

`gomonkey`是通过打桩的方式修改函数、方法或者全局变量，来完成单元测试的`mock`。

<!-- more -->

## 安装

```shell
go get github.com/agiledragon/gomonkey/v2
```

## 原理打桩

运行时打桩是对内存的应用，我们知道程序的函数是在代码段中存储，一个函数的操作对应一个栈帧的存储地址，如果在调用函数时，在一旦访问这个栈帧，我们就使它跳转到我们需要的桩函数去，那么也就实现了函数的打桩。 这种方法要复杂一点，但是不需要对原有的代码进行修改，而是额外增加了打桩和还原的操作，在进行单元测试时也常用。


## Mock函数

很多时候我们需要获得当前时间戳，我们有`GetCurrentTime`这样一个函数，我们要`mock`单元测试，怎么控制这个函数的输出呢？比较方便的做法就是给获取时间戳的函数打桩。

```golang
// m.go
package monkey

import "time"

func GetCurrentTime() int64 {
	return time.Now().Unix()
}
```

下面是`gomonkey`的单元测试


```golang
package monkey

import (
	"testing"

	"github.com/agiledragon/gomonkey/v2"
	"github.com/stretchr/testify/require"
)

func TestGetTime(t *testing.T) {
	p := gomonkey.ApplyFunc(GetCurrentTime, func() int64 { return 15 })
	defer p.Reset()

	require.EqualValues(t, 15, GetCurrentTime())
}
```

## Mock全局变量

`const`是不可取地址的，所以需要用变量定义。

```golang
package monkey

import (
	"testing"

	"github.com/agiledragon/gomonkey/v2"
	"github.com/stretchr/testify/require"
)

var (
	A = 15
)

func TestGlobal(t *testing.T) {
	t.Run("normal", func(t *testing.T) {
		p := gomonkey.ApplyGlobalVar(&A, 150)
		defer p.Reset()
		require.Equal(t, A, 150)
	})
	t.Run("normal", func(t *testing.T) {
		require.Equal(t, A, 15)
	})
}
```

## Mock方法

编写下面的示例

```golang
package monkey

import (
	"fmt"
	"reflect"
	"testing"

	"github.com/agiledragon/gomonkey/v2"
	"github.com/stretchr/testify/require"
)

type (
	B struct{}
)

func (b *B) Pop() int {
	return 100
}

func TestGlobal(t *testing.T) {
	var b *B
	p := gomonkey.ApplyMethod(reflect.TypeOf(b), "Pop", func(_ *B) int { return 0 })
	defer p.Reset()
	fmt.Println(b)
	require.Equal(t, 0, b.Pop())
}
```

运行`go test -v ./...`，我们会发现测试失效了。

```golang
$ go test -v ./...             
=== RUN   TestGlobal
<nil>
    g_test.go:25: 
                Error Trace:    g_test.go:25
                Error:          Not equal: 
                                expected: 0
                                actual  : 100
                Test:           TestGlobal
--- FAIL: TestGlobal (0.00s)
=== RUN   TestGetTime
--- PASS: TestGetTime (0.00s)
FAIL
FAIL    monkey  0.495s
FAIL
```

原因是`gomonkey`无法`mock`内联函数，需要禁用内联命令.

执行`go test -v -gcflags=all=-l  ./...`，测试通过了。

```
$ go test -v -gcflags=all=-l  ./...
=== RUN   TestGlobal
<nil>
--- PASS: TestGlobal (0.00s)
=== RUN   TestGetTime
--- PASS: TestGetTime (0.00s)
PASS
ok      monkey  0.762s

```

由于`method mock`使用的是反射，而私有方法是无法通过反射找到的，所以私有方法无法`mock`

