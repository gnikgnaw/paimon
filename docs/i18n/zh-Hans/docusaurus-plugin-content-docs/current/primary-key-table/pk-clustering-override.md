---
title: "PK 聚簇覆盖（PK Clustering Override）"
sidebar_position: 10
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

# PK 聚簇覆盖（PK Clustering Override） {#pk-clustering-override}

默认情况下，主键表（primary key table）中的数据文件按主键进行物理排序。这对于点查（point lookup）是最优的，但当查询基于非主键列进行过滤时，会损害扫描性能。

**PK 聚簇覆盖（PK Clustering Override）** 模式将数据文件的物理排序方式从主键改为用户指定的聚簇列。这能显著提升基于聚簇列进行过滤或分组的查询的扫描性能，同时仍通过删除向量（Deletion Vector）维持主键的唯一性。

## 快速开始 {#quick-start}

```sql
CREATE TABLE my_table (
    id BIGINT,
    dt STRING,
    city STRING,
    amount DOUBLE,
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'pk-clustering-override' = 'true',
    'clustering.columns' = 'city',
    'deletion-vectors.enabled' = 'true',
    'bucket' = '4'
);
```

对于 `first-row` 合并引擎（merge engine），删除向量已经内置，因此无需显式启用：

```sql
CREATE TABLE my_table (
    id BIGINT,
    dt STRING,
    city STRING,
    amount DOUBLE,
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'pk-clustering-override' = 'true',
    'clustering.columns' = 'city',
    'merge-engine' = 'first-row',
    'bucket' = '4'
);
```

如此设置后，每个桶（bucket）内的数据文件将按 `city` 而不是 `id` 进行物理排序。诸如 `SELECT * FROM my_table WHERE city = 'Beijing'` 这样的查询可以通过检查聚簇列上的 min/max 统计信息来跳过无关的数据文件。

## 要求 {#requirements}

| 配置项 | 要求 |
|--------|-------------|
| `pk-clustering-override` | `true` |
| `clustering.columns` | 必须设置（一个或多个非主键列） |
| `deletion-vectors.enabled` | 必须为 `true`（`first-row` 合并引擎无需此项） |
| `merge-engine` | 仅支持 `deduplicate`（默认）或 `first-row` |

## 何时使用 {#when-to-use}

在以下情况下，PK 聚簇覆盖会带来收益：

- 分析型查询频繁地基于非主键列进行过滤或聚合（例如 `WHERE city = 'Beijing'`）。
- 表使用 `deduplicate` 或 `first-row` 合并引擎。

:::info

尽管数据文件不再按主键排序，但基于桶键字段（默认即为主键）的过滤仍可受益于桶裁剪（bucket pruning）。查询引擎可以跳过不包含匹配值的整个桶，因此诸如 `WHERE id = 12345` 这样的查询仍然高效。

:::

**不支持的模式**：

- 合并引擎：`partial-update` 或 `aggregation`。
- changelog producer：`lookup` 或 `full-compaction`。
- 配置：`sequence.fields` 或 `record-level.expire-time`。
