---
title: "Views"
sidebar_position: 10
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

# 视图（Views） {#views}

视图（view）是一种封装了业务逻辑和领域特定语义的逻辑表。
虽然大多数计算引擎原生支持视图，但每种引擎都以各自专有的格式存储视图元数据，从而在不同平台之间造成了互操作性方面的挑战。
Paimon 视图对引擎特定的查询方言进行抽象，并建立统一的元数据标准。
视图元数据可以实现集中式的视图管理，从而便于跨引擎共享，并在异构计算环境中降低维护复杂度。

## Catalog 支持 {#catalog-support}

只有当 Catalog 实现支持视图时，视图元数据才会被持久化：

- **Hive metastore Catalog** —— 视图元数据与表元数据一起存储在 metastore 的 warehouse（仓库目录）中。
- **REST Catalog** —— 视图元数据保存在 REST 后端中，并通过 Catalog API 暴露出来。

文件系统 Catalog 目前不支持视图，因为它们缺乏持久化的元数据存储能力。


### 表示结构 {#representation-structure}

| 字段     | 类型 | 说明 |
|-----------|------|-------------|
| `query`   | `string` | 定义该视图的规范 SQL `SELECT` 语句。 |
| `dialect` | `string` | SQL 方言标识符（例如 `spark` 或 `flink`）。 |

同一个版本可以存储多个表示，从而让不同引擎能够使用各自的原生方言访问该视图。

## 操作 {#operations}

### 创建或替换视图 {#create-or-replace-view}

使用 `CREATE VIEW` 或 `CREATE OR REPLACE VIEW` 来注册一个视图。Paimon 会分配一个 UUID，写入第一个元数据文件，并记录版本 `1`。

```sql
CREATE VIEW sales_view AS
SELECT region, SUM(amount) AS total_amount
FROM sales
GROUP BY region;
```

### 通过存储过程修改视图方言 {#alter-view-dialect-via-procedure}

Paimon 提供了 `sys.alter_view_dialect` 存储过程（Procedure），使引擎能够为同一视图版本管理多个 SQL 表示。

#### Flink 示例 {#flink-example}

```sql
-- Add a Flink dialect
CALL [catalog.]sys.alter_view_dialect('view_identifier', 'add', 'flink', 'SELECT ...');

-- Update the stored Flink dialect
CALL [catalog.]sys.alter_view_dialect('view_identifier', 'update', 'flink', 'SELECT ...');

-- Drop the Flink dialect representation
CALL [catalog.]sys.alter_view_dialect('view_identifier', 'drop', 'flink');
```

#### Spark 示例 {#spark-example}

```sql
-- Add a Spark dialect
CALL sys.alter_view_dialect('view_identifier', 'add', 'spark', 'SELECT ...');

-- Update the Spark dialect
CALL sys.alter_view_dialect('view_identifier', 'update', 'spark', 'SELECT ...');

-- Drop the Spark dialect
CALL sys.alter_view_dialect('view_identifier', 'drop', 'spark');
```

### 删除视图 {#drop-view}

`DROP VIEW view_name;`

## 参见 {#see-also}

- [Spark SQL DDL – 视图](../spark/sql-ddl#view)
- [REST Catalog 概述](./rest/overview)
- [REST Catalog 视图 API](./rest/rest-api)
