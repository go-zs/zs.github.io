---
title: Spark入门（2）—— RDD
date: 2020-12-31 14:04:21
tags:
---

`RDD`是`Spark`中最基本的数据抽象，初识`RDD`也是非常难以理解。

<!-- more -->

## 基本概念

在学习Spark运行架构之前，需要先了解几个重要的概念：
* RDD：是弹性分布式数据集（Resilient Distributed Dataset）的简称，是分布式内存的一个抽象概念，提供了一种高度受限的共享内存模型；
* DAG：是Directed Acyclic Graph（有向无环图）的简称，反映RDD之间的依赖关系；
* Executor：是运行在工作节点（Worker Node）上的一个进程，负责运行任务，并为应用程序存储数据；
* 应用：用户编写的Spark应用程序；
*任务：运行在Executor上的工作单元；
* 作业：一个作业包含多个RDD及作用于相应RDD上的各种操作；
* 阶段：是作业的基本调度单位，一个作业会分为多组任务，每组任务被称为“阶段”，或者也被称为“任务集”。

## RDD

`RDD（Resilient Distributed Dataset）`叫做弹性分布式数据集，是`Spark`中最基本的数据抽象，它代表一个不可变、可分区、里面的元素可并行计算的集合。RDD具有数据流模型的特点：自动容错、位置感知性调度和可伸缩性。RDD允许用户在执行多个查询时显式地将工作集缓存在内存中，后续的查询能够重用工作集，这极大地提升了查询速度。

下图是以`WordCount`精图解`RDD`

![WordCount](https://pic.hupai.pro/img/1228818-20180421133911520-1150689001.png)

## RDD创建

1. 通过文件创建

```scala
scala> var textFile = sc.textFile("file:///usr/local/Cellar/apache-spark/3.0.1/word.txt")
textFile: org.apache.spark.rdd.RDD[String] = file:///usr/local/Cellar/apache-spark/3.0.1/word.txt MapPartitionsRDD[1] at textFile at <console>:24
```

2. 通过数据集合并行化的方式创建

```scala
scala> val list = List("Hadoop", "Spark", "Hive", "Spark")
list: List[String] = List(Hadoop, Spark, Hive, Spark)

scala> val rdd = sc.parallelize(list)
rdd: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[4] at parallelize at <console>:26

scala> val pairRDD = rdd.map(word => (word, 1))
pairRDD: org.apache.spark.rdd.RDD[(String, Int)] = MapPartitionsRDD[5] at map at <console>:25
```

3. 其他方式
   读取数据库、其它`RDD`转化等方式也可以生成`RDD`

## RDD编程

### Transformation

常用的键值对转换操作包括reduceByKey()、groupByKey()、sortByKey()、join()、cogroup()等，下面我们通过实例来介绍。

准备工作。

```scala
scala> val list = List("Hadoop","Spark","Hive","Spark")
list: List[String] = List(Hadoop, Spark, Hive, Spark)
 
scala> val rdd = sc.parallelize(list)
rdd: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[11] at parallelize at <console>:29
 
scala> val pairRDD = rdd.map(word => (word,1))
pairRDD: org.apache.spark.rdd.RDD[(String, Int)] = MapPartitionsRDD[12] at map at <console>:31
 
scala> pairRDD.foreach(println)
(Hadoop,1)
(Spark,1)
(Hive,1)
(Spark,1)
```

> filter(func)
筛选出满足函数func的元素

```scala
scala> pairRDD1.filter( x => x._1 == "spark").foreach(println)
(spark,1)
(spark,2)
```

> map(func)
将每个元素传递到函数func中

```scala
scala> pairRDD1.map( x => x._2 * 5).foreach(println)
5
10
15
25
```

> flatMap(func)
与map()相似，但每个输入元素都可以映射到0或多个输出结果

```scala
val result = data.flatMap (line => line.split(" ") )
```

> reduceByKey(func)
reduceByKey(func)的功能是，使用func函数合并具有相同键的值

```scala
scala> pairRDD.reduceByKey((a,b)=>a+b).foreach(println)
(Spark,2)
(Hive,1)
(Hadoop,1)
```

> groupByKey()

groupByKey()的功能是，对具有相同键的值进行分组。

```scala
scala> pairRDD.groupByKey().foreach(println)
(Spark,CompactBuffer(1, 1))
(Hive,CompactBuffer(1))
(Hadoop,CompactBuffer(1))

```


> keys

`keys`只会把键值对RDD中的key返回形成一个新的RDD。

```scala
scala> pairRDD.keys.foreach(println)
Hadoop
Spark
Hive
Spark

```

> values

`values`只会把键值对RDD中的value返回形成一个新的RDD。

```scala
scala> pairRDD.values.foreach(println)
1
1
1
1
```

> sortByKey()

`sortByKey()`的功能是返回一个根据键排序的RDD。

```scala
scala> pairRDD.sortByKey().foreach(println)
(Hadoop,1)
(Hive,1)
(Spark,1)
(Spark,1)
```

> mapValues(func)

`mapValues(func)`对键值对RDD中的每个value都应用一个函数，但是，key不会发生变化。

```scala
scala> pairRDD.mapValues(x => x*5).foreach(println)
(Hadoop,5)
(Spark,5)
(Hive,5)
(Spark,5)

```

> join

join(连接)操作是键值对常用的操作。“连接”(join)这个概念来自于关系数据库领域，因此，join的类型也和关系数据库中的join一样，包括内连接(join)、左外连接(leftOuterJoin)、右外连接(rightOuterJoin)等。最常用的情形是内连接，所以，join就表示内连接。

```scala
scala> val pairRDD1 = sc.parallelize(Array(("spark",1),("spark",2),("hadoop",3),("hadoop",5)))
pairRDD1: org.apache.spark.rdd.RDD[(String, Int)] = ParallelCollectionRDD[24] at parallelize at <console>:27
 
scala> val pairRDD2 = sc.parallelize(Array(("spark","fast")))
pairRDD2: org.apache.spark.rdd.RDD[(String, String)] = ParallelCollectionRDD[25] at parallelize at <console>:27
 
scala> pairRDD1.join(pairRDD2)
res9: org.apache.spark.rdd.RDD[(String, (Int, String))] = MapPartitionsRDD[28] at join at <console>:32
 
scala> pairRDD1.join(pairRDD2).foreach(println)
(spark,(1,fast))
(spark,(2,fast))
```


### Action

`Action`是真正触发计算的地方。`Spark`程序执行到行动操作时，才会执行真正的计算，从文件中加载数据，完成一次又一次转换操作，最终，完成行动操作得到结果。
下面列出一些常见的行动操作（`Action API`）:
* count() 返回数据集中的元素个数
* collect() 以数组的形式返回数据集中的所有元素
* first() 返回数据集中的第一个元素
* take(n) 以数组的形式返回数据集中的前n个元素
* reduce(func) 通过函数func（输入两个参数并返回一个值）聚合数据集中的元素
* foreach(func) 将数据集中的每个元素传递到函数func中运行*
