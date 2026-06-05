---
title: "StarRocks"
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

# StarRocks {#starrocks}

本文档是在 StarRocks 中使用 Paimon 的指南。

## 版本 {#version}

Paimon 目前支持 StarRocks 3.1 及以上版本。推荐使用 StarRocks 3.2.6 或以上版本。

## 创建 Paimon Catalog {#create-paimon-catalog}

Paimon Catalog 通过在 StarRocks 中执行 `CREATE EXTERNAL CATALOG` SQL 来注册。
例如，你可以使用以下 SQL 创建一个名为 paimon_catalog 的 Paimon Catalog。

```sql
CREATE EXTERNAL CATALOG paimon_catalog PROPERTIES(
    "type" = "paimon",
    "paimon.catalog.type" = "filesystem",
    "paimon.catalog.warehouse" = "oss://<your_bucket>/user/warehouse/"
);
```

更多 Catalog 类型与配置可参见 [Paimon catalog](https://docs.starrocks.io/docs/data_source/catalog/paimon_catalog/)。

## 查询 {#query}
假设在 `paimon_catalog` 中已经存在一个名为 `test_db` 的数据库以及一个名为 `test_tbl` 的表，
你可以使用以下 SQL 查询该表：
```sql
SELECT * FROM paimon_catalog.test_db.test_tbl;
```

## 查询系统表 {#query-system-tables}

你可以通过 StarRocks 访问各种 Paimon 系统表。例如，你可以读取 `ro`
（读优化，read-optimized）系统表以提升主键表（primary key table）的读取性能。

```sql
SELECT * FROM paimon_catalog.test_db.test_tbl$ro;
```

再举一例，你可以使用以下 SQL 查询表的分区文件：

```sql
SELECT * FROM paimon_catalog.test_db.partition_tbl$partitions;
/*
+-----------+--------------+--------------------+------------+----------------------------+
| partition | record_count | file_size_in_bytes | file_count | last_update_time           |
+-----------+--------------+--------------------+------------+----------------------------+
| [1]       |            1 |                645 |          1 | 2024-01-01 00:00:00.000000 |
+-----------+--------------+--------------------+------------+----------------------------+
*/
```

## StarRocks 与 Paimon 类型映射 {#starrocks-to-paimon-type-mapping}

本节列出了 StarRocks 与 Paimon 之间所有受支持的类型转换。
StarRocks 的所有数据类型可参见此文档 [StarRocks Data type overview](https://docs.starrocks.io/docs/sql-reference/data-types/)。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 10%">StarRocks 数据类型</th>
      <th class="text-left" style="width: 10%">Paimon 数据类型</th>
      <th class="text-left" style="width: 5%">原子类型</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><code>STRUCT</code></td>
      <td><code>RowType</code></td>
      <td>false</td>
    </tr>
    <tr>
      <td><code>MAP</code></td>
      <td><code>MapType</code></td>
      <td>false</td>
    </tr>
    <tr>
      <td><code>ARRAY</code></td>
      <td><code>ArrayType</code></td>
      <td>false</td>
    </tr>
    <tr>
      <td><code>BOOLEAN</code></td>
      <td><code>BooleanType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>TINYINT</code></td>
      <td><code>TinyIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>SMALLINT</code></td>
      <td><code>SmallIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>INT</code></td>
      <td><code>IntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>BIGINT</code></td>
      <td><code>BigIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>FLOAT</code></td>
      <td><code>FloatType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>DOUBLE</code></td>
      <td><code>DoubleType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>CHAR(length)</code></td>
      <td><code>CharType(length)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>VARCHAR(MAX_VARCHAR_LENGTH)</code></td>
      <td><code>VarCharType(VarCharType.MAX_LENGTH)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>VARCHAR(length)</code></td>
      <td><code>VarCharType(length), length is less than VarCharType.MAX_LENGTH</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>DATE</code></td>
      <td><code>DateType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>DATETIME</code></td>
      <td><code>TimestampType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>DECIMAL(precision, scale)</code></td>
      <td><code>DecimalType(precision, scale)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>VARBINARY(length)</code></td>
      <td><code>VarBinaryType(length)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>DATETIME</code></td>
      <td><code>LocalZonedTimestampType</code></td>
      <td>true</td>
    </tr>
    </tbody>
</table>
