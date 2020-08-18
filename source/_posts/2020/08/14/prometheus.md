---
title: 使用prometheus对golang服务业务监测
date: 2020-08-14 10:51:46
tags:
---
对服务器性能指标的监控很多时候无法真正感知到服务的运行状况，往往我们需要对业务指标进行监控。
`Prometheus` 是一个开源的监测解决方案，原生的服务发现支持让它成为动态环境下进行服务监测的一个完美选择。`Prometheus` 支持从 AWS, Kubernetes, Consul 等 拉取服务 !
当使用 `Prometheus` 生成服务级别的指标时，有两个典型的方法：内嵌地运行在一个服务里并在 HTTP 服务器上开放一个 `/metrics` 路由，或者创建一个独立的http服务。

<!-- more -->

[golang prometheus client](https://github.com/prometheus/client_golang) 提供了一个用 Golang 写成的健壮的插桩库，可以用来注册，收集和暴露服务的指标。

prometheus支持下面4种暴露指标。

## Counter (计数器)

`Counter`是一个只增不减的计数器，觉的监控指标，如 `http_requests_total`,`node_cpu`都是`Counter`类型的监控指标。
PromQL内置的聚合操作和函数可以让用户对这些数据进行进一步的分析：

如通过`rate()`函数获取`http`请求量的增长率

```
rate(http_requests_total[5m])
```

如需要查询访问量前10的`http`地址

```
topk(10, http_requests_total)
```

## Gauge (可增减的仪表盘)

`Gauge`类型侧重于反应当前系统的状态，这类指标的样本数据可增可减。常见指标如：node_memory_MemFree（主机当前空闲的内容大小）、node_memory_MemAvailable（可用内存大小）都是Gauge类型的监控指标。

对于Gauge类型的监控指标，通过PromQL内置函数delta()可以获取样本在一段时间返回内的变化情况。例如，计算CPU温度在两个小时内的差异：

```
delta(cpu_temp_celsius{host="zeus"}[2h])
```

## 百分位数（quantile）

`Prometheus`中称为`quantile`，其实叫percentile更准确。百分位数是指小于某个特定数值的采样点达到一定的百分比。例如，假设`0.9-quantile`的值为120，意思就是所有的采样值中，小于120的采样值的数量占总体采样值的90%。相应的，假设`0.5-quantile`的值为x，那么意思就是指小于x的采样值占总体的50%，所以`0.5-quantile`也指中值（`median`）。

相对于简单的平均值来说，百分位数更丰富，更能反应出真实的用户体验。常用的百分位数为`0.5-quantile`，`0.9-quantile`以及`0.99-quantile`。这也是`Prometheus`默认的设置。

## Histogram（分布图)

histogram是柱状图，在Prometheus系统中的查询语言中，有三种作用：
* 对每个采样点进行统计（并不是一段时间的统计），打到各个桶(bucket)中
* 对每个采样点值累计和(sum)
* 对采样点的次数累计和(count)

每定义一个Histogram类型的metrics，实际也会生成几个metrics

```golang
metric := prometheus.NewHistogramVec(prometheus.HistogramOpts{
    Name: "http_request_duration_millisecond",
    Help:  “http request durations in millisecond.",
    Buckets: []float64,
    }, []string{“path”})
```

上面的Histogram会产生下面6类metrics。后面4个可以用于计算quantile值。
```
http_request_duration_millisecond_count
http_request_duration_millisecond_sum
http_request_duration_millisecond_bucket
http_request_duration_millisecond_bucket
http_request_duration_millisecond_bucket
http_request_duration_millisecond_bucket
```

## Summary (摘要)

每定义一个Summary类型的metrics，实际会生成几个metrics。

例如，下面的summary是用于监控http请求的响应时间

```golang
httpCallDurations = prometheus.NewSummaryVec(
prometheus.SummaryOpts{
    Name:       “http_request_duration_millisecond",
    Help:       “http request durations in millisecond.",
    Objectives: map[float64]float64,
}, []string{“path"})
```

上面的`summary`实际上生成的是5类`metrics`，后面3个就是百分位数值

```
http_request_duration_millisecond_count
http_request_duration_millisecond_sum
http_request_duration_millisecond
http_request_duration_millisecond
http_request_duration_millisecond
```