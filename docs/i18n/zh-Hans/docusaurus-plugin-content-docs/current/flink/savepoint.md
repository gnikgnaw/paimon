---
title: "Savepoint"
sidebar_position: 99
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

# Savepoint {#savepoint}

Paimon 拥有自己的快照（snapshot）管理机制，这可能与 Flink 的 checkpoint 管理产生冲突，导致从 Savepoint 恢复时出现异常（不必担心，这不会导致存储被破坏）。

建议你使用以下方法来执行 Savepoint：

1. 使用 Flink 的 [Stop with savepoint](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/savepoints/#stopping-a-job-with-savepoint)。
2. 将 Paimon 标签（Tag）与 Flink Savepoint 结合使用，并在从 Savepoint 恢复前先回滚到对应标签（rollback-to-tag）。

## Stop with savepoint {#stop-with-savepoint}

Flink 的这一特性可确保最后一个 checkpoint 被完整处理，这意味着不会残留任何未提交的元数据。该方式非常安全，因此我们推荐使用此特性来停止和启动作业。

## Tag with Savepoint {#tag-with-savepoint}

在 Flink 中，我们可能从 Kafka 消费数据，然后写入 Paimon。由于 Flink 的 checkpoint 只保留有限的数量，我们会在特定时刻（例如代码升级、数据更新等）触发一次 Savepoint，以确保状态能够被保留更长时间，从而使作业能够进行增量恢复。

Paimon 的快照与 Flink 的 checkpoint 类似，二者都会自动过期，但 Paimon 的标签特性允许快照被长期保留。因此，我们可以将 Paimon 的标签与 Flink 的 Savepoint 这两项特性结合起来，实现从指定 Savepoint 对作业进行增量恢复。

**第 1 步：启用为 Savepoint 自动创建标签。**

你可以将 `sink.savepoint.auto-tag` 设置为 `true`，以启用为 Savepoint 自动创建标签的特性。

**第 2 步：触发 Savepoint。**

你可以参考 [Flink savepoint](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/savepoints/#operations)，了解如何配置和触发 Savepoint。

**第 3 步：选择与该 Savepoint 对应的标签。**

与 Savepoint 对应的标签将以 `savepoint-${savepointID}` 的形式命名。你可以参考 [Tags Table](../concepts/system-tables#tags-table) 进行查询。

**第 4 步：回滚 Paimon 表。**

将 Paimon 表[回滚](../maintenance/manage-tags#rollback-to-tag)到指定的标签。

**第 5 步：从 Savepoint 重启。**

你可以参考[这里](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/savepoints/#resuming-from-savepoints)，了解如何从指定的 Savepoint 重启。
