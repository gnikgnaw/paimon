---
title: "Chain Table"
sidebar_position: 9
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

# Chain Table {#chain-table}

Chain table（链式表）是主键表的一项新能力，它彻底改变了你处理增量数据的方式。
设想这样一个场景：你周期性地存储一份全量数据快照（例如每天一次），尽管两次快照之间只有
一小部分数据发生变化。ODS binlog dump 就是这种模式的典型示例。

以每天的 binlog dump 作业为例。一个批式作业将昨天的全量数据集与今天的增量变更合并，
生成一份新的全量数据集。这种做法有两个明显的缺点：
* 全量计算：合并操作涉及全部数据，并会引入 shuffle，从而导致性能不佳。
* 全量存储：每天都要存储一整套数据，而发生变化的数据通常只占很小的比例。

Paimon 通过直接只消费发生变化的数据并执行读时合并（Merge-on-Read）来解决这个问题。
这样一来，全量计算和存储就被转化为增量模式：
* 增量计算：离线 ETL 的每日作业只需消费当天发生变化的数据，无需合并全部数据。
* 增量存储：每天只存储发生变化的数据，并周期性地异步进行 Compaction（例如每周一次），以在生命周期内构建出全局的 chain table。
  ![](/img/chain-table.png)

在常规表的基础上，chain table 引入了 snapshot 分支和 delta 分支，分别表示全量数据和增量
数据。写入时，你指定写入全量数据或增量数据的分支。读取时，Paimon 会根据读取模式（如全量、
增量或混合）自动选择合适的策略。

要启用 chain table，你必须在建表时将表配置项 `chain-table.enabled` 设为 true，同时还需要
创建 snapshot 和 delta 分支。下面通过一个 Spark SQL 示例来说明：

```sql
CREATE TABLE default.t (
    `t1` string ,
    `t2` string ,
    `t3` string
) PARTITIONED BY (`date` string)
TBLPROPERTIES (
  'chain-table.enabled' = 'true',
  -- props about primary key table  
  'primary-key' = 'date,t1',
  'sequence.field' = 't2',
  'bucket-key' = 't1',
  'bucket' = '2',
  -- props about partition
  'partition.timestamp-pattern' = '$date', 
  'partition.timestamp-formatter' = 'yyyyMMdd'
);

CALL sys.create_branch('default.t', 'snapshot');

CALL sys.create_branch('default.t', 'delta');

ALTER TABLE default.t SET tblproperties 
    ('scan.fallback-snapshot-branch' = 'snapshot', 
     'scan.fallback-delta-branch' = 'delta');
 
ALTER TABLE `default`.`t$branch_snapshot` SET tblproperties
    ('scan.fallback-snapshot-branch' = 'snapshot',
     'scan.fallback-delta-branch' = 'delta');

ALTER TABLE `default`.`t$branch_delta` SET tblproperties 
    ('scan.fallback-snapshot-branch' = 'snapshot',
     'scan.fallback-delta-branch' = 'delta');
```

请注意：
- chain table 仅支持主键表，这意味着你需要为表定义 `bucket` 和 `bucket-key`。
- chain table 应确保每个分支的 schema（表结构）保持一致。
- 目前仅 Spark 支持，Flink 将在后续支持。
- 目前尚不支持 chain compact，将在后续支持。
- chain table 不支持删除向量（Deletion Vector）。

创建 chain table 之后，你可以通过以下方式读写数据。

- 全量写入（Full Write）：将数据写入 t$branch_snapshot。
```sql
insert overwrite `default`.`t$branch_snapshot` partition (date = '20250810') 
    values ('1', '1', '1'); 
```

- 增量写入（Incremental Write）：将数据写入 t$branch_delta。
```sql
insert overwrite `default`.`t$branch_delta` partition (date = '20250811') 
    values ('2', '1', '1');
```

- 全量查询（Full Query）：如果 snapshot 分支拥有全量分区，则直接读取该分区；否则以 chain merge 模式读取。
```sql
select t1, t2, t3 from default.t where date = '20250811'
```
你将得到如下结果：
```text
+---+----+-----+ 
| t1|  t2|   t3| 
+---+----+-----+ 
| 1 |   1|   1 |           
| 2 |   1|   1 |               
+---+----+-----+ 
```

- 增量查询（Incremental Query）：从 t$branch_delta 读取增量分区。
```sql
select t1, t2, t3 from `default`.`t$branch_delta` where date = '20250811'
```
你将得到如下结果：
```text
+---+----+-----+ 
| t1|  t2|   t3| 
+---+----+-----+      
| 2 |   1|   1 |               
+---+----+-----+ 
```

- 混合查询（Hybrid Query）：同时读取全量数据和增量数据。
```sql
select t1, t2, t3 from default.t where date = '20250811'
union all
select t1, t2, t3 from `default`.`t$branch_delta` where date = '20250811'
```
你将得到如下结果：
```text
+---+----+-----+ 
| t1|  t2|   t3| 
+---+----+-----+ 
| 1 |   1|   1 |           
| 2 |   1|   1 |  
| 2 |   1|   1 |               
+---+----+-----+ 
```

## Group Partition {#group-partition}

在真实场景中，一张表往往具有多个分区维度。例如，数据可能同时按 `region` 和 `date` 进行
分区。在这种情况下，不同的 region 是相互独立的数据孤岛——每个 region 都应当独立地维护
自己的链，而不是在所有 region 之间共享一条全局链。

Paimon 通过 **group partition（分组分区）** 支持这种模式：分区键被划分为两部分：
- **Group partition keys（分组分区键，前缀字段）**：用于标识独立数据孤岛的维度（例如 `region`）。
  每一组不同的分组分区取值组合都会构成自己独立的链。
- **Chain partition keys（链分区键，后缀字段）**：在一个分组内构成时间有序链的维度
  （例如 `date`）。

使用 `chain-table.chain-partition-keys` 来指定链维度。该值必须是表分区键的一段
**连续后缀**。位于它之前的分区字段会自动成为分组维度。如果未设置该配置项，则所有分区都
属于单一的隐式分组（即单维度分区表的默认行为）。

考虑这样一个示例：表按 `region` 和 `date` 进行分区，而你希望每个 region 都拥有
自己独立的链：

```sql
CREATE TABLE default.t (
    `t1` string ,
    `t2` string ,
    `t3` string
) PARTITIONED BY (`region` string, `date` string)
TBLPROPERTIES (
  'chain-table.enabled' = 'true',
  'primary-key' = 'region,date,t1',
  'sequence.field' = 't2',
  'bucket-key' = 't1',
  'bucket' = '2',
  'partition.timestamp-pattern' = '$date',
  'partition.timestamp-formatter' = 'yyyyMMdd',
  -- specify that only `date` is the chain dimension; `region` becomes the group dimension
  'chain-table.chain-partition-keys' = 'date'
);
```

在此配置下：
- 分区键：`[region, date]`
- 分组分区键：`[region]`——CN 和 US 各自拥有独立的链
- 链分区键：`[date]`——每个 region 内部的时间有序链

当读取诸如 `(region='CN', date='20250811')` 这样的分区时，Paimon 会在**同一个 region 内**
寻找最近的更早的 snapshot 分区（例如 `(region='CN', date='20250810')`）作为链锚点，
并仅针对 CN 这一分组沿着 delta 数据向前合并。US 分组则使用它自己的锚点独立地完成解析。

对于带有地域维度的按小时分区的表，你可以同时将 `dt` 和 `hour` 设置为链分区键：

```sql
'chain-table.chain-partition-keys' = 'dt,hour'
```

这会将 `(dt, hour)` 作为复合链维度，而将位于它之前的所有字段（例如 `region`）作为
分组维度。
