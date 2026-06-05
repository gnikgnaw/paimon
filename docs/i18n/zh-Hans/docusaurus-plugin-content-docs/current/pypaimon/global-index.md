---
title: "全局索引"
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

# 全局索引 {#global-index}

PyPaimon 支持查询构建在 Data Evolution（append）表上的全局索引。提供三种索引类型：

- **BTree 索引**：基于 B 树的索引，用于标量列查找。支持等值、IN、范围以及组合谓词。
- **向量索引（Lumina）**：用于向量相似度检索的近似最近邻（ANN）索引。
- **全文索引（Tantivy）**：用于文本检索的全文检索索引，并带有相关性评分。

> 全局索引必须事先构建（例如通过 Spark 或 Flink）。关于如何创建索引，请参阅 [全局索引](../append-table/global-index)。

## BTree 索引 {#btree-index}

当过滤谓词匹配到被索引的列时，扫描过程中会自动使用 BTree 索引。无需特殊的 API —— 只需在 read builder 上设置一个过滤条件即可。

```python
import pypaimon

catalog = pypaimon.create_catalog(...)
table = catalog.get_table("db.my_table")

# BTree index is used automatically when filtering on indexed columns
read_builder = table.new_read_builder()
read_builder = read_builder.with_filter(
    pypaimon.PredicateBuilder(table.fields)
    .in_("name", ["a200", "a300"])
)

scan = read_builder.new_scan()
read = read_builder.new_read()
splits = scan.plan().splits
data = read.to_arrow(splits)
```

支持的谓词：`equal`、`not_equal`、`less_than`、`less_or_equal`、`greater_than`、`greater_or_equal`、`in_`、`not_in`、`between`、`is_null`、`is_not_null`。

## 向量索引（Lumina） {#vector-index-lumina}

使用 `VectorSearchBuilder` 在向量列上执行近似最近邻检索，然后读取匹配到的行。

```python
table = catalog.get_table("db.my_table")

# Step 1: vector search to get matching row IDs
builder = table.new_vector_search_builder()
index_result = (
    builder
    .with_vector_column("embedding")
    .with_query_vector([1.0, 2.0, 3.0, ...])
    .with_limit(10)
    .execute_local()
)

# Step 2: read actual data for matched rows
read_builder = table.new_read_builder()
scan = read_builder.new_scan()
scan.with_global_index_result(index_result)
read = read_builder.new_read()
data = read.to_arrow(scan.plan().splits)
```

你还可以添加标量过滤条件，在向量检索之前对行进行预过滤：

```python
predicate = (
    pypaimon.PredicateBuilder(table.fields)
    .equal("category", "electronics")
)

index_result = (
    table.new_vector_search_builder()
    .with_vector_column("embedding")
    .with_query_vector([1.0, 2.0, 3.0, ...])
    .with_limit(10)
    .with_filter(predicate)
    .execute_local()
)

read_builder = table.new_read_builder()
scan = read_builder.new_scan()
scan.with_global_index_result(index_result)
read = read_builder.new_read()
data = read.to_arrow(scan.plan().splits)
```

## 全文索引（Tantivy） {#full-text-index-tantivy}

使用 `FullTextSearchBuilder` 在文本列上执行全文检索，然后读取匹配到的行。

```python
table = catalog.get_table("db.my_table")

# Step 1: full-text search to get matching row IDs
builder = table.new_full_text_search_builder()
index_result = (
    builder
    .with_text_column("content")
    .with_query_text("search keywords")
    .with_limit(20)
    .execute_local()
)

# Step 2: read actual data for matched rows
read_builder = table.new_read_builder()
scan = read_builder.new_scan()
scan.with_global_index_result(index_result)
read = read_builder.new_read()
data = read.to_arrow(scan.plan().splits)
```

为了在从远程存储读取时获得更好的性能，可以考虑启用 [本地缓存](../program-api/file-cache)。
