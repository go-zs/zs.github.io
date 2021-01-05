---
title: Spark入门（4）—— UDF
date: 2020-12-31 16:55:07
tags:
---

`UDF`（User Defined Functions）是用户自定义单行操作的函数。

## UDF

<!-- more -->

最简单无参数示例如下：

```scala
scala> import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.SparkSession
scala> import org.apache.spark.sql.functions.udf
import org.apache.spark.sql.functions.udf
scala> val spark = SparkSession.builder().appName("UDF").getOrCreate()
// 定义random函数
scala> val random = udf(() => Math.random())
// 注册
scala> spark.udf.register("random", random.asNondeterministic())
// 执行
scala> spark.sql("SELECT random()").show()
+------------------+
|             UDF()|
+------------------+
|0.9792369214901931|
+------------------+
```

带参数的示例：

```scala
// 定义并注册带参数 x: Int 的UDF
val plusOne = udf((x: Int) => x + 1)
spark.udf.register("plusOne", plusOne)
spark.sql("SELECT plusOne(5)").show()
// +------+
// |UDF(5)|
// +------+
// |     6|
// +------+

// 一次定义带两个参数的UDF
spark.udf.register("strLenScala", (_: String).length + (_: Int))
spark.sql("SELECT strLenScala('test', 1)").show()
// +--------------------+
// |strLenScala(test, 1)|
// +--------------------+
// |                   5|
// +--------------------+

// 在where条件中使用UDF
spark.udf.register("oneArgFilter", (n: Int) => { n > 5 })
spark.range(1, 10).createOrReplaceTempView("test")
spark.sql("SELECT * FROM test WHERE oneArgFilter(id)").show()
// +---+
// | id|
// +---+
// |  6|
// |  7|
// |  8|
// |  9|
// +---+
```

## UDAFs

`UDAFs`(User Defined Aggregate Functions)是用户自定义聚合操作函数。

它支持三种聚合器

* IN
* BUF
* OUT

```scala
import org.apache.spark.sql.{Encoder, Encoders, SparkSession}
import org.apache.spark.sql.expressions.Aggregator

case class Employee(name: String, salary: Long)
case class Average(var sum: Long, var count: Long)

object MyAverage extends Aggregator[Employee, Average, Double] {
  // A zero value for this aggregation. Should satisfy the property that any b + zero = b
  def zero: Average = Average(0L, 0L)
  // Combine two values to produce a new value. For performance, the function may modify `buffer`
  // and return it instead of constructing a new object
  def reduce(buffer: Average, employee: Employee): Average = {
    buffer.sum += employee.salary
    buffer.count += 1
    buffer
  }
  // Merge two intermediate values
  def merge(b1: Average, b2: Average): Average = {
    b1.sum += b2.sum
    b1.count += b2.count
    b1
  }
  // Transform the output of the reduction
  def finish(reduction: Average): Double = reduction.sum.toDouble / reduction.count
  // Specifies the Encoder for the intermediate value type
  def bufferEncoder: Encoder[Average] = Encoders.product
  // Specifies the Encoder for the final output value type
  def outputEncoder: Encoder[Double] = Encoders.scalaDouble
}

scala> val ds = spark.read.json("employees.json").as[Employee]
ds: org.apache.spark.sql.Dataset[Employee] = [name: string, salary: bigint]

scala> ds.show()
+-------+------+
|   name|salary|
+-------+------+
|Michael|  3000|
|   Jack|  2000|
| Justin|  4500|
|    Ame|  8000|
+-------+------+

scala> val averageSalary = MyAverage.toColumn.name("average_salary")
averageSalary: org.apache.spark.sql.TypedColumn[Employee,Double] = myaverage(knownnotnull(assertnotnull(input[0, Average, true])).sum AS `sum`, knownnotnull(assertnotnull(input[0, Average, true])).count AS `count`, newInstance(class Average), boundreference()) AS `average_salary`

scala> val result = ds.select(averageSalary)
result: org.apache.spark.sql.Dataset[Double] = [average_salary: double]

scala> result.show()
+--------------+
|average_salary|
+--------------+
|        4375.0|
+--------------+
```