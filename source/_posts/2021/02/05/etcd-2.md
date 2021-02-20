---
title: golang系列(35)--etcd源码阅读-mvcc
date: 2021-02-05 14:01:52
tags:
- golang
- etcd
categories:
- golang
---

数据库通常运行在高并发环境下，通常解决冲突的办法有悲观锁和乐观锁，`MVCC（Multi-version Cocurrent Control 多版本并发控制)`是一个比较常见的乐观锁实现机制。`etcd`正是通过`mvcc`来实现事务管理中数据隔离。

<!-- more -->

## revision

`revision`就是`etcd`中的版本，每次`key-value`的操作都会有一个相应的 `revision`。

`revision`由`main`和`sub`两部分`int64`字段组成。

```golang
// A revision indicates modification of the key-value space.
// The set of changes that share same main revision changes the key-value space atomically.
type revision struct {
	// main is the main revision of a set of changes that happen atomically.
	main int64

	// sub is the sub revision of a change in a set of changes that happen
	// atomically. Each change has different increasing sub revision in that
	// set.
	sub int64
}
```

`revisions`是`revision`数组，实现了`sor.Sorter`接口。

```golang
type revisions []revision

func (a revisions) Len() int           { return len(a) }
func (a revisions) Less(i, j int) bool { return a[j].GreaterThan(a[i]) }
func (a revisions) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
```


## keyIndex

`keyIndex`存储了`key`的所有版本。

```golang
type keyIndex struct {
	key         []byte
	modified    revision // 最后的版本
	generations []generation // 历史版本
}
```

`keyIndex`实现了google的`btree`模块中的`Item`接口，因此它可以作为`B树`的节点。

```golang
// Item represents a single object in the tree.
type Item interface {
	// Less tests whether the current item is less than the given argument.
	//
	// This must provide a strict weak ordering.
	// If !a.Less(b) && !b.Less(a), we treat this to mean a == b (i.e. we can only
	// hold one of either a or b in the tree).
	Less(than Item) bool
}

func (a *keyIndex) Less(b btree.Item) bool {
	return bytes.Compare(a.key, b.(*keyIndex).key) == -1
}
```


## treeIndex

首先定义了`index`这个接口。

```golang
type index interface {
	Get(key []byte, atRev int64) (rev, created revision, ver int64, err error)
	Range(key, end []byte, atRev int64) ([][]byte, []revision)
	Revisions(key, end []byte, atRev int64, limit int) []revision
	CountRevisions(key, end []byte, atRev int64, limit int) int
	Put(key []byte, rev revision)
	Tombstone(key []byte, rev revision) error
	RangeSince(key, end []byte, rev int64) []revision
	Compact(rev int64) map[revision]struct{}
	Keep(rev int64) map[revision]struct{}
	Equal(b index) bool

	Insert(ki *keyIndex)
	KeyIndex(ki *keyIndex) *keyIndex
}
```

`treeIndex`是`index`接口的实现，从源码上看，它其实就是一棵`B树`，树的每个节点就是一个`keyIndex`。

```golang
type treeIndex struct {
	sync.RWMutex
	tree *btree.BTree
	lg   *zap.Logger
}
```

