---
title: "Snapshot"
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

# 快照（Snapshot） {#snapshot}

每次提交都会生成一个快照（snapshot）文件，快照文件的版本号从 1 开始，并且必须是连续的。
`EARLIEST` 和 `LATEST` 是位于快照列表开头和结尾的提示（hint）文件，它们可能并不准确。
当提示文件不准确时，读取操作会扫描所有快照文件来确定开头和结尾。

```shell
warehouse
└── default.db
    └── my_table
        ├── snapshot
            ├── EARLIEST
            ├── LATEST
            ├── snapshot-1
            ├── snapshot-2
            └── snapshot-3
```

写入提交会抢占下一个快照 id，一旦快照文件成功写入，该提交即变为可见。

快照文件采用 JSON 格式，它包含：

1. version：快照文件版本，当前为 3。
2. id：快照 id，与文件名相同。
3. schemaId：本次提交对应的 schema（表结构）版本。
4. baseManifestList：一个 manifest 列表，记录此前所有快照的全部变更。
5. deltaManifestList：一个 manifest 列表，记录本快照中发生的所有新增变更。
6. changelogManifestList：一个 manifest 列表，记录本快照中产生的所有 changelog（变更日志），如果未产生 changelog 则为 null。
7. indexManifest：一个 manifest（清单）文件，记录本表的所有索引文件，如果表没有索引文件则为 null。
8. commitUser：通常由 UUID 生成，用于流式写入的恢复，一个流式写入作业对应一个用户。
9. commitIdentifier：流式写入对应的事务 id，每个事务可能因不同的 commitKind 产生多个提交。
10. commitKind：本快照中的变更类型，包括 append、compact、overwrite 和 analyze。
11. timeMillis：提交时间（毫秒）。
12. logOffsets：提交日志偏移量。
13. totalRecordCount：本快照中发生的所有变更的记录数。
14. deltaRecordCount：本快照中发生的所有新增变更的记录数。
15. changelogRecordCount：本快照中产生的所有 changelog 的记录数。
16. watermark：输入记录的 Watermark，来自 Flink 的 Watermark 机制，如果没有 Watermark 则为 Long.MIN_VALUE。
17. statistics：本表统计信息的 stats 文件名。
