---
title: "SQL Alter"
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

# 修改表 {#altering-tables}

## 修改/添加表属性 {#changingadding-table-properties}

以下 SQL 将表属性 `write-buffer-size` 设置为 `256 MB`。

```sql
ALTER TABLE my_table SET TBLPROPERTIES (
    'write-buffer-size' = '256 MB'
);
```

## 移除表属性 {#removing-table-properties}

以下 SQL 移除表属性 `write-buffer-size`。

```sql
ALTER TABLE my_table UNSET TBLPROPERTIES ('write-buffer-size');
```

##  修改/添加表注释 {#changingadding-table-comment}

以下 SQL 将表 `my_table` 的注释修改为 `table comment`。

```sql
ALTER TABLE my_table SET TBLPROPERTIES (
    'comment' = 'table comment'
    );
```

## 移除表注释 {#removing-table-comment}

以下 SQL 移除表注释。

```sql
ALTER TABLE my_table UNSET TBLPROPERTIES ('comment');
```

## 重命名表名 {#rename-table-name}

以下 SQL 将表名重命名为新名称。

最简单的调用 SQL 如下：
```sql
ALTER TABLE my_table RENAME TO my_table_new;
```

注意：我们可以在 Spark 中以这种方式重命名 Paimon 表：
```sql
ALTER TABLE [catalog.[database.]]test1 RENAME to [database.]test2;
```
但是我们不能在重命名后的目标表前面加上 Catalog 名称，如果写成下面这样的 SQL 会抛出错误：
```sql
ALTER TABLE catalog.database.test1 RENAME to catalog.database.test2;
```

:::info

如果你使用不带 REST Catalog 的对象存储（例如 S3 或 OSS），请谨慎使用此语法，因为对象存储的重命名不是原子操作，一旦失败可能只移动了部分文件。

:::

## 添加新列 {#adding-new-columns}

以下 SQL 向表 `my_table` 添加两个列 `c1` 和 `c2`。

```sql
ALTER TABLE my_table ADD COLUMNS (
    c1 INT,
    c2 STRING
);
```

以下 SQL 向 struct 类型添加一个嵌套列 `f3`。

```sql
-- column v previously has type STRUCT<f1: STRING, f2: INT>
ALTER TABLE my_table ADD COLUMN v.f3 STRING;
```

以下 SQL 向作为 array 类型元素类型的 struct 类型添加一个嵌套列 `f3`。

```sql
-- column v previously has type ARRAY<STRUCT<f1: STRING, f2: INT>>
ALTER TABLE my_table ADD COLUMN v.element.f3 STRING;
```

以下 SQL 向作为 map 类型值类型的 struct 类型添加一个嵌套列 `f3`。

```sql
-- column v previously has type MAP<INT, STRUCT<f1: STRING, f2: INT>>
ALTER TABLE my_table ADD COLUMN v.value.f3 STRING;
```

## 重命名列名 {#renaming-column-name}

以下 SQL 将表 `my_table` 中的列 `c0` 重命名为 `c1`。

```sql
ALTER TABLE my_table RENAME COLUMN c0 TO c1;
```

以下 SQL 将 struct 类型中的嵌套列 `f1` 重命名为 `f100`。

```sql
-- column v previously has type STRUCT<f1: STRING, f2: INT>
ALTER TABLE my_table RENAME COLUMN v.f1 to f100;
```

以下 SQL 将作为 array 类型元素类型的 struct 类型中的嵌套列 `f1` 重命名为 `f100`。

```sql
-- column v previously has type ARRAY<STRUCT<f1: STRING, f2: INT>>
ALTER TABLE my_table RENAME COLUMN v.element.f1 to f100;
```

以下 SQL 将作为 map 类型值类型的 struct 类型中的嵌套列 `f1` 重命名为 `f100`。

```sql
-- column v previously has type MAP<INT, STRUCT<f1: STRING, f2: INT>>
ALTER TABLE my_table RENAME COLUMN v.value.f1 to f100;
```

## 删除列 {#dropping-columns}

以下 SQL 从表 `my_table` 中删除两个列 `c1` 和 `c2`。

```sql
ALTER TABLE my_table DROP COLUMNS (c1, c2);
```

以下 SQL 从 struct 类型中删除一个嵌套列 `f2`。

```sql
-- column v previously has type STRUCT<f1: STRING, f2: INT>
ALTER TABLE my_table DROP COLUMN v.f2;
```

以下 SQL 从作为 array 类型元素类型的 struct 类型中删除一个嵌套列 `f2`。

```sql
-- column v previously has type ARRAY<STRUCT<f1: STRING, f2: INT>>
ALTER TABLE my_table DROP COLUMN v.element.f2;
```

以下 SQL 从作为 map 类型值类型的 struct 类型中删除一个嵌套列 `f2`。

```sql
-- column v previously has type MAP<INT, STRUCT<f1: STRING, f2: INT>>
ALTER TABLE my_table DROP COLUMN v.value.f2;
```

在 Hive Catalog 中，你需要确保：

1. 在你的 Hive 服务器中禁用 `hive.metastore.disallow.incompatible.col.type.changes`；
2. 或者在你的 Spark 中使用 `spark-sql --conf spark.hadoop.hive.metastore.disallow.incompatible.col.type.changes=false`。

否则此操作可能失败，抛出类似 `The following columns have types incompatible with the
existing columns in their respective positions` 的异常。

## 删除分区 {#dropping-partitions}

以下 SQL 删除 Paimon 表的分区。对于 Spark SQL，你需要指定所有分区列。

```sql
ALTER TABLE my_table DROP PARTITION (`id` = 1, `name` = 'paimon');
```

## 修改列注释 {#changing-column-comment}

以下 SQL 将列 `buy_count` 的注释修改为 `buy count`。

```sql
ALTER TABLE my_table ALTER COLUMN buy_count COMMENT 'buy count';
```

## 添加列位置 {#adding-column-position}

```sql
ALTER TABLE my_table ADD COLUMN c INT FIRST;

ALTER TABLE my_table ADD COLUMN c INT AFTER b;
```

## 修改列位置 {#changing-column-position}

```sql
ALTER TABLE my_table ALTER COLUMN col_a FIRST;

ALTER TABLE my_table ALTER COLUMN col_a AFTER col_b;
```

## 修改列类型 {#changing-column-type}

```sql
ALTER TABLE my_table ALTER COLUMN col_a TYPE DOUBLE;
```

以下 SQL 将 struct 类型中嵌套列 `f2` 的类型修改为 `BIGINT`。

```sql
-- column v previously has type STRUCT<f1: STRING, f2: INT>
ALTER TABLE my_table ALTER COLUMN v.f2 TYPE BIGINT;
```

以下 SQL 将作为 array 类型元素类型的 struct 类型中嵌套列 `f2` 的类型修改为 `BIGINT`。

```sql
-- column v previously has type ARRAY<STRUCT<f1: STRING, f2: INT>>
ALTER TABLE my_table ALTER COLUMN v.element.f2 TYPE BIGINT;
```

以下 SQL 将作为 map 类型值类型的 struct 类型中嵌套列 `f2` 的类型修改为 `BIGINT`。

```sql
-- column v previously has type MAP<INT, STRUCT<f1: STRING, f2: INT>>
ALTER TABLE my_table ALTER COLUMN v.value.f2 TYPE BIGINT;
```


# ALTER DATABASE {#alter-database}

以下 SQL 在指定数据库中设置一个或多个属性。如果某个属性已在数据库中设置，则用新值覆盖旧值。

```sql
ALTER { DATABASE | SCHEMA | NAMESPACE } my_database
    SET { DBPROPERTIES | PROPERTIES } ( property_name = property_value [ , ... ] )
```

## 修改数据库位置 {#altering-database-location}

以下 SQL 将指定数据库的位置设置为 `file:/temp/my_database.db`。

```sql
ALTER DATABASE my_database SET LOCATION 'file:/temp/my_database.db'
```
