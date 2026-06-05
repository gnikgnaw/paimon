---
title: "Catalog"
sidebar_position: 4
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

# Catalog {#catalog}

Paimon 提供了 Catalog 抽象来管理表的目录和元数据。Catalog 抽象提供了一系列方式，帮助你更好地与计算引擎集成。我们始终建议你使用 Catalog 来访问 Paimon 表。

## Catalogs {#catalogs}

Paimon 的 Catalog 目前支持四种类型的 metastore：

* `filesystem` metastore（默认），它将元数据和表文件都存储在文件系统中。
* `hive` metastore，它会额外将元数据存储在 Hive metastore 中。用户可以直接从 Hive 访问这些表。
* `jdbc` metastore，它会额外将元数据存储在关系型数据库（如 MySQL、Postgres 等）中。
* `rest` metastore，它旨在提供一种轻量级的方式，从单个客户端访问任意 Catalog 后端。

## Filesystem Catalog {#filesystem-catalog}

元数据和表文件存储在 `hdfs:///path/to/warehouse` 下。

```sql
-- Flink SQL
CREATE CATALOG my_catalog WITH (
    'type' = 'paimon',
    'warehouse' = 'hdfs:///path/to/warehouse'
);
```

## REST Catalog {#rest-catalog}

通过使用 Paimon REST Catalog，对 Catalog 的更改将直接存储在通过 REST API 暴露的远程 Catalog 服务器中。
参见 [Paimon REST Catalog](./rest/overview)。

## Hive Catalog {#hive-catalog}

通过使用 Paimon Hive Catalog，对 Catalog 的更改将直接影响对应的 Hive metastore。在这种 Catalog 中创建的表也可以直接从 Hive 访问。元数据和表文件存储在
`hdfs:///path/to/warehouse` 下。此外，schema（表结构）也会存储在 Hive metastore 中。

```sql
-- Flink SQL
CREATE CATALOG my_hive WITH (
    'type' = 'paimon',
    'metastore' = 'hive',
    -- 'warehouse' = 'hdfs:///path/to/warehouse', default use 'hive.metastore.warehouse.dir' in HiveConf
);
```

默认情况下，Paimon 不会将新创建的分区同步到 Hive metastore。用户在 Hive 中会看到一个非分区表。分区下推将改为通过过滤器下推来实现。

如果你希望在 Hive 中看到分区表，并且也将新创建的分区同步到 Hive metastore，
请将表的配置项 `metastore.partitioned-table` 设置为 true。

## JDBC Catalog {#jdbc-catalog}

通过使用 Paimon JDBC Catalog，对 Catalog 的更改将直接存储在关系型数据库（如 SQLite、MySQL、postgres 等）中。

```sql
-- Flink SQL
CREATE CATALOG my_jdbc WITH (
    'type' = 'paimon',
    'metastore' = 'jdbc',
    'uri' = 'jdbc:mysql://<host>:<port>/<databaseName>',
    'jdbc.user' = '...', 
    'jdbc.password' = '...', 
    'catalog-key'='jdbc',
    'warehouse' = 'hdfs:///path/to/warehouse'
);
```
