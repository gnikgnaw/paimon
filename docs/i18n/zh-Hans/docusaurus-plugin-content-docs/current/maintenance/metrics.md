---
title: "指标（Metrics）"
sidebar_position: 9
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

# Paimon 指标（Metrics） {#paimon-metrics}

Paimon 构建了一套指标（Metrics）系统，用于度量读取与写入的行为，例如上一次规划扫描了多少个 manifest（清单）文件、上一次提交操作耗时多久、上一次 Compaction 操作删除了多少个文件。

在 Paimon 的指标系统中，指标以表为粒度进行更新和上报。

Paimon 指标系统提供三种类型的指标：`Gauge`、`Counter`、`Histogram`。
- `Gauge`：提供某个时间点上任意类型的值。
- `Counter`：通过递增和递减来对值进行计数。
- `Histogram`：度量一组值的统计分布，包括最小值、最大值、平均值、标准差和分位数。

Paimon 已支持内置指标来度量 **commit（提交）**、**scan（扫描）**、**write（写入）** 和 **compaction（Compaction）** 操作，这些指标可以桥接到任何受支持的计算引擎，例如 Flink、Spark 等。

## 指标列表 {#metrics-list}

下面列出 Paimon 内置指标。它们被归纳为扫描指标、提交指标、写入指标、写缓冲区指标和 Compaction 指标几大类。

### 扫描指标 {#scan-metrics}

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 225pt">指标名称</th>
      <th class="text-left" style="width: 70pt">类型</th>
      <th class="text-left" style="width: 300pt">说明</th>
    </tr>
    </thead>
    <tbody>
        <tr>
            <td>lastScanDuration</td>
            <td>Gauge</td>
            <td>完成上一次扫描所耗费的时间。</td>
        </tr>
        <tr>
            <td>scanDuration</td>
            <td>Histogram</td>
            <td>最近若干次扫描所耗时间的分布。</td>
        </tr>
        <tr>
            <td>lastScannedManifests</td>
            <td>Gauge</td>
            <td>上一次扫描中已扫描的 manifest 文件数量。</td>
        </tr>
        <tr>
            <td>lastScanSkippedTableFiles</td>
            <td>Gauge</td>
            <td>上一次扫描中被跳过的表文件总数。</td>
        </tr>
        <tr>
            <td>lastScanResultedTableFiles</td>
            <td>Gauge</td>
            <td>上一次扫描结果中的表文件数量。</td>
        </tr>
    </tbody>
</table>

### 提交指标 {#commit-metrics}

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 225pt">指标名称</th>
      <th class="text-left" style="width: 70pt">类型</th>
      <th class="text-left" style="width: 300pt">说明</th>
    </tr>
    </thead>
    <tbody>
        <tr>
            <td>lastCommitDuration</td>
            <td>Gauge</td>
            <td>完成上一次提交所耗费的时间。</td>
        </tr>
        <tr>
            <td>commitDuration</td>
            <td>Histogram</td>
            <td>最近若干次提交所耗时间的分布。</td>
        </tr>
        <tr>
            <td>lastCommitAttempts</td>
            <td>Gauge</td>
            <td>上一次提交所做的尝试次数。</td>
        </tr>
        <tr>
            <td>lastTableFilesAdded</td>
            <td>Gauge</td>
            <td>上一次提交中新增的表文件数量，包括新创建的数据文件以及 Compaction 之后产生的文件。</td>
        </tr>
        <tr>
            <td>lastTableFilesDeleted</td>
            <td>Gauge</td>
            <td>上一次提交中删除的表文件数量，这些文件来自 Compaction 之前的文件。</td>
        </tr>
        <tr>
            <td>lastTableFilesAppended</td>
            <td>Gauge</td>
            <td>上一次提交中追加的表文件数量，即新创建的数据文件。</td>
        </tr>
        <tr>
            <td>lastTableFilesCommitCompacted</td>
            <td>Gauge</td>
            <td>上一次提交中经过 Compaction 的表文件数量，包括 Compaction 之前和之后的文件。</td>
        </tr>
        <tr>
            <td>lastChangelogFilesAppended</td>
            <td>Gauge</td>
            <td>上一次提交中追加的 changelog 文件数量。</td>
        </tr>
        <tr>
            <td>lastChangelogFileCommitCompacted</td>
            <td>Gauge</td>
            <td>上一次提交中经过 Compaction 的 changelog 文件数量。</td>
        </tr>
        <tr>
            <td>lastGeneratedSnapshots</td>
            <td>Gauge</td>
            <td>上一次提交中生成的快照文件数量，可能是 1 个快照或 2 个快照。</td>
        </tr>
        <tr>
            <td>lastDeltaRecordsAppended</td>
            <td>Gauge</td>
            <td>上一次提交中以 APPEND 提交类型追加的 delta 记录数。</td>
        </tr>
        <tr>
            <td>lastChangelogRecordsAppended</td>
            <td>Gauge</td>
            <td>上一次提交中以 APPEND 提交类型追加的 changelog 记录数。</td>
        </tr>
        <tr>
            <td>lastDeltaRecordsCommitCompacted</td>
            <td>Gauge</td>
            <td>上一次提交中以 COMPACT 提交类型产生的 delta 记录数。</td>
        </tr>
        <tr>
            <td>lastChangelogRecordsCommitCompacted</td>
            <td>Gauge</td>
            <td>上一次提交中以 COMPACT 提交类型产生的 changelog 记录数。</td>
        </tr>
        <tr>
            <td>lastPartitionsWritten</td>
            <td>Gauge</td>
            <td>上一次提交中写入的分区数量。</td>
        </tr>
        <tr>
            <td>lastBucketsWritten</td>
            <td>Gauge</td>
            <td>上一次提交中写入的桶数量。</td>
        </tr>
        <tr>
            <td>lastCompactionInputFileSize</td>
            <td>Gauge</td>
            <td>上一次 Compaction 输入文件的总大小。</td>
        </tr>
        <tr>
            <td>lastCompactionOutputFileSize</td>
            <td>Gauge</td>
            <td>上一次 Compaction 输出文件的总大小。</td>
        </tr>
    </tbody>
</table>

### 写缓冲区指标 {#write-buffer-metrics}

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 225pt">指标名称</th>
      <th class="text-left" style="width: 70pt">类型</th>
      <th class="text-left" style="width: 300pt">说明</th>
    </tr>
    </thead>
    <tbody>
        <tr>
            <td>numWriters</td>
            <td>Gauge</td>
            <td>该并行度下的 writer 数量。</td>
        </tr>
        <tr>
            <td>bufferPreemptCount</td>
            <td>Gauge</td>
            <td>内存被抢占的总次数。</td>
        </tr>
        <tr>
            <td>usedWriteBufferSizeByte</td>
            <td>Gauge</td>
            <td>当前已使用的写缓冲区大小（字节）。</td>
        </tr>
        <tr>
            <td>totalWriteBufferSizeByte</td>
            <td>Gauge</td>
            <td>配置的写缓冲区总大小（字节）。</td>
        </tr>
    </tbody>
</table>

### Compaction 指标 {#compaction-metrics}

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 225pt">指标名称</th>
      <th class="text-left" style="width: 70pt">类型</th>
      <th class="text-left" style="width: 300pt">说明</th>
    </tr>
    </thead>
    <tbody>
        <tr>
            <td>maxLevel0FileCount</td>
            <td>Gauge</td>
            <td>该任务当前处理的 level 0 文件的最大数量。如果异步 Compaction 无法及时完成，该值会变大。</td>
        </tr>
        <tr>
            <td>avgLevel0FileCount</td>
            <td>Gauge</td>
            <td>该任务当前处理的 level 0 文件的平均数量。如果异步 Compaction 无法及时完成，该值会变大。</td>
        </tr>
        <tr>
            <td>compactionThreadBusy</td>
            <td>Gauge</td>
            <td>该任务中 Compaction 线程的最大繁忙度。目前每个并行度中只有一个 Compaction 线程，因此繁忙度的取值范围为 0（空闲）到 100（Compaction 一直在运行）。</td>
        </tr>
        <tr>
            <td>avgCompactionTime</td>
            <td>Gauge</td>
            <td>Compaction 线程的平均运行时间，基于记录的 Compaction 时间数据计算，单位为毫秒。该值表示 Compaction 操作的平均时长。值越高表示平均 Compaction 时间越长，可能意味着需要进行性能优化。</td>
        </tr>
       <tr>
            <td>compactionCompletedCount</td>
            <td>Counter</td>
            <td>已完成的 Compaction 总数。</td>
        </tr>
        <tr>
            <td>compactionQueuedCount</td>
            <td>Counter</td>
            <td>排队中/正在运行的 Compaction 总数。</td>
        </tr>
        <tr>
            <td>compactionTotalCount</td>
            <td>Counter</td>
            <td>Compaction 的总数。</td>
        </tr>
        <tr>
            <td>maxCompactionInputSize</td>
            <td>Gauge</td>
            <td>该任务 Compaction 的最大输入文件大小。</td>
        </tr>
        <tr>
            <td>avgCompactionInputSize</td>
            <td>Gauge</td>
            <td>该任务 Compaction 的平均输入文件大小。</td>
        </tr>
        <tr>
            <td>maxCompactionOutputSize</td>
            <td>Gauge</td>
            <td>该任务 Compaction 的最大输出文件大小。</td>
        </tr>
        <tr>
            <td>avgCompactionOutputSize</td>
            <td>Gauge</td>
            <td>该任务 Compaction 的平均输出文件大小。</td>
        </tr>
        <tr>
            <td>maxTotalFileSize</td>
            <td>Gauge</td>
            <td>处于活跃状态（当前正在写入）的桶的最大文件总大小。</td>
        </tr>
        <tr>
            <td>avgTotalFileSize</td>
            <td>Gauge</td>
            <td>所有处于活跃状态（当前正在写入）的桶的平均文件总大小。</td>
        </tr>
        <tr>
            <td>maxSortBufferUsedBytes</td>
            <td>Gauge</td>
            <td>所有活跃 Compaction 桶当前使用的最大排序缓冲区内存（字节）。相对于 <code>maxSortBufferTotalBytes</code> 而言数值较高时，表示 Compaction 期间存在内存压力；可以考虑调低 <code>sort-spill-threshold</code> 或减小 <code>sort-spill-buffer-size</code>。</td>
        </tr>
        <tr>
            <td>avgSortBufferUsedBytes</td>
            <td>Gauge</td>
            <td>所有活跃 Compaction 桶使用的平均排序缓冲区内存（字节）。</td>
        </tr>
        <tr>
            <td>maxSortBufferUtilisationPercent</td>
            <td>Gauge</td>
            <td>所有活跃 Compaction 桶的最大排序缓冲区利用率百分比（0–100）。该值持续接近 100 表示排序缓冲区池已耗尽，正在或即将发生向磁盘的溢写。</td>
        </tr>
        <tr>
            <td>avgSortBufferUtilisationPercent</td>
            <td>Gauge</td>
            <td>所有活跃 Compaction 桶的平均排序缓冲区利用率百分比。</td>
        </tr>
    </tbody>
</table>

## 桥接到 Flink {#bridging-to-flink}

Paimon 已实现将指标桥接到 Flink 的指标系统，使其可由 Flink 上报，并且指标组的生命周期由 Flink 管理。

在使用 Flink 访问 Paimon 时，请将 `<scope>.<infix>.<metric_name>` 拼接起来以获得完整的指标标识符，其中 `metric_name` 可从 [指标列表](./metrics#metrics-list) 中获取。

例如，在名为 `insert_word_count` 的 Flink 作业中，表 `word_count` 的指标 `lastPartitionsWritten` 的标识符为：

`localhost.taskmanager.localhost:60340-775a20.insert_word_count.Global Committer : word_count.0.paimon.table.word_count.commit.lastPartitionsWritten`。

在 Flink Web-UI 中，进入 committer 算子的指标页面，它显示为：

`0.Global_Committer___word_count.paimon.table.word_count.commit.lastPartitionsWritten`。

:::info

1. 请参阅 [System Scope](https://nightlies.apache.org/flink/flink-docs-master/docs/ops/metrics/#system-scope) 以理解 Flink 的 `scope`
2. 扫描指标仅被 Flink 版本 >= 1.18 支持

:::

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 130pt"></th>
      <th class="text-left" style="width: 280pt">Scope</th>
      <th class="text-left" style="width: 250pt">Infix</th>
    </tr>
    </thead>
    <tbody>
        <tr>
            <td>扫描指标</td>
            <td>&lt;host&gt;.jobmanager.&lt;job_name&gt;</td>
            <td>&lt;source_operator_name&gt;.coordinator. enumerator.paimon.table.&lt;table_name&gt;.scan</td>
        </tr>
        <tr>
            <td>提交指标</td>
            <td>&lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;committer_operator_name&gt;.&lt;subtask_index&gt;</td>
            <td>paimon.table.&lt;table_name&gt;.commit</td>
        </tr>
        <tr>
            <td>写入指标</td>
            <td>&lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;writer_operator_name&gt;.&lt;subtask_index&gt;</td>
            <td>paimon.table.&lt;table_name&gt;.partition.&lt;partition_string&gt;.bucket.&lt;bucket_index&gt;.writer</td>
        </tr>
        <tr>
            <td>写缓冲区指标</td>
            <td>&lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;writer_operator_name&gt;.&lt;subtask_index&gt;</td>
            <td>paimon.table.&lt;table_name&gt;.writeBuffer</td>
        </tr>
        <tr>
            <td>Compaction 指标</td>
            <td>&lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;writer_operator_name&gt;.&lt;subtask_index&gt;</td>
            <td>paimon.table.&lt;table_name&gt;.partition.&lt;partition_string&gt;.bucket.&lt;bucket_index&gt;.compaction</td>
        </tr>
        <tr>
            <td>Flink Source 指标</td>
            <td>&lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;source_operator_name&gt;.&lt;subtask_index&gt;</td>
            <td>-</td>
        </tr>
        <tr>
            <td>Flink Sink 指标</td>
            <td>&lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;committer_operator_name&gt;.&lt;subtask_index&gt;</td>
            <td>-</td>
        </tr>  
    </tbody>
</table>

### Flink 连接器标准指标 {#flink-connector-standard-metrics}

在使用 Flink 进行读写时，Paimon 实现了一些关键的标准 Flink 连接器指标，用于度量 Source 的延迟和 Sink 的输出，参见 [FLIP-33: Standardize Connector Metrics](https://cwiki.apache.org/confluence/display/FLINK/FLIP-33%3A+Standardize+Connector+Metrics)。这里列出已实现的 Flink source / sink 指标。

#### Source 指标（Flink） {#source-metrics-flink}

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 225pt">指标名称</th>
      <th class="text-left" style="width: 65pt">级别</th>
      <th class="text-left" style="width: 70pt">类型</th>
      <th class="text-left" style="width: 300pt">说明</th>
    </tr>
    </thead>
    <tbody>
        <tr>
            <td>currentEmitEventTimeLag</td>
            <td>Flink Source 算子</td>
            <td>Gauge</td>
            <td>将记录从 Source 发出与文件创建之间的时间差。</td>
        </tr>
        <tr>
            <td>currentFetchEventTimeLag</td>
            <td>Flink Source 算子</td>
            <td>Gauge</td>
            <td>读取数据文件与文件创建之间的时间差。</td>
        </tr>
        <tr>
            <td>sourceParallelismUpperBound</td>
            <td>Flink Source Enumerator</td>
            <td>Gauge</td>
            <td>面向自动扩缩容系统推荐的并行度上限。注意：这是一个推荐值，而非硬性限制。</td>
        </tr>
    </tbody>
</table>

:::info

请注意，如果你在流式查询中指定了 `consumer-id`，那么 Source 指标的级别应转为 reader 算子，该算子位于 `Monitor` 算子之后。

:::

#### Sink 指标（Flink） {#sink-metrics-flink}

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 225pt">指标名称</th>
      <th class="text-left" style="width: 65pt">级别</th>
      <th class="text-left" style="width: 70pt">类型</th>
      <th class="text-left" style="width: 300pt">说明</th>
    </tr>
    </thead>
    <tbody>
        <tr>
            <td>numBytesOut</td>
            <td>Table</td>
            <td>Counter</td>
            <td>输出字节的总数。</td>
        </tr>
        <tr>
            <td>numBytesOutPerSecond</td>
            <td>Table</td>
            <td>Meter</td>
            <td>每秒输出的字节数。</td>
        </tr>
        <tr>
            <td>numRecordsOut</td>
            <td>Table</td>
            <td>Counter</td>
            <td>输出记录的总数。</td>
        </tr>
        <tr>
            <td>numRecordsOutPerSecond</td>
            <td>Table</td>
            <td>Meter</td>
            <td>每秒输出的记录数。</td>
        </tr>  
    </tbody>
</table>
