---
title: "Compaction"
sidebar_position: 7
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

# Compaction {#compaction}

当越来越多的记录写入 LSM 树（LSM tree）时，sorted run 的数量会不断增加。由于查询 LSM 树需要合并所有的 sorted run，过多的 sorted run 会导致较差的查询性能，甚至引发内存溢出（out of memory）。

为了限制 sorted run 的数量，我们需要时不时地把若干个 sorted run 合并成一个大的 sorted run。这一过程称为 Compaction。

然而，Compaction 是一个资源密集型的过程，会消耗一定的 CPU 时间和磁盘 IO，因此过于频繁的 Compaction 反而可能导致写入变慢。这是查询性能与写入性能之间的权衡。Paimon 目前采用了一种类似于 Rocksdb [universal compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction) 的 Compaction 策略。

Compaction 解决以下问题：

1. 减少 Level 0 文件，避免较差的查询性能。
2. 通过 [changelog-producer](./changelog-producer) 产生 changelog（变更日志）。
3. 为 [MOW 模式](./table-mode#merge-on-write) 产生删除向量（Deletion Vector）。
4. 快照过期（Snapshot Expiration）、标签过期（Tag Expiration）、分区过期（Partitions Expiration）。

限制：

- 对于同一分区的 Compaction，只能有一个作业在运行，否则会产生冲突，并导致其中一方抛出异常失败。

写入性能几乎总是会受到 Compaction 的影响，因此对其进行调优至关重要。

## Asynchronous Compaction {#asynchronous-compaction}

Compaction 本质上是异步的，但如果你希望它完全异步、不阻塞写入，以期望获得最大写入吞吐量的模式，那么 Compaction 可以缓慢、不慌不忙地进行。
你可以为你的表使用以下策略：

```shell
num-sorted-run.stop-trigger = 2147483647
sort-spill-threshold = 10
lookup-wait = false
```

该配置会在写入高峰期生成更多文件，并在写入低谷期逐渐将它们合并，以获得最佳的读取性能。

## Dedicated compaction job {#dedicated-compaction-job}

通常情况下，如果你期望多个作业写入同一张表，你需要将 Compaction 分离出来。你可以使用 [独立 Compaction 作业](../maintenance/dedicated-compaction#dedicated-compaction-job)。

## Record-Level expire {#record-level-expire}

在 Compaction 中，你可以配置记录级（record-Level）过期时间来使记录过期，你需要配置：

1. `'record-level.expire-time'`：记录的保留时间。
2. `'record-level.time-field'`：记录级过期使用的时间字段。

过期发生在 Compaction 中，无法强保证记录会及时过期。
你可以手动触发一次全量 Compaction，以使那些未及时过期的记录过期。

## Full Compaction {#full-compaction}

Paimon 的 Compaction 使用 [Universal-Compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)。
默认情况下，当增量数据过多时，会自动执行全量 Compaction。通常你无需为此担心。

Paimon 还提供了一项配置，允许定期执行全量 Compaction。

1. 'compaction.optimization-interval'：表示执行一次优化型全量 Compaction 的频率，该配置用于确保读优化（read-optimized）系统表的查询时效性。
2. 'full-compaction.delta-commits'：在若干次 delta 提交后会持续触发全量 Compaction。它的缺点是只能同步执行 Compaction，从而会影响写入效率。

## Lookup Compaction {#lookup-compaction}

当主键表配置了 `lookup` [changelog producer](./changelog-producer)，
或 `first-row` [合并引擎](./merge-engine/)，
或为 [MOW 模式](./table-mode#merge-on-write) 启用了 `deletion vectors` 时，Paimon 将
使用一种激进的 Compaction 策略，在每次触发 Compaction 时强制将 level 0 文件 Compaction 到更高的层（level）。

Paimon 还提供了一些配置来优化这种 Compaction 的频率。

1. 'lookup-compact'：用于 Lookup Compaction 的 Compaction 模式。可选值：`radical`，将使用
   `ForceUpLevel0Compaction` 策略激进地 Compaction 新文件；`gentle`，将使用 `UniversalCompaction` 策略
   温和地 Compaction 新文件；
2. 'lookup-compact.max-interval'：在 `gentle` 模式下触发一次强制 L0 Lookup Compaction 的最大间隔。
   该配置项仅在 `lookup-compact` 模式为 `gentle` 时有效。

通过将 'lookup-compact' 配置为 `gentle`，L0 中的新文件不会立即被 Compaction，这可能在某些情况下以较差的数据新鲜度为代价，大幅降低整体资源消耗。

## Compaction Options {#compaction-options}

### Number of Sorted Runs to Pause Writing {#number-of-sorted-runs-to-pause-writing}

当 sorted run 的数量较少时，Paimon 写入器会在独立的线程中异步执行 Compaction，因此
记录可以持续不断地写入表中。然而，为了避免 sorted run 无限增长，当 sorted run 的数量达到阈值时，写入器将
暂停写入。下面的表属性决定了该阈值。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 20%">Option</th>
      <th class="text-left" style="width: 5%">Required</th>
      <th class="text-left" style="width: 5%">Default</th>
      <th class="text-left" style="width: 10%">Type</th>
      <th class="text-left" style="width: 60%">Description</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>num-sorted-run.stop-trigger</h5></td>
      <td>否</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>Integer</td>
      <td>触发停止写入的 sorted run 数量，默认值为 'num-sorted-run.compaction-trigger' + 3。</td>
    </tr>
    </tbody>
</table>

当 `num-sorted-run.stop-trigger` 变大时，写入停顿（write stall）会变得不那么频繁，从而提升写入
性能。然而，如果该值过大，查询表时将需要更多的内存和 CPU 时间。如果你担心 OOM 问题，请配置下面的配置项。
它的取值取决于你的内存大小。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 20%">Option</th>
      <th class="text-left" style="width: 5%">Required</th>
      <th class="text-left" style="width: 5%">Default</th>
      <th class="text-left" style="width: 10%">Type</th>
      <th class="text-left" style="width: 60%">Description</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>sort-spill-threshold</h5></td>
      <td>否</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>Integer</td>
      <td>如果排序读取器（sort reader）的最大数量超过该值，将尝试进行溢写（spill）。这可以防止过多的读取器消耗过多内存并导致 OOM。</td>
    </tr>
    </tbody>
</table>

### Number of Sorted Runs to Trigger Compaction {#number-of-sorted-runs-to-trigger-compaction}

Paimon 使用 [LSM 树](./overview#lsm-trees)，它支持大量的更新。LSM 将文件组织在若干个 [sorted run](./overview#sorted-runs) 中。当从 LSM 树中查询记录时，必须合并所有的 sorted run 才能产生所有记录的完整视图。

可以很容易地看出，过多的 sorted run 会导致较差的查询性能。为了将 sorted run 的数量保持在合理范围内，Paimon 写入器会自动执行 [Compaction](./compaction)。下面的表属性决定了触发一次 Compaction 所需的最小 sorted run 数量。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 20%">Option</th>
      <th class="text-left" style="width: 5%">Required</th>
      <th class="text-left" style="width: 5%">Default</th>
      <th class="text-left" style="width: 10%">Type</th>
      <th class="text-left" style="width: 60%">Description</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>num-sorted-run.compaction-trigger</h5></td>
      <td>否</td>
      <td style="word-wrap: break-word;">5</td>
      <td>Integer</td>
      <td>触发 Compaction 的 sorted run 数量。包括 level0 文件（一个文件即一个 sorted run）和高层 run（一层即一个 sorted run）。</td>
    </tr>
    </tbody>
</table>

当 `num-sorted-run.compaction-trigger` 变大时，Compaction 会变得不那么频繁，从而提升写入性能。然而，如果该值过大，查询表时将需要更多的内存和 CPU 时间。这是写入性能与查询性能之间的权衡。
