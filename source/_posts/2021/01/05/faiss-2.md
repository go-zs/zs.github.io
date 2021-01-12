---
title: faiss使用文档（2）—— 基础
date: 2021-01-05 16:31:26
tags:
- python
- faiss
categories:
- python
---

这篇汇总介绍一下`faiss`里面用到的算法及概念。

<!-- more -->

## index

faiss的核心就是索引（index）概念，它封装了一组向量，并且可以选择是否进行预处理，帮忙高效的检索向量。 faiss中由多种类型的索引，我们可以是呀最简单的索引类型：indexFlatL2，这就是暴力检索L2距离（欧式距离）。 不管建立什么类型的索引，我们都必须先知道向量的维度。

### IndexFlatL2

```python
import numpy as np


def main():
    d = 3  # 向量维度
    nb = 100000  # 向量集大小
    nq = 10000  # 查询次数
    np.random.seed(1234)  # 随机种子,使结果可复现
    xb = np.random.random((nb, d)).astype('float32')
    xb[:, 0] += np.arange(nb) / 1000.  # 每一项增加了一个等差数列的对应项数
    xq = np.random.random((nq, d)).astype('float32')
    xq[:, 0] += np.arange(nq) / 1000.
    
    import faiss
    index = faiss.IndexFlatL2(d)  # 构建FlatL2索引
    print(index.is_trained)
    index.add(xb)  # 向索引中添加向量
    print(index.ntotal)
    
    k = 4  # k=4的 k近邻搜索
    D, I = index.search(xb[:5], k)  # 测试
    print(I)
    print(D)
    D, I = index.search(xq, k)  # 执行搜索
    print(I[:5])  # 最初五次查询的结果
    print(I[-5:])  # 最后五次查询的结果


if __name__ == '__main__':
    main()

```

### IndexIVFFlat

```python
def main():
    import numpy as np
    d = 64  # 向量维度
    nb = 100000  # 向量集大小
    nq = 10000  # 查询次数
    np.random.seed(1234)  # 随机种子,使结果可复现
    xb = np.random.random((nb, d)).astype('float32')
    xb[:, 0] += np.arange(nb) / 1000.
    xq = np.random.random((nq, d)).astype('float32')
    xq[:, 0] += np.arange(nq) / 1000.
    
    import faiss
    
    nlist = 100
    k = 4
    quantizer = faiss.IndexFlatL2(d)  # the other index
    index = faiss.IndexIVFFlat(quantizer, d, nlist, faiss.METRIC_L2)
    # here we specify METRIC_L2, by default it performs inner-product search
    
    assert not index.is_trained
    index.train(xb)
    assert index.is_trained
    
    index.add(xb)  # 添加索引可能会有一点慢
    D, I = index.search(xq, k)  # 搜索
    print(I[-5:])  # 最初五次查询的结果
    index.nprobe = 10  # 默认 nprobe 是1 ,可以设置的大一些试试
    D, I = index.search(xq, k)
    print(I[-5:])  # 最后五次查询的结果


if __name__ == '__main__':
    main()

```

### IndexIVFPQ

```python
def main():
    import numpy as np
    d = 64  # 向量维度
    nb = 100000  # 向量集大小
    nq = 10000  # 查询次数
    np.random.seed(1234)  # 随机种子,使结果可复现
    xb = np.random.random((nb, d)).astype('float32')
    xb[:, 0] += np.arange(nb) / 1000.
    xq = np.random.random((nq, d)).astype('float32')
    xq[:, 0] += np.arange(nq) / 1000.
    
    import faiss
    
    nlist = 100
    m = 8
    k = 4
    quantizer = faiss.IndexFlatL2(d)  # 内部的索引方式依然不变
    index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)
    # 每个向量都被编码为8个字节大小
    index.train(xb)
    index.add(xb)
    D, I = index.search(xb[:5], k)  # 测试
    print(I)
    print(D)
    index.nprobe = 10  # 与以前的方法相比
    D, I = index.search(xq, k)  # 检索
    print(I[-5:])


if __name__ == '__main__':
    main()


```


## k-means聚类

聚类算法有很多种，`K-Means` 是聚类算法中的最常用的一种，算法最大的特点是简单，好理解，运算速度快，但是只能应用于连续型的数据，并且一定要在聚类前需要手工指定要分成几类。

## PCA降维

PCA（Principal Component Analysis） 是一种常见的数据分析方式，常用于高维数据的降维，可用于提取数据的主要特征分量。

```python
# eg:
# IndexIVFPQ索引应该设定为256维，而不是2048维   
coarse_quantizer = faiss.IndexFlatL2(256) 
sub_index = faiss.IndexIVFPQ(coarse_quantizer, 256, ncoarse, 16, 8)
# PCA 2048->256   
# 在降维后进行了随机旋转(第四个参数)   
pca_matrix = faiss.PCAMatrix(2048, 256, 0, True)  

# 封装索引   
index = faiss.IndexPreTransform(pca_matrix, sub_index)
 
# 也需要对PCA进行训练   
index.train(...)
# PCA要比添加操作更早执行   
index.add(...)
```

## 乘积量化(Product Quantizer)，PQ解码

 为了扩展到非常大的数据集，Faiss提供了基于产品量化器的有损压缩来压缩存储的向量的变体。压缩的方法基于乘积量化(`Product Quantizer`)。


