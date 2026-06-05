---
title: "SQL 查询"
sidebar_position: 8
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

# SQL 查询 {#sql-query}

PyPaimon 支持在 Paimon 表上执行 SQL 查询，其底层由 [pypaimon-rust](https://github.com/apache/paimon-rust/tree/main/bindings/python) 和 [DataFusion](https://datafusion.apache.org/) 提供支持。

## 安装 {#installation}

SQL 查询支持需要额外的依赖。请使用以下命令安装：

```shell
pip install pypaimon[sql]
```

这会安装 `pypaimon-rust`（其中已捆绑 DataFusion）。

## 用法 {#usage}

创建一个 `SQLContext`，注册一个或多个 Catalog 及其配置项，然后运行 SQL 查询。

### 基本查询 {#basic-query}

```python
from pypaimon_rust.datafusion import SQLContext
import pyarrow as pa

ctx = SQLContext()
ctx.register_catalog("paimon", {"warehouse": "/path/to/warehouse"})
ctx.set_current_catalog("paimon")
ctx.set_current_database("default")

# Execute SQL and get PyArrow RecordBatches
batches = ctx.sql("SELECT * FROM my_table")
table = pa.Table.from_batches(batches)
print(table)

# Convert to Pandas DataFrame
df = table.to_pandas()
print(df)
```

`SQLContext` 也可以从 `pypaimon` 中导入：

```python
from pypaimon import SQLContext
```

### 表引用格式 {#table-reference-format}

默认 Catalog 和默认数据库可以通过 `set_current_catalog()` 和 `set_current_database()` 进行配置，因此你可以用多种方式引用表：

```python
# Direct table name (uses default database)
ctx.sql("SELECT * FROM my_table")

# Two-part: database.table
ctx.sql("SELECT * FROM mydb.my_table")

# Three-part: catalog.database.table
ctx.sql("SELECT * FROM paimon.mydb.my_table")
```

### 多 Catalog 查询 {#multi-catalog-query}

`SQLContext` 支持注册多个 Catalog 以进行跨 Catalog 查询：

```python
ctx = SQLContext()
ctx.register_catalog("a", {"warehouse": "/path/to/warehouse_a"})
ctx.register_catalog("b", {
    "metastore": "rest",
    "uri": "http://localhost:8080",
    "warehouse": "warehouse_b",
})
ctx.set_current_catalog("a")
ctx.set_current_database("default")

# Cross-catalog join
batches = ctx.sql("""
    SELECT a_users.name, b_orders.amount
    FROM a.default.users AS a_users
    JOIN b.default.orders AS b_orders ON a_users.id = b_orders.user_id
""")
```

### 注册 Arrow Batches {#register-arrow-batches}

你可以将 PyArrow RecordBatches 注册为临时表：

```python
batch = pa.record_batch([[1, 2], ["alice", "bob"]], names=["id", "name"])
ctx.register_batch("paimon.default.my_temp", batch)
batches = ctx.sql("SELECT * FROM paimon.default.my_temp")
```

## 支持的 SQL 语法 {#supported-sql-syntax}

该 SQL 引擎由 Apache DataFusion 提供支持，支持丰富的 SQL 语法集。完整的 SQL 参考请查看 [paimon-rust SQL 文档](https://paimon-rust.apache.org/sql.html)，其中涵盖：

- **DDL**：`CREATE SCHEMA`、`CREATE TABLE`（带 `PARTITIONED BY`、`PRIMARY KEY`、`WITH` 配置项）、`DROP TABLE`、`ALTER TABLE`、`CREATE TEMPORARY TABLE/VIEW`
- **DML**：`INSERT INTO`、`INSERT OVERWRITE`（动态/静态分区）、`UPDATE`、`DELETE`、`MERGE INTO`、`TRUNCATE TABLE`
- **存储过程（Procedure）**：`CALL sys.create_tag`、`CALL sys.rollback_to` 等
- **查询**：`SELECT`、列投影、谓词下推、`COUNT(*)` 下推
- **时间旅行**：`VERSION AS OF`、`TIMESTAMP AS OF`
- **向量检索**：`vector_search()` 表函数
- **全文检索**：`full_text_search()` 表函数
- **动态参数**：`SET` / `RESET`
- **系统表**：`$options`、`$schemas`、`$snapshots`、`$tags`、`$manifests`

关于 DataFusion 的查询语法（JOIN、聚合、子查询、CTE、窗口函数等），请查看 [DataFusion SQL 文档](https://datafusion.apache.org/user-guide/sql/index.html)。
