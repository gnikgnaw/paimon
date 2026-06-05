---
title: "部分更新"
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

# 部分更新（Partial Update） {#partial-update}

通过指定 `'merge-engine' = 'partial-update'`，用户能够通过多次更新来逐步更新一条记录的各个列，直到该记录完整为止。其实现方式是在相同主键下，使用最新数据逐个更新各个值字段。但在此过程中，null 值不会覆盖已有值。

例如，假设 Paimon 收到三条记录：

- `<1, 23.0, 10, NULL>`-
- `<1, NULL, NULL, 'This is a book'>`
- `<1, 25.2, NULL, NULL>`

假设第一列是主键，则最终结果为 `<1, 25.2, 10, 'This is a book'>`。

:::info

对于流式查询，`partial-update` 合并引擎必须与 `lookup` 或 `full-compaction`
[changelog producer](../changelog-producer) 配合使用。（也支持 'input' changelog producer，
但它只返回输入记录。）

:::

:::info

默认情况下，部分更新无法接受删除记录，你可以选择以下解决方案之一：

- 配置 'ignore-delete' 以忽略删除记录。
- 配置 'partial-update.remove-record-on-delete'，在收到删除记录时移除整行。
- 配置 'sequence-group' 以回撤部分列。同时配置 'partial-update.remove-record-on-sequence-group'，在收到 `specified sequence group` 的删除记录时移除整行。

:::

## 序列组（Sequence Group） {#sequence-group}

对于具有多条流更新的部分更新表，单个序列字段（sequence field）可能无法解决乱序问题，因为
在多流更新过程中，该序列字段可能被另一条流的最新数据覆盖。

因此，我们为部分更新表引入了序列组（sequence group）机制。它可以解决：

1. 多流更新过程中的乱序问题。每条流定义自己的序列组。
2. 实现真正的部分更新，而不仅仅是非空更新。

参见示例：

```sql
CREATE TABLE t
(
    k   INT,
    a   INT,
    b   INT,
    g_1 INT,
    c   INT,
    d   INT,
    g_2 INT,
    PRIMARY KEY (k) NOT ENFORCED
) WITH (
      'merge-engine' = 'partial-update',
      'fields.g_1.sequence-group' = 'a,b',
      'fields.g_2.sequence-group' = 'c,d'
      );

INSERT INTO t
VALUES (1, 1, 1, 1, 1, 1, 1);

-- g_2 is null, c, d should not be updated
INSERT INTO t
VALUES (1, 2, 2, 2, 2, 2, CAST(NULL AS INT));

SELECT *
FROM t;
-- output 1, 2, 2, 2, 1, 1, 1

-- g_1 is smaller, a, b should not be updated
INSERT INTO t
VALUES (1, 3, 3, 1, 3, 3, 3);

SELECT *
FROM t; -- output 1, 2, 2, 2, 3, 3, 3
```

对于 `fields.<field-name>.sequence-group`，有效的可比较数据类型包括：DECIMAL、TINYINT、SMALLINT、INTEGER、
BIGINT、FLOAT、DOUBLE、DATE、TIME、TIMESTAMP 和 TIMESTAMP_LTZ。

你也可以在一个 `sequence-group` 中配置多个排序字段，
例如 `fields.<field-name1>,<field-name2>.sequence-group`，多个字段将按顺序进行比较。

参见示例：

```sql
CREATE TABLE SG
(
    k   INT,
    a   INT,
    b   INT,
    g_1 INT,
    c   INT,
    d   INT,
    g_2 INT,
    g_3 INT,
    PRIMARY KEY (k) NOT ENFORCED
) WITH (
      'merge-engine' = 'partial-update',
      'fields.g_1.sequence-group' = 'a,b',
      'fields.g_2,g_3.sequence-group' = 'c,d'
      );

INSERT INTO SG
VALUES (1, 1, 1, 1, 1, 1, 1, 1);

-- g_3 is null, g_2, g_3 are not bigger, c, d should not be updated
INSERT INTO SG
VALUES (1, 2, 2, 2, 2, 2, 1, CAST(NULL AS INT));

SELECT *
FROM SG;
-- output 1, 2, 2, 2, 1, 1, 1, 1

-- g_1 is smaller, a, b should not be updated
INSERT INTO SG
VALUES (1, 3, 3, 1, 3, 3, 3, 1);

SELECT *
FROM SG;
-- output 1, 2, 2, 2, 3, 3, 3, 1
```

## 部分更新中的聚合（Aggregation For Partial Update） {#aggregation-for-partial-update}

你可以为输入字段指定聚合（aggregation）函数，[聚合](./aggregation) 中的所有函数都受支持。

:::info

**当涉及聚合函数时，序列组（sequence-group）的行为会发生变化。**

在不使用聚合函数的情况下，序列组字段充当**版本过滤器**：对于关联的列，序列值不超过已存储值的
传入记录会被忽略。

在使用聚合函数的情况下，序列组字段充当**排序键**：每一条序列值非 NULL 的传入记录都会参与聚合，
无论其序列值比已存储值更大还是更小。仅当传入值更大时，已存储的序列值才会向前推进。
对于与顺序无关的函数（`sum`、`product`、`max`、`min`），顺序对结果没有影响；
对于与顺序相关的函数（`last_non_null_value`、`first_value`、`listagg`），
序列组的值决定了哪一条记录的贡献被视为“最后一条”或“第一条”。

序列组值为 NULL 的记录始终会被跳过。

:::

参见示例：

```sql
CREATE TABLE t
(
    k INT,
    a INT,
    b INT,
    c INT,
    d INT,
    PRIMARY KEY (k) NOT ENFORCED
) WITH (
      'merge-engine' = 'partial-update',
      'fields.a.sequence-group' = 'b',
      'fields.b.aggregate-function' = 'first_value',
      'fields.c.sequence-group' = 'd',
      'fields.d.aggregate-function' = 'sum'
      );
INSERT INTO t
VALUES (1, 1, 1, CAST(NULL AS INT), CAST(NULL AS INT));
INSERT INTO t
VALUES (1, CAST(NULL AS INT), CAST(NULL AS INT), 1, 1);
INSERT INTO t
VALUES (1, 2, 2, CAST(NULL AS INT), CAST(NULL AS INT));
INSERT INTO t
VALUES (1, CAST(NULL AS INT), CAST(NULL AS INT), 2, 2);


SELECT *
FROM t; -- output 1, 2, 1, 2, 3
```

你也可以为包含多个排序字段的 `sequence-group` 配置聚合函数。

参见示例：

```sql
CREATE TABLE AGG
(
    k   INT,
    a   INT,
    b   INT,
    g_1 INT,
    c   VARCHAR,
    g_2 INT,
    g_3 INT,
    PRIMARY KEY (k) NOT ENFORCED
) WITH (
      'merge-engine' = 'partial-update',
      'fields.a.aggregate-function' = 'sum',
      'fields.g_1,g_3.sequence-group' = 'a',
      'fields.g_2.sequence-group' = 'c');
-- a in sequence-group g_1, g_3 with sum agg
-- b not in sequence-group
-- c in sequence-group g_2 without agg

INSERT INTO AGG
VALUES (1, 1, 1, 1, '1', 1, 1);

-- g_2 is null, c should not be updated
INSERT INTO AGG
VALUES (1, 2, 2, 2, '2', CAST(NULL AS INT), 2);

SELECT *
FROM AGG;
-- output 1, 3, 2, 2, "1", 1, 2

-- (g_1, g_3) = (2, 1) is smaller than stored (2, 2), so the stored sequence values are not advanced,
-- but the sum aggregate for a still applies: a = 3 + 3 = 6
INSERT INTO AGG
VALUES (1, 3, 3, 2, '3', 3, 1);

SELECT *
FROM AGG;
-- output 1, 6, 3, 2, "3", 3, 2
```

你可以使用 `fields.default-aggregate-function` 为所有输入字段指定默认聚合函数，参见
示例：

```sql
CREATE TABLE t
(
    k INT,
    a INT,
    b INT,
    c INT,
    d INT,
    PRIMARY KEY (k) NOT ENFORCED
) WITH (
      'merge-engine' = 'partial-update',
      'fields.a.sequence-group' = 'b',
      'fields.c.sequence-group' = 'd',
      'fields.default-aggregate-function' = 'last_non_null_value',
      'fields.d.aggregate-function' = 'sum'
      );

INSERT INTO t
VALUES (1, 1, 1, CAST(NULL AS INT), CAST(NULL AS INT));
INSERT INTO t
VALUES (1, CAST(NULL AS INT), CAST(NULL AS INT), 1, 1);
INSERT INTO t
VALUES (1, 2, 2, CAST(NULL AS INT), CAST(NULL AS INT));
INSERT INTO t
VALUES (1, CAST(NULL AS INT), CAST(NULL AS INT), 2, 2);


SELECT *
FROM t; -- output 1, 2, 2, 2, 3

```
