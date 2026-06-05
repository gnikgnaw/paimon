---
title: "Mysql CDC"
sidebar_position: 2
---

```mdx-code-block
import ConfigTable from '@site/src/components/ConfigTable';
import mysqlSyncTableHtml from '@site/generated/mysql_sync_table.html';
import mysqlSyncDatabaseHtml from '@site/generated/mysql_sync_database.html';
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

# MySQL CDC {#mysql-cdc}

Paimon 支持使用变更数据捕获（CDC）来同步来自不同数据库的变更。该功能需要 Flink 及其 [CDC 连接器](https://nightlies.apache.org/flink/flink-cdc-docs-stable/)。

## 准备 CDC Bundled Jar {#prepare-cdc-bundled-jar}

下载 `CDC Bundled Jar` 并将其放到 \<FLINK_HOME>/lib/ 目录下。

| 版本    | Bundled Jar                                                                                                                                                                                                                                                                                                                                |
|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3.5.0   | <li> <a href="https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-mysql-cdc/3.5.0/flink-sql-connector-mysql-cdc-3.5.0.jar">flink-sql-connector-mysql-cdc-3.5.0.jar</a> <li> <a href="https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.27/mysql-connector-java-8.0.27.jar">mysql-connector-java-8.0.27.jar</a> |

:::danger

仅支持 CDC 3.5.0 及以上版本。

:::

## 同步表 {#synchronizing-tables}

通过在 Flink DataStream 作业中使用 [MySqlSyncTableAction](https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/cdc/mysql/MySqlSyncTableAction)，或直接通过 `flink run`，用户可以将 MySQL 中的一张或多张表同步到一张 Paimon 表中。

要通过 `flink run` 使用此功能，请运行以下 shell 命令。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mysql_sync_table \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <table-name> \
    [--partition_keys <partition_keys>] \
    [--primary_keys <primary-keys>] \
    [--type_mapping <option1,option2...>] \
    [--computed_column <'column-name=expr-name(args[, ...])'> [--computed_column ...]] \
    [--metadata_column <metadata-column>] \
    [--mysql_conf <mysql-cdc-source-conf> [--mysql_conf <mysql-cdc-source-conf> ...]] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]] \
    [--table_conf <paimon-table-sink-conf> [--table_conf <paimon-table-sink-conf> ...]]
```

```mdx-code-block
```mdx-code-block
<ConfigTable html={mysqlSyncTableHtml} />
```
```

如果你指定的 Paimon 表不存在，该 Action 会自动创建该表。其 schema（表结构）将由所有指定的 MySQL 表推导得出。如果 Paimon 表已存在，则会将其 schema 与所有指定的 MySQL 表的 schema 进行比较。

### 示例 1：将多张表同步到一张 Paimon 表 {#example-1-synchronize-tables-into-one-paimon-table}

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mysql_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --partition_keys pt \
    --primary_keys pt,uid \
    --computed_column '_year=year(age)' \
    --mysql_conf hostname=127.0.0.1 \
    --mysql_conf username=root \
    --mysql_conf password=123456 \
    --mysql_conf database-name='source_db' \
    --mysql_conf table-name='source_table1|source_table2' \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

如示例所示，mysql_conf 的 table-name 支持正则表达式，以监控所有满足该正则表达式的多张表。所有这些表的 schema 将被合并成一个 Paimon 表的 schema。

### 示例 2：将多个分片同步到一张 Paimon 表 {#example-2-synchronize-shards-into-one-paimon-table}

你也可以将 'database-name' 设置为正则表达式来捕获多个数据库。一个典型的场景是：一张表 'source_table' 被拆分到数据库 'source_db1'、'source_db2' ……，那么你可以将所有 'source_table' 的数据同步到一张 Paimon 表中。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mysql_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --partition_keys pt \
    --primary_keys pt,uid \
    --computed_column '_year=year(age)' \
    --mysql_conf hostname=127.0.0.1 \
    --mysql_conf username=root \
    --mysql_conf password=123456 \
    --mysql_conf database-name='source_db.+' \
    --mysql_conf table-name='source_table' \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

## 同步数据库 {#synchronizing-databases}

通过在 Flink DataStream 作业中使用 [MySqlSyncDatabaseAction](https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/cdc/mysql/MySqlSyncDatabaseAction)，或直接通过 `flink run`，用户可以将整个 MySQL 数据库同步到一个 Paimon 数据库中。

要通过 `flink run` 使用此功能，请运行以下 shell 命令。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mysql_sync_database \
    --warehouse <warehouse-path> \
    --database <database-name> \
    [--ignore_incompatible <true/false>] \
    [--merge_shards <true/false>] \
    [--table_prefix <paimon-table-prefix>] \
    [--table_suffix <paimon-table-suffix>] \
    [--including_tables <mysql-table-name|name-regular-expr>] \
    [--excluding_tables <mysql-table-name|name-regular-expr>] \
    [--mode <sync-mode>] \
    [--metadata_column <metadata-column>] \
    [--type_mapping <option1,option2...>] \
    [--partition_keys <partition_keys>] \
    [--primary_keys <primary-keys>] \
    [--mysql_conf <mysql-cdc-source-conf> [--mysql_conf <mysql-cdc-source-conf> ...]] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]] \
    [--table_conf <paimon-table-sink-conf> [--table_conf <paimon-table-sink-conf> ...]]
```

```mdx-code-block
```mdx-code-block
<ConfigTable html={mysqlSyncDatabaseHtml} />
```
```

只有带主键的表才会被同步。

对于每张要被同步的 MySQL 表，如果对应的 Paimon 表不存在，该 Action 会自动创建该表。其 schema 将由所有指定的 MySQL 表推导得出。如果 Paimon 表已存在，则会将其 schema 与所有指定的 MySQL 表的 schema 进行比较。

### 示例 1：同步整个数据库 {#example-1-synchronize-entire-database}

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mysql_sync_database \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --mysql_conf hostname=127.0.0.1 \
    --mysql_conf username=root \
    --mysql_conf password=123456 \
    --mysql_conf database-name=source_db \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

### 示例 2：同步数据库下新增的表 {#example-2-synchronize-newly-added-tables-under-database}

假设最初有一个 Flink 作业正在同步数据库 `source_db` 下的表 [product, user, address]。提交该作业的命令如下：

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mysql_sync_database \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --mysql_conf hostname=127.0.0.1 \
    --mysql_conf username=root \
    --mysql_conf password=123456 \
    --mysql_conf database-name=source_db \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4 \
    --including_tables 'product|user|address'
```

在之后的某个时刻，我们希望该作业也同步包含历史数据的表 [order, custom]。我们可以通过从该作业之前的快照恢复，从而复用该作业现有的状态来实现这一点。恢复后的作业会首先对新增的表做快照，然后自动从之前的位置继续读取 changelog。

从之前的快照恢复并新增要同步的表的命令如下：

```bash
<FLINK_HOME>/bin/flink run \
    --fromSavepoint savepointPath \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mysql_sync_database \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --mysql_conf hostname=127.0.0.1 \
    --mysql_conf username=root \
    --mysql_conf password=123456 \
    --mysql_conf database-name=source_db \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --including_tables 'product|user|address|order|custom'
```

:::info

你可以设置 `--mode combined` 来启用无需重启作业即可同步新增表的功能。

:::

### 示例 3：同步并合并多个分片 {#example-3-synchronize-and-merge-multiple-shards}

假设你有多个数据库分片 `db1`、`db2` ……，且每个数据库都有表 `tbl1`、`tbl2` ……。你可以通过以下命令将所有 `db.+.tbl.+` 同步到表 `test_db.tbl1`、`test_db.tbl2` ……：

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mysql_sync_database \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --mysql_conf hostname=127.0.0.1 \
    --mysql_conf username=root \
    --mysql_conf password=123456 \
    --mysql_conf database-name='db.+' \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4 \
    --including_tables 'tbl.+'
```

通过将 database-name 设置为正则表达式，同步作业将捕获所有匹配数据库下的全部表，并将同名的表合并到一张表中。

:::info

你可以设置 `--merge_shards false` 来阻止合并分片。同步后的表将被命名为 'databaseName_tableName'，以避免潜在的命名冲突。

:::

## FAQ {#faq}

1. 从 MySQL 接入的记录中的中文字符出现乱码。

* 尝试在 `flink-conf.yaml`（Flink 版本 \< 1.19）或 `config.yaml`（Flink 版本 >= 1.19）中设置 `env.java.opts: -Dfile.encoding=UTF-8`
（自 Flink-1.17 起，该配置项更改为 `env.java.opts.all`）。

2. 同步 MySQL 的表注释和列注释。

* 要将 MySQL 的建表注释同步到 Paimon 表，你需要配置 `--mysql_conf jdbc.properties.useInformationSchema=true`。
* 要将 MySQL 的 alter table 或列注释同步到 Paimon 表，你需要配置 `--mysql_conf debezium.include.schema.comments=true`。
