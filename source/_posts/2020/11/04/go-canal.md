---
title: golang版Canal源码阅读
date: 2020-11-04 10:43:25
tags:
---

`Canal`是阿里开源的一款`Java`语言编写的中间件，主要用途是基于MySQL 数据库增量日志解析。[go-mysql](https://github.com/siddontang/go-mysql)是`golang`版本的实现。它不仅支持增量`binlog`消费，也支持全量数据的解析。

<!-- more -->

## 源码结构
```
├── canal    // 核心类
├── client   // 连接数据库的client
├── cmd      // 功能
│   ├── go-binlogparser
│   ├── go-canal
│   ├── go-mysqlbinlog
│   └── go-mysqldump
├── docker
├── driver   
├── dump     // 导出
├── failover
├── mysql
├── notes
├── packet
├── replication  // slave注册,binlog解析
├── schema       // schema解析
├── server
├── test_util
└── utils
```

## Canal类解析
