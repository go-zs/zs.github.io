---
title: go(33)--框架beego源码阅读(2)
date: 2021-01-22 09:57:44
tags:
- golang
categories:
- golang
---

这期看下`beego`服务及路由相关的源码实现。

<!-- more -->

## Server

`Server`结构体定义如下：

```golang
type Server struct {
	*http.Server  // 继承标准库的http.Server
	ln           net.Listener
	SignalHooks  map[int]map[os.Signal][]func()
	sigChan      chan os.Signal
	isChild      bool
	state        uint8
	Network      string
	terminalChan chan error
}
```

`SignalHooks`允许自定义处理拦截的信号。

```golang
// RegisterSignalHook registers a function to be run PreSignal or PostSignal for a given signal.
func (srv *Server) RegisterSignalHook(ppFlag int, sig os.Signal, f func()) (err error) {
	if ppFlag != PreSignal && ppFlag != PostSignal {
		err = fmt.Errorf("Invalid ppFlag argument. Must be either grace.PreSignal or grace.PostSignal")
		return
	}
	for _, s := range hookableSignals {
		if s == sig {
			srv.SignalHooks[ppFlag][sig] = append(srv.SignalHooks[ppFlag][sig], f)
			return
		}
	}
	err = fmt.Errorf("Signal '%v' is not supported", sig)
	return
}
```

拦截系统信号，实现`graceful`退出

```golang
// handleSignals listens for os Signals and calls any hooked in function that the
// user had registered with the signal.
func (srv *Server) handleSignals() {
	var sig os.Signal

  // 监听相关信号
	signal.Notify(
		srv.sigChan,
		hookableSignals...,
	)

	pid := syscall.Getpid()
	for {
    sig = <-srv.sigChan
    // 执行退出之前处理操作
		srv.signalHooks(PreSignal, sig)
		switch sig {
		case syscall.SIGHUP:
			log.Println(pid, "Received SIGHUP. forking.")
			err := srv.fork()
			if err != nil {
				log.Println("Fork err:", err)
			}
		case syscall.SIGINT:
			log.Println(pid, "Received SIGINT.")
			srv.shutdown()
		case syscall.SIGTERM:
			log.Println(pid, "Received SIGTERM.")
			srv.shutdown()
		default:
			log.Printf("Received %v: nothing i care about...\n", sig)
    }
    // 执行退出后的操作
		srv.signalHooks(PostSignal, sig)
	}
}
```

## Router

路由是由`ControllerRegister`管理的。

```golang
// ControllerRegister containers registered router rules, controller handlers and filters.
type ControllerRegister struct {
	routers      map[string]*Tree 
	enablePolicy bool
	policies     map[string]*Tree
	enableFilter bool
	filters      [FinishRouter + 1][]*FilterRouter
	pool         sync.Pool

	// the filter created by FilterChain
	chainRoot *FilterRouter

	// keep registered chain and build it when serve http
	filterChains []filterChainConfig

	cfg *Config
}
```

定义路由

```golang
// Get add get method
// usage:
//    Get("/", func(ctx *context.Context){
//          ctx.Output.Body("hello world")
//    })
func (p *ControllerRegister) Get(pattern string, f FilterFunc) {
	(*web.ControllerRegister)(p).Get(pattern, func(ctx *context.Context) {
		f((*beecontext.Context)(ctx))
	})
}
```

路由也是用树实现的.

```golang
// Tree has three elements: FixRouter/wildcard/leaves
// fixRouter stores Fixed Router
// wildcard stores params
// leaves store the endpoint information
type Tree struct {
	// prefix set for static router
	prefix string
	// search fix route first
	fixrouters []*Tree
	// if set, failure to match fixrouters search then search wildcard
	wildcard *Tree
	// if set, failure to match wildcard search
	leaves []*leafInfo
}
```

添加路由的方法.

```golang
// AddTree will add tree to the exist Tree
// prefix should has no params
func (t *Tree) AddTree(prefix string, tree *Tree) {
	t.addtree(splitPath(prefix), tree, nil, "")
}

func (t *Tree) addtree(segments []string, tree *Tree, wildcards []string, reg string) {
	if len(segments) == 0 {
		panic("prefix should has path")
	}
	seg := segments[0]
	iswild, params, regexpStr := splitSegment(seg)
	// if it's ? meaning can igone this, so add one more rule for it
	if len(params) > 0 && params[0] == ":" {
		params = params[1:]
		if len(segments[1:]) > 0 {
			t.addtree(segments[1:], tree, append(wildcards, params...), reg)
		} else {
			filterTreeWithPrefix(tree, wildcards, reg)
		}
	}
	// Rule: /login/*/access match /login/2009/11/access
	// if already has *, and when loop the access, should as a regexpStr
	if !iswild && utils.InSlice(":splat", wildcards) {
		iswild = true
		regexpStr = seg
	}
	// Rule: /user/:id/*
	if seg == "*" && len(wildcards) > 0 && reg == "" {
		regexpStr = "(.+)"
	}
	if len(segments) == 1 {
		if iswild {
			if regexpStr != "" {
				if reg == "" {
					rr := ""
					for _, w := range wildcards {
						if w == ":splat" {
							rr = rr + "(.+)/"
						} else {
							rr = rr + "([^/]+)/"
						}
					}
					regexpStr = rr + regexpStr
				} else {
					regexpStr = "/" + regexpStr
				}
			} else if reg != "" {
				if seg == "*.*" {
					regexpStr = "([^.]+).(.+)"
				} else {
					for _, w := range params {
						if w == "." || w == ":" {
							continue
						}
						regexpStr = "([^/]+)/" + regexpStr
					}
				}
			}
			reg = strings.Trim(reg+"/"+regexpStr, "/")
			filterTreeWithPrefix(tree, append(wildcards, params...), reg)
			t.wildcard = tree
		} else {
			reg = strings.Trim(reg+"/"+regexpStr, "/")
			filterTreeWithPrefix(tree, append(wildcards, params...), reg)
			tree.prefix = seg
			t.fixrouters = append(t.fixrouters, tree)
		}
		return
	}

	if iswild {
		if t.wildcard == nil {
			t.wildcard = NewTree()
		}
		if regexpStr != "" {
			if reg == "" {
				rr := ""
				for _, w := range wildcards {
					if w == ":splat" {
						rr = rr + "(.+)/"
					} else {
						rr = rr + "([^/]+)/"
					}
				}
				regexpStr = rr + regexpStr
			} else {
				regexpStr = "/" + regexpStr
			}
		} else if reg != "" {
			if seg == "*.*" {
				regexpStr = "([^.]+).(.+)"
				params = params[1:]
			} else {
				for range params {
					regexpStr = "([^/]+)/" + regexpStr
				}
			}
		} else {
			if seg == "*.*" {
				params = params[1:]
			}
		}
		reg = strings.TrimRight(strings.TrimRight(reg, "/")+"/"+regexpStr, "/")
		t.wildcard.addtree(segments[1:], tree, append(wildcards, params...), reg)
	} else {
		subTree := NewTree()
		subTree.prefix = seg
		t.fixrouters = append(t.fixrouters, subTree)
		subTree.addtree(segments[1:], tree, append(wildcards, params...), reg)
	}
}
```
