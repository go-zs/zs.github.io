---
title: conda
date: 2020-10-06 15:58:03
tags:
---

`conda`是`Anoconda`提供的包管理工具。

<!-- more -->

## 安装conda

`conda`是`Anoconda`提供的包管理工具。

我们可以在[Annoconda下载地址](https://repo.anaconda.com/archive/)找到`Anoconda`的安装包，下载后安装。

如果我们只需要安装`conda`，而不需要`Anoconda`提供的集成环境，也可以选择安装`MiniConda`。

[miniconda官方安装地址](https://docs.conda.io/projects/conda/en/latest/user-guide/install/)

```shell
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-x86_64.sh -O ~/miniconda.sh
bash ~/miniconda.sh -b -p $HOME/miniconda
```

安装完成后可以执行`conda -V`查看安装情况及版本

```shell
$ conda -V
conda 4.9.2
```

## conda镜像源

`conda`国外源地址非常不稳定，我们可以选国内的镜像源。

首先执行`conda config --set show_channel_urls yes`，生成`.condarc`文件。然后添加以下内容:

```conf
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/pro
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

运行 `conda clean -i` 清除索引缓存，保证用的是镜像站提供的索引。


## conda基本使用

### 1.常用命令

```
1）conda list 查看安装了哪些包。

2）conda env list 或 conda info -e 查看当前存在哪些虚拟环境

3）conda update conda 检查更新当前conda
```


### 2.创建python虚拟环境


     使用 conda create -n your_env_name python=X.X（2.7、3.6等)命令创建python版本为X.X、名字为your_env_name的虚拟环境。your_env_name文件可以在Anaconda安装目录envs文件下找到。

### 3.使用激活(或切换不同python版本)的虚拟环境

    打开命令行输入python --version可以检查当前python的版本。

    使用如下命令即可 激活你的虚拟环境(即将python的版本改变)。

    Linux:  source activate your_env_name(虚拟环境名称)

    Windows: activate your_env_name(虚拟环境名称)

   这是再使用python --version可以检查当前python版本是否为想要的。

### 4.对虚拟环境中安装额外的包

    使用命令conda install -n your_env_name [package]即可安装package到your_env_name中

### 5.关闭虚拟环境(即从当前环境退出返回使用PATH环境中的默认python版本)

   使用如下命令即可。

   Linux: source deactivate

   Windows: deactivate

### 6.删除虚拟环境

   使用命令conda remove -n your_env_name(虚拟环境名称) --all， 即可删除。

### 7.删除环境中的某个包

   使用命令conda remove --name your_env_name  package_name 即可。