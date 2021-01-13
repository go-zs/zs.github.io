---
title: Python(1)——Typing系统用法总结
date: 2021-01-08 14:35:21
tags:
- python
- typing
categories:
- python
---

`Python3.9`版本发布后再次提升了`Typing`系统的易用性。下面对当前的`Typing`做一个用法的总结。

<!-- more -->

## Type Hint

2015年9月 `Guido van Rossum` 在 Python 3.5 引入了一个类型系统，允许开发者指定变量类型。

它的主要作用不是把`Python`变成静态语言，而是为了方便开发，供`IDE`使用，对代码运行不产生影响，运行时会过滤类型信息。


## 基本使用

变量、函数参数后面都可以按`: type`的方式添加类型注释，函数的返回值则是以`->`的方式添加类型注解。

```python
def greeting(name: str) -> str:
    a: str = "x"
    return 'Hello ' + name
```

## Type aliases

类型别名可以以变量赋值的方式创建。

```python
Vector = list[float]

def scale(scalar: float, vector: Vector) -> Vector:
    return [scalar * num for num in vector]

# typechecks; a list of floats qualifies as a Vector.
new_vector = scale(2.0, [1.0, -4.2, 5.4])
```

## NewType

自定义类型，允许将一个类定义成一个新的类型

```python
from typing import NewType

UserId = NewType('UserId', int)
some_id = UserId(524313)
```

## Callable

函数类型可以通过`Callable[[Arg1Type, Arg2Type], ReturnType]`的方式定义。

```python
from collections.abc import Callable

def feeder(get_next_item: Callable[[], str]) -> None:
    # Body

def async_query(on_success: Callable[[int], None],
                on_error: Callable[[int, Exception], None]) -> None:
    # Body

```

## 范型

范型可以实现保持入参与出参类型的一致。

```python
from collections.abc import Sequence
from typing import TypeVar

T = TypeVar('T')      # Declare type variable

def first(l: Sequence[T]) -> T:   # Generic function
    return l[0]
```

## Union，组合类型
联合类型 `Union[X, Y]`，非X即Y

```python
# 会过滤冗余
Union[int, str, int] == Union[int, str]
```

## Optinal

 Optional[X] 等价于 Union[X, None]。有默认值的参数本身就是可选参数。

```python
def foo(arg: int = 0) -> None:
def foo(arg: Optional[int]) -> None: 
```

## typing.Dict, typing.List, typing.Set, typing.Frozenset

可以定义常用数据容器`dict`、`list`和`set`等的成员类型。

```python
def count_words(text: str) -> Dict[str, int]:
  ...

T = TypeVar('T', int, float)

def vec2(x: T, y: T) -> List[T]:
    return [x, y]

def keep_positives(vector: Sequence[T]) -> List[T]:
    return [item for item in vector if item > 0]
```


## typing.TypeDict

这个可以定义字典的结构，非常好用。

```python
class Point2D(TypedDict):
    x: int
    y: int
    label: str

a: Point2D = {'x': 1, 'y': 2, 'label': 'good'}  # OK
b: Point2D = {'z': 3, 'label': 'bad'}           # Fails type check

assert Point2D(x=1, y=2, label='first') == dict(x=1, y=2, label='first')
```

## Iterable

定义一个指定类型的迭代器

```python
from collections.abc import Iterable

def zero_all_vars(vars: Iterable[LoggedVar[int]]) -> None:
    for var in vars:
        var.set(0)
```


## 万能Any

实在不知道类型怎么写可以用`Any`临时解决一下。如果都写`Any`那又何必使用`Type Hint`呢。。

```python
from typing import Any

a = None    # type: Any
a = []      # OK
a = 2       # OK

s = ''      # type: str
s = a       # OK

def foo(item: Any) -> int:
    # Typechecks; 'item' could be any type,
    # and that type might have a 'bar' method
    item.bar()
    ...
```


现在`Python`的类型系统几个版本的迭代已经非常好用了，美中不足的是整个生态上还没有广泛普及。跟`TypeScript`一样，哪怕只有自己项目上添加类型，已经能够大幅提升工作效率。

更完整的类型用法可以参考[官方文档](https://docs.python.org/3/library/typing.html)。