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

# 概述 {#overview}

当 Paimon Sink 接收到两条或更多具有相同主键的记录时，它会将它们合并为一条记录，以保持主键唯一。通过指定 `merge-engine` 表属性，用户可以选择记录的合并方式。

:::info

在 Flink SQL 的 TableConfig 中，请始终将 `table.exec.sink.upsert-materialize` 设置为 `NONE`，Sink 的 upsert-materialize 可能会导致奇怪的行为。当输入乱序时，我们建议你使用[序列字段（sequence field）](../sequence-rowkind#sequence-field)来纠正乱序。

:::

## 去重（Deduplicate） {#deduplicate}

`deduplicate` 合并引擎（merge engine）是默认的合并引擎。Paimon 只会保留最新的记录，并丢弃其他具有相同主键的记录。

具体来说，如果最新的记录是一条 `DELETE` 记录，则所有具有相同主键的记录都会被删除。你可以配置 `ignore-delete` 来忽略它。
