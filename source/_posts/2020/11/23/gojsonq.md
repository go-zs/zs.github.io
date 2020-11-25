---
title: go(23)--golang json操作库gojsonq
date: 2020-11-23 17:31:15
tags:
---

最近做日志消费需要解析`json`数据，只需要取其中几个字段，但是原始数据确有很多层嵌套，用`golang`一层一层解析，操作起来异常繁琐。`gojsonq`可以极大简化操作`json`。

<!-- more -->

## 安装

[源码地址](https://github.com/thedevsaddam/gojsonq)

```shell
go get github.com/thedevsaddam/gojsonq/v2

```


## 代码示例

```golang
package main

import gojsonq "github.com/thedevsaddam/gojsonq/v2"

func main() {
	const json = `{"name":{"first":"Tom","last":"Hanks"},"age":61}`
	name := gojsonq.New().FromString(json).Find("name.first")
	println(name.(string)) // 输出： Tom
}

```


## 常用函数

### Find(path)

```golang
item := gojsonq.New().File("./data.json").Find("vendor.items.[0]");
fmt.Printf("%#v\n", item)
```

### FindR(path)

使用上和`Find`类似，返回的不是`inteface`而是一个`Result`对象

```golang
result, err := gojsonq.New().JSONString(jsonStr).FindR("name.first")
if err != nil {
	log.Fatal(err)
}
name, _ := result.String()
fmt.Printf("%#v\n", name)
```


### From(path)

```golang
jq := gojsonq.New().File("./data.json").From("items").Where("price", ">", 1200)
fmt.Printf("%#v\n", jq.Get())
```


### Select(properties)

```golang
jq := gojsonq.New().File("./data.json").From("items").Select("id", "name").WhereNotNil("id")
fmt.Printf("%#v\n", jq.Get())

// output
[]interface {}{
    map[string]interface {}{"id":1, "name":"MacBook Pro 13 inch retina"},
    map[string]interface {}{"id":2, "name":"MacBook Pro 15 inch retina"},
    map[string]interface {}{"id":3, "name":"Sony VAIO"},
    map[string]interface {}{"id":4, "name":"Fujitsu"},
}
```


## 源码分析

`JSONQ`对象中包含了所有操作函数。

```golang
type JSONQ struct {
	option           option               // contains options for JSONQ
	queryMap         map[string]QueryFunc // contains query functions
	node             string               // contains node name
	raw              json.RawMessage      // raw message from source (reader, string or file)
	rootJSONContent  interface{}          // original decoded json data
	jsonContent      interface{}          // copy of original decoded json data for further processing
	queryIndex       int                  // contains number of orWhere query call
	queries          [][]query            // nested queries
	attributes       []string             // select attributes that will be available in final resuls
	offsetRecords    int                  // number of records that will be skipped in final result
	limitRecords     int                  // number of records that will be available in final result
	distinctProperty string               // contain the distinct attribute name
	errors           []error              // contains all the errors when processing
}
```

构造函数支持指定`Decoder`，如果不传，默认使用标准库`encoding/json`进行解码。


```golang
func New(options ...OptionFunc) *JSONQ {
	jq := &JSONQ{
		queryMap: defaultQueries(),
		option: option{
			decoder:   &DefaultDecoder{},
			separator: defaultSeparator,
		},
	}
	for _, option := range options {
		if err := option(jq); err != nil {
			jq.addError(err)
		}
	}
	return jq
}

type OptionFunc func(*JSONQ) error

// WithDecoder take a custom decoder to decode JSON
func WithDecoder(u Decoder) OptionFunc {
	return func(j *JSONQ) error {
		if u == nil {
			return errors.New("decoder can not be nil")
		}
		j.option.decoder = u
		return nil
	}
}

// WithSeparator set custom separator for traversing child node, default separator is DOT (.)
func WithSeparator(s string) OptionFunc {
	return func(j *JSONQ) error {
		if s == "" {
			return errors.New("separator can not be empty")
		}
		j.option.separator = s
		return nil
	}
}

```

`getNestedValue`是最重要的解析函数。
* `input`是原始的`json`数据。
* `node`为传入的节点信息,如`user.name`
* `separator`为`node`中`key`的分隔符,如`.`


对于`map`的情况，一层层解析成`input.(map[string]interface{})`和`input.(map[string][]interface{})`将原始数据逐层解析出来。

对于`array`则用`isIndex`处理`json`中数组的情况，原始数据解析成`[]interface`。

```golang
func getNestedValue(input interface{}, node, separator string) (interface{}, error) {
	pp := strings.Split(node, separator)
	for _, n := range pp {
		if isIndex(n) {
			// find slice/array
			if arr, ok := input.([]interface{}); ok {
				indx, err := getIndex(n)
				if err != nil {
					return input, err
				}
				arrLen := len(arr)
				if arrLen == 0 ||
					indx > arrLen-1 {
					return empty, errors.New("empty array")
				}
				input = arr[indx]
			}
		} else {
			// map内是对象
			validNode := false
			if mp, ok := input.(map[string]interface{}); ok {
				input, ok = mp[n]
				validNode = ok
			}

			// map是同列表
			if mp, ok := input.(map[string][]interface{}); ok {
				input, ok = mp[n]
				validNode = ok
			}

			if !validNode {
				return empty, fmt.Errorf("invalid node name %s", n)
			}
		}
	}

	return input, nil
}
```