---
title: go(23)--goland简化单元测试的利器-testify
date: 2020-09-15 11:15:22
tags:
---

`golang`自带了一个非常强大的单元测试组件，让我们可以非常方便的编写单元测试，美中不足的是它缺少断言功能，很多时候不得不写很多的`if`判断。`testify`是一个非常强大的测试包，提供了很多非常实用的断言功能，大大提升了单元测试的可读性。

<!-- more -->

## 安装testify

执行下面的安装

```
go get github.com/stretchr/testify/assert
```

在`test`文件中引入`testify`

```golang
package xx_test

import (
  "testing"
  "github.com/stretchr/testify/assert"
)
```

## assert断言

参考官方示例

可用于判断`err`是否为`nil`,值是否相等，大于、小于, true、false等

```golang

package yours

import (
  "testing"
  "github.com/stretchr/testify/assert"
)

func TestSomething(t *testing.T) {

  // assert equality
  assert.Equal(t, 123, 123, "they should be equal")

  // assert inequality
  assert.NotEqual(t, 123, 456, "they should not be equal")

  // assert for nil (good for errors)
  assert.Nil(t, object)

  // assert for greater, less
	assert.Greater(t, 1, 5)
	assert.LessOrEqual(t, 1, 5)

  // assert for true
  assert.True(t, true)

  // assert for not nil (good when you expect something)
  if assert.NotNil(t, object) {
    // now we know that object isn't nil, we are safe to make
    // further assertions without causing any errors
    assert.Equal(t, "Something", object.Value)
  }
}
```

由于`golang`强类型的特性，要比较不同类型的值是否相等，不能用`assert.Equal`，可以用`assert.EqualValues`

```golang
assert.EqualValues(t, uint64(1), 1)

```

## accuire 

它和`assert`非常类似，参考源码，它只是在`assert`基础上，增加了`断言`失败的时候，退出`test`的处理。

```golang
// Equal asserts that two objects are equal.
//
//    assert.Equal(t, 123, 123)
//
// Pointer variable equality is determined based on the equality of the
// referenced values (as opposed to the memory addresses). Function equality
// cannot be determined and will always fail.
func Equal(t TestingT, expected interface{}, actual interface{}, msgAndArgs ...interface{}) {
	if h, ok := t.(tHelper); ok {
		h.Helper()
	}
	if assert.Equal(t, expected, actual, msgAndArgs...) {
		return
	}
	t.FailNow()
}
```

## Table Driven 表驱动测试

`golang`的匿名`struct`可以非常方便的实现表驱动测试。

```golang
// x.go
package x

func Filter(userIds []uint64) []uint64 {
	var i, j int
	m := make(map[uint64]struct{}, len(userIds))
	for ; j < len(userIds); j++ {
		if _, ok := m[userIds[j]]; !ok {
			m[userIds[j]] = struct{}{}
			userIds[i] = userIds[j]
			i++
		}
	}

	return userIds[:i]
}

// x_test.go
func TestFilter(t *testing.T) {
	var (
		testCases = []struct {
			userIds  []uint64
			expected []uint64
		}{
			{
				userIds:  nil,
				expected: nil,
			},
			{
				userIds:  []uint64{5, 2, 3},
				expected: []uint64{5, 2, 3},
			},
			{
				userIds:  []uint64{3, 3, 4},
				expected: []uint64{3, 4},
			},
			{
				userIds:  []uint64{6, 6, 6},
				expected: []uint64{6},
			},
			{
				userIds:  []uint64{7, 22, 22},
				expected: []uint64{7, 22},
			},
			{
				userIds:  []uint64{66, 15, 1, 66, 22, 11, 22},
				expected: []uint64{66, 15, 1, 22, 11},
			},
		}
	)
	for _, c := range testCases {
		r := Filter(c.userIds)
		assert.Equal(t, len(r), len(c.expected))
		for i := 0; i < len(c.expected); i++ {
			assert.Equal(t, c.expected[i], r[i])
		}
	}
}
```


## mock 使用

`mock`可以让我们非常方便的隔离外部依赖，比如发送邮件、连接数据库等。

```golang
import "github.com/stretchr/testify/mock"

type MockSettingGetter struct {
    mock.Mock
}

func (m *MockSettingGetter) Get(key string) (value []byte, err error) {
    args := m.Called(key)
    return args.Get(0).([]byte), args.Error(1)
}

func TestUpdateThreshold(t *testing.T) {
    tests := []struct {
        v      string
        err    error
        rs     int64
        hasErr bool
    }{
        {v: "1000", err: nil, rs: 1000, hasErr: false},
        {v: "a", err: nil, rs: 0, hasErr: true},
        {v: "", err: fmt.Errorf("consul is down"), rs: 0, hasErr: true},
    }

    for idx, test := range tests {
        mockSettingGetter := new(MockSettingGetter)
        mockSettingGetter.On("Get", mock.Anything).Return([]byte(test.v), test.err)

        limiter := &IPLimit{SettingGetter: mockSettingGetter}
        err := limiter.UpdateThreshold()
        if test.hasErr {
            assert.Error(t, err, "row %d", idx)
        } else {
            assert.NoError(t, err, "row %d", idx)
        }
        assert.Equal(t, test.rs, limiter.Threshold, "thredshold should equal, row %d", idx)
    }
}

```

## suite

```golang
// Basic imports
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"
)

// Define the suite, and absorb the built-in basic suite
// functionality from testify - including a T() method which
// returns the current testing context
type ExampleTestSuite struct {
    suite.Suite
    VariableThatShouldStartAtFive int
}

// Make sure that VariableThatShouldStartAtFive is set to five
// before each test
func (suite *ExampleTestSuite) SetupTest() {
    suite.VariableThatShouldStartAtFive = 5
}

// All methods that begin with "Test" are run as tests within a
// suite.
func (suite *ExampleTestSuite) TestExample() {
    assert.Equal(suite.T(), 5, suite.VariableThatShouldStartAtFive)
}

// In order for 'go test' to run this suite, we need to create
// a normal test function and pass our suite to suite.Run
func TestExampleTestSuite(t *testing.T) {
    suite.Run(t, new(ExampleTestSuite))
}
```