---
title: "SQL Write"
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

# SQL 写入 {#sql-write}

## 插入表 {#insert-table}

`INSERT` 语句用于向表中插入新行，或覆盖（Overwrite）表中的现有数据。被插入的行可以通过值表达式指定，也可以来自一个查询的结果。

**语法**

```sql
INSERT { INTO | OVERWRITE } table_identifier [ part_spec ] [ column_list ] { value_expr | query };
```
**参数**

- **table_identifier**：指定表名，可以选择性地用数据库名进行限定。

- **part_spec**：可选参数，为分区指定一组以逗号分隔的键值对列表。

- **column_list**：可选参数，指定一组以逗号分隔的、属于 table_identifier 表的列。Spark 会根据指定的列列表，重新排列输入查询的列，以匹配表的 schema（表结构）。

  注意：从 Spark 3.4 开始，如果 INSERT INTO 命令带有显式列列表，且其包含的列数少于目标表的列数，则会自动为其余列添加相应的默认值（对于没有显式指定默认值的列则填充 NULL）。在 Spark 3.3 或更早版本中，column_list 的大小必须等于目标表的列数，否则这些命令将会失败。

- **value_expr** ( { value | NULL } [ , … ] ) [ , ( … ) ]：指定要插入的值。可以插入显式指定的值，也可以插入 NULL。子句中每个值之间必须用逗号分隔。可以指定多组值以插入多行。

更多信息请查阅语法文档：[Spark INSERT 语句](https://spark.apache.org/docs/latest/sql-ref-syntax-dml-insert-table.html)

### Insert Into {#insert-into}

使用 `INSERT INTO` 向表中应用记录和变更。

```sql
INSERT INTO my_table SELECT ...
```

### Insert Overwrite {#insert-overwrite}

使用 `INSERT OVERWRITE` 覆盖整张表。

```sql
INSERT OVERWRITE my_table SELECT ...
```

#### Insert Overwrite Partition {#insert-overwrite-partition}

使用 `INSERT OVERWRITE` 覆盖某个分区。

```sql
INSERT OVERWRITE my_table PARTITION (key1 = value1, key2 = value2, ...) SELECT ...
```

#### Dynamic Overwrite Partition {#dynamic-overwrite-partition}

Spark 默认的覆盖模式是 `static`（静态）分区覆盖。要启用动态覆盖，你需要将 Spark 会话配置 `spark.sql.sources.partitionOverwriteMode` 设置为 `dynamic`。

例如：

```sql
CREATE TABLE my_table (id INT, pt STRING) PARTITIONED BY (pt);
INSERT INTO my_table VALUES (1, 'p1'), (2, 'p2');

-- Static overwrite (Overwrite the whole table)
INSERT OVERWRITE my_table VALUES (3, 'p1');
-- or 
INSERT OVERWRITE my_table PARTITION (pt) VALUES (3, 'p1');

SELECT * FROM my_table;
/*
+---+---+
| id| pt|
+---+---+
|  3| p1|
+---+---+
*/

-- Static overwrite with specified partitions (Only overwrite pt='p1')
INSERT OVERWRITE my_table PARTITION (pt='p1') VALUES (3);

SELECT * FROM my_table;
/*
+---+---+
| id| pt|
+---+---+
|  2| p2|
|  3| p1|
+---+---+
*/
  
-- Dynamic overwrite (Only overwrite pt='p1')
SET spark.sql.sources.partitionOverwriteMode=dynamic;
INSERT OVERWRITE my_table VALUES (3, 'p1');

SELECT * FROM my_table;
/*
+---+---+
| id| pt|
+---+---+
|  2| p2|
|  3| p1|
+---+---+
*/
```

## 清空表 {#truncate-table}

`TRUNCATE TABLE` 语句用于删除表或分区中的所有行。

```sql
TRUNCATE TABLE my_table;
```

## 更新表 {#update-table}

更新与谓词匹配的行的列值。当没有提供谓词时，更新所有行的列值。

注意：

:::info

当目标表是主键表时，不支持更新主键列。

:::

Spark 支持更新 PrimitiveType 和 StructType，例如：

```sql
-- Syntax
UPDATE table_identifier SET column1 = value1, column2 = value2, ... WHERE condition;

CREATE TABLE t (
  id INT, 
  s STRUCT<c1: INT, c2: STRING>, 
  name STRING)
TBLPROPERTIES (
  'primary-key' = 'id', 
  'merge-engine' = 'deduplicate'
);

-- you can use
UPDATE t SET name = 'a_new' WHERE id = 1;
UPDATE t SET s.c2 = 'a_new' WHERE s.c1 = 1;
```

## 从表中删除 {#delete-from-table}

删除与谓词匹配的行。当没有提供谓词时，删除所有行。

```sql
DELETE FROM my_table WHERE id = 1;
```

## Merge Into 表 {#merge-into-table}

基于源表，将一组更新、插入和删除操作合并到目标表中。

注意：

:::info

在 update 子句中，当目标表是主键表时，不支持更新主键列。

:::

**示例：一**

这是一个简单的演示：如果某行在目标表中已存在则更新它，否则插入它。

```sql
-- Here both source and target tables have the same schema: (a INT, b INT, c STRING), and a is a primary key.

MERGE INTO target
USING source
ON target.a = source.a
WHEN MATCHED THEN
UPDATE SET *
WHEN NOT MATCHED
THEN INSERT *
```

**示例：二**

这是一个带有多个条件子句的演示。

```sql
-- Here both source and target tables have the same schema: (a INT, b INT, c STRING), and a is a primary key.

MERGE INTO target
USING source
ON target.a = source.a
WHEN MATCHED AND target.a = 5 THEN
   UPDATE SET b = source.b + target.b      -- when matched and meet the condition 1, then update b;
WHEN MATCHED AND source.c > 'c2' THEN
   UPDATE SET *    -- when matched and meet the condition 2, then update all the columns;
WHEN MATCHED THEN
   DELETE      -- when matched, delete this row in target table;
WHEN NOT MATCHED AND c > 'c9' THEN
   INSERT (a, b, c) VALUES (a, b * 1.1, c)      -- when not matched but meet the condition 3, then transform and insert this row;
WHEN NOT MATCHED THEN
INSERT *      -- when not matched, insert this row without any transformation;
```

## Write Merge Schema {#write-merge-schema}

:::info

由于在写入过程中表的 schema（表结构）可能会被更新，使用该特性需要禁用 Catalog 缓存。请将 `spark.sql.catalog.<catalogName>.cache-enabled` 配置为 `false`。

:::

Write Merge Schema 是一项特性，它允许用户轻松地修改表的当前 schema，以适配现有数据或随时间变化的新数据，同时保持数据的完整性和一致性。

Paimon 支持在写入数据时自动合并源数据 schema 与当前表数据 schema，并将合并后的 schema 作为表的最新 schema，这只需要配置 `write.merge-schema`。

```scala
data.write
  .format("paimon")
  .mode("append")
  .option("write.merge-schema", "true")
  .save(location)
```

启用 `write.merge-schema` 后，Paimon 默认允许用户对表 schema 执行以下操作：
- 添加列
- 上转型（up-cast）列的类型（例如 Int -> Long）

Paimon 也支持某些类型之间的显式类型转换（例如 String -> Date、Long -> Int），这需要显式配置 `write.merge-schema.explicit-cast`。

Write Merge Schema 同时也可以在流式模式下使用。

```scala
val inputData = MemoryStream[(Int, String)]
inputData
  .toDS()
  .toDF("col1", "col2")
  .writeStream
  .format("paimon")
  .option("checkpointLocation", "/path/to/checkpoint")
  .option("write.merge-schema", "true")
  .option("write.merge-schema.explicit-cast", "true")
  .start(location)
```

以下列出相关配置。

<table class="configuration table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 20%">Scan Mode</th>
            <th class="text-left" style="width: 60%">描述</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><h5>write.merge-schema</h5></td>
            <td>若为 true，则在写入数据前自动合并数据 schema 与表 schema。</td>
        </tr>
        <tr>
            <td><h5>write.merge-schema.explicit-cast</h5></td>
            <td>若为 true，当两种类型满足显式转换规则时，允许合并这两种数据类型。</td>
        </tr>
    </tbody>
</table>

该模式同样支持 Spark SQL。以下是一个示例：

```sql
SET `spark.paimon.write.merge-schema` = true;

CREATE TABLE t (a INT, b STRING);
INSERT INTO t VALUES (1, '1'), (2, '2');

-- Need using `BY NAME` statement (requires Spark 3.5+)
INSERT INTO t BY NAME SELECT 3 AS a, '3' AS b, 3 AS c;
```
