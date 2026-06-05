---
title: "表模式"
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

# 表模式 {#table-mode}

![](/img/lsm-inside-bucket.png)

主键表（primary key table）的文件结构大致如上图所示。表或分区包含多个桶（bucket），每个桶都是一棵独立的 LSM 树（LSM tree）结构，其中包含多个文件。

LSM 的写入过程大致如下：Flink checkpoint 将 L0 文件刷盘，并按需触发一次 Compaction 来合并数据。根据写入时不同的处理方式，存在三种模式：

1. MOR（Merge On Read，读时合并）：默认模式，只执行 minor Compaction，读取时需要进行合并。
2. COW（Copy On Write，写时复制）：使用 `'full-compaction.delta-commits' = '1'`，将同步执行全量 Compaction，也就是说合并在写入时完成。
3. MOW（Merge On Write，写时合并）：使用 `'deletion-vectors.enabled' = 'true'`，在写入阶段会查询 LSM 为数据文件生成删除向量（Deletion Vector）文件，从而在读取时直接过滤掉不需要的行。

对于一般的主键表（合并引擎默认为 `deduplicate`），推荐使用 Merge On Write 模式。

## 读时合并（Merge On Read） {#merge-on-read}

MOR 是主键表的默认模式。

![](/img/mor.png)

当模式为 MOR 时，读取需要合并所有文件，因为所有文件都是有序的，并且需要进行多路归并，其中包含对主键的比较计算。

这里存在一个明显的问题：单棵 LSM 树只能由单个线程读取，因此读取的并行度是受限的。如果桶中的数据量过大，可能导致读取性能不佳。因此，为了读取性能，建议分析查询需求表，并将桶中的数据量设置在 200MB 到 1GB 之间。但如果桶太小，会产生大量小文件的读写，对文件系统造成压力。

此外，由于存在合并过程，无法在非主键列上执行基于 Filter 的数据跳过（data skipping），否则新数据会被过滤掉，从而导致返回错误的旧数据。

- 写性能：非常好。
- 读性能：不太好。

## 写时复制（Copy On Write） {#copy-on-write}

```sql
ALTER TABLE orders SET ('full-compaction.delta-commits' = '1');
```

将 `full-compaction.delta-commits` 设置为 1，意味着每次写入都会被完整合并，所有数据都会被合并到最高层（level）。此时读取无需进行合并，读取性能最高。但每次写入都需要全量合并，写放大非常严重。

![](/img/cow.png)

- 写性能：非常差。
- 读性能：非常好。

## 写时合并（Merge On Write） {#merge-on-write}

```sql
ALTER TABLE orders SET ('deletion-vectors.enabled' = 'true');
```

得益于 Paimon 的 LSM 结构，它具备按主键查询的能力。我们可以在写入时生成删除向量文件，表示文件中的哪些数据已被删除。这样在读取时直接过滤掉不需要的行，相当于完成了合并，并且不影响读取性能。

![](/img/mow.png)

一个简单的示例如下：

![](/img/mow-example.png)

更新数据的方式是先删除旧记录，再添加新记录。

- 写性能：好。
- 读性能：好。

:::info

可见性保证：在删除向量模式下的表，level 0 的文件只有在 Compaction 之后才会变得可见。因此默认情况下，Compaction 是同步的；如果开启异步 Compaction，数据可能会存在延迟。

:::

## MOR 读优化（MOR Read Optimized） {#mor-read-optimized}

如果你不想使用删除向量模式，又希望在 MOR 模式下查询足够快，但只能查到较旧的数据，你也可以：

1. 在写入数据时配置 'compaction.optimization-interval'。
2. 从[读优化系统表](../concepts/system-tables#read-optimized-table)查询。从优化后的文件结果中读取可以避免合并具有相同 key 的记录，从而提升读取性能。

读取时你可以灵活地在查询性能和数据延迟之间进行权衡。
