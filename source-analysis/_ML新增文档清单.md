# ML 新增文档清单（基于缺口核查，2026-06）

> 过程文件。结论：现有 25-28/41 已充分覆盖大部分官方 ML 特性，真缺口对应新增 3 篇（编号 29-31）。源码位置均已核验。

## 29 - 全局索引构建管线：分布式建索引与增量索引
- 缺口：**完全缺失**（28 只讲索引查询/读路径与框架抽象，未讲构建/写侧）
- 要点：
  1. `create_global_index` SQL → Flink `GenericGlobalIndexBuilder`/`GenericIndexTopoBuilder` 与 Spark `DefaultGlobalIndexBuilder`/`*TopoBuilder` 两套拓扑的算子链与并行模型
  2. `GlobalIndexParallelWriter` 局部 RowId(`rowId-rangeStart`) 编址 + `OffsetGlobalIndexReader` 读时回填全局 RowId 的写读对称
  3. 索引与 Data Evolution 文件组对齐：`GlobalIndexScanner`/`GlobalIndexFileReadWrite`、`IndexedSplit`
  4. 增量/部分索引覆盖语义（未索引数据不返回）与重建；`BTreeGlobalIndexBuilder` split 切分
  5. 三引擎(Lumina/Tantivy/BTree)走同一 `GlobalIndexerFactory` SPI
- 源码：`paimon-core/.../globalindex/{GlobalIndexBuilderUtils,GlobalIndexScanner,GlobalIndexFileReadWrite}`、`btree/BTreeGlobalIndexBuilder`；`paimon-common/.../globalindex/GlobalIndexParallelWriter`；`paimon-flink-common/.../flink/globalindex/Generic{GlobalIndexBuilder,IndexTopoBuilder}`；`paimon-spark-common/.../spark/globalindex/`
- 官方：`append-table/global-index.mdx`(构建段)、`pypaimon/global-index.md`
- 区分 28：28=查询/读+框架抽象；29=写入/构建侧分布式工程化

## 30 - Data Evolution 列演进写入：MERGE INTO / Self-Merge / 按 Shard 回填
- 缺口：**仅提及未展开**（28 讲读路径与文件组装配，写侧未展开）
- 要点：
  1. Spark `MERGE INTO` 部分列更新 vs Flink `data_evolution_merge_into` 过程的能力差异（Flink 不支持插新行）
  2. Self-Merge：基于 `$row_tracking` 系统表暴露 `_ROW_ID` + 临时视图 + UDF 原地改列
  3. PyPaimon 三写法：`update_by_arrow_with_row_id`、`upsert_by_arrow_with_key`(业务键 upsert、分区键自动剥离)、`new_shard_updator`(分片读改写派生列)
  4. Batch vs Stream API 差异（commit_identifier、实例复用）
  5. raw-data BLOB 列在部分列 MERGE 中被拒、descriptor 列被允许的边界
- 源码：`paimon-flink-common/.../procedure/DataEvolutionMergeIntoProcedure`、`action/DataEvolutionMergeIntoAction`、`dataevolution/DataEvolutionPartialWriteOperator`；`paimon-core/.../append/dataevolution/DataEvolutionCompact*`
- 官方：`append-table/data-evolution.md`、`pypaimon/data-evolution.md`
- 区分：28=读路径；30=写入侧 SQL/Python API 与样本特征回填操作手册

## 31 - 增量聚簇与 ML 大宽表布局优化
- 缺口：**仅提及未展开**
- 要点：
  1. append unaware-bucket 专属 `clustering.incremental`，与 12 篇通用 SortCompaction 的本质区别（只选子集文件、避免全量重写）
  2. `clustering.columns/strategy`(order/zorder/hilbert 按列数自动选) 对特征/样本宽表查询裁剪的收益
  3. 历史分区 auto-cluster(`history-partition.idle-to-full-sort/limit`)
  4. `append/cluster/IncrementalClusterManager`+`Strategy` 选文件算法；与 write-time/dedicated compaction 互斥
  5. ML 场景：向量召回表/样本宽表按聚簇键加速过滤
  6. 可并入 `bucketed.mdx` 的 data skipping/bucketed streaming/watermark(26 未覆盖)
- 源码：`paimon-core/.../append/cluster/{IncrementalClusterManager,IncrementalClusterStrategy}`；`mergetree/compact/clustering/Clustering*`；`paimon-flink-common/.../flink/{cluster/IncrementalClusterSplitSource,compact/IncrementalClusterCompact}`
- 官方：`append-table/incremental-clustering.mdx`、`bucketed.mdx`
- 区分：12=通用排序+LSM聚簇原理；31=append 增量聚簇新机制+ML宽表落地
