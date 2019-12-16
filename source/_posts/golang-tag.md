---
title: golang系列(2)——struct tags
date: 2019-12-16 07:48:40
tags:
---

golang tag的语法是非常具有语言特色的，当然对于新手来说，也是非常懵逼。今天来总结一下tags的用法。
对于tag,官方定义如下。简单说就是tag是字段可选的声明，除非通过反射获取，不然会被忽略。
> A field declaration may be followed by an optional string literal tag, which becomes an attribute for all the fields in the corresponding field declaration. The tags are made visible through a reflection interface but are otherwise ignored.



<!-- more -->


首先tag最常见的用法就是定义`json序列化的字段名`。
```golang
package main

import (
	"encoding/json"
	"fmt"
)

type Student struct {
	FirstName  string
	SecondName  string
	Age   int
	Grade int
}

func main() {
	s := Student{
		FirstName:  "Xiao",
		SecondName:  "Ming",
		Age:   18,
		Grade: 10,
	}
	j, _ := json.Marshal(s)
	fmt.Println(string(j))
}

// 输出： {"FirstName":"Xiao","SecondName":"Ming","Age":18,"Grade":10}


```
我们会发现这样序列化出来的字段名是驼峰的，而通常`json`更习惯于蛇形命名。我们可以通过添加字段tag实现这个。

```golang
type Student struct {
	FirstName  string `json:"first_name"`
	SecondName  string `json:"second_name"`
	Age   int `json:"age"`
	Grade int `json:"grade"`
}
// 输出： {"first_name":"Xiao","second_name":"Ming","age":18,"grade":10}

```

前面说到，tag可以用反射获取，参考`reflect`文档。

```golang
st := reflect.TypeOf(s)
for i:=0; i < st.NumField(); i++ {
    f := st.Field(i)
    t := f.Tag.Get("json")
    fmt.Println(t)
}
// 输出
// first_name
// second_name
// age
// grade
```

使用 `Tag.Get` 当字段不存在的时候，会返回空字符串，reflect包还提供了一个`Tag.Lookup`可以更好的判断这种情况。

```golang
type Student struct {
    FirstName  string `json:"first_name"`
    SecondName  string
    Age   int `json:"age"`
    Grade int `json:"grade"`
}

s := Student{
    FirstName:  "Xiao",
    SecondName:  "Ming",
    Age:   18,
    Grade: 10,
}

st := reflect.TypeOf(s)
for i:=0; i < st.NumField(); i++ {
    f := st.Field(i)
    if t, ok := f.Tag.Lookup("json"); ok {
        fmt.Println(t)
    } else {
        fmt.Println("no json tag")
    }
}
// 输出
// first_name
// no json tag
// age
// grade
```

我们也可以自定义一些字段，通过`key`访问到，从而实现一些逻辑处理。
```golang
type Student struct {
		FirstName  string `json:"first_name" print:"姓"`
		SecondName  string `json:"second_name" print:"名"`
		Age   int `json:"age" print:"年龄"`
		Grade int `json:"grade" print:"年级"`
	}

s := Student{
    FirstName:  "Xiao",
    SecondName:  "Ming",
    Age:   18,
    Grade: 10,
}

st := reflect.TypeOf(s)
vt := reflect.ValueOf(s)
for i:=0; i < st.NumField(); i++ {
    f := st.Field(i)
    v := vt.Field(i)
    t := f.Tag.Get("print")
    fmt.Printf("%s是%v\n", t, v)
}

// 输出
// 姓是Xiao
// 名是Ming
// 年龄是18
// 年级是10

```