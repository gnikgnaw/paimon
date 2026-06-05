---
title: "概述"
sidebar_position: 1
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

# 概述 {#overview}

如果你定义了带主键的表，就可以在表中插入、更新或删除记录。

主键由一组列构成，这些列对每条记录都包含唯一值。Paimon 通过在每个桶（bucket）内对主键进行排序来强制数据有序，从而让用户能够通过在主键上施加过滤条件来获得高性能。参见 [CREATE TABLE](../flink/sql-ddl#create-table)。

## 桶（Bucket） {#bucket}

非分区表，或分区表中的分区，会被进一步细分为桶（bucket），以便为数据提供额外的结构，从而支持更高效的查询。

每个桶目录都包含一棵 LSM 树及其 [changelog 文件](./changelog-producer)。

桶的范围由记录中一个或多个列的哈希值决定。用户可以通过提供 [`bucket-key` 配置项](../maintenance/configurations#coreoptions) 来指定分桶列。如果未指定 `bucket-key` 配置项，则会使用主键（如果已定义）或完整记录作为桶键（bucket key）。

桶是读写的最小存储单元，因此桶的数量限制了最大处理并行度。不过，这个数量也不应过大，否则会产生大量小文件并降低读性能。一般来说，建议每个桶中的数据大小约为 200MB - 1GB。

此外，如果你想在表创建之后调整桶数（rescale bucket），请参见 [调整桶数](../maintenance/rescale-bucket)。

## LSM 树 {#lsm-trees}

Paimon 采用 LSM 树（log-structured merge-tree，日志结构合并树）作为文件存储的数据结构。本文档简要介绍 LSM 树的相关概念。

### Sorted Run {#sorted-runs}

LSM 树将文件组织成若干个 sorted run。一个 sorted run 由一个或多个数据文件组成，且每个数据文件恰好属于一个 sorted run。

数据文件内的记录按其主键排序。在同一个 sorted run 内，各数据文件的主键范围互不重叠。

![](/img/sorted-runs.png)

如你所见，不同的 sorted run 可能具有重叠的主键范围，甚至可能包含相同的主键。在查询 LSM 树时，必须合并所有 sorted run，并根据用户指定的 [合并引擎](./merge-engine/overview) 以及每条记录的时间戳，对所有具有相同主键的记录进行合并。

写入 LSM 树的新记录会首先在内存中缓冲。当内存缓冲区满时，内存中的所有记录会被排序并刷写到磁盘。此时便创建了一个新的 sorted run。
