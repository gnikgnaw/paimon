---
title: "Row Tracking"
sidebar_position: 5
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

# 行跟踪（Row tracking） {#row-tracking}

行跟踪（Row tracking）允许 Paimon 在 Paimon Append 表（追加表）中进行行级别的跟踪。一旦在某个 Paimon 表上启用，表 schema（表结构）中会额外添加两个隐藏列：
- `_ROW_ID`：BIGINT，这是表中每一行的唯一标识符。它用于跟踪行的更新，并可在更新、merge into 或删除时用于标识该行。
- `_SEQUENCE_NUMBER`：BIGINT，该字段指示这条记录的 `version`（版本）。它实际上是该行所属快照的 snapshot-id。它用于跟踪行版本的更新。

隐藏列遵循以下规则：
- 每当我们从启用了行跟踪的表中读取数据时，`_ROW_ID` 和 `_SEQUENCE_NUMBER` 都将是 `NOT NULL`。
- 如果我们第一次向行跟踪表追加记录，实际上并不会把它们写入数据文件，而是由提交者（committer）延迟赋值。
- 如果某一行由于**任何原因**从一个文件移动到另一个文件，`_ROW_ID` 列都应被复制到目标文件。如果记录发生了变更，`_SEQUENCE_NUMBER` 字段应被设置为 `NULL`，否则同样复制它。
- 每当我们从行跟踪表读取数据时，我们首先从数据文件中读取 `_ROW_ID` 和 `_SEQUENCE_NUMBER`，然后从数据文件中读取值列。如果发现它们为 `NULL`，则从 `DataFileMeta` 读取以回退到延迟赋值的值。无论如何，它都不可能为 `NULL`。

要启用行跟踪，你必须在创建 Append 表时，在表的配置项中将 `row-tracking.enabled` 配置为 `true`。
来看一个通过 Flink SQL 实现的示例：
```sql
CREATE TABLE part_t (
    f0 INT,
    f1 STRING,
    dt STRING
) PARTITIONED BY (dt)
WITH ('row-tracking.enabled' = 'true');
```
请注意：
- 行跟踪仅支持 unaware 的 Append 表，不支持主键表。这意味着你不能为该表定义 `bucket` 和 `bucket-key`。
- 仅 Spark 支持对行跟踪表执行更新、merge into 和删除操作，Flink SQL 尚不支持这些操作。
- 此功能为实验性功能，此行说明将在功能稳定后移除。

创建行跟踪表后，你可以像往常一样向其中插入数据。`_ROW_ID` 和 `_SEQUENCE_NUMBER` 列将由 Paimon 自动管理。
```sql
CREATE TABLE t (id INT, data STRING) TBLPROPERTIES ('row-tracking.enabled' = 'true');
INSERT INTO t VALUES (11, 'a'), (22, 'b')
```

你可以在 Spark 中使用以下 SQL 选择行跟踪元数据列：
```sql
SELECT id, data, _ROW_ID, _SEQUENCE_NUMBER FROM t;
```
你将得到如下结果：
```text
+---+----+-------+----------------+
| id|data|_ROW_ID|_SEQUENCE_NUMBER|
+---+----+-------+----------------+
| 11|   a|      0|               1|
| 22|   b|      1|               1|
+---+----+-------+----------------+
```

然后你可以更新表并再次查询：
```sql
UPDATE t SET data = 'new-data-update' WHERE id = 11;
-- Alternatively, update using the hidden row id `_ROW_ID`
UPDATE t SET data = 'new-data-update' WHERE _ROW_ID = 0;
SELECT id, data, _ROW_ID, _SEQUENCE_NUMBER FROM t;
```

你将得到：
```text
+---+---------------+-------+----------------+
| id|           data|_ROW_ID|_SEQUENCE_NUMBER|
+---+---------------+-------+----------------+
| 22|              b|      1|               1|
| 11|new-data-update|      0|               2|
+---+---------------+-------+----------------+
```

你也可以对表执行 merge into，假设你有一个源表 `s`，其中包含 (22, 'new-data-merge') 和 (33, 'c')：
```sql
MERGE INTO t USING s
ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.data = s.data
WHEN NOT MATCHED THEN INSERT *;
```

你将得到：
```text
+---+---------------+-------+----------------+
| id|           data|_ROW_ID|_SEQUENCE_NUMBER|
+---+---------------+-------+----------------+
| 11|new-data-update|      0|               2|
| 22| new-data-merge|      1|               3|
| 33|              c|      2|               3|
+---+---------------+-------+----------------+
```

你也可以从表中删除数据：

```sql
DELETE FROM t WHERE id = 11;
-- Alternatively, delete using the hidden row id `_ROW_ID`
DELETE FROM t WHERE _ROW_ID = 0;
```

你将得到：
```text
+---+---------------+-------+----------------+
| id|           data|_ROW_ID|_SEQUENCE_NUMBER|
+---+---------------+-------+----------------+
| 22| new-data-merge|      1|               3|
| 33|              c|      2|               3|
+---+---------------+-------+----------------+
```
