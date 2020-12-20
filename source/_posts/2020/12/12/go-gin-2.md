---
title: go(26)--gin源码阅读(2)
date: 2020-12-12 11:04:12
tags:
---

`Context`是`gin`框架中最重要的概念。

<!-- more -->

## 结构

`Context`是一个结构体，源码如下。

```golang
// Context is the most important part of gin. It allows us to pass variables between middleware,
// manage the flow, validate the JSON of a request and render a JSON response for example.
type Context struct {
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter

	Params   Params
	handlers HandlersChain
	index    int8
	fullPath string

	engine *Engine
	params *Params

	// This mutex protect Keys map
	mu sync.RWMutex
	// Keys is a key/value pair exclusively for the context of each request.
	Keys map[string]interface{}

	// Errors is a list of errors attached to all the handlers/middlewares who used this context.
	Errors errorMsgs

	// Accepted defines a list of manually accepted formats for content negotiation.
	Accepted []string

	// queryCache use url.ParseQuery cached the param query result from c.Request.URL.Query()
	queryCache url.Values

	// formCache use url.ParseQuery cached PostForm contains the parsed form data from POST, PATCH,
	// or PUT body parameters.
	formCache url.Values

	// SameSite allows a server to define a cookie attribute making it impossible for
	// the browser to send this cookie along with cross-site requests.
	sameSite http.SameSite
}
```

`gin.Context`实现了标准库中`Context`接口。

```golang
// Deadline always returns that there is no deadline (ok==false),
// maybe you want to use Request.Context().Deadline() instead.
func (c *Context) Deadline() (deadline time.Time, ok bool) {
	return
}

// Done always returns nil (chan which will wait forever),
// if you want to abort your work when the connection was closed
// you should use Request.Context().Done() instead.
func (c *Context) Done() <-chan struct{} {
	return nil
}

// Err always returns nil, maybe you want to use Request.Context().Err() instead.
func (c *Context) Err() error {
	return nil
}

// Value returns the value associated with this context for key, or nil
// if no value is associated with key. Successive calls to Value with
// the same key returns the same result.
func (c *Context) Value(key interface{}) interface{} {
	if key == 0 {
		return c.Request
	}
	if keyAsString, ok := key.(string); ok {
		val, _ := c.Get(keyAsString)
		return val
	}
	return nil
}
```


## Bind

`Context`支持非常多的数据绑定方式。

```golang
Bind(obj interface{}) error
BindJSON(obj interface{}) error
BindXML(obj interface{}) error
BindQuery(obj interface{}) error
BindYAML(obj interface{}) error
BindHeader(obj interface{}) error
BindUri(obj interface{}) error
MustBindWith(obj interface{}, b binding.Binding) error
ShouldBind(obj interface{}) error
ShouldBindJSON(obj interface{}) error
ShouldBindXML(obj interface{}) error
ShouldBindQuery(obj interface{}) error
ShouldBindYAML(obj interface{}) error
ShouldBindHeader(obj interface{}) error
ShouldBindUri(obj interface{}) error
ShouldBindWith(obj interface{}, b binding.Binding) error
ShouldBindBodyWith(obj interface{}, bb binding.BindingBody) (err error)
```

最常用的`Bind`方法支持自动通过请求方法和请求头，自动参数绑定。

```golang
func (c *Context) Bind(obj interface{}) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.MustBindWith(obj, b)
}

// Default returns the appropriate Binding instance based on the HTTP method
// and the content type.
func Default(method, contentType string) Binding {
	if method == http.MethodGet {
		return Form
	}

	switch contentType {
	case MIMEJSON:
		return JSON
	case MIMEXML, MIMEXML2:
		return XML
	case MIMEPROTOBUF:
		return ProtoBuf
	case MIMEMSGPACK, MIMEMSGPACK2:
		return MsgPack
	case MIMEYAML:
		return YAML
	case MIMEMultipartPOSTForm:
		return FormMultipart
	default: // case MIMEPOSTForm:
		return Form
	}
}
```

其它方法基本都是通过指定数据类型进行绑定。

```golang
// BindJSON is a shortcut for c.MustBindWith(obj, binding.JSON).
func (c *Context) BindJSON(obj interface{}) error {
	return c.MustBindWith(obj, binding.JSON)
}
```

### Binding

对于不同的数据类型处理，`gin`定义了`Binding`这个接口，这样扩展数据类型只需要实现`Binding`接口。

```golang
type Binding interface {
	Name() string
	Bind(*http.Request, interface{}) error
}
```

看下最常见的`json`的数据绑定的实现。

```golang
type jsonBinding struct{}

func (jsonBinding) Name() string {
	return "json"
}

func (jsonBinding) Bind(req *http.Request, obj interface{}) error {
	if req == nil || req.Body == nil {
		return fmt.Errorf("invalid request")
	}
	return decodeJSON(req.Body, obj)
}

func (jsonBinding) BindBody(body []byte, obj interface{}) error {
	return decodeJSON(bytes.NewReader(body), obj)
}

func decodeJSON(r io.Reader, obj interface{}) error {
	decoder := json.NewDecoder(r)
	if EnableDecoderUseNumber {
		decoder.UseNumber()
	}
	if EnableDecoderDisallowUnknownFields {
		decoder.DisallowUnknownFields()
	}
	if err := decoder.Decode(obj); err != nil {
		return err
	}
	return validate(obj)
}

```

通过`+build`方式条件编译实现了自定义的`json`解析方式。使用`go build main.go`的编译方式使用的是标准库`encoding/json`，而使用`go build -tags=jsoniter main.go`进行编译则会使用`json-iterator`来进行`json`的处理。

```golang
// gin/internal/json/json.go
// +build !jsoniter

package json

import "encoding/json"

var (
	// Marshal is exported by gin/json package.
	Marshal = json.Marshal
	// Unmarshal is exported by gin/json package.
	Unmarshal = json.Unmarshal
	// MarshalIndent is exported by gin/json package.
	MarshalIndent = json.MarshalIndent
	// NewDecoder is exported by gin/json package.
	NewDecoder = json.NewDecoder
	// NewEncoder is exported by gin/json package.
	NewEncoder = json.NewEncoder
)

// -----------------------------------

// gin/internal/json/jsonister.go
// +build jsoniter

package json

import jsoniter "github.com/json-iterator/go"

var (
	json = jsoniter.ConfigCompatibleWithStandardLibrary
	// Marshal is exported by gin/json package.
	Marshal = json.Marshal
	// Unmarshal is exported by gin/json package.
	Unmarshal = json.Unmarshal
	// MarshalIndent is exported by gin/json package.
	MarshalIndent = json.MarshalIndent
	// NewDecoder is exported by gin/json package.
	NewDecoder = json.NewDecoder
	// NewEncoder is exported by gin/json package.
	NewEncoder = json.NewEncoder
)


```


## Params, Query

`Params`获取可以获取`url`路径中的参数，`Query`则是可以解析`url`问号后面附带的请求参数。

```golang
// Param returns the value of the URL param.
// It is a shortcut for c.Params.ByName(key)
//     router.GET("/user/:id", func(c *gin.Context) {
//         // a GET request to /user/john
//         id := c.Param("id") // id == "john"
//     })
func (c *Context) Param(key string) string {
	return c.Params.ByName(key)
}

// Query returns the keyed url query value if it exists,
// otherwise it returns an empty string `("")`.
// It is shortcut for `c.Request.URL.Query().Get(key)`
//     GET /path?id=1234&name=Manu&value=
// 	   c.Query("id") == "1234"
// 	   c.Query("name") == "Manu"
// 	   c.Query("value") == ""
// 	   c.Query("wtf") == ""
func (c *Context) Query(key string) string {
	value, _ := c.GetQuery(key)
	return value
}
// 允许设置默认值
func (c *Context) DefaultQuery(key, defaultValue string) string {
	if value, ok := c.GetQuery(key); ok {
		return value
	}
	return defaultValue
}
```

## Keys

`Keys`提供了`Set`和`Get`这两个方法可以帮助我们传递线程安全的变量。

```golang
// Set is used to store a new key/value pair exclusively for this context.
// It also lazy initializes  c.Keys if it was not used previously.
func (c *Context) Set(key string, value interface{}) {
	c.mu.Lock()
	if c.Keys == nil {
		c.Keys = make(map[string]interface{})
	}

	c.Keys[key] = value
	c.mu.Unlock()
}

// Get returns the value for the given key, ie: (value, true).
// If the value does not exists it returns (nil, false)
func (c *Context) Get(key string) (value interface{}, exists bool) {
	c.mu.RLock()
	value, exists = c.Keys[key]
	c.mu.RUnlock()
	return
}
```

由于返回值是`interface{}`，`gin`也提供了很多API帮助我们从`map`中读取指定类型的数据。

```golang
GetString(key string) (s string)
GetBool(key string) (b bool)
GetInt(key string) (i int)
GetInt64(key string) (i64 int64)
GetUint(key string) (ui uint)
GetUint64(key string) (ui64 uint64)
GetFloat64(key string) (f64 float64)
GetTime(key string) (t time.Time)
GetDuration(key string) (d time.Duration)
GetStringSlice(key string) (ss []string)
GetStringMap(key string) (sm map[string]interface{})
GetStringMapString(key string) (sms map[string]string)
GetStringMapStringSlice(key string) (smss map[string][]string)
```

## Copy

`Context`并不是并发安全的，所以如果我们需要多个`goroutine`处理`Context`，需要用`Copy`方法复制一份。

```golang
// Copy returns a copy of the current context that can be safely used outside the request's scope.
// This has to be used when the context has to be passed to a goroutine.
func (c *Context) Copy() *Context {
	cp := Context{
		writermem: c.writermem,
		Request:   c.Request,
		Params:    c.Params,
		engine:    c.engine,
	}
	cp.writermem.ResponseWriter = nil
	cp.Writer = &cp.writermem
	cp.index = abortIndex
	cp.handlers = nil
	cp.Keys = map[string]interface{}{}
	for k, v := range c.Keys {
		cp.Keys[k] = v
	}
	paramCopy := make([]Param, len(cp.Params))
	copy(paramCopy, cp.Params)
	cp.Params = paramCopy
	return &cp
}
```