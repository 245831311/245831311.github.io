---
title: Flink内存模型
date: 2023-06-29 11:16:39
categories: 
- 大数据
tags:
- Flink
---



# Flink内存模型

## JobManager 内存模型
JobManager 是 Flink 集群的控制单元,负责调度与协调TaskManager的任务执行

### 内存组成讲解
![JobManager](/images/bdata/FlinkJobManager任务图.png)


![Flink-1.16JobManager内存模型](https://nightlies.apache.org/flink/flink-docs-release-1.16/fig/process_mem_model.svg)


| 组成部分 | 配置参数 | 描述 |
| --- | --- | --- |
| JVM Heap | 	jobmanager.memory.heap.size | JobManager 的 JVM 堆内存。 |
| Off-heap Memory | jobmanager.memory.off-heap.size | JobManager 的堆外内存（直接内存或本地内存）。
 |
| JVM Metaspace | jobmanager.memory.jvm-metaspace.size | Flink JVM 进程的 Metaspace |
| JVM Overhead | jobmanager.memory.jvm-overhead.min; jobmanager.memory.jvm-overhead.max; jobmanager.memory.jvm-overhead.fraction | 用于其他 JVM 开销的本地内存，例如栈空间、垃圾回收空间等。该内存部分为基于进程总内存的受限的等比内存部分。 |

  	 

## TaskManager 内存模型
Flink 的 TaskManager 负责执行用户代码

### 内存组成讲解

![taskManager flink ui1.16](/images/bdata/FlinkTaskManager任务图.png)

![TaskManager1.16内存模型](https://nightlies.apache.org/flink/flink-docs-release-1.16/fig/detailed-mem-model.svg)

| 组成部分 | 配置参数 | 描述 |
| --- | --- | --- |
| Framework Heap Memory | 	taskmanager.memory.framework.heap.size | Flink JVM框架内存 |
| Task Heap Memory | taskmanager.memory.task.heap.size | 用于 Flink 应用的算子及用户代码的 JVM 堆内存。 |
| Managed memory | taskmanager.memory.managed.size；taskmanager.memory.managed.fraction | 由 Flink 管理的用于排序、哈希表、缓存中间结果及 RocksDB State Backend 的本地内存。 |
| Framework Off-heap Memory |taskmanager.memory.framework.off-heap.size | 框架堆外内存（Framework Off-heap Memory）	taskmanager.memory.framework.off-heap.size	用于 Flink 框架的堆外内存（直接内存或本地内存） |
| Task Off-heap Memory | taskmanager.memory.task.off-heap.size | 任务堆外内存（Task Off-heap Memory）	taskmanager.memory.task.off-heap.size	用于 Flink 应用的算子及用户代码的堆外内存（直接内存或本地内存）。 |
| Network Memory | taskmanager.memory.network.min；taskmanager.memory.network.max；taskmanager.memory.network.fraction | 用于任务之间数据传输的直接内存（例如网络传输缓冲）。该内存部分为基于 Flink 总内存的受限的等比内存部分。这块内存被用于分配网络缓冲 |
| JVM Metaspace | taskmanager.memory.jvm-metaspace.size | Flink JVM 进程的 Metaspace |
| JVM Overhead	 | taskmanager.memory.jvm-overhead.min；taskmanager.memory.jvm-overhead.max；taskmanager.memory.jvm-overhead.fraction | 用于其他 JVM 开销的本地内存，例如栈空间、垃圾回收空间等。该内存部分为基于进程总内存的受限的等比内存部分。 |



## 参考
1. [Flink1.16 TaskManager内存配置](https://nightlies.apache.org/flink/flink-docs-release-1.16/zh/docs/deployment/memory/mem_setup_tm/)
1. [Flink1.16 JobManager内存配置](https://nightlies.apache.org/flink/flink-docs-release-1.16/zh/docs/deployment/memory/mem_setup_jobmanager/)