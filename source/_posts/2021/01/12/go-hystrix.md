---
title: go(31)--hystrix-go源码分析
date: 2021-01-12 13:23:43
tags:
- golang
categories:
- golang
---

`hystrix`是一个用来隔离远程系统调用、第三方库调用、服务调用、提供熔断机制、避免雪崩效应的库， `Hystrix`的`go`版本。 注`Hystrixs`是`Netflix`开源的一个java库。

<!-- more -->

## 基本使用

`Go`方法接受执行函数及异常函数。

```golang
import "github.com/afex/hystrix-go/hystrix"

hystrix.Go("my_command", func() error {
	// talk to other services
	return nil
}, func(err error) error {
	// do this when services are down
	return nil
})

```

## Go, GoC

`Go`和`GoC`函数差别是`GoC`多了一个待传入的上下文`ctx`参数，在实现上，`Go`也是用`context.Backgroud()`作为默认上下文来调用`GoC`。

```golang
func Go(name string, run runFunc, fallback fallbackFunc) chan error
func GoC(ctx context.Context, name string, run runFuncC, fallback fallbackFuncC) chan error 
```

重点看下`GoC`

```golang
// GoC runs your function while tracking the health of previous calls to it.
// If your function begins slowing down or failing repeatedly, we will block
// new calls to it for you to give the dependent service time to repair.
//
// Define a fallback function if you want to define some code to execute during outages.
func GoC(ctx context.Context, name string, run runFuncC, fallback fallbackFuncC) chan error {
  // 构造成command对象
	cmd := &command{
		run:      run,
		fallback: fallback,
		start:    time.Now(),
		errChan:  make(chan error, 1),
		finished: make(chan bool, 1),
	}

	circuit, _, err := GetCircuit(name)
	if err != nil {
		cmd.errChan <- err
		return cmd.errChan
	}
	cmd.circuit = circuit // 断路器
	ticketCond := sync.NewCond(cmd)
	ticketChecked := false

  // 返回令牌
	returnTicket := func() {
		cmd.Lock()
		// Avoid releasing before a ticket is acquired.
		for !ticketChecked {
			ticketCond.Wait()
		}
		cmd.circuit.executorPool.Return(cmd.ticket)
		cmd.Unlock()
	}
	// 保证只有先结束的goroutine返回结果
	returnOnce := &sync.Once{}
	reportAllEvent := func() {
		err := cmd.circuit.ReportEvent(cmd.events, cmd.start, cmd.runDuration)
		if err != nil {
			log.Printf(err.Error())
		}
	}

  // 断路 & 限流
	go func() {
		defer func() { cmd.finished <- true }()

		// Circuits get opened when recent executions have shown to have a high error rate.
		// Rejecting new executions allows backends to recover, and the circuit will allow
		// new traffic when it feels a healthly state has returned.
		if !cmd.circuit.AllowRequest() {
			cmd.Lock()
			// It's safe for another goroutine to go ahead releasing a nil ticket.
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ErrCircuitOpen)
				reportAllEvent()
			})
			return
		}

    // 后端服务响应慢时，降低后端的负载
    // 部分请求直接失败
		cmd.Lock()
		select {
		case cmd.ticket = <-circuit.executorPool.Tickets:
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
		default:
			ticketChecked = true
			ticketCond.Signal()
			cmd.Unlock()
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ErrMaxConcurrency)
				reportAllEvent()
			})
			return
		}

		runStart := time.Now()
		runErr := run(ctx)
		returnOnce.Do(func() {
			defer reportAllEvent()
			cmd.runDuration = time.Since(runStart)
			returnTicket()
			if runErr != nil {
				cmd.errorWithFallback(ctx, runErr)
				return
			}
			cmd.reportEvent("success")
		})
	}()

  // 超时退出
	go func() {
		timer := time.NewTimer(getSettings(name).Timeout)
		defer timer.Stop()

		select {
		case <-cmd.finished:
			// returnOnce has been executed in another goroutine
		case <-ctx.Done():
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ctx.Err())
				reportAllEvent()
			})
			return
		case <-timer.C:
			returnOnce.Do(func() {
				returnTicket()
				cmd.errorWithFallback(ctx, ErrTimeout)
				reportAllEvent()
			})
			return
		}
	}()

	return cmd.errChan
}

```

## Do, DoC

`DoC`是同步执行，函数执行完成或者执行出错之前都会堵塞程序执行。

```golang
// DoC runs your function in a synchronous manner, blocking until either your function succeeds
// or an error is returned, including hystrix circuit errors
func DoC(ctx context.Context, name string, run runFuncC, fallback fallbackFuncC) error {
	done := make(chan struct{}, 1)

	r := func(ctx context.Context) error {
		err := run(ctx)
		if err != nil {
			return err
		}

		done <- struct{}{}
		return nil
	}

	f := func(ctx context.Context, e error) error {
		err := fallback(ctx, e)
		if err != nil {
			return err
		}

		done <- struct{}{}
		return nil
	}

	var errChan chan error
	if fallback == nil {
		errChan = GoC(ctx, name, r, nil)
	} else {
		errChan = GoC(ctx, name, r, f)
	}

  // 堵塞函数直到执行完成或者返回error
	select {
	case <-done:
		return nil
	case err := <-errChan:
		return err
	}
}
```


## executorPool

`executorPool`是一个令牌桶的限流器。

```golang

type executorPool struct {
	Name    string
	Metrics *poolMetrics
	Max     int
	Tickets chan *struct{}
}

func newExecutorPool(name string) *executorPool {
	p := &executorPool{}
	p.Name = name
	p.Metrics = newPoolMetrics(name)
	p.Max = getSettings(name).MaxConcurrentRequests

  // 初始化p.Max个令牌
	p.Tickets = make(chan *struct{}, p.Max)
	for i := 0; i < p.Max; i++ {
		p.Tickets <- &struct{}{}
	}

	return p
}

// 归还令牌
func (p *executorPool) Return(ticket *struct{}) {
	if ticket == nil {
		return
	}

	p.Metrics.Updates <- poolMetricsUpdate{
		activeCount: p.ActiveCount(),
	}
	p.Tickets <- ticket
}

func (p *executorPool) ActiveCount() int {
	return p.Max - len(p.Tickets)
}
```

## CircuitBreaker

`CircuitBreaker`是一个断路器。

```golang
// CircuitBreaker is created for each ExecutorPool to track whether requests
// should be attempted, or rejected if the Health of the circuit is too low.
type CircuitBreaker struct {
	Name                   string
	open                   bool
	forceOpen              bool
	mutex                  *sync.RWMutex // 修改状态需要加锁
	openedOrLastTestedTime int64
 
	executorPool *executorPool    // 限流器
	metrics      *metricExchange  // 监控状态收集
}
```

看下断路的方法实现。

```golang
// 服务在执行之前都会调用AllowRequest校验状态
func (circuit *CircuitBreaker) AllowRequest() bool {
	return !circuit.IsOpen() || circuit.allowSingleTest()
}

// 断路器是否打开，True表示打开了，服务不可用
func (circuit *CircuitBreaker) IsOpen() bool {
	circuit.mutex.RLock()
	o := circuit.forceOpen || circuit.open
	circuit.mutex.RUnlock()

	if o {
		return true
	}

	if uint64(circuit.metrics.Requests().Sum(time.Now())) < getSettings(circuit.Name).RequestVolumeThreshold {
		return false
	}

	if !circuit.metrics.IsHealthy(time.Now()) {
		// 当前存在大量请求失败，直接断开
		circuit.setOpen()
		return true
	}

	return false
}

func (circuit *CircuitBreaker) allowSingleTest() bool {
	circuit.mutex.RLock()
	defer circuit.mutex.RUnlock()

	now := time.Now().UnixNano()
  openedOrLastTestedTime := atomic.LoadInt64(&circuit.openedOrLastTestedTime)

  // 每间隔一个SleepWindows的时间允许通过一个测试请求
	if circuit.open && now > openedOrLastTestedTime+getSettings(circuit.Name).SleepWindow.Nanoseconds() {
		swapped := atomic.CompareAndSwapInt64(&circuit.openedOrLastTestedTime, openedOrLastTestedTime, now)
		if swapped {
			log.Printf("hystrix-go: allowing single test to possibly close circuit %v", circuit.Name)
		}
		return swapped
	}

	return false
}
```

之前的代码我们可以看到每次处理结果都会上报，这个时候也会检查是否关闭断路器。

```golang
// ReportEvent records command metrics for tracking recent error rates and exposing data to the dashboard.
func (circuit *CircuitBreaker) ReportEvent(eventTypes []string, start time.Time, runDuration time.Duration) error {
	if len(eventTypes) == 0 {
		return fmt.Errorf("no event types sent for metrics")
	}

	circuit.mutex.RLock()
	o := circuit.open
	circuit.mutex.RUnlock()
	if eventTypes[0] == "success" && o {
		circuit.setClose()
	}

	var concurrencyInUse float64
	if circuit.executorPool.Max > 0 {
		concurrencyInUse = float64(circuit.executorPool.ActiveCount()) / float64(circuit.executorPool.Max)
	}
  
  // 记录结果
	select {
	case circuit.metrics.Updates <- &commandExecution{
		Types:            eventTypes,
		Start:            start,
		RunDuration:      runDuration,
		ConcurrencyInUse: concurrencyInUse,
	}:
	default:
		return CircuitError{Message: fmt.Sprintf("metrics channel (%v) is at capacity", circuit.Name)}
	}

	return nil
}
```
