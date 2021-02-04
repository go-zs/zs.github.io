---
title: golang系列(26)--gin源码阅读(1)
date: 2020-12-11 11:38:33
tags:
- golang
categories:
- golang
---


`gin`是`golang`里使用最为广泛的`Web`开发框架。

<!-- more -->

## IRouter

`gin`把路由定义成接口，看下它的定义。

```golang
// IRouter defines all router handle interface includes single and group router.
type IRouter interface {
	IRoutes
	Group(string, ...HandlerFunc) *RouterGroup
}

// IRoutes defines all router handle interface.
type IRoutes interface {
	Use(...HandlerFunc) IRoutes

	Handle(string, string, ...HandlerFunc) IRoutes
	Any(string, ...HandlerFunc) IRoutes
	GET(string, ...HandlerFunc) IRoutes
	POST(string, ...HandlerFunc) IRoutes
	DELETE(string, ...HandlerFunc) IRoutes
	PATCH(string, ...HandlerFunc) IRoutes
	PUT(string, ...HandlerFunc) IRoutes
	OPTIONS(string, ...HandlerFunc) IRoutes
	HEAD(string, ...HandlerFunc) IRoutes

	StaticFile(string, string) IRoutes
	Static(string, string) IRoutes
	StaticFS(string, http.FileSystem) IRoutes
}
```

内置的实现`IRouter`的结构体是`RouterGroup`，它是一系列相同前缀的路由的集合。

```golang
// RouterGroup is used internally to configure router, a RouterGroup is associated with
// a prefix and an array of handlers (middleware).
type RouterGroup struct {
	Handlers HandlersChain // 请求处理器
	basePath string        // 前缀
	engine   *Engine       
	root     bool          // 是否根节点
}

type HandlersChain []HandlerFunc
```

## HandlerFunc


`HandlerFunc`是一个非常重要的定义，用于处理请求，类似标准库`http`中定义的`HandlerFunc`。

```golang
// gin
type HandlerFunc func(*Context)

// http
type HandlerFunc func(ResponseWriter, *Request)
```

## Engine

`Engine`是`gin`中`HTTP`服务的实体，它包含了路由、中间件及服务相关配置。

```golang
type Engine struct {
	RouterGroup

  // 自动处理末尾反斜杠如 /foo/ -> /foo
	RedirectTrailingSlash bool
  
  // 自动处理路由中多余的项  /FOO and /..//Foo -> /foo
	RedirectFixedPath bool

  // method转换
  HandleMethodNotAllowed bool

  // 省略。。。

	pool             sync.Pool    // Context池
	trees            methodTrees  // 路由树
	maxParams        uint16       // poll最大上限
}
```

`Engine`实现了标准库`http`中的`Handler`接口，因此可以方便的同其它`Web`服务的组件集成。

```golang
// http
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
// gin
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)
  // 放入池子中
	engine.pool.Put(c)
}
```


`Engine`支持多种服务启动方式。

```golang
Run(addr ...string) (err error)
RunTLS(addr string, certFile string, keyFile string) (err error)
RunUnix(file string) (err error)
RunFd(fd int) (err error)
RunListener(listener net.Listener) (err error)
```

看下最常见的`Run`方法的源码，前面说到`Engine`实现了`Handler`接口，所以可以直接调用`http`的`ListenAndServe`方法启动服务。

```golang
func (engine *Engine) Run(addr ...string) (err error) {
	defer func() { debugPrintError(err) }()

	address := resolveAddress(addr)
	debugPrint("Listening and serving HTTP on %s\n", address)
	err = http.ListenAndServe(address, engine)
	return
}

// 两种参数方式
func resolveAddress(addr []string) string {
	switch len(addr) {
	case 0:
		if port := os.Getenv("PORT"); port != "" {
			debugPrint("Environment variable PORT=\"%s\"", port)
			return ":" + port
		}
		debugPrint("Environment variable PORT is undefined. Using port :8080 by default")
		return ":8080"
	case 1:
		return addr[0]
	default:
		panic("too many parameters")
	}
```


## RouteInfo

`RouteInfo`中存储了路由的基本信息。

```golang
// RoutesInfo defines a RouteInfo array.
type RoutesInfo []RouteInfo

// RouteInfo represents a request route's specification which contains method and path and its handler.
type RouteInfo struct {
	Method      string
	Path        string
	Handler     string
	HandlerFunc HandlerFunc
}
```

通过这个对象，可以获取服务中注册的所有路由。

```golang
// Routes returns a slice of registered routes, including some useful information, such as:
// the http method, path and the handler name.
func (engine *Engine) Routes() (routes RoutesInfo) {
	for _, tree := range engine.trees {
		routes = iterate("", tree.method, routes, tree.root)
	}
	return routes
}

func iterate(path, method string, routes RoutesInfo, root *node) RoutesInfo {
	path += root.path
	if len(root.handlers) > 0 {
		handlerFunc := root.handlers.Last()
		routes = append(routes, RouteInfo{
			Method:      method,
			Path:        path,
			Handler:     nameOfFunction(handlerFunc),
			HandlerFunc: handlerFunc,
		})
  }
  // 递归实现
	for _, child := range root.children {
		routes = iterate(path, method, routes, child)
	}
	return routes
}
```

这里又回到`engine`结构体中之前没有提到的`trees`这个`methodTrees`类型的成员变量，看下它的定义。

```golang
type methodTrees []methodTree

type methodTree struct {
	method string
	root   *node
}

type node struct {
	path      string
	indices   string
	wildChild bool
	nType     nodeType
	priority  uint32
	children  []*node
	handlers  HandlersChain
	fullPath  string
}
```

很明显，每种`method`都是一棵多叉树，`node`中`children`代表树的子节点。






