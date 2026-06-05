---
title: "SQL Upsert"
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

# SQL Upsert {#sql-upsert}

对于没有主键的表，Paimon 支持 Upsert 写入模式：如果具有相同 Upsert 键的行已存在，则执行更新；否则执行插入。

## 用法 {#usage}

在创建表时指定以下表属性

* `upsert-key`：定义用于 Upsert 的键列，不能与主键（primary key）一起使用。
与主键不同，Upsert 键的值可以为 `null`，并且支持 null 相等匹配。
多个列以逗号分隔。

* `sequence.field`（可选）：当新记录共享相同的 Upsert 键时，保留 `sequence.field` 值较大的那一行作为合并结果。
同时它还会对写入的数据进行去重。
如果未设置 `sequence.field`，共享相同 Upsert 键的新记录只会直接更新已有记录，不执行去重。
多个列以逗号分隔。

## 示例 {#example}

创建表：

```sql
CREATE TABLE t (k1 INT, k2 INT, ts1 INT, ts2 INT, v STRING)
TBLPROPERTIES ('upsert-key' = 'k1,k2', 'sequence.field' = 'ts1,ts2')
```

插入 data1：

```sql
INSERT INTO t values
(null, null, 2, 1, 'v1'),
(null, null, 2, 2, 'v4'),
(1, null, 1, 1, 'v1'),
(1, 2, 1, 1, 'v1'),
(1, 2, 2, 1, 'v2')
```

查询结果：

```sql
SELECT * FROM t ORDER BY k1, k2

-- null, null, 2, 2, "v4"
-- 1, null, 1, 1, "v1"
-- 1, 2, 2, 1, "v2"
```

插入 data2：

```sql
INSERT INTO t values
(null, null, 2, 1, 'v5'),
(null, 1, 1, 1, 'v1'),
(1, null, 2, 1, 'v2'),
(1, 1, 1, 1, 'v1'),
(1, 2, 2, 0, 'v3')
```

查询结果：

```sql
SELECT * FROM t ORDER BY k1, k2

-- null, null, 2, 2, "v4"
-- null, 1, 1, 1, "v1"
-- 1, null, 2, 1, "v2"
-- 1, 1, 1, 1, "v1"
-- 1, 2, 2, 1, "v2"
```
