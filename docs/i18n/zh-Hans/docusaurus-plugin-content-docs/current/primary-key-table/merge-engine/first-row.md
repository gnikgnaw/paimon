---
title: "First Row"
sidebar_position: 4
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

# First Row {#first-row}

通过指定 `'merge-engine' = 'first-row'`，用户可以保留相同主键的第一行。它与 `deduplicate` 合并引擎（merge engine）的不同之处在于，在 `first-row` 合并引擎中，它只会生成 insert only 的 changelog。

:::info

`first-row` 合并引擎仅支持 `none` 和 `lookup` 这两种 changelog producer。
对于流式查询，必须与 `lookup` [changelog producer](../changelog-producer) 配合使用。

:::

:::info

1. 你不能指定 [sequence.field](../sequence-rowkind#sequence-field)。
2. 不接受 `DELETE` 和 `UPDATE_BEFORE` 消息。你可以配置 `ignore-delete` 来忽略这两类记录。
3. 可见性保证：对于使用 First Row 引擎的表，level 0 层的文件只有在 Compaction 之后才会可见。
   因此默认情况下，Compaction 是同步的；如果开启了异步方式，数据可能会有延迟。

:::

这在流式计算中替换日志去重（log deduplication）方面有很大帮助。
