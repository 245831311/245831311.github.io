---
title: Spark Streaming基础知识点
date: 2021-08-27 11:16:39
categories: 
- 大数据
tags:
- Spark
---

# Spark Streaming基础知识点

## Spark Streaming 功能图

## Spark Streaming 特性

1. 易用：能够通过API创建应用
2. 容错：自身能够恢复丢失的操作或者工作的状态
3. 易整合到spark体系中

## 基础知识点

- DStream概念：DStream是基础抽象,代表了持续性的流式数据。在内部实现上，DStream表示一系列连续的RDD

- DStream相关操作(参照官网)

DStream的Transformations
名称 | 功能
---|---
map（func） | Return a new DStream by passing each element of the source DStream through a function func.
flatMap(func) | Similar to map, but each input item can be mapped to 0 or more output items.
filter(func) | Return a new DStream by selecting only the records of the source DStream on which func returns true.
repartition(numPartitions) | Changes the level of parallelism in this DStream by creating more or fewer partitions.
union(otherStream) | Return a new DStream that contains the union of the elements in the source DStream and otherDStream.
count() | Return a new DStream of single-element RDDs by counting the number of elements in each RDD of the source DStream.
reduce(func) | Return a new DStream of single-element RDDs by aggregating the elements in each RDD of the source DStream using a function func (which takes two arguments and returns one). The function should be associative so that it can be computed in parallel.
countByValue() | When called on a DStream of elements of type K, return a new DStream of (K, Long) pairs where the value of each key is its frequency in each RDD of the source DStream.
reduceByKey(func, [numTasks]) | When called on a DStream of (K, V) pairs, return a new DStream of (K, V) pairs where the values for each key are aggregated using the given reduce function. Note: By default, this uses Spark's default number of parallel tasks (2 for local mode, and in cluster mode the number is determined by the config property spark.default.parallelism) to do the grouping. You can pass an optional numTasks argument to set a different number of tasks.
join(otherStream, [numTasks]) | When called on two DStreams of (K, V) and (K, W) pairs, return a new DStream of (K, (V, W)) pairs with all pairs of elements for each key.
cogroup(otherStream, [numTasks]) | When called on a DStream of (K, V) and (K, W) pairs, return a new DStream of (K, Seq[V], Seq[W]) tuples.
transform(func) | Return a new DStream by applying a RDD-to-RDD function to every RDD of the source DStream. This can be used to do arbitrary RDD operations on the DStream.
updateStateByKey(func) | Return a new "state" DStream where the state for each key is updated by applying the given function on the previous state of the key and the new values for the key. This can be used to maintain arbitrary state data for each key.


主要讲解常用Operation

1.UpdateStateByKey Operation

能够将前后的数据通过给出的函数整合起来，如统计之类的功能

Java的Demo
```java
Function2<List<Integer>, Optional<Integer>, Optional<Integer>> updateFunction =
  (values, state) -> {
    Integer newSum = ...  // add the new values with the previous running count to get the new count
    return Optional.of(newSum);
  };
```

```java
JavaPairDStream<String, Integer> runningCounts = pairs.updateStateByKey(updateFunction);
```

2.Window Operations

窗口函数，通过滑动窗口间隔来进行计算

3.Transform Operation

Transform允许DStream上执行任意的RDD-to-RDD函数。通过该函数可以方便的扩展Spark API

4.Output Operations

能够将DStream的数据输出到多个外部系统



## 参考
1. [Spark Streaming guide](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)