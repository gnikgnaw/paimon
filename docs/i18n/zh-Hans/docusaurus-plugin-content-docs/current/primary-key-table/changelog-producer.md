---
title: "Changelog Producer"
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

# Changelog Producer {#changelog-producer}

流式写入可以为流式读取持续产生最新的变更。

通过在建表时指定 `changelog-producer` 表属性，用户可以选择从表文件中产生变更的模式。

:::info

`changelog-producer` 可能会显著降低 Compaction 性能，除非确有必要，否则请不要启用它。

:::

## None {#none}

默认情况下，不会对表的 writer 应用任何额外的 changelog producer。Paimon source 只能看到跨快照合并后的变更，例如哪些 key 被删除了、某些 key 的新值是什么。

然而，这些合并后的变更无法构成完整的 changelog，因为我们无法直接从中读取到 key 的旧值。合并后的变更要求消费者「记住」每个 key 的值，并在看不到旧值的情况下重写这些值。但是，某些消费者需要旧值来保证正确性或效率。

设想一个在某些分组 key（不一定等于主键）上计算总和的消费者。如果该消费者只看到一个新值 `5`，它无法确定应该向求和结果中加入什么值。例如，如果旧值是 `4`，那么它应该向结果中加 `1`；但如果旧值是 `6`，它则应该从结果中减去 `1`。对于这类消费者而言，旧值非常重要。

总而言之，`none` changelog producer 最适合诸如数据库系统这类消费者。Flink 还有一个
内置的「normalize」算子，它会在状态中持久化每个 key 的值。可以很容易看出，该算子
代价会非常高昂，应当尽量避免。（你可以通过 `'scan.remove-normalize'` 强制移除「normalize」算子。）

![](/img/changelog-producer-none.png)

## Input {#input}

通过指定 `'changelog-producer' = 'input'`，Paimon writer 会依赖其输入作为完整 changelog 的来源。所有输入记录都会被保存在独立的 changelog 文件中，并由 Paimon source 提供给消费者。

当 Paimon writer 的输入本身就是完整的 changelog（例如来自数据库 CDC，或由 Flink 有状态计算生成）时，可以使用 `input` changelog producer。

![](/img/changelog-producer-input.png)

## Lookup {#lookup}

如果你的输入无法产生完整的 changelog，但你仍然希望摆脱代价高昂的 normalize 算子，那么你
可以考虑使用 `'lookup'` changelog producer。

通过指定 `'changelog-producer' = 'lookup'`，Paimon 将在 Compaction 期间通过 `'lookup'` 生成 changelog（你也可以启用 [异步 Compaction](./compaction#asynchronous-compaction)）。默认情况下，lookup compaction 会在提交已写入数据之前执行，除非通过 `write-only` 属性将其禁用。

![](/img/changelog-producer-lookup.png)

Lookup 会将数据缓存到内存和本地磁盘上，你可以使用以下配置项来调优性能：

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
        <td><h5>lookup.cache-file-retention</h5></td>
        <td style="word-wrap: break-word;">1 h</td>
        <td>Duration</td>
        <td>Lookup 缓存文件的保留时间。文件过期后，如果有访问需求，将会重新从 DFS 读取以在本地磁盘上构建索引。</td>
    </tr>
    <tr>
        <td><h5>lookup.cache-max-disk-size</h5></td>
        <td style="word-wrap: break-word;">unlimited</td>
        <td>MemorySize</td>
        <td>Lookup 缓存的最大磁盘大小，你可以使用该配置项来限制本地磁盘的使用。</td>
    </tr>
    <tr>
        <td><h5>lookup.cache-max-memory-size</h5></td>
        <td style="word-wrap: break-word;">256 mb</td>
        <td>MemorySize</td>
        <td>Lookup 缓存的最大内存大小。</td>
    </tr>
    </tbody>
</table>

Lookup changelog-producer 支持 `changelog-producer.row-deduplicate`，以避免为同一条记录生成 -U、+U
changelog。

（注意：请增大 `'execution.checkpointing.max-concurrent-checkpoints'` Flink 配置，这对性能而言
非常重要）。

## Full Compaction {#full-compaction}

你也可以考虑使用 'full-compaction' changelog producer 来生成 changelog，它更适用于
延迟较大的场景（例如 30 分钟）。

1. 通过指定 `'changelog-producer' = 'full-compaction'`，Paimon 会比较各次全量 Compaction 之间的结果，并
将差异作为 changelog 产生。changelog 的延迟受全量 Compaction 频率的影响。
2. 通过指定 `full-compaction.delta-commits` 表属性，全量 Compaction 将在每若干次 delta
提交（checkpoint）之后被持续触发。该值默认设置为 1，因此每个 checkpoint 都会执行一次全量 Compaction 并生成一份
changelog。

一般来说，全量 Compaction 的代价和资源消耗都很高，因此我们推荐使用 `'lookup'` changelog
producer。

![](/img/changelog-producer-full-compaction.png)

:::info

Full compaction changelog producer 可以为任意类型的 source 产生完整的 changelog。但它不如
input changelog producer 高效，且产生 changelog 的延迟可能较高。

:::

Full-compaction changelog-producer 支持 `changelog-producer.row-deduplicate`，以避免为同一条记录生成 -U、+U
changelog。

## Changelog Merging {#changelog-merging}

适用于 `input`、`lookup`、`full-compaction` 这三种 'changelog-producer'。

如果 Flink 的 checkpoint 间隔很短（例如 30 秒）且桶的数量很大，那么每个快照可能会
产生大量的小 changelog 文件。文件过多可能会给分布式存储集群带来负担。

为了将小的 changelog 文件合并成大文件，你可以设置表配置项 `precommit-compact = true`。
该配置项的默认值为 false，如果设置为 true，它会在 writer 算子之后添加一个 compact coordinator 和 worker 算子，
将 changelog 文件合并成大文件。
