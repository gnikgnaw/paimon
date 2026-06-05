---
title: "Manifest"
sidebar_position: 4
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

# Manifest {#manifest}

## Manifest 列表 {#manifest-list}

```shell
├── manifest
    └── manifest-list-51c16f7b-421c-4bc0-80a0-17677f343358-1
```

manifest 列表（manifest list）包含若干 manifest（清单）文件的元信息。它的名称中包含 UUID，是一个 avro 文件，其 schema（表结构）为：

1. _FILE_NAME: STRING，manifest 文件名。
2. _FILE_SIZE: BIGINT，manifest 文件大小。
3. _NUM_ADDED_FILES: BIGINT，manifest 中新增的文件数量。
4. _NUM_DELETED_FILES: BIGINT，manifest 中删除的文件数量。
5. _PARTITION_STATS: SimpleStats，分区统计信息，该 manifest 中分区字段的最小值与最大值，有助于在查询时跳过某些 manifest 文件，它是一个 SimpleStats。
6. _SCHEMA_ID: BIGINT，写入该 manifest 文件时的 schema id。

## Manifest {#manifest-1}

manifest 包含若干数据文件、changelog（变更日志）文件或表索引（table-index）文件的元信息。它的名称中包含 UUID，是一个 avro 文件。

文件的变更被保存在 manifest 中，文件可以被新增或删除。manifest 应当保持有序，同一个文件可能被多次新增或删除，应当读取最后一个版本。这样的设计可以让提交更加轻量，从而支持由 Compaction 产生的文件删除。

### 数据 Manifest {#data-manifest}

数据 manifest 包含若干数据文件或 changelog 文件的元信息。

```shell
├── manifest
    └── manifest-6758823b-2010-4d06-aef0-3b1b597723d6-0
```

其 schema 为：

1. _KIND: TINYINT，ADD 或 DELETE，
2. _PARTITION: BYTES，分区规格（partition spec），一个 BinaryRow。
3. _BUCKET: INT，该文件所属的桶。
4. _TOTAL_BUCKETS: INT，写入该文件时的桶总数，用于桶数变更后的校验。
5. _FILE: 数据文件元信息。

数据文件元信息为：

1. _FILE_NAME: STRING，文件名。
2. _FILE_SIZE: BIGINT，文件大小。
3. _ROW_COUNT: BIGINT，该文件中的总行数（包括新增与删除）。
4. _MIN_KEY: STRING，该文件的最小键。
5. _MAX_KEY: STRING，该文件的最大键。
6. _KEY_STATS: SimpleStats，键的统计信息。
7. _VALUE_STATS: SimpleStats，值的统计信息。
8. _MIN_SEQUENCE_NUMBER: BIGINT，最小序列号。
9. _MAX_SEQUENCE_NUMBER: BIGINT，最大序列号。
10. _SCHEMA_ID: BIGINT，写入该文件时的 schema id。
11. _LEVEL: INT，该文件在 LSM 中所处的层（level）。
12. _EXTRA_FILES: ARRAY<STRING>，该文件的额外文件，例如数据文件索引文件。
13. _CREATION_TIME: TIMESTAMP_MILLIS，该文件的创建时间。
14. _DELETE_ROW_COUNT: BIGINT，rowCount = addRowCount + deleteRowCount。
15. _EMBEDDED_FILE_INDEX: BYTES，如果数据文件索引过小，则将该索引存储在 manifest 中。
16. _FILE_SOURCE: TINYINT，指示该文件是作为 APPEND 文件还是 COMPACT 文件生成的。
17. _VALUE_STATS_COLS: ARRAY<STRING>，元数据中的统计列。
18. _EXTERNAL_PATH: 该文件的外部路径，如果位于 warehouse（仓库目录）中则为 null。

### 索引 Manifest {#index-manifest}

索引 manifest 包含若干[表索引（table-index）](./tableindex)文件的元信息。

```shell
├── manifest
    └── index-manifest-5d670043-da25-4265-9a26-e31affc98039-0
```

其 schema 为：

1. _KIND: TINYINT，ADD 或 DELETE，
2. _PARTITION: BYTES，分区规格（partition spec），一个 BinaryRow。
3. _BUCKET: INT，该文件所属的桶。
4. _INDEX_TYPE: STRING，"HASH" 或 "DELETION_VECTORS"。
5. _FILE_NAME: STRING，文件名。
6. _FILE_SIZE: BIGINT，文件大小。
7. _ROW_COUNT: BIGINT，总行数。
8. _DELETIONS_VECTORS_RANGES: 仅供 "DELETION_VECTORS" 使用的元数据，是一个删除向量（Deletion Vector）元信息的数组，每个删除向量元信息的 schema 为：
   1. f0: 该删除向量对应的数据文件名。
   2. f1: 该删除向量在索引文件中的起始偏移量。
   3. f2: 该删除向量在索引文件中的长度。
   4. _CARDINALITY: 被删除的行数。

## 附录 {#appendix}

### SimpleStats {#simplestats}

SimpleStats 是一个嵌套行（nested row），其 schema 为：

1. _MIN_VALUES: BYTES，BinaryRow，各列的最小值。
2. _MAX_VALUES: BYTES，BinaryRow，各列的最大值。
3. _NULL_COUNTS: ARRAY<BIGINT>，各列的 null 数量。

### BinaryRow {#binaryrow}

BinaryRow 由字节（bytes）而非 Object 支撑。它可以显著减少 Java 对象的序列化/反序列化开销。

一个 Row 包含两个部分：定长部分（Fixed-length part）和变长部分（variable-length part）。定长部分包含 1 字节的头部、null 位集（null bit set）以及字段值。null 位集用于 null 追踪，并按 8 字节字边界对齐。`Field values` 保存定长的基本类型，以及可以存储在 8 字节以内的变长值。如果某个变长字段无法放入其中，则存储变长部分的长度和偏移量。
