---
title: "概述"
sidebar_position: 1
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

# 规格概述 {#spec-overview}

这是 Paimon 表格式的规格说明，本文档对 Paimon 的底层文件结构与设计进行了标准化定义。

![](/img/file-layout.png)

## 术语 {#terms}

- Schema：字段、主键定义、分区键定义以及配置项。
- Snapshot（快照）：在某个特定时间点提交的所有数据的入口。
- Manifest 列表：包含若干个 manifest（清单）文件。
- Manifest（清单）文件：包含若干个数据文件或 changelog（变更日志）文件。
- 数据文件：包含增量记录。
- Changelog 文件：包含由 changelog producer 产生的记录。
- 全局索引：针对某个桶或分区的索引。
- 数据文件索引：针对某个数据文件的索引。

使用 Paimon 运行 Flink SQL：

```sql
CREATE CATALOG my_catalog WITH (
    'type' = 'paimon',
    'warehouse' = '/your/path'
);       
USE CATALOG my_catalog;

CREATE TABLE my_table (
    k INT PRIMARY KEY NOT ENFORCED,
    f0 INT,
    f1 STRING
);

INSERT INTO my_table VALUES (1, 11, '111');
```

查看磁盘上的内容：

```shell
warehouse
└── default.db
    └── my_table
        ├── bucket-0
        │   └── data-59f60cb9-44af-48cc-b5ad-59e85c663c8f-0.orc
        ├── index
        │   └── index-5625e6d9-dd44-403b-a738-2b6ea92e20f1-0
        ├── manifest
        │   ├── index-manifest-5d670043-da25-4265-9a26-e31affc98039-0
        │   ├── manifest-6758823b-2010-4d06-aef0-3b1b597723d6-0
        │   ├── manifest-list-9f856d52-5b33-4c10-8933-a0eddfaa25bf-0
        │   └── manifest-list-9f856d52-5b33-4c10-8933-a0eddfaa25bf-1
        ├── schema
        │   └── schema-0
        └── snapshot
            ├── EARLIEST
            ├── LATEST
            └── snapshot-1
```
