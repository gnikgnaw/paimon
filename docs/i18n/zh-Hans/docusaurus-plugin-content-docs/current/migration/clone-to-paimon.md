---
title: "克隆到 Paimon"
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

# 克隆到 Paimon {#clone-to-paimon}

克隆（Clone）支持将表克隆为 Paimon 表。

1. 克隆采用 `OVERWRITE`（覆盖写）语义，会根据数据覆盖目标表的对应分区。
2. 克隆是可重入的，但要求已存在的表包含源表的全部字段，并具有相同的分区字段。

目前，克隆支持将 Hive Catalog 中的 Hive 表克隆到 Paimon Catalog，支持 Parquet、ORC、Avro 格式，目标表将是 Append 表（追加表）。

## 克隆 Hive 表 {#clone-hive-table}

```bash
<FLINK_HOME>/flink run ./paimon-flink-action-@@VERSION@@.jar \
clone \
--database default \
--table hivetable \
--catalog_conf metastore=hive \
--catalog_conf uri=thrift://localhost:9088 \
--target_database test \
--target_table test_table \
--target_catalog_conf warehouse=my_warehouse \
--parallelism 10 \
--where <filter_spec>
```

你可以使用 filter spec 来指定分区的过滤条件。

## 克隆 Hive 数据库 {#clone-hive-database}

```bash
<FLINK_HOME>/flink run ./paimon-flink-action-@@VERSION@@.jar \
clone \
--database default \
--catalog_conf metastore=hive \
--catalog_conf uri=thrift://localhost:9088 \
--target_database test \
--parallelism 10 \
--target_catalog_conf warehouse=my_warehouse
--included_tables <included_tables_spec> \
--excluded_tables <excluded_tables_spec>
```
"--included_tables" 和 "--excluded_tables" 是可选参数，用于指定需要或不需要克隆的表。
其格式为 `<database1>.<table1>,<database2>.<table2>,<database3>.<table3>`。
如果同时指定了两者，"--excluded_tables" 的优先级高于 "--included_tables"。

## 克隆 Hive Catalog {#clone-hive-catalog}

```bash
<FLINK_HOME>/flink run ./paimon-flink-action-@@VERSION@@.jar \
clone \
--catalog_conf metastore=hive \
--catalog_conf uri=thrift://localhost:9088 \
--parallelism 10 \
--target_catalog_conf warehouse=my_warehouse \
--included_tables <included_tables_spec> \
--excluded_tables <excluded_tables_spec>
```
"--included_tables" 和 "--excluded_tables" 是可选参数，用于指定需要或不需要克隆的表。
其格式为 `<database1>.<table1>,<database2>.<table2>,<database3>.<table3>`。
如果同时指定了两者，"--excluded_tables" 的优先级高于 "--included_tables"。
