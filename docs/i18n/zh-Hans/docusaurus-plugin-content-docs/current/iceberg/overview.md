---
title: "概述"
sidebar_position: 1
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

# 概述 {#overview}

Paimon 支持生成与 Iceberg 兼容的元数据，
从而使 Paimon 表能够直接被 Iceberg 读取器消费。

设置以下表配置项，即可让 Paimon 表生成与 Iceberg 兼容的元数据。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 20%">配置项</th>
      <th class="text-left" style="width: 5%">默认值</th>
      <th class="text-left" style="width: 10%">类型</th>
      <th class="text-left" style="width: 60%">描述</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>metadata.iceberg.storage</h5></td>
      <td style="word-wrap: break-word;">disabled</td>
      <td>Enum</td>
      <td>
        设置后，在快照提交之后生成 Iceberg 元数据，从而使 Iceberg 读取器能够读取 Paimon 的原始数据文件。
        <ul>
          <li><code>disabled</code>：禁用 Iceberg 兼容支持。</li>
          <li><code>table-location</code>：将 Iceberg 元数据存储在每个表的目录中。</li>
          <li><code>hadoop-catalog</code>：将 Iceberg 元数据存储在一个单独的目录中。该目录可被指定为 Iceberg Hadoop catalog 的 warehouse（仓库目录）。</li>
          <li><code>hive-catalog</code>：不仅像 hadoop-catalog 那样存储 Iceberg 元数据，还会在 Hive 中创建 Iceberg 外部表。</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td><h5>metadata.iceberg.storage-location</h5></td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>Enum</td>
      <td>
        指定 Iceberg 元数据文件的存储位置。如果未设置，存储位置将根据所选的 metadata.iceberg.storage 类型采用默认值。
        <ul>
          <li><code>table-location</code>：将 Iceberg 元数据存储在每个表的目录中。适用于独立的 Iceberg 表或通过 Iceberg Java API 访问的场景。也可与 Hive Catalog 配合使用。</li>
          <li><code>catalog-location</code>：将 Iceberg 元数据存储在一个单独的目录中。这是使用 Hive Catalog 或 Hadoop Catalog 时的默认行为。</li>
        </ul>
      </td>
    </tr>
    </tbody>
</table>

对于大多数 SQL 用户，我们建议设置 `'metadata.iceberg.storage' = 'hadoop-catalog'
或 `'metadata.iceberg.storage' = 'hive-catalog'`，
这样所有表都可以作为一个 Iceberg warehouse（仓库目录）被访问。
对于 Iceberg Java API 用户，你可以考虑设置 `'metadata.iceberg.storage' = 'table-location'`，
这样你就可以通过每个表的表路径来访问它。
当使用 `metadata.iceberg.storage = hadoop-catalog` 或 `hive-catalog` 时，
你可以选择性地配置 `metadata.iceberg.storage-location` 来控制元数据的存储位置。
如果未设置，默认行为取决于存储类型。

## 支持的类型 {#supported-types}

Paimon 的 Iceberg 兼容性目前支持以下数据类型。

| Paimon 数据类型 | Iceberg 数据类型 |
|----------------|-------------------|
| `BOOLEAN`      | `boolean`         |
| `INT`          | `int`             |
| `BIGINT`       | `long`            |
| `FLOAT`        | `float`           |
| `DOUBLE`       | `double`          |
| `DECIMAL`      | `decimal`         |
| `CHAR`         | `string`          |
| `VARCHAR`      | `string`          |
| `BINARY`       | `binary`          |
| `VARBINARY`    | `binary`          |
| `DATE`         | `date`            |
| `TIMESTAMP` (precision 3-6)   | `timestamp`       |
| `TIMESTAMP_LTZ` (precision 3-6) | `timestamptz`     |
| `TIMESTAMP` (precision 7-9)  | `timestamp_ns`    |
| `TIMESTAMP_LTZ` (precision 7-9) | `timestamptz_ns`  |
| `ARRAY`        | `list`            |
| `MAP`          | `map`             |
| `ROW`          | `struct`          |

:::info

**关于 Timestamp 类型的说明：**
- 精度为 3 到 6 的 `TIMESTAMP` 和 `TIMESTAMP_LTZ` 类型会映射为标准的 Iceberg timestamp 类型
- 精度为 7 到 9 的 `TIMESTAMP` 和 `TIMESTAMP_LTZ` 类型使用纳秒精度，并要求 Iceberg v3 格式

:::
