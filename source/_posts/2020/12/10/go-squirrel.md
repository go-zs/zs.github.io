---
title: golang系列(25)--SQL生成库squirrel
date: 2020-12-10 10:04:45
tags:
- golang
categories:
- golang
---

在`golang`中如果我们不用`ORM`框架的话，需要我们自己手动拼接`SQL`，`squirrel`是一个能够帮助我们简化这个过程的第三方库。

<!-- more -->


# 基本使用

`squirrel`不是`ORM`，它是通过表结构构造的方式完成`SQL`的拼接，通过`ToSql`方式，生成执行的`SQL`及对应的参数。

```golang

import sq "github.com/Masterminds/squirrel"

users := sq.Select("*").From("users").Join("emails USING (email_id)")

active := users.Where(sq.Eq{"deleted_at": nil})

sql, args, err := active.ToSql()

sql == "SELECT * FROM users JOIN emails USING (email_id) WHERE deleted_at IS NULL"

```

## Builder

`Builder`是实现增删改查的类

> squirrel.InsertBuilder 插入
> squirrel.UpdateBuilder 修改
> squirrel.DeleteBuilder 删除
> squirrel.SelectBuilder 查找

所有的`Builder`都实现了`Sqlizer`这个接口，所以它们都可以生成对应`SQL`语句。

```golang
type Sqlizer interface {
	ToSql() (string, []interface{}, error)
}
```


## squirrel.Expr

`squirrel.Expr`可以用于表达式，实现类似字段自增这样的功能。

```golang
squirrel.Update("money").Where(...).Set("value", squirrel.Expr(fmt.Sprintf("%s + %d", "value", 1))
```

## Prefix, Suffix

这两个方式可以在字段前后添加信息。

```golang
	nestedBuilder := StatementBuilder.PlaceholderFormat(Dollar).Select("*").Prefix("NOT EXISTS (").
		From("bar").Where("y = ?", 42).Suffix(")")
	outerSql, _, err := StatementBuilder.PlaceholderFormat(Dollar).Select("*").
		From("foo").Where("x = ?").Where(nestedBuilder).ToSql()
```

# 源码解析

## Builder

`InsertBuilder`这些`builder`是项目`github.com/lann/builder`中结构体`builder.Builder`的重定义。

```golang
"github.com/lann/builder"

type InsertBuilder builder.Builder
```

`builder`这个库[github主页](https://github.com/lann/builder)上是这样介绍的。

> Builder was originally written for Squirrel, a fluent SQL generator. It is probably the best example of Builder in action.
> Builder helps you write fluent DSLs for your libraries with method chaining:

```golang
resp := ReqBuilder.
    Url("http://golang.org").
    Header("User-Agent", "Builder").
		Get()
```

来看下`builder.Builder`的定义，参考注释说明，这个结构体主要用于存储一系列不可变的值。

```golang
// Builder stores a set of named values.
//
// New types can be declared with underlying type Builder and used with the
// functions in this package. See example.
//
// Instances of Builder should be treated as immutable. It is up to the
// implementor to ensure mutable values set on a Builder are not mutated while
// the Builder is in use.
type Builder struct {
	builderMap ps.Map  // ps.Map是下面定义的接口
}

type Map interface {
	// IsNil returns true if the Map is empty
	IsNil() bool

	// Set returns a new map in which key and value are associated.
	// If the key didn't exist before, it's created; otherwise, the
	// associated value is changed.
	// This operation is O(log N) in the number of keys.
	Set(key string, value Any) Map

	// Delete returns a new map with the association for key, if any, removed.
	// This operation is O(log N) in the number of keys.
	Delete(key string) Map

	// Lookup returns the value associated with a key, if any.  If the key
	// exists, the second return value is true; otherwise, false.
	// This operation is O(log N) in the number of keys.
	Lookup(key string) (Any, bool)

	// Size returns the number of key value pairs in the map.
	// This takes O(1) time.
	Size() int

	// ForEach executes a callback on each key value pair in the map.
	ForEach(f func(key string, val Any))

	// Keys returns a slice with all keys in this map.
	// This operation is O(N) in the number of keys.
	Keys() []string

	String() string
}
```

`builder.Builder`支持作为下列这些函数的参数调用

```golang
func Set(builder interface{}, name string, v interface{}) interface{}
func Delete(builder interface{}, name string) interface{}
func Append(builder interface{}, name string, vs ...interface{}) interface{}
func Extend(builder interface{}, name string, vs interface{}) interface{}
func Get(builder interface{}, name string) (interface{}, bool)
func GetMap(builder interface{}) map[string]interface{}
```

## ToSql实现

插入操作最后是由`insertData`这个结构实现的，`ToSql``主要实现的也是字符串的拼接。

```golang
type insertData struct {
	PlaceholderFormat PlaceholderFormat
	RunWith           BaseRunner
	Prefixes          []Sqlizer
	StatementKeyword  string
	Options           []string
	Into              string
	Columns           []string
	Values            [][]interface{}
	Suffixes          []Sqlizer
	Select            *SelectBuilder
}

func (d *insertData) ToSql() (sqlStr string, args []interface{}, err error) {
	if len(d.Into) == 0 {
		err = errors.New("insert statements must specify a table")
		return
	}
	if len(d.Values) == 0 && d.Select == nil {
		err = errors.New("insert statements must have at least one set of values or select clause")
		return
	}

	sql := &bytes.Buffer{}

	if len(d.Prefixes) > 0 {
		args, err = appendToSql(d.Prefixes, sql, " ", args)
		if err != nil {
			return
		}

		sql.WriteString(" ")
	}

	if d.StatementKeyword == "" {
		sql.WriteString("INSERT ")
	} else {
		sql.WriteString(d.StatementKeyword)
		sql.WriteString(" ")
	}

	if len(d.Options) > 0 {
		sql.WriteString(strings.Join(d.Options, " "))
		sql.WriteString(" ")
	}

	sql.WriteString("INTO ")
	sql.WriteString(d.Into)
	sql.WriteString(" ")

	if len(d.Columns) > 0 {
		sql.WriteString("(")
		sql.WriteString(strings.Join(d.Columns, ","))
		sql.WriteString(") ")
	}

	if d.Select != nil {
		args, err = d.appendSelectToSQL(sql, args)
	} else {
		args, err = d.appendValuesToSQL(sql, args)
	}
	if err != nil {
		return
	}

	if len(d.Suffixes) > 0 {
		sql.WriteString(" ")
		args, err = appendToSql(d.Suffixes, sql, " ", args)
		if err != nil {
			return
		}
	}

	sqlStr, err = d.PlaceholderFormat.ReplacePlaceholders(sql.String())
	return
}
```

