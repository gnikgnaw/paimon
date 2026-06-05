---
title: "基本概念"
sidebar_position: 2
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

# 基本概念 {#basic-concepts}

## 文件布局 {#file-layouts}

一张表的所有文件都存储在一个基础目录下。Paimon 的文件以分层的方式组织。下图展示了文件布局。从一个快照文件开始，Paimon 的读取器可以递归地访问表中的所有记录。

![](/img/file-layout.png)

## 快照 {#snapshot}

所有快照文件都存储在 `snapshot` 目录下。

快照文件是一个 JSON 文件，包含关于该快照的信息，包括

* 正在使用的 schema 文件
* 包含该快照所有变更的 manifest 列表

快照（snapshot）捕获了某个时间点上一张表的状态。用户可以通过最新的快照访问表的最新数据。通过时间旅行，用户还可以通过更早的快照访问表先前的状态。

## Manifest 文件 {#manifest-files}

所有 manifest 列表和 manifest 文件都存储在 `manifest` 目录下。

manifest 列表是一个 manifest 文件名的列表。

manifest（清单）文件是一个包含关于 LSM 数据文件和 changelog 文件变更的文件。例如，在对应的快照中创建了哪个 LSM 数据文件、删除了哪个文件。

## 数据文件 {#data-files}

数据文件按分区分组。目前，Paimon 支持使用 parquet（默认）、orc 和 avro 作为数据文件的格式。

## 分区 {#partition}

Paimon 采用与 Apache Hive 相同的分区概念来分隔数据。

分区是一种可选的方式，根据特定列（如日期、城市和部门）的值将一张表划分为相关的部分。每张表可以拥有一个或多个分区键来标识某个特定的分区。

通过分区，用户可以高效地操作表中某一片记录。

## 一致性保证 {#consistency-guarantees}

Paimon 的写入器使用两阶段提交协议来原子地将一批记录提交到表中。每次提交在提交时最多产生两个[快照](./basic-concepts#snapshot)。这取决于增量写入和 Compaction 策略。如果只执行增量写入而没有触发 Compaction 操作，则只会创建一个增量快照。如果触发了 Compaction 操作，则会创建一个增量快照和一个经过 Compaction 的快照。

对于同时修改一张表的任意两个写入器，只要它们不修改同一个分区，它们的提交就可以并行发生。如果它们修改同一个分区，则只保证快照隔离。也就是说，最终的表状态可能是两次提交的混合，但不会丢失任何变更。
更多信息请参见[独立 Compaction 作业](../maintenance/dedicated-compaction#dedicated-compaction-job)。
