---
title: go(27)--gin源码阅读(3)
date: 2020-12-19 14:19:28
tags:
---

学习一下`gin`是如何处理响应以及管理并发请求的。

<!-- more -->

## ServeHTTP

`ServeHTTP`方法实现了`Handler`接口。

请求进来的时候，首先从连接池里获取`Context`，然后调用`Context`的`reset`方法，在完成请求之后，调用`Put`方法，将`Context`归还到连接池中。

```golang
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```

`reset`方法主要是将相关变量恢复初始值,其中`c.Writer = &c.writermem`这行代码是将`Writer`赋值为`gin`中定义的结构体`writermem`。

```golang
func (w *responseWriter) reset(writer http.ResponseWriter) {
	w.ResponseWriter = writer // 更换为新请求传入的writer
	w.size = noWritten
	w.status = defaultStatus
}

func (c *Context) reset() {
	c.Writer = &c.writermem 
	c.Params = c.Params[0:0]
	c.handlers = nil
	c.index = -1

	c.fullPath = ""
	c.Keys = nil
	c.Errors = c.Errors[0:0]
	c.Accepted = nil
	c.queryCache = nil
	c.formCache = nil
	*c.params = (*c.params)[0:0]
}
```

我们需要知道`Context`中`Writer`和`writermem`的区别。

成员变量`Writer`类型是`ResponseWriter`这个`gin`中对`http`标准库相关接口的扩展出来的接口，`writermem`则是结构体`responseWriter`。

```golang
type Context struct {
	writermem responseWriter
	Writer    ResponseWriter
}
// ResponseWriter ...
type ResponseWriter interface {
	http.ResponseWriter
	http.Hijacker
	http.Flusher
	http.CloseNotifier

	// Returns the HTTP response status code of the current request.
	Status() int

	// Returns the number of bytes already written into the response http body.
	// See Written()
	Size() int

	// Writes the string into the response body.
	WriteString(string) (int, error)

	// Returns true if the response body was already written.
	Written() bool

	// Forces to write the http header (status code + headers).
	WriteHeaderNow()

	// get the http.Pusher for server push
	Pusher() http.Pusher
}

type responseWriter struct {
	http.ResponseWriter
	size   int
	status int
}
```


`responseWriter`内嵌了`http.ResponseWriter`，这样外部传入的`http.ResponseWriter`可以非常方便的扩展成结构体`responseWriter`，而`responseWriter`实现了`ResponseWriter`这个接口。


## Context对象池

`Context`是使用标准库`sync.Pool`进行管理的，使用对象池大大减少了GC的开销。

```golang
Engine struct {
    ...
    pool sync.Pool
    ...
}
```

`pool`的`New`方法是在`Engine`创建的时候定义的。

```golang
func New() *Engine {
	...
	engine.pool.New = func() interface{} {
		return engine.allocateContext()
	}

func (engine *Engine) allocateContext() *Context {
	v := make(Params, 0, engine.maxParams)
	return &Context{engine: engine, params: &v}
}
```
