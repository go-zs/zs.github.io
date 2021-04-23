---
title: Makefile
date: 2021-04-23 15:32:35
tags:
- shell
---

总结下`Makefile`的常见用法。


## 基本用法 

`Makefile`基本语法

```
<target> : <prerequisites>  # prerequisites前置条件
[tab]  <commands>
```

## 目标

我们要执行下面的`Makefile`，运行`make name`，此时`name`就是命令执行的目标，它是一个命令的名字，并不是文件，所以它是一个伪目标。

```Makefile
name:
    echo "z"
```

当当前路径下刚好有一个`name`的文件或文件夹时，命令不会执行，并且输出以下的内容。

```
make: `name' is up to date.
```

我们可以通过 `.PONY`声明`name`是一个伪目标，这样就可以顺利执行了。

```Makefile
.PHONY: name

name:
    echo "z"
```

如果`make`命令没有附带任何目标，它会自动执行`Makefile`第一个目标。

## 前置条件

我们给`name`加上一个`prev`的前置条件，执行`make name`，可以发现`prev`会在执行`name`之前优先被自动执行。

```Makefile
.PHONY: name

name: prev
	echo "z"

prev:
	echo "prev"
```


## 常见符号

### @

前面我们执行`make`的时候会发现执行的命令被自动打印出来了，如果我们不希望打印命令，可以在命令前加上`@`。

```Makefile
.PHONY: name

name: prev
	@echo "z"

prev:
	@echo "prev"
```

这时候就不会输出命令了。

```
$ make
prev
z
```

### *,？通配符

通配符用法同`shell`脚本，常见于文件的匹配。

```Makefile
clean:
    rm -f *.pyc
```

### % 模式匹配

`make`通过模式来匹配目标

下面是一个最简单的示例

```Makefile
%.c:
	touch $@
```

执行`make test.c`，会创建`test.c`文件。

`%.o:%.c`这种模式能匹配到类似`1.o:1.c`这种。

### 自动变量

#### 



## 常见的坑

经常编写一个Makefile，粘贴过来，运行就会如下的错误。这个是因为`Makefile`换行只能用`tab`，而不能用空格。如果用`vim`，我们可能配置了`:set expandtab`，`tab`自动转换成空格，这时候需要输入命令`:set noexpandtab`。

```shell
Makefile:2: *** missing separator.  Stop.

```
