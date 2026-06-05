---
title: "管理分区"
sidebar_position: 12
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

# 管理分区 {#manage-partitions}
Paimon 提供了多种管理分区的方式，包括按不同策略让历史分区过期，或将分区标记为完成（mark done），以通知下游应用该分区已经写入完毕。

## 分区过期 {#expiring-partitions}

你可以在创建分区表时设置 `partition.expiration-time`。Paimon 流式 Sink 会周期性地检查分区状态，并按时间删除已过期的分区。

如何判断分区是否已过期：你可以在创建分区表时设置 `partition.expiration-strategy`，该策略决定了如何提取分区时间并与当前时间进行比较，以判断其存活时间是否已经超过 `partition.expiration-time`。过期策略支持的取值如下：

- `values-time` ：该策略将从分区值中提取的时间与当前时间进行比较，此为默认策略。
- `update-time` ：该策略将分区的最近更新时间与当前时间进行比较。
该策略适用的场景：
   - 你的分区值不是日期格式。
   - 你只想保留最近 n 天 / 月 / 年内有过更新的数据。
   - 数据初始化时导入了大量历史数据。

:::info

__注意：__ 分区过期后，它会被逻辑删除，最新快照将无法查询到其数据。但文件系统中的文件不会立即被物理删除，何时物理删除取决于对应快照何时过期。
参见 [过期快照](./manage-snapshots#expire-snapshots)。

:::

单个分区字段的示例：

`values-time` 策略。
```sql
CREATE TABLE t (...) PARTITIONED BY (dt) WITH (
    'partition.expiration-time' = '7 d',
    'partition.expiration-check-interval' = '1 d',
    'partition.timestamp-formatter' = 'yyyyMMdd'   -- this is required in `values-time` strategy.
);
-- Let's say now the date is 2024-07-09，so before the date of 2024-07-02 will expire.
insert into t values('pk', '2024-07-01');

-- An example for multiple partition fields
CREATE TABLE t (...) PARTITIONED BY (other_key, dt) WITH (
    'partition.expiration-time' = '7 d',
    'partition.expiration-check-interval' = '1 d',
    'partition.timestamp-formatter' = 'yyyyMMdd',
    'partition.timestamp-pattern' = '$dt'
);
```

`update-time` 策略。
```sql
CREATE TABLE t (...) PARTITIONED BY (dt) WITH (
    'partition.expiration-time' = '7 d',
    'partition.expiration-check-interval' = '1 d',
    'partition.expiration-strategy' = 'update-time'
);

-- The last update time of the partition is now, so it will not expire.
insert into t values('pk', '2024-01-01');
-- Support non-date formatted partition.
insert into t values('pk', 'par-1'); 

```

更多配置项：

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
            <td><h5>partition.expiration-strategy</h5></td>
            <td style="word-wrap: break-word;">values-time</td>
            <td>String</td>
            <td>
                指定分区过期所使用的过期策略。
                可选值：
                <li>values-time：该策略将从分区值中提取的时间与当前时间进行比较。</li>
                <li>update-time：该策略将分区的最近更新时间与当前时间进行比较。</li>
            </td>
        </tr>
        <tr>
            <td><h5>partition.expiration-check-interval</h5></td>
            <td style="word-wrap: break-word;">1 h</td>
            <td>Duration</td>
            <td>分区过期的检查间隔。</td>
        </tr>
        <tr>
            <td><h5>partition.expiration-time</h5></td>
            <td style="word-wrap: break-word;">(none)</td>
            <td>Duration</td>
            <td>分区的过期间隔。如果某个分区的存活时间超过此值，该分区将会过期。分区时间从分区值中提取。</td>
        </tr>
        <tr>
            <td><h5>partition.timestamp-formatter</h5></td>
            <td style="word-wrap: break-word;">(none)</td>
            <td>String</td>
            <td>用于从字符串格式化时间戳的格式化器。它可以与 'partition.timestamp-pattern' 配合使用，以根据指定值创建格式化器。<ul><li>默认格式化器为 'yyyy-MM-dd HH:mm:ss' 和 'yyyy-MM-dd'。</li><li>支持多个分区字段，例如 '$year-$month-$day $hour:00:00'。</li><li>该 timestamp-formatter 与 Java 的 DateTimeFormatter 兼容。</li></ul></td>
        </tr>
        <tr>
            <td><h5>partition.timestamp-pattern</h5></td>
            <td style="word-wrap: break-word;">(none)</td>
            <td>String</td>
            <td>你可以指定一个模式（pattern）来从分区中获取时间戳。该格式化器模式由 'partition.timestamp-formatter' 定义。<ul><li>默认情况下，从第一个字段读取。</li><li>如果分区中的时间戳是名为 'dt' 的单个字段，你可以使用 '$dt'。</li><li>如果它分散在 year、month、day、hour 等多个字段中，你可以使用 '$year-$month-$day $hour:00:00'。</li><li>如果时间戳位于 dt 和 hour 两个字段中，你可以使用 '$dt $hour:00:00'。</li></ul></td>
        </tr>
        <tr>
            <td><h5>end-input.check-partition-expire</h5></td>
            <td style="word-wrap: break-word;">false</td>
            <td>Boolean</td>
            <td>在批模式或有界流作业结束后，是否检查分区过期。</td>
        </tr>
    </tbody>
</table>

## 分区标记完成 {#partition-mark-done}

你可以使用配置项 `'partition.mark-done-action'` 来配置在需要将分区标记完成时所执行的动作。
- `success-file`：向目录中添加 '_success' 文件。
- `done-partition`：向 metastore 添加 'xxx.done' 分区。
- `mark-event`：向 metastore 标记分区事件。
- `http-report`：向远程 http 服务器上报分区标记完成。
- `custom`：使用策略类来创建一个分区标记策略。
这些动作可以同时配置：'done-partition,success-file,mark-event,custom'。

Paimon 的分区标记完成既可以由流式写入触发，也可以由批式写入触发。

### 流式标记完成 {#streaming-mark-done}

你可以使用配置项 `'partition.idle-time-to-done'` 来设置分区从空闲到标记完成的时长。当某个分区在该时长之后没有新数据写入时，便会触发标记完成动作，以表明数据已就绪。

默认情况下，Flink 会使用处理时间（process time）作为空闲时间来触发分区标记完成。你也可以使用 Watermark 来触发分区标记完成，这样在数据延迟的情况下能让分区标记完成的时间更加准确。你可以通过设置 `'partition.mark-done-action.mode' = 'watermark'` 来启用此功能。

### 批式标记完成 {#batch-mark-done}

对于批模式，你可以通过设置 `'partition.end-input-to-done'='true'` 在输入结束时触发分区标记完成，此时该批次中写入的所有分区都将被标记完成。
