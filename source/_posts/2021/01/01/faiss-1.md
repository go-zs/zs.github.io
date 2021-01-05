---
title: faiss总结（1）—— 环境搭建
date: 2021-01-01 15:39:50
tags:
---

`faiss`是`Facebook Ai Research`开源的一款向量检索工具。这篇介绍一下它的环境搭建。

<!-- more -->

首先，用`pip`安装`faiss`，执行代码时会抛出`ModuleNotFoundError: No module named '_swigfaiss'`这个`Exception`，通常科学计算相关的环境搭建更推荐使用`conda`


## 安装CPU版本

```
#先安装mkl（英特尔数学核心函数库，可以大幅提升CPU训练速度）
conda install mkl
#安装faiss-cpu
conda install faiss-cpu -c pytorch
#测试安装是否成功
python -c "import faiss"
```


## 安装CUDA

首先安装依赖

```shell
sudo apt update # 更新 apt
sudo apt install gcc g++ make # 安装 gcc g++ make
sudo apt install libglu1-mesa libxi-dev libxmu-dev libglu1-mesa-dev freeglut3-dev # 安装依赖库
```

在[cuda官方下载页面](https://developer.nvidia.com/zh-cn/cuda-toolkit)选择合适的版本。

下载`toolkit`

```shell
wget http://developer.download.nvidia.com/compute/cuda/10.1/Prod/local_installers/cuda_10.1.243_418.87.00_linux.run
```

执行安装

```shell
sudo sh cuda_10.1.243_418.87.00_linux.run
```

## 安装GPU版本

```
# 确保已经安装了CUDA，否则会自动安装CPU版本。
conda install faiss-gpu -c pytorch # 默认 For CUDA8.0
conda install faiss-gpu cuda90 -c pytorch # For CUDA9.0
conda install faiss-gpu cuda91 -c pytorch # For CUDA9.1
```

