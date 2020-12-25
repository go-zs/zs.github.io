---
title: go(29)--高性能JSON解析器`jsoniter`
date: 2020-12-19 14:01:08
tags:
---

`jsoniter`是一个完全兼容官方`json`库的一个高性能`json`解析器。


<!-- more -->

## 基本的使用

使用上只需要替换调用的函数，其它跟标准库完全一致。


```golang
import "encoding/json"
json.Marshal(&data)
```
替换为

```golang
import jsoniter "github.com/json-iterator/go"

var json = jsoniter.ConfigCompatibleWithStandardLibrary
json.Marshal(&data)
```

```golang
import "encoding/json"
json.Unmarshal(input, &data)
```

替换为

```golang
import jsoniter "github.com/json-iterator/go"

var json = jsoniter.ConfigCompatibleWithStandardLibrary
json.Unmarshal(input, &data)
```

## 允许字符串与数字互转

```golang
import "github.com/json-iterator/go/extra"
extra.RegisterFuzzyDecoders()

var val string
jsoniter.UnmarshalFromString(`100`, &val)
```

## 私有字段

```golang
import "github.com/json-iterator/go/extra"
extra.SupportPrivateFields()
type TestObject struct {
    field1 string
}
obj := TestObject{}
jsoniter.UnmarshalFromString(`{"field1":"Hello"}`, &obj)
should.Equal("Hello", obj.field1)
```