---
title: "Mongo CDC"
sidebar_position: 4
---

```mdx-code-block
import ConfigTable from '@site/src/components/ConfigTable';
import mongodbSyncTableHtml from '@site/generated/mongodb_sync_table.html';
import mongodbOperatorHtml from '@site/generated/mongodb_operator.html';
import mongodbFunctionsHtml from '@site/generated/mongodb_functions.html';
import mongodbPathExampleHtml from '@site/generated/mongodb_path_example.html';
import mongodbSyncDatabaseHtml from '@site/generated/mongodb_sync_database.html';
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

# Mongo CDC {#mongo-cdc}

## 准备 MongoDB bundled jar {#prepare-mongodb-bundled-jar}


| 版本 | Bundled Jar                                                                                                                                                                                  |
|---------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3.5.0   | <li> <a href="https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-mongodb-cdc/3.5.0/flink-sql-connector-mongodb-cdc-3.5.0.jar">flink-sql-connector-mongodb-cdc-3.5.0.jar</a> |

:::danger

仅支持 CDC 3.5.0 及以上版本。

:::

## 同步表 {#synchronizing-tables}

通过在 Flink DataStream 作业中使用 [MongoDBSyncTableAction](https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/cdc/mongodb/MongoDBSyncTableAction)，或直接通过 `flink run`，用户可以将 MongoDB 中的一个集合（collection）同步到一张 Paimon 表中。

要通过 `flink run` 使用该功能，请运行以下 shell 命令。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mongodb_sync_table \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <table-name> \
    [--partition_keys <partition_keys>] \
    [--computed_column <'column-name=expr-name(args[, ...])'> [--computed_column ...]] \
    [--mongodb_conf <mongodb-cdc-source-conf> [--mongodb_conf <mongodb-cdc-source-conf> ...]] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]] \
    [--table_conf <paimon-table-sink-conf> [--table_conf <paimon-table-sink-conf> ...]]
```

```mdx-code-block
```mdx-code-block
<ConfigTable html={mongodbSyncTableHtml} />
```
```

以下是一些需要注意的要点：

1. `mongodb_conf` 在 MongoDB CDC source 配置之上引入了 `schema.start.mode` 参数。`schema.start.mode` 提供两种模式：`dynamic`（默认）和 `specified`。
   在 `dynamic` 模式下，MongoDB 的 schema 信息在某一层级被解析，这构成了 schema 变更演进的基础。
   在 `specified` 模式下，同步将按照指定的条件进行。
   这可以通过配置 `field.name` 来指定要同步的字段，并通过 `parser.path` 来指定这些字段的 JSON 解析路径来实现。
   两者的区别在于：`specify` 模式要求用户明确指定要使用的字段，并基于这些字段创建映射表。
   而 `dynamic` 模式则确保 Paimon 与 MongoDB 始终保持顶层字段一致，无需关注具体字段。
   当使用嵌套字段中的值时，需要对数据表进行进一步处理。
2. `mongodb_conf` 引入了 `default.id.generation` 参数，作为对 MongoDB CDC source 配置的增强。`default.id.generation` 设置提供两种不同的行为：设为 true 时和设为 false 时。
   当 `default.id.generation` 设为 true 时，MongoDB CDC source 遵循默认的 `_id` 生成策略，即剥离外层 $oid 嵌套，以提供更直接的标识符。该模式简化了 `_id` 的表示，使其更直接、更易于使用。
   相反，当 `default.id.generation` 设为 false 时，MongoDB CDC source 保留原始的 `_id` 结构，不做任何额外处理。该模式让用户能够灵活地使用 MongoDB 提供的原始 `_id` 格式，保留诸如 `$oid` 之类的任何嵌套元素。
   二者之间的选择取决于用户的偏好：前者用于获得更整洁、简化的 `_id`，后者用于直接表示 MongoDB 的 `_id` 结构。

```mdx-code-block
```mdx-code-block
<ConfigTable html={mongodbOperatorHtml} />
```
```


函数可以在路径的末尾被调用——函数的输入是路径表达式的输出。函数的输出由函数本身决定。

```mdx-code-block
```mdx-code-block
<ConfigTable html={mongodbFunctionsHtml} />
```
```

路径示例
```json
{
    "store": {
        "book": [
            {
                "category": "reference",
                "author": "Nigel Rees",
                "title": "Sayings of the Century",
                "price": 8.95
            },
            {
                "category": "fiction",
                "author": "Evelyn Waugh",
                "title": "Sword of Honour",
                "price": 12.99
            },
            {
                "category": "fiction",
                "author": "Herman Melville",
                "title": "Moby Dick",
                "isbn": "0-553-21311-3",
                "price": 8.99
            },
            {
                "category": "fiction",
                "author": "J. R. R. Tolkien",
                "title": "The Lord of the Rings",
                "isbn": "0-395-19395-8",
                "price": 22.99
            }
        ],
        "bicycle": {
            "color": "red",
            "price": 19.95
        }
    },
    "expensive": 10
}
```

```mdx-code-block
```mdx-code-block
<ConfigTable html={mongodbPathExampleHtml} />
```
```

1. 同步的表必须将其主键设置为 `_id`。
   这是因为 MongoDB 的变更事件在消息中记录的是更新之前的状态。
   因此，我们只能将它们转换为 Flink 的 UPSERT 变更日志流。
   Upsert 流要求有唯一键，这就是为什么我们必须将 `_id` 声明为主键。
   声明其他列为主键是不可行的，因为删除操作只包含 _id 和分片键，而不包含其他键和值。

2. MongoDB Change Streams 被设计为返回不带任何数据类型定义的简单 JSON 文档。这是因为 MongoDB 是面向文档的数据库，其核心特性之一是动态 schema，即文档可以包含不同的字段，且字段的数据类型可以是灵活的。因此，Change Streams 中缺少数据类型定义是为了保持这种灵活性和可扩展性。
   出于这一原因，我们将同步到 Paimon 的 MongoDB 所有字段的数据类型都设置为 String，以解决无法获取数据类型的问题。

如果你指定的 Paimon 表不存在，该 Action 会自动创建该表。其 schema 将从 MongoDB 集合派生。

示例 1：将集合同步到一张 Paimon 表

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mongodb_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --partition_keys pt \
    --computed_column '_year=year(age)' \
    --mongodb_conf hosts=127.0.0.1:27017 \
    --mongodb_conf username=root \
    --mongodb_conf password=123456 \
    --mongodb_conf database=source_db \
    --mongodb_conf collection=source_table1 \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

示例 2：根据指定的字段映射将集合同步到一张 Paimon 表。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mongodb_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --partition_keys pt \
    --mongodb_conf hosts=127.0.0.1:27017 \
    --mongodb_conf username=root \
    --mongodb_conf password=123456 \
    --mongodb_conf database=source_db \
    --mongodb_conf collection=source_table1 \
    --mongodb_conf schema.start.mode=specified \
    --mongodb_conf field.name=_id,name,description \
    --mongodb_conf parser.path=$._id,$.name,$.description \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

## 同步数据库 {#synchronizing-databases}

通过在 Flink DataStream 作业中使用 [MongoDBSyncDatabaseAction](https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/cdc/mongodb/MongoDBSyncDatabaseAction)，或直接通过 `flink run`，用户可以将整个 MongoDB 数据库同步到一个 Paimon 数据库中。

要通过 `flink run` 使用该功能，请运行以下 shell 命令。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mongodb_sync_database \
    --warehouse <warehouse-path> \
    --database <database-name> \
    [--table_prefix <paimon-table-prefix>] \
    [--table_suffix <paimon-table-suffix>] \
    [--including_tables <mongodb-table-name|name-regular-expr>] \
    [--excluding_tables <mongodb-table-name|name-regular-expr>] \
    [--partition_keys <partition_keys>] \
    [--primary_keys <primary-keys>] \
    [--mongodb_conf <mongodb-cdc-source-conf> [--mongodb_conf <mongodb-cdc-source-conf> ...]] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]] \
    [--table_conf <paimon-table-sink-conf> [--table_conf <paimon-table-sink-conf> ...]]
```

```mdx-code-block
```mdx-code-block
<ConfigTable html={mongodbSyncDatabaseHtml} />
```
```

所有要同步的集合都需要将 _id 设置为主键。
对于每个要同步的 MongoDB 集合，如果对应的 Paimon 表不存在，该 Action 会自动创建该表。
其 schema 将从所有指定的 MongoDB 集合派生。如果 Paimon 表已存在，其 schema 将与所有指定的 MongoDB 集合的 schema 进行比较。
任务启动之后创建的任何 MongoDB 表都会自动被纳入同步。

示例 1：同步整个数据库

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    mongodb_sync_database \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --mongodb_conf hosts=127.0.0.1:27017 \
    --mongodb_conf username=root \
    --mongodb_conf password=123456 \
    --mongodb_conf database=source_db \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

示例 2：同步指定的表。

```bash
<FLINK_HOME>/bin/flink run \
--fromSavepoint savepointPath \
/path/to/paimon-flink-action-@@VERSION@@.jar \
mongodb_sync_database \
--warehouse hdfs:///path/to/warehouse \
--database test_db \
--mongodb_conf hosts=127.0.0.1:27017 \
--mongodb_conf username=root \
--mongodb_conf password=123456 \
--mongodb_conf database=source_db \
--catalog_conf metastore=hive \
--catalog_conf uri=thrift://hive-metastore:9083 \
--table_conf bucket=4 \
--including_tables 'product|user|address|order|custom'
```
