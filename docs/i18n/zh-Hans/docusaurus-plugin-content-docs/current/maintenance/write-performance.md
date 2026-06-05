---
title: "写性能"
sidebar_position: 3
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

# 写性能 {#write-performance}

Paimon 的写性能与 checkpoint 密切相关，因此如果你需要更高的写吞吐量：

1. Flink 配置（`'flink-conf.yaml'/'config.yaml'` 或在 SQL 中使用 `SET`）：增大 checkpoint 间隔
   （`'execution.checkpointing.interval'`），将最大并发 checkpoint 数增加到 3
   （`'execution.checkpointing.max-concurrent-checkpoints'`），或者直接使用批式模式。
2. 增大 `write-buffer-size`。
3. 启用 `write-buffer-spillable`。
4. 如果你使用固定分桶（Fixed-Bucket）模式，则调整桶数。

配置项 `'changelog-producer' = 'lookup' or 'full-compaction'` 以及配置项 `'full-compaction.delta-commits'` 对写性能有
较大影响，如果处于快照 / 全量同步阶段，你可以取消这些配置项，然后在增量阶段再次启用它们。

如果你发现作业的输入在出现反压时呈现锯齿状模式，这可能是工作节点不均衡所致。你可以考虑开启 [异步 Compaction](../primary-key-table/compaction#asynchronous-compaction) 来观察吞吐量
是否有所提升。

## 并行度 {#parallelism}

建议 Sink 的并行度应小于或等于桶数，最好相等。你可以通过 `sink.parallelism` 表属性来控制 Sink 的并行度。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 20%">配置项</th>
      <th class="text-left" style="width: 5%">是否必填</th>
      <th class="text-left" style="width: 5%">默认值</th>
      <th class="text-left" style="width: 10%">类型</th>
      <th class="text-left" style="width: 60%">描述</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>sink.parallelism</h5></td>
      <td>否</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>Integer</td>
      <td>定义 Sink 算子的并行度。默认情况下，并行度由框架决定，使用与上游链式算子相同的并行度。</td>
    </tr>
    </tbody>
</table>

## 本地合并 {#local-merging}

如果你的作业存在主键数据倾斜
（例如，你想统计网站中每个页面的浏览次数，
而某些特定页面在用户中非常受欢迎），
你可以设置 `'local-merge-buffer-size'`，使输入记录在按桶 shuffle 并写入 Sink 之前先被缓冲并合并。
这在同一主键在快照之间频繁更新时尤其有用。

缓冲区满时会被刷出。当你面临数据倾斜但不知道从何处开始调整缓冲区大小时，我们建议从 `64 mb`
开始尝试。

（目前，本地合并不适用于 CDC 接入）

## 文件格式 {#file-format}

如果你想获得极致的 Compaction 性能，可以考虑使用行存储文件格式 AVRO。
- 优点是你可以获得很高的写吞吐量和 Compaction 性能。
- 缺点是你的分析查询会变慢，行存储最大的问题是它
  没有查询投影。例如，如果表有 100 列但只查询其中几列，
  行存储的 IO 开销不可忽略。此外，压缩效率会下降，存储成本会
  增加。

这是一种权衡。

通过以下配置项启用行存储：
```properties
file.format = avro
metadata.stats-mode = none
```

行存储统计信息的收集开销有点大，因此我建议同时关闭统计
信息。

如果你不想把所有文件都改为 Avro 格式，那么至少可以考虑把前几
层的文件改为 Avro 格式。你可以使用 `'file.format.per.level' = '0:avro,1:avro'` 来指定前两
层的文件采用 Avro 格式。

## 文件压缩 {#file-compression}

默认情况下，Paimon 使用 level 为 1 的 zstd，你可以修改压缩算法：

`'file.compression.zstd-level'`：默认 zstd level 为 1。如需更高的压缩率，可以配置为 9，但读写速度会显著下降。

## 稳定性 {#stability}

如果桶数或资源过少，全量 Compaction 可能导致 checkpoint 超时，Flink 默认的
checkpoint 超时时间为 10 分钟。

如果在这种情况下你仍期望保持稳定，可以调高 checkpoint 超时时间，例如：

```properties
execution.checkpointing.timeout = 60 min
```

## 写初始化 {#write-initialize}

在写初始化时，桶的写入器需要读取所有历史文件。如果这里存在瓶颈
（例如，同时写入大量分区），你可以使用 `sink.writer-coordinator.enabled`
来使用 Flink coordinator 缓存读取到的 manifest 数据，从而加速初始化。coordinator 的缓存内存
由 `sink.writer-coordinator.cache-memory` 控制，默认在 Job Manager 中为 1GB。

## 写内存 {#write-memory}

Paimon 写入器中主要有三个地方占用内存：

* 写入器的内存缓冲区，由单个任务的所有写入器共享并抢占。该内存值可以通过 `write-buffer-size` 表属性进行调整。
* 在 Compaction 时合并多个 sorted run 所消耗的内存。可以通过 `num-sorted-run.compaction-trigger` 配置项调整待合并的 sorted run 数量。
* 如果行非常大，一次读取过多行数据会在进行 Compaction 时消耗大量内存。减小 `read.batch-size` 配置项可以缓解这种情况的影响。
* 写列式 ORC 文件所消耗的内存。减小 `orc.write.batch-size` 配置项可以减少 ORC 格式的内存消耗。
* 如果文件在写任务中自动 Compaction，某些大列的字典在 Compaction 期间可能显著消耗内存。
  * 要禁用 Parquet 格式中所有字段的字典编码，设置 `'parquet.enable.dictionary'= 'false'`。
  * 要禁用 ORC 格式中所有字段的字典编码，设置 `orc.dictionary.key.threshold='0'`。此外，设置 `orc.column.encoding.direct='field1,field2'` 可禁用特定列的字典编码。

如果你的 Flink 作业不依赖状态，请避免使用托管内存（managed memory），你可以通过以下 Flink 参数进行控制：
```properties
taskmanager.memory.managed.size=1m
```
或者你可以将 Flink 托管内存用于你的写缓冲区以避免 OOM，设置表属性：
```properties
sink.use-managed-memory-allocator=true
```

## 提交内存 {#commit-memory}

如果写入表的数据量特别大，Committer 节点可能会使用大量内存，如果内存太小可能会发生 OOM。
在这种情况下，你需要增加 Committer 的堆内存，但你可能不想统一增加
Flink TaskManager 的内存，那样可能会导致内存浪费。

你可以使用 Flink 的细粒度资源管理（fine-grained-resource-management）来仅增加 Committer 的堆内存：
1. 配置 Flink 配置 `cluster.fine-grained-resource-management.enabled: true`。（在 Flink 1.18 之后默认开启）
2. 配置 Paimon 表配置项：`sink.committer-memory`，例如 300 MB，具体取决于你的 `TaskManager`。
   （也支持 `sink.committer-cpu`）
3. 如果你使用 Flink 批式作业将数据写入 Paimon 或运行独立 Compaction，请配置 Flink 配置 `fine-grained.shuffle-mode.all-blocking: true`。
