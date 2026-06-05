---
title: "管理标签"
sidebar_position: 3
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

# 管理标签 {#manage-tags}

与 Paimon 的 Java API 一样，你可以基于某个快照（snapshot）创建[标签（Tag）](../maintenance/manage-tags)。该标签会保留对应快照的 manifest（清单）文件和数据文件。
一个典型的用法是每天创建标签，这样你就可以保留每一天的历史数据以供批式读取。
## 创建与删除标签 {#create-and-delete-tag}

你可以使用给定的名称和快照 ID 创建一个标签，也可以使用给定的名称删除一个标签。

```python

table = catalog.get_table('database_name.table_name')
table.create_tag("tag2", snapshot_id=2)  # create tag2 based on snapshot 2
table.create_tag("tag2")  # create tag2 based on latest snapshot
table.delete_tag("tag2")  # delete tag2
```

如果未设置 snapshot_id，则 snapshot_id 默认为最新快照。

## 重命名标签 {#rename-tag}

你可以将一个标签重命名为新的名称。

```python

table = catalog.get_table('database_name.table_name')
table.rename_tag("old_tag", "new_tag")  # rename old_tag to new_tag
```

## 读取标签 {#read-tag}
你可以从指定的标签读取数据。
```python

table = catalog.get_table('database_name.table_name')
table.create_tag("tag2", snapshot_id=2)

# Read from tag2 using scan.tag-name option
table_with_tag = table.copy({"scan.tag-name": "tag2"})
read_builder = table_with_tag.new_read_builder()
table_scan = read_builder.new_scan()
table_read = read_builder.new_read()
result = table_read.to_arrow(table_scan.plan().splits())
```


