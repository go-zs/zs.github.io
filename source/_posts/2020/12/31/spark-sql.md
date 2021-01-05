---
title: Spark入门（3）—— SQL&DataFrame
date: 2020-12-31 14:53:40
tags:
---

`Spark SQL`是Spark生态系统中非常重要的组件，其前身为Shark。Shark是Spark上的数据仓库，最初设计成与Hive兼容，但是该项目于2014年开始停止开发，转向Spark SQL。Spark SQL全面继承了Shark，并进行了优化。

<!-- more -->


## Spark SQL

spark SQL是spark的一个模块，主要用于进行结构化数据的处理。它提供的最核心的编程抽象就是DataFrame。

提供一个编程抽象（DataFrame） 并且作为分布式 SQL 查询引擎

DataFrame：它可以根据很多源进行构建，包括：结构化的数据文件，hive中的表，外部的关系型数据库，以及RDD

（1）容易整合

（2）统一的数据访问方式

（3）兼容 Hive

（4）标准的数据连接

## DataFrame
`DataFrame`是一种以`RDD`为基础的分布式数据集，类似于传统数据库中的二维表格。
`DataFrame`的推出，让`Spark`具备了处理大规模结构化数据的能力，不仅比原有的`RDD`转化方式更加简单易用，而且获得了更高的计算性能。`Spark`能够轻松实现从`MySQL`到`DataFrame`的转化，并且支持`SQL`查询。

创建一个`people.json`文件并写入

```json
{"name":"Michael"}
{"name":"Andy", "age":30}
{"name":"Justin", "age":19}
```

在`shell`中执行下面语句

```scala
scala> import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.SparkSession

scala> val  spark=SparkSession.builder().getOrCreate()
spark: org.apache.spark.sql.SparkSession = org.apache.spark.sql.SparkSession@e83d546

scala> import spark.implicits._
import spark.implicits._

scala> val df = spark.read.json("file:///usr/local/Cellar/apache-spark/3.0.1/people.json")
df: org.apache.spark.sql.DataFrame = [age: bigint, name: string]
```

然后就可以用生成的`df`对象执行一系列的操作，这些`API`和`Python`中`pandas`生成的`DataFrame`非常相似。

```scala
// 打印模式信息
scala> df.printSchema()
root
 |-- age: long (nullable = true)
 |-- name: string (nullable = true)
 
// 选择多列
scala> df.select(df("name"),df("age")+1).show()
+-------+---------+
|   name|(age + 1)|
+-------+---------+
|Michael|     null|
|   Andy|       31|
| Justin|       20|
+-------+---------+
 
// 条件过滤
scala> df.filter(df("age") > 20 ).show()
+---+----+
|age|name|
+---+----+
| 30|Andy|
+---+----+
 
// 分组聚合
scala> df.groupBy("age").count().show()
+----+-----+
| age|count|
+----+-----+
|  19|    1|
|null|    1|
|  30|    1|
+----+-----+
 
// 排序
scala> df.sort(df("age").desc).show()
+----+-------+
| age|   name|
+----+-------+
|  30|   Andy|
|  19| Justin|
|null|Michael|
+----+-------+
 
//多列排序
scala> df.sort(df("age").desc, df("name").asc).show()
+----+-------+
| age|   name|
+----+-------+
|  30|   Andy|
|  19| Justin|
|null|Michael|
+----+-------+
 
//对列进行重命名
scala> df.select(df("name").as("username"),df("age")).show()
+--------+----+
|username| age|
+--------+----+
| Michael|null|
|    Andy|  30|
|  Justin|  19|
+--------+----+

```

## RDD转换成DataFrame

1. 通过`case class`创建（反射）

```scala
case class People(var name:String,var age:Int)
object TestDataFrame1 {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("RDDToDataFrame").setMaster("local")
    val sc = new SparkContext(conf)
    val context = new SQLContext(sc)
    // 将本地的数据读入 RDD， 并将 RDD 与 case class 关联
    val peopleRDD = sc.textFile("E:\\666\\people.txt")
      .map(line => People(line.split(",")(0), line.split(",")(1).trim.toInt))
    import context.implicits._
    // 将RDD 转换成 DataFrames
    val df = peopleRDD.toDF
    //将DataFrames创建成一个临时的视图
    df.createOrReplaceTempView("people")
    //使用SQL语句进行查询
    context.sql("select * from people").show()
  }
}
```

2. 通过`structType`创建（编程接口）

```
object TestDataFrame2 {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("TestDataFrame2").setMaster("local")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)
    val fileRDD = sc.textFile("E:\\666\\people.txt")
    // 将 RDD 数据映射成 Row，需要 import org.apache.spark.sql.Row
    val rowRDD: RDD[Row] = fileRDD.map(line => {
      val fields = line.split(",")
      Row(fields(0), fields(1).trim.toInt)
    })
    // 创建 StructType 来定义结构
    val structType: StructType = StructType(
      //字段名，字段类型，是否可以为空
      StructField("name", StringType, true) ::
        StructField("age", IntegerType, true) :: Nil
    )
    /**
      * rows: java.util.List[Row],
      * schema: StructType
      * */
    val df: DataFrame = sqlContext.createDataFrame(rowRDD,structType)
    df.createOrReplaceTempView("people")
    sqlContext.sql("select * from people").show()
  }
}

```

3. 通过`json`文件创建

参考前面的示例。


## 使用SQL

可以直接使用`SQL`对`DataFrame`操作

```scala
// 注册临时视图
scala> df.createOrReplaceTempView("people")

scala> val sqlDF = spark.sql("SELECT * FROM people")
sqlDF: org.apache.spark.sql.DataFrame = [age: bigint, name: string]

scala> sqlDF.show()
+----+-------+
| age|   name|
+----+-------+
|null|Michael|
|  30|   Andy|
|  19| Justin|
+----+-------+
```

