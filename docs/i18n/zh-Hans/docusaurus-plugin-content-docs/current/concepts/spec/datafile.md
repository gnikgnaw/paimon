---
title: "DataFile"
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

# DataFile {#datafile}

## 分区（Partition） {#partition}

考虑通过 Flink SQL 创建一张分区表：

```sql
CREATE TABLE part_t (
    f0 INT,
    f1 STRING,
    dt STRING
) PARTITIONED BY (dt);

INSERT INTO part_t VALUES (1, '11', '20240514');
```

此时文件系统结构如下：

```shell
part_t
├── dt=20240514
│   └── bucket-0
│       └── data-ca1c3c38-dc8d-4533-949b-82e195b41bd4-0.orc
├── manifest
│   ├── manifest-08995fe5-c2ac-4f54-9a5f-d3af1fcde41d-0
│   ├── manifest-list-51c16f7b-421c-4bc0-80a0-17677f343358-0
│   └── manifest-list-51c16f7b-421c-4bc0-80a0-17677f343358-1
├── schema
│   └── schema-0
└── snapshot
    ├── EARLIEST
    ├── LATEST
    └── snapshot-1
```

Paimon 采用与 Apache Hive 相同的分区概念来分离数据。分区的文件将被放置在一个单独的分区目录中。

## 桶（Bucket） {#bucket}

所有 Paimon 表的存储都依赖于桶，数据文件存储在桶目录中。Paimon 中各类表类型与桶的关系如下：

1. 主键表（Primary Key Table）：
   1. bucket = -1：默认模式，即动态分桶模式，通过索引文件记录键对应到哪个桶。索引记录了主键的哈希值与桶之间的对应关系。
   2. bucket = 10：数据根据桶键（bucket key，默认为主键）的哈希值分布到对应的桶中。
2. Append 表（追加表）：
   1. bucket = -1：默认模式，忽略桶的概念，虽然所有数据都写入 bucket-0，但读和写的并行度不受限制。
   2. bucket = 10：你还需要定义桶键（bucket-key），数据根据桶键的哈希值分布到对应的桶中。

## 数据文件（Data File） {#data-file}

数据文件的名称为 `data-${uuid}-${id}.${format}`。对于 Append 表，文件存储表的数据，不会添加任何新列。但对于主键表，每行数据都会存储额外的系统列：

### 含主键的数据文件（Table with Primary key Data File） {#table-with-primary-key-data-file}

1. 主键列，以 `_KEY_` 前缀作为键列的前缀，这是为了避免与表的列发生冲突。它是可选的，Paimon 1.0 及以上版本将从 value_columns 中获取主键字段。
2. `_VALUE_KIND`：TINYINT，表示该行是被删除还是被添加。与 RocksDB 类似，每行数据都可以被删除或添加，这将用于更新主键表。
3. `_SEQUENCE_NUMBER`：BIGINT，该数字用于更新时的比较，确定哪条数据在前、哪条数据在后。
4. 值列（Value columns）。表中声明的所有列。

例如，下面这张表的数据文件：

```sql
CREATE TABLE T (
    a INT PRIMARY KEY NOT ENFORCED,
    b INT,
    c INT
);
```

它的文件有 6 列：`_KEY_a`、`_VALUE_KIND`、`_SEQUENCE_NUMBER`、`a`、`b`、`c`。

当启用 `data-file.thin-mode` 时，它的文件有 5 列：`_VALUE_KIND`、`_SEQUENCE_NUMBER`、`a`、`b`、`c`。

### 不含主键的数据文件（Table w/o Primary key Data File） {#table-wo-primary-key-data-file}

- 值列（Value columns）。表中声明的所有列。

例如，下面这张表的数据文件：

```sql
CREATE TABLE T (
    a INT,
    b INT,
    c INT
);
```

它的文件有 3 列：`a`、`b`、`c`。

## Changelog 文件（Changelog File） {#changelog-file}

changelog（变更日志）文件与数据文件完全相同，它仅在主键表上生效。它类似于数据库中的 Binlog，记录表中数据的变更。
