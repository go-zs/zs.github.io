---
title: golang系列(36)--etcd源码阅读-raft
date: 2021-03-13 09:55:02
tags:
- golang
- etcd
- raft
categories:
- golang
---

`raft`是近年来最流行的一致性算法,好好研究一下`raft`在`etcd`中是如何实现的。

<!-- more -->

