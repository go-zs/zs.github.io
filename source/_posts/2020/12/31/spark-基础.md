---
title: Spark入门（1）—— 安装及环境
date: 2020-12-31 13:42:57
tags:
---

最近入了大数据的坑，需要好好系统整理一下知识。

<!-- more -->

## 安装

`mac`下安装非常容易。

```shell
brew install apache-spark
```

其它平台可以登录 [Spark官方下载地址](https://spark.apache.org/downloads.html)，选择相应的版本下载。

解压

```shell
tar xvzf spark.1.6.tar.gz
```

配置环境变量

```shell
export SPARK_HOME=/usr/local/spark/spark-2.4.3
export PATH=$PATH:/usr/local/spark/spark-2.4.3/bin:/usr/local/spark/spark-2.4.3/sbin
```

`pyspark`可以直接通过`pip`安装，运行`pip install pyspark`即可完成安装。

## spark-shell

我们进到`/usr/local/Cellar/apache-spark/3.0.1`目录下，执行`ls bin/`，可以看到下面的文件。

```
$ ls bin/
docker-image-tool.sh load-spark-env.sh    run-example          spark-class          spark-sql            sparkR
find-spark-home      pyspark              spark-beeline        spark-shell          spark-submit
```

`pyspark`是`python`的`shell`，而`spark-shell`则是`scala`的`shell`。

