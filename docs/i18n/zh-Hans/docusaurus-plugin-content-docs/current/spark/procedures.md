---
title: "Procedures"
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

# 存储过程（Procedures） {#procedures}

本节介绍 Paimon 在 Spark 中提供的所有可用存储过程（Procedure）。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 4%">存储过程名称</th>
      <th class="text-left" style="width: 20%">说明</th>
      <th class="text-left" style="width: 4%">示例</th>
    </tr>
    </thead>
    <tbody style="font-size: 12px; ">
    <tr>
      <td>compact</td>
      <td>
         对文件执行 Compaction。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>partitions：分区过滤条件。逗号（","）表示「AND」，分号（";"）表示「OR」。如果你想对 date=01 且 day=01 的某个分区执行 Compaction，需要写成 'date=01,day=01'。留空表示所有分区。（不能与 "where" 同时使用）</li>
            <li>where：分区谓词。留空表示所有分区。（不能与 "partitions" 同时使用）</li>          
            <li>order_strategy：'order' 或 'zorder' 或 'hilbert' 或 'none'。留空表示 'none'。</li>
            <li>order_by：需要排序的列。如果 'order_strategy' 为 'none'，则留空。</li>
            <li>options：表的额外动态参数。其优先级高于原始的 `tableProp`，低于 `procedureArg`。</li>
            <li>partition_idle_time：用于对在 'partition_idle_time' 时间内没有收到任何新数据的分区执行全量 Compaction。并且只有这些分区会被执行 Compaction。该参数不能与 order compact 一起使用。</li>
            <li>compact_strategy：决定如何选取要合并的文件，默认值由运行时的执行模式决定。'full' 策略仅支持批式模式，所有文件都会被选中进行合并。'minor' 策略：根据指定条件选取需要合并的文件集合。</li>
      </td>
      <td>
         SET spark.sql.shuffle.partitions=10; --设置 compact 的并行度 <br/><br/>
         CALL sys.compact(table => 'T', partitions => 'p=0;p=1',  order_strategy => 'zorder', order_by => 'a,b') <br/><br/>
         CALL sys.compact(table => 'T', where => 'p>0 and p<3', order_strategy => 'zorder', order_by => 'a,b') <br/><br/>
         CALL sys.compact(table => 'T', where => 'dt>10 and h<20', order_strategy => 'zorder', order_by => 'a,b', options => 'target-file-size=128m')<br/><br/> 
         CALL sys.compact(table => 'T', partition_idle_time => '60s')<br/><br/>
         CALL sys.compact(table => 'T', compact_strategy => 'minor')<br/><br/>
      </td>
    </tr>
    <tr>
      <td>compact_database</td>
      <td>
         对一个或多个数据库中的所有表执行 Compaction。参数：
            <li>including_databases：用于匹配要执行 Compaction 的数据库的正则表达式。留空表示匹配所有数据库（即 '.*'）。</li>
            <li>including_tables：用于匹配要执行 Compaction 的表标识符（'db.table' 形式）的正则表达式。留空表示匹配所有表（即 '.*'）。</li>
            <li>excluding_tables：用于匹配要从 Compaction 中排除的表标识符的正则表达式。</li>
            <li>options：表的额外动态参数。其优先级高于原始的 `tableProp`，低于 `procedureArg`。</li>
      </td>
      <td>
         -- 对所有数据库执行 Compaction<br/>
         CALL sys.compact_database()<br/><br/>
         -- 对部分数据库执行 Compaction（接受正则表达式）<br/>
         CALL sys.compact_database(including_databases => 'db1|db2')<br/><br/>
         -- 对部分表执行 Compaction（接受正则表达式）<br/>
         CALL sys.compact_database(including_databases => 'db1', including_tables => 'db1.table1|db1.table2')<br/><br/>
         -- 排除部分表（接受正则表达式）<br/>
         CALL sys.compact_database(including_databases => 'db1', including_tables => '.*', excluding_tables => '.*ignore_table')<br/><br/>
         -- 设置表配置项<br/>
         CALL sys.compact_database(including_databases => 'db1', options => 'target-file-size=128m')
      </td>
    </tr>
    <tr>
      <td>expire_snapshots</td>
      <td>
         使快照过期。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>retain_max：要保留的已完成快照的最大数量。</li>
            <li>retain_min：要保留的已完成快照的最小数量。</li>
            <li>older_than：在此时间戳之前的快照将被移除。</li>
            <li>max_deletes：一次最多可删除的快照数量。</li>
            <li>options：表的额外动态参数。其优先级高于原始的 `tableProp`，低于 `procedureArg`。</li>
      </td>
      <td>CALL sys.expire_snapshots(table => 'default.T', retain_max => 10, options => 'snapshot.expire.limit=1')</td>
    </tr>
    <tr>
      <td>expire_partitions</td>
      <td>
         使分区过期。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>expiration_time：分区的过期间隔。如果某个分区的存活时间超过此值，则该分区将过期。分区时间从分区值中提取。</li>
            <li>timestamp_formatter：用于从字符串格式化时间戳的格式化器。</li>
            <li>timestamp_pattern：用于从分区中获取时间戳的模式。</li>
            <li>expire_strategy：指定分区过期的过期策略，可选值为 'values-time' 或 'update-time'，默认为 'values-time'。</li>
            <li>max_expires：过期分区的最大数量限制，可选。</li>
            <li>options：表的额外动态参数。其优先级高于原始的 `tableProp`，低于 `procedureArg`。</li>
      </td>
      <td>CALL sys.expire_partitions(table => 'default.T', expiration_time => '1 d', timestamp_formatter => 
'yyyy-MM-dd', timestamp_pattern => '$dt', expire_strategy => 'values-time', options => 'partition.expiration-max-num=2')</td>
    </tr>
    <tr>
      <td>create_tag</td>
      <td>
         基于给定快照创建标签（Tag）。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tag：新标签的名称。不能为空。</li>
            <li>snapshot(Long)：新标签所基于的快照 id。</li>
            <li>time_retained：新创建标签的最大保留时间。</li>
      </td>
      <td>
         -- 基于快照 10，保留时间为 1d <br/>
         CALL sys.create_tag(table => 'default.T', tag => 'my_tag', snapshot => 10, time_retained => '1 d') <br/><br/>
         -- 基于最新快照 <br/>
         CALL sys.create_tag(table => 'default.T', tag => 'my_tag')
      </td>
    </tr>
    <tr>
      <td>create_tag_from_timestamp</td>
      <td>
         基于给定时间戳创建标签。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tag：新标签的名称。</li>
            <li>timestamp (Long)：查找提交时间（commit-time）大于此时间戳的第一个快照。</li>
            <li>time_retained：新创建标签的最大保留时间。</li>
      </td>
      <td>
         CALL sys.create_tag_from_timestamp(`table` => 'default.T', `tag` => 'my_tag', `timestamp` => 1724404318750, time_retained => '1 d')
      </td>
    </tr>
    <tr>
      <td>rename_tag</td>
      <td>
         用新的标签名称重命名标签。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tag：标签的名称。不能为空。</li>
            <li>target_tag：重命名后的新标签名称。不能为空。</li>
      </td>
      <td>
         CALL sys.rename_tag(table => 'default.T', tag => 'tag1', target_tag => 'tag2')
      </td>
    </tr>
    <tr>
      <td>replace_tag</td>
      <td>
         用新的标签信息替换已存在的标签。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tag：已存在标签的名称。不能为空。</li>
            <li>snapshot(Long)：标签所基于的快照 id，可选。</li>
            <li>time_retained：已存在标签的最大保留时间，可选。</li>
      </td>
      <td>
         CALL sys.replace_tag(table => 'default.T', tag_name => 'tag1', snapshot => 10, time_retained => '1 d')
      </td>
    </tr>
    <tr>
      <td>delete_tag</td>
      <td>
         删除标签。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>tag：要删除的标签的名称。如果你指定多个标签，分隔符为 ','。</li>
      </td>
      <td>CALL sys.delete_tag(table => 'default.T', tag => 'my_tag')</td>
    </tr>
    <tr>
      <td>expire_tags</td>
      <td>
         按时间使标签过期。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>older_than：在此 tagCreateTime 之前的标签将被移除。</li>
      </td>
      <td>
         CALL sys.expire_tags(table => 'default.T', older_than => '2024-09-06 11:00:00')
      </td>
    </tr>
    <tr>
      <td>trigger_tag_automatic_creation</td>
      <td>
         触发标签自动创建。参数：
            <li>table：目标表标识符。不能为空。</li>
      </td>
      <td>
         CALL sys.trigger_tag_automatic_creation(table => 'default.T')
      </td>
    </tr>
    <tr>
      <td>rollback</td>
      <td>
         回滚到目标表的指定版本，注意 version/snapshot/tag 必须设置其中之一。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>version：要回滚到的快照 id 或标签名称，version 将被弃用（Deprecated）。</li>
            <li>snapshot：要回滚到的快照。</li>
            <li>tag：要回滚到的标签。</li>
      </td>
      <td>
          CALL sys.rollback(table => 'default.T', version => 'my_tag')<br/><br/>
          CALL sys.rollback(table => 'default.T', version => 10)<br/><br/>
          CALL sys.rollback(table => 'default.T', tag => 'tag1')
          CALL sys.rollback(table => 'default.T', snapshot => 2)
      </td>
    </tr>
    <tr>
      <td>rollback_to_timestamp</td>
      <td>
         回滚到早于或等于指定时间戳的快照。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>timestamp：回滚到早于或等于该时间戳的快照。</li>
      </td>
      <td>
          CALL sys.rollback_to_timestamp(table => 'default.T', timestamp => 1730292023000)<br/><br/>
      </td>
    </tr>
    <tr>
      <td>rollback_to_watermark</td>
      <td>
         回滚到早于或等于指定 Watermark 的快照。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>watermark：回滚到早于或等于该 Watermark 的快照。</li>
      </td>
      <td>
          CALL sys.rollback_to_watermark(table => 'default.T', watermark => 1730292023000)<br/><br/>
      </td>
    </tr>
    <tr>
      <td>purge_files</td>
      <td>
         通过清除文件来清空表。参数：
            <li>table：目标表标识符。不能为空。</li>
      </td>
      <td>
          CALL sys.purge_files(table => 'default.T')<br/>
      </td>
    </tr>
    <tr>
      <td>migrate_database</td>
      <td>
         将数据库中的所有 Hive 表迁移为 Paimon 表。参数：
            <li>source_type：要迁移的源数据库类型，例如 hive。不能为空。</li>
            <li>database：要迁移的源数据库名称。不能为空。</li>
            <li>options：要迁移的 Paimon 表的表配置项。</li>
            <li>options_map：用于以 map 形式添加键值对配置项的配置项 map。</li>
            <li>parallelism：迁移过程的并行度，默认为机器的核心数。</li>
      </td>
      <td>CALL sys.migrate_database(source_type => 'hive', database => 'db01', options => 'file.format=parquet', options_map => map('k1','v1'), parallelism => 6)</td>
    </tr>
    <tr>
      <td>migrate_table</td>
      <td>
         将 Hive 表迁移为 Paimon 表。参数：
            <li>source_type：要迁移的源表类型，例如 hive。不能为空。</li>
            <li>table：要迁移的源表名称。不能为空。</li>
            <li>options：要迁移的 Paimon 表的表配置项。</li>
            <li>target_table：要迁移到的目标 Paimon 表名称。如果未设置，则与源表保持相同名称。</li>
            <li>delete_origin：如果设置了 target_table，可以设置 delete_origin 来决定迁移后是否从 hms 中删除源表的元数据。默认为 true。</li>
            <li>options_map：用于以 map 形式添加键值对配置项的配置项 map。</li>
            <li>parallelism：迁移过程的并行度，默认为机器的核心数。</li>
      </td>
      <td>CALL sys.migrate_table(source_type => 'hive', table => 'default.T', options => 'file.format=parquet', options_map => map('k1','v1'), parallelism => 6)</td>
    </tr>
    <tr>
      <td>remove_orphan_files</td>
      <td>
         移除孤立的数据文件和元数据文件。参数：
            <li>table：目标表标识符。不能为空，你可以使用 database_name.* 来清理整个数据库。</li>
            <li>older_than：为避免删除新写入的文件，该存储过程默认只删除超过 1 天的孤立文件。此参数可以修改该时间间隔。</li>
            <li>dry_run：为 true 时，仅查看孤立文件，不实际移除文件。默认为 false。</li>
            <li>parallelism：并发删除文件的最大数量。默认为 Java 虚拟机可用的处理器数量。</li>
            <li>mode：移除孤立文件清理存储过程的模式（local 或 distributed）。默认为 distributed。</li>
      </td>
      <td>
          CALL sys.remove_orphan_files(table => 'default.T', older_than => '2023-10-31 12:00:00')<br/><br/>
          CALL sys.remove_orphan_files(table => 'default.*', older_than => '2023-10-31 12:00:00')<br/><br/>
          CALL sys.remove_orphan_files(table => 'default.T', older_than => '2023-10-31 12:00:00', dry_run => true)<br/><br/>
          CALL sys.remove_orphan_files(table => 'default.T', older_than => '2023-10-31 12:00:00', dry_run => true, parallelism => 5)<br/><br/>
          CALL sys.remove_orphan_files(table => 'default.T', older_than => '2023-10-31 12:00:00', dry_run => true, parallelism => 5, mode => 'local')
      </td>
    </tr>
    <tr>
      <td>remove_unexisting_files</td>
      <td>
        用于从 manifest 条目中移除不存在的数据文件的存储过程。详细使用场景请参阅 <a href="https://paimon.apache.org/docs/master/api/java/org/apache/paimon/flink/action/RemoveUnexistingFilesAction.html">Java 文档</a>。参数：
            <li>table：目标表标识符。不能为空，你可以使用 database_name.* 来清理整个数据库。</li>
            <li>dry_run（可选）：仅检查哪些文件将被移除，但不实际移除它们。默认为 false。</li>
            <li>parallelism（可选）：检查 manifest 中文件的并行度。</li>
         <br>
         请注意，使用此存储过程的风险由用户自行承担，如果在 Java 文档所列使用场景之外使用，可能会导致数据丢失。
      </td>
      <td>
        -- 移除表 `mydb.myt` 中不存在的数据文件<br/>
        CALL sys.remove_unexisting_files(table => 'mydb.myt')<br/><br/>
        -- 仅检查哪些文件将被移除，但不实际移除它们（dry run）<br/>
        CALL sys.remove_unexisting_files(table => 'mydb.myt', dry_run = true)
      </td>
   </tr>
    <tr>
      <td>repair</td>
      <td>
         将信息从文件系统同步到 Metastore。参数：
            <li>database_or_table：留空，或目标数据库名称，或目标表标识符，如果你指定多个，则分隔符为 ','。</li>
      </td>
      <td>
          CALL sys.repair('test_db.T')<br/><br/>
          CALL sys.repair('test_db.T,test_db01,test_db.T2')
      </td>
    </tr>
    <tr>
      <td>create_branch</td>
      <td>
         将分支合并到主分支。参数：
            <li>table：目标表标识符或分支标识符。不能为空。</li>
            <li>branch：要合并的分支名称。</li>
            <li>tag：新标签的名称。不能为空。</li>
            <li>ignoreIfExists：如果分支已存在则忽略，默认为 false。</li>
      </td>
      <td>
          CALL sys.create_branch(table => 'test_db.T', branch => 'test_branch')<br/><br/>
          CALL sys.create_branch(table => 'test_db.T', branch => 'test_branch', tag => 'my_tag')<br/><br/>
          CALL sys.create_branch(table => 'test_db.T$branch_existBranchName', branch => 'test_branch', tag => 'my_tag')<br/><br/>
      </td>
    </tr>
    <tr>
      <td>delete_branch</td>
      <td>
         将分支合并到主分支。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>branch：要合并的分支名称。如果你指定多个分支，分隔符为 ','。</li>
      </td>
      <td>
          CALL sys.delete_branch(table => 'test_db.T', branch => 'test_branch')
      </td>
    </tr>
    <tr>
      <td>rename_branch</td>
      <td>
         重命名分支。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>from_branch：要重命名的分支名称。</li>
            <li>to_branch：分支的新名称。</li>
      </td>
      <td>
          CALL sys.rename_branch(table => 'test_db.T', from_branch => 'test_branch', to_branch => 'new_branch')
      </td>
    </tr>
    <tr>
      <td>fast_forward</td>
      <td>
         将一个分支快进（fast_forward）到主分支。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>branch：要合并的分支名称。</li>
      </td>
      <td>
          CALL sys.fast_forward(table => 'test_db.T', branch => 'test_branch')
      </td>
    </tr>
   <tr>
      <td>reset_consumer</td>
      <td>
         重置或删除消费者。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>consumerId：要重置或删除的消费者。</li>
            <li>nextSnapshotId (Long)：消费者新的下一个快照 id。</li>
      </td>
      <td>
         -- 重置消费者中新的下一个快照 id<br/>
         CALL sys.reset_consumer(table => 'default.T', consumerId => 'myid', nextSnapshotId => 10)<br/><br/>
         -- 删除消费者<br/>
         CALL sys.reset_consumer(table => 'default.T', consumerId => 'myid')
      </td>
   </tr>
   <tr>
      <td>clear_consumers</td>
      <td>
         清除消费者。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>includingConsumers：要清除的消费者。</li>
            <li>excludingConsumers：不被清除的消费者。</li>
      </td>
      <td>
         -- 清除表中的所有消费者<br/>
         CALL sys.clear_consumers(table => 'default.T')<br/><br/>
         -- 清除表中的部分消费者（接受正则表达式）<br/>
         CALL sys.clear_consumers(table => 'default.T', includingConsumers => 'myid.*')<br/><br/>
         -- 清除表中除 excludingConsumers 之外的所有消费者（接受正则表达式）<br/>
         CALL sys.clear_consumers(table => 'default.T', includingConsumers => '', excludingConsumers => 'myid1.*')<br/><br/>
         -- 同时使用 includingConsumers 和 excludingConsumers 清除消费者（接受正则表达式）<br/>
         CALL sys.clear_consumers(table => 'default.T', includingConsumers => 'myid.*', excludingConsumers => 'myid1.*')
      </td>
   </tr>
    <tr>
      <td>mark_partition_done</td>
      <td>
         将分区标记为已完成。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>partitions：需要标记为已完成的分区，如果你指定多个分区，分隔符为 ';'。</li>
      </td>
      <td>
         -- 标记单个分区为已完成<br/>
         CALL sys.mark_partition_done(table => 'default.T', partitions => 'day=2024-07-01')<br/><br/>
         -- 标记多个分区为已完成<br/>
         CALL sys.mark_partition_done(table => 'default.T', partitions => 'day=2024-07-01;day=2024-07-02')
      </td>
   </tr>
   <tr>
      <td>compact_manifest</td>
      <td>
         对 manifest（清单）文件执行 compact_manifest。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>options：表的额外动态参数。其优先级高于原始的 `tableProp`，低于 `procedureArg`。</li>
      </td>
      <td>
         CALL sys.compact_manifest(`table` => 'default.T')
      </td>
   </tr>
   <tr>
      <td>alter_view_dialect</td>
      <td>
         修改视图方言。参数：
            <li>view：目标视图标识符。不能为空。</li>
            <li>action：定义变更动作，例如：add、update、drop。不能为空。</li>
            <li>engine：当引擎不是 spark 时需要定义它。</li>
            <li>query：当 action 为 add 和 update 时方言对应的查询，不能为空。</li>
      </td>
      <td>
         -- 在视图中添加方言<br/>
         CALL sys.alter_view_dialect('view_identifier', 'add', 'spark', 'query')<br/>
         CALL sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'add', `query` => 'query')<br/><br/>
         -- 在视图中更新方言<br/>
         CALL sys.alter_view_dialect('view_identifier', 'update', 'spark', 'query')<br/>
         CALL sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'update', `query` => 'query')<br/><br/>
         -- 在视图中删除方言<br/>
         CALL sys.alter_view_dialect('view_identifier', 'drop', 'spark')<br/>
         CALL sys.alter_view_dialect(`view` => 'view_identifier', `action` => 'drop')<br/><br/>
      </td>
   </tr>
      <tr>
      <td>create_function</td>
      <td>
         创建函数。参数：
            <li>function：目标函数标识符。不能为空。</li>
            <li>inputParams：函数的 inputParams。</li>
            <li>returnParams：函数的 returnParams。</li>
            <li>deterministic：函数是否为确定性的。</li>
            <li>comment：函数的注释。</li>
            <li>options：函数的额外动态参数。</li>
      </td>
      <td>
         CALL sys.create_function(`function` => 'function_identifier',<br/>
              `inputParams` => '[{"id": 0, "name":"length", "type":"INT"}, {"id": 1, "name":"width", "type":"INT"}]',<br/>
              `returnParams` => '[{"id": 0, "name":"area", "type":"BIGINT"}]',<br/>
              `deterministic` => true,<br/>
              `comment` => 'comment',<br/>
              `options` => 'k1=v1,k2=v2'<br/>
         )<br/>
      </td>
   </tr>
   <tr>
      <td>alter_function</td>
      <td>
         修改函数。参数：
            <li>function：目标函数标识符。不能为空。</li>
            <li>change：函数的变更。</li>
      </td>
      <td>
         CALL sys.alter_function(`function` => 'function_identifier',<br/>
              `change` => '{"action" : "addDefinition", "name" : "spark", "definition" : {"type" : "lambda", "definition" : "(Integer length, Integer width) -> { return (long) length * width; }", "language": "JAVA" } }'<br/>
         )<br/>
      </td>
   </tr>
   <tr>
      <td>drop_function</td>
      <td>
         删除函数。参数：
            <li>function：目标函数标识符。不能为空。</li>
      </td>
      <td>
         CALL sys.drop_function(`function` => 'function_identifier')<br/>
      </td>
   </tr>
   <tr>
      <td>rewrite_file_index</td>
      <td>
         重写表的文件索引。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>where：分区谓词。留空表示所有分区。</li>
      </td>
      <td>
         CALL sys.rewrite_file_index(table => "t")<br/>
         CALL sys.rewrite_file_index(table => "t", where => "day = '2025-08-17'")<br/>
      </td>
   </tr>
   <tr>
      <td>create_global_index</td>
      <td>
         为给定列创建全局索引文件。该表必须设置 <code>row-tracking.enabled=true</code>。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>index_column：要建立索引的列名。不能为空。</li>
            <li>index_type：要构建的索引类型，例如 'btree' 或 'bitmap'。不能为空。</li>
            <li>partitions：用于限定构建索引的分区的分区过滤条件。逗号（","）表示「AND」，分号（";"）表示「OR」。留空表示所有分区。</li>
            <li>options：表的额外动态参数。其优先级高于原始的 `tableProp`，低于 `procedureArg`。</li>
      </td>
      <td>
         CALL sys.create_global_index(table => 'default.T', index_column => 'name', index_type => 'bitmap')<br/><br/>
         CALL sys.create_global_index(table => 'default.T', index_column => 'name', index_type => 'btree')<br/><br/>
         CALL sys.create_global_index(table => 'default.T', index_column => 'name', index_type => 'btree', partitions => 'pt=p1;pt=p2')
      </td>
   </tr>
   <tr>
      <td>drop_global_index</td>
      <td>
         删除给定列的全局索引文件。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>index_column：被建立索引的列名。不能为空。</li>
            <li>index_type：要删除的索引类型，例如 'btree' 或 'bitmap'。不能为空。</li>
            <li>partitions：用于限定删除索引的分区的分区过滤条件。逗号（","）表示「AND」，分号（";"）表示「OR」。留空表示所有分区。</li>
      </td>
      <td>
         CALL sys.drop_global_index(table => 'default.T', index_column => 'name', index_type => 'bitmap')<br/><br/>
         CALL sys.drop_global_index(table => 'default.T', index_column => 'name', index_type => 'bitmap', partitions => 'pt=p1')
      </td>
   </tr>
   <tr>
      <td>copy</td>
      <td>
         复制表文件。参数：
            <li>source_table：源表标识符。不能为空。</li>
            <li>target_table：目标表标识符。不能为空。</li>
            <li>where：分区谓词。留空表示所有分区。</li>
      </td>
      <td>
         CALL sys.copy(source_table => "t1", target_table => "t1_copy")<br/>
         CALL sys.copy(source_table => "t1", target_table => "t1_copy", where => "day = '2025-08-17'")<br/>
      </td>
   </tr>
   <tr>
      <td>rescale</td>
      <td>
         通过更改桶数来调整表分区的桶数。参数：
            <li>table：目标表标识符。不能为空。</li>
            <li>bucket_num：调整后得到的桶数。默认值为表当前的桶数。对于 postpone bucket 表不能为空。</li>
            <li>partitions：分区过滤条件。留空表示所有分区。（不能与 "where" 同时使用）</li>
            <li>where：分区谓词。留空表示所有分区。（不能与 "partitions" 同时使用）</li>
      </td>
      <td>
         CALL sys.rescale(table => 'default.T', bucket_num => 16, partitions => 'dt=20250217,hh=08;dt=20250217,hh=09')<br/>
      </td>
   </tr>
   </tbody>
</table>
