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

## ColScanner, Queryer, Execer

`sqlx`中定义了几种操作接口，`ColScanner`接口是读取结果，`Queryer`定义了查询行为，`Execer`则是执行`SQL`。

```golang
// ColScanner is an interface used by MapScan and SliceScan
type ColScanner interface {
	Columns() ([]string, error)
	Scan(dest ...interface{}) error
	Err() error
}

// Queryer is an interface used by Get and Select
type Queryer interface {
	Query(query string, args ...interface{}) (*sql.Rows, error)
	Queryx(query string, args ...interface{}) (*Rows, error)
	QueryRowx(query string, args ...interface{}) *Row
}

// Execer is an interface used by MustExec and LoadFile
type Execer interface {
	Exec(query string, args ...interface{}) (sql.Result, error)
}
```

`SliceScan`和`MapScan`方法接收`ColScanner`作为参数，实现了高扩展性。

```golang
// 序列到slice上
func SliceScan(r ColScanner) ([]interface{}, error) {
	// ignore r.started, since we needn't use reflect for anything.
	columns, err := r.Columns()
	if err != nil {
		return []interface{}{}, err
	}

	values := make([]interface{}, len(columns))
	for i := range values {
		values[i] = new(interface{})
	}

	err = r.Scan(values...)

	if err != nil {
		return values, err
	}

	for i := range columns {
		values[i] = *(values[i].(*interface{}))
	}

	return values, r.Err()
}

// 序列目标map上
func MapScan(r ColScanner, dest map[string]interface{}) error {
	// ignore r.started, since we needn't use reflect for anything.
	columns, err := r.Columns()
	if err != nil {
		return err
	}

	values := make([]interface{}, len(columns))
	for i := range values {
		values[i] = new(interface{})
	}

	err = r.Scan(values...)
	if err != nil {
		return err
	}

	for i, column := range columns {
		dest[column] = *(values[i].(*interface{}))
	}

	return r.Err()
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

## Rows

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

## Row

`Row`同样通过组合`sql.Rows`实现了`ColScanner`接口。

```golang
// Row is a reimplementation of sql.Row in order to gain access to the underlying
// sql.Rows.Columns() data, necessary for StructScan.
type Row struct {
	err    error
	unsafe bool
	rows   *sql.Rows
	Mapper *reflectx.Mapper
}

// 封装了sql.Scan
func (r *Row) Scan(dest ...interface{}) error {
	if r.err != nil {
		return r.err
	}

	defer r.rows.Close()
	for _, dp := range dest {
		if _, ok := dp.(*sql.RawBytes); ok {
			return errors.New("sql: RawBytes isn't allowed on Row.Scan")
		}
	}

	if !r.rows.Next() {
		if err := r.rows.Err(); err != nil {
			return err
		}
		return sql.ErrNoRows
	}
	// 扫瞄结果到dest上
	err := r.rows.Scan(dest...)
	if err != nil {
		return err
	}
	// Make sure the query can be processed to completion with no errors.
	if err := r.rows.Close(); err != nil {
		return err
	}
	return nil

```

## Tx

与`DB`相同，`Tx`对象同样扩展了`sql.Tx`，因此也通过继承的方式，实现了标准库中`stmtConnGrabber`和`Tx`这两个接口。

```golang
type Tx struct {
	*sql.Tx
	driverName string
	unsafe     bool
	Mapper     *reflectx.Mapper
}

func (tx *Tx) Select(dest interface{}, query string, args ...interface{}) error {
	return Select(tx, dest, query, args...)
}

// Get within a transaction.
// Any placeholder parameters are replaced with supplied args.
// An error is returned if the result set is empty.
func (tx *Tx) Get(dest interface{}, query string, args ...interface{}) error {
	return Get(tx, dest, query, args...)
}

```

## Stmt

`Stmt`同样被扩展了。

```golang
// Stmt is an sqlx wrapper around sql.Stmt with extra functionality
type Stmt struct {
	*sql.Stmt
	unsafe bool
	Mapper *reflectx.Mapper
}

func (s *Stmt) Select(dest interface{}, args ...interface{}) error {
	return Select(&qStmt{s}, dest, "", args...)
}

// Get using the prepared statement.
// Any placeholder parameters are replaced with supplied args.
// An error is returned if the result set is empty.
func (s *Stmt) Get(dest interface{}, args ...interface{}) error {
	return Get(&qStmt{s}, dest, "", args...)
}
```
