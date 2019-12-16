---
title: strings
date: 2019-12-09 08:39:40
tags:
---

golang的官方库strings库非常的强大，用了很久感觉需要整理一下相关api才能更好的掌握。


<!-- more -->

## 中文处理

首先，需要分清楚 `rune`和`byte`, 查看源码，我们可以知道，`byte`其实是字节，而`rune`是4个字节，很明显，`rune`其实就是`unicode`码。

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

