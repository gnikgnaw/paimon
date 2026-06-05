---
title: "Doris"
sidebar_position: 3
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

# Doris {#doris}

本文档是在 Doris 中使用 Paimon 的指南。

> 更多细节请参阅 [Apache Doris 官网](https://doris.apache.org/docs/dev/lakehouse/catalogs/paimon-catalog)

## 版本 {#version}

Paimon 目前支持 Apache Doris 2.0.6 及以上版本。

## 创建 Paimon Catalog {#create-paimon-catalog}

在 Apache Doris 中使用 `CREATE CATALOG` 语句来创建 Paimon Catalog。

Doris 支持多种类型的 Paimon Catalog。下面是一些示例：

```sql
-- HDFS based Paimon Catalog
CREATE CATALOG `paimon_hdfs` PROPERTIES (
    "type" = "paimon",
    "warehouse" = "hdfs://172.21.0.1:8020/user/paimon",
    "hadoop.username" = "hadoop"
);

-- Aliyun OSS based Paimon Catalog
CREATE CATALOG `paimon_oss` PROPERTIES (
    "type" = "paimon",
    "warehouse" = "oss://paimon-bucket/paimonoss",
    "oss.endpoint" = "oss-cn-beijing.aliyuncs.com",
    "oss.access_key" = "ak",
    "oss.secret_key" = "sk"
);

-- Hive Metastore based Paimon Catalog
CREATE CATALOG `paimon_hms` PROPERTIES (
    "type" = "paimon",
    "paimon.catalog.type" = "hms",
    "warehouse" = "hdfs://172.21.0.1:8020/user/zhangdong/paimon2",
    "hive.metastore.uris" = "thrift://172.21.0.44:7004",
    "hadoop.username" = "hadoop"
);

-- Integrate with Aliyun DLF 1.0
CREATE CATALOG paimon_dlf PROPERTIES (
    'type' = 'paimon',
    'paimon.catalog.type' = 'dlf',
    'warehouse' = 'oss://paimon-bucket/paimonoss/',
    'dlf.proxy.mode' = 'DLF_ONLY',
    'dlf.uid' = 'xxxxx',
    'dlf.region' = 'cn-beijing',
    'dlf.access_key' = 'ak',
    'dlf.secret_key' = 'sk'
);

-- Integrate with Aliyun DLF 3.0 Paimon Rest
-- Apache Doris supported since version 3.1.0
CREATE CATALOG dlf_paimon_rest PROPERTIES (
    'type' = 'paimon',
    'uri' = 'http://cn-beijing-vpc.dlf.aliyuncs.com',
    'warehouse' = 'catalog_name',
    'paimon.rest.token.provider' = 'dlf',
    'paimon.rest.dlf.access-key-id' = 'ak',
    'paimon.rest.dlf.access-key-secret' = 'sk'
);
```

更多示例请参阅 [Apache Doris 官网](https://doris.apache.org/docs/dev/lakehouse/catalogs/paimon-catalog#examples)。

## 访问 Paimon Catalog {#access-paimon-catalog}

1. 使用全限定名查询 Paimon 表

    ```sql
    SELECT * FROM paimon_hdfs.paimon_db.paimon_table;
    ```

2. 切换到 Paimon Catalog 后再查询

    ```sql
    SWITCH paimon_hdfs;
    USE paimon_db;
    SELECT * FROM paimon_table;
    ```

## 查询优化 {#query-optimization}

- 主键表的读优化（Read optimized）

    Doris 可以利用主键表的[读优化（Read optimized）](https://paimon.apache.org/docs/0.8/primary-key-table/read-optimized/)特性（在 Paimon 0.6 中发布），通过原生 Parquet/ORC 读取器读取 base 数据文件、通过 JNI 读取 delta 文件。

- 删除向量（Deletion Vectors）

    Doris（2.1.4+）原生支持[删除向量（Deletion Vectors）](https://paimon.apache.org/docs/0.8/primary-key-table/deletion-vectors/)（在 Paimon 0.8 中发布）。

## Doris 到 Paimon 的类型映射 {#doris-to-paimon-type-mapping}

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 10%">Doris 数据类型</th>
      <th class="text-left" style="width: 10%">Paimon 数据类型</th>
      <th class="text-left" style="width: 5%">原子类型</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><code>Boolean</code></td>
      <td><code>BooleanType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>TinyInt</code></td>
      <td><code>TinyIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>SmallInt</code></td>
      <td><code>SmallIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Int</code></td>
      <td><code>IntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>BigInt</code></td>
      <td><code>BigIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Float</code></td>
      <td><code>FloatType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Double</code></td>
      <td><code>DoubleType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>VarChar</code></td>
      <td><code>VarCharType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Char</code></td>
      <td><code>CharType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Binary</code></td>
      <td><code>VarBinaryType, BinaryType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Decimal(precision, scale)</code></td>
      <td><code>DecimalType(precision, scale)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Datetime</code></td>
      <td><code>TimestampType,LocalZonedTimestampType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Date</code></td>
      <td><code>DateType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>Array</code></td>
      <td><code>ArrayType</code></td>
      <td>false</td>
    </tr>
    <tr>
      <td><code>Map</code></td>
      <td><code>MapType</code></td>
      <td>false</td>
    </tr>
    <tr>
      <td><code>Struct</code></td>
      <td><code>RowType</code></td>
      <td>false</td>
    </tr>
    </tbody>
</table>
