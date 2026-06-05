---
title: "Hive Catalogs"
sidebar_position: 5
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

# Hive Catalog {#hive-catalog}

在创建 Paimon 表时，设置 `'metadata.iceberg.storage' = 'hive-catalog'`。
该配置项的值不仅会像 hadoop-catalog 那样存储 Iceberg 元数据，还会在 Hive 中创建 Iceberg 外部表。
之后即可通过 Iceberg Hive Catalog 访问该 Paimon 表。

为了提供 Hive metastore 的相关信息，
你在创建 Paimon 表时还需要设置以下部分（或全部）表配置项。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 20%">配置项</th>
      <th class="text-left" style="width: 5%">默认值</th>
      <th class="text-left" style="width: 10%">类型</th>
      <th class="text-left" style="width: 60%">说明</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>metadata.iceberg.uri</h5></td>
      <td style="word-wrap: break-word;"></td>
      <td>String</td>
      <td>Iceberg Hive Catalog 使用的 Hive metastore uri。</td>
    </tr>
    <tr>
      <td><h5>metadata.iceberg.hive-conf-dir</h5></td>
      <td style="word-wrap: break-word;"></td>
      <td>String</td>
      <td>Iceberg Hive Catalog 使用的 hive-conf-dir。</td>
    </tr>
    <tr>
      <td><h5>metadata.iceberg.hadoop-conf-dir</h5></td>
      <td style="word-wrap: break-word;"></td>
      <td>String</td>
      <td>Iceberg Hive Catalog 使用的 hadoop-conf-dir。</td>
    </tr>
    <tr>
      <td><h5>metadata.iceberg.manifest-compression</h5></td>
      <td style="word-wrap: break-word;">snappy</td>
      <td>String</td>
      <td>Iceberg manifest（清单）文件的压缩方式。</td>
    </tr>
    <tr>
      <td><h5>metadata.iceberg.manifest-legacy-version</h5></td>
      <td style="word-wrap: break-word;">false</td>
      <td>Boolean</td>
      <td>是否使用旧版 manifest 版本来生成 Iceberg 1.4 的 manifest 文件。</td>
    </tr>
    <tr>
      <td><h5>metadata.iceberg.hive-client-class</h5></td>
      <td style="word-wrap: break-word;">org.apache.hadoop.hive.metastore.HiveMetaStoreClient</td>
      <td>String</td>
      <td>Iceberg Hive Catalog 使用的 Hive 客户端类名。</td>
    </tr>
    <tr>
      <td><h5>metadata.iceberg.glue.skip-archive</h5></td>
      <td style="word-wrap: break-word;">false</td>
      <td>Boolean</td>
      <td>跳过 AWS Glue catalog 的归档。</td>
    </tr>
    <tr>
      <td><h5>metadata.iceberg.hive-skip-update-stats</h5></td>
      <td style="word-wrap: break-word;">false</td>
      <td>Boolean</td>
      <td>跳过更新 Hive 统计信息。</td>
    </tr>
    </tbody>
</table>

## AWS Glue Catalog {#aws-glue-catalog}

你可以使用 Hive Catalog 连接 AWS Glue metastore，将 `'metadata.iceberg.hive-client-class'` 设置为
`'com.amazonaws.glue.catalog.metastore.AWSCatalogMetastoreClient'`。

> **注意：** 你可以使用这个 [仓库](https://github.com/promotedai/aws-glue-data-catalog-client-for-apache-hive-metastore) 构建所需的 jar，将其加入到你的路径中并配置 AWSCatalogMetastoreClient。
