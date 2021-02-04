---
title: golang系列(30)--build参数全解析
date: 2020-12-25 14:42:05
tags:
- golang
categories:
- golang
---

`golang`编译速度非常快，平时一般也就用到`go build`，其实`build`功能异常的强大。

<!-- more -->


## 跨平台编译

`golang`可以在不同的平台之间交叉编译。如在`mac`下编译

```shell
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```


## 常用参数

下表的参数是`go build`经常使用的，其中`-race`参数可以检查出多个`goroutine`操作一个变量的竞争情况。

| 参数  | 说明                                        |
| ----- | ------------------------------------------- |
| -v    | 编译时显示包名                              |
| -p n  | 开启并发编译，默认情况下该值为 CPU 逻辑核数 |
| -a    | 强制重新构建                                |
| -n    | 强制重新构建                                |
| -x    | 打印编译时会用到的所有命令，但不真正执行    |
| -race | 开启竞态检测                                |


```golang
package main

import(
	"time"
	"fmt"
	"math/rand"
)

func main() {
	start := time.Now()
	var t *time.Timer
	t = time.AfterFunc(randomDuration(), func() {
		fmt.Println(time.Now().Sub(start))
		t.Reset(randomDuration())
	})
	time.Sleep(5 * time.Second)
}

func randomDuration() time.Duration {
	return time.Duration(rand.Int63n(1e9))
}
```

上面的代码直接运行，看起来似乎没问题，如果用`go run -race main.go`运行，就会出现`DATA RACE`的警告。

```golang
// output
952.879406ms
==================
WARNING: DATA RACE
Read at 0x00c00012a018 by goroutine 8:
  main.main.func1()
      /Users/zs/projects/main.go:14 +0x121

Previous write at 0x00c00012a018 by main goroutine:
  main.main()
      /Users/zs/projects/main.go:12 +0x18d

Goroutine 8 (running) created at:
  time.goFunc()
      /usr/local/Cellar/go/1.14/libexec/src/time/sleep.go:168 +0x51
==================
1.038131519s
1.709255072s
1.948771443s
2.241263754s
2.793600067s
3.431263086s
3.766549402s
3.954944541s
4.44053012s
Found 1 data race(s)

```


## 条件编译（Build tags）

我们可以在源码中插入特殊的注释——构建标签(`Build tags`)，当Go编译一个包时，它会分析包内的每个源码文件并查找构建标签。标签决定了这个源码文件是否被编译。

构建标签遵循以下规则：

* 每个源码允许存在多个构建标签
* 空格隔开的选项是或（`OR`）的关系
* 逗号隔开的选项是与（`AND`）的关系
* 每个选项由字母和数字组成。如果前面加上`!`，则表示反义
* 构建标签与`package`声明之间必须换行

我们以`gin`框架为例，它的源码中`json`模块使用了构建标签来实现替换`json`库。

`internal/json/jsonister.json`文件中声明了`+build jsoniter`，即指明`jsoniter`这个`tag`时编译此文件。`go build -tags jsoniter main.go`这样构建`gin`服务时，会使用`jsonister`作为框架的`json`模块。

```golang
// +build jsoniter

package json

import jsoniter "github.com/json-iterator/go"
```

`internal/json/json.json`文件中声明了`+build !jsoniter`，即指明在未指明`jsoniter`这个`tag`时编译，即默认使用标准库的`json`模块。

```golang
// +build !jsoniter

package json

import "encoding/json"
```


## 文件名后缀

如果我们的文件带有特定平台后缀，并且文件名非`.`和`_`开头时，`go build`会只在该平台下编译。

```golang
mypkg_freebsd_arm.go // 只在 freebsd/arm 系统编译
mypkg_plan9.go       // 只在 plan9 编译
```

## 编译参数注入

有时候我们需要在编译出来的`exe`中注入版本信息或者是构建时间，我们可以这样使用`-ldflags -X importpath.name=value`构建。

```golang
package main

import(
	"fmt"
)

var (
	commit = "ab"
	buildAt = "ab"
)

func main() {
	fmt.Println(commit)
	fmt.Println(buildAt)
}

```

我们使用如下命令构建。

```shell
// 注意 -ldflags后面的参数需要整体用双引号包裹，-X后面的参数用单引号包裹
go build -ldflags  "-X 'main.commit=$(git rev-parse --short HEAD)' -X 'main.buildAt=$(date +'%F %T %z')'" main.go
```

运行后输出：

```shell
// ./main
567b580
2020-12-25 15:42:29 +0800
```

## 查看GC分析

使用如下命令运行，可以看到`gc`的`debug`信息。

```shell
GODEBUG=gctrace=1 go run main.go
```


## 逃逸分析

逃逸分析也是非常常见的`debug`需求。

```shell
go build -gcflags=-m main.go
```

## 二进制发布

如果我们不想提供源码但是又想提供给别人调用，你只需要提供一个编译好的库，同时提供为这个package提供一个源文件。这个源文件不用包含任何代码逻辑，只需增加`//go:binary-only-package`指令即可(注意//和go:...之间不要加空格)。 这样用户在使用的时候，就可以直接使用你这个二进制的库了。

