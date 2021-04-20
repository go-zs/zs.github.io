---
title: Golang的官方mock工具——gomock
date: 2021-04-20 17:45:21
tags:
- golang

---

线上业务的单元测试是非常必要的，自己写接口`mock`方法多的时候非常麻烦，而`mock`包是官方提供的接口测试工具，通过`generate`代码生成的方式非常方便的实现接口的`mock`测试。

<!-- more -->

## 安装

```shell
go get github.com/golang/mock/gomock
```


## 基本使用

我们定义一个最简单的接口

```golang
package demo

//go:generate mockgen -package=demo -destination=gen.go gen Demo

type Demo interface {
	Create(key string) (string, error)
}
```

执行 `go generatge` 或者 `mockgen -package=demo -destination=gen.go gen Demo`。

执行成功后，会生成`gen.go`这个文件。

可以发现，它自动生成了`MockDemo`这个对象实现了`Demo`接口。

```golang
// MockDemo is a mock of Demo interface.
type MockDemo struct {
	ctrl     *gomock.Controller
	recorder *MockDemoMockRecorder
}
```

## 编写单元测试

我们要用`MockDemo`替换实际的接口，就需要用依赖注入的方式来实现接口的替换。

```golang
func Get(d Demo, key string) (string, error) {
	return d.Create(key)
}
```

然后新建`demo_test.go`

```golang
package demo

import (
	"testing"

	. "github.com/golang/mock/gomock"
	"github.com/stretchr/testify/require"
)

func TestGet(t *testing.T) {
	ctrl := NewController(t)
	defer ctrl.Finish()
    // 新建的mockDemo接口
	mockDemo := NewMockDemo(ctrl)
	testCases := []struct {
		name        string
		msg         string
		expectedErr error
		expectedRes string
	}{
		{
			name:        "1",
			msg:         "a1",
			expectedErr: nil,
			expectedRes: "a1",
		},
		{
			name:        "2",
			msg:         "a2",
			expectedErr: nil,
			expectedRes: "a2",
		},
	}
	for _, tt := range testCases {
		t.Run(tt.name, func(t *testing.T) {
            // 指定Mock接口Create方法返回值
			mockDemo.EXPECT().Create(tt.msg).Return(tt.msg, nil)
			r, err := Get(mockDemo, tt.msg)
			require.Equal(t, tt.expectedErr, err)
			require.Equal(t, tt.expectedRes, r)
		})
	}
}
```