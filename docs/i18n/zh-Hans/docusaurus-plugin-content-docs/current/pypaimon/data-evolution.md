---
title: "数据演进"
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

# 数据演进 {#data-evolution}

数据演进（Data Evolution）模式下的 PyPaimon。参见 [Data Evolution](../append-table/data-evolution)。

## 前提条件 {#prerequisites}

要使用部分更新 / 数据演进，请在创建表时同时启用以下两个配置项：

- **`row-tracking.enabled`**：`true`
- **`data-evolution.enabled`**：`true`

## 批式与流式 {#batch-vs-stream}

数据演进同时支持批式（batch）与流式（stream）两种模式。API 差异如下：

|                     | 批式（Batch）                                | 流式（Stream）                                 |
|---------------------|----------------------------------------------|------------------------------------------------|
| Builder             | `table.new_batch_write_builder()`            | `table.new_stream_write_builder()`             |
| 写入                | `BatchTableWrite`                            | `StreamTableWrite`                             |
| 更新                | `BatchTableUpdate`                           | `StreamTableUpdate`                            |
| 提交                | `BatchTableCommit`                           | `StreamTableCommit`                            |
| `commit_identifier` | 不需要                                       | 必需（单调递增的整数）                          |
| 生命周期            | 一次性：每个实例只能提交一次                  | 可复用：同一实例可多轮提交                      |

不同模式之间存在差异的方法签名：

| 方法                            | 批式（Batch）                                  | 流式（Stream）                                                    |
|---------------------------------|------------------------------------------------|-------------------------------------------------------------------|
| `prepare_commit()`              | `write.prepare_commit()`                       | `write.prepare_commit(commit_identifier)`                         |
| `update_by_arrow_with_row_id()` | `update.update_by_arrow_with_row_id(table)`    | `update.update_by_arrow_with_row_id(table, commit_identifier)`    |
| `upsert_by_arrow_with_key()`    | `update.upsert_by_arrow_with_key(table, keys)` | `update.upsert_by_arrow_with_key(table, keys, commit_identifier)` |
| `commit()`                      | `commit.commit(messages)`                      | `commit.commit(messages, commit_identifier)`                      |

## 按 Row ID 更新列 {#update-columns-by-row-id}

你可以使用 `update_by_arrow_with_row_id` 来更新数据演进表中的列。

输入数据应包含 `_ROW_ID` 列。更新操作会自动对每个 `_ROW_ID` 进行排序，并将其与对应的 `first_row_id` 匹配，然后把具有相同 `first_row_id` 的行分组，并写入到一个单独的文件中。

**`_ROW_ID` 更新的要求**

- **仅更新列**：包含 `_ROW_ID` 以及你想要更新的列即可（部分 schema 也可以）。

### 批式模式 {#batch-mode}

```python
import pyarrow as pa
from pypaimon import CatalogFactory, Schema

catalog = CatalogFactory.create({'warehouse': '/tmp/warehouse'})
catalog.create_database('default', False)

simple_pa_schema = pa.schema([
  ('f0', pa.int8()),
  ('f1', pa.int16()),
])
schema = Schema.from_pyarrow_schema(simple_pa_schema,
                                    options={'row-tracking.enabled': 'true', 'data-evolution.enabled': 'true'})
catalog.create_table('default.test_row_tracking', schema, False)
table = catalog.get_table('default.test_row_tracking')

# write all columns
write_builder = table.new_batch_write_builder()
table_write = write_builder.new_write()
table_commit = write_builder.new_commit()
expect_data = pa.Table.from_pydict({
  'f0': [-1, 2],
  'f1': [-1001, 1002]
}, schema=simple_pa_schema)
table_write.write_arrow(expect_data)
table_commit.commit(table_write.prepare_commit())
table_write.close()
table_commit.close()

# update partial columns
write_builder = table.new_batch_write_builder()
table_update = write_builder.new_update().with_update_type(['f0'])
table_commit = write_builder.new_commit()
data2 = pa.Table.from_pydict({
  '_ROW_ID': [0, 1],
  'f0': [5, 6],
}, schema=pa.schema([
  ('_ROW_ID', pa.int64()),
  ('f0', pa.int8()),
]))
cmts = table_update.update_by_arrow_with_row_id(data2)
table_commit.commit(cmts)
table_commit.close()

# content should be:
#   'f0': [5, 6],
#   'f1': [-1001, 1002]
```

### 流式模式 {#stream-mode}

```python
import pyarrow as pa
from pypaimon import CatalogFactory, Schema

catalog = CatalogFactory.create({'warehouse': '/tmp/warehouse'})
catalog.create_database('default', False)

simple_pa_schema = pa.schema([
  ('f0', pa.int8()),
  ('f1', pa.int16()),
])
schema = Schema.from_pyarrow_schema(simple_pa_schema,
                                    options={'row-tracking.enabled': 'true', 'data-evolution.enabled': 'true'})
catalog.create_table('default.test_stream', schema, False)
table = catalog.get_table('default.test_stream')

# write initial data
write_builder = table.new_batch_write_builder()
table_write = write_builder.new_write()
table_commit = write_builder.new_commit()
table_write.write_arrow(pa.Table.from_pydict({
  'f0': [-1, 2],
  'f1': [-1001, 1002]
}, schema=simple_pa_schema))
table_commit.commit(table_write.prepare_commit())
table_write.close()
table_commit.close()

# stream update: each round uses a new commit_identifier
stream_builder = table.new_stream_write_builder()
table_update = stream_builder.new_update().with_update_type(['f0'])
table_commit = stream_builder.new_commit()

data1 = pa.Table.from_pydict({
  '_ROW_ID': [0],
  'f0': [5],
}, schema=pa.schema([('_ROW_ID', pa.int64()), ('f0', pa.int8())]))
cmts1 = table_update.update_by_arrow_with_row_id(data1, commit_identifier=1)
table_commit.commit(cmts1, commit_identifier=1)

data2 = pa.Table.from_pydict({
  '_ROW_ID': [1],
  'f0': [6],
}, schema=pa.schema([('_ROW_ID', pa.int64()), ('f0', pa.int8())]))
cmts2 = table_update.update_by_arrow_with_row_id(data2, commit_identifier=2)
table_commit.commit(cmts2, commit_identifier=2)

table_commit.close()
```

## 按 _ROW_ID 过滤 {#filter-by-_row_id}

需要满足相同的[前提条件](#prerequisites)（启用 row-tracking 与 data-evolution）。在这类表上，你可以按 `_ROW_ID` 过滤，从而在扫描时裁剪文件。支持：`equal('_ROW_ID', id)`、`is_in('_ROW_ID', [id1, ...])`、`between('_ROW_ID', low, high)`。

```python
pb = table.new_read_builder().new_predicate_builder()
rb = table.new_read_builder().with_filter(pb.equal('_ROW_ID', 0))
result = rb.new_read().to_arrow(rb.new_scan().plan().splits())
```

## 按 Key 进行 Upsert {#upsert-by-key}

如果你想按一个或多个业务键列对行进行 **upsert**（更新或插入）——而无需手动提供 `_ROW_ID`——请使用 `upsert_by_arrow_with_key`。对于每一条输入行：

- **Key 匹配**到某条已有行 → 就地更新该行。
- **未匹配** → 作为新行追加。

**要求**

- 表必须设置 `data-evolution.enabled = true` 与 `row-tracking.enabled = true`。
- 所有 `upsert_keys` 都必须同时存在于表 schema 与输入数据中。
- 对于**分区表**，输入数据必须包含全部分区键列。在匹配过程中分区键会被**自动剥离**出 `upsert_keys`（因为每个分区都是独立处理的），因此你**不需要**把它们包含在 `upsert_keys` 中。

### 批式模式 {#batch-mode-1}

**示例：基础 upsert**

```python
import pyarrow as pa
from pypaimon import CatalogFactory, Schema

catalog = CatalogFactory.create({'warehouse': '/tmp/warehouse'})
catalog.create_database('default', False)

pa_schema = pa.schema([
    ('id', pa.int32()),
    ('name', pa.string()),
    ('age', pa.int32()),
])
schema = Schema.from_pyarrow_schema(
    pa_schema,
    options={'row-tracking.enabled': 'true', 'data-evolution.enabled': 'true'},
)
catalog.create_table('default.users', schema, False)
table = catalog.get_table('default.users')

# write initial data
write_builder = table.new_batch_write_builder()
write = write_builder.new_write()
commit = write_builder.new_commit()
write.write_arrow(pa.Table.from_pydict(
    {'id': [1, 2], 'name': ['Alice', 'Bob'], 'age': [30, 25]},
    schema=pa_schema,
))
commit.commit(write.prepare_commit())
write.close()
commit.close()

# upsert: update id=1, insert id=3
write_builder = table.new_batch_write_builder()
table_update = write_builder.new_update()
table_commit = write_builder.new_commit()

upsert_data = pa.Table.from_pydict(
    {'id': [1, 3], 'name': ['Alice_v2', 'Charlie'], 'age': [31, 28]},
    schema=pa_schema,
)
cmts = table_update.upsert_by_arrow_with_key(upsert_data, upsert_keys=['id'])
table_commit.commit(cmts)
table_commit.close()

# content should be:
#   id=1: name='Alice_v2', age=31   (updated)
#   id=2: name='Bob',      age=25   (unchanged)
#   id=3: name='Charlie',  age=28   (new)
```

**示例：使用 `update_cols` 进行部分列 upsert**

将 `with_update_type` 与 `upsert_by_arrow_with_key` 结合使用，可以对匹配到的行仅更新指定的列，同时对新键仍然追加完整的行：

```python
write_builder = table.new_batch_write_builder()
table_update = write_builder.new_update().with_update_type(['age'])
table_commit = write_builder.new_commit()

upsert_data = pa.Table.from_pydict(
    {'id': [1, 4], 'name': ['ignored', 'David'], 'age': [99, 22]},
    schema=pa_schema,
)
cmts = table_update.upsert_by_arrow_with_key(upsert_data, upsert_keys=['id'])
table_commit.commit(cmts)
table_commit.close()

# id=1: only 'age' is updated to 99; 'name' remains 'Alice_v2'
# id=4: appended as a full new row
```

**示例：使用复合键的分区表**

```python
partitioned_schema = pa.schema([
    ('id', pa.int32()),
    ('name', pa.string()),
    ('region', pa.string()),
])
schema = Schema.from_pyarrow_schema(
    partitioned_schema,
    partition_keys=['region'],
    options={'row-tracking.enabled': 'true', 'data-evolution.enabled': 'true'},
)
catalog.create_table('default.users_partitioned', schema, False)
table = catalog.get_table('default.users_partitioned')

# ... write initial data ...

write_builder = table.new_batch_write_builder()
table_update = write_builder.new_update()
table_commit = write_builder.new_commit()

upsert_data = pa.Table.from_pydict(
    {'id': [1, 3], 'name': ['Alice_v2', 'Charlie'], 'region': ['US', 'EU']},
    schema=partitioned_schema,
)
# upsert_keys=['id'] only; partition key 'region' is auto-stripped
cmts = table_update.upsert_by_arrow_with_key(upsert_data, upsert_keys=['id'])
table_commit.commit(cmts)
table_commit.close()
```

**注意**

- 执行是**逐分区**进行的：内存中一次只加载一个分区的键集合。
- 输入数据中的重复键会被自动去重——保留**最后一次出现**的那条。
- upsert 在每次提交内是原子的——所有匹配的更新与新的追加都包含在同一次提交中。

### 流式模式 {#stream-mode-1}

```python
import pyarrow as pa
from pypaimon import CatalogFactory, Schema

catalog = CatalogFactory.create({'warehouse': '/tmp/warehouse'})
catalog.create_database('default', False)

pa_schema = pa.schema([
    ('id', pa.int32()),
    ('name', pa.string()),
    ('age', pa.int32()),
])
schema = Schema.from_pyarrow_schema(
    pa_schema,
    options={'row-tracking.enabled': 'true', 'data-evolution.enabled': 'true'},
)
catalog.create_table('default.users_stream', schema, False)
table = catalog.get_table('default.users_stream')

# write initial data
write_builder = table.new_batch_write_builder()
write = write_builder.new_write()
commit = write_builder.new_commit()
write.write_arrow(pa.Table.from_pydict(
    {'id': [1, 2], 'name': ['Alice', 'Bob'], 'age': [30, 25]},
    schema=pa_schema,
))
commit.commit(write.prepare_commit())
write.close()
commit.close()

# stream upsert: each round uses a new commit_identifier
stream_builder = table.new_stream_write_builder()
table_update = stream_builder.new_update()
table_commit = stream_builder.new_commit()

upsert_data1 = pa.Table.from_pydict(
    {'id': [1, 3], 'name': ['Alice_v2', 'Charlie'], 'age': [31, 28]},
    schema=pa_schema,
)
cmts1 = table_update.upsert_by_arrow_with_key(upsert_data1, upsert_keys=['id'], commit_identifier=1)
table_commit.commit(cmts1, commit_identifier=1)

upsert_data2 = pa.Table.from_pydict(
    {'id': [2, 4], 'name': ['Bob_v2', 'David'], 'age': [26, 40]},
    schema=pa_schema,
)
cmts2 = table_update.upsert_by_arrow_with_key(upsert_data2, upsert_keys=['id'], commit_identifier=2)
table_commit.commit(cmts2, commit_identifier=2)

table_commit.close()
```

批式模式中展示的 `with_update_type` 以及分区表用法在流式模式下同样适用——只需在 `upsert_by_arrow_with_key` 与 `commit` 调用中加上 `commit_identifier` 即可。

## 按分片更新列 {#update-columns-by-shards}

如果你想**计算一个派生列**（或**基于其他列更新某个已有列**）而又不提供 `_ROW_ID`，可以使用分片扫描 + 重写（shard scan + rewrite）的工作流：

- 仅读取你需要的列（投影）
- 以相同的行顺序计算新值
- 仅写回被更新的列
- 按分片提交

这对于回填新增的列，或基于其他列重新计算某列非常有用。

### 批式模式 {#batch-mode-2}

**示例：计算 `d = c + b - a`**

```python
import pyarrow as pa
from pypaimon import CatalogFactory, Schema

catalog = CatalogFactory.create({'warehouse': '/tmp/warehouse'})
catalog.create_database('default', False)

table_schema = pa.schema([
    ('a', pa.int32()),
    ('b', pa.int32()),
    ('c', pa.int32()),
    ('d', pa.int32()),
])

schema = Schema.from_pyarrow_schema(
    table_schema,
    options={'row-tracking.enabled': 'true', 'data-evolution.enabled': 'true'},
)
catalog.create_table('default.t', schema, False)
table = catalog.get_table('default.t')

# write initial data (a, b, c only)
write_builder = table.new_batch_write_builder()
write = write_builder.new_write().with_write_type(['a', 'b', 'c'])
commit = write_builder.new_commit()
write.write_arrow(pa.Table.from_pydict({'a': [1, 2], 'b': [10, 20], 'c': [100, 200]}))
commit.commit(write.prepare_commit())
write.close()
commit.close()

# shard update: read (a, b, c), write only (d)
update = write_builder.new_update()
update.with_read_projection(['a', 'b', 'c'])
update.with_update_type(['d'])

shard_idx = 0
num_shards = 1
upd = update.new_shard_updator(shard_idx, num_shards)
reader = upd.arrow_reader()

for batch in iter(reader.read_next_batch, None):
    a = batch.column('a').to_pylist()
    b = batch.column('b').to_pylist()
    c = batch.column('c').to_pylist()
    d = [ci + bi - ai for ai, bi, ci in zip(a, b, c)]

    upd.update_by_arrow_batch(
        pa.RecordBatch.from_pydict({'d': d}, schema=pa.schema([('d', pa.int32())]))
    )

commit_messages = upd.prepare_commit()
commit = write_builder.new_commit()
commit.commit(commit_messages)
commit.close()
```

**示例：更新一个已有列 `c = b - a`**

```python
update = write_builder.new_update()
update.with_read_projection(['a', 'b'])
update.with_update_type(['c'])

upd = update.new_shard_updator(0, 1)
reader = upd.arrow_reader()
for batch in iter(reader.read_next_batch, None):
    a = batch.column('a').to_pylist()
    b = batch.column('b').to_pylist()
    c = [bi - ai for ai, bi in zip(a, b)]
    upd.update_by_arrow_batch(
        pa.RecordBatch.from_pydict({'c': c}, schema=pa.schema([('c', pa.int32())]))
    )

commit_messages = upd.prepare_commit()
commit = write_builder.new_commit()
commit.commit(commit_messages)
commit.close()
```

### 流式模式 {#stream-mode-2}

```python
import pyarrow as pa
from pypaimon import CatalogFactory, Schema

catalog = CatalogFactory.create({'warehouse': '/tmp/warehouse'})
catalog.create_database('default', False)

table_schema = pa.schema([
    ('a', pa.int32()),
    ('b', pa.int32()),
    ('c', pa.int32()),
    ('d', pa.int32()),
])

schema = Schema.from_pyarrow_schema(
    table_schema,
    options={'row-tracking.enabled': 'true', 'data-evolution.enabled': 'true'},
)
catalog.create_table('default.t_stream', schema, False)
table = catalog.get_table('default.t_stream')

# write initial data (a, b, c only)
write_builder = table.new_batch_write_builder()
write = write_builder.new_write().with_write_type(['a', 'b', 'c'])
commit = write_builder.new_commit()
write.write_arrow(pa.Table.from_pydict({'a': [1, 2], 'b': [10, 20], 'c': [100, 200]}))
commit.commit(write.prepare_commit())
write.close()
commit.close()

# stream shard update: each round uses a new commit_identifier
stream_builder = table.new_stream_write_builder()
table_update = stream_builder.new_update()
table_update.with_read_projection(['a', 'b', 'c'])
table_update.with_update_type(['d'])
table_commit = stream_builder.new_commit()

upd = table_update.new_shard_updator(0, 1)
reader = upd.arrow_reader()

for batch in iter(reader.read_next_batch, None):
    a = batch.column('a').to_pylist()
    b = batch.column('b').to_pylist()
    c = batch.column('c').to_pylist()
    d = [ci + bi - ai for ai, bi, ci in zip(a, b, c)]

    upd.update_by_arrow_batch(
        pa.RecordBatch.from_pydict({'d': d}, schema=pa.schema([('d', pa.int32())]))
    )

commit_messages = upd.prepare_commit()
table_commit.commit(commit_messages, commit_identifier=1)
```

**注意**

- **行顺序很重要**：你写入的批次必须与读取的批次具有**相同的行数**，且在该分片中保持相同的顺序。
- **并行度**：通过为每个分片调用 `new_shard_updator(shard_idx, num_shards)` 来运行多个分片。
