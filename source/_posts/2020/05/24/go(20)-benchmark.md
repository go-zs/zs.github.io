---
title: golang系列(20)-benchmark
date: 2020-05-24 18:39:20
tags:
---

go自带的benchmark是一个性能测试的利器，利用这个工具，开发者可以方便快捷地在测试一个函数在串行或并发环境下的基准表现。

<!-- more -->

## benchmark常用API

* b.StopTimer()
* b.StartTimer()
* b.ResetTimer()
* b.Run(name string, f func(b *B))
* b.RunParallel(body func(*PB))
* b.ReportAllocs()
* b.SetParallelism(p int)
* b.SetBytes(n int64)
* testing.Benchmark(f func(b *B)) BenchmarkResult

## 串行测试

```golang
func BenchmarkFoo(b *testing.B) {
  for i:=0; i<b.N; i++ {
    dosomething()
  }
}

```

## 并发测试

```golang
func BenchmarkFoo(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			dosomething()
		}
	})
}


```