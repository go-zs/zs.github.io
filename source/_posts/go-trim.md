---
title: golang strings.TrimLeft的坑
date: 2019-11-21 13:01:51
updated: 2019-11-21 13:01:51
tags:
---

最近遇到一个偶尔出现的bug，查看代码却看不出来任何问题。经过反复查看日志后定位到问题出现在 `strings.TrimLeft`这个函数上。

<!-- more -->

`TrimLeft`这个函数名称表面上是删除字符串左侧的字符，实际上却不是这样工作的。它的源码注释上也非常清楚的说明了这一点。
```golang
// TrimLeft returns a slice of the string s with all leading
// Unicode code points contained in cutset removed.
//
// To remove a prefix, use TrimPrefix instead.
func TrimLeft(s string, cutset string) string {
	if s == "" || cutset == "" {
		return s
	}
	return TrimLeftFunc(s, makeCutsetFunc(cutset))
}
```

参考文档所说，查看`strings.TrimPrefix`源码如下，这个函数功能才是真正需要的。
```golang
// TrimPrefix returns s without the provided leading prefix string.
// If s doesn't start with prefix, s is returned unchanged.
func TrimPrefix(s, prefix string) string {
	if HasPrefix(s, prefix) {
		return s[len(prefix):]
	}
	return s
}
```

总结下来，还是单元测试覆盖不够，少量的测试用例下，两个函数表现完全一致，而实际是有本质差别的。。