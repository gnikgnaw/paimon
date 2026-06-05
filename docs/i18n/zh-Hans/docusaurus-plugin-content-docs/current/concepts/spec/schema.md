---
title: "Schema"
sidebar_position: 2
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

# Schema {#schema}

schema（表结构）文件的版本号从 0 开始，目前会保留 schema 的所有历史版本。可能存在依赖旧 schema 版本的旧文件，因此删除时应当格外谨慎。

schema 文件为 JSON 格式，它包含：

1. fields：数据字段列表，每个数据字段包含 `id`、`name`、`type`，其中字段 id 用于支持 schema 演进。
2. partitionKeys：字段名列表，表的分区定义，不可修改。
3. primaryKeys：字段名列表，表的主键定义，不可修改。
4. options：map<string, string>，无序，表的配置项，包含大量能力与优化项。

## 示例 {#example}

```json
{
  "version" : 3,
  "id" : 0,
  "fields" : [ {
    "id" : 0,
    "name" : "order_id",
    "type" : "BIGINT NOT NULL"
  }, {
    "id" : 1,
    "name" : "order_name",
    "type" : "STRING"
  }, {
    "id" : 2,
    "name" : "order_user_id",
    "type" : "BIGINT"
  }, {
    "id" : 3,
    "name" : "order_shop_id",
    "type" : "BIGINT"
  } ],
  "highestFieldId" : 3,
  "partitionKeys" : [ ],
  "primaryKeys" : [ "order_id" ],
  "options" : {
    "bucket" : "5"
  },
  "comment" : "",
  "timeMillis" : 1720496663041
}
```

## 兼容性 {#compatibility}

对于旧版本：
- version 1：如果 options 中没有 `bucket` 键，应当向其中加入 `bucket -> 1`。
- version 1 与 2：如果 options 中没有 `file.format` 键，应当向其中加入 `file.format -> orc`。

## DataField {#datafield}

DataField 表示表的一列。

1. id：int 类型，列 id，自动递增，用于 schema 演进。
2. name：string 类型，列名。
3. type：数据类型，与 SQL 类型字符串非常相似。
4. description：string 类型。

## 更新 Schema {#update-schema}

更新 schema 应当生成一个新的 schema 文件。

```shell
warehouse
└── default.db
    └── my_table
        ├── schema
            ├── schema-0
            ├── schema-1
            └── schema-2
```

快照中存在对 schema 的引用。数值最大的 schema 文件通常就是最新的 schema 文件。

旧的 schema 文件不能被直接删除，因为可能仍有旧的数据文件引用着这些旧的 schema 文件。在读取表时，需要依赖它们来进行 schema 演进读取。
