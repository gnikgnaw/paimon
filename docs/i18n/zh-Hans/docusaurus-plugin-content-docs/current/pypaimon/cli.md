---
title: "命令行接口"
sidebar_position: 99
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

# 命令行接口 {#command-line-interface}

PyPaimon 提供了一个命令行接口（CLI），用于与 Paimon 的 Catalog 和表进行交互。
该 CLI 允许你直接从命令行读取 Paimon 表中的数据。

## 安装 {#installation}

当你安装 PyPaimon 时，CLI 会被自动安装：

```shell
pip install pypaimon
```

安装完成后，`paimon` 命令将在你的终端中可用。

## 基本用法 {#basic-usage}

在使用 CLI 之前，你需要创建一个 Catalog 配置文件。
默认情况下，CLI 会在当前目录下查找 `paimon.yaml` 文件。

创建一个包含你的 Catalog 设置的 `paimon.yaml` 文件：

**文件系统 Catalog：**

```yaml
metastore: filesystem
warehouse: /path/to/warehouse
```

**REST Catalog：**

```yaml
metastore: rest
uri: http://localhost:8080
warehouse: catalog_name
```

**用法：**

```shell
paimon [OPTIONS] COMMAND [ARGS]...
```

- `-c, --config PATH`：Catalog 配置文件的路径（默认：`paimon.yaml`）
- `--help`：显示帮助信息并退出

## 表命令 {#table-commands}

### 读取表 {#table-read}

从 Paimon 表中读取数据并以表格形式展示。

```shell
paimon table read mydb.users
```

**选项：**

- `--select, -s`：选择要读取的特定列（以逗号分隔）
- `--where, -w`：SQL 风格语法的过滤条件
- `--limit, -l`：要展示的最大结果数量（默认：100）
- `--format, -f`：输出格式：`table`（默认）或 `json`

**示例：**

```shell
# Read with limit
paimon table read mydb.users -l 50

# Read specific columns
paimon table read mydb.users -s id,name,age

# Filter with WHERE clause
paimon table read mydb.users --where "age > 18"

# Combine select, where, and limit
paimon table read mydb.users -s id,name -w "age >= 20 AND city = 'Beijing'" -l 50

# Output as JSON (for programmatic use)
paimon table read mydb.users --format json
```

**WHERE 运算符**

`--where` 选项支持 SQL 风格的过滤表达式：

| 运算符 | 示例 |
|---|---|
| `=`, `!=`, `<>` | `name = 'Alice'` |
| `<`, `<=`, `>`, `>=` | `age > 18` |
| `IS NULL`, `IS NOT NULL` | `deleted_at IS NULL` |
| `IN (...)`, `NOT IN (...)` | `status IN ('active', 'pending')` |
| `BETWEEN ... AND ...` | `age BETWEEN 20 AND 30` |
| `LIKE` | `name LIKE 'A%'` |

多个条件可以用 `AND` 和 `OR` 组合（`AND` 的优先级更高）。支持使用括号进行分组：

```shell
# AND condition
paimon table read mydb.users -w "age >= 20 AND age <= 30"

# OR condition
paimon table read mydb.users -w "city = 'Beijing' OR city = 'Shanghai'"

# Parenthesized grouping
paimon table read mydb.users -w "(age > 18 OR name = 'Bob') AND city = 'Beijing'"

# IN list
paimon table read mydb.users -w "city IN ('Beijing', 'Shanghai', 'Hangzhou')"

# BETWEEN
paimon table read mydb.users -w "age BETWEEN 25 AND 35"

# LIKE pattern
paimon table read mydb.users -w "name LIKE 'A%'"

# IS NULL / IS NOT NULL
paimon table read mydb.users -w "email IS NOT NULL"
```

字面量值会根据表 schema（表结构）自动转换为对应的 Python 类型（例如，`INT` 字段转换为 `int`，`DOUBLE` 转换为 `float`）。

输出：
```
 id    name  age      city
  1   Alice   25   Beijing
  2     Bob   30  Shanghai
  3 Charlie   35 Guangzhou
  4   David   28  Shenzhen
  5     Eve   32  Hangzhou
```

### 解释查询计划 {#table-explain}

在不读取任何数据的情况下展示一次查询的扫描计划：目标快照、下推的谓词 / 投影 / limit、分区 / 桶 / 文件统计裁剪漏斗，以及 split 级别的信号（可原始转换比例、删除向量比例、层（level）直方图、每个 split 的文件数与 split 大小分布）。在实际执行读取之前，可用于预览某个谓词的裁剪效果。

```shell
paimon table explain mydb.events
```

**选项：**

- `--select, -s`：投影特定列（以逗号分隔）
- `--where, -w`：SQL 风格语法的过滤条件（与 `table read` 使用相同的运算符）
- `--limit, -l`：要下推的行数限制
- `--verbose, -v`：列出每个 split 及其文件
- `--format, -f`：输出格式：`table`（默认）或 `json`

**示例：**

```shell
# Whole-table scan plan
paimon table explain mydb.events

# Push filter and projection through the planner
paimon table explain mydb.events --where "dt = '2026-05-16' AND id = 7" -s dt,id,val

# List every split (and its files) instead of just the aggregates
paimon table explain mydb.events -w "dt = '2026-05-16'" --verbose

# Machine-readable output for scripting (level_histogram keys are JSON strings)
paimon table explain mydb.events --format json
```

输出：
```
== PyPaimon Scan Plan ==
Table:              mydb.events (PK, HASH_FIXED)
Snapshot:           5  (schema 0)
Predicate:          (dt = '2026-05-16') AND (id = 7)
Projection:         [dt, id, val]
Limit:              <none>

Partition pruning:  20 -> 4  (pruned 16)
Bucket pruning:     4 -> 1  (pruned 3)
File skipping:      1 -> 1  (pruned 0)

Splits:             1
  raw-convertible:  1 / 1
  with DV:          0 / 1
  all-above-L0:     0 / 1
  files/split:      min=1  max=1  avg=1.00
  size/split:       min=2.6 KiB  p50=2.6 KiB  p95=2.6 KiB  max=2.6 KiB

Files:              1
Total size:         2.6 KiB
Estimated rows:     10   (merged: 10)
Level histogram:    L0=1
Deletion files:     0
```

`explain` 会读取 manifest 列表和 manifest（清单）文件，但绝不会打开任何数据文件，因此在大表上它比真正的读取要便宜得多。

### 获取表信息 {#table-get}

以 JSON 格式获取并展示表的 schema（表结构）信息。其输出格式与 table create 中使用的 schema JSON 格式相同，便于导出和复用表 schema。

```shell
paimon table get mydb.users
```

输出：
```json
{
  "fields": [
    {"id": 0, "name": "user_id", "type": "BIGINT"},
    {"id": 1, "name": "username", "type": "STRING"},
    {"id": 2, "name": "email", "type": "STRING"},
    {"id": 3, "name": "age", "type": "INT"},
    {"id": 4, "name": "city", "type": "STRING"},
    {"id": 5, "name": "created_at", "type": "TIMESTAMP"},
    {"id": 6, "name": "is_active", "type": "BOOLEAN"}
  ],
  "partitionKeys": ["city"],
  "primaryKeys": ["user_id"],
  "options": {
    "bucket": "4",
    "changelog-producer": "input"
  },
  "comment": "User information table"
}
```

**注意：** 输出的 JSON 可以保存到文件中，并直接配合 `table create` 命令使用，以重新创建表结构。

### 获取表快照 {#table-snapshot}

以 JSON 格式获取并展示 Paimon 表的最新快照信息。快照包含表当前状态的元数据。

```shell
paimon table snapshot mydb.users
```

输出：
```json
{
  "version": 3,
  "id": 5,
  "schemaId": 1,
  "baseManifestList": "manifest-list-5-base-...",
  "deltaManifestList": "manifest-list-5-delta-...",
  "changelogManifestList": null,
  "totalRecordCount": 1000,
  "deltaRecordCount": 100,
  "changelogRecordCount": null,
  "commitUser": "user-123",
  "commitIdentifier": 1709123456789,
  "commitKind": "APPEND",
  "timeMillis": 1709123456789,
  "watermark": null,
  "statistics": null,
  "nextRowId": null
}
```

### 创建表 {#table-create}

使用 JSON 文件中定义的 schema（表结构）创建一个新的 Paimon 表。该 schema JSON 格式与 `table get` 的输出相同，从而保证一致性并便于 schema 复用。

**选项：**

- `--schema, -s`：schema JSON 文件的路径 - **必填**
- `--ignore-if-exists, -i`：若表已存在则不抛出错误

schema JSON 文件遵循与 `table get` 输出相同的格式：

**字段属性：**

- `id`：字段 ID（整数，通常从 0 开始）- **必填**
- `name`：字段名 - **必填**
- `type`：字段数据类型（例如 `INT`、`BIGINT`、`STRING`、`TIMESTAMP`、`DECIMAL(10,2)`）- **必填**
- `description`：可选的字段描述

**Schema 属性：**

- `fields`：字段定义列表 - **必填**
- `partitionKeys`：分区键列名列表
- `primaryKeys`：主键列名列表
- `options`：以键值对表示的表配置项
- `comment`：表注释

**示例工作流：**

1. 从一个已有的表导出 schema：
   ```shell
   paimon table get mydb.users > users_schema.json
   ```

2. 使用相同的 schema 创建一个新表：
   ```shell
   paimon table create mydb.users_copy --schema users_schema.json
   ```

### 导入表数据 {#table-import}

从 CSV 或 JSON 文件向一个已有的 Paimon 表导入数据。这对于从外部数据源批量加载数据非常有用。

**选项：**

- `--input, -i`：输入文件的路径（CSV 或 JSON 格式）- **必填**

**支持的格式：**

- **CSV**（`.csv`）：逗号分隔值文件
- **JSON**（`.json`）：对象数组格式的 JSON 文件

#### 从 CSV 导入 {#import-from-csv}

CSV 文件应满足：
- 包含一个表头行，其列名与表 schema 匹配
- 数据类型与表的各列兼容

```csv
id,name,age,city
1,Alice,25,Beijing
2,Bob,30,Shanghai
3,Charlie,35,Guangzhou
```

输出：
```
Successfully imported 3 rows into 'mydb.users'.
```

#### 从 JSON 导入 {#import-from-json}

JSON 文件应为一个对象数组，其中各键与表的列名匹配。

```json
[
  {"id": 1, "name": "Alice", "age": 25, "city": "Beijing"},
  {"id": 2, "name": "Bob", "age": 30, "city": "Shanghai"},
  {"id": 3, "name": "Charlie", "age": 35, "city": "Guangzhou"}
]
```

输出：
```
Successfully imported 3 rows into 'mydb.users'.
```

#### 重要提示 {#important-notes}

- 在导入数据之前，目标表必须已存在
- 文件中的列名必须与表 schema 匹配
- 数据类型应与表 schema 兼容
- 导入操作会向已有的表追加数据

### 列出表分区 {#table-list-partitions}

列出 Paimon 表的分区。支持可选的模式过滤以匹配特定分区。

```shell
paimon table list-partitions mydb.orders
```

**选项：**

- `--pattern, -p`：用于过滤分区的分区名模式
- `--format, -f`：输出格式：`table`（默认）或 `json`

**示例：**

```shell
# List all partitions
paimon table list-partitions mydb.orders

# List partitions matching a pattern
paimon table list-partitions mydb.orders --pattern "dt=2024*"

# Output as JSON (for programmatic use)
paimon table list-partitions mydb.orders --format json
```

输出：
```
              Partition  RecordCount  FileSizeInBytes  FileCount  LastFileCreationTime       UpdatedAt  UpdatedBy
dt=2024-01-01,region=us          500          1048576         10         1704067200000  1704153600000      admin
dt=2024-01-02,region=eu          300           524288          5         1704153600000  1704240000000      user1
dt=2024-01-03,region=us          200           262144          3         1704240000000  1704326400000      admin
```

### 重命名表 {#table-rename}

在 Catalog 中重命名一个表。源和目标都必须以 `database.table` 格式指定。

```shell
paimon table rename mydb.old_name mydb.new_name
```

输出：
```
Table 'mydb.old_name' renamed to 'mydb.new_name' successfully.
```

**注意：** 文件系统 Catalog 和 REST Catalog 都支持表重命名。对于文件系统 Catalog，重命名是通过重命名底层的表目录来执行的。

### 表全文检索 {#table-full-text-search}

在一个带有 Tantivy 全文索引的 Paimon 表上执行全文检索，并展示匹配的行。

```shell
paimon table full-text-search mydb.articles --column content --query "paimon lake"
```

**选项：**

- `--column, -c`：要检索的文本列 - **必填**
- `--query, -q`：要检索的查询文本 - **必填**
- `--limit, -l`：要返回的最大结果数量（默认：10）
- `--select, -s`：选择要展示的特定列（以逗号分隔）
- `--format, -f`：输出格式：`table`（默认）或 `json`

**示例：**

```shell
# Basic full-text search
paimon table full-text-search mydb.articles -c content -q "paimon lake"

# Search with limit
paimon table full-text-search mydb.articles -c content -q "streaming data" -l 20

# Search with column projection
paimon table full-text-search mydb.articles -c content -q "paimon" -s "id,title,content"

# Output as JSON
paimon table full-text-search mydb.articles -c content -q "paimon" -f json
```

输出：
```
 id                                            content
  0  Apache Paimon is a streaming data lake platform
  2  Paimon supports real-time data ingestion and...
  4  Data lake platforms like Paimon handle large-...
```

**注意：** 该表必须在目标列上构建了 Tantivy 全文索引。关于如何创建全文索引，请参阅 [全局索引](../append-table/global-index)。

### 删除表 {#table-drop}

从 Catalog 中删除一个表。这将永久删除该表及其所有数据。

**选项：**

- `--ignore-if-not-exists, -i`：若表不存在则不抛出错误

```shell
paimon table drop mydb.old_table
```

输出：
```
Table 'mydb.old_table' dropped successfully.
```

**警告：** 此操作无法撤销。表中的所有数据将被永久删除。

### 修改表 {#table-alter}

修改一个表的 schema 或配置项。该命令支持多个子命令，用于不同类型的 schema 变更。

#### 基本语法 {#basic-syntax}

```shell
paimon table alter DATABASE.TABLE [--ignore-if-not-exists] SUBCOMMAND [OPTIONS]
```

**全局选项：**

- `--ignore-if-not-exists, -i`：若表不存在则不抛出错误

#### 设置配置项 {#set-option}

设置一个表配置项（键值对）：

```shell
paimon table alter mydb.users set-option -k snapshot.num-retained-max -v 10
```

#### 移除配置项 {#remove-option}

移除一个表配置项：

```shell
paimon table alter mydb.users remove-option -k snapshot.num-retained-max
```

#### 新增列 {#add-column}

向表中新增一列：

**示例：**

```shell
paimon table alter mydb.users add-column -n email -t STRING -c "User email address"
```

**带位置的示例（first）：**

```shell
paimon table alter mydb.users add-column -n row_id -t BIGINT --first
```

**带位置的示例（after）：**

```shell
paimon table alter mydb.users add-column -n email -t STRING --after name
```

#### 删除列 {#drop-column}

从表中删除一列：

```shell
paimon table alter mydb.users drop-column -n email
```

#### 重命名列 {#rename-column}

重命名一个已有的列：

```shell
paimon table alter mydb.users rename-column -n username -m user_name
```

#### 修改列 {#alter-column}

修改一个已有列的类型、注释或位置。可以在单条命令中指定多项变更。

**修改列类型：**

```shell
paimon table alter mydb.users alter-column -n age -t BIGINT
```

**修改列注释：**

```shell
paimon table alter mydb.users alter-column -n age -c 'User age in years'
```

**修改列位置：**

```shell
paimon table alter mydb.users alter-column -n age --first

paimon table alter mydb.users alter-column -n age --after name
```

**在单条命令中进行多项变更：**

```shell
paimon table alter mydb.users alter-column -n age -t BIGINT -c 'User age in years'
```

#### 更新注释 {#update-comment}

```shell
paimon table alter mydb.users update-comment -c "Updated user information table"
```

## 数据库命令 {#database-commands}

### 获取数据库信息 {#db-get}

以 JSON 格式获取并展示数据库信息。

```shell
paimon db get mydb
```

输出：
```json
{
  "name": "mydb",
  "options": {}
}
```

### 创建数据库 {#db-create}

创建一个新的数据库。

```shell
# Create a simple database
paimon db create mydb

# Create with properties
paimon db create mydb -p '{"key1": "value1", "key2": "value2"}'

# Create and ignore if already exists
paimon db create mydb -i
```

### 删除数据库 {#db-drop}

删除一个已有的数据库。

```shell
# Drop a database
paimon db drop mydb

# Drop and ignore if not exists
paimon db drop mydb -i

# Drop with all tables (cascade)
paimon db drop mydb --cascade
```

### 修改数据库 {#db-alter}

通过设置或移除属性来修改数据库属性。

```shell
# Set properties
paimon db alter mydb --set '{"key1": "value1", "key2": "value2"}'

# Remove properties
paimon db alter mydb --remove key1 key2

# Set and remove properties in one command
paimon db alter mydb --set '{"key1": "new_value"}' --remove key2
```

### 列出数据库中的表 {#db-list-tables}

列出一个数据库中的所有表。

```shell
paimon db list-tables mydb
```

输出：
```
orders
products
users
```

## Catalog 命令 {#catalog-commands}

### 列出 Catalog 中的数据库 {#catalog-list-dbs}

列出 Catalog 中的所有数据库。

```shell
paimon catalog list-dbs
```

输出：
```
default
mydb
analytics
```

## SQL 命令 {#sql-command}

直接从命令行在 Paimon 表上执行 SQL 查询。该功能由 pypaimon-rust 和 DataFusion 提供支持。

**前置条件：**

```shell
pip install pypaimon[sql]
```

### 一次性查询 {#one-shot-query}

执行单条 SQL 查询并展示结果：

```shell
paimon sql "SELECT * FROM users LIMIT 10"
```

输出：
```
 id    name  age      city
  1   Alice   25   Beijing
  2     Bob   30  Shanghai
  3 Charlie   35 Guangzhou
```

**选项：**

- `--format, -f`：输出格式：`table`（默认）或 `json`

**示例：**

```shell
# Direct table name (uses default catalog and database)
paimon sql "SELECT * FROM users"

# Two-part: database.table
paimon sql "SELECT * FROM mydb.users"

# Query with filter and aggregation
paimon sql "SELECT city, COUNT(*) AS cnt FROM users GROUP BY city ORDER BY cnt DESC"

# Output as JSON
paimon sql "SELECT * FROM users LIMIT 5" --format json
```

### 交互式 REPL {#interactive-repl}

不带查询参数运行 `paimon sql` 即可启动一个交互式 SQL 会话。该 REPL 支持使用方向键进行行编辑，并且命令历史会在 `~/.paimon_history` 中跨会话持久化保存。

```shell
paimon sql
```

输出：
```
    ____        _
   / __ \____ _(_)___ ___  ____  ____
  / /_/ / __ `/ / __ `__ \/ __ \/ __ \
 / ____/ /_/ / / / / / / / /_/ / / / /
/_/    \__,_/_/_/ /_/ /_/\____/_/ /_/

  Powered by pypaimon-rust + DataFusion
  Type 'help' for usage, 'exit' to quit.

paimon> SHOW DATABASES;
default
mydb

paimon> USE mydb;
Using database 'mydb'.

paimon> SHOW TABLES;
orders
users

paimon> SELECT count(*) AS cnt
     > FROM users
     > WHERE age > 18;
 cnt
  42
(1 row in 0.05s)

paimon> exit
Bye!
```

SQL 语句以 `;` 结尾，并且可以跨多行。续行提示符 `     >` 表示还需要更多的输入。

**REPL 命令：**

| 命令 | 描述 |
|---|---|
| `USE <database>;` | 切换默认数据库 |
| `SHOW DATABASES;` | 列出所有数据库 |
| `SHOW TABLES;` | 列出当前数据库中的表 |
| `SELECT ...;` | 执行一条 SQL 查询 |
| `help` | 显示用法信息 |
| `exit` / `quit` | 退出 REPL |

关于 SQL 语法和 Python API 的更多细节，请参阅 [SQL 查询](./sql)。
