---
title: "默认值"
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

# 默认值 {#default-value}

Paimon 允许为列指定默认值。当用户写入这些表时，如果没有为某些列显式提供值，Paimon 会自动为这些列生成默认值。

## 创建表 {#create-table}

你可以使用以下 SQL 创建一个带有默认值列的表：

```sql
CREATE TABLE my_table (
    a BIGINT,
    b STRING DEFAULT 'my_value',
    c INT DEFAULT 5,
    tags ARRAY<STRING> DEFAULT ARRAY('tag1', 'tag2', 'tag3'),
    properties MAP<STRING, STRING> DEFAULT MAP('key1', 'value1', 'key2', 'value2'),
    nested STRUCT<x: INT, y: STRING> DEFAULT STRUCT(42, 'default_value')
);
```

## 插入表 {#insert-table}

对于执行表写入的 SQL 命令，例如 `INSERT`、`UPDATE` 和 `MERGE` 命令，`DEFAULT` 关键字或 `NULL` 值会被解析为对应列所指定的默认值。

例如：

```sql
INSERT INTO my_table (a) VALUES (1), (2);

SELECT * FROM my_table;
-- result: [[1, my_value, 5, [tag1, tag2, tag3], {'key1' -> 'value1', 'key2' -> 'value2'}, {42, default_value}],
--          [2, my_value, 5, [tag1, tag2, tag3], {'key1' -> 'value1', 'key2' -> 'value2'}, {42, default_value}]]
```

## 修改默认值 {#alter-default-value}

Paimon 支持修改列的默认值。

例如：

```sql
CREATE TABLE T (a INT, b INT DEFAULT 2);

INSERT INTO T (a) VALUES (1);
-- result: [[1, 2]]

ALTER TABLE T ALTER COLUMN b SET DEFAULT 3;

INSERT INTO T (a) VALUES (2);
-- result: [[1, 2], [2, 3]]
```

`'b'` 列的默认值已经从 2 改为 3。你也可以修改复杂类型的默认值：

```sql
ALTER TABLE my_table ALTER COLUMN tags SET DEFAULT ARRAY('new_tag1', 'new_tag2');

INSERT INTO my_table (a) VALUES (3);
-- result: [[1, my_value, 5, [tag1, tag2, tag3], {'key1' -> 'value1', 'key2' -> 'value2'}, {42, default_value}],
--          [2, my_value, 5, [tag1, tag2, tag3], {'key1' -> 'value1', 'key2' -> 'value2'}, {42, default_value}],
--          [3, my_value, 5, [new_tag1, new_tag2], {'key1' -> 'value1', 'key2' -> 'value2'}, {42, default_value}]]

ALTER TABLE my_table ALTER COLUMN properties SET DEFAULT MAP('new_key', 'new_value');

INSERT INTO my_table (a) VALUES (4);
-- result: [[1, my_value, 5, [tag1, tag2, tag3], {'key1' -> 'value1', 'key2' -> 'value2'}, {42, default_value}],
--          [2, my_value, 5, [tag1, tag2, tag3], {'key1' -> 'value1', 'key2' -> 'value2'}, {42, default_value}],
--          [3, my_value, 5, [new_tag1, new_tag2], {'key1' -> 'value1', 'key2' -> 'value2'}, {42, default_value}],
--          [4, my_value, 5, [new_tag1, new_tag2], {'new_key' -> 'new_value'}, {42, default_value}]]
```

## 限制 {#limitation}

不支持在 alter table 添加列时指定默认值，例如：`ALTER TABLE T ADD COLUMN d INT DEFAULT 5;`。
