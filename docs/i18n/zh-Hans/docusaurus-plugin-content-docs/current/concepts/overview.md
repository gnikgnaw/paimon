---
title: "概述"
sidebar_position: 1
---

<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# 概述 {#overview}

Apache Paimon 的架构：

![](/img/architecture.png)

如上方架构图所示：

**读/写：** Paimon 支持多种灵活的方式来读/写数据并执行 OLAP 查询。
- 对于读，它支持消费以下数据
  - 来自历史快照（批式模式），
  - 来自最新偏移量（流式模式），或
  - 以混合方式读取增量快照。
- 对于写，它支持
  - 从数据库的 changelog 进行流式同步（CDC）
  - 从离线数据进行批量 insert/overwrite。

**生态系统：** 除了 Apache Flink，Paimon 还支持由其他计算引擎读取，例如 Apache Spark、StarRocks、Apache Doris、Apache Hive 和 Trino。

**内部实现：**
- 在底层，Paimon 将列式文件存储在文件系统/对象存储上
- 文件的元数据保存在 manifest（清单）文件中，提供大规模存储和数据跳过（data skipping）能力。
- 对于主键表，使用 LSM 树结构来支持大量数据更新和高性能查询。

## 统一存储 {#unified-storage}

对于像 Apache Flink 这样的流式引擎，通常有三种类型的连接器：
- 消息队列，例如 Apache Kafka，它在该流水线中同时用于 source 和
  中间阶段，以保证延迟保持在
  秒级以内。
- OLAP 系统，例如 ClickHouse，它以流式方式接收已处理的数据
  并服务于用户的即席（ad-hoc）查询。
- 批式存储，例如 Apache Hive，它支持传统批处理的各种操作，
  包括 `INSERT OVERWRITE`。

Paimon 提供了表抽象。它的使用方式
与传统数据库没有区别：
- 在 `batch` 执行模式下，它的行为类似于 Hive 表，并
  支持 Batch SQL 的各种操作。查询它可以看到
  最新的快照。
- 在 `streaming` 执行模式下，它的行为类似于消息队列。
  查询它就像从消息队列中查询流式 changelog 一样，
  其中历史数据永不过期。
