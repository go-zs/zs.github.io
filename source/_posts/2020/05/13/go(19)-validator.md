---
title: golang系列(19)——Validator参数校验
date: 2020-05-13 10:19:20
tags:
---

http服务中参数校验是非常重要的一环，[go-playground/validator](https://github.com/go-playground/validator)是golang非常好用的开源库，它通过给`struct`添加`tag`的方式，自动完成参数的校验。

<!-- more -->

通常在`golang`中我们会为每个请求声明一个`struct`，在`json`序列化过程中我们是添加`json`这个`tag`实现字段之间的映射。同样的，我们使用`validator`这个库只需要添加`validate`这个`tag`，使用预定义的类型即可自动完成`struct`上的参数校验。

下面是一个简单的示例

```golang
type Student struct {
  Age int `validate:"required,max=60,min=18"`  // 18<=age<= 60
  Name string `validate:"required"`  // 非空
  ClassArr []string `validate:"required,min=1" // 长度至少为1
}
```


## 常见内置标识

`validator`内置了一些标识

### required标识

> // 要求字段非空
> Usage `required`
> // Field1或Field2存在时要求非空
> Usage `required_with=Field1 Field2`
> // Filed1和Field2都存在时要求非空
> Usage `required_with_all==Field1 Field2`
> // Field1或Field2不存在时要求非空
> Usage `required_without=Field1 Field2`
> // Field1和Field2都不存在时要求非空
> Usage `required_without_all=Field1 Field2`

### 

### Skip标识

> Usage `-`


### Or标识

> Usage `|`

### omitempty标识

允许字段不存在，此时如果字段不存在时不会进行校验，如果字段存在，`min`之类的校验会生效

> Usage `omitempty`

### Dive标识
这个字段会让`validator`校验`slice`或者`map`中的元素
> Usage `dive`

```golang
type Test struct {
  [][]string with validation tag "gt=0,dive,len=1,dive,required"
  // gt=0 will be applied to []
  // len=1 will be applied to []string
  // required will be applied to string
}
```
### 比较符

比较符号就只有以下这 6 种：

* `eq` 等于
* `ne` 不等于
* `gt` 大于
* `gte` 大于等于
* `lt` 小于
* `lte` 小于等于

### 跨字段验证

> Usage `eqfield=Field` 必须等于 Field 的值
> Usage `nefield=Field` 必须不等于 Field 的值
> Usage `gtfield=Field` 必须大于 Field 的值
> Usage `eqcsfield=Other.Field` 必须等于 struct Other 中 Field 的值；



### One Of

> Usage: oneof=5 7 9

### Unique

> Usage: unique  // for arrays, slices, and maps
> Usage: unique=field // for slices of structs

### Contains

> Usage: contains=@ 

### Excludes

> Usage: excludes=@
> Usage: excludesall=!@#?

### Starts With

> Usage: startswith=hello

### Ends With

> Usage: endswith=hello


### 字符串类型

> Usage: alpha  // ASCII alpha characters  only
> Usage: numeric  // 数字
> Usage: email   // email
> Usage: file       // File Path
> Usage: url      // must contain a schema eg: http://
> Usage: ip     
> Usage: ipv4     
> Usage: ipv6     
> Usage: tcp_addr     



