---
title: "并发控制"
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

# 并发控制 {#concurrency-control}

Paimon 为多个并发写入作业提供乐观并发（optimistic concurrency）支持。

每个作业都按自己的节奏写入数据，并在提交时基于当前快照应用增量文件（删除或新增文件），从而生成一个新的快照（snapshot）。

这里可能存在两类提交失败：
1. 快照冲突：快照 id 已被抢占，该表已经由另一个作业生成了新的快照。没关系，我们再提交一次即可。
2. 文件冲突：本作业想要删除的文件已经被其他作业删除了。此时该作业只能失败。（对于流式作业，它会失败并重启，相当于有意地进行一次 failover）

## 快照冲突 {#snapshot-conflict}

Paimon 的快照 ID 是唯一的，因此只要作业把它的快照文件写入到文件系统，就视为提交成功。

![](/img/snapshot-conflict.png)

Paimon 使用文件系统的重命名机制来提交快照，对于 HDFS 而言这是安全的，因为它能保证重命名的事务性和原子性。

但对于 OSS、S3 这类对象存储，它们的 `'RENAME'` 不具备原子语义。我们需要配置 Hive 或 jdbc metastore，并为 Catalog 启用 `'lock.enabled'` 配置项。否则，可能会出现丢失快照的风险。

## 文件冲突 {#files-conflict}

当 Paimon 提交一次文件删除（这只是逻辑删除）时，它会检查与最新快照之间是否存在冲突。如果存在冲突（即意味着该文件已经被逻辑删除），它便无法在当前提交节点上继续下去，因此只能有意地触发一次 failover 来重启，作业将从文件系统中获取最新状态，期望以此解决该冲突。

![](/img/files-conflict.png)

Paimon 会确保这里不会发生数据丢失或重复，但如果有两个流式作业同时写入并产生冲突，你会看到它们不断地重启，这并不是一件好事。

冲突的本质在于（逻辑上）删除文件，而删除文件源自 Compaction，因此只要我们关闭写入作业的 Compaction（将 'write-only' 设为 true），并启动一个单独的作业来执行 Compaction 工作，一切就都很好了。

更多信息请参见 [独立 Compaction 作业](../maintenance/dedicated-compaction#dedicated-compaction-job)。
