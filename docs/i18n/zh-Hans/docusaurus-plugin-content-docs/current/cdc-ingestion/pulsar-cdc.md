---
title: "Pulsar CDC"
sidebar_position: 5
---

```mdx-code-block
import ConfigTable from '@site/src/components/ConfigTable';
import pulsarSyncTableHtml from '@site/generated/pulsar_sync_table.html';
import pulsarSyncDatabaseHtml from '@site/generated/pulsar_sync_database.html';
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

# Pulsar CDC {#pulsar-cdc}

## 准备 Pulsar Bundled Jar {#prepare-pulsar-bundled-jar}

```
flink-connector-pulsar-*.jar
```

## 支持的格式 {#supported-formats}
Flink 提供了多种 Pulsar CDC 格式：Canal Json、Debezium Json、Debezium Avro、Ogg Json、Maxwell Json 以及 Normal Json。
如果 Pulsar topic 中的某条消息是使用 CDC（Change Data Capture，变更数据捕获）工具从另一个数据库捕获的变更事件，那么你可以使用 Paimon Pulsar CDC，将解析出的 INSERT、UPDATE、DELETE 消息写入 Paimon 表。
<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">格式</th>
        <th class="text-left">是否支持</th>
      </tr>
    </thead>
    <tbody>
        <tr>
         <td><a href="https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/table/formats/canal/">Canal CDC</a></td>
          <td>True</td>
        </tr>
        <tr>
         <td><a href="https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/table/formats/debezium/">Debezium CDC</a></td>
         <td>True</td>
        </tr>
        <tr>
         <td><a href="https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/table/formats/maxwell/">Maxwell CDC</a></td>
        <td>True</td>
        </tr>
        <tr>
         <td><a href="https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/table/formats/ogg/">OGG CDC</a></td>
        <td>True</td>
        </tr>
        <tr>
         <td><a href="https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/table/formats/json/">JSON</a></td>
        <td>True</td>
        </tr>
    </tbody>
</table>

:::info

JSON 源可能会缺失一些信息。例如，Ogg 和 Maxwell 格式标准不包含字段类型；当你把 JSON 源写入 Flink Pulsar Sink 时，它只会保留数据和行类型，并丢弃其他信息。
同步作业会尽力按如下方式处理该问题：
1. 如果缺失字段类型，Paimon 将默认使用 'STRING' 类型。
2. 如果缺失数据库名或表名，则无法进行数据库同步，但仍可进行表同步。
3. 如果缺失主键，作业可能会创建非主键表。你可以在提交表同步作业时设置主键。

:::

## 同步表 {#synchronizing-tables}

通过在 Flink DataStream 作业中使用 [PulsarSyncTableAction](https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/cdc/pulsar/PulsarSyncTableAction)，或直接通过 `flink run`，用户可以将 Pulsar 的一个 topic 中的一张或多张表同步到一张 Paimon 表中。

要通过 `flink run` 使用该功能，请运行以下 shell 命令。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    pulsar_sync_table \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <table-name> \
    [--partition_keys <partition_keys>] \
    [--primary_keys <primary-keys>] \
    [--type_mapping to-string] \
    [--computed_column <'column-name=expr-name(args[, ...])'> [--computed_column ...]] \
    [--pulsar_conf <pulsar-source-conf> [--pulsar_conf <pulsar-source-conf> ...]] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]] \
    [--table_conf <paimon-table-sink-conf> [--table_conf <paimon-table-sink-conf> ...]]
```

```mdx-code-block
```mdx-code-block
<ConfigTable html={pulsarSyncTableHtml} />
```
```

如果你指定的 Paimon 表不存在，该 Action 将自动创建该表。其 schema 将派生自所有指定 Pulsar topic 中的表，即从 topic 中获取最早的非 DDL 数据来解析 schema。如果该 Paimon 表已存在，则会将其 schema 与所有指定 Pulsar topic 中表的 schema 进行比较。

示例 1：

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    pulsar_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --partition_keys pt \
    --primary_keys pt,uid \
    --computed_column '_year=year(age)' \
    --pulsar_conf topic=order \
    --pulsar_conf value.format=canal-json \
    --pulsar_conf pulsar.client.serviceUrl=pulsar://127.0.0.1:6650 \
    --pulsar_conf pulsar.admin.adminUrl=http://127.0.0.1:8080 \
    --pulsar_conf pulsar.consumer.subscriptionName=paimon-tests \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

如果在你启动同步作业时 Pulsar topic 中尚不包含任何消息，则必须在提交作业之前手动创建该表。你可以只定义分区键和主键，其余列将由同步作业自动添加。

注意：在这种情况下你不应使用 --partition_keys 或 --primary_keys，因为这些键是在创建表时定义的，无法被修改。此外，如果你指定了计算列，还应定义计算列所用到的全部参数列。

示例 2：
如果你想同步一张具有主键 'id INT' 的表，并想计算一个分区键 'part=date_format(create_time,yyyy-MM-dd)'，
你可以先创建这样一张表（其他列可以省略）：

```sql
CREATE TABLE test_db.test_table (
    id INT,                 -- primary key
    create_time TIMESTAMP,  -- the argument of computed column part
    part STRING,            -- partition key
    PRIMARY KEY (id, part) NOT ENFORCED
) PARTITIONED BY (part);
```

然后即可提交同步作业：

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    pulsar_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --computed_column 'part=date_format(create_time,yyyy-MM-dd)' \
    ... (other conf)
```

示例 3：
对于某些追加型数据（例如日志数据），可以将其视为只有 INSERT 操作类型的特殊 CDC 数据，因此你可以使用 'format=json' 将此类数据同步到 Paimon 表中。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    kafka_sync_table \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --table test_table \
    --partition_keys pt \
    --computed_column 'pt=date_format(event_tm, yyyyMMdd)' \
    --kafka_conf properties.bootstrap.servers=127.0.0.1:9020 \
    --kafka_conf topic=test_log \
    --kafka_conf properties.group.id=123456 \
    --kafka_conf value.format=json \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf sink.parallelism=4
```

## 同步数据库 {#synchronizing-databases}

通过在 Flink DataStream 作业中使用 [PulsarSyncDatabaseAction](https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/cdc/pulsar/PulsarSyncDatabaseAction)，或直接通过 `flink run`，用户可以将多个 topic 或一个 topic 同步到一个 Paimon 数据库中。

要通过 `flink run` 使用该功能，请运行以下 shell 命令。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    pulsar_sync_database \
    --warehouse <warehouse-path> \
    --database <database-name> \
    [--table_prefix <paimon-table-prefix>] \
    [--table_suffix <paimon-table-suffix>] \
    [--including_tables <table-name|name-regular-expr>] \
    [--excluding_tables <table-name|name-regular-expr>] \
    [--type_mapping to-string] \
    [--partition_keys <partition_keys>] \
    [--primary_keys <primary-keys>] \
    [--pulsar_conf <pulsar-source-conf> [--pulsar_conf <pulsar-source-conf> ...]] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]] \
    [--table_conf <paimon-table-sink-conf> [--table_conf <paimon-table-sink-conf> ...]]
```

```mdx-code-block
```mdx-code-block
<ConfigTable html={pulsarSyncDatabaseHtml} />
```
```

只有具有主键的表才会被同步。

该 Action 将为所有表构建一个统一的组合 Sink。对于每张待同步的 Pulsar topic 表，如果对应的 Paimon 表不存在，该 Action 将自动创建该表，其 schema 将派生自所有指定的 Pulsar topic 表。如果该 Paimon 表已存在，且其 schema 与从 Pulsar 记录中解析出的 schema 不同，该 Action 将尝试执行 schema 演进。

示例

从一个 Pulsar topic 同步到 Paimon 数据库。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    pulsar_sync_database \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --pulsar_conf topic=order \
    --pulsar_conf value.format=canal-json \
    --pulsar_conf pulsar.client.serviceUrl=pulsar://127.0.0.1:6650 \
    --pulsar_conf pulsar.admin.adminUrl=http://127.0.0.1:8080 \
    --pulsar_conf pulsar.consumer.subscriptionName=paimon-tests \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

从多个 Pulsar topic 同步到 Paimon 数据库。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    pulsar_sync_database \
    --warehouse hdfs:///path/to/warehouse \
    --database test_db \
    --pulsar_conf topic=order,logistic_order,user \
    --pulsar_conf value.format=canal-json \
    --pulsar_conf pulsar.client.serviceUrl=pulsar://127.0.0.1:6650 \
    --pulsar_conf pulsar.admin.adminUrl=http://127.0.0.1:8080 \
    --pulsar_conf pulsar.consumer.subscriptionName=paimon-tests \
    --catalog_conf metastore=hive \
    --catalog_conf uri=thrift://hive-metastore:9083 \
    --table_conf bucket=4 \
    --table_conf changelog-producer=input \
    --table_conf sink.parallelism=4
```

## 其他 pulsar_config {#additional-pulsar-config}

有一些用于构建 Flink Pulsar Source 的实用配置项，但它们并未在 flink-pulsar-connector 文档中提供。它们是：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">Key</th>
        <th class="text-left">默认值</th>
        <th class="text-left">类型</th>
        <th class="text-left">描述</th>
      </tr>
    </thead>
    <tbody>
        <tr>
         <td>value.format</td>
          <td>(none)</td>
          <td>String</td>
          <td>定义用于对 value 数据进行编码的格式标识符。</td>
        </tr>
        <tr>
          <td>topic</td>
          <td>(none)</td>
          <td>String</td>
          <td>读取数据的 topic 名称。它还支持通过分号分隔多个 topic 来指定 topic 列表，
              例如 'topic-1;topic-2'。注意，"topic-pattern" 和 "topic" 只能指定其中之一。
          </td>
        </tr>
        <tr>
          <td>topic-pattern</td>
          <td>(none)</td>
          <td>String</td>
          <td>用于匹配待读取 topic 名称模式的正则表达式。当作业开始运行时，所有名称与指定正则表达式
              匹配的 topic 都将被消费者订阅。注意，"topic-pattern" 和 "topic" 只能指定其中之一。
          </td>
        </tr>
        <tr>
          <td>pulsar.startCursor.fromMessageId</td>
          <td>EARLIEST</td>
          <td>Sting</td>
          <td>使用单条消息的唯一标识符来定位起始位置。常见格式为三元组
              '&ltlong&gtledgerId,&ltlong&gtentryId,&ltint&gtpartitionIndex'。特别地，你可以将其设置为
              EARLIEST (-1, -1, -1) 或 LATEST (Long.MAX_VALUE, Long.MAX_VALUE, -1)。
          </td>
        </tr>
        <tr>
          <td>pulsar.startCursor.fromPublishTime</td>
          <td>(none)</td>
          <td>Long</td>
          <td>使用消息发布时间来定位起始位置。</td>
        </tr>
        <tr>
          <td>pulsar.startCursor.fromMessageIdInclusive</td>
          <td>true</td>
          <td>Boolean</td>
          <td>是否包含给定的 message id。该选项仅在 message id 不为 EARLIEST 或 LATEST 时生效。</td>
        </tr>
        <tr>
          <td>pulsar.stopCursor.atMessageId</td>
          <td>(none)</td>
          <td>String</td>
          <td>当 message id 等于或大于指定的 message id 时停止消费。等于指定 message id 的消息
              将不会被消费。常见格式为三元组 '&ltlong&gtledgerId,&ltlong&gtentryId,&ltint&gtpartitionIndex'。
              特别地，你可以将其设置为 LATEST (Long.MAX_VALUE, Long.MAX_VALUE, -1)。
        <tr>
          <td>pulsar.stopCursor.afterMessageId</td>
          <td>(none)</td>
          <td>String</td>
          <td>当 message id 大于指定的 message id 时停止消费。等于指定 message id 的消息
              将会被消费。常见格式为三元组 '&ltlong&gtledgerId,&ltlong&gtentryId,&ltint&gtpartitionIndex'。
              特别地，你可以将其设置为 LATEST (Long.MAX_VALUE, Long.MAX_VALUE, -1)。
          </td>
        </tr>
        <tr>
          <td>pulsar.stopCursor.atEventTime</td>
          <td>(none)</td>
          <td>Long</td>
          <td>当消息事件时间大于或等于指定时间戳时停止消费。
              事件时间等于指定时间戳的消息将不会被消费。
          </td>
        </tr>
        <tr>
          <td>pulsar.stopCursor.afterEventTime</td>
          <td>(none)</td>
          <td>Long</td>
          <td>当消息事件时间大于指定时间戳时停止消费。
              事件时间等于指定时间戳的消息将会被消费。
          </td>
        </tr>
        <tr>
          <td>pulsar.source.unbounded</td>
          <td>true</td>
          <td>Boolean</td>
          <td>用于指定流的有界性。</td>
        </tr>
        <tr>
          <td>schema.registry.url</td>
          <td>(none)</td>
          <td>String</td>
          <td>当配置 "value.format=debezium-avro" 时，需要使用 Confluence schema registry 模型进行 Apache Avro 序列化，此时你需要提供 schema registry URL。</td>
        </tr>
    </tbody>
</table>
