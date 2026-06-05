---
title: "系统表"
sidebar_position: 6
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

# 系统表 {#system-tables}

PyPaimon 通过既有的 Catalog 和 read-builder API 暴露 `table$<name>` 系统表。支持的短名称有：
`snapshots`、`schemas`、`options`、`manifests`、`files`、`partitions`、
`tags` 和 `branches`。`sys` 数据库下的全局表
（`sys.all_tables`、`sys.catalog_options` 等）以及流式的
`audit_log` / `binlog` 系列尚未暴露。

## 基本用法 {#basic-usage}

请为扫描（scan）和读取（read）复用同一个 read builder，这样在其上设置的任何
投影（projection）或 limit 都能被双方同时遵守：

```python
from pypaimon import CatalogFactory

catalog = CatalogFactory.create({'warehouse': '/path/to/warehouse'})
snapshots = catalog.get_table('default.my_table$snapshots')

read_builder = snapshots.new_read_builder()
splits = read_builder.new_scan().plan().splits()
print(read_builder.new_read().to_pandas(splits))
```

`with_projection` 和 `with_limit` 可在同一个 builder 上链式调用：

```python
read_builder = (
    snapshots.new_read_builder()
    .with_projection(['snapshot_id', 'commit_user', 'commit_time'])
    .with_limit(10)
)
splits = read_builder.new_scan().plan().splits()
arrow_table = read_builder.new_read().to_arrow(splits)
```

返回的对象暴露了常规的 `Table` 接口，因此同一个 read builder 可与
`to_pandas`、`to_arrow`、`to_iterator`、
`to_record_batch_iterator` 以及 `to_duckdb` 配合使用。写操作会抛出
`NotImplementedError`——系统表是只读的。

## 可用的表 {#available-tables}

下面列出每张系统表及其列布局（包括是否可空）和主键选择。这些表按照它们在
`SystemTableLoader` 中出现的顺序列出。

### `$snapshots` {#snapshots}

每个已持久化的快照对应一行。

| 列                        | 类型            | 说明                           |
|---------------------------|-----------------|--------------------------------|
| `snapshot_id`             | BIGINT NOT NULL | 主键                           |
| `schema_id`               | BIGINT NOT NULL |                                |
| `commit_user`             | STRING NOT NULL |                                |
| `commit_identifier`       | BIGINT NOT NULL |                                |
| `commit_kind`             | STRING NOT NULL | `APPEND`、`COMPACT`、...        |
| `commit_time`             | TIMESTAMP(3) NOT NULL |                          |
| `base_manifest_list`      | STRING NOT NULL |                                |
| `delta_manifest_list`     | STRING NOT NULL |                                |
| `changelog_manifest_list` | STRING          |                                |
| `total_record_count`      | BIGINT          |                                |
| `delta_record_count`      | BIGINT          |                                |
| `changelog_record_count`  | BIGINT          |                                |
| `watermark`               | BIGINT          |                                |
| `next_row_id`             | BIGINT          |                                |

### `$schemas` {#schemas}

每一个已提交的 schema（表结构）版本，其中 `fields` / `partition_keys` /
`primary_keys` / `options` 被编码为紧凑的 JSON 字符串。

| 列               | 类型            | 说明         |
|------------------|-----------------|--------------|
| `schema_id`      | BIGINT NOT NULL | 主键         |
| `fields`         | STRING NOT NULL | JSON         |
| `partition_keys` | STRING NOT NULL | JSON 列表    |
| `primary_keys`   | STRING NOT NULL | JSON 列表    |
| `options`        | STRING NOT NULL | JSON map     |
| `comment`        | STRING          |              |
| `update_time`    | TIMESTAMP(3) NOT NULL |        |

### `$options` {#options}

两列，回显当前生效的表配置项。

| 列      | 类型            | 说明         |
|---------|-----------------|--------------|
| `key`   | STRING NOT NULL | 主键         |
| `value` | STRING NOT NULL |              |

### `$manifests` {#manifests}

最新快照的 manifest（清单）列表。

| 列                    | 类型            | 说明                          |
|-----------------------|-----------------|-------------------------------|
| `file_name`           | STRING NOT NULL | 主键                          |
| `file_size`           | BIGINT NOT NULL |                               |
| `num_added_files`     | BIGINT NOT NULL |                               |
| `num_deleted_files`   | BIGINT NOT NULL |                               |
| `schema_id`           | BIGINT NOT NULL |                               |
| `min_partition_stats` | STRING          | 占位符（见“局限性”）          |
| `max_partition_stats` | STRING          | 占位符（见“局限性”）          |
| `min_row_id`          | BIGINT          |                               |
| `max_row_id`          | BIGINT          |                               |

### `$files` {#files}

最新快照中每个存活的 ADD 条目对应一行。统计列是以列名为键的紧凑 JSON 字典。
线格式名称 `deleteRowCount` 故意采用驼峰命名（camelCase）。

| 列                      | 类型                | 说明                        |
|-------------------------|---------------------|-----------------------------|
| `partition`             | STRING              | `pt=v/pt2=v2`               |
| `bucket`                | INT NOT NULL        |                             |
| `file_path`             | STRING NOT NULL     | 主键                        |
| `file_format`           | STRING NOT NULL     |                             |
| `schema_id`             | BIGINT NOT NULL     |                             |
| `level`                 | INT NOT NULL        |                             |
| `record_count`          | BIGINT NOT NULL     |                             |
| `file_size_in_bytes`    | BIGINT NOT NULL     |                             |
| `min_key`               | STRING              | JSON 列表（仅主键表）       |
| `max_key`               | STRING              | JSON 列表（仅主键表）       |
| `null_value_counts`     | STRING NOT NULL     | JSON map                    |
| `min_value_stats`       | STRING NOT NULL     | JSON map                    |
| `max_value_stats`       | STRING NOT NULL     | JSON map                    |
| `min_sequence_number`   | BIGINT              |                             |
| `max_sequence_number`   | BIGINT              |                             |
| `creation_time`         | TIMESTAMP(3)        |                             |
| `deleteRowCount`        | BIGINT              | 驼峰命名的线格式名称        |
| `file_source`           | STRING              |                             |
| `first_row_id`          | BIGINT              |                             |
| `write_cols`            | ARRAY<STRING>       |                             |

### `$partitions` {#partitions}

最新快照的聚合分区统计信息。

| 列                    | 类型                  | 说明                           |
|-----------------------|-----------------------|--------------------------------|
| `partition`           | STRING                | `pt=v/pt2=v2`；主键            |
| `record_count`        | BIGINT NOT NULL       |                                |
| `file_size_in_bytes`  | BIGINT NOT NULL       |                                |
| `file_count`          | BIGINT NOT NULL       |                                |
| `last_update_time`    | TIMESTAMP(3)          |                                |
| `created_at`          | TIMESTAMP(3)          | 文件系统路径返回 `NULL`        |
| `created_by`          | STRING                | 文件系统路径返回 `NULL`        |
| `updated_by`          | STRING                | 文件系统路径返回 `NULL`        |
| `options`             | STRING                | 文件系统路径返回 `NULL`        |
| `total_buckets`       | INT NOT NULL          |                                |
| `done`                | BOOLEAN NOT NULL      | 文件系统路径返回 `False`       |

### `$tags` {#tags}

每个标签（Tag）的快照元数据。

| 列              | 类型                  | 说明                           |
|-----------------|-----------------------|--------------------------------|
| `tag_name`      | STRING NOT NULL       | 主键                           |
| `snapshot_id`   | BIGINT NOT NULL       |                                |
| `schema_id`     | BIGINT NOT NULL       |                                |
| `commit_time`   | TIMESTAMP(3) NOT NULL |                                |
| `record_count`  | BIGINT                |                                |
| `create_time`   | TIMESTAMP(3)          | 当前输出为 `NULL`              |
| `time_retained` | STRING                | 当前输出为 `NULL`              |

### `$branches` {#branches}

每个命名分支（Branch），附带该分支目录的修改时间。

| 列            | 类型                  | 说明         |
|---------------|-----------------------|--------------|
| `branch_name` | STRING NOT NULL       | 主键         |
| `create_time` | TIMESTAMP(3) NOT NULL |              |

## 局限性 {#limitations}

* **谓词下推尚未实现。** 调用
  `with_filter(...)` 会被接受，但之后调用 `new_read()` 时会
  抛出 `NotImplementedError`，而不是默默地丢弃该
  谓词。请改为在客户端侧对结果 Arrow 表 / DataFrame 进行过滤。
* **`$manifests` 中的 `min_partition_stats` / `max_partition_stats`**
  会输出为 `NULL`。PyPaimon 尚未提供将分区行转换为其字符串形式的
  辅助方法。
* **`tag.time_retained` 和 `tag.create_time` 为 `NULL`。** PyPaimon 的
  `Tag` dataclass 尚未携带这些字段——这与
  `FileSystemCatalog.get_tag` 当前的行为一致。
* **`branch.create_time` 在底层
  存储无法提供 mtime 时回退为纪元 0（epoch 0）**（某些通过
  `PyArrowFileIO` 访问的远程对象存储）。本地文件系统 Catalog 始终会填充
  真实时间。
* **`partitions.created_at / created_by / updated_by / options / done`**
  对于文件系统路径会以占位符填充。暴露这些字段的 REST 托管
  Catalog 将在后续工作中接入。
* **`list_tables` 不会枚举系统表。** 系统表
  仍可通过 `get_table('db.t$name')` 访问。

## 通过 Catalog 支持的情况 {#supported-via-catalogs}

* `FilesystemCatalog`——完全支持。
* `RESTCatalog`——完全支持；依赖于 Catalog
  元数据的列（例如 `$partitions.created_by`）会在服务器暴露它们时
  通过 REST API 填充。
