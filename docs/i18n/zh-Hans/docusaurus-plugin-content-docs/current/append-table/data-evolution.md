---
title: "数据演进"
sidebar_position: 6
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

# 数据演进 {#data-evolution}

## 概述 {#overview}

Paimon 支持完整的 schema 演进（schema evolution），允许你自由地添加、修改或删除列结构。但是该如何回填新增的列，或者更新列数据呢？

数据演进模式（Data Evolution Mode）是 Append 表（追加表）的一项新特性，它彻底改变了你处理数据演进的方式，尤其是在添加新列时。该模式允许你在不重写整个数据文件的前提下更新部分列。它会把新的列数据写入独立的文件，并在读取操作时智能地将其与原始数据合并。

数据演进模式为你的数据湖架构带来了显著的优势：

* 高效的部分列更新：借助该模式，你可以使用 Spark 的 MERGE INTO 语句更新一部分列。这避免了重写整个文件带来的高 I/O 成本，因为只有被更新的列才会被写入。

* 减少文件重写：在频繁变更 schema 的场景下（例如添加新列），传统方式需要不断重写文件。数据演进模式通过将新的列数据追加到专用文件中，消除了这种开销。这种方式高效得多，并减轻了存储系统的负担。

* 优化的读取性能：新模式专为无缝的数据检索而设计。在查询执行期间，Paimon 的引擎会高效地将原始数据与新的列数据组合在一起，确保读取性能不受损害。合并过程经过高度优化，因此你的查询运行速度与在单个合并后的文件上运行时一样快。

要启用数据演进，你必须启用行追踪（row-tracking），并在创建 Append 表时将 `row-tracking.enabled` 和 `data-evolution.enabled` 属性设置为 `true`。这能确保该表已为高效的 schema 演进操作做好准备。

以 Spark Sql 为例：

```sql
CREATE TABLE target_table (id INT, b INT, c INT) TBLPROPERTIES (
    'row-tracking.enabled' = 'true',
    'data-evolution.enabled' = 'true'
);

INSERT INTO target_table VALUES (1, 1, 1), (2, 2, 2);
```

现在我们可以通过 spark 的 'MERGE INTO' 语句或 flink 的 'data_evolution_merge_into' 存储过程（Procedure）来更新部分列：

### Spark {#spark}

```sql
CREATE TABLE source_table (id INT, b INT);
INSERT INTO source_table VALUES (1, 11), (2, 22), (3, 33);

MERGE INTO target_table AS t
USING source_table AS s
ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.b = s.b
WHEN NOT MATCHED THEN INSERT (id, b, c) VALUES (id, b, 0);

SELECT * FROM target_table;
+----+----+----+
| id | b  | c  |
+----+----+----+
| 1  | 11 | 1  |
| 2  | 22 | 2  |
| 3  | 33 | 0  |
```

该语句仅根据来自源表 `source_table` 的匹配记录，更新目标表 `target_table` 中的 `b` 列。`id` 列和 `c` 列保持不变，新记录则以指定的值插入。与未启用数据演进的表相比，区别在于只有 `b` 列的数据会被写入新文件。

请注意：
* 数据演进表暂不支持 'Delete' 和 'Update' 语句。
* 数据演进表的 Merge Into 不支持 'WHEN NOT MATCHED BY SOURCE' 子句。

### Flink {#flink}
由于 Flink 目前不支持 MERGE INTO 语法，我们使用 data_evolution_merge_into 存储过程来模拟 merge-into 流程，如下所示：

```sql
CREATE TABLE source_table (id INT, b INT);
INSERT INTO source_table VALUES (1, 11), (2, 22), (3, 33);

CALL sys.data_evolution_merge_into(
    'my_db.target_table', 
    '',   /* Optional target alias */
    '',   /* Optional source sqls */
    'source_table',
    'source_table.id=target_table.id',
    'b=source_table.b',
    2     /* Specify sink parallelism */
);

SELECT * FROM source_table
+----+----+----+
| id | b  | c  |
+----+----+----+
| 1  | 11 | 1  |
| 2  | 22 | 2  |
```
请注意：
* 与 Spark 的实现相比，Flink 的 data_evolution_merge_into 存储过程目前仅支持更新/插入新列。暂不支持插入新行。

#### 自合并（Self Merge） {#self-merge}

自合并（Self-merge）指的是合并操作的源与目标是**同一张表**的情况。当你想要就地转换已有的列值时（例如，应用一个 UDF 来重写某一列），这会非常有用。

由于源表不能直接与目标表相同，你需要基于系统表 `T$row_tracking`（它会暴露隐藏的 `_ROW_ID` 列）创建一个**临时视图（temporary view）**，并使用 `_ROW_ID` 作为合并条件。

```sql
-- 1. Register a UDF
CREATE TEMPORARY FUNCTION concat_string AS 'com.example.StringConcatUdf';

-- 2. Create a view from the row-tracking system table
CREATE TEMPORARY VIEW source_view AS
SELECT _ROW_ID, concat_string(name) AS name
FROM my_db.target_table$row_tracking;

-- 3. Self-merge: update the name column using the UDF result
CALL sys.data_evolution_merge_into(
    'my_db.target_table',
    'TempT',
    -- alternatively, you could also pass the create sqls in procedure directly
    -- like: 'CREATE TEMPORARY FUNCTION concat_string AS ''com.example.StringConcatUdf''; CREATE TEMPORARY VIEW XXX'
    '',
    'source_view',
    'TempT._ROW_ID=source_view._ROW_ID',
    'name=source_view.name',
    2
);
```

请注意：
* 源表与目标表的名称不能相同。你必须创建一个临时视图作为源。
* 使用 `view._ROW_ID` = `source._ROW_ID` 来标识自合并模式。
* `_ROW_ID` 仅可通过 `$row_tracking` 系统表获取。
* 自合并仅支持 `WHEN MATCHED THEN UPDATE` 语义。

## 文件组规范（File Group Spec） {#file-group-spec}

通过 RowId 元数据，文件被组织成一个文件组（file group）。

写入时：数据演进表的 MERGE INTO 子句仅更新指定的列，并将更新后的列数据写入新文件。原始数据文件保持不变。

读取时：Paimon 会同时读取原始数据文件和包含更新列数据的新文件，然后合并这两个数据源的数据，以呈现该表的统一视图。这一合并过程经过优化，以确保读取性能不会受到显著影响。

写入之后，`target_table` 中的文件如下所示：

![](/img/data-evolution.png)

读取时，具有相同 `first row id` 的文件将进行字段合并。

![](/img/data-evolution2.png)

该模式的优势在于：

* 更新部分列时避免重写整个文件，从而降低 I/O 成本。
* 由于合并过程经过优化，读取性能不会受到显著影响。
* 磁盘空间使用更高效，因为只有被更新的列才会写入新文件。
