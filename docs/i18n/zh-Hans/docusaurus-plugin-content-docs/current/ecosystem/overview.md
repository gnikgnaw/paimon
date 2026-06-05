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

## 兼容性矩阵 {#compatibility-matrix}

|                                     引擎                                      |    版本    |  批式读 | 批式写 |  建表 |  修改表  | 流式写  |  流式读  | 批式覆盖写 | DELETE 与 UPDATE | MERGE INTO | 时间旅行 |
|:-------------------------------------------------------------------------------:|:-------------:|:-----------:|:-----------:|:-------------:|:-------------:|:----------------:|:----------------:|:---------------:|:---------------:|:----------:|:-----------:|
|                                      Flink                                      |  1.16 - 1.20  |     ✅      |      ✅      |      ✅       |  ✅(1.17+)   |        ✅        |       ✅        |        ✅        |    ✅(1.17+)     |     ❌      |      ✅      |
|                                      Spark                                      |   3.2 - 4.0   |     ✅      |      ✅      |      ✅       |      ✅      |      ✅(3.3+)    |    ✅(3.3+)     |        ✅        |        ✅        |     ✅      |   ✅(3.3+)   |
|                                      Hive                                       |   2.1 - 3.1   |     ✅      |      ✅      |      ✅       |      ❌      |        ❌        |       ❌        |        ❌        |        ❌        |     ❌      |      ✅      |
|                                      Trino                                      |   420 - 440   |     ✅      |   ✅(427+)   |   ✅(427+)    |   ✅(427+)   |        ❌        |       ❌        |        ❌        |        ❌        |     ❌      |      ✅      |
|                                     Presto                                      | 0.236 - 0.280 |     ✅      |      ❌      |      ✅       |      ✅      |        ❌        |       ❌        |        ❌        |        ❌        |     ❌      |      ❌      |
| [StarRocks](https://docs.starrocks.io/docs/data_source/catalog/paimon_catalog/) |     3.1+      |     ✅      |      ❌      |      ❌       |      ❌      |        ❌        |       ❌        |        ❌        |        ❌        |     ❌      |      ✅      |
| [Doris](https://doris.apache.org/docs/dev/lakehouse/catalogs/paimon-catalog)      |    2.0.6+     |     ✅      |      ❌      |      ❌       |      ❌      |        ❌        |       ❌        |        ❌        |        ❌        |     ❌      |      ✅      |

## 流式引擎 {#streaming-engines}

### Flink Streaming {#flink-streaming}

Flink 是功能最全面的流计算引擎，被广泛用于数据 CDC 接入以及流式管道的构建。

推荐版本为 Flink 1.17.2。

### Spark Streaming {#spark-streaming}

你也可以使用 Spark Streaming 来构建流式管道。Spark 的 schema 演进能力会有更好的实现，但你必须接受 mini-batch（微批）的机制。

## 批式引擎 {#batch-engines}

### Spark Batch {#spark-batch}

Spark Batch 是使用最广泛的批计算引擎。

推荐版本为 Spark 3.5.8。

### Flink Batch {#flink-batch}

Flink Batch 也可使用，它能让你的管道在流批一体方面更加融合。

## OLAP 引擎 {#olap-engines}

### StarRocks {#starrocks}

StarRocks 是最为推荐的 OLAP 引擎，具备最先进的集成能力。

推荐版本为 StarRocks 3.2.6。

### 其他 OLAP {#other-olap}

你也可以使用 Doris、Trino 和 Presto，或者，你也可以直接使用 Spark、Flink 和 Hive 来查询 Paimon 表。

## 下载 {#download}

[下载链接](../project/download#engine-jars)
