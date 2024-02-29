
---
title: 深挖Iceberg
date: 2021-07-26 11:16:39
categories: 
- 大数据
tags:
- Flink
- Iceberg
---


# Iceberg 是什么?

Iceberg是一个开源表格式的存储中间层，基于hdfs/Amazon S3等存储引擎之上，spark/flink等计算引擎之下的中间层。

Iceberg能够支持秒级延迟的流批一体组件

# Iceberg特性

1.schema演变：支持增，改，删表、重命名、重排序

2.隐藏分区：能够通过函数方式动态指定分区，减少不必要的分区值的生成

3.分区模式演变：支持表分区更改，并不会影响原来sql逻辑

4.版本回溯:可以恢复到任务一个版本的数据

5.无需分布式引擎，单表支持

# Iceberg技术点以及背后的原理

![image](/images/bdata/Iceberg基础图.jpg)

## 基础元素：

Metadata:元数据信息，记录Table结构信息、 分区策略、数据版本以及当前使用的snapshot序列号；命名xxxxx.metadata.json

Snapshot:快照，记录了Table某一时刻的状态，记录了当前manifest list的引用。 命名snap-xxxxx.avro

Manifest list:保存了一系列的manifest file 的列表清单，同时伴随着每个分区字段的数值范围信息。

Manifest file:记录了一系列的Data files文件信息，而且还记录Data file的分区数据和列级统计信息，包括分区字段最大值、最小值、文件大小、文件

路径、行数等。

Data file:实际数据文件，一般格式orc、parquet等

上述每次基于checkpoint commit都会生成。

## V1、V2表：是iceberg table两种不同类型的表

V1表：支持流批读写，只有datafile的概念，数据不能被删除，适用的场景适合日志场景、用户行为数据这种一次性数据的场景（没有删改）

V2表：基于V1表做了扩展，支持upsert的功能，基于Merge On Read理念，支持批读写、流写入，不支持流读。在datafile概念上增加了deletefile的

概念，实现了数据的去重方案。具体通过equality delete 与position delete两种方式。一般在manifest list中记录哪些是data file或者delete file文件

&#x9;equality delete:等值删除

&#x9;position delete:位置删除

由于 datafile与deletefile存在顺序的问题，无法确定数据的一致性，因此引入了sequence number的序列号，以解决数据读取的一致性。

## 快速扫描的原理：目的是快速获取指定文件

两步：

&#x9;Manifest file 过滤：先通过manefest list上的分区字段数值范围定位到manifest file列表

&#x9;Data file 过滤：然后通过manifest file上的分区数据，以及分区字段最大最小值过滤数据文件

# iceberg使用过程中的问题

&#x9;1.坏表：出现在commit的时候，iceberg异常导致了文件丢失，以致于整个Table崩溃.

&#x9;2.基于checkpoint时间批量插入，导致小文件众多
