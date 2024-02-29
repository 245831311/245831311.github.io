---
title: Spark入门知识
date: 2021-08-27 11:16:39
categories: 
- 大数据
tags:
- Spark
---

# Spark入门知识

## Spark总体图

![image](/images/bdata/spark.jpg)

## Spark基础知识

### Spark RDD算子

- RDD算子：分布式数据集,能并行计算操作的且具有高可靠、可容错、自动感知的集合


- RDD分为两种：1.Transformation(转换):延迟执行；2.Action(动作)：马上执行

- RDD主要属性：

    1.作用于分区
    
    2.多个不同算子RDD
    
    3.可以依赖于不同RDD
    
    4.默认key-value 分区
    
    5.能自动选择数据位置执行RDD操作（计算任务会尽可能分配到数据库的存储位置）


- RDD 依赖关系

窄依赖：指的是每一个父RDD只会传递到唯一的子RDD中

宽依赖：指定是每一个父RDD能传递到1个或多个的子RDD中


- RDD常用算子介绍

Transformation算子
名称 | 功能
---|---
map（func） | 返回新rdd,可用于将输入元素经过转换获得类似map形式
filter(func) | 返回新rdd,可用于过滤元素
flatMap（func） | 返回新rdd,先扁平化,后map
mapPartitions(func) | 作用于分区的map
mapPartitionsWithIndex(func) | 作用于分区的map，有每个分区的索引值
reduceByKey(func, [numTasks]) | 经过函数func合并相同的key,返回rdd
groupByKey([numTasks]) |    合并相同的key,生成迭代器
union(otherDataset) |  两个RDD求并集，返回RDD
intersection(otherDataset)  |  对源RDD和参数RDD求交集后返回一个新的RDD
distinct([numTasks]))  |  对源RDD去重
sortBy(func,[ascending], [numTasks]) | 经过函数排序RDD


Action算子

名称 | 功能
---|---
reduce(func) | 通过func函数聚集RDD中的所有元素
collect() | 以数组的形式返回数据集的所有元素
count() | 返回RDD的元素个数
first() | 返回RDD的第一个元素（类似于take(1)）
take(n) | 返回第n个元素
saveAsTextFile(path) | 将RDD数据保存至文件系统中
countByKey() | 针对(K,V)类型的RDD，返回一个(K,Int)的map，表示每一个key对应的元素个数。
foreach(func) | 在每个RDD中执行func元素

- RDD 其他信息

广播变量：针对同一个app，共享数据到不同的worker上面

Lineage:将创建RDD的一系列信息记录下来，记录RDD的元数据和转换行为，以防RDD部分分区丢失的时候可以重新对这部分分区进行计算

RDD提供缓存功能:cache()

RDD提供checkPoint功能,可以将某阶段RDD数据存储到分布式的系统中（如hdfs）


### Spark执行结构

![spark-sql](/images/bdata/spark.png)

Driver:用户提交的应用程序代码在spark中运行起来就是一个driver，用户提交的程序运行起来就是一个driver，他是一个一段特殊的excutor进程，这个进程除了一般excutor都具有的运行环境外，这个进程里面运行着DAGscheduler,Tasksheduler Schedulerbackedn等组件。

Master:管理所有的worker,进而资源调度
Worker:管理当前计算节点,worker会启动一个executor来完成真正计算
Executor:真正执行任务的jvm










