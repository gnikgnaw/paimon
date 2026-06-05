---
title: "Ray Data"
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

# Ray Data {#ray-data}

需要先安装 `ray`。

`pypaimon.ray` 暴露了一个顶层的 `read_paimon` / `write_paimon` 门面，它直接接收表标识符和 Catalog 配置项，与 Ray 内置的 Iceberg 集成形状保持一致。对于已经解析出 `(read_builder, splits)` 对，或已通过常规 pypaimon API 构造出 `table_write` 的调用方，更底层的 `TableRead.to_ray()` 与 `TableWrite.write_ray()` 入口点仍然可用。

## 读 {#read}

### `read_paimon`（推荐） {#read_paimon-recommended}

```python
from pypaimon.ray import read_paimon

ray_dataset = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
)

print(ray_dataset)
# MaterializedDataset(num_blocks=1, num_rows=9, schema={f0: int32, f1: string})

print(ray_dataset.take(3))
# [{'f0': 1, 'f1': 'a'}, {'f0': 2, 'f1': 'b'}, {'f0': 3, 'f1': 'c'}]

print(ray_dataset.to_pandas())
#    f0 f1
# 0   1  a
# 1   2  b
# 2   3  c
# 3   4  d
# ...
```

`read_paimon` 会打开自己的 Catalog 并解析表，因此它等价于把 `CatalogFactory.create → get_table → new_read_builder → to_ray` 这四步样板代码合并为一次调用。

**投影与限制：**

```python
ray_dataset = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    projection=["id", "score"],
    limit=1000,
)
```

**分布 / 调度：**

```python
ray_dataset = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    override_num_blocks=4,
    ray_remote_args={"num_cpus": 2, "max_retries": 3},
    concurrency=8,
)
```

**时间旅行：**

```python
# Read a specific snapshot.
ray_dataset = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    snapshot_id=42,
)

# Read a tagged snapshot.
ray_dataset = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    tag_name="release-2026-04",
)
```

`snapshot_id` 和 `tag_name` 互斥。

**参数：**
- `table_identifier`：完整表名，例如 `"db_name.table_name"`。
- `catalog_options`：转发给 `CatalogFactory.create()` 的关键字参数，
  例如 `{"warehouse": "/path/to/warehouse"}`。
- `filter`：可选的 `Predicate`，用于下推到扫描中。
- `projection`：可选的待读取列名列表。
- `limit`：可选的行数限制，在扫描计划阶段应用。
- `snapshot_id`：可选的快照 id，用于时间旅行。与
  `tag_name` 互斥。
- `tag_name`：可选的标签（Tag）名，用于时间旅行。与
  `snapshot_id` 互斥。
- `override_num_blocks`：可选，用于覆盖输出 block 的数量。
  必须 `>= 1`。
- `ray_remote_args`：可选的关键字参数，在读任务中传递给 `ray.remote()`
  （例如 `{"num_cpus": 2, "max_retries": 3}`）。
- `concurrency`：可选，并发运行的 Ray 任务的最大数量。
- `**read_args`：转发给 `ray.data.read_datasource` 的额外关键字参数
  （例如 Ray 2.52.0+ 中的 `per_task_row_limit`）。

### `TableRead.to_ray()`（更底层） {#tablereadto_ray-lower-level}

如果你已经有了 `read_builder` 和 `splits`，可以直接将它们转换为
Ray Dataset：

```python
table_read = read_builder.new_read()
splits = read_builder.new_scan().plan().splits()
ray_dataset = table_read.to_ray(
    splits,
    override_num_blocks=4,
    ray_remote_args={"num_cpus": 2, "max_retries": 3},
)
```

`to_ray()` 接受与 `read_paimon` 相同的 `override_num_blocks`、`ray_remote_args`、
`concurrency` 和 `**read_args` 参数。

### Ray Block 大小配置 {#ray-block-size-configuration}

如果你需要配置 Ray 的 block 大小（例如，当 Paimon split 超过
Ray 默认的 128MB block 大小时），请在调用
`read_paimon` 或 `to_ray` 之前在 `DataContext` 上进行设置：

```python
from ray.data import DataContext

ctx = DataContext.get_current()
ctx.target_max_block_size = 256 * 1024 * 1024  # 256MB (default is 128MB)
```

更多细节请参阅 [Ray Data API 文档](https://docs.ray.io/en/latest/data/api/doc/ray.data.read_datasource.html)。

## 写 {#write}

### `write_paimon`（推荐） {#write_paimon-recommended}

```python
import ray
from pypaimon.ray import write_paimon

ray_dataset = ray.data.read_json("/path/to/data.jsonl")

write_paimon(
    ray_dataset,
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
)
```

`write_paimon` 会打开自己的 Catalog、解析表，并通过 Ray 的 Datasink API 提交
写入——无需单独运行 `prepare_commit` 或
`close` 步骤。

**覆盖（overwrite）模式：**

```python
write_paimon(
    ray_dataset,
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    overwrite=True,
)
```

**分布 / 调度：**

```python
write_paimon(
    ray_dataset,
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    concurrency=4,
    ray_remote_args={"num_cpus": 2},
)
```

**针对 HASH_FIXED 表的自动（分区，桶）聚簇：**

对于 HASH_FIXED 表，`write_paimon` 会在写入前自动按
`(partition_keys..., bucket)` 对行进行聚簇，使得每个（分区，
桶）落入单个 Ray 任务中——一个写入器、一个文件组。这
避免了 Ray 默认的轮询（round-robin）
分布原本会产生的小文件风暴（生成 `partitions × buckets ×
ray_tasks` 个文件，而不是 `partitions × buckets` 个）。

桶分配使用的是与写入器相同的哈希例程，因此 groupby 看到的
桶与写入器
计算出的桶在字节上是等价的。无需用户进行任何配置。对于非 HASH_FIXED
表，数据集会按原样写入。

**参数：**
- `dataset`：要写入的 Ray Dataset。
- `table_identifier`：完整表名，例如 `"db_name.table_name"`。
- `catalog_options`：转发给 `CatalogFactory.create()` 的关键字参数。
- `overwrite`：若为 `True`，则覆盖表中已有的数据。
- `concurrency`：可选，并发运行的 Ray 写任务的最大数量。
- `ray_remote_args`：可选的关键字参数，在写任务中传递给 `ray.remote()`
  （例如 `{"num_cpus": 2}`）。

### `TableWrite.write_ray()`（更底层） {#tablewritewrite_ray-lower-level}

如果你已经从写入构建器构造出了 `table_write`，可以
直接把一个 Ray Dataset 交给它。`write_ray()` 通过 Ray
Datasink API 提交，因此对于 Ray 写入本身无需运行 `prepare_commit` / `commit`
步骤——只需在用完写入器后将其关闭：

```python
import ray

table = catalog.get_table('database_name.table_name')

# 1. Create table write and commit (commit is only needed for non-Ray writes
#    on the same table_write instance — see below).
write_builder = table.new_batch_write_builder()
table_write = write_builder.new_write()
table_commit = write_builder.new_commit()

# 2. Write Ray Dataset
ray_dataset = ray.data.read_json("/path/to/data.jsonl")
table_write.write_ray(ray_dataset, overwrite=False, concurrency=2)
# Parameters:
#   - dataset: Ray Dataset to write
#   - overwrite: Whether to overwrite existing data (default: False)
#   - concurrency: Optional max number of concurrent Ray tasks
#   - ray_remote_args: Optional kwargs passed to ray.remote() (e.g., {"num_cpus": 2})

# 3. Commit data (required for write_pandas/write_arrow/write_arrow_batch only)
commit_messages = table_write.prepare_commit()
table_commit.commit(commit_messages)

# 4. Close resources
table_write.close()
table_commit.close()
```

### 构建器层面的覆盖（overwrite） {#overwrite-at-builder-level}

通过 `write_paimon` 进行覆盖的推荐方式是上文中的 `overwrite=True`
标志。在使用更底层的构建器 API 时，你也可以
在写入构建器本身上配置覆盖模式：

```python
# overwrite whole table
write_builder = table.new_batch_write_builder().overwrite()

# overwrite partition 'dt=2024-01-01'
write_builder = table.new_batch_write_builder().overwrite({'dt': '2024-01-01'})
```
