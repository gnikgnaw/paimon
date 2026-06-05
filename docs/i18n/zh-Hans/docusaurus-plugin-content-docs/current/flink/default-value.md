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

Flink SQL 原生并不支持默认值，因此我们只能创建一个不带默认值的表：

```sql
CREATE TABLE my_table (
    a BIGINT,
    b STRING,
    c INT,
    tags ARRAY<STRING>,
    properties MAP<STRING, STRING>,
    nested ROW<x INT, y STRING>
);
```

我们支持在 Flink 中通过存储过程修改列的默认值。你可以在创建表之后再添加默认值定义：

```sql
-- Set simple type default values
CALL sys.alter_column_default_value('default.my_table', 'b', 'my_value');
CALL sys.alter_column_default_value('default.my_table', 'c', '5');

-- Set complex type default values
CALL sys.alter_column_default_value('default.my_table', 'tags', '[tag1, tag2, tag3]');
CALL sys.alter_column_default_value('default.my_table', 'properties', '{key1 -> value1, key2 -> value2}');
CALL sys.alter_column_default_value('default.my_table', 'nested', '{42, default_value}');
```

## 插入表 {#insert-table}

对于执行表写入的 SQL 命令，例如 `INSERT`、`UPDATE` 和 `MERGE` 命令，`NULL` 值会被解析为对应列所指定的默认值。

例如：

```sql
INSERT INTO my_table (a) VALUES (1), (2);

SELECT * FROM my_table;
-- result: [[1, my_value, 5, [tag1, tag2, tag3], {key1=value1, key2=value2}, +I[42, default_value]],
--          [2, my_value, 5, [tag1, tag2, tag3], {key1=value1, key2=value2}, +I[42, default_value]]]
```
