---
title: "查询性能"
sidebar_position: 8
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

# 查询性能 {#query-performance}

## 表模式 {#table-mode}

表的 schema（表结构）对查询性能影响最大。参见 [表模式](./table-mode)。

对于读时合并（Merge-on-Read）表，你最需要关注的是桶的数量，它会限制读取数据的并发度。

对于 MOW（删除向量 Deletion Vectors）表、COW 表或 [读优化（Read Optimized）](../concepts/system-tables#read-optimized-table) 表，读取数据的并发度没有限制，并且它们还可以利用一些针对非主键列的过滤条件。

## 聚合下推 {#aggregate-push-down}

启用了删除向量（Deletion Vector）的表支持聚合下推：

```sql
SELECT COUNT(*) FROM TABLE WHERE DT = '20230101';
```

该查询可以在编译期被加速，并能非常快速地返回结果。

对于 Spark SQL，使用默认 `metadata.stats-mode` 的表可以被加速：

```sql
SELECT MIN(a), MAX(b) FROM TABLE WHERE DT = '20230101';
```

Min max 查询同样可以在编译期被加速，并能非常快速地返回结果。

## 通过主键过滤进行数据跳过 {#data-skipping-by-primary-key-filter}

对于常规的分桶表（例如 bucket = 5），主键的过滤条件将极大地加速查询，并减少对大量文件的读取。

## 通过文件索引进行数据跳过 {#data-skipping-by-file-index}

对于经过全量 Compaction 的文件，或对于设置了 `'deletion-vectors.enabled'` 的主键表，你可以使用文件索引，它会在读取侧通过索引来过滤文件。

定义 `file-index.bitmap.columns` 后，数据文件索引是一个外部索引文件，Paimon 会为每个文件创建其对应的索引文件。如果索引文件太小，它会直接存储在 manifest（清单）文件中，否则存储在数据文件所在的目录中。每个数据文件对应一个索引文件，该索引文件具有独立的文件定义，并且可以包含针对多列的不同类型的索引。

不同的文件索引在不同场景下可能各有效率。例如，布隆过滤器可以加速点查（point lookup）场景下的查询。使用位图可能会消耗更多空间，但能带来更高的精确度。

* [布隆过滤器（BloomFilter）](../concepts/spec/fileindex#index-bloomfilter)：`file-index.bloom-filter.columns`。
* [位图（Bitmap）](../concepts/spec/fileindex#index-bitmap)：`file-index.bitmap.columns`。
* [范围位图（Range Bitmap）](../concepts/spec/fileindex#index-range-bitmap)：`file-index.range-bitmap.columns`。

如果你想为现有表添加文件索引，且无需任何重写，可以使用 `rewrite_file_index` 存储过程（Procedure）。在使用该存储过程之前，你应当在目标表中配置好相应的配置项。你可以使用 ALTER 子句为表配置 `file-index.<filter-type>.columns`。

如何调用：参见 [Flink 存储过程](../flink/procedures)

## 分桶关联（Bucketed Join） {#bucketed-join}

固定分桶表（例如 bucket = 10）可用于在批式查询中按需避免 shuffle，例如，你可以使用以下 Spark SQL 来读取一张 Paimon 表：

```sql
SET spark.sql.sources.v2.bucketing.enabled = true;

CREATE TABLE FACT_TABLE (order_id INT, f1 STRING) TBLPROPERTIES ('bucket'='10', 'primary-key' = 'order_id');

CREATE TABLE DIM_TABLE (order_id INT, f2 STRING) TBLPROPERTIES ('bucket'='10', 'primary-key' = 'order_id');

SELECT * FROM FACT_TABLE JOIN DIM_TABLE on t1.order_id = t4.order_id;
```

`spark.sql.sources.v2.bucketing.enabled` 配置项用于为 V2 数据源启用分桶。开启后，Spark 会识别 V2 数据源通过 SupportsReportPartitioning 报告的特定分布，并会在必要时尝试避免 shuffle。

如果两张表具有相同的分桶策略和相同的桶数，那么代价高昂的关联 shuffle 将被避免。
