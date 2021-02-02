---
title: Python(3)——协程
date: 2021-02-01 17:32:04
tags:
- python
categories:
- python
---


`Python`协程已经出了很多个版本了，这篇好好总结一下协程的用法。

<!-- more -->

首先我们需要知道的是使用了协程，还是用不了多核，它解决了还是`IO`的问题。

## 初识

通过`async/await`关键字声明是官方推荐的协程使用方式。

看一个最简单的例子:

```python
import asyncio
import time

async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)

async def main():
    print(f"started at {time.strftime('%X')}")

    await say_after(1, 'hello')
    await say_after(2, 'world')

    print(f"finished at {time.strftime('%X')}")

if __name__ == '__main__':
    print(main())
    asyncio.run(main())

# output
# <coroutine object main at 0x103d69c40>
# started at 17:47:49
# /Users/zs/projects/yunzhanghu/python/demo/main.py:17: RuntimeWarning: coroutine 'main' was never awaited
#   print(main())
# RuntimeWarning: Enable tracemalloc to get the object allocation traceback
# hello
# world
# finished at 17:47:52
```

`main`函数前添加了关键字`async`，变成了`Coroutines`对象。因此`main`函数需要在`asyncio.run`中才能执行。


## Awaitables

`await`关键字后面只能跟`Awaitables object`，有三个主要的类型。

### Coroutines

`Coroutines`即协程，协程之间是可以相互`await`。

```python
import asyncio

async def nested():
    return 42

async def main():
    # Nothing happens if we just call "nested()".
    # A coroutine object is created but not awaited,
    # so it *won't run at all*.
    nested()

    # Let's do it differently now and await it:
    print(await nested())  # will print "42".

asyncio.run(main())
```


### Tasks

`Tasks`是用于`Coroutines`调度。

```python
import asyncio

async def nested():
    return 42

async def main():
    # Schedule nested() to run soon concurrently
    # with "main()".
    task = asyncio.create_task(nested())

    # "task" can now be used to cancel "nested()", or
    # can simply be awaited to wait until it is complete:
    await task

asyncio.run(main())
```

### Futures

 `Futures`代表的是协程任务的最终结果。

 通常协程相关的库及相关异步API都会以`Futures`对象的形式暴露给外界使用。

 ```python
 async def main():
    await function_that_returns_a_future_object()

    # this is also valid:
    await asyncio.gather(
        function_that_returns_a_future_object(),
        some_python_coroutine()
    )
```

## Sleep

通常我们使用协程的时候需要`Sleep`也只是希望当前协程`Sleep`，这个时候可以用`asyncio.sleep`这个API。


```python
import asyncio
import datetime

async def display_date():
    loop = asyncio.get_running_loop()
    end_time = loop.time() + 5.0
    while True:
        print(datetime.datetime.now())
        if (loop.time() + 1.0) >= end_time:
            break
        await asyncio.sleep(1)

asyncio.run(display_date())
```

## Running Tasks Concurrently

如果我们需要并发跑多个任务，可以用`asyncio.gather`，这个函数接收多个`Awaitables`对象。


```python
import asyncio

async def factorial(name, number):
    f = 1
    for i in range(2, number + 1):
        print(f"Task {name}: Compute factorial({i})...")
        await asyncio.sleep(1)
        f *= i
    print(f"Task {name}: factorial({number}) = {f}")

async def main():
    # Schedule three calls *concurrently*:
    await asyncio.gather(
        factorial("A", 2),
        factorial("B", 3),
        factorial("C", 4),
    )

asyncio.run(main())

# output:
#
#     Task A: Compute factorial(2)...
#     Task B: Compute factorial(2)...
#     Task C: Compute factorial(2)...
#     Task A: factorial(2) = 2
#     Task B: Compute factorial(3)...
#     Task C: Compute factorial(3)...
#     Task B: factorial(3) = 6
#     Task C: Compute factorial(4)...
#     Task C: factorial(4) = 24
```

## Timeouts

我们可以用`asyncio.wait_for`来给每个异步任务设置一个过期时间。

```python
async def eternity():
    # Sleep for one hour
    await asyncio.sleep(3600)
    print('yay!')

async def main():
    # Wait for at most 1 second
    try:
        await asyncio.wait_for(eternity(), timeout=1.0)
    except asyncio.TimeoutError:
        print('timeout!')

asyncio.run(main())

```

## cancel

`Task`是可以`cancel`的。

```python
async def cancel_me():
    print('cancel_me(): before sleep')

    try:
        # Wait for 1 hour
        await asyncio.sleep(3600)
    except asyncio.CancelledError:
        print('cancel_me(): cancel sleep')
        raise
    finally:
        print('cancel_me(): after sleep')

async def main():
    # Create a "cancel_me" Task
    task = asyncio.create_task(cancel_me())

    # Wait for 1 second
    await asyncio.sleep(1)

    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("main(): cancel_me is cancelled now")

asyncio.run(main())

#  output:
#
#     cancel_me(): before sleep
#     cancel_me(): cancel sleep
#     cancel_me(): after sleep
#     main(): cancel_me is cancelled now
```


`shield`可以保护`aswitable object`不被取消。当然它自身也是可能被取消的。

```python
# 保护任务不被取消
res = await shield(something())

# shield 也可能被取消
try:
    res = await shield(something())
except CancelledError:
    res = None
```