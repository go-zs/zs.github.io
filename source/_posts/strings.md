---
title: golang系列(2)——strings包
date: 2019-09-09 08:39:40
tags:
---

golang的官方库strings库非常的强大，用了很久感觉需要整理一下相关api才能更好的掌握。


<!-- more -->

## 中文处理

首先，需要分清楚 `rune`和`byte`, 查看源码，我们可以知道，`byte`其实是单字节，而`rune`是4个字节，很明显，`rune`其实就是`unicode`码。

```golang
type byte uint8
type rune int32
```

我们遍历一个中文字符串看看会是什么结果输出。

```golang
func main() {
	cnStr := "中文abc"
    for _, x := range cnStr {
		fmt.Println(x, " ", string(x))
	}
}
// 输出
// 20013   中
// 25991   文
// 97   a
// 98   b
// 99   c

```

这个遍历结果跟将`string`转成`[]rune`遍历结果一致。

```golang
func main() {
	cnStr := "中文abc"
    for _, x := range []rune(cnStr) {
		fmt.Println(x, " ", string(x))
	}
}
// 输出
// 20013   中
// 25991   文
// 97   a
// 98   b
// 99   c

```


然后我们把这个字符串按`[]byte`遍历看看。不出所料出现了乱码。

```golang
func main() {
	cnStr := "中文abc"
	for _, x := range []byte(cnStr) {
		fmt.Println(x, " ", string(x))
	}
}
// 输出
// 228   ä
// 184   ¸
// 173   ­
// 230   æ
// 150   
// 135   
// 97   a
// 98   b
// 99   c
```

## 子串检查
```
// 子串 substr 在 s 中，返回 true
func Contains(s, substr string) bool
// chars 中任何一个 Unicode 代码点在 s 中，返回 true
func ContainsAny(s, chars string) bool
// Unicode 代码点 r 在 s 中，返回 true
func ContainsRune(s string, r rune) bool
```

需要注意的是空字符串, `strings.Contains`和`strings.ContainsAny`判断结果不一样。

```shell
b1 := strings.Contains("abc", "a")
b2 := strings.Contains("abc", "")
b3 := strings.ContainsAny("abc", "a")
b4 := strings.ContainsAny("abc", "")
b5 := strings.ContainsRune("中国", '中')
fmt.Println("b1 =", b1)
fmt.Println("b2 =", b2)
fmt.Println("b3 =", b3)
fmt.Println("b4 =", b4)
fmt.Println("b5 =", b5)
# 输出
b1 = true
b2 = true
b3 = true
b4 = false
b5 = true
```

## 字符串分割
字符串分割应该是最常见的使用了，strings包提供了这6个方法：Fields 和 FieldsFunc、Split 和 SplitAfter、SplitN 和 SplitAfterN。

```
func Fields(s string) []string
func FieldsFunc(s string, f func(rune) bool) []string
func Split(s, sep string) []string { return genSplit(s, sep, 0, -1) }
func SplitAfter(s, sep string) []string { return genSplit(s, sep, len(sep), -1) }
func SplitN(s, sep string, n int) []string { return genSplit(s, sep, 0, n) }
func SplitAfterN(s, sep string, n int) []string { return genSplit(s, sep, len(sep), n) }
```

方法使用示例如下

```
s1 := "abc def hig lmn pgo"
fmt.Printf("%q\n", strings.Fields(s1))
fmt.Printf("%q\n", strings.FieldsFunc(s1, func(r rune) bool {
	if r == ' ' {
		return true
	}
	return false
}))

fmt.Printf("%q\n", strings.Split(s1, " "))
fmt.Printf("%q\n", strings.SplitN(s1, " ", 2)) // 最多N个子串
fmt.Printf("%q\n", strings.SplitAfter(s1, " "))
fmt.Printf("%q\n", strings.SplitAfterN(s1, " ", 2))

// 输出
[abc def hig lmn pgo]
[abc def hig lmn pgo]
[abc def hig lmn pgo]
["abc" "def hig lmn pgo"]
["abc " "def " "hig " "lmn " "pgo"]
["abc " "def hig lmn pgo"]


```

## 字符串index
`strings.Index`这个函数使用也很简单, 注意查找中文字符应该用`strings.IndexRune`

```
s1 := "abc中国!"
fmt.Println(strings.Index(s1, "a"))
fmt.Println(strings.Index(s1, ""))
fmt.Println(strings.Index(s1, "!"))
fmt.Println(strings.Index(s1, "中"))
fmt.Println(strings.Index(s1, "z"))

fmt.Println(strings.IndexRune(s1, '中'))
```