# Flink 写入 Paimon 源码流程深度分析

> **版本**：1.5-SNAPSHOT　**源码模块**：`paimon-flink/paimon-flink-common`（写入算子链）+ `paimon-core`（存储层写入/提交）　**核对日期**：2026-06　**Flink 版本**：1.16 ~ 2.2

**一句话定位**：本文讲透 Flink 把一条 `RowData` 从算子链写到 Paimon 落盘、再随 checkpoint 两阶段提交成 Snapshot 的**完整写入链路**——它要在 Flink 的分布式快照协议之上，叠加 Paimon 的 LSM 写缓冲、共享内存抢占、异步 compaction 与冲突重试，最终交付 exactly-once。

读完本文你应能回答：① 一条 `RowData` 经过哪几个算子、几次 shuffle 才落成 Level 0 文件；② `StoreSinkWrite` 三种实现各自在什么场景被工厂选中、解决什么；③ checkpoint barrier 到达时 `prepareSnapshotPreBarrier` 为什么必须 flush、它和 `notifyCheckpointComplete` 如何构成两阶段提交；④ exactly-once 到底靠哪四件事（强制 EXACTLY_ONCE、commitUser 幂等、单并行度 Committer、冲突重试）共同兜底；⑤ Writer 故障与 Committer 故障恢复时分别发生什么、为什么不会重复或丢数；⑥ 共享内存池的抢占 flush 在什么情况下会"误伤"成小文件；⑦ 批作业 `endInput` 与流作业 checkpoint 的提交路径差在哪、为什么批作业要做幂等检查；⑧ 五种 BucketMode 的 Sink 拓扑差异（尤其动态桶为什么两次 shuffle）。

> 阅读约定：本文每个机制按"① 要解决什么问题 → ② 设计原理与取舍 → ③ 关键源码（精选片段 + `路径:行号`）→ ④ 风险/陷阱/边界 → ⑤ 收益与代价"组织。源码行号以本次核对为准；与旧稿不符处用 `（已修正）` 标注。
>
> **重叠交叉引用**：Flink 整体集成架构（Source/Lookup/CDC 入口、Catalog 接入）由 **05 号文档**主讲，本文只聚焦写入链路；存储层 `FileStoreCommit` 的冲突检测与原子重命名细节见 **01 号文档 §13**，本文只讲它在两阶段提交里的位置；LSM Merge/Compaction 基础见 **01 号文档 §3/§8**；CDC 数据源同步见 **14 号文档**，本文只讲 RowKind 在写入侧的处理。

---

## 目录

- [1. 快速理解（核心问题 / 概念速查 / 高频陷阱）](#1-快速理解核心问题--概念速查--高频陷阱)
  - [1.1 核心问题：Flink 写入要在快照协议上叠加什么](#11-核心问题flink-写入要在快照协议上叠加什么)
  - [1.2 核心概念速查表](#12-核心概念速查表)
  - [1.3 一条记录的写入全生命周期](#13-一条记录的写入全生命周期)
  - [1.4 高频生产陷阱](#14-高频生产陷阱)
- [2. 写入链路全景：算子链与类继承](#2-写入链路全景算子链与类继承)
- [3. SQL INSERT 到 Sink 构建：DynamicTableSink 入口](#3-sql-insert-到-sink-构建dynamictablesink-入口)
- [4. BucketMode 分发：五种 Sink 拓扑](#4-bucketmode-分发五种-sink-拓扑)
- [5. Writer 算子三层：算子链如何承载写入](#5-writer-算子三层算子链如何承载写入)
- [6. StoreSinkWrite 策略分派：三种写入实现](#6-storesinkwrite-策略分派三种写入实现)
- [7. 数据写入核心路径：从 RowData 到 Level 0 文件](#7-数据写入核心路径从-rowdata-到-level-0-文件)
- [8. Checkpoint 两阶段提交与 exactly-once](#8-checkpoint-两阶段提交与-exactly-once)
- [9. 共享内存池与抢占机制](#9-共享内存池与抢占机制)
- [10. 异步 Compaction 与写入反压](#10-异步-compaction-与写入反压)
- [11. CDC / RowKind 在写入侧的处理](#11-cdc--rowkind-在写入侧的处理)
- [12. 批写入 vs 流写入的差异](#12-批写入-vs-流写入的差异)
- [13. 与 Iceberg Flink Sink 的对比](#13-与-iceberg-flink-sink-的对比)
- [14. 设计决策总结](#14-设计决策总结)
- [附录 A：核心源码文件索引](#附录-a核心源码文件索引)

---

## 1. 快速理解（核心问题 / 概念速查 / 高频陷阱）

### 1.1 核心问题：Flink 写入要在快照协议上叠加什么

**① 要解决什么问题**

Flink 已经提供了 checkpoint 这一分布式快照协议来保证状态一致性，但它只管"算子状态"，不管"外部存储里那一堆文件是否原子可见"。Paimon 要把流式/批式数据写成对象存储上的 ACID 快照，必须在 Flink 协议之上额外解决四件事：

- **原子可见**：N 个 writer 子任务各自写出一批文件，必须"要么一起对读可见、要么都不可见"——不能让读到一半提交的表。
- **不丢不重**：writer 崩溃后未提交的文件要被丢弃（不重复），已 emit 给 committer 的 committable 要在恢复后补提交（不丢失）。
- **写得快**：直接每条记录写文件会爆小文件，必须有内存写缓冲（LSM SortBuffer）+ 异步 compaction，且在有限内存下支撑成百上千个 partition+bucket 并发写。
- **读得对**：CDC 的 INSERT/UPDATE/DELETE 要按 merge-engine 语义正确合并，需要把 RowKind 嵌进存储单元。

**② 设计原理与取舍**

Paimon 的答案是把写入拆成**两类算子 + 两阶段提交**：

- **Writer 算子**（多并行度）：负责把数据写进内存缓冲、flush 成文件、触发异步 compaction，但**不提交**。它在 checkpoint barrier 之前（`prepareSnapshotPreBarrier`）把这一轮产生的文件元数据（`Committable`）发给下游。
- **Committer 算子**（**固定并行度 1**）：收集所有 writer 的 committable，在 checkpoint **完成后**（`notifyCheckpointComplete`）一次性原子写出 Snapshot。

这正是 Flink `TwoPhaseCommit` 思想的落地，但 Paimon 没用 Flink 的 `TwoPhaseCommittingSink` 接口，而是自己实现 `PrepareCommitOperator` + `CommitterOperator`，以便控制内存池、compaction 协调和冲突重试。

一句话设计哲学：**Writer 只"备货"，Committer 才"过账"；备货可重做（幂等丢弃），过账靠 commitUser + 冲突重试保证只过一次。**

**与"每个 writer 自己提交"的替代方案对比**：

| 维度 | 各 writer 自行提交 | Paimon：单 Committer 两阶段提交 |
|------|-------------------|-------------------------------|
| 原子性 | 差（部分 writer 提交成功就可见） | 优（一次 commit 覆盖全部 bucket） |
| 冲突处理 | 多写者竞争同一 snapshot 指针，冲突频繁 | 串行提交，冲突只来自其他作业，重试简单 |
| 故障恢复 | 难判断哪些已提交 | commitUser 幂等 + snapshot 存在性检查 |
| 吞吐瓶颈 | 无单点 | Committer 单点（但只做元数据操作，通常不是瓶颈） |

### 1.2 核心概念速查表

| 概念 | 一句话定义 | 关键源码 |
|------|-----------|---------|
| **FlinkSinkBuilder** | Sink 构建入口，按 BucketMode 路由到具体 Sink 子类 | `FlinkSinkBuilder.java:211` |
| **FlinkSink / FlinkWriteSink** | 抽象基类，定义 `doWrite → doCommit` 骨架；强制 EXACTLY_ONCE | `FlinkSink.java:270` |
| **PrepareCommitOperator** | Writer 算子最底层基类：建内存池、checkpoint 前 `emitCommittables` | `PrepareCommitOperator.java:92` |
| **RowDataStoreWriteOperator** | 实际写入算子：`processElement → write.write(row)` | `RowDataStoreWriteOperator.java:54` |
| **StoreSinkWrite** | Flink 侧写入策略接口 + 工厂 `createWriteProvider` | `StoreSinkWrite.java:100` |
| **TableWriteImpl** | core 侧写入：非空检查、提取 partition/bucket/key、转 KeyValue | `TableWriteImpl.java:182` |
| **MergeTreeWriter** | LSM 写入器：分配 seqNum、写 WriteBuffer、flush、触发 compaction | `MergeTreeWriter.java:163` |
| **CommitterOperator** | 提交算子（并行度 1）：`notifyCheckpointComplete → commitUpToCheckpoint` | `CommitterOperator.java:189` |
| **StoreCommitter** | 包装 `TableCommitImpl.commitMultiple` 做原子提交 | `StoreCommitter.java:98` |
| **Committable** | 待提交单元：`checkpointId + CommitMessage(文件增量)` | `Committable`（paimon-flink） |
| **CommitUser** | 作业唯一标识，从算子状态恢复，用于提交幂等 | `CommitterOperator.java:129` |
| **MemoryPoolFactory** | 所有 writer 共享的内存池 + 抢占 flush | `MemoryPoolFactory.java:73` |
| **BucketMode** | 五种数据分桶策略，决定 Sink 拓扑 | `BucketMode.java` |

### 1.3 一条记录的写入全生命周期

以流式 upsert 一行 `+U(pk=42, ...)` 为例，串起全文各章：

```
1. RowData 进入算子链      FlinkRowWrapper 零拷贝包成 InternalRow（§7.2）
2. [可选] LocalMerge       主键表本地预合并，同 key 连续更新只下发最终值（§5.3）
3. shuffle                 按 BucketMode 分区：固定桶一次、动态桶两次 shuffle（§4）
4. RowDataStoreWriteOperator.processElement → write.write(row)（§5.1）
5. TableWriteImpl.writeAndReturn   非空检查→默认值→取 RowKind→提取 partition/bucket/key→转 KeyValue（§7.3）
6. MergeTreeWriter.write    分配 sequenceNumber，put 进 WriteBuffer；满则 flush（§7.4）
7. flushWriteBuffer         排序+MergeFunction 预合并→落 Level 0 文件；INPUT 模式同时写 changelog（§7.4，详见 01 §4.3）
8. compaction 异步触发      CompactManager 选 run 异步合并；run 过多则写入反压（§10，策略见 01 §8）
9. checkpoint barrier 前    prepareSnapshotPreBarrier → prepareCommit → drainIncrement → emit Committable（§8.2）
10. checkpoint 完成后       Committer.notifyCheckpointComplete → commitMultiple → 冲突检测+原子重命名生成 Snapshot（§8.3，详见 01 §13）
```

写快（内存缓冲 + 异步 compaction）、提交对（两阶段 + 单 Committer）、恢复稳（commitUser 幂等 + 冲突重试）、内存省（共享池 + 抢占）——这条链贯穿全文。

### 1.4 高频生产陷阱

| 陷阱 | 后果 | 规避 |
|------|------|------|
| **流式作业用 AT_LEAST_ONCE 或不开 checkpoint** | 直接抛异常（`FlinkSink.java:270`），或同一 cp 被提交多次导致重复 | 必须 `execution.checkpointing.mode=exactly-once` |
| **checkpoint 超时设得太短** | flush + 同步 compaction（尤其 full-compaction/lookup 模式 `waitCompaction=true`）耗时超 timeout，cp 反复失败 | timeout 至少 5 分钟；间隔 1–5 分钟 |
| **误以为 `write-buffer-size` 是每 writer 的内存** | 实际是整个算子实例所有 writer 共享的总量（§9）；按"每 writer"估算会严重高估 | 监控 `bufferPreemptCount`，频繁抢占说明总量不足 |
| **抢占风暴致小文件爆炸** | 多 partition+bucket 并发 + 内存紧张 → 频繁抢占 flush → 每次 flush 出的文件很小（§9.2） | 调大 `write-buffer-size`、减少同时活跃的 bucket、开 spillable |
| **流式作业用 INSERT OVERWRITE** | 抛 `Paimon doesn't support streaming INSERT OVERWRITE`（`FlinkTableSinkBase.java:104`） | OVERWRITE 仅批模式 |
| **主键表设 `bucket=0`** | 抛"Unaware bucket mode only works with append-only table"（`FlinkSinkBuilder.java:338`） | 主键表用固定/动态/postpone 桶 |
| **跨分区 upsert 误判成 HASH_DYNAMIC** | 同一主键的新旧版本落到不同分区，造成重复 | 主键不含全部分区键时 `bucket=-1` 会自动选 KEY_DYNAMIC（需全量 bootstrap，启动慢） |
| **commitUser 未固定** | 作业重启后 commitUser 变化，旧未提交数据无法被幂等识别 | 显式配置 `commit.user`（生产强烈建议） |
| **full-compaction 模式 checkpoint 被 compaction 阻塞** | `GlobalFullCompactionSinkWrite` 每 `deltaCommits` 个 cp 强制全量 compaction 并 `waitCompaction=true`，那一次 cp 会很慢 | 合理设 `full-compaction.delta-commits`，配足 timeout |

## 2. 写入链路全景：算子链与类继承

### 2.1 端到端算子链（以 HASH_FIXED 流式为例）

```
INSERT INTO paimon_table ...
   │  Planner 经 Catalog 解析到 FileStoreTable，构造 FlinkTableSinkBase（DynamicTableSink）
   ▼
FlinkTableSinkBase.getSinkRuntimeProvider() → PaimonDataStreamSinkProvider（Lambda 延迟构建）
   ▼
FlinkSinkBuilder.build()
   │  ① mapToInternalRow：RowData → InternalRow（FlinkRowWrapper 零拷贝包装）
   │  ② [可选] LocalMergeOperator：主键表本地预合并（forward 连接，不 shuffle）
   │  ③ switch(BucketMode) 路由到具体 Sink
   ▼
=========================== 以下是真正的 Flink 算子链 ===========================
DataStream<InternalRow>
   │  FlinkStreamPartitioner（按 partition+bucket hash 重分区）        ← 一次 shuffle
   ▼
"Writer : <table>"  = RowDataStoreWriteOperator（多并行度）
   │   processElement → StoreSinkWrite.write → TableWriteImpl → MergeTreeWriter.write
   │   checkpoint barrier 前：prepareSnapshotPreBarrier → emit Committable
   ▼
DataStream<Committable>
   │  [可选] Changelog Compact Coordinator/Worker（PRECOMMIT_COMPACT 时）
   ▼
"Global Committer : <table>" = CommitterOperator（并行度强制 1）
   │   notifyCheckpointComplete → commitUpToCheckpoint → StoreCommitter.commit
   ▼
DiscardingSink
```

要点：**写入算子是多并行度、提交算子是单并行度**；二者之间隔着一次 checkpoint。Writer 在 barrier 前把文件备好发下游，Committer 在 cp 完成后才过账（§8）。整体的 SQL→Catalog→DynamicTableSink 接入细节属于 Flink 集成层，**详见 05 号文档**。

### 2.2 Sink 类继承层次

```
FlinkSink<T> (abstract)                         ← doWrite/doCommit 骨架、EXACTLY_ONCE 校验
  └── FlinkWriteSink<T> (abstract)              ← 提供 StoreCommitter/CommittableStateManager
        ├── FixedBucketSink                     ← HASH_FIXED
        ├── PostponeBucketSink                  ← POSTPONE_MODE (流式/首次批)
        ├── PostponeFixedBucketSink             ← POSTPONE_MODE (已知桶数批写)
        ├── DynamicBucketSink<T> (abstract)
        │     ├── RowDynamicBucketSink          ← HASH_DYNAMIC
        │     └── DynamicBucketCompactSink      ← HASH_DYNAMIC compact
        ├── GlobalDynamicBucketSink             ← KEY_DYNAMIC
        └── AppendTableSink<T> (abstract)
              └── RowAppendTableSink            ← BUCKET_UNAWARE
```

**设计哲学（模板方法）**：`FlinkSink` 把"建 Writer 算子 → 建 Committer 算子 → 连成拓扑"的骨架固定下来，子类只重写 `createWriteOperatorFactory()` 和数据分区策略。新增一种 BucketMode 只需加一个 Sink 子类 + 一个 `ChannelComputer`，**不碰提交逻辑**——这是这套继承体系的全部价值。所有子类共享同一个 `CommitterOperator`，因此 exactly-once 语义只实现一遍。

---

## 3. SQL INSERT 到 Sink 构建：DynamicTableSink 入口

> 本节属于 Flink 集成层的"最后一公里"——SQL 如何落到 `FlinkSinkBuilder`。Catalog 接入、Planner 优化、Source 侧等整体集成由 **05 号文档**主讲，这里只讲与**写入**直接相关的两个决策点：ChangelogMode 协商、Sink 延迟构建。

### 3.1 ChangelogMode 协商：决定要不要 UPDATE_BEFORE

**① 要解决什么问题**

Flink Planner 需要知道 Sink 能接收哪些 RowKind，以决定是否在 Sink 前插入 `SinkMaterializer`（一个把 changelog 流物化成 upsert 流的算子，开销不小）。如果 Sink 声明"我不要 UPDATE_BEFORE"，上游就能省掉一半的更新数据传输。

**② 设计原理与取舍**

主键表默认走 **upsert 语义**：UPDATE_AFTER 已经携带完整最新值，不需要 UPDATE_BEFORE。但三种情况例外，必须保留全部 RowKind：

```java
// FlinkTableSinkBase.java:70-100
public ChangelogMode getChangelogMode(ChangelogMode requestedMode) {
    if (table.primaryKeys().isEmpty()) {
        return requestedMode;                       // 无主键表：原样接受
    }
    Options options = Options.fromMap(table.options());
    if (options.get(CHANGELOG_PRODUCER) == ChangelogProducer.INPUT) return requestedMode;        // 例外1：INPUT 要完整 changelog
    if (options.get(MERGE_ENGINE) == MergeEngine.AGGREGATE) return requestedMode;                 // 例外2：聚合需 UPDATE_BEFORE 撤回
    if (options.get(MERGE_ENGINE) == MergeEngine.PARTIAL_UPDATE
            && new CoreOptions(options).definedAggFunc()) return requestedMode;                    // 例外3：部分更新带聚合
    // 默认：过滤掉 UPDATE_BEFORE
    ChangelogMode.Builder builder = ChangelogMode.newBuilder();
    for (RowKind kind : requestedMode.getContainedKinds())
        if (kind != RowKind.UPDATE_BEFORE) builder.addContainedKind(kind);
    return builder.build();
}
```

**④ 风险/陷阱**：若 Paimon 声明只接受 INSERT/UPDATE_AFTER/DELETE，而 Flink 仍判定需要物化，会插入 `SinkMaterializer`——但 Paimon Sink 与该算子不兼容，`StoreSinkWrite.createWriteProvider` 里有 `assertNoSinkMaterializer` 兜底，会直接报错让你把 `table.exec.sink.upsert-materialize` 设为 `none`（见 `StoreSinkWrite.java:106-115`）。aggregate 表若误把 changelog 配成 upsert 过滤了 UPDATE_BEFORE，会导致聚合撤回失效、结果偏大。

**⑤ 收益**：对 deduplicate/partial-update 这类只需最新值的场景，省掉 UPDATE_BEFORE 等于网络与序列化量减半。

### 3.2 Sink 延迟构建：getSinkRuntimeProvider

`getSinkRuntimeProvider` 返回的是一个 Lambda，真正的拓扑在 DataStream 执行期才构建——这样能拿到运行时的 `isBounded`，按流/批动态决定是否启用 Clustering（仅批模式有效）。

```java
// FlinkTableSinkBase.java:102-146（精简）
public SinkRuntimeProvider getSinkRuntimeProvider(Context context) {
    if (overwrite && !context.isBounded())                                  // 流式禁止 OVERWRITE
        throw new UnsupportedOperationException("Paimon doesn't support streaming INSERT OVERWRITE.");
    if (table instanceof FormatTable)                                       // 日志表走独立路径，无 LSM
        return new PaimonDataStreamSinkProvider(ds -> new FlinkFormatTableDataStreamSink(...).sinkFrom(ds), name, table);
    return new PaimonDataStreamSinkProvider(dataStream -> {
        FlinkSinkBuilder builder = createSinkBuilder();
        builder.forRowData(dataStream);
        if (!conf.get(CLUSTERING_INCREMENTAL) || conf.get(CLUSTERING_INCREMENTAL_OPTIMIZE_WRITE))
            builder.clusteringIfPossible(...);                              // 仅批 + BUCKET_UNAWARE 实际生效
        if (overwrite) builder.overwrite(staticPartitions);
        conf.getOptional(SINK_PARALLELISM).ifPresent(builder::parallelism); // sink.parallelism 优先级最高
        return builder.build();
    }, name, table);
}
```

**陷阱**：`FormatTable`（日志存储表）完全不走本文的写入链路——没有主键、没有 LSM、没有 compaction，别拿本文的调优经验套它。

---

## 4. BucketMode 分发：五种 Sink 拓扑

### 4.1 一处 switch 决定整条拓扑

**① 要解决什么问题**：不同 bucket 配置对"正确性"和"性能"的最优拓扑完全不同——固定桶可一次 shuffle 直达 writer，但跨分区 upsert 必须先建全局索引才能知道某主键的旧桶在哪。BucketMode 的本质是把"如何把数据正确路由到 writer"这一决策从用户手里收归框架。

**② 设计原理**：`FlinkSinkBuilder.build()` 把通用前处理（并行度冲突、排序、RowData→InternalRow、可选 LocalMerge）做完后，用一个 `switch` 路由：

```java
// FlinkSinkBuilder.java:211-251（精简）
public DataStreamSink<?> build() {
    setParallelismIfAdaptiveConflict();
    input = trySortInput(input);
    DataStream<InternalRow> input = mapToInternalRow(this.input, table.rowType(), ...);
    if (table.coreOptions().localMergeEnabled() && table.schema().primaryKeys().size() > 0) {
        // 主键表本地预合并：forward 连接（同并行度、不跨网络），见 §5.3
        SingleOutputStreamOperator<InternalRow> newInput =
            input.forward().transform("local merge", input.getType(), new LocalMergeOperator.Factory(table.schema()));
        forwardParallelism(newInput, input);
        input = newInput;
    }
    switch (table.bucketMode()) {
        case POSTPONE_MODE:   return buildPostponeBucketSink(input);
        case HASH_FIXED:      return buildForFixedBucket(input);
        case HASH_DYNAMIC:    return buildDynamicBucketSink(input, false);
        case KEY_DYNAMIC:     return buildDynamicBucketSink(input, true);
        case BUCKET_UNAWARE:  return buildUnawareBucketSink(input);
        default: throw new UnsupportedOperationException("Unsupported bucket mode: " + table.bucketMode());
    }
}
```

注意 `bucketMode()` 由表 schema 自动推导（`bucket>0`→HASH_FIXED，`=0`→BUCKET_UNAWARE，`=-1`→主键含全部分区键则 HASH_DYNAMIC 否则 KEY_DYNAMIC，`=-2`→POSTPONE），用户只配 `bucket` 数即可。BucketMode 的存储层语义（每桶一棵 LSM、动态桶索引维护）见 **01 号文档 §10**。

### 4.2 五种模式：拓扑与代价对照

| BucketMode | 触发 | 适用表 | Sink 类 | shuffle 次数 | 关键代价 |
|---|---|---|---|---|---|
| `HASH_FIXED` | `bucket=N (N>0)` | 主键/追加表 | `FixedBucketSink` | 1（按 partition+bucket） | 桶数固定，倾斜需重建 |
| `HASH_DYNAMIC` | `bucket=-1`，主键含全部分区键 | 主键表 | `RowDynamicBucketSink` | **2**（先 key、后 partition+bucket） | 网络翻倍 + assigner 维护 hash 索引 |
| `KEY_DYNAMIC` | `bucket=-1`，主键不含全部分区键 | 跨分区主键表 | `GlobalDynamicBucketSink` | 2 + **bootstrap** | 启动需全量扫主键建 RocksDB 索引，大表数分钟 |
| `BUCKET_UNAWARE` | `bucket=0` | 仅追加表 | `RowAppendTableSink` | 0~1（可选按 partition） | 无确定路由，易小文件，需独立 compact 拓扑（§12.4） |
| `POSTPONE_MODE` | `bucket=-2` | 主键表 | `PostponeBucketSink`/`PostponeFixedBucketSink` | 1 | 桶号推迟到 writer/已知时分配 |

### 4.3 关键拓扑细节

**HASH_FIXED——非分区表自动收并行度**（`FlinkSinkBuilder.java:292-307`）：若用户没指定并行度、桶数 < 输入并行度、且无分区键，writer 并行度自动降为桶数。原因：非分区表只有 4 个 bucket 却开 128 并行度时，124 个 writer 子任务永远收不到数据，纯浪费。

**HASH_DYNAMIC——为什么两次 shuffle**（`buildDynamicBucketSink`，`FlinkSinkBuilder.java:280-290`）：
- 第一次按 key hash → 保证**同一主键永远到同一个 assigner**，否则同 key 可能被两个 assigner 分到不同 bucket，破坏"一个主键一个桶"的不变式。
- assigner 算子据 hash 索引给每条记录打上 bucket 号，输出 `Tuple2<row, bucket>`。
- 第二次按 partition+bucket → 保证**同一 partition+bucket 到同一 writer**，否则两个 writer 写同一 LSM 树必然冲突。
两次 shuffle 是正确性的代价，不是可选优化。

**KEY_DYNAMIC——多一个 bootstrap 阶段**：跨分区 upsert 要知道"主键 X 之前落在哪个分区+桶"，否则会在新分区再插一份造成重复。所以 assigner 前插 `INDEX_BOOTSTRAP`，启动时全量扫主键载入 RocksDB 索引——这是它启动慢、占内存的根源。

**POSTPONE_MODE——流批两种行为**（`buildPostponeBucketSink`，`FlinkSinkBuilder.java:309-335`）：
- 流式或首次批写：`PostponeBucketSink`，桶号推迟到 writer 侧动态定。
- 批写且 `postpone-batch-write-fixed-bucket=true` 且桶数已知：`PostponeFixedBucketSink`，shuffle 时就按已知桶数分配，性能更好。

**BUCKET_UNAWARE——只允许追加表**（`buildUnawareBucketSink`，`FlinkSinkBuilder.java:337-348`）：
```java
checkArgument(table.primaryKeys().isEmpty(),
    "Unaware bucket mode only works with append-only table for now.");
```
无桶模式没有确定性路由，无法保证同一主键的更新到同一 writer，所以主键表配 `bucket=0` 直接报错。

---

## 5. Writer 算子三层：算子链如何承载写入

### 5.1 三层算子的职责切分

**① 要解决什么问题**：写入算子要同时干三件互不相干的事——管内存池、管 Flink 状态、写数据。如果揉在一个类里，每加一种数据类型（InternalRow / CDC Record）都要复制全部逻辑。

**② 设计原理（三层继承）**：

```
AbstractStreamOperator<Committable>
  └── PrepareCommitOperator       建内存池 + checkpoint 前 emitCommittables（与数据类型无关）
        └── TableWriteOperator     持有 StoreSinkWrite、管理 commitUser/write 状态、状态恢复
              └── RowDataStoreWriteOperator   只负责 processElement → write.write(row)
```

每层只关心一件事：底层管"什么时候把货发出去"，中层管"状态怎么存/恢复"，顶层管"一条数据怎么写"。CDC 场景换一个顶层算子即可复用下面两层（§11）。

**③ 关键源码**——顶层薄得只剩一句：
```java
// RowDataStoreWriteOperator.java:53-65
public void processElement(StreamRecord<InternalRow> element) throws Exception {
    write(element.getValue());
}
protected SinkRecord write(InternalRow row) throws Exception {
    try { return write.write(row); }            // 委托给 StoreSinkWrite
    catch (Exception e) { throw new IOException(e); }
}
```

### 5.2 PrepareCommitOperator：备货与发货的总闸

这是整个写入侧的"心脏"——它定义了 checkpoint 前后的行为，所有 writer 子类都继承它：

```java
// PrepareCommitOperator.java:92-117
public void prepareSnapshotPreBarrier(long checkpointId) throws Exception {
    if (!endOfInput) emitCommittables(false, checkpointId);   // 流式：barrier 前发货，不等 compaction
}
public void endInput() throws Exception {
    endOfInput = true;
    emitCommittables(true, Long.MAX_VALUE);                   // 批式：结尾发货，等 compaction
}
private void emitCommittables(boolean waitCompaction, long checkpointId) throws IOException {
    prepareCommit(waitCompaction, checkpointId)
        .forEach(committable -> output.collect(new StreamRecord<>(committable)));
}
```

两个发货时机的差异是流批分野的关键（§12）：流式在 `prepareSnapshotPreBarrier`（barrier 之前）发，`waitCompaction=false`；批式在 `endInput` 发，`waitCompaction=true` 强制把 compaction 等完。`setup()`（`PrepareCommitOperator.java:67-90`）在这里建好 `MemoryPoolFactory`，整个算子实例的所有 writer 共享它（§9）。

### 5.3 LocalMergeOperator：shuffle 前的本地预合并

**① 要解决什么问题**：主键热点。某些主键被高频更新（如某爆款商品的库存），所有更新都 shuffle 到同一个 writer，单点压力大、网络也浪费。

**② 设计原理**：在第一次 shuffle **之前**用 `forward()` 连接（同并行度、本地直连不跨网络）插一个本地合并算子。同一主键的连续更新在本地先 merge 成一条，只把最终值下发，shuffle 数据量大减。

**③ 关键源码**——两种 merger 自动选型：
```java
// LocalMergeOperator.java:108-145（精简）
boolean canHashMerger = true;
for (DataField field : valueType.getFields()) {
    if (primaryKeys.contains(field.name())) continue;
    if (!BinaryRow.isInFixedLengthPart(field.type())) { canHashMerger = false; break; }  // 有变长字段
}
merger = canHashMerger ? new HashMapLocalMerger(...)   // 全定长：HashMap，内存高效
                       : new SortBufferLocalMerger(...); // 有变长（STRING/ARRAY）：排序缓冲，可溢写
```
flush 时机：缓冲满、`prepareSnapshotPreBarrier`、`endInput` 三处都会 flush（`LocalMergeOperator.java:152/181/189`）。

**④ 风险**：`local-merge-buffer-size` 配太大 → OOM；配太小 → 合不动、白白多一层算子开销。它不改变最终结果，只是吞吐优化，热点不明显时可不开。

### 5.4 FlinkStreamPartitioner：把 ChannelComputer 接进 Flink

分区器本身极薄，真正的路由逻辑在 `ChannelComputer`：
```java
// FlinkStreamPartitioner.java:49-51
public int selectChannel(SerializationDelegate<StreamRecord<T>> record) {
    return channelComputer.channel(record.getInstance().getValue());
}
```
按 BucketMode 换不同 `ChannelComputer`：`RowDataChannelComputer`（partition+bucket，固定桶）、`RowAssignerChannelComputer`（主键 hash，动态桶 assigner 前）、`RowWithBucketChannelComputer`（partition+bucket，动态桶 writer 前）、`RowDataHashPartitionChannelComputer`（仅 partition，无桶）。bucket 计算公式见 **01 号文档 §10.2**。

---

## 6. StoreSinkWrite 策略分派：三种写入实现

### 6.1 工厂按配置选实现

**① 要解决什么问题**：写入器在"基础写入"之外，有时还要承担额外职责——定期触发全局 full compaction（为了产 changelog）、恢复时重新触发 compact（lookup changelog）。这些职责不该硬编码进基础写入器，而应按表配置在工厂里挑实现。

**② 设计原理（工厂 + 继承）**：`StoreSinkWrite.createWriteProvider` 是唯一入口，按 `write-only` / `changelog-producer` / `full-compaction.delta-commits` 选三种实现之一，全部以 `StoreSinkWriteImpl` 为基类：

```
createWriteProvider（StoreSinkWrite.java:100-187）
  ├── writeOnly                                         → StoreSinkWriteImpl（waitCompaction=false，纯写）
  ├── changelog-producer=FULL_COMPACTION 或配了 delta  → GlobalFullCompactionSinkWrite
  ├── needLookup()（changelog-producer=LOOKUP）          → LookupSinkWrite
  └── 默认                                              → StoreSinkWriteImpl
```

每条分支返回的 Provider Lambda 里都先跑 `assertNoSinkMaterializer.run()`（`StoreSinkWrite.java:106-115`）——这是与 ChangelogMode 协商（§3.1）配套的兜底：一旦 Flink 误插了 `SinkMaterializer` 就直接报错。`waitCompaction` 来源：write-only 恒 false，否则取 `prepareCommitWaitCompaction()`，full-compaction 触发点会临时置 true（§6.2）。

### 6.2 三种实现对比

| | StoreSinkWriteImpl | GlobalFullCompactionSinkWrite | LookupSinkWrite |
|---|---|---|---|
| 关系 | 基类 | 继承 Impl | 继承 Impl |
| 职责 | 基础写入 + prepareCommit | 定期全局 full compaction 产 changelog | 恢复时对活跃 bucket 重触发 compact |
| 额外状态 | 无 | `writtenBuckets`（每 cp 改了哪些桶） | `activeBuckets`（活跃桶集合） |
| 适用 | 默认 / write-only | `changelog-producer=full-compaction` | `changelog-producer=lookup` |

**GlobalFullCompactionSinkWrite——为什么要"全局同步"**：full-compaction 模式靠"全量 compact 前后 diff"生成 changelog。若各 writer 各自决定何时 full compact，产出的 changelog 在时间线上不一致。所以它记录每个 cp 修改过的桶，到达 `deltaCommits` 倍数的 cp 时统一对所有 `writtenBuckets` 触发 full compaction 并 `waitCompaction=true`：
```java
// GlobalFullCompactionSinkWrite.java:149-185（精简）
public List<Committable> prepareCommit(boolean waitCompaction, long checkpointId) {
    checkSuccessfulFullCompaction();                                         // 校验上次 full compaction 落地
    /* 记录本 cp 改过的桶到 writtenBuckets ... */
    if (!writtenBuckets.isEmpty() && isFullCompactedIdentifier(checkpointId, deltaCommits))
        waitCompaction = true;                                               // 到触发点
    if (waitCompaction) submitFullCompaction(checkpointId);                  // 对所有 writtenBuckets 提交
    return super.prepareCommit(waitCompaction, checkpointId);
}
```
**陷阱**：这一次 cp 会因等 full compaction 而变慢，timeout 要配足（§1.4）。

**LookupSinkWrite——为什么恢复时要 compact**：lookup changelog 在 compaction 时查旧值来生成。failover 后必须对所有曾活跃的桶重新触发 compact（构造函数里从 `activeBuckets` 状态读出后逐个 `write.compact(...)`），否则会漏 changelog。

> changelog producer 四种机制的存储层原理见 **01 号文档 §8.6** 与 **24 号文档**，这里只讲它在 Flink 写入器选型上的体现。

---

## 7. 数据写入核心路径：从 RowData 到 Level 0 文件

### 7.1 调用链总览

一条 `InternalRow` 从 Flink 写入算子穿到 LSM 内存缓冲，跨越 Flink 层（`StoreSinkWrite`）、core 表层（`TableWriteImpl`）、core 存储层（`FileStoreWrite` → `MergeTreeWriter`）：

```
RowDataStoreWriteOperator.processElement(InternalRow)        Flink 层
  → StoreSinkWriteImpl.write(row)                            Flink 层
    → TableWriteImpl.writeAndReturn(row)                     core 表层 ── §7.3
      ① checkNullability   非空约束
      ② wrapDefaultValue   填默认值
      ③ getRowKind         取 +I/-U/+U/-D（来自行或 rowkind-field）
      ④ rowKindFilter      行类型过滤（如 first-row 丢 UPDATE/DELETE）
      ⑤ toSinkRecord       KeyAndBucketExtractor 提取 partition/bucket/trimmedKey
      ⑥ recordExtractor    SinkRecord → KeyValue（嵌入 valueKind）
      → write.write(partition, bucket, keyValue)             core 存储层
        → AbstractFileStoreWrite.getWriterWrapper(...)       路由到具体 bucket 的 writer
          → MergeTreeWriter.write(kv)                        core 存储层 ── §7.4
            ① seq = newSequenceNumber()
            ② writeBuffer.put(seq, kind, key, value)         内存排序缓冲
            ③ 满则 flushWriteBuffer → 排序+merge → Level 0 文件
```

设计上把"业务校验/路由"（core 表层）和"LSM 写入"（core 存储层）分开：换 merge-engine 只动存储层的 `MergeFunction`，换计算引擎只动 Flink 层，互不影响。

### 7.2 FlinkRowWrapper：零拷贝跨类型系统

`mapToInternalRow` 把 Flink `RowData` 用 `FlinkRowWrapper` 包成 Paimon `InternalRow`（`FlinkSinkBuilder.java:260-278`）。它是**零拷贝代理**——所有 `getXxx()` 委托给底层 RowData，不深拷贝，高吞吐下省掉大量对象分配。代价：底层 RowData 在被引用期间不能被复用修改（Flink 算子链内默认满足）。RowKind 也在这里映射（`fromFlinkRowKind`：Flink 的 INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE 一一对应 Paimon 同名枚举）。

### 7.3 TableWriteImpl：六步把行变成 KeyValue

```java
// TableWriteImpl.java:182-216（精简）
public SinkRecord writeAndReturn(InternalRow row, int bucket) throws Exception {
    checkNullability(row);                                           // ① NOT NULL 列为 null 直接抛异常
    row = wrapDefaultValue(row);                                     // ② 用 fields.<f>.default-value 填 null
    RowKind rowKind = RowKindGenerator.getRowKind(rowKindGenerator, row);  // ③
    if (rowKindFilter != null && !rowKindFilter.test(rowKind)) return null; // ④ 被过滤的行直接丢弃
    SinkRecord record = bucket == -1 ? toSinkRecord(row) : toSinkRecord(row, bucket);  // ⑤
    write.write(record.partition(), record.bucket(), recordExtractor.extract(record, rowKind)); // ⑥
    return record;
}
private SinkRecord toSinkRecord(InternalRow row) {                  // KeyAndBucketExtractor 提取三元组
    keyAndBucketExtractor.setRecord(row);
    return new SinkRecord(keyAndBucketExtractor.partition(),
        keyAndBucketExtractor.bucket(), keyAndBucketExtractor.trimmedPrimaryKey(), row);
}
```

**陷阱**：`fields.<f>.default-value` 只在该字段为 null 时填充，不覆盖已有值；NOT NULL 列写 null 会在 ① 直接抛 `Cannot write null to non-null column`，不是静默丢弃。RowKind 的两种来源与各 merge-engine 处理见 §11。

### 7.4 MergeTreeWriter：写缓冲 + flush 预合并

**① 要解决什么问题**：每条记录直接写文件会爆小文件、且无法在写入侧预先合并同 key 的多次更新。

**② 设计原理**：先 put 进内存排序缓冲 `SortBufferWriteBuffer`，缓冲满才 flush；flush 时按 key 排序并用 `MergeFunction` 预合并，再落成 Level 0 文件——把"一个 key 写 N 次"压成"flush 时合并后写一次"。

```java
// MergeTreeWriter.java:163-174
public void write(KeyValue kv) throws Exception {
    long sequenceNumber = newSequenceNumber();                       // 单调递增，决定 merge 顺序
    boolean success = writeBuffer.put(sequenceNumber, kv.valueKind(), kv.key(), kv.value());
    if (!success) {                                                  // 缓冲满
        flushWriteBuffer(false, false);
        success = writeBuffer.put(sequenceNumber, kv.valueKind(), kv.key(), kv.value());
        if (!success) throw new RuntimeException("Mem table is too small to hold a single element.");
    }
}
```

flush 是写入与 compaction 的交汇点：
```java
// MergeTreeWriter.java:209-249（精简）
private void flushWriteBuffer(boolean waitForLatestCompaction, boolean forcedFullCompaction) {
    if (writeBuffer.size() > 0) {
        if (compactManager.shouldWaitForLatestCompaction()) waitForLatestCompaction = true;  // run 过多→反压（§10）
        // INPUT changelog-producer 时同时写 changelog 文件
        RollingFileWriter changelogWriter = changelogProducer == ChangelogProducer.INPUT
                ? writerFactory.createRollingChangelogFileWriter(0) : null;
        RollingFileWriter dataWriter = writerFactory.createRollingMergeTreeFileWriter(0, FileSource.APPEND);
        writeBuffer.forEach(keyComparator, mergeFunction,            // 排序 + MergeFunction 预合并
                changelogWriter == null ? null : changelogWriter::write, dataWriter::write);
        writeBuffer.clear();
        for (DataFileMeta f : dataWriter.result()) { newFiles.add(f); compactManager.addNewFile(f); } // 注册到 Levels
    }
    trySyncLatestCompaction(waitForLatestCompaction);                // 取回已完成的 compaction 结果
    compactManager.triggerCompaction(forcedFullCompaction);          // 触发新一轮异步 compaction
}
```

**③ 关键点**：`sequenceNumber` 单调递增，是 merge 正确性的基石——同 key 多版本乱序到达时靠它定先后（KeyValue 结构与 merge 语义见 **01 号文档 §3.4**）。Level 0 文件、SortBuffer 溢写、MergeFunction 等存储层细节见 **01 号文档 §4**。

**④ 风险**：`write-buffer-size` 太小 → 频繁 flush → 小文件多、compaction 压力大；太大 → 占内存、抢占其他 writer（§9）。INPUT 模式 flush 时双写 changelog，写放大约翻倍。



## 8. Checkpoint 两阶段提交与 exactly-once

### 8.1 为什么是两阶段，问题出在哪

**① 要解决什么问题**：N 个 writer 各写了一批文件，谁来"过账"成 Snapshot？如果 writer 各自提交，会出现"部分 writer 提交成功就被读到"的中间态；如果只在作业结束提交，流式作业永不结束。必须借 Flink 的 checkpoint 把"提交"这件事卡在一个**全局一致的时间点**上。

**② 设计原理（两阶段）**：
- **阶段一（barrier 前，写侧）**：barrier 流到每个 writer 时，Flink 回调 `prepareSnapshotPreBarrier`。writer 此刻 flush 缓冲、把这一轮产生的文件元数据打包成 `Committable` 发给下游 Committer，并把"未提交的 committable"存进自己的算子状态。**此时文件已落盘，但尚未对读可见**。
- **阶段二（cp 完成后，提交侧）**：当这一轮 cp 在所有算子都成功后，JobManager 回调 Committer 的 `notifyCheckpointComplete`，Committer 才真正 `commit` 成 Snapshot。

为什么提交必须放在 `notifyCheckpointComplete` 而不是 `snapshotState`？因为只有收到该回调，才能确定"这一轮 cp 已全局持久化"——此时提交，failover 时一定能从该 cp 恢复，提交与状态一致。若在 cp 完成前就提交，cp 一旦失败，已提交的 Snapshot 就与恢复点对不上，破坏 exactly-once。

```
JobManager      Writer×N (RowDataStoreWriteOperator)          Committer (并行度1)
   │ triggerCheckpoint(cpId)
   ├──────────────►│ prepareSnapshotPreBarrier(cpId)
   │               │   flush → prepareCommit(false,cpId) → drainIncrement
   │               │   emit Committable(cpId, CommitMessage) ──────────►│ processElement: inputs.add
   │               │ snapshotState: 存"未提交 committable"             │ snapshotState: 按 cp 分组存盘
   │ notifyCheckpointComplete(cpId)                                     │
   ├────────────────────────────────────────────────────────────────► │ commitUpToCheckpoint(cpId)
   │                                                                    │   StoreCommitter.commit → 冲突检测 → 原子 Snapshot
```

### 8.2 阶段一：writer 如何"备货发货"

发货入口在 `PrepareCommitOperator`（§5.2 已列），它调 `prepareCommit`，最终落到 `StoreSinkWriteImpl.prepareCommit`：

```java
// StoreSinkWriteImpl.java:136-151
public List<Committable> prepareCommit(boolean waitCompaction, long checkpointId) throws IOException {
    List<Committable> committables = new ArrayList<>();
    for (CommitMessage committable : write.prepareCommit(this.waitCompaction || waitCompaction, checkpointId))
        committables.add(new Committable(checkpointId, committable));   // 打上 cpId
    return committables;
}
```

`write.prepareCommit` 进存储层，对每个活跃 writer 调 `MergeTreeWriter.prepareCommit`（`MergeTreeWriter.java:251-267`）：先 `flushWriteBuffer`，再按需 `waitCompaction`，最后 `drainIncrement()` 把这一轮的六类文件增量（new/deleted/changelog + compactBefore/After/changelog）打包成 `CommitMessage`。注意这里有第二次 `shouldWaitForPreparingCheckpoint()` 判断——即便流式 `waitCompaction=false`，若 Level 0 文件堆太多也会强制等一等，避免反复 failover 把 L0 越堆越高。

**Committable / CommitMessage 结构**（理解提交内容）：
```
Committable { long checkpointId; CommitMessage commitMessage; }
CommitMessageImpl {
    BinaryRow partition; int bucket; int totalBuckets;
    DataIncrement    { newFiles; deletedFiles; changelogFiles; }       // 本轮写入
    CompactIncrement { compactBefore; compactAfter; changelogFiles; }  // 本轮 compaction
}
```

### 8.3 阶段二：Committer 如何"过账"

```java
// CommitterOperator.java:189-218
public void notifyCheckpointComplete(long checkpointId) throws Exception {
    super.notifyCheckpointComplete(checkpointId);
    commitUpToCheckpoint(endInput ? END_INPUT_CHECKPOINT_ID : checkpointId);
}
private void commitUpToCheckpoint(long checkpointId) throws Exception {
    NavigableMap<Long, GlobalCommitT> headMap = committablesPerCheckpoint.headMap(checkpointId, true);
    List<GlobalCommitT> committables = committables(headMap);
    if (committables.isEmpty() && committer.forceCreatingSnapshot())          // 即便空也可能要造 snapshot（如 watermark）
        committables = Collections.singletonList(toCommittables(checkpointId, Collections.emptyList()));
    if (checkpointId == END_INPUT_CHECKPOINT_ID)
        committer.filterAndCommit(committables, false, true);                 // 批结尾：幂等提交（§8.5/§12）
    else
        committer.commit(committables);                                       // 流式：直接提交
    headMap.clear();
}
```

`committer.commit` 进 `StoreCommitter`（`StoreCommitter.java:98-104`）→ `TableCommitImpl.commitMultiple` → `FileStoreCommitImpl.commit`：在这里做**冲突检测 + 原子重命名生成 Snapshot**。这一步的细节（要删的文件是否还在、key 区间是否冲突、冲突后从最新 snapshot 重扫重试）属于存储层提交，**详见 01 号文档 §13**，本文不展开。这里只强调它在两阶段提交里的位置：Committer 是唯一的提交者、串行执行，冲突只可能来自**其他作业**，因此重试逻辑简单可控。

### 8.4 exactly-once 的四道防线

单靠 Flink checkpoint 不足以保证"文件级"的不丢不重，Paimon 叠了四道防线：

| 防线 | 作用 | 源码 |
|------|------|------|
| ① 强制 EXACTLY_ONCE | AT_LEAST_ONCE 下同一 cp 可能被处理多次→重复，直接禁掉 | `FlinkSink.java:270` |
| ② commitUser 幂等标识 | 从算子状态恢复的作业唯一 ID，写进 Snapshot；提交前据此识别"是不是本作业已提交过" | `CommitterOperator.java:129` |
| ③ Committer 并行度 1 | 串行提交，杜绝多提交者竞争同一 snapshot 指针造成部分提交 | `FlinkWriteSink` 固定 |
| ④ 提交冲突检测 + 重试 | 提交前校验文件存在性与 key 区间，冲突则重扫重试 | `FileStoreCommitImpl`（见 01 §13） |

```java
// FlinkSink.java:270-271 —— 防线①
checkArgument(env.getCheckpointConfig().getCheckpointingMode() == CheckpointingMode.EXACTLY_ONCE,
    "Paimon sink currently only supports EXACTLY_ONCE checkpoint mode. ...");
// CommitterOperator.java:129-131 —— 防线②
commitUser = StateUtils.getSingleValueFromState(context, "commit_user_state", String.class, initialCommitUser);
```

### 8.5 故障恢复：为什么不丢也不重

- **Writer 崩溃**：从上一个 cp 恢复，writer 重放上轮数据写出**新文件**。崩溃前写的旧文件因为从未被任何成功的 Snapshot 引用，等同孤儿文件，后续被孤儿清理回收——**不重复**。已 emit 给 Committer 的 committable 存在 writer 自己的状态里，恢复后会重发——**不丢失**。
- **Committer 崩溃**：committable 按 cp 分组存在 Committer 状态（`snapshotState` 里 `pollInputs → groupByCheckpoint → CommittableStateManager.snapshotState`）。恢复后重放 `notifyCheckpointComplete`，凭 commitUser 幂等识别"这批是否已提交"，已提交则跳过——**不重复**；未提交则补提交——**不丢失**。
- **批作业 endInput 被重复调用**：新版 Flink 批作业 failover 可能从中间算子重启，导致 `endInput` 多次触发。所以 `END_INPUT_CHECKPOINT_ID` 走 `filterAndCommit`（先查 snapshot 是否已存在再决定提不提），而非无脑 commit（§12.3）。

### 8.6 风险 / 陷阱

- **checkpoint 超时**：阶段一可能因 flush + `waitCompaction`（full-compaction/lookup/批模式）很慢，timeout 配太短会让 cp 反复失败、作业卡死。timeout ≥ 5 分钟。
- **commitUser 漂移**：默认 commitUser 含启动信息，作业以新 ID 重启时旧未提交数据无法被幂等识别，可能在极端时序下重复或残留。生产显式配 `commit.user`。
- **Committer 单点**：它只做元数据操作，通常不是瓶颈；但海量 committable（分区/桶极多）+ 频繁 cp 时，提交耗时会上升，需控制 cp 频率与分区数。

---

## 9. 共享内存池与抢占机制

### 9.1 为什么要共享池而非各自分配

**① 要解决什么问题**：一个 writer 算子实例可能同时持有成百上千个 `MergeTreeWriter`（每个活跃 partition+bucket 一个）。若每个 writer 独占 `write-buffer-size`（默认 256MB），1600 个 bucket 就要 400GB，必然 OOM。

**② 设计原理**：一个算子实例只建**一个** `MemoryPoolFactory`（在 `PrepareCommitOperator.setup` 里），所有 `MergeTreeWriter` 作为 `MemoryOwner` 从它分配 `MemorySegment`。**总内存 = write-buffer-size，与 writer 数量无关**。当池满而某 writer 还要内存时，不阻塞等待（会死锁），而是**抢占**：强制当前内存占用最大的另一个 writer flush 出内存。

一句话哲学：**总量恒定、按需流动、抢占最大者**——用一笔固定预算喂养任意多个写入器。

> 重要更正：`write-buffer-size` 是**整个算子实例所有 writer 共享的总量**，不是每个 writer 的配额。按"每 writer"理解会严重高估内存（§1.4 陷阱）。

### 9.2 两种内存来源

```java
// PrepareCommitOperator.java:67-90（精简）
if (options.get(SINK_USE_MANAGED_MEMORY)) {
    // Flink Managed Memory：从 TaskManager 托管内存段分配，受 Flink 统一管控，生产推荐
    memoryPool = new FlinkMemorySegmentPool(computeManagedMemory(this), memoryManager.getPageSize(), memoryAllocator);
} else {
    // 堆内存（默认）：简单，但不受 Flink 框架管控，配错可能 OOM
    memoryPool = new HeapMemorySegmentPool(coreOptions.writeBufferSize(), coreOptions.pageSize());
}
memoryPoolFactory = new MemoryPoolFactory(memoryPool);
```

### 9.3 抢占机制：选最大者 flush

`MemoryPoolFactory` 把内部池包成给每个 owner 用的 `OwnerMemoryPool`：分配 segment 时若池空，就触发抢占——遍历所有 owner，挑**当前内存占用最大的另一个 writer** 强制 flush，腾出内存后重试分配。

```java
// MemoryPoolFactory.java:73-93
private void preemptMemory(MemoryOwner owner) {
    long maxMemory = 0; MemoryOwner max = null;
    for (MemoryOwner other : owners) {
        // 关键：绝不抢占自己——边写边 flush 会破坏自身状态
        if (other != owner && other.memoryOccupancy() > maxMemory) {
            maxMemory = other.memoryOccupancy(); max = other;
        }
    }
    if (max != null) { max.flushMemory(); ++bufferPreemptCount; }  // bufferPreemptCount 是关键监控指标
}
```

被抢占者执行 `flushMemory`（`MergeTreeWriter.java:201-207`）：能部分释放则部分释放，否则整体 `flushWriteBuffer` 落盘。

**为什么抢占而非等待**：LSM 下若所有 writer 都阻塞等内存，谁也释放不了 → 死锁。抢占保证至少有一个 writer 能推进；选"最大者"是因为它积累最多、flush 收益最高。

### 9.4 内存生命周期与风险

```
PrepareCommitOperator.setup()        建 MemoryPoolFactory（整个算子实例一个）
  └─ 首次 write(part, bucket)         AbstractFileStoreWrite.getWriterWrapper → 新建 MergeTreeWriter
       └─ notifyNewWriter             writeBufferPool.notifyNewOwner(writer) → writer.setMemoryPool(OwnerMemoryPool)
  └─ write 数据                       writeBuffer.put 分配 segment；池空 → preemptMemory（别人 flush 让出）
  └─ prepareCommit                    所有 writer flush → segment 归还 innerPool
  └─ close                            释放全部内存
```

**④ 风险/陷阱**：
- **抢占风暴 → 小文件**：内存太紧、活跃 bucket 太多时，writer 互相抢占、被迫小批量 flush，Level 0 小文件激增、compaction 压力大。这是把 §1.4 该陷阱落到机制上的根因。监控 `bufferPreemptCount`，居高不下就调大 `write-buffer-size`、减少同时活跃 bucket、或开 `write-buffer-spillable`。
- **堆模式 OOM**：堆模式不受 Flink 框架管控，`write-buffer-size` > 实际可用堆会 OOM；生产用 `sink.use-managed-memory=true` 更安全。
- **page-size**：默认 64KB，一般不动；过大降低利用率、过小增加分配开销。

---

## 10. 异步 Compaction 与写入反压

> Compaction 的策略本身（UniversalCompaction 选 run、各级触发、record-level expire）由 **01 号文档 §8** 主讲。本节只讲**它如何嵌进 Flink 写入链路**：何时异步触发、何时反压写入与 checkpoint、flush 与 compaction 如何在 `MergeTreeWriter` 里协调。

### 10.1 异步触发：写入与合并并行

**① 要解决什么问题**：compaction 是重 IO 操作（读旧文件+排序合并+写新文件），若同步执行会卡住写入路径。**② 设计**：每次 flush 后 `compactManager.triggerCompaction()` 把任务提交到线程池异步跑，写入继续推进，只在"文件堆太多"时才同步等。

```java
// MergeTreeCompactManager.java:208 起 submitCompaction（精简）
private void submitCompaction(CompactUnit unit, boolean dropDelete) {
    CompactTask task = unit.fileRewrite()
        ? new FileRewriteCompactTask(rewriter, unit, dropDelete, metricsReporter)
        : new MergeTreeCompactTask(keyComparator, compactionFileSize, rewriter, unit, dropDelete,
                levels.maxLevel(), metricsReporter, compactDfSupplier, recordLevelExpire, forceRewriteAllFiles);
    taskFuture = executor.submit(task);                     // 异步
}
```

### 10.2 写入反压：三档阈值

**① 要解决什么问题**：compaction 异步跑，万一追不上写入速度，Level 0 的 sorted run 会无限堆积——读放大爆炸，且越堆越合不动，形成恶性循环。**② 设计（按 sorted run 数三档刹车）**：

```
sorted run 数量 ↑
  numSortedRunStopTrigger + 1  ── checkpoint 也被阻塞（shouldWaitForPreparingCheckpoint，MergeTreeWriter.prepareCommit）
  numSortedRunStopTrigger      ── 写入被阻塞，等 compaction（shouldWaitForLatestCompaction，flushWriteBuffer 内）
  numSortedRunCompactTrigger   ── 触发 compaction（strategy.pick），但不阻塞写入
```

三档的意义：先尽量异步消化（compactTrigger）→ 实在追不上就反压写入（stopTrigger）→ 连 checkpoint 都先等等（stopTrigger+1），避免反复 failover 把 L0 越堆越高。`shouldWaitForLatestCompaction` / `shouldWaitForPreparingCheckpoint` 的判定在 `MergeTreeCompactManager`，阈值默认值与三参数的连带影响见 **01 号文档 §1.3 陷阱 2/7**。

### 10.3 flush 与 compaction 的协调点

`MergeTreeWriter.flushWriteBuffer`（§7.4 已列）是二者唯一交汇点，每次都按序做四件事：① 若 `shouldWaitForLatestCompaction()` 则把本次 flush 升级为"等上一轮 compaction 完成"；② 写出 Level 0 文件；③ `addNewFile` 注册进 Levels；④ `trySyncLatestCompaction` 取回已完成结果、`triggerCompaction` 触发新一轮。

**④ 风险**：
- `num-sorted-run.stop-trigger` 太大 → L0 堆积、读放大；太小 → 频繁反压、写吞吐低。
- `full-compaction.delta-commits=1` → 每个 cp 都全量 compact，IO 爆炸（§6.2）。
- compaction 与写入争抢磁盘/CPU IO；线程数过少追不上、过多互相竞争。
- full-compaction 的全局同步由 `GlobalFullCompactionSinkWrite` 在 Flink 侧保证（§6.2），不是存储层自己决定的。

---

## 11. CDC / RowKind 在写入侧的处理

> CDC 数据源的接入与同步动作（MySQL/Kafka → Paimon、整库同步）由 **14 号文档**主讲。本节只讲 RowKind 在**写入链路里**怎么被取出、过滤、嵌进 KeyValue，以及各 merge-engine 的语义差异在写入侧的体现。

### 11.1 RowKind 在写入侧的四步

CDC 流里每条记录带 RowKind（+I/-U/+U/-D）。写入侧对它做四步处理（都在 §7.3 的 `writeAndReturn` 内）：

1. **取**：`RowKindGenerator.getRowKind`——优先从行自带的 RowKind 取；若配了 `rowkind-field` 则从该字段解析（兼容 Debezium/Canal 把操作类型放在字段里的格式）。
```java
// RowKindGenerator.java:66-68
public static RowKind getRowKind(@Nullable RowKindGenerator rowKindGenerator, InternalRow row) {
    return rowKindGenerator == null ? row.getRowKind() : rowKindGenerator.generate(row);
}
```
2. **滤**：`rowKindFilter.test(rowKind)`（`TableWriteImpl.java:187`）——按配置丢弃某些 RowKind，如 `first-row` 丢掉 UPDATE/DELETE 只留首次 INSERT。
3. **嵌**：`recordExtractor.extract` 把 RowKind 写进 `KeyValue.valueKind`（与 key/seq/value 一起）。
4. **合**：compaction/读取时 `MergeFunction` 据 `valueKind` 决定如何合并同 key 多版本。

### 11.2 各 merge-engine 对 RowKind 的处理差异

| merge-engine | INSERT | UPDATE_BEFORE | UPDATE_AFTER | DELETE | 写入侧要点 |
|---|---|---|---|---|---|
| **deduplicate**（默认） | 写入 | 忽略（upsert 过滤掉） | 覆盖旧值 | 删除 | 只需最新值，可过滤 UPDATE_BEFORE |
| **partial-update** | 写入 | 忽略 | 仅更新非 null 字段 | 可选（`partial-update.remove-record-on-delete`） | **null = 不更新该字段，不是更新为 null** |
| **aggregate** | 累加 | **撤回（需要！）** | 累加 | **撤回（需要！）** | 必须保留 UPDATE_BEFORE，否则聚合偏大 |
| **first-row** | 写入（仅首次） | 忽略 | 忽略 | 忽略 | UPDATE/DELETE 被 RowKindFilter 丢弃 |

这张表与 §3.1 的 ChangelogMode 协商是一体两面：协商决定**上游是否传** UPDATE_BEFORE，merge-engine 决定**收到后怎么用**。aggregate 必须两头都保留 UPDATE_BEFORE，所以 §3.1 把它列为不过滤的例外。各 merge-engine 的存储层合并算法见 **08 号文档**（PartialUpdate/Aggregation 主讲）。

**④ 常见事故**：
- aggregate 表把 changelog 配成 upsert/过滤了 UPDATE_BEFORE → 撤回失效，sum 越加越大。
- partial-update 用户期望"写 null 清空字段"，实际 null 被当作"不更新" → 字段没被清掉。
- first-row 表期望 UPDATE 生效 → 被静默忽略。

---

## 12. 批写入 vs 流写入的差异

**① 要解决什么问题**：同一套写入框架既要跑无界流（持续、靠 checkpoint 提交、exactly-once），又要跑有界批（一次性、靠 `endInput` 提交、可幂等重试）。如果流批分开实现，代码两套、行为不一致，且没法在同一张表上流式写当天 + 批式回填历史。

**② 设计原理**：流批共用所有 Sink/Writer/Committer，只用 `isBounded` 和 `streamingCheckpointEnabled` 两个开关分流行为。差异集中在三处——发货时机、`waitCompaction`、提交方式。

### 12.1 核心差异对比

| 维度 | 流写入 | 批写入 |
|---|---|---|
| 数据流 | 无界 | 有界 |
| Checkpoint | 必须开 EXACTLY_ONCE | 不需要 |
| 发货时机 | `prepareSnapshotPreBarrier`（barrier 前） | `endInput`（结尾） |
| `waitCompaction` | false（不等 compaction） | **true**（等 compaction 完，避免遗留小文件） |
| 提交触发 | `notifyCheckpointComplete` | `endInput` → `commitUpToCheckpoint(END_INPUT_CHECKPOINT_ID)` |
| 提交方式 | `commit`（直接） | `filterAndCommit`（幂等，先查 snapshot 存在性） |
| ignorePreviousFiles | false | true（仅 OVERWRITE 时） |
| 内存抢占 | 持续发生 | 较少 |
| Clustering | 不支持 | 支持（BUCKET_UNAWARE） |
| OVERWRITE | 禁止（抛异常） | 支持 |
| POSTPONE_MODE | `PostponeBucketSink` | 桶数已知时 `PostponeFixedBucketSink` |

### 12.2 提交路径差异

```
流式：prepareSnapshotPreBarrier(cpId) → emitCommittables(false, cpId)  // 不等 compaction
      notifyCheckpointComplete(cpId)  → commitUpToCheckpoint(cpId) → StoreCommitter.commit

批式：endInput() → emitCommittables(true, Long.MAX_VALUE)              // 等 compaction
      CommitterOperator.endInput() → if (!streamingCheckpointEnabled)
        commitUpToCheckpoint(END_INPUT_CHECKPOINT_ID) → StoreCommitter.filterAndCommit  // 幂等
```

`streamingCheckpointEnabled` 标志的作用：启用了 checkpoint 的流作业在 `notifyCheckpointComplete` 提交；否则（含批作业）在 `endInput` 提交。

### 12.3 批写入幂等检查

```java
// CommitterOperator.java:205-213
if (checkpointId == END_INPUT_CHECKPOINT_ID) {
    // 批作业重启后可能再次 endInput，需要检查 snapshot 是否已存在
    committer.filterAndCommit(committables, false, true);
} else {
    committer.commit(committables);
}
```

**为什么批写入需要幂等检查**: Flink 新版本中，批作业失败后可能从中间算子重启（而不是从头），导致 `endInput()` 被多次调用。`filterAndCommit` 先检查对应 snapshot 是否已存在——存在则跳过、不存在才提交——避免重复写入（与 §8.5 的故障恢复呼应）。

### 12.4 AppendTableSink 的流式额外拓扑

```java
// AppendTableSink 流式场景（精简）
if (enableCompaction && isStreamingMode) {
    written.transform("Compact Coordinator: " + table.name(), ...)  // 单并行度协调器
           .forceNonParallel()
           .transform("Compact Worker: " + table.name(), ...);      // 多并行度 worker
}
```

**为什么追加表需要单独的 compaction 拓扑**: 追加表（BUCKET_UNAWARE）没有固定的 partition+bucket → writer 映射，无法像主键表那样在 writer 内部就地 compaction。所以单独挂一个**协调器（单并行度）+ worker（多并行度）**拓扑，由协调器统一调度跨 writer 的 compaction 任务。这是 §4.2 中 BUCKET_UNAWARE "需独立 compact 拓扑"的落地。



---

## 13. 与 Iceberg Flink Sink 的对比

**一句话差异**：Paimon 在写入路径里用 LSM 内存缓冲 + 异步 compaction，把更新做成"追加新版本、后台合并"；Iceberg 直接写文件，更新靠 Delete File（MoR）或重写整文件（CoW）。这导致两者的 Flink Sink 拓扑、内存模型、更新代价都不同。

**Sink 拓扑对比**：
```
Paimon:   RowData → mapToInternalRow → [LocalMerge] → [Partitioner] → Writer ↔ Compaction(异步) → Committer(1)
Iceberg:  RowData → IcebergStreamWriter → [Equality Delete Writer] → IcebergFilesCommitter
```
关键区别：Paimon 有内存写缓冲、共享内存池抢占、writer 内异步 compaction、动态桶两次 shuffle；Iceberg 直写文件、无跨 writer 内存抢占、compaction 靠独立作业/maintenance API。两者的 checkpoint 两阶段提交骨架相似（都 `prepareSnapshotPreBarrier` + `notifyCheckpointComplete` + 原子提交），但 Paimon 的提交还叠了 commitUser 幂等 + 冲突重试（§8.4）。

**整体权衡**：

| 维度 | Paimon (LSM) | Iceberg (Delete File) |
|---|---|---|
| 写入/更新性能 | 优（内存缓冲 + 追加） | 中（MoR）/ 差（CoW 重写整文件） |
| 读取性能 | 中（需 merge sorted run） | 优（CoW 零开销）/ 中（MoR apply delete） |
| compaction | writer 内异步自动 + 反压 | 独立作业 / maintenance |
| 小文件 | 自动收敛 | 需手动治理 |
| 跨分区 upsert | 支持（KEY_DYNAMIC） | 不原生支持 |
| 选型倾向 | 高频实时更新、CDC、流批一体 | 低频批量覆盖、读多写少、已有 Iceberg 生态 |



### 13.1 架构与机制对比

| 维度 | Paimon | Iceberg |
|---|---|---|
| **存储引擎** | LSM-tree (Merge-tree) | Copy-on-Write / Merge-on-Read |
| **数据组织** | Partition → Bucket → Sorted Runs → Data Files | Partition → Data Files（无 bucket） |
| **更新机制** | LSM merge（写入 + 后台 compaction） | 行级 Delete File（Equality / Position） |
| **数据分区 shuffle** | partition+bucket hash；动态桶两次 shuffle | 通常按 partition 或不分区 |
| **写入缓冲** | SortBufferWriteBuffer 内存排序 | 直写文件，无缓冲 |
| **内存管理** | MemoryPoolFactory 共享池 + 抢占 | 依赖 JVM 堆，无跨 writer 抢占 |
| **compaction** | writer 内异步 + 反压 | 独立作业 / maintenance |
| **本地预合并** | LocalMergeOperator（可选） | 无 |
| **CDC** | 原生 RowKind → MergeFunction | Equality Delete + Upsert |
| **exactly-once** | commitUser + cpId + 冲突重试 | Flink 快照协议 + commit 幂等 |

### 13.2 写入路径与更新代价

```
Paimon: InternalRow → KeyAndBucketExtractor → FileStoreWrite → MergeTreeWriter
        → SortBufferWriteBuffer.put（内存排序）→ flush merge → DataFileMeta → 异步 compaction
        按 bucket 隔离独立 LSM、写缓冲预合并、流式友好

Iceberg: RowData → PartitionedFanoutWriter → BaseTaskWriter → AppendWriter 直写
        →（有主键）EqualityDeleteWriter 写 delete file → close flush → DataFile
        直写无缓冲、写后不合并、批式优先
```

| 操作 | Paimon | Iceberg (MoR) | Iceberg (CoW) |
|---|---|---|---|
| INSERT | 写 SortBuffer → flush L0 | 写新 Data File | 写新 Data File |
| UPDATE | 写新 KV → compaction 合并 | Equality Delete + 新文件 | 重写整文件 |
| DELETE | 写 DELETE 标记 → compaction 清除 | Position/Equality Delete | 重写整文件 |
| 读取代价 | merge 多 sorted run | apply delete files | 无 |
| 写入代价 | 低（只追加） | 中 | 高 |

### 13.3 选型建议

| 场景 | 推荐 | 原因 |
|---|---|---|
| 高频实时更新 / CDC | Paimon | LSM 写入优异、原生 CDC |
| 低频批量覆盖、读多写少 | Iceberg | CoW 读取零开销 |
| 流批一体 Lakehouse | Paimon | 统一流批读写、实时 changelog |
| 跨分区 Upsert | Paimon | KEY_DYNAMIC 支持，Iceberg 不原生支持 |
| 已有 Iceberg 生态 | Iceberg | 兼容、工具链成熟 |

---

## 14. 设计决策总结

| 决策点 | 选择 | 取舍 / 代价 | 收益 |
|---|---|---|---|
| 写入与提交分离 | Writer 多并行度备货 + Committer 单并行度过账 | Committer 单点 | 原子可见、冲突可控、exactly-once 只实现一遍 |
| 提交时机 | `notifyCheckpointComplete` 才提交 | 提交延后一个 cp | 提交与恢复点一致，failover 不丢不重 |
| exactly-once 兜底 | 强制 EXACTLY_ONCE + commitUser 幂等 + 单 Committer + 冲突重试 | 禁用 AT_LEAST_ONCE、配置门槛 | 文件级不丢不重 |
| BucketMode 自动推导 | 由 bucket 数 + 主键/分区关系推导 | 跨分区需 bootstrap、动态桶两次 shuffle | 用户只配桶数；正确路由 |
| LSM 写缓冲 + 预合并 | SortBufferWriteBuffer flush 时 merge | 占内存、增写入延迟 | 高吞吐、降写放大与小文件 |
| 共享内存池 + 抢占 | 全 writer 共享 write-buffer-size，满则抢占最大者 flush | 抢占可能产小文件 | 固定预算撑海量 bucket，避免 OOM/死锁 |
| 异步 compaction + 三档反压 | 后台合并，run 超阈值才反压写入/checkpoint | run 多时反压、读放大 | 写入不被 compaction 阻塞 |
| LocalMerge（可选） | shuffle 前本地预合并 | 多一层算子、占内存 | 削主键热点、减 shuffle 量 |
| ChangelogMode 协商 | 主键表默认过滤 UPDATE_BEFORE | aggregate 等需例外保留 | 减半更新数据传输 |
| 零拷贝 FlinkRowWrapper | 代理 RowData 不深拷贝 | 引用期不可复用底层行 | 高吞吐省 CPU/内存 |
| FullCompaction 全局同步 | `GlobalFullCompactionSinkWrite` 按 deltaCommits 统一触发 | 那一次 cp 变慢 | 产出时间线一致的完整 changelog |
| 批写幂等 | `endInput` 走 `filterAndCommit` | 多一次 snapshot 存在性检查 | 批作业中途重启不重复提交 |

---

## 附录 A: 核心源码文件索引

**注**: 以下源码路径基于 `paimon-flink-common` 模块。对于特定 Flink 版本（1.16-2.2），相应的版本特定模块（如 `paimon-flink-1.16`、`paimon-flink-2.0` 等）会继承这些类并进行版本适配。

| 文件 | 路径 | 职责 |
|---|---|---|
| FlinkSinkBuilder | `paimon-flink/paimon-flink-common/.../flink/sink/FlinkSinkBuilder.java` | Sink 构建入口，BucketMode 路由 |
| FlinkSink | `paimon-flink/paimon-flink-common/.../flink/sink/FlinkSink.java` | 抽象 Sink 基类，doWrite/doCommit |
| FlinkWriteSink | `paimon-flink/paimon-flink-common/.../flink/sink/FlinkWriteSink.java` | 提供 Committer/StateManager 创建 |
| FlinkTableSinkBase | `paimon-flink/paimon-flink-common/.../flink/sink/FlinkTableSinkBase.java` | DynamicTableSink 实现，SQL 入口 |
| FixedBucketSink | `paimon-flink/paimon-flink-common/.../flink/sink/FixedBucketSink.java` | 固定桶 Sink |
| RowDynamicBucketSink | `paimon-flink/paimon-flink-common/.../flink/sink/RowDynamicBucketSink.java` | 动态桶 Sink |
| GlobalDynamicBucketSink | `paimon-flink/paimon-flink-common/.../flink/sink/index/GlobalDynamicBucketSink.java` | 全局索引动态桶 Sink |
| RowAppendTableSink | `paimon-flink/paimon-flink-common/.../flink/sink/RowAppendTableSink.java` | 追加表 Sink |
| PostponeBucketSink | `paimon-flink/paimon-flink-common/.../flink/sink/PostponeBucketSink.java` | 延迟桶 Sink |
| PrepareCommitOperator | `paimon-flink/paimon-flink-common/.../flink/sink/PrepareCommitOperator.java` | Checkpoint 前刷数据 |
| TableWriteOperator | `paimon-flink/paimon-flink-common/.../flink/sink/TableWriteOperator.java` | Writer 算子基类 |
| RowDataStoreWriteOperator | `paimon-flink/paimon-flink-common/.../flink/sink/RowDataStoreWriteOperator.java` | 行数据写入算子 |
| CommitterOperator | `paimon-flink/paimon-flink-common/.../flink/sink/CommitterOperator.java` | 提交算子 |
| StoreCommitter | `paimon-flink/paimon-flink-common/.../flink/sink/StoreCommitter.java` | Committer 实现 |
| LocalMergeOperator | `paimon-flink/paimon-flink-common/.../flink/sink/LocalMergeOperator.java` | 本地预合并算子 |
| FlinkStreamPartitioner | `paimon-flink/paimon-flink-common/.../flink/sink/FlinkStreamPartitioner.java` | 数据分区器 |
| FlinkRowWrapper | `paimon-flink/paimon-flink-common/.../flink/FlinkRowWrapper.java` | Flink→Paimon 行转换 |
| StoreSinkWrite | `paimon-flink/paimon-flink-common/.../flink/sink/StoreSinkWrite.java` | 写入策略接口+工厂 |
| StoreSinkWriteImpl | `paimon-flink/paimon-flink-common/.../flink/sink/StoreSinkWriteImpl.java` | 默认写入实现 |
| GlobalFullCompactionSinkWrite | `paimon-flink/paimon-flink-common/.../flink/sink/GlobalFullCompactionSinkWrite.java` | 全局 Full Compaction 写入 |
| LookupSinkWrite | `paimon-flink/paimon-flink-common/.../flink/sink/LookupSinkWrite.java` | Lookup 变更日志写入 |
| TableWriteImpl | `paimon-core/.../table/sink/TableWriteImpl.java` | 写入核心实现 |
| AbstractFileStoreWrite | `paimon-core/.../operation/AbstractFileStoreWrite.java` | 文件存储写入基类 |
| MemoryFileStoreWrite | `paimon-core/.../operation/MemoryFileStoreWrite.java` | 共享内存写入基类 |
| KeyValueFileStoreWrite | `paimon-core/.../operation/KeyValueFileStoreWrite.java` | KV 文件写入实现 |
| MergeTreeWriter | `paimon-core/.../mergetree/MergeTreeWriter.java` | LSM-tree 写入器 |
| SortBufferWriteBuffer | `paimon-core/.../mergetree/SortBufferWriteBuffer.java` | 内存排序缓冲区 |
| MemoryPoolFactory | `paimon-core/.../memory/MemoryPoolFactory.java` | 内存池工厂 |
| MergeTreeCompactManager | `paimon-core/.../mergetree/compact/MergeTreeCompactManager.java` | Compaction 管理器 |
| FileStoreCommitImpl | `paimon-core/.../operation/FileStoreCommitImpl.java` | 原子提交实现 |
| RowKindGenerator | `paimon-core/.../table/sink/RowKindGenerator.java` | RowKind 生成器 |
| BucketMode | `paimon-common/.../table/BucketMode.java` | 桶模式枚举 |
