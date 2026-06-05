---
title: "数据分布"
sidebar_position: 2
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

# 数据分布 {#data-distribution}

桶（bucket）是读写的最小存储单元，每个桶目录中包含一棵 [LSM 树](./overview#lsm-trees)。

## 固定分桶（Fixed Bucket） {#fixed-bucket}

配置一个大于 0 的桶数即使用固定分桶（Fixed Bucket）模式，按照 `Math.abs(key_hashcode % numBuckets)` 来计算记录所属的桶。

调整桶数只能通过离线流程完成，参见 [调整桶数](../maintenance/rescale-bucket)。
桶数过大会导致小文件过多，桶数过小则会导致写入性能不佳。

## 动态分桶（Dynamic Bucket） {#dynamic-bucket}

这是主键表的默认模式，也可以通过配置 `'bucket' = '-1'` 来启用。

先到达的键会落入旧桶，新的键会落入新桶，桶与键的分布取决于数据到达的顺序。Paimon 会维护一个索引，用于确定哪个键对应哪个桶。

Paimon 会自动扩展桶的数量。

- 选项 1：`'dynamic-bucket.target-row-num'`：控制单个桶的目标行数。
- 选项 2：`'dynamic-bucket.initial-buckets'`：控制初始化桶的数量。
- 选项 3：`'dynamic-bucket.max-buckets'`：控制最大桶的数量。

:::info

动态分桶仅支持单个写入作业。请不要启动多个作业同时写入同一个分区（这会导致数据重复）。即使你启用了 `'write-only'` 并启动了一个独立 Compaction 作业，也无法解决这个问题。

:::

当你的更新不跨分区（没有分区，或主键包含所有分区字段）时，动态分桶模式使用 HASH 索引来维护键到桶的映射，它比固定分桶模式需要更多内存。

性能：

1. 一般来说不会有性能损失，但会有一些额外的内存消耗，一个分区中的 **1 亿** 条记录会多占用 **1 GB** 内存，不再活跃的分区不会占用内存。
2. 对于更新率较低的表，推荐使用此模式以显著提升性能。

## 延迟分桶（Postpone Bucket） {#postpone-bucket}

延迟分桶模式通过配置 `'bucket' = '-2'` 来启用。
该模式旨在解决难以确定固定桶数的问题，并支持为不同分区设置不同的桶数。

当向表中写入记录时，所有记录会先存储在每个分区的 `bucket-postpone` 目录中，读取方暂时无法访问这些记录。

为了将这些记录移动到正确的桶中并使其可读，你需要运行一个 Compaction 作业。
参见 `compact` [存储过程](../flink/procedures)。
首次进行 Compaction 的分区的桶数由配置项 `postpone.default-bucket-num` 控制，其默认值为 `1`。

最后，当你觉得某些分区的桶数过小时，也可以运行一个 rescale 作业。
参见 `rescale` [存储过程](../flink/procedures)。

## 跨分区 Upsert（Cross Partitions Upsert） {#cross-partitions-upsert}

当你需要跨分区 Upsert（主键不包含所有分区字段）时，推荐使用 '-1' 桶。
键动态（Key Dynamic）模式直接维护键到分区和桶的映射，使用本地磁盘，并在启动流式写入作业时通过读取表中所有现有键来初始化索引。不同的合并引擎有不同的行为：

1. 去重（Deduplicate）：从旧分区删除数据，并将新数据插入到新分区。
2. 部分更新（PartialUpdate）与聚合（Aggregation）：将新数据插入到旧分区。
3. First Row（FirstRow）：如果已存在旧值，则忽略新数据。

性能：对于数据量较大的表，会有显著的性能损失。此外，初始化也需要很长时间。

如果你的 Upsert 不依赖太旧的数据，可以考虑配置索引 TTL 以减少索引和初始化时间：
- `'cross-partition-upsert.index-ttl'`：rocksdb 索引及初始化中的 TTL，这可以避免维护过多索引而导致性能越来越差。

你也可以将跨分区 Upsert 与桶（N > 0）或桶（-2）一起使用，在这些模式下，没有全局索引来确保你的数据经过合理的去重，因此需要依赖你的输入具有完整的 changelog 才能保证数据的唯一性。

## 选择分区字段（Pick Partition Fields） {#pick-partition-fields}

在 warehouse（仓库目录）中，以下三类字段可被定义为分区字段：

- 创建时间（推荐）：创建时间通常是不可变的，因此你可以放心地将其作为分区字段，并将其加入主键。
- 事件时间（Event Time）：事件时间是原始表中的一个字段。对于 CDC 数据，例如从 MySQL CDC 同步的表，或由 Paimon 生成的 Changelog，它们都是完整的 CDC 数据，包括 `UPDATE_BEFORE` 记录，即使你声明了包含分区字段的主键，也可以达到唯一的效果（需要 `'changelog-producer'='input'`）。
- CDC op_ts：它不能被定义为分区字段，因为无法获知上一条记录的时间戳。因此你需要使用跨分区 Upsert，这会消耗更多资源。
