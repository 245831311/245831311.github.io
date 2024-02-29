---
title: Iceberg v1与v2表读取分析
date: 2023-06-01 12:00:00
categories: 
- 大数据
tags:
- 大数据
- Iceberg
---


# 1.背景

对于千万级别数据量的情况下，出现了v1,v2表读数据的超时问题，在这篇文章中，具体对比一下相同v1,v2表中读取的表现。

# 2.测试方式

## 2.1 准备

V1建表sql

```
-- from table hycan-ums.message_push_detail_0
CREATE TABLE IF NOT EXISTS iceberg.ods_hycan_ums.ods_message_push_detail_v1_notpart (
    `id` bigint not null,
    `message_config_id` bigint,
    `message_content_id` bigint,
    `task_id` bigint,
    `push_type` string,
    `user_id` int,
    `phone` string,
    `nickname` string,
    `message_snapshot` string,
    `push_status` string,
    `third_msg_id` string,
    `view` int,
    `open_status` string,
    `create_id` bigint,
    `create_time` timestamp,
    `update_id` bigint,
    `update_time` timestamp,
    `_create_date` string,
    primary key(`id`) NOT ENFORCED
)
with (
    'format' = 'PARQUET',
    'format-version' = '1',
    'write.metadata.previous-versions-max' = '10',
    'write.metadata.delete-after-commit.enabled' = 'true'
)

```

V2建表sql

```
-- from table hycan-ums.message_push_detail_0
CREATE TABLE IF NOT EXISTS iceberg.ods_hycan_ums.ods_message_push_detail_v2_notpart (
    `id` bigint not null,
    `message_config_id` bigint,
    `message_content_id` bigint,
    `task_id` bigint,
    `push_type` string,
    `user_id` int,
    `phone` string,
    `nickname` string,
    `message_snapshot` string,
    `push_status` string,
    `third_msg_id` string,
    `view` int,
    `open_status` string,
    `create_id` bigint,
    `create_time` timestamp,
    `update_id` bigint,
    `update_time` timestamp,
    `_create_date` string,
    primary key(`id`) NOT ENFORCED
)
with (
    'format' = 'PARQUET',
    'format-version' = '2',
    'write.metadata.previous-versions-max' = '10',
    'write.metadata.delete-after-commit.enabled' = 'true',
    'write.upsert.enabled' = 'true'
)

```

## 2.2 读取结果

### 2.2.1 V1表读取结果

![v1读取结果](/images/bdata/v1读取分析.jpg)

### 2.2.2 V2表读取结果

![v2读取结果](/images/bdata/v2读取分析.jpg)

## 3 思考

### 为什么造成这个问题

1.v1表与v2表功能差异

V1定义了如何来管理大型分析型的表，包括元数据文件、属性、数据类型、表的模式，分区信息，以及如何写入与读取。
V2表则是在V1表基础上进行了扩展，首要的是实现了行级数据的更新与删除功能，引入了delete file记录需要删除的数据

删除行数据的方式分为两种：Equality Deletes和Position Deletes。

2.如何确认猜想?

可以通过源码进行debug,目前经过排查确认在获取数据的时候进行了delete file过滤，导致问题.
![delete file慢问题](/images/bdata/Iceberg源码读取分析.jpg)

3.解决方案
定期合并delete files小文件
order优化、bloomfilter？
