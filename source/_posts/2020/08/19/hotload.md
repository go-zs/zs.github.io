---
title: go(21)--golang服务热更新
date: 2020-08-19 15:05:36
tags:
---

线上服务经常需要更新，通常的做法是，通过前端的负载均衡（如nginx）来保证升级时至少有一个服务可用，依次（灰度）升级。
如果用到了k8s，则是先创建一个新的pod，待新pod启动成功后关闭旧的pod。这两种方式都会带来一个很短暂服务不可用。
还有一种更方便的方法是在应用上做热重启，直接更新源码、配置或升级应用而不停服务，这样便可提升服务可用性以及用户体验。

<!-- more -->


## 热重启原理
`nginx reload`采取的就是进程热更新的模式进行升级。
热重启的原理比较简单，但是涉及到一些系统调用以及父子进程之间文件句柄的传递等等细节比较多。
处理过程分为以下几个步骤：

1. 监听信号（USR2..）
2. 收到信号时fork子进程（使用相同的启动命令），将服务监听的socket文件描述符传递给子进程
3. 子进程监听父进程的socket，这个时候父进程和子进程都可以接收请求
4. 子进程启动成功之后，父进程停止接收新的连接，等待旧连接处理完成（或超时）
5. 父进程退出，重启完成


## golang热更新

server.Shutdown() 优雅关闭是go 1.8的特性。

> Shutdown gracefully shuts down the server without interrupting any active connections. Shutdown works by first closing all open listeners, then closing all idle connections, and then waiting indefinitely for connections to return to idle and then shut down. If the provided context expires before the shutdown is complete, Shutdown returns the context's error, otherwise it returns any error returned from closing the Server's underlying Listener(s).

```golang
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
)

func main() {
	var srv http.Server

	idleConnsClosed := make(chan struct{})
	go func() {
		sigint := make(chan os.Signal, 1)
		signal.Notify(sigint, os.Interrupt)
		<-sigint

		// We received an interrupt signal, shut down.
		if err := srv.Shutdown(context.Background()); err != nil {
			// Error from closing listeners, or context timeout:
			log.Printf("HTTP server Shutdown: %v", err)
		}
		close(idleConnsClosed)
	}()

	if err := srv.ListenAndServe(); err != http.ErrServerClosed {
		// Error starting or closing listener:
		log.Fatalf("HTTP server ListenAndServe: %v", err)
	}

	<-idleConnsClosed
}
```


demo

```golang
package main

import (
   "net"
   "net/http"
   "time"
   "log"
   "syscall"
   "os"
   "os/signal"
   "context"
   "fmt"
   "os/exec"
   "flag"
)
var (
   listener net.Listener
   err error
   server http.Server
   graceful =  flag.Bool("g", false, "listen on fd open 3 (internal use only)")
)

type MyHandler struct {

}

func (*MyHandler)ServeHTTP(w http.ResponseWriter, r *http.Request){
   fmt.Println("request start at ", time.Now(),  r.URL.Path+"?"+r.URL.RawQuery,  "request done at ", time.Now(), "  pid:", os.Getpid())
   time.Sleep(10 * time.Second)
   w.Write([]byte("this is test response"))
   fmt.Println("request done at ", time.Now(), "  pid:", os.Getpid() )

}

func main() {
   flag.Parse()
   fmt.Println("start-up at " , time.Now(), *graceful)
   if *graceful {
      f := os.NewFile(3, "")
      listener, err = net.FileListener(f)
      fmt.Printf( "graceful-reborn  %v %v  %#v \n", f.Fd(), f.Name(), listener)
   }else{
      listener, err = net.Listen("tcp", ":1111")
      tcp,_ := listener.(*net.TCPListener)
      fd,_ := tcp.File()
      fmt.Printf( "first-boot  %v %v %#v \n ", fd.Fd(),fd.Name(), listener)
   }


   server := http.Server{
      Handler: &MyHandler{},
      ReadTimeout: 6 * time.Second,
   }
   log.Printf("Actual pid is %d\n", syscall.Getpid())
   if err != nil {
      println(err)
      return
   }
   log.Printf(" listener: %v\n",   listener)
   go func(){//不要阻塞主进程
      err := server.Serve(listener)
      if err != nil {
         log.Println(err)
      }
   }()

   //signals
   func(){
      ch := make(chan os.Signal, 1)
      signal.Notify(ch, syscall.SIGHUP, syscall.SIGTERM)
      for{//阻塞主进程， 不停的监听系统信号
         sig := <- ch
         log.Printf("signal: %v", sig)
         ctx, _ := context.WithTimeout(context.Background(), 20*time.Second)
         switch sig {
         case syscall.SIGTERM, syscall.SIGHUP:
            println("signal cause reloading")
            signal.Stop(ch)
            {//fork new child process
               tl, ok := listener.(*net.TCPListener)
               if !ok {
                  fmt.Println("listener is not tcp listener")
                  return
               }
               currentFD, err := tl.File()
               if err != nil {
                  fmt.Println("acquiring listener file failed")
                  return
               }
               cmd := exec.Command(os.Args[0], "-g")
               cmd.ExtraFiles, cmd.Stdout,cmd.Stderr = []*os.File{currentFD} ,os.Stdout, os.Stderr
               err = cmd.Start()

               if err != nil {
                  fmt.Println("cmd.Start fail: ", err)
                  return
               }
               fmt.Println("forked new pid : ",cmd.Process.Pid)
            }

            server.Shutdown(ctx)
            fmt.Println("graceful shutdown at ", time.Now())
         }

      }
   }()
}
```


测试 `ab -v -k -c2 -n100 '127.0.0.1:1111/aa/bb?c=d'`