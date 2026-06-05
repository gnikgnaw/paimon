---
title: "Postgres CDC"
sidebar_position: 2
---

```mdx-code-block
import ConfigTable from '@site/src/components/ConfigTable';
import postgresSyncTableHtml from '@site/generated/postgres_sync_table.html';
```



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

# Postgres CDC {#postgres-cdc}

Paimon 支持通过变更数据捕获（CDC）来同步来自不同数据库的变更。该特性需要 Flink 及其 [CDC 连接器](https://nightlies.apache.org/flink/flink-cdc-docs-stable/)。

## 准备 CDC Bundled Jar {#prepare-cdc-bundled-jar}

下载 `CDC Bundled Jar` 并将其放置到 \<FLINK_HOME>/lib/ 目录下。

| 版本    | Bundled Jar                                                                                                                                                                                                                                                                                                                          |
|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3.5.0   | <li> <a href="https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-postgres-cdc/3.5.0/flink-sql-connector-postgres-cdc-3.5.0.jar">flink-sql-connector-postgres-cdc-3.5.0.jar</a> |

:::danger

仅支持 CDC 3.5.0 及以上版本。

:::

## 同步表 {#synchronizing-tables}

通过在 Flink DataStream 作业中使用 [PostgresSyncTableAction](https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/cdc/postgres/PostgresSyncTableAction)，或直接通过 `flink run`，用户可以将 PostgreSQL 中的一张或多张表同步到一张 Paimon 表中。

要通过 `flink run` 使用该特性，请运行以下 shell 命令。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    postgres_sync_table \
    --warehouse <warehouse_path> \
    --database <database_name> \
    --table <table_name> \
    [--partition_keys <partition_keys>] \
    [--primary_keys <primary_keys>] \
    [--type_mapping <option1,option2...>] \
    [--computed_column <'column-name=expr-name(args[, ...])'> [--computed_column ...]] \
    [--metadata_column <metadata_column>] \
    [--postgres_conf <postgres_cdc_source_conf> [--postgres_conf <postgres_cdc_source_conf> ...]] \
    [--catalog_conf <paimon_catalog_conf> [--catalog_conf <paimon_catalog_conf> ...]] \
    [--table_conf <paimon_table_sink_conf> [--table_conf <paimon_table_sink_conf> ...]]
```

```mdx-code-block
```mdx-code-block
<ConfigTable html={postgresSyncTableHtml} />
```
```

如果你指定的 Paimon 表不存在，该 Action 将自动创建该表。它的 schema 将从所有指定的 PostgreSQL 表中推导得出。如果该 Paimon 表已存在，则会将其 schema 与所有指定的 PostgreSQL 表的 schema 进行比较。

示例 1：将多张表同步到一张 Paimon 表中

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    postgres_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --partition_keys pt \
    --primary_keys pt,uid \
    --computed_column '_year=year(age)' \
    --postgres_conf hostname=127.0.0.1 \
    --postgres_conf username=root \
    --postgres_conf password=123456 \
    --postgres_conf database-name='source_db' \
    --postgres_conf schema-name='public' \
    --postgres_conf table-name='source_table1|source_table2' \
    --postgres_conf slot.name='paimon_cdc' \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

如示例所示，postgres_conf 的 table-name 支持使用正则表达式来监控满足该正则表达式的多张表。所有这些表的 schema 将被合并为一张 Paimon 表的 schema。

示例 2：将多个分片同步到一张 Paimon 表中

你也可以将 'schema-name' 设置为一个正则表达式以捕获多个 schema。一个典型的场景是：一张表 'source_table' 被拆分到 schema 'source_schema1'、'source_schema2' ……中，那么你可以将所有这些 'source_table' 的数据同步到一张 Paimon 表中。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    postgres_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --partition_keys pt \
    --primary_keys pt,uid \
    --computed_column '_year=year(age)' \
    --postgres_conf hostname=127.0.0.1 \
    --postgres_conf username=root \
    --postgres_conf password=123456 \
    --postgres_conf database-name='source_db' \
    --postgres_conf schema-name='source_schema.+' \
    --postgres_conf table-name='source_table' \
    --postgres_conf slot.name='paimon_cdc' \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```
