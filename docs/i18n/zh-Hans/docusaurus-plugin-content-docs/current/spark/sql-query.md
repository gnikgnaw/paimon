---
title: "SQL 查询"
sidebar_position: 4
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

# SQL 查询 {#sql-query}

与其他所有表一样，Paimon 表可以通过 `SELECT` 语句进行查询。

## 批式查询 {#batch-query}

Paimon 的批式读取会返回表的某个快照中的全部数据。默认情况下，批式读取返回最新的快照。

```sql
-- read all columns
SELECT * FROM t;
```

Paimon 还支持读取一些隐藏的元数据列，目前支持以下列：

- `__paimon_partition`：记录所属的分区。
- `__paimon_bucket`：记录所属的桶。
- `__paimon_row_index`：记录的行索引。（仅适用于非主键表、删除向量表或经过全量 Compaction 的主键表。）
- `__paimon_file_path`：记录所在的文件路径。（仅适用于非主键表、删除向量表或经过全量 Compaction 的主键表。）
- `_ROW_ID`：记录的唯一行 id。（仅适用于启用行追踪（row-tracking）的表。）
- `_SEQUENCE_NUMBER`：记录的序列号。（仅适用于启用行追踪（row-tracking）的表。）

例如：

```sql
-- read all columns and the corresponding file path, partition, bucket, rowIndex of the record
SELECT *, __paimon_file_path, __paimon_partition, __paimon_bucket, __paimon_row_index FROM t;
```

### 批式时间旅行 {#batch-time-travel}

Paimon 的批式读取结合时间旅行（time travel）可以指定一个快照或一个标签（Tag），并读取相应的数据。

需要 Spark 3.3+。

你可以在查询中使用 `VERSION AS OF` 和 `TIMESTAMP AS OF` 来进行时间旅行：

```sql
-- read the snapshot with id 1L (use snapshot id as version)
SELECT * FROM t VERSION AS OF 1;

-- read the snapshot from specified timestamp 
SELECT * FROM t TIMESTAMP AS OF '2023-06-01 00:00:00.123';

-- read the snapshot from specified timestamp in unix seconds
SELECT * FROM t TIMESTAMP AS OF 1678883047;

-- read tag 'my-tag'
SELECT * FROM t VERSION AS OF 'my-tag';

-- read the snapshot from specified watermark. will match the first snapshot after the watermark
SELECT * FROM t VERSION AS OF 'watermark-1678883047356';

```

:::warning

如果标签（Tag）的名称是一个数字且等于某个快照 id，那么 VERSION AS OF 语法会优先考虑标签。例如，如果
你有一个基于快照 2 创建、名为 '1' 的标签，那么语句 `SELECT * FROM t VERSION AS OF '1'` 实际上查询的是快照 2，
而不是快照 1。

:::

### 批式增量查询 {#batch-incremental}

读取起始快照（不含）和结束快照之间的增量变更。

例如：
- '5,10' 表示快照 5 和快照 10 之间的变更。
- 'TAG1,TAG3' 表示 TAG1 和 TAG3 之间的变更。

默认情况下，对于会产生 changelog 文件的表，将扫描其 changelog 文件；否则，扫描新变更的文件。
你也可以强制指定 `'incremental-between-scan-mode'`。

Paimon 支持使用 Spark SQL 进行增量查询，该功能通过 Spark Table Valued Function（表值函数）实现。

```sql
-- read the incremental data between snapshot id 12 and snapshot id 20.
SELECT * FROM paimon_incremental_query('tableName', 12, 20);

-- read the incremental data between ts 1692169900000 and ts 1692169900000.
SELECT * FROM paimon_incremental_between_timestamp('tableName', '1692169000000', '1692169900000');
SELECT * FROM paimon_incremental_between_timestamp('tableName', '2025-03-12 00:00:00', '2025-03-12 00:08:00');

-- read the incremental data to tag '2024-12-04'.
-- Paimon will find an earlier tag and return changes between them.
-- If the tag doesn't exist or the earlier tag doesn't exist, return empty.
SELECT * FROM paimon_incremental_to_auto_tag('tableName', '2024-12-04');
```

在批式 SQL 中，不允许返回 `DELETE` 记录，因此 `-D` 类型的记录会被丢弃。
如果你想查看 `DELETE` 记录，可以查询 audit_log 表。

## 查询优化 {#query-optimization}

强烈建议在查询时同时指定分区和主键过滤条件，
这将加速查询中的数据跳过（data skipping）。

可以加速数据跳过的过滤函数有：
- `=`
- `<`
- `<=`
- `>`
- `>=`
- `IN (...)`
- `LIKE 'abc%'`
- `IS NULL`

Paimon 会按主键对数据进行排序，这会加速点查询和范围查询。
当使用复合主键时，最好让查询过滤条件构成主键的
[最左前缀（leftmost prefix）](https://dev.mysql.com/doc/refman/5.7/en/multiple-column-indexes.html)，
以获得良好的加速效果。

假设某张表具有以下规格：

```sql
CREATE TABLE orders (
    catalog_id BIGINT,
    order_id BIGINT,
    .....,
) TBLPROPERTIES (
    'primary-key' = 'catalog_id,order_id'
);
```

通过为主键的最左前缀指定范围过滤条件，查询可以获得良好的加速。

```sql
SELECT * FROM orders WHERE catalog_id=1025;

SELECT * FROM orders WHERE catalog_id=1025 AND order_id=29495;

SELECT * FROM orders
  WHERE catalog_id=1025
  AND order_id>2035 AND order_id<6000;
```

然而，下面的过滤条件无法很好地加速查询。

```sql
SELECT * FROM orders WHERE order_id=29495;

SELECT * FROM orders WHERE catalog_id=1025 OR order_id=29495;
```
