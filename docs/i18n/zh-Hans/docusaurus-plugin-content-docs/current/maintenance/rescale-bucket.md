---
title: "Rescale Bucket"
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

# 调整桶数（Rescale Bucket） {#rescale-bucket}

由于桶（bucket）的总数会显著影响性能，Paimon 允许用户通过 `ALTER TABLE` 命令调整桶数，并通过 `INSERT OVERWRITE`
重新组织数据布局，而无需重建表/分区。在执行 overwrite 作业时，框架会自动按旧的桶数扫描数据，并依据当前的桶数对记录进行哈希。

## Rescale Overwrite {#rescale-overwrite}
```sql
-- rescale number of total buckets
ALTER TABLE table_identifier SET ('bucket' = '...');

-- reorganize data layout of table/partition
INSERT OVERWRITE table_identifier [PARTITION (part_spec)]
SELECT ... 
FROM table_identifier
[WHERE part_spec];
``` 

请注意
- `ALTER TABLE` 只会修改表的元数据，**不会**重新组织或重新格式化已有的数据。
  重新组织已有数据必须通过 `INSERT OVERWRITE` 来实现。
- 调整桶数不会影响读取以及正在运行的写入作业。
- 一旦桶数发生变更，任何新调度的、写入尚未重新组织的已有表/分区的 `INSERT INTO` 作业都会抛出 `TableException`，消息形如
  ```text
  Try to write table/partition ... with a new bucket num ..., 
  but the previous bucket num is ... Please switch to batch mode, 
  and perform INSERT OVERWRITE to rescale current data layout first.
  ```
- 对于分区表，不同分区可以拥有不同的桶数。*例如*
  ```sql
  ALTER TABLE my_table SET ('bucket' = '4');
  INSERT OVERWRITE my_table PARTITION (dt = '2022-01-01')
  SELECT * FROM ...;
  
  ALTER TABLE my_table SET ('bucket' = '8');
  INSERT OVERWRITE my_table PARTITION (dt = '2022-01-02')
  SELECT * FROM ...;
  ```
- 在 overwrite 期间，请确保没有其他作业在写入同一张表/分区。

## Use Case {#use-case}

调整桶数有助于应对吞吐量的突发峰值。假设有一个每日运行的流式 ETL 任务，用于同步交易数据。该表的 DDL 和数据流水线
如下所示。

```sql
-- table DDL
CREATE TABLE verified_orders (
    trade_order_id BIGINT,
    item_id BIGINT,
    item_price DOUBLE,
    dt STRING,
    PRIMARY KEY (dt, trade_order_id, item_id) NOT ENFORCED 
) PARTITIONED BY (dt)
WITH (
    'bucket' = '16'
);

-- like from a kafka table 
CREATE temporary TABLE raw_orders(
    trade_order_id BIGINT,
    item_id BIGINT,
    item_price BIGINT,
    gmt_create STRING,
    order_status STRING
) WITH (
    'connector' = 'kafka',
    'topic' = '...',
    'properties.bootstrap.servers' = '...',
    'format' = 'csv'
    ...
);

-- streaming insert as bucket num = 16
INSERT INTO verified_orders
SELECT trade_order_id,
       item_id,
       item_price,
       DATE_FORMAT(gmt_create, 'yyyy-MM-dd') AS dt
FROM raw_orders
WHERE order_status = 'verified';
```
该流水线在过去几周里一直运行良好。然而，最近数据量快速增长，作业的延迟也持续上升。为了提升数据新鲜度，用户可以
- 使用 Savepoint 挂起流式作业（参见
  [Suspended State](https://nightlies.apache.org/flink/flink-docs-stable/docs/internals/job_scheduling/) 与
  [Stopping a Job Gracefully Creating a Final Savepoint](https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/cli/#terminating-a-job)）
  ```bash
  $ ./bin/flink stop \
        --savepointPath /tmp/flink-savepoints \
        $JOB_ID
   ```
- 增大桶数
  ```sql
  -- scaling out
  ALTER TABLE verified_orders SET ('bucket' = '32');
  ```
- 切换到批式模式，并对流式作业正在写入的当前分区执行 overwrite
  ```sql
  SET 'execution.runtime-mode' = 'batch';
  -- suppose today is 2022-06-22
  -- case 1: there is no late event which updates the historical partitions, thus overwrite today's partition is enough
  INSERT OVERWRITE verified_orders PARTITION (dt = '2022-06-22')
  SELECT trade_order_id,
         item_id,
         item_price
  FROM verified_orders
  WHERE dt = '2022-06-22';
  
  -- case 2: there are late events updating the historical partitions, but the range does not exceed 3 days
  INSERT OVERWRITE verified_orders
  SELECT trade_order_id,
         item_id,
         item_price,
         dt
  FROM verified_orders
  WHERE dt IN ('2022-06-20', '2022-06-21', '2022-06-22');
  ```
- 在 overwrite 作业完成后，切换回流式模式。此时，可以在增大桶数的同时增大并行度，从而从 Savepoint 恢复流式作业
（参见 [Start a SQL Job from a savepoint](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/table/sqlclient/#start-a-sql-job-from-a-savepoint)）
  ```sql
  SET 'execution.runtime-mode' = 'streaming';
  SET 'execution.savepoint.path' = <savepointPath>;

  INSERT INTO verified_orders
  SELECT trade_order_id,
       item_id,
       item_price,
       DATE_FORMAT(gmt_create, 'yyyy-MM-dd') AS dt
  FROM raw_orders
  WHERE order_status = 'verified';
  ```
