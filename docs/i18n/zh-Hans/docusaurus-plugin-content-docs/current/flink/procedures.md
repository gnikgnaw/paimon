---
title: "Procedures"
sidebar_position: 97
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

# 存储过程（Procedures） {#procedures}

Flink 1.18 及更高版本支持 [Call 语句](https://nightlies.apache.org/flink/flink-docs-master/docs/sql/reference/utility/call/)，
它通过编写 SQL 而非提交 Flink 作业，使得操作 Paimon 表的数据和元数据更加便捷。

在 1.18 中，存储过程只支持按位置传递参数。你必须按顺序传入所有参数，如果某些参数不想传，必须使用 `''` 作为占位符。例如，如果你想以并行度 4 对表 `default.t`
执行 Compaction，但不想指定分区和排序策略，那么 Call 语句应写为 \
`CALL sys.compact('default.t', '', '', '', 'sink.parallelism=4')`。

在更高版本中，存储过程支持按名称传递参数。你可以以任意顺序传入参数，并且任何可选
参数都可以省略。对于上面的示例，Call 语句为 \
``CALL sys.compact(`table` => 'default.t', options => 'sink.parallelism=4')``。

指定分区：我们使用字符串表示分区过滤条件。"," 表示 "AND"，";" 表示 "OR"。例如，如果你想
指定两个分区 date=01 和 date=02，需要写成 'date=01;date=02'；如果你想指定一个分区
date=01 且 day=01，需要写成 'date=01,day=01'。

表配置项语法：我们使用字符串表示表配置项。格式为 'key1=value1,key2=value2...'。

下面列出了所有可用的存储过程。

<table class="table table-bordered">
   <thead>
   <tr>
      <th class="text-left" style="width: 4%">存储过程名称</th>
      <th class="text-left" style="width: 4%">用法</th>
      <th class="text-left" style="width: 20%">说明</th>
      <th class="text-left" style="width: 4%">示例</th>
   </tr>
   </thead>
   <tbody style="font-size: 11px; ">
   <tr>
      <td>compact</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.compact(
            `table` => 'table', 
            partitions => 'partitions', 
            order_strategy => 'order_strategy', 
            order_by => 'order_by', 
            options => 'options', 
            `where` => 'where', 
            partition_idle_time => 'partition_idle_time',
            compact_strategy => 'compact_strategy') <br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.compact('table') <br/>
         CALL [catalog.]sys.compact('table', 'partitions') <br/>
         CALL [catalog.]sys.compact('table', 'order_strategy', 'order_by') <br/>
         CALL [catalog.]sys.compact('table', 'partitions', 'order_strategy', 'order_by') <br/>
         CALL [catalog.]sys.compact('table', 'partitions', 'order_strategy', 'order_by', 'options') <br/>
         CALL [catalog.]sys.compact('table', 'partitions', 'order_strategy', 'order_by', 'options', 'where') <br/>
         CALL [catalog.]sys.compact('table', 'partitions', 'order_strategy', 'order_by', 'options', 'where', 'partition_idle_time') <br/>
         CALL [catalog.]sys.compact('table', 'partitions', 'order_strategy', 'order_by', 'options', 'where', 'partition_idle_time', 'compact_strategy') <br/><br/>
      </td>
      <td>
         对一张表执行 Compaction。参数：
            <li>table（必填）：目标表标识符。</li>
            <li>partitions（可选）：分区过滤条件。</li>
            <li>order_strategy（可选）：'order'、'zorder'、'hilbert' 或 'none'。</li>
            <li>order_by（可选）：需要排序的列。当 'order_strategy' 为 'none' 时留空。</li>
            <li>options（可选）：表的额外动态参数。其优先级高于原始 `tableProp`，低于 `procedureArg`。</li>
            <li>where（可选）：分区谓词（不能与 "partitions" 同时使用）。注意：由于 where 是关键字，需要在两侧加上一对反引号，写成 `where`。</li>
            <li>partition_idle_time（可选）：用于对在 'partition_idle_time' 时间内未接收到任何新数据的分区执行全量 Compaction，并且只有这些分区会被执行 Compaction。该参数不能与 order Compaction 一起使用。</li>
            <li>compact_strategy（可选）：决定如何挑选要合并的文件，默认值由运行时执行模式决定。'full' 策略只支持批模式，会选中所有文件进行合并；'minor' 策略：根据指定条件挑选需要合并的文件集合。</li>
      </td>
      <td>
         -- use partition filter <br/>
         CALL sys.compact(`table` => 'default.T', partitions => 'p=0', order_strategy => 'zorder', order_by => 'a,b', options => 'sink.parallelism=4') <br/><br/>
         -- use partition predicate <br/>
         CALL sys.compact(`table` => 'default.T', `where` => 'dt>10 and h<20', order_strategy => 'zorder', order_by => 'a,b', options => 'sink.parallelism=4')
      </td>
   </tr>
   <tr>
      <td>compact_database</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.compact_database(
            including_databases => 'includingDatabases', 
            mode => 'mode', 
            including_tables => 'includingTables', 
            excluding_tables => 'excludingTables', 
            table_options => 'tableOptions', 
            partition_idle_time => 'partitionIdleTime',
            compact_strategy => 'compact_strategy') <br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.compact_database() <br/>
         CALL [catalog.]sys.compact_database('includingDatabases') <br/>
         CALL [catalog.]sys.compact_database('includingDatabases', 'mode') <br/>
         CALL [catalog.]sys.compact_database('includingDatabases', 'mode', 'includingTables') <br/>
         CALL [catalog.]sys.compact_database('includingDatabases', 'mode', 'includingTables', 'excludingTables') <br/>
         CALL [catalog.]sys.compact_database('includingDatabases', 'mode', 'includingTables', 'excludingTables', 'tableOptions') <br/>
         CALL [catalog.]sys.compact_database('includingDatabases', 'mode', 'includingTables', 'excludingTables', 'tableOptions', 'partitionIdleTime')<br/>
         CALL [catalog.]sys.compact_database('includingDatabases', 'mode', 'includingTables', 'excludingTables', 'tableOptions', 'partitionIdleTime', 'compact_strategy')
      </td>
      <td>
         对多个数据库执行 Compaction。参数：
            <li>includingDatabases：用于指定数据库。可以使用正则表达式。</li>
            <li>mode：Compaction 模式。"divided"：为每张表启动一个 Sink，检测新表需要重启作业；
               "combined"（默认）：为所有表启动一个统一合并的 Sink，新表会被自动检测到。
            </li>
            <li>includingTables：用于指定表。可以使用正则表达式。</li>
            <li>excludingTables：用于指定不执行 Compaction 的表。可以使用正则表达式。</li>
            <li>tableOptions：表的额外动态参数。</li>
            <li>partition_idle_time：用于对在 'partition_idle_time' 时间内未接收到任何新数据的分区执行全量 Compaction，并且只有这些分区会被执行 Compaction。</li>
            <li>compact_strategy（可选）：决定如何挑选要合并的文件，默认值由运行时执行模式决定。'full' 策略只支持批模式，会选中所有文件进行合并；'minor' 策略：根据指定条件挑选需要合并的文件集合。</li>
      </td>
      <td>
         CALL sys.compact_database(
            including_databases => 'db1|db2', 
            mode => 'combined', 
            including_tables => 'table_.*', 
            excluding_tables => 'ignore', 
            table_options => 'sink.parallelism=4',
            compat_strategy => 'full')
      </td>
   </tr>
   <tr>
      <td>create_tag</td>
      <td>
         -- Use named argument<br/>
         -- based on the specified snapshot <br/>
         CALL [catalog.]sys.create_tag(`table` => 'identifier', tag => 'tagName', snapshot_id => snapshotId) <br/>
         -- based on the latest snapshot <br/>
         CALL [catalog.]sys.create_tag(`table` => 'identifier', tag => 'tagName') <br/><br/>
         -- Use indexed argument<br/>
         -- based on the specified snapshot <br/>
         CALL [catalog.]sys.create_tag('identifier', 'tagName', snapshotId) <br/>
         -- based on the latest snapshot <br/>
         CALL [catalog.]sys.create_tag('identifier', 'tagName')
      </td>
      <td>
         基于给定快照创建一个标签（Tag）。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tagName：新标签的名称。</li>
            <li>snapshotId (Long)：新标签所基于的快照 id。</li>
            <li>time_retained：新创建标签的最长保留时间。</li>
      </td>
      <td>
         CALL sys.create_tag(`table` => 'default.T', tag => 'my_tag', snapshot_id => cast(10 as bigint), time_retained => '1 d')
      </td>
   </tr>
   <tr>
      <td>create_tag_from_timestamp</td>
      <td>
         -- Create a tag from the first snapshot whose commit-time greater than the specified timestamp. <br/>
         -- Use named argument<br/>
         CALL [catalog.]sys.create_tag_from_timestamp(`table` => 'identifier', tag => 'tagName', timestamp => timestamp, time_retained => time_retained) <br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.create_tag_from_timestamp('identifier', 'tagName', timestamp, time_retained)
      </td>
      <td>
         基于给定时间戳创建一个标签。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tag：新标签的名称。</li>
            <li>timestamp (Long)：查找第一个 commit-time 大于该时间戳的快照。</li>
            <li>time_retained：新创建标签的最长保留时间。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.create_tag_from_timestamp('default.T', 'my_tag', 1724404318750, '1 d')<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.create_tag_from_timestamp(`table` => 'default.T', `tag` => 'my_tag', `timestamp` => 1724404318750, time_retained => '1 d')
      </td>
   </tr>
    <tr>
      <td>create_tag_from_watermark</td>
      <td>
         -- Create a tag from the first snapshot whose watermark greater than the specified timestamp.<br/>
         -- Use named argument<br/>
         CALL [catalog.]sys.create_tag_from_watermark(`table` => 'identifier', tag => 'tagName', watermark => watermark, time_retained => time_retained) <br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.create_tag_from_watermark('identifier', 'tagName', watermark, time_retained)
      </td>
      <td>
         基于给定的 Watermark 时间戳创建一个标签。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tag：新标签的名称。</li>
            <li>watermark (Long)：查找第一个 Watermark 大于指定 Watermark 的快照。</li>
            <li>time_retained：新创建标签的最长保留时间。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.create_tag_from_watermark('default.T', 'my_tag', 1724404318750, '1 d')<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.create_tag_from_watermark(`table` => 'default.T', `tag` => 'my_tag', `watermark` => 1724404318750, time_retained => '1 d')
      </td>
   </tr>
   <tr>
      <td>delete_tag</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.delete_tag(`table` => 'identifier', tag => 'tagName') <br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.delete_tag('identifier', 'tagName')
      </td>
      <td>
         删除一个标签。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tagName：要删除的标签名称。如果指定多个标签，分隔符为 ','。</li>
      </td>
      <td>
         CALL sys.delete_tag(`table` => 'default.T', tag => 'my_tag')
      </td>
   </tr>
   <tr>
      <td>replace_tag</td>
      <td>
         -- Use named argument<br/>
         -- replace tag with new time retained <br/>
         CALL [catalog.]sys.replace_tag(`table` => 'identifier', tag => 'tagName', time_retained => 'timeRetained') <br/>
         -- replace tag with new snapshot id and time retained <br/>
         CALL [catalog.]sys.replace_tag(`table` => 'identifier', snapshot_id => 'snapshotId') <br/><br/>
         -- Use indexed argument<br/>
         -- replace tag with new snapshot id and time retained <br/>
         CALL [catalog.]sys.replace_tag('identifier', 'tagName', 'snapshotId', 'timeRetained') <br/>
      </td>
      <td>
         用新的标签信息替换一个已存在的标签。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tag：已存在标签的名称。不能为空。</li>
            <li>snapshot(Long)：标签所基于的快照 id，可选。</li>
            <li>time_retained：已存在标签的最长保留时间，可选。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.replace_tag('default.T', 'my_tag', 5, '1 d')<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.replace_tag(`table` => 'default.T', tag => 'my_tag', snapshot_id => 5, time_retained => '1 d')<br/>
      </td>
   </tr>
   <tr>
      <td>expire_tags</td>
      <td>
         CALL [catalog.]sys.expire_tags('identifier', 'older_than')
      </td>
      <td>
         按时间使标签过期。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>older_than：在此 tagCreateTime 之前创建的标签将被移除。</li>
      </td>
      <td>
         CALL sys.expire_tags(table => 'default.T', older_than => '2024-09-06 11:00:00')
      </td>
   </tr>
   <tr>
      <td>trigger_tag_automatic_creation</td>
      <td>
         CALL [catalog.]sys.trigger_tag_automatic_creation('identifier')
      </td>
      <td>
         触发标签的自动创建。参数：
            <li>table：目标表标识符。不能为空。</li>
      </td>
      <td>
         CALL sys.trigger_tag_automatic_creation(table => 'default.T')
      </td>
   </tr>
   <tr>
      <td>merge_into</td>
      <td>
         -- for Flink 1.18<br/>
         CALL [catalog.]sys.merge_into('identifier','targetAlias',<br/>
            'sourceSqls','sourceTable','mergeCondition',<br/>
            'matchedUpsertCondition','matchedUpsertSetting',<br/>
            'notMatchedInsertCondition','notMatchedInsertValues',<br/>
            'matchedDeleteCondition')<br/><br/>
         -- for Flink 1.19 and later <br/>
         CALL [catalog.]sys.merge_into(<br/>
            target_table => 'identifier',<br/>
            target_alias => 'targetAlias',<br/>
            source_sqls => 'sourceSqls',<br/>
            source_table => 'sourceTable',<br/>
            merge_condition => 'mergeCondition',<br/>
            matched_upsert_condition => 'matchedUpsertCondition',<br/>
            matched_upsert_setting => 'matchedUpsertSetting',<br/>
            not_matched_insert_condition => 'notMatchedInsertCondition',<br/>
            not_matched_insert_values => 'notMatchedInsertValues',<br/>
            matched_delete_condition => 'matchedDeleteCondition',<br/>
            not_matched_by_source_upsert_condition => 'notMatchedBySourceUpsertCondition',<br/>
            not_matched_by_source_upsert_setting => 'notMatchedBySourceUpsertSetting',<br/>
            not_matched_by_source_delete_condition => 'notMatchedBySourceDeleteCondition') <br/><br/>
      </td>
      <td>
         执行 "MERGE INTO" 语法。参数详情请参见 <a href="/flink/action-jars#merging-into-table">merge_into action</a>。
      </td>
      <td>
         -- for matched order rows,<br/>
         -- increase the price,<br/>
         -- and if there is no match,<br/> 
         -- insert the order from<br/>
         -- the source table<br/>
         -- for Flink 1.18<br/>
         CALL sys.merge_into('default.T','','','default.S','T.id=S.order_id','','price=T.price+20','','*','')<br/><br/>
         -- for Flink 1.19 and later <br/>
         CALL sys.merge_into(<br/>
            target_table => 'default.T',<br/>
            source_table => 'default.S',<br/>
            merge_condition => 'T.id=S.order_id',<br/>
            matched_upsert_setting => 'price=T.price+20',<br/>
            not_matched_insert_values => '*')<br/><br/>
      </td>
   </tr>
   <tr>
      <td>data_evolution_merge_into</td>
      <td>
         -- Use indexed argument<br/>
         CALL [catalog].sys.data_evolution_merge_into('targetTable','targetAlias',<br/>
            'sourceSqls','sourceTable','mergeCondition','matchedUpdateSet',sinkParallelism)<br/><br/>
         -- Use named argument<br/>
         CALL [catalog].sys.data_evolution_merge_into(<br/>
            target_table => 'identifier',<br/>
            target_alias => 'targetAlias',<br/>
            source_sqls => 'sourceSqls',<br/>
            source_table => 'sourceTable',<br/>
            merge_condition => 'mergeCondition',<br/>
            matched_update_set => 'matchedUpdateSet',<br/>
            sink_parallelism => sinkParallelism) <br/><br/>
      </td>
      <td>执行专为数据演进（data-evolution）表实现的 "MERGE INTO" 语法。更多信息请参见 <a href="/docs/master/append-table/data-evolution/">data evolution</a>。</td>
      <td>
         -- for Flink 1.18<br/>
         CALL [catalog].sys.data_evolution_merge_into('default.T', '', '', 'S', 'T.id=S.id', 'name=S.name', 2) <br/><br/>
         -- for Flink 1.19 and later <br/>
         CALL [catalog].sys.data_evolution_merge_into(<br/>
            target_table => 'default.T',<br/>
            source_table => 'S',<br/>
            merge_condition => 'T.id=S.id',<br/>
            matched_update_set => 'name=S.name',<br/>
            sink_parallelism => 2)
      </td>
   </tr>
   <tr>
      <td>remove_orphan_files</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.remove_orphan_files(`table` => 'identifier', older_than => 'olderThan', dry_run => 'dryRun', mode => 'mode') <br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.remove_orphan_files('identifier')<br/>
         CALL [catalog.]sys.remove_orphan_files('identifier', 'olderThan')<br/>
         CALL [catalog.]sys.remove_orphan_files('identifier', 'olderThan', 'dryRun')<br/>
         CALL [catalog.]sys.remove_orphan_files('identifier', 'olderThan', 'dryRun','parallelism')<br/>
         CALL [catalog.]sys.remove_orphan_files('identifier', 'olderThan', 'dryRun','parallelism','mode')
      </td>
      <td>
         移除孤儿数据文件和元数据文件。参数：
            <li>table：目标表标识符。不能为空，你可以使用 database_name.* 来清理整个数据库。</li>
            <li>olderThan：为避免删除新写入的文件，该存储过程默认只删除早于 1 天的孤儿文件。
               此参数可以修改该时间间隔。
            </li>
            <li>dryRun：当为 true 时，只查看孤儿文件，并不实际删除文件。默认为 false。</li>
            <li>parallelism：并发删除文件的最大数量。默认值为 Java 虚拟机可用的处理器数量。</li>
            <li>mode：移除孤儿文件清理存储过程的模式（local 或 distributed）。默认为 distributed。</li>
      </td>
      <td>CALL sys.remove_orphan_files(`table` => 'default.T', older_than => '2023-10-31 12:00:00')<br/><br/>
          CALL sys.remove_orphan_files(`table` => 'default.*', older_than => '2023-10-31 12:00:00')<br/><br/>
          CALL sys.remove_orphan_files(`table` => 'default.T', older_than => '2023-10-31 12:00:00', dry_run => true)<br/><br/>
          CALL sys.remove_orphan_files(`table` => 'default.T', older_than => '2023-10-31 12:00:00', dry_run => false, parallelism => 5)<br/><br/>
          CALL sys.remove_orphan_files(`table` => 'default.T', older_than => '2023-10-31 12:00:00', dry_run => false, parallelism => 5, mode => 'local')
      </td>
   </tr>
   <tr>
      <td>remove_unexisting_files</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.remove_unexisting_files(`table` => 'identifier', dry_run => 'dryRun', parallelism => parallelism) <br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.remove_unexisting_files('identifier')<br/>
         CALL [catalog.]sys.remove_unexisting_files('identifier', 'dryRun', 'parallelism')
      </td>
      <td>
         从 manifest 条目中移除不存在的数据文件的存储过程。详细用例请参见 <a href="https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/RemoveUnexistingFilesAction.html">Java docs</a>。参数：
            <li>table：目标表标识符。不能为空，你可以使用 database_name.* 来清理整个数据库。</li>
            <li>dry_run（可选）：只检查哪些文件会被移除，但并不真正移除它们。默认为 false。</li>
            <li>parallelism（可选）：检查 manifest 中文件时的并行度。</li>
         <br>
         注意，使用此存储过程风险自负，在 Java docs 中列出的用例之外使用时，可能会导致数据丢失。
      </td>
      <td>
        -- remove unexisting data files in the table `mydb.myt`<br/>
        CALL sys.remove_unexisting_files(`table` => 'mydb.myt')<br/><br/>
        -- only check what files will be removed, but not really remove them (dry run)
        CALL sys.remove_unexisting_files(`table` => 'mydb.myt', `dry_run` = true)
      </td>
   </tr>
    <tr>
      <td>remove_unexisting_manifests</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.remove_unexisting_files(`table` => 'identifier') <br/><br/>
      </td>
      <td>
         从 manifest 列表中移除不存在的 manifest 文件的存储过程。详细用例如下。参数：
            <li>table：目标表标识符。不能为空，你可以使用 database.table$branch_xx 来移除分支表中不存在的 manifest 文件。</li>
         <br>
         注意，使用此存储过程风险自负，在 Java docs 中列出的用例之外使用时，可能会导致数据丢失。
      </td>
      <td>
        -- remove unexisting manifest file in the table `mydb.myt`<br/>
        CALL sys.remove_unexisting_manifests(`table` => 'mydb.myt')<br/><br/>
        -- remove unexisting manifest file in the branch table `mydb.myt$branch_rt`<br/>
        CALL sys.remove_unexisting_manifests(`table` => 'mydb.myt$branch_rt')
      </td>
   </tr>
   <tr>
      <td>reset_consumer</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.reset_consumer(`table` => 'identifier', consumer_id => 'consumerId', next_snapshot_id => 'nextSnapshotId') <br/><br/>
         -- Use indexed argument<br/>
         -- reset the new next snapshot id in the consumer<br/>
         CALL [catalog.]sys.reset_consumer('identifier', 'consumerId', nextSnapshotId)<br/><br/>
         -- delete consumer<br/>
         CALL [catalog.]sys.reset_consumer('identifier', 'consumerId')
      </td>
      <td>
         重置或删除消费者。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>consumerId：要重置或删除的消费者。</li>
            <li>nextSnapshotId (Long)：消费者新的 next snapshot id。</li>
      </td>
      <td>CALL sys.reset_consumer(`table` => 'default.T', consumer_id => 'myid', next_snapshot_id => cast(10 as bigint))</td>
   </tr>
   <tr>
      <td>clear_consumers</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.clear_consumers(`table` => 'identifier', including_consumers => 'includingConsumers', excluding_consumers => 'excludingConsumers') <br/><br/>
         -- Use indexed argument<br/>
         -- clear all consumers in the table<br/>
         CALL [catalog.]sys.clear_consumers('identifier')<br/><br/>
         -- clear some consumers in the table (accept regular expression)<br/>
         CALL [catalog.]sys.clear_consumers('identifier', 'includingConsumers')<br/><br/>
         -- exclude some consumers (accept regular expression)<br/>
         CALL [catalog.]sys.clear_consumers('identifier', 'includingConsumers', 'excludingConsumers')
      </td>
      <td>
         重置或删除消费者。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>includingConsumers：要清除的消费者。</li>
            <li>excludingConsumers：不被清除的消费者。</li>
      </td>
      <td>CALL sys.clear_consumers(`table` => 'default.T')<br/><br/>
          CALL sys.clear_consumers(`table` => 'default.T', including_consumers => 'myid.*')<br/><br/>
          CALL sys.clear_consumers(table => 'default.T', including_consumers => '', excluding_consumers => 'myid1.*')<br/><br/>
          CALL sys.clear_consumers(table => 'default.T', including_consumers => 'myid.*', excluding_consumers => 'myid1.*')
     </td>
   </tr>
   <tr>
      <td>rollback_to</td>
      <td>
         -- for Flink 1.18<br/>
         -- rollback to a snapshot<br/>
         CALL [catalog.]sys.rollback_to('identifier', snapshotId)<br/><br/>
         -- rollback to a tag<br/>
         CALL [catalog.]sys.rollback_to('identifier', 'tagName')<br/><br/>
         -- for Flink 1.19 and later<br/>
         -- rollback to a snapshot<br/>
         CALL [catalog.]sys.rollback_to(`table` => 'identifier', snapshot_id => snapshotId)<br/><br/>
         -- rollback to a tag<br/>
         CALL [catalog.]sys.rollback_to(`table` => 'identifier', tag => 'tagName')
      </td>
      <td>
         回滚到目标表的某个特定版本。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>snapshotId (Long)：将要回滚到的快照 id。</li>
            <li>tagName：将要回滚到的标签名称。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.rollback_to('default.T', 10)<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.rollback_to(`table` => 'default.T', snapshot_id => 10)
      </td>
   </tr>
   <tr>
      <td>rollback_to_timestamp</td>
      <td>
         -- for Flink 1.18<br/>
         -- rollback to the snapshot which earlier or equal than timestamp.<br/>
         CALL [catalog.]sys.rollback_to_timestamp('identifier', timestamp)<br/><br/>
         -- for Flink 1.19 and later<br/>
         -- rollback to the snapshot which earlier or equal than timestamp.<br/>
         CALL [catalog.]sys.rollback_to_timestamp(`table` => 'default.T', `timestamp` => timestamp)<br/><br/>
      </td>
      <td>
         回滚到时间戳早于或等于指定值的那个快照。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>timestamp (Long)：回滚到时间戳早于或等于该值的那个快照。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.rollback_to_timestamp('default.T', 10)<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.rollback_to_timestamp(`table` => 'default.T', timestamp => 1730292023000)
      </td>
   </tr>
   <tr>
          <td>rollback_to_watermark</td>
      <td>
         -- for Flink 1.18<br/>
         -- rollback to the snapshot which earlier or equal than watermark.<br/>
         CALL [catalog.]sys.rollback_to_watermark('identifier', watermark)<br/><br/>
         -- for Flink 1.19 and later<br/>
         -- rollback to the snapshot which earlier or equal than watermark.<br/>
         CALL [catalog.]sys.rollback_to_watermark(`table` => 'default.T', `watermark` => watermark)<br/><br/>
      </td>
      <td>
         回滚到 Watermark 早于或等于指定值的那个快照。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>watermark (Long)：回滚到 Watermark 早于或等于该值的那个快照。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.rollback_to_watermark('default.T', 1730292023000)<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.rollback_to_watermark(`table` => 'default.T', watermark => 1730292023000)
      </td>
   </tr>
   <tr>
          <td>purge_files</td>
      <td>
         -- clear table with purge files.<br/>
         CALL [catalog.]sys.purge_files('identifier')<br/>
      </td>
      <td>
         通过清除文件来清空表。参数：
            <li>table：目标表标识符。不能为空。</li>
      </td>
      <td>
         CALL sys.purge_files('default.T')<br/>
      </td>
   </tr>
   <tr>
      <td>migrate_database</td>
      <td>
         -- for Flink 1.18<br/>
         -- migrate all hive tables in database to paimon tables.<br/>
         CALL [catalog.]sys.migrate_database('connector', 'dbIdentifier', 'options'[, &ltparallelism&gt])<br/><br/>
         -- for Flink 1.19 and later<br/>
         -- migrate all hive tables in database to paimon tables.<br/>
         CALL [catalog.]sys.migrate_database(connector => 'connector', source_database => 'dbIdentifier', options => 'options'[, &ltparallelism => parallelism&gt])<br/><br/>
      </td>
      <td>
         将数据库中所有 Hive 表迁移为 Paimon 表。参数：
            <li>connector：要迁移的源数据库类型，例如 hive。不能为空。</li>
            <li>source_database：要迁移的源数据库名称。不能为空。</li>
            <li>options：迁移所得 Paimon 表的表配置项。</li>
            <li>parallelism：迁移过程的并行度，默认为机器的核心数。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.migrate_database('hive', 'db01', 'file.format=parquet', 6)<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.migrate_database(connector => 'hive', source_database => 'db01', options => 'file.format=parquet', parallelism => 6)
      </td>
   </tr>
   <tr>
      <td>migrate_table</td>
      <td>
         -- migrate hive table to a paimon table.<br/>
         CALL [catalog.]sys.migrate_table(connector => 'connector', source_table => 'tableIdentifier', options => 'options'[, &ltparallelism => parallelism&gt])<br/><br/>
      </td>
      <td>
         将一张 Hive 表迁移为 Paimon 表。参数：
            <li>connector：要迁移的源表类型，例如 hive。不能为空。</li>
            <li>source_table：要迁移的源表名称。不能为空。</li>
            <li>target_table：迁移所得目标 Paimon 表的名称。若未设置，则与源表保持相同名称。</li>
            <li>options：迁移所得 Paimon 表的表配置项。</li>
            <li>parallelism：迁移过程的并行度，默认为机器的核心数。</li>
            <li>delete_origin：如果设置了 target_table，可以设置 delete_origin 来决定迁移后是否从 hms 中删除源表元数据。默认为 true。</li>
      </td>
      <td>
         CALL sys.migrate_table(connector => 'hive', source_table => 'db01.t1', options => 'file.format=parquet', parallelism => 6)
      </td>
   </tr>
   <tr>
      <td>migrate_iceberg_table</td>
      <td>
         -- Use named argument<br/>
        CALL sys.migrate_iceberg_table(source_table => 'database_name.table_name', iceberg_options => 'iceberg_options', options => 'paimon_options', parallelism => parallelism);<br/><br/>
        -- Use indexed argument<br/>
        CALL sys.migrate_iceberg_table('source_table','iceberg_options', 'options', 'parallelism');
      </td>
      <td>
         将 Iceberg 表迁移到 Paimon。参数：
            <li>source_table：字符串类型，用于指定要迁移的源 Iceberg 表，必填。</li>
            <li>iceberg_options：字符串类型，用于指定迁移的配置，多个配置项以逗号分隔，必填。</li>
            <li>options：字符串类型，用于为目标 Paimon 表指定额外的配置项，可选。</li>
            <li>parallelism：整数类型，用于指定迁移作业的并行度，可选。</li>
      </td>
      <td>
         CALL sys.migrate_iceberg_table(source_table => 'iceberg_db.iceberg_tbl',iceberg_options => 'metadata.iceberg.storage=hadoop-catalog,iceberg_warehouse=/path/to/iceberg/warehouse');
      </td>
   </tr>
   <tr>
      <td>expire_snapshots</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.expire_snapshots(<br/>
            `table` => 'identifier', <br/>
            retain_max => 'retain_max', <br/>
            retain_min => 'retain_min', <br/>
            older_than => 'older_than', <br/>
            max_deletes => 'max_deletes', <br/>
            options => 'key1=value1,key2=value2') <br/><br/>
         -- Use indexed argument<br/>
         -- for Flink 1.18<br/>
         CALL [catalog.]sys.expire_snapshots(table, retain_max)<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL [catalog.]sys.expire_snapshots(table, retain_max, retain_min, older_than, max_deletes)<br/><br/>
      </td>
      <td>
         使快照过期。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>retain_max：要保留的已完成快照的最大数量。</li>
            <li>retain_min：要保留的已完成快照的最小数量。</li>
            <li>order_than：在此时间戳之前的快照将被移除。</li>
            <li>max_deletes：一次最多可以删除的快照数量。</li>
            <li>options：表的额外动态参数。其优先级高于原始 `tableProp`，低于 `procedureArg`。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.expire_snapshots('default.T', 2)<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.expire_snapshots(`table` => 'default.T', retain_max => 2)<br/>
         CALL sys.expire_snapshots(`table` => 'default.T', older_than => '2024-01-01 12:00:00')<br/>
         CALL sys.expire_snapshots(`table` => 'default.T', older_than => '2024-01-01 12:00:00', retain_min => 10)<br/>
         CALL sys.expire_snapshots(`table` => 'default.T', older_than => '2024-01-01 12:00:00', max_deletes => 10, options => 'snapshot.expire.limit=1')<br/>
      </td>
   </tr>
   <tr>
      <td>expire_changelogs</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.expire_changelogs(<br/>
            `table` => 'identifier', <br/>
            retain_max => 'retain_max', <br/>
            retain_min => 'retain_min', <br/>
            older_than => 'older_than', <br/>
            max_deletes => 'max_deletes') <br/>
            delete_all => 'delete_all') <br/><br/>
         -- Use indexed argument<br/>
         -- for Flink 1.18<br/>
         CALL [catalog.]sys.expire_changelogs(table, retain_max, retain_min, older_than, max_deletes)<br/>
         CALL [catalog.]sys.expire_changelogs(table, delete_all)<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL [catalog.]sys.expire_changelogs(table, retain_max, retain_min, older_than, max_deletes, delete_all)<br/><br/>
      </td>
      <td>
         使 changelog 过期。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>retain_max：要保留的已完成 changelog 的最大数量。</li>
            <li>retain_min：要保留的已完成 changelog 的最小数量。</li>
            <li>order_than：在此时间戳之前的 changelog 将被移除。</li>
            <li>max_deletes：一次最多可以删除的 changelog 数量。</li>
            <li>delete_all：是否删除所有独立存储的 changelog。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.expire_changelogs('default.T', 4, 2, '2024-01-01 12:00:00', 2)<br/>
         CALL sys.expire_changelogs('default.T', true)<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.expire_changelogs(`table` => 'default.T', retain_max => 2)<br/>
         CALL sys.expire_changelogs(`table` => 'default.T', older_than => '2024-01-01 12:00:00')<br/>
         CALL sys.expire_changelogs(`table` => 'default.T', older_than => '2024-01-01 12:00:00', retain_min => 10)<br/>
         CALL sys.expire_changelogs(`table` => 'default.T', older_than => '2024-01-01 12:00:00', max_deletes => 10)<br/>
         CALL sys.expire_changelogs(`table` => 'default.T', delete_all => true)<br/>
      </td>
   </tr>
<tr>
      <td>expire_partitions</td>
      <td>
         CALL [catalog.]sys.expire_partitions(table, expiration_time, timestamp_formatter, expire_strategy, options)<br/><br/>
      </td>
      <td>
         使分区过期。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>expiration_time：分区的过期时间间隔。如果分区的存活时间超过该值，则该分区将过期。分区时间从分区值中提取。</li>
            <li>timestamp_formatter：用于从字符串解析时间戳的格式化器。</li>
            <li>timestamp_pattern：用于从分区中获取时间戳的模式。</li>
            <li>expire_strategy：指定分区过期的过期策略，可能的取值：'values-time' 或 'update-time'，默认为 'values-time'。</li>
            <li>max_expires：限制过期分区的最大数量，可选。</li>
            <li>options：表的额外动态参数。其优先级高于原始 `tableProp`，低于 `procedureArg`。</li>
      </td>
      <td>
         -- for Flink 1.18<br/>
         CALL sys.expire_partitions('default.T', '1 d', 'yyyy-MM-dd', '$dt', 'values-time')<br/><br/>
         -- for Flink 1.19 and later<br/>
         CALL sys.expire_partitions(`table` => 'default.T', expiration_time => '1 d', timestamp_formatter => 'yyyy-MM-dd', expire_strategy => 'values-time')<br/>
         CALL sys.expire_partitions(`table` => 'default.T', expiration_time => '1 d', timestamp_formatter => 'yyyy-MM-dd HH:mm', timestamp_pattern => '$dt $hm', expire_strategy => 'values-time', options => 'partition.expiration-max-num=2')<br/><br/>
      </td>
   </tr>
    <tr>
      <td>repair</td>
      <td>
         -- repair all databases and tables in catalog<br/>
         CALL [catalog.]sys.repair()<br/><br/>
         -- repair all tables in a specific database<br/>
         CALL [catalog.]sys.repair('databaseName')<br/><br/>
         -- repair a table<br/>
         CALL [catalog.]sys.repair('databaseName.tableName')<br/><br/>
         -- repair database and table in a string if you specify multiple tags, delimiter is ','<br/>
         CALL [catalog.]sys.repair('databaseName01,database02.tableName01,database03')
      </td>
      <td>
         将信息从文件系统同步到 Metastore。参数：
            <li>empty：Catalog 中所有的数据库和表。</li>
            <li>databaseName：目标数据库名称。</li>
            <li>tableName：目标表标识符。</li>
      </td>
      <td>CALL sys.repair(`table` => 'test_db.T')</td>
   </tr>
    <tr>
      <td>rewrite_file_index</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.rewrite_file_index(&lt`table` => identifier&gt [, &ltpartitions => partitions&gt])<br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.rewrite_file_index(&ltidentifier&gt [, &ltpartitions&gt])<br/><br/>
      </td>
      <td>
         重写表的文件索引。参数：
            <li>table：&ltdatabaseName&gt.&lttableName&gt。</li>
            <li>partitions：特定分区。</li>
      </td>
      <td>
         -- rewrite the file index for the whole table<br/>
         CALL sys.rewrite_file_index(`table` => 'test_db.T')<br/><br/>
         -- rewrite the file index for the specified partition in the table<br/>
         CALL sys.rewrite_file_index(`table` => 'test_db.T', partitions => 'pt=a')<br/><br/>
     </td>
   <tr>
      <td>create_branch</td>
      <td>
         CALL [catalog.]sys.create_branch(`table` => 'identifier', branch => 'branchName', tag => 'tagName')<br/><br/>
      </td>
      <td>
         基于给定标签创建一个分支（Branch），或者只创建一个空分支。参数：
            <li>table：目标表标识符或分支标识符。不能为空。</li>
            <li>branch：新分支的名称。</li>
            <li>tag：新分支所基于的标签名称。</li>
            <li>ignoreIfExists：如果分支已存在则忽略，默认为 false。</li>
      </td>
      <td>
         CALL sys.create_branch(`table` => 'default.T', branch => 'branch1', tag => 'tag1')<br/>
         -- based on the specified branch's tag <br/>
         CALL sys.create_branch(`table` => 'default.T$branch_existBranchName', branch => 'branch1', tag => 'tag1')<br/><br/>
         CALL sys.create_branch(`table` => 'default.T', branch => 'branch1')<br/><br/>
      </td>
   </tr>
   <tr>
      <td>delete_branch</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.delete_branch(`table` => 'identifier', branch => 'branchName')<br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.delete_branch('identifier', 'branchName')
      </td>
      <td>
         删除一个分支。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>branchName：要删除的分支名称。如果指定多个分支，分隔符为 ','。</li>
      </td>
      <td>
         CALL sys.delete_branch(`table` => 'default.T', branch => 'branch1')
      </td>
   </tr>
   <tr>
      <td>rename_branch</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.rename_branch(`table` => 'identifier', from_branch => 'branchName', to_branch => 'newBranchName')<br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.rename_branch('identifier', 'branchName', 'newBranchName')
      </td>
      <td>
         重命名一个分支。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>from_branch：要重命名的分支名称。</li>
            <li>to_branch：分支的新名称。</li>
      </td>
      <td>
         CALL sys.rename_branch(`table` => 'default.T', from_branch => 'branch1', to_branch => 'branch2')
      </td>
   </tr>
   <tr>
      <td>fast_forward</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.fast_forward(`table` => 'identifier', branch => 'branchName')<br/><br/>
         -- Use indexed argument<br/>
         CALL [catalog.]sys.fast_forward('identifier', 'branchName')
      </td>
      <td>
         将一个分支 fast_forward 到主分支。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>branchName：要合并的分支名称。</li>
      </td>
      <td>
         CALL sys.fast_forward(`table` => 'default.T', branch => 'branch1')
      </td>
   </tr>
   <tr>
      <td>merge_branch</td>
      <td>
         -- Use named argument<br/>
         CALL [catalog.]sys.merge_branch(`table` => 'identifier', source_branch => 'sourceBranchName')<br/><br/>
         CALL [catalog.]sys.merge_branch(`table` => 'identifier', source_branch => 'sourceBranchName', target_branch => 'targetBranchName')
      </td>
      <td>
         将 Append 表的数据文件从源分支合并到目标分支。
         该表必须以 <code>'branch-merge.enabled' = 'true'</code> 创建。此配置项通过拒绝 Compaction 和 INSERT OVERWRITE 来强制保持纯追加的表历史，并且它与删除向量（Deletion Vector）不兼容。
         要求源分支与目标分支之间具有兼容的 schema 历史以及一致的 row-tracking 设置。参数：
            <li>table：表标识符。不能为空。</li>
            <li>source_branch：要从中合并的源分支名称。不能为空。</li>
            <li>target_branch（可选）：要合并到的目标分支名称。默认为 'main'。</li>
      </td>
      <td>
         CALL sys.merge_branch(`table` => 'default.T', source_branch => 'branch1')<br/><br/>
         CALL sys.merge_branch(`table` => 'default.T', source_branch => 'branch1', target_branch => 'branch2')
      </td>
   </tr>
   <tr>
      <td>compact_manifest</td>
      <td>
         CALL [catalog.]sys.compact_manifest(`table` => 'identifier')<br/>
         CALL [catalog.]sys.compact_manifest(`table` => 'identifier', 'options' => 'key1=value1,key2=value2')
      </td>
      <td>
         对 manifest 文件执行 compact_manifest。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>options：表的额外动态参数。其优先级高于原始 `tableProp`，低于 `procedureArg`。</li>
      </td>
      <td>
         CALL sys.compact_manifest(`table` => 'default.T')
      </td>
   </tr>
   <tr>
      <td>rescale</td>
      <td>
         CALL [catalog.]sys.rescale(`table` => 'identifier', `bucket_num` => bucket_num, `partition` => 'partition', `scan_parallelism` => scan_parallelism, `sink_parallelism` => sink_parallelism)
      </td>
      <td>
         调整一张表的某一个分区的桶数。参数：
         <li>table：目标表标识符。不能为空。</li>
         <li>bucket_num：调整桶数后产生的桶数量。参数 bucket_num 的默认值为表当前的桶数量。对于 postpone bucket 表不能为空。</li>
         <li>partition：要调整桶数的分区。对于分区表，该参数不能为空。</li>
         <li>scan_parallelism：Source 算子的并行度。默认值为该分区当前的桶数量。</li>
         <li>sink_parallelism：Sink 算子的并行度。默认值等于 bucket_num。</li>
      </td>
      <td>
         CALL sys.rescale(`table` => 'default.T', `bucket_num` => 16, `partition` => 'dt=20250217,hh=08')
      </td>
   </tr>
   <tr>
      <td>alter_view_dialect</td>
      <td>
         -- add dialect in the view<br/>
         CALL [catalog.]sys.alter_view_dialect('view_identifier', 'add', 'flink', 'query')<br/>
         CALL [catalog.]sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'add', `query` => 'query')<br/><br/>
         -- update dialect in the view<br/>
         CALL [catalog.]sys.alter_view_dialect('view_identifier', 'update', 'flink', 'query')<br/>
         CALL [catalog.]sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'update', `query` => 'query')<br/><br/>
         -- drop dialect in the view<br/>
         CALL [catalog.]sys.alter_view_dialect('view_identifier', 'drop', 'flink')<br/>
         CALL [catalog.]sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'drop')<br/><br/>
      </td>
      <td>
         修改视图方言（dialect）。参数：
            <li>view：目标视图标识符。不能为空。</li>
            <li>action：定义变更动作，例如：add、update、drop。不能为空。</li>
            <li>engine：当引擎不是 flink 时需要定义它。</li>
            <li>query：方言对应的查询，当 action 为 add 和 update 时不能为空。</li>
      </td>
      <td>
         -- add dialect in the view<br/>
         CALL sys.alter_view_dialect('view_identifier', 'add', 'flink', 'query')<br/>
         CALL sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'add', `query` => 'query')<br/><br/>
         -- update dialect in the view<br/>
         CALL sys.alter_view_dialect('view_identifier', 'update', 'flink', 'query')<br/>
         CALL sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'update', `query` => 'query')<br/><br/>
         -- drop dialect in the view<br/>
         CALL sys.alter_view_dialect('view_identifier', 'drop', 'flink')<br/>
         CALL sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'drop')<br/><br/>
      </td>
   </tr>
   <tr>
      <td>create_function</td>
      <td>
         CALL [catalog.]sys.create_function(<br/>
                'function_identifier',<br/>
                '[{"id": 0, "name":"length", "type":"INT"}, {"id": 1, "name":"width", "type":"INT"}]',<br/>
                '[{"id": 0, "name":"area", "type":"BIGINT"}]',<br/>
                true, 'comment', 'k1=v1,k2=v2')<br/>
      </td>
      <td>
         创建一个函数。参数：
            <li>function：目标函数标识符。不能为空。</li>
            <li>inputParams：函数的 inputParams。</li>
            <li>returnParams：函数的 returnParams。</li>
            <li>deterministic：函数是否是确定性的。</li>
            <li>comment：函数的注释。</li>
            <li>options：函数的额外动态参数。</li>
      </td>
      <td>
         CALL sys.create_function(`function` => 'function_identifier',<br/>
              inputParams => '[{"id": 0, "name":"length", "type":"INT"}, {"id": 1, "name":"width", "type":"INT"}]',<br/>
              returnParams => '[{"id": 0, "name":"area", "type":"BIGINT"}]',<br/>
              deterministic => true,<br/>
              comment => 'comment',<br/>
              options => 'k1=v1,k2=v2'<br/>
         )<br/>
      </td>
   </tr>
   <tr>
      <td>alter_function</td>
      <td>
         CALL [catalog.]sys.alter_function(<br/>
                'function_identifier',<br/>
                '{"action" : "addDefinition", "name" : "flink", "definition" : {"type" : "file", "fileResources" : [{"resourceType": "JAR", "uri": "oss://mybucket/xxxx.jar"}], "language": "JAVA", "className": "xxxx", "functionName": "functionName" } }')<br/>
      </td>
      <td>
         修改一个函数。参数：
            <li>function：目标函数标识符。不能为空。</li>
            <li>change：函数的变更。</li>
      </td>
      <td>
         CALL sys.alter_function(`function` => 'function_identifier',<br/>
              `change` => '{"action" : "addDefinition", "name" : "flink", "definition" : {"type" : "file", "fileResources" : [{"resourceType": "JAR", "uri": "oss://mybucket/xxxx.jar"}], "language": "JAVA", "className": "xxxx", "functionName": "functionName" } }'<br/>
         )<br/>
      </td>
   </tr>
   <tr>
      <td>drop_function</td>
      <td>
         CALL [catalog.]sys.drop_function('function_identifier')<br/>
      </td>
      <td>
         删除一个函数。参数：
            <li>function：目标函数标识符。不能为空。</li>
      </td>
      <td>
         CALL sys.drop_function(`function` => 'function_identifier')<br/>
      </td>
   </tr>
   <tr>
      <td>create_global_index</td>
      <td>
         CALL [catalog.]sys.create_global_index(<br/>
            `table` => 'table',<br/>
            `index_column` => 'columnName',<br/>
            `index_type` => 'indexType',<br/>
            `partitions` => 'partitions',<br/>
            `options` => 'key1=value1,key2=value2')<br/>
      </td>
      <td>
         在一张表上创建全局索引以加速查询。参数：
            <li>table（必填）：目标表标识符。</li>
            <li>index_column（必填）：要在其上构建索引的列名。</li>
            <li>index_type（必填）：全局索引的类型，支持的类型包括 'bitmap'、'btree'、'lumina'、'tantivy-fulltext'。</li>
            <li>partitions（可选）：用于选择性创建索引的分区过滤条件。</li>
            <li>options（可选）：创建索引时的额外动态参数。</li>
      </td>
      <td>
         -- Create bitmap index<br/>
         CALL sys.create_global_index(<br/>
            `table` => 'default.T',<br/>
            `index_column` => 'name',<br/>
            `index_type` => 'bitmap')<br/><br/>
         -- Create index for specific partitions<br/>
         CALL sys.create_global_index(<br/>
            `table` => 'default.T',<br/>
            `index_column` => 'name',<br/>
            `index_type` => 'bitmap',<br/>
            `partitions` => 'pt=p1;pt=p2')
      </td>
   </tr>
   <tr>
      <td>drop_global_index</td>
      <td>
         CALL [catalog.]sys.drop_global_index(<br/>
            `table` => 'table',<br/>
            `index_column` => 'columnName',<br/>
            `index_type` => 'indexType',<br/>
            `partitions` => 'partitions')<br/>
      </td>
      <td>
         从一张表中删除全局索引文件。参数：
            <li>table（必填）：目标表标识符。</li>
            <li>index_column（必填）：要删除索引的列名。</li>
            <li>index_type（必填）：要删除的全局索引类型，例如 'bitmap'、'btree'。</li>
            <li>partitions（可选）：用于选择性删除索引的分区规格。</li>
      </td>
      <td>
         -- Drop all bitmap indexes for column 'name'<br/>
         CALL sys.drop_global_index(<br/>
            `table` => 'default.T',<br/>
            `index_column` => 'name',<br/>
            `index_type` => 'bitmap')<br/><br/>
         -- Drop indexes only for specific partitions<br/>
         CALL sys.drop_global_index(<br/>
            `table` => 'default.T',<br/>
            `index_column` => 'name',<br/>
            `index_type` => 'bitmap',<br/>
            `partitions` => 'pt=p1;pt=p2')
      </td>
   </tr>
   <tr>
      <td>vector_search</td>
      <td>
         CALL [catalog.]sys.vector_search(<br/>
            `table` => 'identifier',<br/>
            vector_column => 'columnName',<br/>
            query_vector => 'v1,v2,...',<br/>
            top_k => topK,<br/>
            projection => 'col1,col2',<br/>
            options => 'key1=value1;key2=value2')<br/>
      </td>
      <td>
         在带有全局向量索引的表上执行向量相似度检索。返回 JSON 序列化的行。参数：
            <li>table（必填）：目标表标识符。</li>
            <li>vector_column（必填）：要检索的向量列名称。</li>
            <li>query_vector（必填）：以逗号分隔的浮点值，表示查询向量，例如 '1.0,2.0,3.0'。</li>
            <li>top_k（必填）：要返回的最近邻数量。</li>
            <li>projection（可选）：以逗号分隔的列名，用于包含在结果中。如果省略，则返回所有列。</li>
            <li>options（可选）：表的额外动态参数。</li>
      </td>
      <td>
         CALL sys.vector_search(<br/>
            `table` => 'default.T',<br/>
            vector_column => 'embedding',<br/>
            query_vector => '1.0,2.0,3.0',<br/>
            top_k => 5)<br/><br/>
         CALL sys.vector_search(<br/>
            `table` => 'default.T',<br/>
            vector_column => 'embedding',<br/>
            query_vector => '1.0,2.0,3.0',<br/>
            top_k => 5,<br/>
            projection => 'id,name')
      </td>
   </tr>
   </tbody>
</table>
