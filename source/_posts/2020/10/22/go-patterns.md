---
title: golang系列(23)--golang常用设计模式
date: 2020-10-22 20:33:28
tags:
- golang
- design patterns
categories: 
- golang
---

设计模式（Design pattern）代表了最佳的实践，通常面向对象语言会有很多常见的设计范式，`golang`作为一门不够彻底的面向对象编程语言，也有很多常见的设计范式。

<!-- more -->

设计模式总共23种，但很多设计模式对`golagn`并不完全适用，本文主要总结下`golang`中常用的一些设计模式。

下述代码源码可以在 [Golang常用设计模式](https://github.com/go-zs/go-patterns) 中获取。

## 选项模式

`golang`函数式选项模式是利用`golang`函数可变参数，很好的解决了缺少构造函数缺少默认值以及构造参数变更的问题。

通常用于`struct`的构造函数中。

```golang

package options

type (
	school struct {
		students int
		teachers int
	}

	SchoolOption interface {
		apply(*school)
	}
)

const (
	defaultStudents = 100
	defaultTeachers = 500
)

func NewSchool(options ...SchoolOption) *school {
	s := &school{
		students: defaultStudents,
		teachers: defaultTeachers,
	}
	for _, o := range options {
		o.apply(s)
	}

	return s
}

func SetStudents(num int) SchoolOption {
	return studentOption(num)
}

func SetTeachers(num int) SchoolOption {
	return teacherOption(num)
}

type studentOption int

func (o studentOption) apply(s *school) {
	s.students = int(o)
}

type teacherOption int

func (o teacherOption) apply(s *school) {
	s.teachers = int(o)
}

```

## 工厂模式

工厂方法模式（`Factory method pattern`）是一种实现了`工厂`概念的面向对象设计模式。

就像其他创建型模式一样，它也是处理在不指定对象具体类型的情况下创建对象的问题。

工厂方法模式的实质是*定义一个创建对象的接口，但让实现这个接口的类来决定实例化哪个类*。

```golang
package factory

import "fmt"

type (
	CarFactory struct{}

	ICar interface {
		Run()
	}

	Benz struct{}
	Bmw  struct{}

	CarType int
)

const (
	CarTypeBenz CarType = iota + 1
	CarTypeBmw
)

func NewCarFactory() *CarFactory {
	return &CarFactory{}
}

func (b Benz) Run() {
	fmt.Println("Benz run")
}

func (b Bmw) Run() {
	fmt.Println("Bmw run")
}

func (CarFactory) Build(tp CarType) ICar {
	switch tp {
	case CarTypeBenz:
		return Benz{}
	case CarTypeBmw:
		return Bmw{}
	}

	return nil
}

```

## 单例模式

单例模式是一种常用的软件设计模式，在它的核心结构中只包含一个实例的特殊类。

`golang`中可以用`sync.Once`来实现类的单次初始化，这种方式也可以很好的实现 *实例延迟加载*

```golang
package singleton

import "sync"

type Singleton struct {
	name string
}

var (
	once   sync.Once
	single *Singleton
)

func NewSingleton() *Singleton {
	once.Do(func() {
		single = &Singleton{}
	})

	return single
}


```

## 策略模式

策略模式定义了算法家族，在调用算法家族的时候不感知算法的变化，客户也不会受到影响。

例如用户在使用浏览器时，使用的搜索引擎可以自由选择。

```golang
package strategy

import "fmt"

type (
	ISearcher interface {
		Search() string
	}

	Baidu struct{}

	Google struct{}

	Sougou struct{}

	Chrome struct {
		searcher ISearcher
	}

	ChromeOption func(chrome *Chrome)
)

func (Google) Search() string {
	return "google"
}

func (Baidu) Search() string {
	return "baidu"
}

func (Sougou) Search() string {
	return "sougou"
}

var (
	defaultChromeSearcher = Google{}
)

func NewChrome(options ...ChromeOption) *Chrome {
	c := &Chrome{
		searcher: defaultChromeSearcher,
	}
	for _, o := range options {
		o(c)
	}
	return c
}

func (c *Chrome) SetSearcher(s ISearcher) {
	c.searcher = s
}

func (c *Chrome) Find() {
	fmt.Println(c.searcher.Search())
}

```


