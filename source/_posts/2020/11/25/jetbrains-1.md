---
title: jetbrains技巧总结(1)——git
date: 2020-11-25 10:29:41
tags:
---

`git`命令行的功能足够强大，但是像修改记录对比、合并冲突等这些操作可视化更为方便。`jetbrains`家族的`IDE`都内置了`git`客户端，能够通过可视化的方式极大的简化操作。

<!-- more -->


## Annotate

如果需要查看当前文件最新的变更，右键点击行号的右侧，弹出的菜单中选择`Annotate`功能。

![Annnotate](https://pic.hupai.pro/img/xdfasdfs.png)


然后可以看到这个文件最新的变更记录的日期及修改人，选择任一个变更记录，右键选择`Show Diff`可以查看这次变更前后的对比。

![](https://pic.hupai.pro/img/xcafsdqwer.png)


## 撤销工作区变更

当我们对工作区有修改之后，变更位置右侧会变有修改标识，点击可以选择查看、撤回等操作

![变更](https://pic.hupai.pro/img/20201125132041.png)


## 查看文件修改历史

选择某一文件，右键 -> git -> Show History，可以看到此文件的所有历史提交记录，选择对应`commit`可以在面板右侧看到变更前后的记录。

![文件历史](https://pic.hupai.pro/img/20201125132501.png)


## 代码对比

在打开的`git`历史中，选中分支，右键 -> Show Diff，可以查看当前代码与此分支的对比。

![分支](https://pic.hupai.pro/img/234234x.png)


## 合并分支，解决冲突

可视化操作对于冲突的解决是非常方便的。我们可以用`git`菜单栏上的合并分支功能进行合并。如果没有冲突会直接合并，出现冲突则会弹出`resolve`面板。

![合并分支](https://pic.hupai.pro/img/212341.png)

在打开的面板中，左侧是当前分支，右侧是合并分支，中间是合并之后的代码。我们可以非常方便的对比，快速选取代码的保留和抛弃。

![冲突](https://pic.hupai.pro/img/20201125134326.png)