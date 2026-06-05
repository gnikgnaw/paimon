---
title: "Action Jars"
sidebar_position: 98
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

# Action Jars {#action-jars}

启动 Flink 本地集群（Local Cluster）后，你可以使用以下命令来执行 Action jar。

```bash
<FLINK_HOME>/bin/flink run \
 /path/to/paimon-flink-action-@@VERSION@@.jar \
 <action>
 <args>
``` 

下面的命令用于对一张表执行 Compaction。

```bash
<FLINK_HOME>/bin/flink run \
 /path/to/paimon-flink-action-@@VERSION@@.jar \
 compact \
 --path <TABLE_PATH>
```

## 合并到表中（Merging into table） {#merging-into-table}

Paimon 支持通过 `flink run` 提交 'merge_into' 作业来实现 "MERGE INTO"。

:::info

重要的表属性设置：
1. 只有[主键表（primary key table）](../primary-key-table/overview)支持该特性。
2. 该 Action 不会产生 UPDATE_BEFORE，因此不推荐设置 'changelog-producer' = 'input'。

:::

其设计参考了如下语法：
```sql
MERGE INTO target-table
  USING source_table | source-expr AS source-alias
  ON merge-condition
  WHEN MATCHED [AND matched-condition]
    THEN UPDATE SET xxx
  WHEN MATCHED [AND matched-condition]
    THEN DELETE
  WHEN NOT MATCHED [AND not_matched_condition]
    THEN INSERT VALUES (xxx)
  WHEN NOT MATCHED BY SOURCE [AND not-matched-by-source-condition]
    THEN UPDATE SET xxx
  WHEN NOT MATCHED BY SOURCE [AND not-matched-by-source-condition]
    THEN DELETE
```
merge_into action 使用 "upsert" 语义而非 "update" 语义，也就是说如果该行已存在，
则执行更新，否则执行插入。例如，对于非主键表，你可以更新每一列，
但对于主键表，如果你想更新主键，就必须插入一个新行，其主键与表中已有行不同。
在这种场景下，"upsert" 非常有用。

运行以下命令为该表提交一个 'merge_into' 作业。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    merge_into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <target-table> \
    [--target_as <target-table-alias>] \
    --source_table <source_table-name> \
    [--source_sql <sql> ...]\
    --on <merge-condition> \
    --merge_actions <matched-upsert,matched-delete,not-matched-insert,not-matched-by-source-upsert,not-matched-by-source-delete> \
    --matched_upsert_condition <matched-condition> \
    --matched_upsert_set <upsert-changes> \
    --matched_delete_condition <matched-condition> \
    --not_matched_insert_condition <not-matched-condition> \
    --not_matched_insert_values <insert-values> \
    --not_matched_by_source_upsert_condition <not-matched-by-source-condition> \
    --not_matched_by_source_upsert_set <not-matched-upsert-changes> \
    --not_matched_by_source_delete_condition <not-matched-by-source-condition> \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]]
    
You can pass sqls by '--source_sql <sql> [, --source_sql <sql> ...]' to config environment and create source table at runtime.
    
-- Examples:
-- Find all orders mentioned in the source table, then mark as important if the price is above 100 
-- or delete if the price is under 10.
./flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    merge_into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table T \
    --source_table S \
    --on "T.id = S.order_id" \
    --merge_actions \
    matched-upsert,matched-delete \
    --matched_upsert_condition "T.price > 100" \
    --matched_upsert_set "mark = 'important'" \
    --matched_delete_condition "T.price < 10" 
    
-- For matched order rows, increase the price, and if there is no match, insert the order from the 
-- source table:
./flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    merge_into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table T \
    --source_table S \
    --on "T.id = S.order_id" \
    --merge_actions \
    matched-upsert,not-matched-insert \
    --matched_upsert_set "price = T.price + 20" \
    --not_matched_insert_values * 

-- For not matched by source order rows (which are in the target table and does not match any row in the
-- source table based on the merge-condition), decrease the price or if the mark is 'trivial', delete them:
./flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    merge_into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table T \
    --source_table S \
    --on "T.id = S.order_id" \
    --merge_actions \
    not-matched-by-source-upsert,not-matched-by-source-delete \
    --not_matched_by_source_upsert_condition "T.mark <> 'trivial'" \
    --not_matched_by_source_upsert_set "price = T.price - 20" \
    --not_matched_by_source_delete_condition "T.mark = 'trivial'"
    
-- A --source_sql example: 
-- Create a temporary view S in new catalog and use it as source table
./flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    merge_into \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table T \
    --source_sql "CREATE CATALOG test_cat WITH (...)" \
    --source_sql "CREATE TEMPORARY VIEW test_cat.`default`.S AS SELECT order_id, price, 'important' FROM important_order" \
    --source_table test_cat.default.S \
    --on "T.id = S.order_id" \
    --merge_actions not-matched-insert\
    --not_matched_insert_values *
```

关于 'matched'（匹配）一词的说明：
1. matched（匹配）：发生变更的行来自目标表（target table），且每一行都能根据 merge-condition 以及可选的
   matched-condition 匹配到一条源表（source table）的行（source ∩ target）。
2. not matched（未匹配）：发生变更的行来自源表，且所有行都无法根据 merge-condition 以及可选的
   not_matched_condition 匹配到任何目标表的行（source - target）。
3. not matched by source（源端未匹配）：发生变更的行来自目标表，且所有行都无法根据 merge-condition 以及可选的
   not-matched-by-source-condition 匹配到任何源表的行（target - source）。

参数格式：
1. matched_upsert_changes：\
   col = \<source_table>.col | expression [, ...]（表示用给定值设置 \<target_table>.col。请勿
   在 'col' 前面加上 '\<target_table>.'。）\
   特别地，你可以使用 '*' 来用所有源列设置各列（要求目标表的
   schema 与源表相同）。
2. not_matched_upsert_changes 与 matched_upsert_changes 类似，但你不能引用
   源表的列，也不能使用 '*'。
3. insert_values：\
   col1, col2, ..., col_end\
   必须指定所有列的值。对于每一列，你可以引用 \<source_table>.col，或者
   使用一个表达式。\
   特别地，你可以使用 '*' 来用所有源列进行插入（要求目标表的 schema
   与源表相同）。
4. not_matched_condition 不能使用目标表的列来构造条件表达式。
5. not_matched_by_source_condition 不能使用源表的列来构造条件表达式。

:::warning

1. 目标别名（Target alias）不能与已存在的表名重复。
2. 如果源表不在当前 Catalog 和当前数据库中，则源表名（source-table-name）必须是
   限定名（database.table，或者在创建了新 Catalog 时为 catalog.database.table）。
   例如：\
   (1) 如果源表 'my_source' 位于 'my_db' 中，请限定它：\
   \--source_table "my_db.my_source"\
   (2) sqls 的示例：\
   当 sqls 改变了当前 Catalog 和数据库时，可以不限定源表名：\
   \--source_sql "CREATE CATALOG my_cat WITH (...)"\
   \--source_sql "USE CATALOG my_cat"\
   \--source_sql "CREATE DATABASE my_db"\
   \--source_sql "USE my_db"\
   \--source_sql "CREATE TABLE S ..."\
   \--source_table S\
   但在以下情况下必须限定它：\
   \--source_sql "CREATE CATALOG my_cat WITH (...)"\
   \--source_sql "CREATE TABLE my_cat.\`default`.S ..."\
   \--source_table my_cat.default.S\
   在后续参数中你可以仅使用 'S' 作为源表名。
3. 至少必须指定一个 merge action。
4. 如果同时存在 matched-upsert 与 matched-delete 两个 action，那么它们的条件也必须同时存在
   （not-matched-by-source-upsert 与 not-matched-by-source-delete 同理）。否则，所有条件均为可选。
5. 所有条件、set 变更和值都应使用 Flink SQL 语法。为确保整个命令在 Shell 中正常运行，
   请用 \"\" 将它们引起来以转义空格，并在语句中使用 '\\' 来转义特殊字符。
   例如：\
   \--source_sql "CREATE TABLE T (k INT) WITH ('special-key' = '123\\!')"

:::

有关 'merge_into' 的更多信息，请参见

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    merge_into --help
```

## 从表中删除（Deleting from table） {#deleting-from-table}

在 Flink 1.16 及更早的版本中，Paimon 只支持通过 `flink run` 提交 'delete' 作业来删除记录。

运行以下命令为该表提交一个 'delete' 作业。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    delete \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <table-name> \
    --where <filter_spec> \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]]
    
filter_spec is equal to the 'WHERE' clause in SQL DELETE statement. Examples:
    age >= 18 AND age <= 60
    animal <> 'cat'
    id > (SELECT count(*) FROM employee)
```

有关 'delete' 的更多信息，请参见

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    delete --help
```

## 删除分区（Drop Partition） {#drop-partition}

运行以下命令为该表提交一个 'drop_partition' 作业。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    drop_partition \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <table-name> \
    [--partition <partition_spec> [--partition <partition_spec> ...]] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]]

partition_spec:
key1=value1,key2=value2...
```

有关 'drop_partition' 的更多信息，请参见

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    drop_partition --help
```

## 重写文件索引（Rewrite File Index） {#rewrite-file-index}

运行以下命令为该表提交一个 'rewrite_file_index' 作业。

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    rewrite_file_index \
    --warehouse <warehouse-path> \
    --identifier <database.table> \
    [--partitions <partition_spec>] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]]
```

有关 'rewrite_file_index' 的更多信息，请参见

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    rewrite_file_index --help
```

## 强制启动 Flink 作业（Force Start Flink Job） {#force-start-flink-job}

某些 Action（如 `create_tag`）较为轻量，默认不会作为作业提交到 Flink 集群。如果你
希望无论何种 Action 都能获得统一的体验，可以使用 `--force_start_flink_job` 标志来确保
将它们作为作业提交。例如，

```bash
<FLINK_HOME>/bin/flink run \
    /path/to/paimon-flink-action-@@VERSION@@.jar \
    drop_partition \
    --warehouse <warehouse-path> \
    --database <database-name> \
    --table <table-name> \
    [--partition <partition_spec> [--partition <partition_spec> ...]] \
    [--catalog_conf <paimon-catalog-conf> [--catalog_conf <paimon-catalog-conf> ...]]
    --force_start_flink_job true
```
