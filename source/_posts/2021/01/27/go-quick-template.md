---
title: go(34)--quicktemplate源码分析 
date: 2021-01-27 19:53:25
tags:
- golang
categories:
- golang
---

平时工作生活中难免需要用到代码模板，对比标准库，`quicktemplate`最大优势就是模板被编译为单个二进制文件，因此无需将模板文件复制到服务器。

<!-- more -->

## quick start

安装

```shell
go get -u github.com/valyala/quicktemplate
go get -u github.com/valyala/quicktemplate/qtc
```

编写一个模板文件`hello.qtpl`,放在`templates`目录下

```
All text outside function templates is treated as comments,
i.e. it is just ignored by quicktemplate compiler (`qtc`). It is for humans.

Hello is a simple template function.
{% func Hello(name string) %}
	Hello, {%s name %}!
{% endfunc %}
```

运行`qtc`，`templates`目录下会生成一个`hello.qtpl.go`文件。

```golang
package templates

//line templates/hello.qtpl:5
import (
	qtio422016 "io"

	qt422016 "github.com/valyala/quicktemplate"
)

//line templates/hello.qtpl:5
var (
	_ = qtio422016.Copy
	_ = qt422016.AcquireByteBuffer
)

//line templates/hello.qtpl:5
func StreamHello(qw422016 *qt422016.Writer, name string) {
//line templates/hello.qtpl:5
	qw422016.N().S(`
	Hello, `)
//line templates/hello.qtpl:6
	qw422016.E().S(name)
//line templates/hello.qtpl:6
	qw422016.N().S(`!
`)
//line templates/hello.qtpl:7
}

//line templates/hello.qtpl:7
func WriteHello(qq422016 qtio422016.Writer, name string) {
//line templates/hello.qtpl:7
	qw422016 := qt422016.AcquireWriter(qq422016)
//line templates/hello.qtpl:7
	StreamHello(qw422016, name)
//line templates/hello.qtpl:7
	qt422016.ReleaseWriter(qw422016)
//line templates/hello.qtpl:7
}

//line templates/hello.qtpl:7
func Hello(name string) string {
//line templates/hello.qtpl:7
	qb422016 := qt422016.AcquireByteBuffer()
//line templates/hello.qtpl:7
	WriteHello(qb422016, name)
//line templates/hello.qtpl:7
	qs422016 := string(qb422016.B)
//line templates/hello.qtpl:7
	qt422016.ReleaseByteBuffer(qb422016)
//line templates/hello.qtpl:7
	return qs422016
//line templates/hello.qtpl:7
}
```

然后就可以直接调用`templates`包中的`Hello`方法了。

## QWriter

`QWriter`实现了`Writer`接口。

```golang
// QWriter is auxiliary writer used by Writer.
type QWriter struct {
	w   io.Writer
	err error
	b   []byte
}
```

`QWriter`通过`strconv`包实现了`float`、`int`等类型的高效转换写入，避免了大量的内存分配。

```golang
// F writes f to w.
func (w *QWriter) F(f float64) {
	n := int(f)
	if float64(n) == f {
		// Fast path - just int.
		w.D(n)
		return
	}

	// Slow path.
	w.FPrec(f, -1)
}
```

`float64`类型

```golang
// FPrec writes f to w using the given floating point precision.
func (w *QWriter) FPrec(f float64, prec int) {
	bb, ok := w.w.(*ByteBuffer)
	if ok {
		bb.B = strconv.AppendFloat(bb.B, f, 'f', prec, 64)
	} else {
		w.b = strconv.AppendFloat(w.b[:0], f, 'f', prec, 64)
		w.Write(w.b)
	}
}
```

`int`类型

```golang
// D writes n to w.
func (w *QWriter) D(n int) {
	bb, ok := w.w.(*ByteBuffer)
	if ok {
		bb.B = strconv.AppendInt(bb.B, int64(n), 10)
	} else {
		w.b = strconv.AppendInt(w.b[:0], int64(n), 10)
		w.Write(w.b)
	}
}
```

## string与[]byte

通过`unsafe`包实现了,`string`与`[]byte`零内存分配的转换。

```golang
func unsafeStrToBytes(s string) (b []byte) {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
	bh.Data = sh.Data
	bh.Len = sh.Len
	bh.Cap = sh.Len
	return b
}

func unsafeBytesToStr(z []byte) string {
	return *(*string)(unsafe.Pointer(&z))
}

```