---
title: "Daft"
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

# Daft {#daft}

[Daft](https://www.daft.io/) 是一个面向 Python 的分布式 DataFrame 引擎。

这需要安装 `daft`：

```bash
pip install pypaimon[daft]
```

`pypaimon.daft` 暴露了一套顶层的 `read_paimon` / `write_paimon` API，可直接接收表标识符与 Catalog 配置项。

## 读 {#read}

### `read_paimon`（推荐） {#read_paimon-recommended}

```python
from pypaimon.daft import read_paimon

df = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
)

df.show()
```

`read_paimon` 会打开自己的 Catalog，并在一次调用中完成对表的解析。

返回的 DataFrame 是惰性的。请使用标准的 Daft 操作进行过滤、投影和限制（limit）——它们会通过 Daft 的 DataSource 协议自动下推到 Paimon 扫描中：

```python
import daft

df = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
)

# Filter pushdown (partition pruning + file-level skipping)
df = df.where(daft.col("dt") == "2024-01-01")

# Projection pushdown (only requested columns are read from disk)
df = df.select("id", "name")

# Limit pushdown
df = df.limit(100)

df.show()
```

**时间旅行：**

```python
# Read a specific snapshot.
df = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    snapshot_id=42,
)

# Read a tagged snapshot.
df = read_paimon(
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    tag_name="release-2026-04",
)
```

`snapshot_id` 与 `tag_name` 互斥。

**参数：**
- `table_identifier`：完整表名，例如 `"db_name.table_name"`。
- `catalog_options`：转发给 `CatalogFactory.create()` 的关键字参数，
  例如 `{"warehouse": "/path/to/warehouse"}`。
- `snapshot_id`：可选，用于时间旅行的快照 id。与
  `tag_name` 互斥。
- `tag_name`：可选，用于时间旅行的标签（Tag）名。与
  `snapshot_id` 互斥。
- `io_config`：可选的 Daft `IOConfig`，用于访问对象存储。
  若为 `None`，将从 Catalog 配置项中推断。

对于位于对象存储上的表，凭据会自动从 Catalog 配置项中推断，你也可以显式传入一个 `IOConfig`：

```python
from daft.io import IOConfig, S3Config

df = read_paimon(
    "my_db.my_table",
    catalog_options={
        "warehouse": "s3://my-bucket/warehouse",
        "fs.s3.accessKeyId": "...",
        "fs.s3.accessKeySecret": "...",
    },
)
df.show()
```

**特性：**
- 使用 Parquet 格式的 Append 表（追加表）会使用 Daft 原生的高性能 Parquet 读取器。
- 需要 LSM 合并的主键表会回退到 pypaimon 内置的读取器。
- 支持分区裁剪、谓词下推、投影下推以及 limit 下推。

## 写 {#write}

### `write_paimon`（推荐） {#write_paimon-recommended}

```python
import daft
from pypaimon.daft import write_paimon

df = daft.from_pydict({
    "id": [1, 2, 3],
    "name": ["alice", "bob", "charlie"],
    "dt": ["2024-01-01", "2024-01-01", "2024-01-01"],
})

write_paimon(
    df,
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
)
```

`write_paimon` 会打开自己的 Catalog、解析表，并通过 Daft 的 DataSink API 提交写入。

**覆盖（overwrite）模式：**

```python
write_paimon(
    df,
    "database_name.table_name",
    catalog_options={"warehouse": "/path/to/warehouse"},
    mode="overwrite",
)
```

**参数：**
- `df`：要写入的 Daft DataFrame。
- `table_identifier`：完整表名，例如 `"db_name.table_name"`。
- `catalog_options`：转发给 `CatalogFactory.create()` 的关键字参数。
- `mode`：写入模式——`"append"`（默认）或 `"overwrite"`。

## Catalog 抽象 {#catalog-abstraction}

Paimon 的 Catalog 可以与 Daft 统一的 `Catalog` / `Table` 接口集成：

```python
import pypaimon
from pypaimon.daft import PaimonCatalog

inner = pypaimon.CatalogFactory.create({"warehouse": "/path/to/warehouse"})
catalog = PaimonCatalog(inner, name="my_paimon")

# Browse
catalog.list_namespaces()
catalog.list_tables()

# Read / write through catalog
table = catalog.get_table("my_db.my_table")
df = table.read()
table.append(df)
table.overwrite(df)
```

你也可以直接包装单个表：

```python
from pypaimon.daft import PaimonTable

inner_table = inner.get_table("my_db.my_table")
table = PaimonTable(inner_table)
df = table.read()
```

### 创建表 {#creating-tables}

```python
import daft
from daft.io.partitioning import PartitionField

schema = daft.from_pydict({"id": [1], "name": ["a"], "dt": ["2024-01-01"]}).schema()
dt_field = schema["dt"]
partition_fields = [PartitionField.create(dt_field)]

table = catalog.create_table(
    "my_db.new_table",
    schema,
    partition_fields=partition_fields,
)

# Primary-key table
table = catalog.create_table(
    "my_db.pk_table",
    schema,
    properties={"primary_keys": ["id", "dt"]},
    partition_fields=partition_fields,
)
```
