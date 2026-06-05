---
title: "SQL Alter"
sidebar_position: 7
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
ALTER TABLE my_table SET (
    'write-buffer-size' = '256 MB'
);
```

## 移除表属性 {#removing-table-properties}

以下 SQL 移除表属性 `write-buffer-size`。

```sql
ALTER TABLE my_table RESET ('write-buffer-size');
```

##  修改/添加表注释 {#changingadding-table-comment}

以下 SQL 将表 `my_table` 的注释修改为 `table comment`。

```sql
ALTER TABLE my_table SET (
    'comment' = 'table comment'
    );
```

## 移除表注释 {#removing-table-comment}

以下 SQL 移除表注释。

```sql
ALTER TABLE my_table RESET ('comment');
```

## 重命名表名 {#rename-table-name}

以下 SQL 将表名重命名为新名称。

```sql
ALTER TABLE my_table RENAME TO my_table_new;
```

:::info

如果你使用未配合 REST Catalog 的对象存储（如 S3 或 OSS），请谨慎使用此语法，因为对象存储的重命名不是原子操作，一旦失败可能只移动了部分文件。

:::

## 添加新列 {#adding-new-columns}

以下 SQL 向表 `my_table` 添加两列 `c1` 和 `c2`。

:::info

要在 row 类型中添加列，请参见 [修改列类型](#changing-column-type)。

:::

```sql
ALTER TABLE my_table ADD (c1 INT, c2 STRING);
```

## 重命名列名 {#renaming-column-name}

以下 SQL 将表 `my_table` 中的列 `c0` 重命名为 `c1`。

```sql
ALTER TABLE my_table RENAME c0 TO c1;
```

## 删除列 {#dropping-columns}

以下 SQL 从表 `my_table` 中删除两列 `c1` 和 `c2`。

```sql
ALTER TABLE my_table DROP (c1, c2);
```

:::info

要删除 row 类型中的列，请参见 [修改列类型](#changing-column-type)。

:::

在 hive catalog 中，你需要确保：

1. 在你的 hive server 中禁用 `hive.metastore.disallow.incompatible.col.type.changes`
2. 或者在你的 paimon catalog 中设置 `hadoop.hive.metastore.disallow.incompatible.col.type.changes=false`。

否则此操作可能失败，抛出类似 `The following columns have types incompatible with the
existing columns in their respective positions` 的异常。

## 删除分区 {#dropping-partitions}

以下 SQL 删除 paimon 表的分区。

对于 flink sql，你可以指定分区列的部分列，也可以同时指定多个分区值。

```sql
ALTER TABLE my_table DROP PARTITION (`id` = 1);

ALTER TABLE my_table DROP PARTITION (`id` = 1, `name` = 'paimon');

ALTER TABLE my_table DROP PARTITION (`id` = 1), PARTITION (`id` = 2);

```

## 修改列可空性 {#changing-column-nullability}

以下 SQL 修改列 `coupon_info` 的可空性。

```sql
CREATE TABLE my_table (id INT PRIMARY KEY NOT ENFORCED, coupon_info FLOAT NOT NULL);

-- Change column `coupon_info` from NOT NULL to nullable
ALTER TABLE my_table MODIFY coupon_info FLOAT;

-- Change column `coupon_info` from nullable to NOT NULL
-- If there are NULL values already, set table option as below to drop those records silently before altering table.
SET 'table.exec.sink.not-null-enforcer' = 'DROP';
ALTER TABLE my_table MODIFY coupon_info FLOAT NOT NULL;
```

:::info

将可空列修改为 NOT NULL 目前仅 Flink 支持。

:::

## 修改列注释 {#changing-column-comment}

以下 SQL 将列 `buy_count` 的注释修改为 `buy count`。

```sql
ALTER TABLE my_table MODIFY buy_count BIGINT COMMENT 'buy count';
```

## 添加列位置 {#adding-column-position}

要在指定位置添加新列，请使用 FIRST 或 AFTER col_name。

```sql
ALTER TABLE my_table ADD c INT FIRST;

ALTER TABLE my_table ADD c INT AFTER b;
```

## 修改列位置 {#changing-column-position}

要将已有列修改到新位置，请使用 FIRST 或 AFTER col_name。

```sql
ALTER TABLE my_table MODIFY col_a DOUBLE FIRST;

ALTER TABLE my_table MODIFY col_a DOUBLE AFTER col_b;
```

## 修改列类型 {#changing-column-type}

以下 SQL 将列 `col_a` 的类型修改为 `DOUBLE`。

```sql
ALTER TABLE my_table MODIFY col_a DOUBLE;
```

Paimon 也支持修改 row 类型、array 类型和 map 类型的列。

```sql
-- col_a previously has type ARRAY<MAP<INT, ROW(f1 INT, f2 STRING)>>
-- the following SQL changes f1 to BIGINT, drops f2, and adds f3
ALTER TABLE my_table MODIFY col_a ARRAY<MAP<INT, ROW(f1 BIGINT, f3 DOUBLE)>>;
```

## 添加 Watermark {#adding-watermark}

以下 SQL 基于已有列 `log_ts` 添加计算列 `ts`，并在列 `ts` 上添加策略为 `ts - INTERVAL '1' HOUR` 的 Watermark，将该列标记为表 `my_table` 的事件时间属性。

```sql
ALTER TABLE my_table ADD (
    ts AS TO_TIMESTAMP(log_ts) AFTER log_ts,
    WATERMARK FOR ts AS ts - INTERVAL '1' HOUR
);
```

## 删除 Watermark {#dropping-watermark}

以下 SQL 删除表 `my_table` 的 Watermark。

```sql
ALTER TABLE my_table DROP WATERMARK;
```

## 修改 Watermark {#changing-watermark}

以下 SQL 将 Watermark 策略修改为 `ts - INTERVAL '2' HOUR`。

```sql
ALTER TABLE my_table MODIFY WATERMARK FOR ts AS ts - INTERVAL '2' HOUR;
```

# ALTER DATABASE {#alter-database}

以下 SQL 在指定的数据库中设置一个或多个属性。如果某个属性已在数据库中设置，则用新值覆盖旧值。

```sql
ALTER DATABASE [catalog_name.]db_name SET (key1=val1, key2=val2, ...);
```

## 修改数据库位置 {#altering-database-location}

以下 SQL 将数据库 `my_database` 的位置修改为 `file:/temp/my_database`。

```sql
ALTER DATABASE my_database SET ('location' =  'file:/temp/my_database');
```
