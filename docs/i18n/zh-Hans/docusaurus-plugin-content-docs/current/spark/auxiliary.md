---
title: "Auxiliary"
sidebar_position: 7
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

# 辅助语句（Auxiliary Statements） {#auxiliary-statements}

## Set / Reset {#set-reset}
SET 命令用于设置一个属性、返回某个已有属性的值，或返回所有 SQLConf 属性及其取值和含义。
RESET 命令用于将当前会话中通过 SET 命令设置的运行时配置重置为其默认值。

若要在全局范围内设置动态参数（dynamic options），需要添加 `spark.paimon.` 前缀。你也可以按以下格式设置动态表配置项：
`spark.paimon.${catalogName}.${dbName}.${tableName}.${config_key}`。其中 catalogName/dbName/tableName 可以为 `*`，表示匹配对应部分的所有内容。当存在冲突时，动态表配置项会覆盖全局配置项。

```sql
-- set spark conf
SET spark.sql.sources.partitionOverwriteMode=dynamic;

-- set paimon conf
SET spark.paimon.file.block-size=512M;

-- reset conf
RESET spark.paimon.file.block-size;

-- set scan.snapshot-id=1 for the table default.T in any catalogs
SET spark.paimon.*.default.T.scan.snapshot-id=1;
SELECT * FROM default.T;

-- set scan.snapshot-id=1 for the table T in any databases and catalogs
SET spark.paimon.*.*.T.scan.snapshot-id=1;
SELECT * FROM default.T;

-- set scan.snapshot-id=2 for the table default.T1 in any catalogs and scan.snapshot-id=1 on other tables
SET spark.paimon.scan.snapshot-id=1;
SET spark.paimon.*.default.T1.scan.snapshot-id=2;
SELECT * FROM default.T1 JOIN default.T2 ON xxxx;
```

## Describe table {#describe-table}
DESCRIBE TABLE 语句返回一张表或视图的基本元数据信息。元数据信息包括列名、列类型和列注释。

```sql
-- describe table or view
DESCRIBE TABLE my_table;

-- describe table or view with additional metadata
DESCRIBE TABLE EXTENDED my_table;
```

## Show create table {#show-create-table}
SHOW CREATE TABLE 返回用于创建给定表或视图的 CREATE TABLE 语句或 CREATE VIEW 语句。

```sql
SHOW CREATE TABLE my_table;
```

## Show columns {#show-columns}
返回一张表中的列清单。如果该表不存在，则抛出异常。

```sql
SHOW COLUMNS FROM my_table;
```

## Show partitions {#show-partitions}
SHOW PARTITIONS 语句用于列出一张表的分区。可以指定一个可选的分区规约（partition spec），以返回与所提供分区规约相匹配的分区。

```sql
-- Lists all partitions for my_table
SHOW PARTITIONS my_table;

-- Lists partitions matching the supplied partition spec for my_table
SHOW PARTITIONS my_table PARTITION (dt='20230817');
```

## Show table extended {#show-table-extended}
SHOW TABLE EXTENDED 语句用于列出表或分区信息。

```sql
-- Lists tables that satisfy regular expressions
SHOW TABLE EXTENDED IN db_name LIKE 'test*';

-- Lists the specified partition information for the table
SHOW TABLE EXTENDED IN db_name LIKE 'table_name' PARTITION(pt = '2024');
```

## Show views {#show-views}
SHOW VIEWS 语句返回某个可选指定数据库中的所有视图。

```sql
-- Lists all views
SHOW VIEWS;

-- Lists all views that satisfy regular expressions
SHOW VIEWS LIKE 'test*';
```

## Analyze table {#analyze-table}

ANALYZE TABLE 语句用于收集关于表的统计信息，这些信息将被查询优化器用于寻找更优的查询执行计划。
Paimon 支持通过 analyze 收集表级别的统计信息和列统计信息。

```sql
-- collect table-level statistics
ANALYZE TABLE my_table COMPUTE STATISTICS;

-- collect table-level statistics and column statistics for col1
ANALYZE TABLE my_table COMPUTE STATISTICS FOR COLUMNS col1;

-- collect table-level statistics and column statistics for all columns
ANALYZE TABLE my_table COMPUTE STATISTICS FOR ALL COLUMNS;
```

## Refresh table {#refresh-table}

REFRESH TABLE 语句用于使缓存的条目失效，这些条目包括给定表的数据和元数据。

特别地，当启用缓存 Catalog 时，Paimon 会自动缓存表的元数据。在多会话场景下，当一张表在某个会话中被重新创建后，必须在另一个会话中使用该命令来清除缓存。

```sql
REFRESH TABLE table_identifier;
```
