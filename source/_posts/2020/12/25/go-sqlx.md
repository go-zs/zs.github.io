---
title: go(30)--sqlx源码分析
date: 2020-12-25 16:52:02
tags:
---

`sqlx`是标准库`sql`最强的扩展，大大简化了`sql`操作。

<!-- more -->

## sqlx.DB

首先它扩展了标准库中的`sql.DB`对象，源码定义如下。

```golang
// DB is a wrapper around sql.DB which keeps track of the driverName upon Open,
// used mostly to automatically bind named queries using the right bindvars.
type DB struct {
	*sql.DB
	driverName string  // 驱动 如:mysql
	unsafe     bool
	Mapper     *reflectx.Mapper // 字段映射
}

```


## Get, Select

标准库`sql`获取查询数据，需要遍历`rows`，`sqlx`提供了`Get`和`Select`两个方法，可以将查询结果直接映射到传入的结构体上。

```golang
// Select using this DB.
// Any placeholder parameters are replaced with supplied args.
func (db *DB) Select(dest interface{}, query string, args ...interface{}) error {
	return Select(db, dest, query, args...)
}

func (db *DB) Queryx(query string, args ...interface{}) (*Rows, error) {
	r, err := db.DB.Query(query, args...)
	if err != nil {
		return nil, err
	}
	return &Rows{Rows: r, unsafe: db.unsafe, Mapper: db.Mapper}, err
}

func Select(q Queryer, dest interface{}, query string, args ...interface{}) error {
	rows, err := q.Queryx(query, args...)
	if err != nil {
		return err
  }
	// if something happens here, we want to make sure the rows are Closed
	defer rows.Close()
	return scanAll(rows, dest, false)
}
```

扩展的`Queryx`方法，基于标准库查询，返回`sqlx.Rows`对象，这个`sqlx.Rows`也是对标准库的中`sql.Rows`的扩展。

```golang
// Rows is a wrapper around sql.Rows which caches costly reflect operations
// during a looped StructScan
type Rows struct {
	*sql.Rows
	unsafe bool
	Mapper *reflectx.Mapper
	// these fields cache memory use for a rows during iteration w/ structScan
	started bool
	fields  [][]int
	values  []interface{}
}
```