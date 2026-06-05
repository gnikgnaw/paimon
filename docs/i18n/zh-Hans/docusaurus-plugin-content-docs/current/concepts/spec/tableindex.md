---
title: "表索引"
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

# 表索引 {#table-index}

表索引（Table Index）文件位于 `index` 目录中。

## 动态分桶索引 {#dynamic-bucket-index}

动态分桶索引用于存储主键（primary-key）的哈希值与桶（bucket）之间的对应关系。

它的结构非常简单，文件中仅存储哈希值：

HASH_VALUE | HASH_VALUE | HASH_VALUE | HASH_VALUE | ...

HASH_VALUE 是主键的哈希值。4 字节，BIG_ENDIAN。

## 删除向量 {#deletion-vectors}

删除文件用于存储每个数据文件中被删除记录的位置。对于主键表（primary key table），每个桶有一个删除文件。

![](/img/deletion-file.png)

删除文件是一个二进制文件，其格式如下：

- 首先，用一个字节记录版本号。当前版本为 1。
- 然后，依次记录 <序列化 bin 的大小, 序列化 bin, 序列化 bin 的校验和>。
- 大小（Size）和校验和（checksum）均为 BIG_ENDIAN 整数。

对于每个序列化 bin，其序列化格式由 `deletion-vectors.bitmap64` 决定。
Paimon 默认使用 32 位位图（bitmap）来存储被删除的记录，但如果将 `deletion-vectors.bitmap64` 设置为 true，则会使用 64 位位图。
两种位图的序列化方式不同。注意，只有 64 位位图实现才与 Iceberg 兼容。

32 位位图的序列化 bin：（默认）
- 首先，用一个 int（BIG_ENDIAN）记录一个常量魔数（magic number）。当前魔数为 1581511376。
- 然后，记录一个 32 位的序列化位图。它是一个 [RoaringBitmap](https://github.com/RoaringBitmap/RoaringBitmap)（org.roaringbitmap.RoaringBitmap）。

64 位位图的序列化 bin：
- 首先，用一个 int（LITTLE_ENDIAN）记录一个常量魔数。当前魔数为 1681511377。
- 然后，记录一个 64 位的序列化位图。它支持正的 64 位位置（最高有效位必须为 0），
  但通过使用一个由 32 位 Roaring 位图组成的数组，对大多数位置可容纳于 32 位的情况进行了优化。内部位图数组会按需增长，以容纳最大的位置。
  64 位位图的序列化方式如下：
  - 首先，用一个 long（LITTLE_ENDIAN）记录位图数组的大小。
  - 然后，依次记录每个位图在数组中的索引（用一个 int，LITTLE_ENDIAN）以及该位图的序列化字节。
