---
title: "Structured Streaming"
sidebar_position: 11
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

# Structured Streaming {#structured-streaming}

Paimon 通过 [Spark Structured Streaming](https://spark.apache.org/docs/latest/streaming/index.html) 支持流式数据处理，可同时实现流式写入与流式查询。

## 流式写入 {#streaming-write}

:::info

Paimon 的 Structured Streaming 仅支持 `append` 和 `complete` 两种模式。

:::

```scala
// Create a paimon table if not exists.
spark.sql(s"""
           |CREATE TABLE T (k INT, v STRING)
           |TBLPROPERTIES ('primary-key'='k', 'bucket'='3')
           |""".stripMargin)

// Here we use MemoryStream to fake a streaming source.
val inputData = MemoryStream[(Int, String)]
val df = inputData.toDS().toDF("k", "v")

// Streaming Write to paimon table.
val stream = df
  .writeStream
  .outputMode("append")
  .option("checkpointLocation", "/path/to/checkpoint")
  .format("paimon")
  .start("/path/to/paimon/sink/table")
```

流式写入同样支持 [Write merge schema](./sql-write#write-merge-schema)。

## 流式查询 {#streaming-query}

:::info

Paimon 目前支持在 Spark 3.3+ 上进行流式读取。

:::

Paimon 支持丰富的流式读取扫描模式。下表列出了这些模式：
<table class="configuration table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 20%">扫描模式</th>
            <th class="text-left" style="width: 60%">描述</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><h5>latest</h5></td>
            <td>对于流式 Source，持续读取最新的变更，启动时不产出快照。</td>
        </tr>
        <tr>
            <td><h5>latest-full</h5></td>
            <td>对于流式 Source，首次启动时产出表的最新快照，随后继续读取最新的变更。</td>
        </tr>
        <tr>
            <td><h5>from-timestamp</h5></td>
            <td>对于流式 Source，从 "scan.timestamp-millis" 指定的时间戳开始持续读取变更，启动时不产出快照。</td>
        </tr>
        <tr>
            <td><h5>from-snapshot</h5></td>
            <td>对于流式 Source，从 "scan.snapshot-id" 指定的快照开始持续读取变更，启动时不产出快照。</td>
        </tr>
        <tr>
            <td><h5>from-snapshot-full</h5></td>
            <td>对于流式 Source，首次启动时从表中 "scan.snapshot-id" 指定的快照开始产出，随后持续读取变更。</td>
        </tr>
        <tr>
            <td><h5>default</h5></td>
            <td>如果指定了 "scan.snapshot-id"，则等价于 from-snapshot；如果指定了 "timestamp-millis"，则等价于 from-timestamp；否则等价于 latest-full。</td>
        </tr>
    </tbody>
</table>

使用默认扫描模式的一个简单示例：

```scala
// no any scan-related configs are provided, that will use latest-full scan mode.
val query = spark.readStream
  .format("paimon")
  // by table name
  .table("table_name") 
  // or by location
  // .load("/path/to/paimon/source/table")
  .writeStream
  .format("console")
  .start()
```

Paimon 的 Structured Streaming 还支持多种流式读取模式，可支持多种触发器（Trigger）和多种读取限制。

支持以下读取限制：

<table class="configuration table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 20%">配置项</th>
            <th class="text-left" style="width: 15%">默认值</th>
            <th class="text-left" style="width: 10%">类型</th>
            <th class="text-left" style="width: 55%">描述</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><h5>read.stream.maxFilesPerTrigger</h5></td>
            <td style="word-wrap: break-word;">(none)</td>
            <td>Integer</td>
            <td>单个批次返回的最大文件数。</td>
        </tr>
        <tr>
            <td><h5>read.stream.maxBytesPerTrigger</h5></td>
            <td style="word-wrap: break-word;">(none)</td>
            <td>Long</td>
            <td>单个批次返回的最大字节数。</td>
        </tr>
        <tr>
            <td><h5>read.stream.maxRowsPerTrigger</h5></td>
            <td style="word-wrap: break-word;">(none)</td>
            <td>Long</td>
            <td>单个批次返回的最大行数。</td>
        </tr>
        <tr>
            <td><h5>read.stream.minRowsPerTrigger</h5></td>
            <td style="word-wrap: break-word;">(none)</td>
            <td>Long</td>
            <td>单个批次返回的最小行数，与 read.stream.maxTriggerDelayMs 一起用于创建 MinRowsReadLimit。</td>
        </tr>
        <tr>
            <td><h5>read.stream.maxTriggerDelayMs</h5></td>
            <td style="word-wrap: break-word;">(none)</td>
            <td>Long</td>
            <td>两个相邻批次之间的最大延迟，与 read.stream.minRowsPerTrigger 一起用于创建 MinRowsReadLimit。</td>
        </tr>
    </tbody>
</table>

**示例一**

使用 `org.apache.spark.sql.streaming.Trigger.AvailableNow()` 以及由 Paimon 定义的 `maxBytesPerTrigger`。

```scala
// Trigger.AvailableNow()) processes all available data at the start
// of the query in one or multiple batches, then terminates the query.
// That set read.stream.maxBytesPerTrigger to 128M means that each
// batch processes a maximum of 128 MB of data.
val query = spark.readStream
  .format("paimon")
  .option("read.stream.maxBytesPerTrigger", "134217728")
  .table("table_name")
  .writeStream
  .format("console")
  .trigger(Trigger.AvailableNow())
  .start()
```

**示例二**

使用 `org.apache.spark.sql.connector.read.streaming.ReadMinRows`。

```scala
// It will not trigger a batch until there are more than 5,000 pieces of data,
// unless the interval between the two batches is more than 300 seconds.
val query = spark.readStream
  .format("paimon")
  .option("read.stream.minRowsPerTrigger", "5000")
  .option("read.stream.maxTriggerDelayMs", "300000")
  .table("table_name")
  .writeStream
  .format("console")
  .start()
```

Paimon 的 Structured Streaming 支持以 changelog（变更日志）的形式读取行（在行中增加 rowkind 列以表示其变更类型），有两种方式：

- 直接对系统表 audit_log 进行流式读取
- 将 `read.changelog` 设为 true（默认为 false），然后通过表位置进行流式读取

**示例：**

```scala
// Option 1
val query1 = spark.readStream
  .format("paimon")
  .table("`table_name$audit_log`")
  .writeStream
  .format("console")
  .start()

// Option 2
val query2 = spark.readStream
  .format("paimon")
  .option("read.changelog", "true")
  .table("table_name")
  .writeStream
  .format("console")
  .start()

/*
+I   1  Hi
+I   2  Hello
*/
```
