# Flink 写入 Paimon 源码流程深度分析

> 基于 Apache Paimon 1.5-SNAPSHOT (commit: 55f4fd175)
> 分析日期: 2026-04-22
> Flink 版本支持: 1.16 ~ 2.2

---

## 目录

- [1. 全局架构概览](#1-全局架构概览)
- [2. Flink SQL INSERT INTO 完整链路](#2-flink-sql-insert-into-完整链路)
- [3. FlinkSinkBuilder 的 BucketMode 分发策略](#3-flinksinkbuilder-的-bucketmode-分发策略)
- [4. Writer 算子拓扑图](#4-writer-算子拓扑图)
- [5. StoreSinkWrite 策略分派](#5-storesinkwrite-策略分派)
- [6. 数据写入核心路径](#6-数据写入核心路径)
- [7. Checkpoint 触发的提交流程](#7-checkpoint-触发的提交流程)
- [8. 内存管理](#8-内存管理)
- [9. 异步 Compaction](#9-异步-compaction)
- [10. CDC 写入场景](#10-cdc-写入场景)
- [11. 批写入 vs 流写入的差异](#11-批写入-vs-流写入的差异)
- [12. 与 Iceberg Flink Sink 写入流程的详细对比](#12-与-iceberg-flink-sink-写入流程的详细对比)

---

## 1. 全局架构概览

### 解决什么问题

**核心业务问题**: 如何将 Flink 的流式/批式数据高效、可靠地写入到 Paimon 表中,同时保证 Exactly-Once 语义和高吞吐量?

**没有这个设计的后果**:
- 数据丢失或重复: 没有统一的写入框架,无法保证故障恢复时的数据一致性
- 性能低下: 直接写入文件系统会产生大量小文件,严重影响读取性能
- 无法支持更新: 追加式写入无法处理 CDC 场景的 UPDATE/DELETE 操作
- 资源浪费: 没有内存管理和反压机制,容易 OOM 或资源利用率低

**实际场景**:
```sql
-- 场景1: 实时数仓 - 从 Kafka 同步 MySQL binlog 到 Paimon
INSERT INTO paimon_orders 
SELECT * FROM kafka_cdc_source;

-- 场景2: 批量 ETL - 每天凌晨全量覆盖分区
INSERT OVERWRITE paimon_dim_user PARTITION (dt='2026-04-22')
SELECT * FROM hive_user_snapshot;

-- 场景3: 流批一体 - 同一张表既支持实时写入又支持批量回填
-- 流式作业持续写入
INSERT INTO paimon_events SELECT * FROM kafka_stream;
-- 批作业回填历史数据
INSERT INTO paimon_events SELECT * FROM hive_history WHERE dt < '2026-01-01';
```

### 有什么坑

**误区陷阱**:
1. **误以为 Paimon 写入不需要 Checkpoint**: 流式写入必须开启 EXACTLY_ONCE checkpoint,否则会抛异常
2. **忽略 BucketMode 的选择**: 错误的 bucket 配置会导致数据倾斜或写入性能差
   - `bucket=1` 导致所有数据写入单个 bucket,完全串行化
   - `bucket=-1` 但主键包含分区键时,应该用 HASH_DYNAMIC 而非 KEY_DYNAMIC
3. **LocalMerge 配置不当**: `local-merge-buffer-size` 设置过大导致 OOM,过小则失去预合并效果

**错误配置示例**:
```sql
-- 错误1: 流式作业未开启 checkpoint
SET 'execution.checkpointing.interval' = '0';  -- 错误!会导致写入失败

-- 错误2: 主键表设置 bucket=0
CREATE TABLE t (id INT PRIMARY KEY NOT ENFORCED, name STRING) 
WITH ('bucket' = '0');  -- 错误!主键表不支持 BUCKET_UNAWARE

-- 错误3: 动态桶表设置了固定并行度
CREATE TABLE t (id INT PRIMARY KEY NOT ENFORCED, name STRING) 
WITH ('bucket' = '-1', 'sink.parallelism' = '1');  
-- 性能差!动态桶的优势在于自动扩展,固定并行度限制了扩展能力
```

**生产环境注意事项**:
1. **Checkpoint 间隔**: 建议 1-5 分钟,过短会增加 HDFS 压力,过长会导致故障恢复时间长
2. **内存配置**: `write-buffer-size` 默认 256MB,高吞吐场景建议调整到 512MB-1GB
3. **Compaction 配置**: 默认异步 compaction 可能跟不上写入速度,需要监控 `sorted-run` 数量
4. **并行度设置**: Writer 并行度应该 >= bucket 数量,否则会有空闲 subtask

**性能陷阱**:
1. **热点分区**: 如果按日期分区且只写当天数据,会导致只有少数 writer 工作
2. **小文件问题**: 高频 checkpoint + 低吞吐会产生大量小文件,需要调整 `target-file-size`
3. **内存抢占风暴**: 多个 partition+bucket 同时写入时,频繁的内存抢占会降低吞吐量

### 核心概念解释

**术语定义**:
- **BucketMode**: 数据分桶策略,决定了数据如何分布到不同的 bucket 中
  - `HASH_FIXED`: 固定桶数,按 hash(partition+bucket) 分区
  - `HASH_DYNAMIC`: 动态桶,先按 key hash 分配 bucket,再按 partition+bucket 分区
  - `KEY_DYNAMIC`: 全局索引动态桶,支持跨分区 upsert
  - `BUCKET_UNAWARE`: 无桶模式,仅用于追加表
  - `POSTPONE_MODE`: 延迟桶分配,首次写入时动态决定桶数

- **Committable**: 写入阶段产生的待提交数据,包含新增/删除的文件元数据
- **CommitUser**: 每个 Flink 作业的唯一标识,用于幂等性检查
- **Snapshot**: Paimon 的版本快照,类似 Git commit,记录某个时间点的完整表状态

**概念关系**:
```
Table
  ├── Partition (分区,如 dt=2026-04-22)
  │     ├── Bucket 0 (桶,数据分片单元)
  │     │     ├── Level 0 (LSM-tree 层级)
  │     │     │     ├── data-file-1.parquet
  │     │     │     └── data-file-2.parquet
  │     │     └── Level 1
  │     │           └── data-file-3.parquet (compaction 后的合并文件)
  │     └── Bucket 1
  └── Snapshot (版本快照)
        ├── ManifestList (清单列表)
        │     ├── manifest-1 (记录 Partition A 的文件变更)
        │     └── manifest-2 (记录 Partition B 的文件变更)
        └── Schema (表结构版本)
```

**与其他系统对比**:
- **vs Hive**: Hive 直接写 Parquet/ORC 文件,无 LSM-tree 结构,不支持更新
- **vs Iceberg**: Iceberg 用 Delete File 实现更新,Paimon 用 LSM merge,写入性能更优
- **vs Delta Lake**: Delta Lake 用事务日志 + Parquet,Paimon 的 Snapshot 机制更轻量

### 设计理念

**为什么这样设计**:
1. **分层架构**: FlinkSink → StoreSinkWrite → TableWrite → FileStoreWrite → MergeTreeWriter
   - 每层职责清晰: Flink 层处理算子拓扑,Table 层处理业务逻辑,FileStore 层处理存储
   - 易于扩展: 新增 MergeEngine 只需修改 TableWrite 层,不影响 Flink 集成

2. **模板方法模式**: FlinkSink 定义 `doWrite → doCommit` 骨架,子类实现具体策略
   - 好处: 新增 BucketMode 只需继承 FlinkSink,不需修改核心提交逻辑
   - 例如: FixedBucketSink / DynamicBucketSink 共享相同的 CommitterOperator

3. **LSM-tree 存储引擎**: 写入先进入内存 SortBuffer,定期 flush 到 Level 0,后台异步 compaction
   - 写入性能优异: 内存写入 + 批量 flush,避免随机 IO
   - 支持高频更新: 同一 key 的多次更新在 compaction 时合并,减少存储空间
   - 读写分离: 写入不阻塞读取,compaction 在后台进行

**权衡取舍**:
- **写放大 vs 读放大**: LSM-tree 牺牲了一定的读性能(需要 merge 多个 sorted run),换取极致的写入性能
- **内存 vs 磁盘**: SortBuffer 占用内存,但避免了频繁的磁盘 IO,适合流式场景
- **一致性 vs 可用性**: 强制 EXACTLY_ONCE checkpoint,牺牲了一定的灵活性,保证了数据一致性

**架构演进**:
1. **早期版本**: 只支持 HASH_FIXED 模式,bucket 数量固定,无法动态扩展
2. **引入动态桶**: 支持 HASH_DYNAMIC,自动分配 bucket,解决了数据倾斜问题
3. **全局索引**: 支持 KEY_DYNAMIC,实现跨分区 upsert,满足复杂业务需求
4. **延迟分配**: 支持 POSTPONE_MODE,首次写入时动态决定桶数,兼顾灵活性和性能

**业界对比**:
- **Flink + Iceberg**: 写入路径更简单,但更新性能较差,需要独立的 compaction 作业
- **Flink + Hudi**: 支持 MoR/CoW 两种模式,但架构复杂,运维成本高
- **Flink + Paimon**: 原生 LSM-tree,写入性能最优,流批一体支持最好

### 1.1 写入链路全景图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Flink SQL / DataStream API                         │
│                          INSERT INTO paimon_table ...                            │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │   FlinkTableSinkBase           │
                    │   getSinkRuntimeProvider()      │
                    │   → FlinkSinkBuilder.build()    │
                    └───────────────┬───────────────┘
                                    │
           ┌────────────────────────▼─────────────────────────┐
           │            FlinkSinkBuilder.build()               │
           │  1. mapToInternalRow (RowData → InternalRow)      │
           │  2. LocalMergeOperator (可选, 主键表+开启配置)      │
           │  3. 根据 BucketMode 选择 Sink 类型                │
           └────────────────────────┬─────────────────────────┘
                                    │
  ┌──────────────┬──────────────────┼──────────────┬──────────────────┐
  ▼              ▼                  ▼              ▼                  ▼
HASH_FIXED   HASH_DYNAMIC    KEY_DYNAMIC    BUCKET_UNAWARE    POSTPONE_MODE
  │              │                  │              │                  │
  ▼              ▼                  ▼              ▼                  ▼
FixedBucket  RowDynamic      GlobalDynamic  RowAppendTable    PostponeBucket
   Sink      BucketSink      BucketSink       Sink               Sink
  │              │                  │              │                  │
  └──────────────┴──────────────────┼──────────────┴──────────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │   FlinkSink.sinkFrom()         │
                    │   doWrite() → doCommit()       │
                    └───────────────┬───────────────┘
                                    │
              ┌─────────────────────▼─────────────────────┐
              │  Writer 算子 (PrepareCommitOperator)        │
              │  → TableWriteOperator                      │
              │    → RowDataStoreWriteOperator              │
              │      → StoreSinkWrite.write(InternalRow)    │
              │        → TableWriteImpl.writeAndReturn()    │
              │          → FileStoreWrite.write()           │
              │            → MergeTreeWriter.write()        │
              └─────────────────────┬─────────────────────┘
                                    │
              ┌─────────────────────▼─────────────────────┐
              │  Committer 算子 (CommitterOperator)         │
              │  → StoreCommitter                          │
              │    → FileStoreCommitImpl                   │
              │      → 原子写入 Snapshot                    │
              └───────────────────────────────────────────┘
```

### 1.2 核心类继承层次

```
FlinkSink<T> (abstract)
  └── FlinkWriteSink<T> (abstract) ─── 提供 StoreCommitter/CommittableStateManager 创建
        ├── FixedBucketSink                     ← HASH_FIXED
        ├── PostponeBucketSink                  ← POSTPONE_MODE (流式/首次批)
        ├── PostponeFixedBucketSink             ← POSTPONE_MODE (已知桶数批写)
        ├── DynamicBucketSink<T> (abstract)     ← 动态桶基类
        │     ├── RowDynamicBucketSink          ← HASH_DYNAMIC
        │     └── DynamicBucketCompactSink      ← HASH_DYNAMIC (compact sink)
        ├── GlobalDynamicBucketSink             ← KEY_DYNAMIC
        └── AppendTableSink<T> (abstract)       ← BUCKET_UNAWARE 基类
              └── RowAppendTableSink            ← BUCKET_UNAWARE
```

**为什么这样设计**: 通过模板方法模式，`FlinkSink` 定义了写入流程骨架 (`doWrite → doCommit`)，各子类只需关注自己的 `createWriteOperatorFactory()` 和数据分区策略，实现了开闭原则。

**好处**: 新增 BucketMode 只需新增 Sink 子类和对应的分区器，不需修改核心写入/提交逻辑。

---

## 2. Flink SQL INSERT INTO 完整链路

### 解决什么问题

**核心业务问题**: 如何将 Flink SQL 的 INSERT 语句无缝对接到 Paimon 的写入框架,让用户无需关心底层实现细节?

**没有这个设计的后果**:
- 用户需要手动编写 DataStream 代码,学习成本高
- SQL 和 DataStream API 行为不一致,容易出错
- 无法利用 Flink SQL 的优化器(谓词下推、列裁剪等)

**实际场景**:
```sql
-- 场景1: 简单的 INSERT INTO
INSERT INTO paimon_table SELECT * FROM source_table;

-- 场景2: INSERT OVERWRITE 覆盖分区
INSERT OVERWRITE paimon_table PARTITION (dt='2026-04-22')
SELECT id, name, age FROM source_table WHERE dt='2026-04-22';

-- 场景3: 多表 JOIN 后写入
INSERT INTO paimon_result
SELECT a.id, a.name, b.amount
FROM paimon_user a
JOIN paimon_order b ON a.id = b.user_id;
```

### 有什么坑

**误区陷阱**:
1. **流式作业使用 INSERT OVERWRITE**: 会直接抛异常,OVERWRITE 只支持批模式
2. **忽略 ChangelogMode 协商**: 如果上游算子产生 UPDATE_BEFORE,但 Sink 不接受,Flink 会自动插入 SinkMaterializer,影响性能
3. **FormatTable 的特殊处理**: 日志存储表(FormatTable)走的是完全不同的写入路径,不支持主键和 compaction

**错误配置示例**:
```sql
-- 错误1: 流式作业使用 OVERWRITE
SET 'execution.runtime-mode' = 'streaming';
INSERT OVERWRITE paimon_table SELECT * FROM kafka_source;
-- 报错: Paimon doesn't support streaming INSERT OVERWRITE

-- 错误2: 主键表未开启 changelog-producer,但期望读取 changelog
CREATE TABLE t (id INT PRIMARY KEY NOT ENFORCED, name STRING);
-- 默认 changelog-producer=none,读取时无法获取完整的 +I/-U/+U/-D 数据
```

**生产环境注意事项**:
1. **Catalog 配置**: 确保 Flink SQL 能正确识别 Paimon Catalog,否则会被当作普通文件系统表
2. **并行度设置**: `sink.parallelism` 优先级高于全局并行度,建议显式配置
3. **Clustering 优化**: 仅在批模式 + BUCKET_UNAWARE 时生效,流式作业配置无效

**性能陷阱**:
1. **SinkMaterializer 开销**: 如果 ChangelogMode 协商失败,Flink 会插入物化算子,增加内存和 CPU 开销
2. **Clustering 排序开销**: `clustering.columns` 配置不当会导致全局排序,严重影响性能

### 核心概念解释

**术语定义**:
- **DynamicTableSink**: Flink Table API 的 Sink 接口,负责将逻辑表转换为物理执行计划
- **SinkRuntimeProvider**: 提供实际的 DataStream Sink 实现,是 DynamicTableSink 和 DataStream API 的桥梁
- **ChangelogMode**: 描述数据流包含哪些 RowKind(INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE)
- **SinkMaterializer**: Flink 自动插入的物化算子,用于将 changelog 流转换为 upsert 流

**概念关系**:
```
Flink SQL Parser
  ↓ 解析 SQL 文本
Logical Plan (逻辑计划)
  ↓ 优化器优化
Physical Plan (物理计划)
  ↓ 识别目标表
FlinkCatalog.getTable()
  ↓ 返回 FileStoreTable
FlinkTableSinkBase (DynamicTableSink 实现)
  ↓ getSinkRuntimeProvider()
PaimonDataStreamSinkProvider
  ↓ consumeDataStream()
FlinkSinkBuilder.build()
  ↓ 构建 DataStream 拓扑
FlinkSink.sinkFrom()
```

**与其他系统对比**:
- **Hive**: 通过 HiveOutputFormat 写入,不支持流式
- **Iceberg**: 通过 IcebergTableSink 实现,架构类似但细节不同
- **Kafka**: 通过 KafkaSink 写入,无需 Catalog,配置更简单

### 设计理念

**为什么这样设计**:
1. **分层解耦**: SQL 层(FlinkTableSinkBase) → DataStream 层(FlinkSinkBuilder) → 存储层(FileStoreTable)
   - SQL 层只负责协商 ChangelogMode 和构建 SinkRuntimeProvider
   - DataStream 层负责算子拓扑构建和数据分区
   - 存储层负责实际的文件写入和提交

2. **ChangelogMode 协商机制**: 让 Sink 告诉 Planner 自己能接受哪些 RowKind
   - 避免不必要的 UPDATE_BEFORE 传输,减少网络开销
   - 对于 AGGREGATE merge engine,保留 UPDATE_BEFORE 用于撤回
   - 对于 INPUT changelog producer,保留全部 RowKind 用于完整 changelog

3. **延迟构建**: getSinkRuntimeProvider 返回的是 Lambda,真正的 Sink 构建发生在 DataStream 执行时
   - 好处: 可以根据运行时信息(如 isBounded)动态调整策略
   - 例如: 批模式自动启用 Clustering 优化

**权衡取舍**:
- **灵活性 vs 复杂性**: ChangelogMode 协商增加了复杂性,但避免了不必要的数据传输
- **通用性 vs 性能**: 统一的 DynamicTableSink 接口保证了通用性,但某些场景(如 FormatTable)需要特殊处理

**架构演进**:
1. **早期版本**: 直接实现 Flink 的 OutputFormat,只支持批模式
2. **引入 DynamicTableSink**: 支持流批一体,统一 SQL 和 DataStream API
3. **ChangelogMode 协商**: 优化 CDC 场景的性能,减少不必要的数据传输
4. **Clustering 优化**: 支持批模式的数据排序,提升查询性能

**业界对比**:
- **Iceberg**: 也实现了 DynamicTableSink,但 ChangelogMode 协商逻辑不同
- **Hudi**: 通过 HoodieTableSink 实现,支持更复杂的 ChangelogMode(如 CDC 格式)
- **Delta Lake**: Flink 集成较弱,主要通过 Spark 写入

### 2.1 从 SQL 解析到 Sink 构建

```
用户执行: INSERT INTO paimon_table SELECT ...

           ┌─────────────────────────┐
           │  Flink SQL Parser        │  SQL 文本解析为逻辑计划
           └────────────┬────────────┘
                        ▼
           ┌─────────────────────────┐
           │  Flink SQL Planner       │  逻辑优化 → 物理计划
           │  识别目标表为 Paimon      │  通过 Catalog 解析到 PaimonTable
           └────────────┬────────────┘
                        ▼
           ┌─────────────────────────────────────────────┐
           │  FlinkCatalog                                │
           │  通过 SPI 加载 Paimon CatalogFactory         │
           │  getTable() 返回 FileStoreTable              │
           └────────────┬────────────────────────────────┘
                        ▼
           ┌─────────────────────────────────────────────┐
           │  FlinkTableSinkBase                          │
           │  实现 DynamicTableSink 接口                   │
           │  getSinkRuntimeProvider() 返回                │
           │  PaimonDataStreamSinkProvider                │
           └────────────┬────────────────────────────────┘
                        ▼
           ┌─────────────────────────────────────────────┐
           │  PaimonDataStreamSinkProvider.consumeDataStream()│
           │  调用 FlinkSinkBuilder.build()                │
           └─────────────────────────────────────────────┘
```

**源码路径**:
- `FlinkTableSinkBase.getSinkRuntimeProvider()` — `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/sink/FlinkTableSinkBase.java:103`
- `FlinkSinkBuilder.build()` — `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/sink/FlinkSinkBuilder.java:207`

### 2.2 FlinkTableSinkBase.getSinkRuntimeProvider 关键逻辑

```java
// FlinkTableSinkBase.java:103-146
public SinkRuntimeProvider getSinkRuntimeProvider(Context context) {
    // 1. 流式不支持 INSERT OVERWRITE
    if (overwrite && !context.isBounded()) {
        throw new UnsupportedOperationException("Paimon doesn't support streaming INSERT OVERWRITE.");
    }
    String name = tableIdentifier.asSummaryString();
    
    // 2. FormatTable 特殊处理（日志存储表）
    if (table instanceof FormatTable) {
        return new PaimonDataStreamSinkProvider(
            (dataStream) -> new FlinkFormatTableDataStreamSink(formatTable, overwrite, staticPartitions)
                    .sinkFrom(dataStream),
            name, table);
    }
    
    // 3. 创建 PaimonDataStreamSinkProvider，Lambda 内构建 FlinkSinkBuilder
    Options conf = Options.fromMap(table.options());
    return new PaimonDataStreamSinkProvider(
        (dataStream) -> {
            FlinkSinkBuilder builder = createSinkBuilder();
            builder.forRowData(dataStream);
            // 4. 可选：Clustering 排序优化（仅批模式 + BUCKET_UNAWARE）
            if (!conf.get(CLUSTERING_INCREMENTAL) || conf.get(CLUSTERING_INCREMENTAL_OPTIMIZE_WRITE)) {
                builder.clusteringIfPossible(
                    conf.get(CLUSTERING_COLUMNS),
                    conf.get(CLUSTERING_STRATEGY),
                    conf.get(CLUSTERING_SORT_IN_CLUSTER),
                    conf.get(CLUSTERING_SAMPLE_FACTOR));
            }
            // 5. 可选：INSERT OVERWRITE
            if (overwrite) builder.overwrite(staticPartitions);
            // 6. 可选：自定义并行度
            conf.getOptional(SINK_PARALLELISM).ifPresent(builder::parallelism);
            return builder.build();
        }, name, table);
}
```

### 2.3 ChangelogMode 协商

**为什么需要 ChangelogMode 协商**: Flink Planner 需要知道 Sink 能接收哪些 RowKind，以决定是否需要在 Sink 前插入 SinkMaterializer 做 upsert 物化。

```java
// FlinkTableSinkBase.java:71-100
public ChangelogMode getChangelogMode(ChangelogMode requestedMode) {
    if (table.primaryKeys().isEmpty()) {
        // 无主键表：接受所有 RowKind
        return requestedMode;
    } else {
        Options options = Options.fromMap(table.options());
        
        // 特殊情况 1: INPUT changelog producer，保留全部 RowKind
        if (options.get(CHANGELOG_PRODUCER) == ChangelogProducer.INPUT) {
            return requestedMode;
        }

        // 特殊情况 2: AGGREGATE merge engine，需要 UPDATE_BEFORE 做撤回
        if (options.get(MERGE_ENGINE) == MergeEngine.AGGREGATE) {
            return requestedMode;
        }

        // 特殊情况 3: PARTIAL_UPDATE + 定义了聚合函数，保留全部 RowKind
        if (options.get(MERGE_ENGINE) == MergeEngine.PARTIAL_UPDATE
                && new CoreOptions(options).definedAggFunc()) {
            return requestedMode;
        }

        // 主键表默认 Upsert 模式：过滤掉 UPDATE_BEFORE
        ChangelogMode.Builder builder = ChangelogMode.newBuilder();
        for (RowKind kind : requestedMode.getContainedKinds()) {
            if (kind != RowKind.UPDATE_BEFORE) {
                builder.addContainedKind(kind);
            }
        }
        return builder.build();
    }
}
```

**好处**: 对于 deduplicate/partial-update 等只需 INSERT+UPDATE_AFTER+DELETE 的场景，避免了不必要的 UPDATE_BEFORE 传输，减少数据量。

---

## 3. FlinkSinkBuilder 的 BucketMode 分发策略

### 解决什么问题

**核心业务问题**: 如何根据表的配置(bucket 数量、主键、分区键)自动选择最优的数据分区和写入策略?

**没有这个设计的后果**:
- 数据倾斜: 所有数据写入单个 bucket,无法并行
- 跨分区更新失败: 主键不包含分区键时,无法正确路由更新操作
- 资源浪费: 固定桶数无法适应数据量变化,要么桶太少(倾斜),要么桶太多(空闲)
- 小文件问题: 追加表没有合理的分区策略,产生大量小文件

**实际场景**:
```sql
-- 场景1: 固定桶主键表 (HASH_FIXED)
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY NOT ENFORCED,
    user_id BIGINT,
    amount DECIMAL(10,2)
) WITH ('bucket' = '16');  -- 16 个固定桶

-- 场景2: 动态桶主键表 (HASH_DYNAMIC)
CREATE TABLE user_profile (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    name STRING,
    age INT
) WITH ('bucket' = '-1');  -- 动态桶,自动扩展

-- 场景3: 跨分区更新 (KEY_DYNAMIC)
CREATE TABLE user_behavior (
    user_id BIGINT,
    event_time TIMESTAMP,
    action STRING,
    PRIMARY KEY (user_id, event_time) NOT ENFORCED
) PARTITIONED BY (dt STRING)
WITH ('bucket' = '-1');  -- 主键不包含分区键,需要全局索引

-- 场景4: 追加表 (BUCKET_UNAWARE)
CREATE TABLE logs (
    log_time TIMESTAMP,
    level STRING,
    message STRING
) WITH ('bucket' = '0');  -- 无桶,适合日志追加

-- 场景5: 延迟分配 (POSTPONE_MODE)
CREATE TABLE dynamic_table (
    id BIGINT PRIMARY KEY NOT ENFORCED,
    data STRING
) WITH ('bucket' = '-2');  -- 首次写入时动态决定桶数
```

### 有什么坑

**误区陷阱**:
1. **混淆 HASH_DYNAMIC 和 KEY_DYNAMIC**: 
   - HASH_DYNAMIC: 主键包含全部分区键,只需在分区内分配 bucket
   - KEY_DYNAMIC: 主键不包含分区键,需要全局索引支持跨分区 upsert
   - 错误使用会导致数据重复或更新失败

2. **BUCKET_UNAWARE 用于主键表**: 会直接抛异常,主键表必须有 bucket

3. **动态桶表设置过小的并行度**: 
   ```sql
   WITH ('bucket' = '-1', 'sink.parallelism' = '1')
   -- 动态桶的优势在于自动扩展,单并行度完全浪费了这个特性
   ```

4. **POSTPONE_MODE 的两种行为**: 流式和批式行为不同,容易混淆
   - 流式: 使用 PostponeBucketSink,桶号在 writer 侧动态分配
   - 批式(已知桶数): 使用 PostponeFixedBucketSink,桶号在 shuffle 时分配

**错误配置示例**:
```sql
-- 错误1: 主键表使用 bucket=0
CREATE TABLE t (id INT PRIMARY KEY NOT ENFORCED, name STRING) 
WITH ('bucket' = '0');
-- 报错: Unaware bucket mode only works with append-only table

-- 错误2: 跨分区更新使用 HASH_DYNAMIC
CREATE TABLE t (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    event_time TIMESTAMP,
    data STRING
) PARTITIONED BY (dt STRING)
WITH ('bucket' = '-1');
-- 问题: 同一 user_id 的数据可能分布在不同分区,HASH_DYNAMIC 无法正确处理
-- 应该使用 bucket=-1 且让 Paimon 自动识别为 KEY_DYNAMIC

-- 错误3: 非分区表设置 partition-sink-strategy
CREATE TABLE t (id INT, name STRING) 
WITH ('bucket' = '0', 'sink.partition-strategy' = 'hash');
-- 无效配置,非分区表没有分区可以 hash
```

**生产环境注意事项**:
1. **固定桶数选择**: 建议设置为并行度的 1-2 倍,避免数据倾斜
2. **动态桶的 assigner 并行度**: 默认为 writer 并行度,高吞吐场景建议单独配置
3. **全局索引的 bootstrap 时间**: KEY_DYNAMIC 模式启动时需要扫描全表,大表可能需要数分钟
4. **POSTPONE_MODE 的适用场景**: 适合桶数不确定的场景,但性能略低于固定桶

**性能陷阱**:
1. **动态桶的两次 shuffle**: 先按 key hash,再按 partition+bucket,网络开销是固定桶的 2 倍
2. **全局索引的内存开销**: KEY_DYNAMIC 需要在 assigner 中维护全局索引(RocksDB),内存占用大
3. **BUCKET_UNAWARE 的小文件**: 无桶模式下每个 writer 独立写文件,容易产生小文件

### 核心概念解释

**术语定义**:
- **Bucket**: 数据分片单元,类似 Hive 的 bucket,用于并行写入和读取
- **BucketMode**: 数据分桶策略,决定了如何将数据分配到不同的 bucket
- **Assigner**: 动态桶模式下的桶分配器,维护 key→bucket 的映射关系
- **Bootstrap**: 全局索引模式下的初始化阶段,扫描全表构建 key→partition+bucket 的索引

**BucketMode 详解**:
```
BucketMode 决策树:
  ├── bucket > 0  → HASH_FIXED (固定桶)
  ├── bucket = 0  → BUCKET_UNAWARE (无桶,仅追加表)
  ├── bucket = -1 → 
  │     ├── 主键包含全部分区键  → HASH_DYNAMIC (动态桶)
  │     └── 主键不包含分区键    → KEY_DYNAMIC (全局索引)
  └── bucket = -2 → POSTPONE_MODE (延迟分配)
```

**数据流对比**:
```
HASH_FIXED:
  input → partition by (partition+bucket) → writer → committer

HASH_DYNAMIC:
  input → partition by key → assigner → partition by (partition+bucket) → writer → committer

KEY_DYNAMIC:
  input → bootstrap index → partition by key → global-assigner → partition by bucket → writer → committer

BUCKET_UNAWARE:
  input → [partition by partition] → writer → committer → [compact coordinator]
```

**与其他系统对比**:
- **Hive**: 只支持固定桶(CLUSTERED BY),不支持动态桶
- **Iceberg**: 无 bucket 概念,通过 hidden partition 实现类似功能
- **Hudi**: 支持 BUCKET 和 CONSISTENT_HASHING 两种索引,类似 Paimon 的固定桶和动态桶

### 设计理念

**为什么这样设计**:
1. **多种 BucketMode 并存**: 不同场景有不同的最优策略
   - 固定桶: 简单高效,适合数据分布均匀的场景
   - 动态桶: 自动扩展,适合数据倾斜或数据量不确定的场景
   - 全局索引: 支持跨分区 upsert,适合复杂业务需求
   - 无桶: 最大化并行度,适合纯追加场景

2. **自动识别 BucketMode**: 根据表配置自动选择,用户无需显式指定
   - 好处: 降低使用门槛,避免配置错误
   - 例如: bucket=-1 时,根据主键是否包含分区键自动选择 HASH_DYNAMIC 或 KEY_DYNAMIC

3. **两次 shuffle 的必要性** (HASH_DYNAMIC):
   - 第一次 shuffle: 保证相同主键到同一个 assigner,避免同一 key 被分配到不同 bucket
   - 第二次 shuffle: 保证同一 partition+bucket 到同一个 writer,保证文件写入一致性
   - 虽然增加了网络开销,但保证了正确性

4. **LocalMerge 的位置**: 在第一次 shuffle 之前
   - 好处: 减少 shuffle 数据量,降低网络开销
   - 例如: 一条记录被连续更新 100 次,LocalMerge 只向下游发送最终值

**权衡取舍**:
- **灵活性 vs 性能**: 动态桶更灵活但性能略低(两次 shuffle),固定桶性能更优但不够灵活
- **功能 vs 复杂性**: 全局索引支持跨分区 upsert,但需要维护全局索引,增加了复杂性和内存开销
- **并行度 vs 小文件**: 无桶模式并行度最高,但容易产生小文件,需要额外的 compaction

**架构演进**:
1. **早期版本**: 只支持 HASH_FIXED,bucket 数量固定
2. **引入 HASH_DYNAMIC**: 支持动态桶,自动分配 bucket,解决数据倾斜
3. **引入 KEY_DYNAMIC**: 支持全局索引,实现跨分区 upsert
4. **引入 BUCKET_UNAWARE**: 支持无桶模式,最大化追加表的并行度
5. **引入 POSTPONE_MODE**: 支持延迟分配,兼顾灵活性和性能

**业界对比**:
- **Hudi 的 BUCKET 索引**: 类似 Paimon 的 HASH_FIXED,但不支持动态扩展
- **Hudi 的 CONSISTENT_HASHING**: 类似 Paimon 的 HASH_DYNAMIC,支持动态扩展
- **Iceberg 的 hidden partition**: 通过隐藏分区实现类似 bucket 的功能,但不支持主键表

### 3.1 核心分发代码

```java
// FlinkSinkBuilder.java:207-243
public DataStreamSink<?> build() {
    // Step 1: 自适应并行度冲突处理
    setParallelismIfAdaptiveConflict();
    input = trySortInput(input);
    
    // Step 2: RowData → InternalRow
    CatalogContext contextForDescriptor = BlobDescriptorUtils.getCatalogContext(...);
    DataStream<InternalRow> input = mapToInternalRow(this.input, table.rowType(), contextForDescriptor);

    // Step 3: 可选 LocalMerge 算子（主键表 + local-merge-buffer-size > 0）
    if (table.coreOptions().localMergeEnabled() && table.schema().primaryKeys().size() > 0) {
        SingleOutputStreamOperator<InternalRow> newInput =
            input.forward().transform("local merge", input.getType(), 
                new LocalMergeOperator.Factory(table.schema()));
        forwardParallelism(newInput, input);
        input = newInput;
    }

    // Step 4: 根据 BucketMode 路由到不同 Sink
    BucketMode bucketMode = table.bucketMode();
    switch (bucketMode) {
        case POSTPONE_MODE:   return buildPostponeBucketSink(input);
        case HASH_FIXED:      return buildForFixedBucket(input);
        case HASH_DYNAMIC:    return buildDynamicBucketSink(input, false);
        case KEY_DYNAMIC:     return buildDynamicBucketSink(input, true);
        case BUCKET_UNAWARE:  return buildUnawareBucketSink(input);
        default:
            throw new UnsupportedOperationException("Unsupported bucket mode: " + bucketMode);
    }
}
```

### 3.2 五种 BucketMode 详解

| BucketMode | 配置方式 | 适用表类型 | Sink 实现 | 分区策略 |
|---|---|---|---|---|
| `HASH_FIXED` | `bucket = N` (N>0) | 主键表/追加表 | `FixedBucketSink` | `FlinkStreamPartitioner` 按 partition+bucket hash |
| `HASH_DYNAMIC` | `bucket = -1` | 主键表 | `RowDynamicBucketSink` | 先按 key hash 分区到 assigner，再按 partition+bucket 分区到 writer |
| `KEY_DYNAMIC` | `bucket = -1` + 主键不包含全部分区键 | 主键表（跨分区） | `GlobalDynamicBucketSink` | 先 bootstrap 全量索引，再 assigner 分配 bucket |
| `BUCKET_UNAWARE` | `bucket = 0` | 追加表 | `RowAppendTableSink` | 可选按 partition hash 分区，或不分区 |
| `POSTPONE_MODE` | `bucket = -2` | 主键表 | `PostponeBucketSink` | 按 partition hash 或推迟桶号计算分区 |

### 3.3 各模式的 Sink 构建细节

#### HASH_FIXED 模式

```java
// FlinkSinkBuilder.java:272-287
protected DataStreamSink<?> buildForFixedBucket(DataStream<InternalRow> input) {
    int bucketNums = table.bucketSpec().getNumBuckets();
    // 优化：非分区表如果桶数 < 并行度，自动降低 writer 并行度
    if (parallelism == null && bucketNums < input.getParallelism() && table.partitionKeys().isEmpty()) {
        parallelism = bucketNums;
    }
    // 按 partition + bucket hash 分区
    DataStream<InternalRow> partitioned =
        partition(input, new RowDataChannelComputer(table.schema()), parallelism);
    return new FixedBucketSink(table, overwritePartition).sinkFrom(partitioned);
}
```

**为什么自动降低并行度**: 如果非分区表只有 4 个 bucket 但并行度是 128，则 124 个 writer 子任务永远不会收到数据，白白浪费资源。

#### HASH_DYNAMIC 模式

```java
// DynamicBucketSink.java:75-117
// 拓扑: input → shuffle by key hash → bucket-assigner → shuffle by partition+bucket → writer → committer
public DataStreamSink<?> build(DataStream<T> input, Integer parallelism) {
    // 1. 按 key hash 分区到 assigner
    DataStream<T> partitionByKeyHash = partition(input, assignerChannelComputer(numAssigners), ...);
    // 2. Assigner 算子：维护 key→bucket 映射（基于 hash index）
    SingleOutputStreamOperator<Tuple2<T, Integer>> bucketAssigned =
        partitionByKeyHash.transform("dynamic-bucket-assigner", ..., assignerOperator);
    // 3. 按 partition+bucket 二次分区到 writer
    DataStream<Tuple2<T, Integer>> partitionByBucket = partition(bucketAssigned, channelComputer2(), ...);
    // 4. 写入和提交
    return sinkFrom(partitionByBucket, initialCommitUser);
}
```

**为什么需要两次 shuffle**: 第一次 shuffle 保证相同主键的记录到同一个 assigner（避免同一主键被分配到不同 bucket），第二次 shuffle 保证同一 partition+bucket 的数据到同一个 writer（保证文件写入的一致性）。

#### KEY_DYNAMIC (Global Index) 模式

相比 `HASH_DYNAMIC` 多了一个 `INDEX_BOOTSTRAP` 阶段：在 assigner 之前需要读取表中已有的所有主键，构建全局索引（存储在 RocksDB 中），以支持跨分区 upsert。

```
input → INDEX_BOOTSTRAP → shuffle by key hash → cross-partition-bucket-assigner
      → shuffle by bucket → writer → committer
```

**为什么需要 bootstrap**: 跨分区 upsert 要求知道一个主键之前属于哪个分区+桶，因此必须在启动时加载全量索引。

#### POSTPONE_MODE 模式

```java
// FlinkSinkBuilder.java:289-315
private DataStreamSink<?> buildPostponeBucketSink(DataStream<InternalRow> input) {
    // 流式模式或未启用 postponeBatchWriteFixedBucket：使用 PostponeBucketSink
    if (isStreaming(input) || !table.coreOptions().postponeBatchWriteFixedBucket()) {
        ChannelComputer<InternalRow> channelComputer;
        if (!table.partitionKeys().isEmpty()
                && table.coreOptions().partitionSinkStrategy() == PartitionSinkStrategy.HASH) {
            channelComputer = new RowDataHashPartitionChannelComputer(table.schema());
        } else {
            channelComputer = new PostponeBucketChannelComputer(table.schema());
        }
        DataStream<InternalRow> partitioned = partition(input, channelComputer, parallelism);
        PostponeBucketSink sink = new PostponeBucketSink(table, overwritePartition);
        return sink.sinkFrom(partitioned);
    } else {
        // 批模式且启用 postponeBatchWriteFixedBucket：使用 PostponeFixedBucketSink
        // 这种情况下桶数已知，可以提前分配
        Map<BinaryRow, Integer> knownNumBuckets = PostponeUtils.getKnownNumBuckets(table);
        DataStream<InternalRow> partitioned =
                partition(input,
                    new PostponeFixedBucketChannelComputer(table.schema(), knownNumBuckets),
                    parallelism);

        FileStoreTable tableForWrite = PostponeUtils.tableForFixBucketWrite(table);
        PostponeFixedBucketSink sink =
                new PostponeFixedBucketSink(tableForWrite, overwritePartition, knownNumBuckets);
        return sink.sinkFrom(partitioned);
    }
}
```

**POSTPONE_MODE 的两种行为**:
1. **流式或首次批写**: 使用 `PostponeBucketSink`，桶号在 writer 侧动态分配
2. **已知桶数的批写**: 使用 `PostponeFixedBucketSink`，桶号在 shuffle 时提前分配，性能更优

#### BUCKET_UNAWARE 模式

```java
// FlinkSinkBuilder.java:317-332
private DataStreamSink<?> buildUnawareBucketSink(DataStream<InternalRow> input) {
    checkArgument(
        table.primaryKeys().isEmpty(),
        "Unaware bucket mode only works with append-only table for now.");

    // 可选：分区表+HASH策略时按分区 hash 分区
    if (!table.partitionKeys().isEmpty()
            && table.coreOptions().partitionSinkStrategy() == PartitionSinkStrategy.HASH) {
        input = partition(input, new RowDataHashPartitionChannelComputer(table.schema()), parallelism);
    }

    return new RowAppendTableSink(table, overwritePartition, parallelism).sinkFrom(input);
}
```

**为什么只支持追加表**: 无桶模式下没有确定性的数据路由，无法保证同一主键的更新操作发送到同一个文件写入器，因此不支持主键表。

---

## 4. Writer 算子拓扑图

### 4.1 算子继承层次

```
AbstractStreamOperator<Committable>
  └── PrepareCommitOperator<IN, Committable>    ← 内存管理 + Checkpoint 前刷数据
        └── TableWriteOperator<IN>              ← 状态管理 + StoreSinkWrite 持有
              └── RowDataStoreWriteOperator     ← processElement 调用 write.write(row)

PrepareCommitOperator.Factory<IN, Committable>
  └── TableWriteOperator.Factory<IN>
        └── RowDataStoreWriteOperator.Factory
  └── TableWriteOperator.CoordinatedFactory<IN>  ← 带 OperatorCoordinator
        └── RowDataStoreWriteOperator.CoordinatedFactory
```

### 4.2 完整的算子链 (以 HASH_FIXED 为例)

```
DataStream<RowData>
    │
    │ map (FlinkRowWrapper)
    ▼
DataStream<InternalRow>
    │
    │ forward() + transform("local merge")     ← 可选: LocalMergeOperator
    ▼
DataStream<InternalRow>
    │
    │ FlinkStreamPartitioner (按 partition+bucket hash)
    ▼
DataStream<InternalRow>  (重分区后)
    │
    │ transform("Writer : table_name")         ← RowDataStoreWriteOperator
    │   内部：StoreSinkWrite.write(row)
    │   Checkpoint 前：prepareSnapshotPreBarrier → emitCommittables
    ▼
DataStream<Committable>
    │
    │ [可选] transform("Changelog Compact Coordinator")  ← PRECOMMIT_COMPACT 时
    │ transform("Changelog Compact Worker")
    │ transform("Changelog Sort by Creation Time")
    ▼
DataStream<Committable>
    │
    │ transform("Global Committer : table_name")   ← CommitterOperator (parallelism=1)
    ▼
DataStream<Committable>
    │
    │ sinkTo(DiscardingSink)
    ▼
DataStreamSink
```

### 4.3 LocalMergeOperator 的作用

**源码路径**: `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/sink/LocalMergeOperator.java`

```java
// LocalMergeOperator.java:152-172
public void processElement(StreamRecord<InternalRow> record) throws Exception {
    InternalRow row = record.getValue();
    RowKind rowKind = RowKindGenerator.getRowKind(rowKindGenerator, row);
    // row kind 统一设为 INSERT (key-value 分离后 kind 存储在 value 中)
    row.setRowKind(RowKind.INSERT);
    BinaryRow key = keyProjection.apply(row);
    if (!merger.put(rowKind, key, row)) {
        flushBuffer();  // 缓冲区满，先 flush
        if (!merger.put(rowKind, key, row)) {
            row.setRowKind(rowKind);  // 恢复原始 RowKind
            output.collect(record);   // 直接下发
        }
    }
}

// prepareSnapshotPreBarrier 中 flush 缓冲区
public void prepareSnapshotPreBarrier(long checkpointId) throws Exception {
    if (!endOfInput) {
        flushBuffer();
    }
}

// endInput 时也要 flush
public void endInput() throws Exception {
    endOfInput = true;
    flushBuffer();
}
```

**为什么需要 LocalMerge**: 解决主键数据倾斜问题。在 shuffle 之前先在本地做预合并，减少网络传输量。例如一条记录被连续更新 100 次，LocalMerge 只向下游发送最终值。

**LocalMerger 的两种实现**:
1. **HashMapLocalMerger**: 用于所有非主键字段都是固定长度的情况（如 INT、LONG、DOUBLE）。使用 HashMap 存储 key→value 映射，内存高效。
2. **SortBufferLocalMerger**: 用于存在变长字段（如 STRING、ARRAY）的情况。使用排序缓冲区，支持溢写到磁盘。

**LocalMerger 的选择逻辑**:

```java
// LocalMergeOperator.java:108-145
boolean canHashMerger = true;
for (DataField field : valueType.getFields()) {
    if (primaryKeys.contains(field.name())) {
        continue;  // 跳过主键字段
    }
    if (!BinaryRow.isInFixedLengthPart(field.type())) {
        canHashMerger = false;  // 存在变长字段，不能用 HashMap
        break;
    }
}

if (canHashMerger) {
    merger = new HashMapLocalMerger(...);
} else {
    merger = new SortBufferLocalMerger(...);
}
```

**好处**:
1. 显著减少 shuffle 数据量，降低网络开销
2. 减轻下游 writer 的写放大
3. 自动选择最优的 merger 实现

### 4.4 FlinkStreamPartitioner 数据分区

**源码路径**: `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/sink/FlinkStreamPartitioner.java`

```java
// FlinkStreamPartitioner.java:49-51
public int selectChannel(SerializationDelegate<StreamRecord<T>> record) {
    return channelComputer.channel(record.getInstance().getValue());
}
```

`ChannelComputer` 的实现根据 BucketMode 不同：
- **RowDataChannelComputer**: 按 `partition + bucket` 做 hash 取模 → 固定桶模式
- **RowAssignerChannelComputer**: 按主键 hash 取模 → 动态桶 assigner 前的分区
- **RowWithBucketChannelComputer**: 按 `partition + bucket` 做 hash 取模 → 动态桶 writer 前的分区
- **RowDataHashPartitionChannelComputer**: 仅按 `partition` 做 hash → 无桶模式的分区分发

---

## 5. StoreSinkWrite 策略分派

### 5.1 工厂方法 createWriteProvider

**源码路径**: `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/sink/StoreSinkWrite.java:100-187`

```
StoreSinkWrite.createWriteProvider()
    │
    ├── writeOnly = true?  ──yes──→ StoreSinkWriteImpl (无 compaction)
    │
    ├── changelog-producer = FULL_COMPACTION 或配置了 full-compaction.delta-commits?
    │    ──yes──→ GlobalFullCompactionSinkWrite
    │
    ├── needLookup() = true (changelog-producer = LOOKUP)?
    │    ──yes──→ LookupSinkWrite
    │
    └── 默认 ──→ StoreSinkWriteImpl
```

**源码逻辑**:

```java
// StoreSinkWrite.java:100-187
static StoreSinkWrite.Provider createWriteProvider(
        FileStoreTable fileStoreTable,
        CheckpointConfig checkpointConfig,
        boolean isStreaming,
        boolean ignorePreviousFiles,
        boolean hasSinkMaterializer) {
    
    Options options = fileStoreTable.coreOptions().toConfiguration();
    CoreOptions.ChangelogProducer changelogProducer = fileStoreTable.coreOptions().changelogProducer();
    boolean waitCompaction;
    CoreOptions coreOptions = fileStoreTable.coreOptions();
    
    if (coreOptions.writeOnly()) {
        waitCompaction = false;
    } else {
        waitCompaction = coreOptions.prepareCommitWaitCompaction();
        int deltaCommits = -1;
        
        // 检查 full-compaction.delta-commits 或 changelog-producer-full-compaction-trigger-interval
        if (options.contains(FULL_COMPACTION_DELTA_COMMITS)) {
            deltaCommits = options.get(FULL_COMPACTION_DELTA_COMMITS);
        } else if (options.contains(CHANGELOG_PRODUCER_FULL_COMPACTION_TRIGGER_INTERVAL)) {
            long fullCompactionThresholdMs = options.get(CHANGELOG_PRODUCER_FULL_COMPACTION_TRIGGER_INTERVAL).toMillis();
            deltaCommits = (int)(fullCompactionThresholdMs / checkpointConfig.getCheckpointInterval());
        }

        // 如果配置了 FULL_COMPACTION 或 deltaCommits >= 0，使用 GlobalFullCompactionSinkWrite
        if (changelogProducer == CoreOptions.ChangelogProducer.FULL_COMPACTION || deltaCommits >= 0) {
            int finalDeltaCommits = Math.max(deltaCommits, 1);
            return (table, commitUser, state, ioManager, memoryPoolFactory, metricGroup) -> {
                assertNoSinkMaterializer.run();
                return new GlobalFullCompactionSinkWrite(
                        table, commitUser, state, ioManager, ignorePreviousFiles,
                        waitCompaction, finalDeltaCommits, isStreaming, memoryPoolFactory, metricGroup);
            };
        }

        // 如果需要 Lookup changelog producer，使用 LookupSinkWrite
        if (coreOptions.needLookup()) {
            return (table, commitUser, state, ioManager, memoryPoolFactory, metricGroup) -> {
                assertNoSinkMaterializer.run();
                return new LookupSinkWrite(
                        table, commitUser, state, ioManager, ignorePreviousFiles,
                        waitCompaction, isStreaming, memoryPoolFactory, metricGroup);
            };
        }
    }

    // 默认使用 StoreSinkWriteImpl
    return (table, commitUser, state, ioManager, memoryPoolFactory, metricGroup) -> {
        assertNoSinkMaterializer.run();
        return new StoreSinkWriteImpl(
                table, commitUser, state, ioManager, ignorePreviousFiles,
                waitCompaction, isStreaming, memoryPoolFactory, metricGroup);
    };
}

### 5.2 三种 StoreSinkWrite 实现对比

| 特性 | StoreSinkWriteImpl | GlobalFullCompactionSinkWrite | LookupSinkWrite |
|---|---|---|---|
| **继承关系** | 实现 StoreSinkWrite 接口 | 继承 StoreSinkWriteImpl | 继承 StoreSinkWriteImpl |
| **核心职责** | 基础写入 + prepareCommit | 定期触发全局 Full Compaction | 恢复时重新触发 compact |
| **状态管理** | 无额外状态 | 维护 `writtenBuckets` 状态 | 维护 `activeBuckets` 状态 |
| **适用场景** | 默认 / write-only | `changelog-producer = full-compaction` | `changelog-producer = lookup` |
| **Compaction 行为** | 由 CompactManager 自动触发 | 每隔 `deltaCommits` 个 checkpoint 强制 full compaction | 恢复时对所有活跃 bucket 触发 compact |

#### GlobalFullCompactionSinkWrite

**源码路径**: `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/sink/GlobalFullCompactionSinkWrite.java`

```java
// GlobalFullCompactionSinkWrite.java:149-184
public List<Committable> prepareCommit(boolean waitCompaction, long checkpointId)
        throws IOException {
    checkSuccessfulFullCompaction();  // 检查之前的 full compaction 是否成功
    
    // 记录本 checkpoint 期间修改的 bucket
    if (!currentWrittenBuckets.isEmpty()) {
        writtenBuckets.computeIfAbsent(checkpointId, k -> new HashSet<>())
                      .addAll(currentWrittenBuckets);
        currentWrittenBuckets.clear();
    }
    
    // 判断是否到达 full compaction 触发点
    // isFullCompactedIdentifier 检查 checkpointId 是否是 deltaCommits 的倍数
    if (!writtenBuckets.isEmpty() && isFullCompactedIdentifier(checkpointId, deltaCommits)) {
        waitCompaction = true;
    }
    
    if (waitCompaction) {
        submitFullCompaction(checkpointId);  // 对所有 writtenBuckets 提交 full compaction
        commitIdentifiersToCheck.add(checkpointId);
    }
    
    return super.prepareCommit(waitCompaction, checkpointId);
}
```

**为什么全局同步 Full Compaction**: 为了生成完整的 changelog。Full Compaction 会将所有 level 的数据合并，通过对比前后快照的 diff 生成 changelog。如果每个 writer 独立 compact，可能在不同时间点生成不一致的 changelog。

#### LookupSinkWrite

```java
// LookupSinkWrite.java:64-75 (构造函数中)
List<StoreSinkWriteState.StateValue> activeBucketsStateValues =
    state.get(tableName, ACTIVE_BUCKETS_STATE_NAME);
if (activeBucketsStateValues != null) {
    for (StoreSinkWriteState.StateValue stateValue : activeBucketsStateValues) {
        write.compact(stateValue.partition(), stateValue.bucket(), false);
    }
}
```

**为什么恢复时要触发 compact**: Lookup changelog producer 通过 compaction 时查找旧值来生成 changelog。恢复后需要重新触发 compact 以确保不遗漏任何 changelog 数据。

---

## 6. 数据写入核心路径

### 解决什么问题

**核心业务问题**: 如何将 Flink 的 RowData 高效地转换为 Paimon 的存储格式,并写入到 LSM-tree 中?

**没有这个设计的后果**:
- 类型不兼容: Flink 和 Paimon 的数据类型不一致,无法直接写入
- 性能低下: 每条记录都深拷贝会导致严重的性能问题
- 数据丢失: 没有非空约束检查和默认值填充,可能写入不完整的数据
- 写放大严重: 每条记录直接写文件,产生大量小文件

**实际场景**:
```java
// 场景1: CDC 数据写入
// Flink RowData: +I[1, "Alice", 25]  (INSERT)
// → InternalRow: +I[1, "Alice", 25]
// → KeyValue: {key=[1], value=[1,"Alice",25], valueKind=INSERT, seq=1}
// → SortBuffer 暂存
// → flush 时写入 data-file-1.parquet

// 场景2: 更新操作
// Flink RowData: +U[1, "Alice", 26]  (UPDATE_AFTER)
// → InternalRow: +U[1, "Alice", 26]
// → KeyValue: {key=[1], value=[1,"Alice",26], valueKind=UPDATE_AFTER, seq=2}
// → SortBuffer 中已有 seq=1 的旧值
// → flush 时 merge,只保留最新值

// 场景3: 删除操作
// Flink RowData: -D[1, "Alice", 26]  (DELETE)
// → InternalRow: -D[1, "Alice", 26]
// → KeyValue: {key=[1], value=[1,"Alice",26], valueKind=DELETE, seq=3}
// → compaction 时清除该 key 的所有历史版本
```

### 有什么坑

**误区陷阱**:
1. **误以为 FlinkRowWrapper 会深拷贝**: 实际上是零拷贝的代理模式,修改原始 RowData 会影响 InternalRow
2. **忽略非空约束**: 如果表定义了 NOT NULL 但数据中有 null,会在 checkNullability 时抛异常
3. **默认值配置错误**: `fields.{field_name}.default-value` 只在字段为 null 时生效,不会覆盖已有值
4. **RowKindFilter 配置不当**: 例如 first-row merge engine 配置了 rowkind-filter,可能导致更新被忽略

**错误配置示例**:
```sql
-- 错误1: 非空字段未提供默认值
CREATE TABLE t (
    id INT PRIMARY KEY NOT ENFORCED,
    name STRING NOT NULL,
    age INT
);
INSERT INTO t SELECT id, NULL, age FROM source;  -- 报错: name 不能为 null

-- 错误2: 默认值类型不匹配
CREATE TABLE t (
    id INT,
    create_time TIMESTAMP
) WITH ('fields.create_time.default-value' = 'now()');  -- 错误!应该是具体的时间戳字符串

-- 错误3: RowKindFilter 过滤了必要的 RowKind
CREATE TABLE t (
    id INT PRIMARY KEY NOT ENFORCED,
    name STRING
) WITH (
    'merge-engine' = 'deduplicate',
    'rowkind-filter' = 'insert'  -- 错误!会过滤掉 UPDATE 和 DELETE
);
```

**生产环境注意事项**:
1. **SortBuffer 大小**: `write-buffer-size` 默认 256MB,高吞吐场景建议调整到 512MB-1GB
2. **Flush 频率**: 由 SortBuffer 大小和 checkpoint 间隔共同决定,需要平衡内存和小文件
3. **MergeFunction 选择**: 不同的 merge-engine 有不同的 merge 逻辑,影响写入性能
4. **Sequence Number**: 单调递增的序列号保证了 merge 的正确性,不要手动修改

**性能陷阱**:
1. **频繁的 flush**: SortBuffer 过小导致频繁 flush,产生大量小文件
2. **Merge 开销**: 同一 key 的多次更新在 flush 时需要 merge,CPU 开销大
3. **序列化开销**: KeyValue 的二进制序列化是 CPU 密集型操作,影响吞吐量

### 核心概念解释

**术语定义**:
- **FlinkRowWrapper**: Flink RowData 到 Paimon InternalRow 的零拷贝适配器
- **SinkRecord**: 包含 partition、bucket、primaryKey 和完整行数据的中间结构
- **KeyValue**: LSM-tree 的存储单元,包含 key、value、valueKind 和 sequenceNumber
- **SortBuffer**: 内存排序缓冲区,暂存待写入的 KeyValue,flush 时排序并 merge
- **MergeFunction**: 定义如何合并同一 key 的多个 KeyValue,不同的 merge-engine 有不同的实现

**数据转换流程**:
```
Flink RowData (Flink 类型系统)
  ↓ FlinkRowWrapper (零拷贝适配)
InternalRow (Paimon 类型系统)
  ↓ checkNullability (非空检查)
  ↓ wrapDefaultValue (默认值填充)
  ↓ RowKindGenerator.getRowKind (提取 RowKind)
  ↓ rowKindFilter.test (RowKind 过滤)
  ↓ toSinkRecord (提取 partition/bucket/key)
SinkRecord {partition, bucket, primaryKey, row}
  ↓ recordExtractor.extract (SinkRecord → KeyValue)
KeyValue {key, value, valueKind, sequenceNumber}
  ↓ SortBuffer.put (暂存到内存)
  ↓ flush 时排序并 merge
  ↓ RollingFileWriter.write (写入 Parquet/ORC)
DataFileMeta (文件元数据)
```

**KeyValue 结构**:
```java
KeyValue {
    BinaryRow key;              // 主键(去除分区列)
    long sequenceNumber;        // 单调递增序列号,用于 merge 时排序
    RowKind valueKind;          // INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE
    BinaryRow value;            // 完整行数据(包含主键和非主键列)
}
```

**与其他系统对比**:
- **Hive**: 直接写 Parquet/ORC,无中间缓冲,无 merge 逻辑
- **Iceberg**: 写入时不做 merge,通过 Delete File 实现更新
- **Hudi**: 有类似的 HoodieRecord 结构,但 merge 逻辑在 compaction 时进行

### 设计理念

**为什么这样设计**:
1. **零拷贝适配器 (FlinkRowWrapper)**: 避免深拷贝带来的性能开销
   - 好处: 高吞吐场景下,零拷贝可以节省 30%-50% 的 CPU 和内存
   - 代价: 需要保证原始 RowData 在使用期间不被修改

2. **SortBuffer 预合并**: 在 flush 之前先在内存中 merge
   - 好处: 减少写入的数据量,降低 Level 0 文件大小,减轻后续 compaction 压力
   - 例如: 一条记录被连续更新 100 次,flush 时只写入最终值

3. **Sequence Number 机制**: 保证 merge 的正确性
   - 问题: 同一 key 的多个 KeyValue 可能乱序到达(网络延迟、重试等)
   - 解决: 通过单调递增的 sequenceNumber 排序,保证 merge 时按时间顺序处理
   - 实现: 每个 MergeTreeWriter 维护独立的 sequenceNumber 计数器

4. **分层的数据结构**: RowData → InternalRow → SinkRecord → KeyValue
   - RowData: Flink 层,包含 Flink 特有的类型和 RowKind
   - InternalRow: Paimon 通用层,与计算引擎无关
   - SinkRecord: 业务层,包含 partition/bucket/key 等业务信息
   - KeyValue: 存储层,LSM-tree 的存储单元

**权衡取舍**:
- **内存 vs 磁盘**: SortBuffer 占用内存,但避免了频繁的磁盘 IO
- **写入延迟 vs 吞吐量**: SortBuffer 增加了写入延迟(数据先暂存),但提升了吞吐量(批量写入)
- **CPU vs IO**: flush 时的 merge 消耗 CPU,但减少了磁盘 IO 和存储空间

**架构演进**:
1. **早期版本**: 直接写入文件,无 SortBuffer,写放大严重
2. **引入 SortBuffer**: 支持内存排序和预合并,显著降低写放大
3. **引入 Sequence Number**: 解决乱序问题,保证 merge 正确性
4. **引入 MergeFunction**: 支持多种 merge 策略(deduplicate/partial-update/aggregate)

**业界对比**:
- **RocksDB**: 也使用 MemTable(类似 SortBuffer) + Sequence Number 机制
- **HBase**: 使用 MemStore + MVCC 机制,思路类似但实现不同
- **Cassandra**: 使用 Memtable + Timestamp,不需要 Sequence Number(因为是分布式系统)

### 6.1 完整写入调用链

```
RowData (Flink 类型)
  │
  │ FlinkSinkBuilder.mapToInternalRow()
  │   new FlinkRowWrapper(rowData, catalogContext)
  ▼
InternalRow (Paimon 类型，FlinkRowWrapper 实现)
  │
  │ RowDataStoreWriteOperator.processElement()
  │   → write.write(row)  [StoreSinkWrite 接口]
  ▼
StoreSinkWriteImpl.write(InternalRow rowData)
  │   → write.writeAndReturn(rowData)  [TableWriteImpl]
  ▼
TableWriteImpl.writeAndReturn(InternalRow row, int bucket)
  │ ① checkNullability(row)         ← 非空约束检查
  │ ② wrapDefaultValue(row)         ← 默认值填充
  │ ③ RowKindGenerator.getRowKind() ← 获取 RowKind（CDC 行类型）
  │ ④ rowKindFilter.test(rowKind)   ← 行类型过滤
  │ ⑤ toSinkRecord(row)             ← 提取 partition / bucket / primaryKey
  │ ⑥ recordExtractor.extract()     ← SinkRecord → KeyValue
  │ ⑦ write.write(partition, bucket, keyValue)  [FileStoreWrite 接口]
  ▼
AbstractFileStoreWrite.write(BinaryRow partition, int bucket, KeyValue data)
  │   → getWriterWrapper(partition, bucket)
  │   → container.writer.write(data)  [RecordWriter 接口]
  ▼
MergeTreeWriter.write(KeyValue kv)
  │ ① sequenceNumber = newSequenceNumber()     ← 单调递增序列号
  │ ② success = writeBuffer.put(seq, kind, key, value)
  │ ③ if (!success) {
  │      flushWriteBuffer(false, false);       ← 缓冲区满，flush 到文件
  │      writeBuffer.put(seq, kind, key, value);
  │    }
  ▼
SortBufferWriteBuffer.put(long sequenceNumber, RowKind valueKind, InternalRow key, InternalRow value)
  │   → 序列化为 KeyValue 的二进制行格式
  │   → BinaryInMemorySortBuffer / BinaryExternalSortBuffer 存储
  ▼
(数据暂存在内存排序缓冲区中，等待 flush)
```

### 6.2 FlinkRowWrapper 类型转换

**源码路径**: `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/FlinkRowWrapper.java`

`FlinkRowWrapper` 实现了 Paimon 的 `InternalRow` 接口，内部包装了 Flink 的 `RowData`。核心是类型映射：

```java
// RowKind 映射: Flink → Paimon
public static RowKind fromFlinkRowKind(org.apache.flink.types.RowKind rowKind) {
    switch (rowKind) {
        case INSERT:        return RowKind.INSERT;
        case UPDATE_BEFORE: return RowKind.UPDATE_BEFORE;
        case UPDATE_AFTER:  return RowKind.UPDATE_AFTER;
        case DELETE:        return RowKind.DELETE;
    }
}
```

**为什么用 Wrapper 而不是深拷贝**: 零拷贝设计，避免在高吞吐场景下频繁的对象创建和内存分配。`FlinkRowWrapper` 只是一个代理，所有 `getXxx()` 调用都委托给底层 Flink RowData。

### 6.3 TableWriteImpl 的分区和桶提取

```java
// TableWriteImpl.java:178-193
public SinkRecord writeAndReturn(InternalRow row, int bucket) throws Exception {
    checkNullability(row);
    row = wrapDefaultValue(row);
    RowKind rowKind = RowKindGenerator.getRowKind(rowKindGenerator, row);
    if (rowKindFilter != null && !rowKindFilter.test(rowKind)) {
        return null;  // 被过滤的行类型直接跳过
    }
    SinkRecord record = bucket == -1 ? toSinkRecord(row) : toSinkRecord(row, bucket);
    write.write(record.partition(), record.bucket(), recordExtractor.extract(record, rowKind));
    return record;
}

// toSinkRecord: 通过 KeyAndBucketExtractor 提取
private SinkRecord toSinkRecord(InternalRow row) {
    keyAndBucketExtractor.setRecord(row);
    return new SinkRecord(
        keyAndBucketExtractor.partition(),     // BinaryRow: 分区值
        keyAndBucketExtractor.bucket(),        // int: 桶号
        keyAndBucketExtractor.trimmedPrimaryKey(), // BinaryRow: 去除分区列的主键
        row);
}
```

### 6.4 flushWriteBuffer → 数据文件

```java
// MergeTreeWriter.java:209-249
private void flushWriteBuffer(boolean waitForLatestCompaction, boolean forcedFullCompaction) throws Exception {
    if (writeBuffer.size() > 0) {
        if (compactManager.shouldWaitForLatestCompaction()) {
            waitForLatestCompaction = true;  // sorted run 数量超过阈值，必须等待
        }

        // 创建 changelog 文件写入器（仅 INPUT changelog-producer 模式）
        final RollingFileWriter<KeyValue, DataFileMeta> changelogWriter =
            changelogProducer == ChangelogProducer.INPUT
                ? writerFactory.createRollingChangelogFileWriter(0) : null;
        // 创建数据文件写入器
        final RollingFileWriter<KeyValue, DataFileMeta> dataWriter =
            writerFactory.createRollingMergeTreeFileWriter(0, FileSource.APPEND);

        // 按 key 排序并 merge 后写出
        writeBuffer.forEach(
            keyComparator, mergeFunction,
            changelogWriter == null ? null : changelogWriter::write,
            dataWriter::write);
        writeBuffer.clear();

        // 新文件注册到 CompactManager
        for (DataFileMeta fileMeta : dataWriter.result()) {
            newFiles.add(fileMeta);
            compactManager.addNewFile(fileMeta);
        }
    }

    // 尝试获取已完成的 compaction 结果
    trySyncLatestCompaction(waitForLatestCompaction);
    // 触发新的 compaction
    compactManager.triggerCompaction(forcedFullCompaction);
}
```

**为什么 flush 时要做 merge**: SortBufferWriteBuffer 中可能有同一 key 的多条记录（例如连续的 INSERT→UPDATE→DELETE），flush 时通过 MergeFunction 预合并，减少写出的数据量和后续 compaction 的工作量。

---

## 7. Checkpoint 触发的提交流程

### 解决什么问题

**核心业务问题**: 如何在分布式环境下保证 Exactly-Once 语义,确保数据不丢失、不重复?

**没有这个设计的后果**:
- 数据丢失: Writer 写入的文件未提交,故障恢复后丢失
- 数据重复: 同一批数据被提交多次,导致重复
- 不一致: 多个 Writer 的数据部分提交,表状态不一致
- 无法回滚: 提交失败后无法回滚到之前的一致性状态

**实际场景**:
```
场景1: 正常提交流程
  t0: Writer 写入数据到 SortBuffer
  t1: Checkpoint barrier 到达,触发 prepareSnapshotPreBarrier
  t2: Writer flush SortBuffer,生成 data-file-1.parquet
  t3: Writer emit Committable(cpId=1, files=[data-file-1.parquet])
  t4: Committer 收集所有 Writer 的 Committable
  t5: Checkpoint 完成,触发 notifyCheckpointComplete
  t6: Committer 原子写入 Snapshot,提交成功

场景2: Writer 故障恢复
  t0: Writer 写入数据,但在 checkpoint 前崩溃
  t1: Flink 从上一个 checkpoint 恢复
  t2: Writer 重新处理数据,生成新的文件
  t3: 旧文件(未提交)被忽略,不会造成数据重复

场景3: Committer 故障恢复
  t0: Writer 已 emit Committable,但 Committer 在提交前崩溃
  t1: Flink 从 checkpoint 恢复,Committer 从状态恢复 Committable
  t2: Committer 重新提交,幂等性检查确保不会重复提交
```

### 有什么坑

**误区陷阱**:
1. **误以为 prepareCommit 就是提交**: 实际上只是准备 Committable,真正的提交在 notifyCheckpointComplete
2. **忽略 Barrier 对齐**: 如果 Checkpoint 模式是 AT_LEAST_ONCE,会导致数据重复
3. **CommitUser 不一致**: 作业重启后 commitUser 变化,导致幂等性检查失效
4. **批作业的 endInput 重复调用**: Flink 新版本中,批作业失败后可能从中间算子重启,导致 endInput 被多次调用

**错误配置示例**:
```sql
-- 错误1: 使用 AT_LEAST_ONCE checkpoint
SET 'execution.checkpointing.mode' = 'at-least-once';
-- 报错: Paimon sink currently only supports EXACTLY_ONCE checkpoint mode

-- 错误2: Checkpoint 间隔过短
SET 'execution.checkpointing.interval' = '10s';
-- 问题: 频繁的 checkpoint 会增加 HDFS 压力,影响吞吐量
-- 建议: 1-5 分钟

-- 错误3: Checkpoint 超时时间过短
SET 'execution.checkpointing.timeout' = '30s';
-- 问题: 大表的 flush 和 compaction 可能超过 30 秒,导致 checkpoint 失败
-- 建议: 至少 5 分钟
```

**生产环境注意事项**:
1. **Checkpoint 间隔**: 建议 1-5 分钟,平衡故障恢复时间和系统开销
2. **Checkpoint 超时**: 建议至少 5 分钟,避免 compaction 导致超时
3. **CommitUser 配置**: 建议显式配置 `commit.user`,避免作业重启后变化
4. **状态后端**: 建议使用 RocksDB 状态后端,避免大状态导致 OOM

**性能陷阱**:
1. **频繁的 flush**: Checkpoint 间隔过短导致频繁 flush,产生大量小文件
2. **Compaction 阻塞 Checkpoint**: waitCompaction=true 时,Checkpoint 会等待 compaction 完成,可能超时
3. **Committer 单点瓶颈**: Committer 并行度为 1,大量 Committable 可能导致提交慢

### 核心概念解释

**术语定义**:
- **Checkpoint**: Flink 的分布式快照机制,保证 Exactly-Once 语义
- **Barrier**: Checkpoint 的标记,在数据流中传播,触发算子的快照
- **prepareSnapshotPreBarrier**: Checkpoint barrier 到达前的回调,用于 flush 数据
- **notifyCheckpointComplete**: Checkpoint 完成后的回调,用于提交数据
- **Committable**: Writer 准备提交的数据,包含文件元数据和 checkpoint ID
- **CommitUser**: 每个 Flink 作业的唯一标识,用于幂等性检查

**Checkpoint 流程**:
```
JobManager                    Writer (N个)                Committer (1个)
    │                             │                            │
    │ triggerCheckpoint(cpId)     │                            │
    │────────────────────────────>│                            │
    │                             │                            │
    │         prepareSnapshotPreBarrier(cpId)                  │
    │                             │                            │
    │                   flush SortBuffer                       │
    │                   emit Committable                       │
    │                             │───────────────────────────>│
    │                             │                            │
    │         snapshotState(cpId) │                            │
    │         (保存 Writer 状态)   │                            │
    │                             │                            │
    │                             │         snapshotState(cpId)│
    │                             │         (保存 Committable)  │
    │                             │                            │
    │ notifyCheckpointComplete(cpId)                           │
    │─────────────────────────────────────────────────────────>│
    │                             │                            │
    │                             │         commit()           │
    │                             │         (原子写入 Snapshot) │
    │                             │                            │
```

**Committable 结构**:
```java
Committable {
    long checkpointId;           // Checkpoint ID
    CommitMessage commitMessage; // 提交消息
}

CommitMessageImpl {
    BinaryRow partition;         // 分区
    int bucket;                  // 桶号
    DataIncrement {              // 数据文件变更
        List<DataFileMeta> newFiles;       // 新增文件
        List<DataFileMeta> deletedFiles;   // 删除文件
        List<DataFileMeta> changelogFiles; // Changelog 文件
    }
    CompactIncrement {           // Compaction 文件变更
        List<DataFileMeta> compactBefore;  // Compaction 输入
        List<DataFileMeta> compactAfter;   // Compaction 输出
        List<DataFileMeta> changelogFiles; // Compaction Changelog
    }
}
```

**与其他系统对比**:
- **Hive**: 无 Checkpoint 机制,依赖 HDFS 的原子 rename
- **Iceberg**: 也依赖 Flink Checkpoint,但提交逻辑不同(通过 Iceberg 的事务 API)
- **Hudi**: 支持 Flink Checkpoint,但也支持独立的 commit 机制

### 设计理念

**为什么这样设计**:
1. **两阶段提交 (2PC)**: prepareSnapshotPreBarrier + notifyCheckpointComplete
   - 第一阶段: Writer flush 数据,生成文件,但不提交
   - 第二阶段: Committer 收集所有文件,原子提交
   - 好处: 保证了分布式环境下的原子性,要么全部成功,要么全部失败

2. **Barrier 对齐机制**: 确保所有 Writer 在同一时间点 flush
   - 问题: 如果不对齐,不同 Writer 的数据可能属于不同的逻辑时间点
   - 解决: Flink 的 Barrier 对齐机制保证了所有 Writer 在同一 Barrier 到达时 flush
   - 代价: Barrier 对齐会增加延迟,但保证了一致性

3. **CommitUser 幂等性**: 通过 commitUser 避免重复提交
   - 问题: Committer 故障恢复后,可能重新提交已提交的数据
   - 解决: Snapshot 中记录 commitUser,提交前检查是否已提交
   - 实现: commitUser 从状态恢复,保证作业重启后不变

4. **Committer 单并行度**: 保证提交的原子性
   - 问题: 如果多个 Committer 并行提交,可能导致部分提交
   - 解决: Committer 并行度固定为 1,串行提交
   - 代价: Committer 可能成为瓶颈,但保证了一致性

**权衡取舍**:
- **一致性 vs 性能**: 强制 EXACTLY_ONCE checkpoint,牺牲了一定的性能,保证了一致性
- **延迟 vs 吞吐量**: Barrier 对齐增加了延迟,但保证了数据的一致性
- **单点 vs 并行**: Committer 单并行度是单点,但保证了提交的原子性

**架构演进**:
1. **早期版本**: 直接在 Writer 中提交,无法保证原子性
2. **引入 Committer**: 单独的 Committer 算子,保证原子提交
3. **引入 CommitUser**: 支持幂等性检查,避免重复提交
4. **引入 CommittableStateManager**: 支持状态恢复,保证故障恢复后的一致性

**业界对比**:
- **Flink + Kafka**: 也使用两阶段提交,但 Kafka 的事务机制更复杂
- **Flink + Iceberg**: 类似的两阶段提交,但 Iceberg 的事务 API 更灵活
- **Flink + Hudi**: 支持多种提交模式(同步/异步),更灵活但更复杂

### 7.1 流程时序图

```
  Flink JobManager                Writer Subtask (N个)              Committer (1个)
       │                                │                                │
       │ triggerCheckpoint(cpId)         │                                │
       │───────────────────────────────>│                                │
       │                                │                                │
       │                  prepareSnapshotPreBarrier(cpId)                │
       │                                │                                │
       │                 PrepareCommitOperator                           │
       │                   .emitCommittables(false, cpId)               │
       │                                │                                │
       │                 StoreSinkWrite.prepareCommit(false, cpId)      │
       │                   → MergeTreeWriter.prepareCommit()           │
       │                     → flushWriteBuffer()                      │
       │                     → trySyncLatestCompaction()               │
       │                     → drainIncrement() → CommitIncrement      │
       │                                │                                │
       │                 new Committable(cpId, CommitMessage)           │
       │                   → output.collect(StreamRecord)              │
       │                                │─────────────────────────────>│
       │                                │                                │
       │                                │            CommitterOperator   │
       │                                │            .processElement()   │
       │                                │            → inputs.add()     │
       │                                │                                │
       │                  snapshotState(cpId)                            │
       │                                │                                │
       │                                │         snapshotState(cpId)   │
       │                                │            → pollInputs()     │
       │                                │            → groupByCheckpoint│
       │                                │            → committableState │
       │                                │              Manager          │
       │                                │              .snapshotState() │
       │                                │                                │
       │ notifyCheckpointComplete(cpId)  │                               │
       │────────────────────────────────│──────────────────────────────>│
       │                                │                                │
       │                                │         commitUpToCheckpoint  │
       │                                │           (cpId)              │
       │                                │                                │
       │                                │         StoreCommitter        │
       │                                │           .commit()           │
       │                                │           → TableCommitImpl   │
       │                                │             .commitMultiple() │
       │                                │           → FileStoreCommitImpl│
       │                                │             .commit()         │
       │                                │           → 原子写入 Snapshot │
       │                                │                                │
```

### 7.2 关键代码解析

#### Step 1: prepareSnapshotPreBarrier (Writer 侧)

```java
// PrepareCommitOperator.java:93-98
public void prepareSnapshotPreBarrier(long checkpointId) throws Exception {
    if (!endOfInput) {
        emitCommittables(false, checkpointId);
    }
}

private void emitCommittables(boolean waitCompaction, long checkpointId) throws IOException {
    prepareCommit(waitCompaction, checkpointId)
        .forEach(committable -> output.collect(new StreamRecord<>(committable)));
}

// endInput 时也要 emit committables，但 waitCompaction=true
public void endInput() throws Exception {
    endOfInput = true;
    emitCommittables(true, Long.MAX_VALUE);
}
```

**为什么在 `prepareSnapshotPreBarrier` 中 emit**: 这保证了所有数据文件在 checkpoint barrier 到达 Committer 之前已经准备好。Barrier 对齐机制确保了一致性。

#### Step 2: StoreSinkWriteImpl.prepareCommit

```java
// StoreSinkWriteImpl.java:137-151
public List<Committable> prepareCommit(boolean waitCompaction, long checkpointId) throws IOException {
    List<Committable> committables = new ArrayList<>();
    for (CommitMessage committable : write.prepareCommit(this.waitCompaction || waitCompaction, checkpointId)) {
        committables.add(new Committable(checkpointId, committable));
    }
    return committables;
}
```

#### Step 3: CommitterOperator 接收并提交

```java
// CommitterOperator.java:190-218
public void notifyCheckpointComplete(long checkpointId) throws Exception {
    super.notifyCheckpointComplete(checkpointId);
    commitUpToCheckpoint(endInput ? END_INPUT_CHECKPOINT_ID : checkpointId);
}

private void commitUpToCheckpoint(long checkpointId) throws Exception {
    NavigableMap<Long, GlobalCommitT> headMap =
        committablesPerCheckpoint.headMap(checkpointId, true);
    List<GlobalCommitT> committables = committables(headMap);
    
    if (committables.isEmpty() && committer.forceCreatingSnapshot()) {
        committables = Collections.singletonList(
            toCommittables(checkpointId, Collections.emptyList()));
    }

    if (checkpointId == END_INPUT_CHECKPOINT_ID) {
        // 批作业重启后可能再次 endInput，需要检查 snapshot 是否已存在
        committer.filterAndCommit(committables, false, true);
    } else {
        committer.commit(committables);
    }
    headMap.clear();
}
```

#### Step 4: StoreCommitter → FileStoreCommitImpl

```java
// StoreCommitter.java:99-104
public void commit(List<ManifestCommittable> committables) throws IOException, InterruptedException {
    commit.commitMultiple(committables, false);  // TableCommitImpl
    calcNumBytesAndRecordsOut(committables);
    commitListeners.notifyCommittable(committables);
}
```

### 7.3 CommitMessage 的结构

```java
// CommitMessageImpl 包含:
CommitMessageImpl {
    BinaryRow partition;     // 分区
    int bucket;              // 桶号
    int totalBuckets;        // 总桶数
    DataIncrement {          // 新增数据文件
        List<DataFileMeta> newFiles;       // 新写入的数据文件
        List<DataFileMeta> deletedFiles;   // 被删除的数据文件
        List<DataFileMeta> changelogFiles; // changelog 文件
    }
    CompactIncrement {       // Compaction 相关文件
        List<DataFileMeta> compactBefore;  // 参与 compaction 的输入文件
        List<DataFileMeta> compactAfter;   // compaction 输出文件
        List<DataFileMeta> changelogFiles; // compaction 产生的 changelog
    }
}
```

### 7.5 事务性保证机制

Paimon Flink Sink 通过以下机制保证 Exactly-Once 语义：

#### 1. Checkpoint 模式强制检查

```java
// FlinkSink.java
env.getCheckpointConfig().getCheckpointingMode() == CheckpointingMode.EXACTLY_ONCE,
"Paimon sink currently only supports EXACTLY_ONCE checkpoint mode. Please set "
+ "execution.checkpointing.mode to exactly-once");
```

**为什么强制 EXACTLY_ONCE**: Paimon 的原子提交依赖于 Flink 的 checkpoint 协议。AT_LEAST_ONCE 模式下，同一个 checkpoint 可能被提交多次，导致重复数据。

#### 2. CommitUser 机制

```java
// CommitterOperator.java:126-131
commitUser = StateUtils.getSingleValueFromState(
    context, "commit_user_state", String.class, initialCommitUser);
```

**CommitUser 的作用**:
- 每个 Flink 作业有唯一的 commitUser（基于作业启动时间或用户指定）
- 作业重启后从状态恢复 commitUser，保证一致性
- Snapshot 中记录 commitUser，用于幂等性检查

#### 3. Snapshot 幂等性检查

```java
// CommitterOperator.java:205-213
if (checkpointId == END_INPUT_CHECKPOINT_ID) {
    // 批作业重启后可能再次 endInput，需要检查 snapshot 是否已存在
    committer.filterAndCommit(committables, false, true);
} else {
    committer.commit(committables);
}
```

**幂等性检查流程**:
1. 批作业 endInput 时，检查对应的 snapshot 是否已存在
2. 如果存在，跳过重复提交
3. 如果不存在，正常提交

#### 4. Barrier 对齐保证

```
Writer 侧                          Committer 侧
    │                                  │
    │ prepareSnapshotPreBarrier        │
    │ (flush 所有数据文件)              │
    │                                  │
    │ ─────── Barrier ─────────────>   │
    │                                  │
    │ snapshotState()                  │
    │ (保存状态)                        │
    │                                  │
    │ notifyCheckpointComplete         │
    │ ─────────────────────────────>   │
    │                                  │
    │                          commit() │
    │                          (原子提交) │
```

**Barrier 对齐的意义**:
- 所有 Writer 的数据文件在 Barrier 到达 Committer 前已准备好
- Committer 收到 Barrier 后，所有数据文件已稳定，可以安全提交
- 即使 Committer 失败，重启后可以从状态恢复并重新提交（幂等）

#### 5. 文件冲突检查

```java
// FileStoreCommitImpl.java
// 提交前检查：
// 1. 要删除的文件是否仍然存在
// 2. 新文件的 key 范围是否与已有文件冲突
// 3. 如果冲突，从最新 snapshot 重新扫描并重试
```

**冲突检查的必要性**:
- 多个 Writer 可能同时写入同一 partition+bucket
- 冲突检查确保不会覆盖其他 Writer 的数据
- 重试机制保证最终一致性

---

## 8. 内存管理

### 解决什么问题

**核心业务问题**: 如何在有限的内存下,支持多个 partition+bucket 并发写入,避免 OOM 和内存浪费?

**没有这个设计的后果**:
- OOM: 每个 Writer 独立分配内存,总内存超过 JVM 堆大小
- 内存浪费: 某些 Writer 空闲但占用内存,其他 Writer 内存不足
- 写入阻塞: 内存不足时无法继续写入,导致反压
- 性能下降: 频繁的 GC 导致吞吐量下降

**实际场景**:
```
场景1: 多分区并发写入
  - 表有 100 个分区,每个分区 16 个 bucket,共 1600 个 partition+bucket
  - 如果每个 Writer 分配 256MB,总共需要 400GB 内存,显然不可行
  - 通过 MemoryPoolFactory 统一管理,总内存只需 4GB(write-buffer-size)

场景2: 热点分区
  - 当天分区写入频繁,历史分区几乎不写入
  - 热点分区的 Writer 需要更多内存,历史分区的 Writer 可以释放内存
  - 通过抢占机制,自动将历史分区的内存分配给热点分区

场景3: 内存不足时的抢占
  - Writer A 占用 1GB 内存,但已经 10 分钟没有新数据
  - Writer B 需要内存但 MemoryPool 已满
  - MemoryPoolFactory 强制 Writer A flush,释放内存给 Writer B
```

### 有什么坑

**误区陷阱**:
1. **误以为 write-buffer-size 是每个 Writer 的内存**: 实际上是所有 Writer 共享的总内存
2. **混淆堆内存和 Managed Memory**: 
   - 堆内存模式: 从 JVM 堆分配,不受 Flink 内存框架管理
   - Managed Memory 模式: 从 TaskManager 的 Managed Memory 分配,受 Flink 管理
3. **忽略抢占的副作用**: 频繁抢占会导致频繁 flush,产生大量小文件
4. **page-size 配置不当**: 过大导致内存浪费,过小导致频繁分配

**错误配置示例**:
```sql
-- 错误1: write-buffer-size 过大
WITH ('write-buffer-size' = '10GB')
-- 问题: 如果 TaskManager 堆内存只有 8GB,会 OOM

-- 错误2: write-buffer-size 过小
WITH ('write-buffer-size' = '64MB')
-- 问题: 多个 partition+bucket 并发写入时,频繁抢占,性能差

-- 错误3: 启用 Managed Memory 但未配置足够的内存
SET 'taskmanager.memory.managed.fraction' = '0.1';
WITH ('sink.use-managed-memory' = 'true')
-- 问题: Managed Memory 只有 10%,可能不足

-- 错误4: page-size 过大
WITH ('page-size' = '64MB')
-- 问题: 每次分配 64MB,内存利用率低
```

**生产环境注意事项**:
1. **write-buffer-size 设置**: 建议为 TaskManager 堆内存的 20%-40%
2. **Managed Memory 模式**: 生产环境建议启用,避免 OOM
3. **监控内存抢占**: 通过 metrics 监控 `bufferPreemptCount`,过高说明内存不足
4. **page-size 设置**: 默认 64KB,一般不需要调整

**性能陷阱**:
1. **频繁抢占**: 内存不足导致频繁抢占,Writer 频繁 flush,产生大量小文件
2. **内存碎片**: 长时间运行后,内存碎片增多,分配效率下降
3. **GC 压力**: 堆内存模式下,大量 MemorySegment 对象增加 GC 压力

### 核心概念解释

**术语定义**:
- **MemoryPoolFactory**: 内存池工厂,管理所有 Writer 的内存分配
- **MemorySegment**: 内存段,固定大小的内存块(默认 64KB)
- **OwnerMemoryPool**: 每个 Writer 的内存池,从 MemoryPoolFactory 分配内存
- **MemoryOwner**: 内存拥有者,即 MergeTreeWriter,实现 flushMemory() 接口
- **抢占 (Preempt)**: 当内存不足时,强制占用最多内存的 Writer flush,释放内存

**内存管理架构**:
```
MemoryPoolFactory (per TaskManager)
  ├── innerPool (HeapMemorySegmentPool / FlinkMemorySegmentPool)
  │     ├── totalPages = write-buffer-size / page-size
  │     └── freePages (空闲内存段列表)
  │
  ├── owners: List<MemoryOwner> (所有 MergeTreeWriter)
  │     ├── Writer A (partition=2026-04-22, bucket=0)
  │     ├── Writer B (partition=2026-04-22, bucket=1)
  │     └── Writer C (partition=2026-04-21, bucket=0)
  │
  └── OwnerMemoryPool (per Writer)
        ├── nextSegment()
        │     ├── segment = innerPool.nextSegment()
        │     ├── if (segment == null) preemptMemory(owner)
        │     └── return segment
        │
        └── returnAll() (flush 后归还内存)
```

**抢占机制**:
```
Writer B 需要内存,但 innerPool 已满
  ↓
MemoryPoolFactory.preemptMemory(Writer B)
  ↓
遍历所有 owners,找到内存占用最大的 Writer A (排除 Writer B 自己)
  ↓
Writer A.flushMemory()
  ↓
Writer A flush SortBuffer,释放所有 MemorySegment
  ↓
innerPool.freePages 增加
  ↓
Writer B 重新尝试 nextSegment(),成功获取内存
```

**与其他系统对比**:
- **RocksDB**: 使用 Block Cache 统一管理内存,类似 MemoryPoolFactory
- **HBase**: 使用 MemStore + Block Cache,内存管理更复杂
- **Flink**: 使用 Managed Memory 管理算子内存,Paimon 可以集成

### 设计理念

**为什么这样设计**:
1. **统一内存池**: 所有 Writer 共享一个内存池,避免内存浪费
   - 问题: 如果每个 Writer 独立分配内存,总内存 = Writer 数量 × 单个内存
   - 解决: 统一内存池,总内存 = write-buffer-size,与 Writer 数量无关
   - 好处: 支持大量 partition+bucket 并发写入,不会 OOM

2. **抢占机制**: 内存不足时,自动抢占占用最多的 Writer
   - 问题: 如果不抢占,Writer 会阻塞等待,可能死锁
   - 解决: 抢占占用最多的 Writer,强制 flush 释放内存
   - 好处: 保证至少有一个 Writer 能推进,避免死锁

3. **LRU 式抢占**: 选择内存占用最大的 Writer 抢占
   - 原因: 内存占用大说明该 Writer 已经积累了大量数据,flush 的收益最大
   - 好处: 公平且高效,避免频繁抢占小 Writer

4. **两种内存模式**: 堆内存 vs Managed Memory
   - 堆内存: 简单直接,适合小规模写入
   - Managed Memory: 与 Flink 内存框架集成,适合大规模生产环境
   - 好处: 灵活性,用户可以根据场景选择

**权衡取舍**:
- **内存利用率 vs 复杂性**: 统一内存池提高了利用率,但增加了复杂性
- **抢占 vs 小文件**: 抢占避免了死锁,但可能产生小文件
- **堆内存 vs Managed Memory**: 堆内存简单但可能 OOM,Managed Memory 安全但配置复杂

**架构演进**:
1. **早期版本**: 每个 Writer 独立分配内存,容易 OOM
2. **引入 MemoryPoolFactory**: 统一内存池,支持大量 Writer 并发
3. **引入抢占机制**: 避免死锁,保证写入推进
4. **引入 Managed Memory 模式**: 与 Flink 内存框架集成,避免 OOM

**业界对比**:
- **RocksDB 的 Block Cache**: 类似的统一内存池,但不支持抢占
- **HBase 的 MemStore**: 支持 flush,但内存管理更复杂(需要考虑 Region)
- **Flink 的 Managed Memory**: Paimon 可以集成,但需要额外配置

### 8.1 MemoryPoolFactory 的内存分配和抢占机制

**源码路径**: `paimon-core/src/main/java/org/apache/paimon/memory/MemoryPoolFactory.java`

```
┌──────────────────────────────────────────────────────────┐
│                    MemoryPoolFactory                       │
│                                                            │
│  innerPool (HeapMemorySegmentPool / FlinkMemorySegmentPool)│
│  totalPages = innerPool.freePages()                       │
│                                                            │
│  owners: Iterable<MemoryOwner>                            │
│    → 所有 MergeTreeWriter 实例 (每个 partition+bucket 一个) │
│                                                            │
│  OwnerMemoryPool (per writer):                            │
│    nextSegment():                                          │
│      segment = innerPool.nextSegment()                    │
│      if (segment == null):                                │
│        preemptMemory(owner)  ← 抢占内存                   │
│        segment = innerPool.nextSegment()                  │
│      return segment                                       │
│                                                            │
│  preemptMemory(owner):                                    │
│    找到内存占用最大的 other writer (排除自己)               │
│    → other.flushMemory()  ← 强制 flush 释放内存           │
│                                                            │
└──────────────────────────────────────────────────────────┘
```

### 8.2 内存分配策略

#### 内存来源选择

```java
// PrepareCommitOperator.java:74-89
if (options.get(SINK_USE_MANAGED_MEMORY)) {
    // 使用 Flink Managed Memory (从 TaskManager 的 Managed Memory 段分配)
    memoryPool = new FlinkMemorySegmentPool(computeManagedMemory(this), pageSize, memoryAllocator);
} else {
    // 使用堆内存 (默认)
    memoryPool = new HeapMemorySegmentPool(coreOptions.writeBufferSize(), coreOptions.pageSize());
}
memoryPoolFactory = new MemoryPoolFactory(memoryPool);
```

**为什么提供两种内存模式**:
- **堆内存模式** (默认): 简单直接，适合小规模写入。缺点是不受 Flink 内存框架管理，可能导致 OOM。
- **Managed Memory 模式**: 与 Flink 的内存管理框架集成，受 TaskManager 统一管控，适合大规模生产环境。

#### 抢占机制详解

```java
// MemoryPoolFactory.java:73-93
private void preemptMemory(MemoryOwner owner) {
    long maxMemory = 0;
    MemoryOwner max = null;
    for (MemoryOwner other : owners) {
        // 不能抢占自己！同时写入和 flush 可能导致状态不一致
        if (other != owner && other.memoryOccupancy() > maxMemory) {
            maxMemory = other.memoryOccupancy();
            max = other;
        }
    }
    if (max != null) {
        max.flushMemory();  // 强制占用最多的 writer flush
        ++bufferPreemptCount;
    }
}
```

**为什么抢占而不是等待**: 在 LSM-tree 架构下，如果所有 writer 都阻塞等待内存，会导致死锁。抢占策略保证至少有一个 writer 能获得内存推进写入，LRU 式地选择最大内存占用者 flush，是公平且高效的。

#### MergeTreeWriter 的 flushMemory 实现

```java
// MergeTreeWriter.java:202-207
public void flushMemory() throws Exception {
    boolean success = writeBuffer.flushMemory();
    if (!success) {
        flushWriteBuffer(false, false);  // SortBuffer 无法部分释放，全量 flush
    }
}
```

### 8.3 内存生命周期

```
Writer 算子启动
  │
  ├── PrepareCommitOperator.setup()  → 创建 MemoryPoolFactory (一个算子实例共享)
  │
  ├── 第一次 write(partition_A, bucket_0)
  │     → AbstractFileStoreWrite.getWriterWrapper()
  │       → 创建新 MergeTreeWriter
  │       → MemoryFileStoreWrite.notifyNewWriter()
  │         → writeBufferPool.notifyNewOwner(writer)
  │           → writer.setMemoryPool(new OwnerMemoryPool(writer))
  │             → 创建 SortBufferWriteBuffer
  │
  ├── 写入数据
  │     → writeBuffer.put()  → 从 OwnerMemoryPool 分配 MemorySegment
  │     → 内存不足时  → preemptMemory()  → 其他 writer flush 释放内存
  │
  ├── prepareCommit()  → 所有 writer flush → 内存归还到 innerPool
  │
  └── close()  → 关闭所有 writer → 释放全部内存
```

---

## 9. 异步 Compaction

### 解决什么问题

**核心业务问题**: 如何在不阻塞写入的情况下,持续合并小文件,控制读放大和存储空间?

**没有这个设计的后果**:
- 读放大严重: Level 0 文件越来越多,每次读取需要合并大量文件
- 存储空间浪费: 同一 key 的多个版本占用大量空间
- 写入阻塞: 文件数量过多导致写入无法继续
- 性能下降: 大量小文件导致 HDFS NameNode 压力大

**实际场景**:
```
场景1: 高频更新场景
  - 每秒写入 10000 条更新,每分钟 flush 一次
  - 1 小时产生 60 个 Level 0 文件
  - 如果不 compaction,读取时需要合并 60 个文件,性能极差
  - 通过异步 compaction,Level 0 文件数量控制在 10 个以内

场景2: 写入反压
  - Level 0 文件数量达到 numSortedRunStopTrigger (默认 10)
  - 写入被阻塞,等待 compaction 完成
  - Compaction 完成后,Level 0 文件减少,写入继续
  - 这是 LSM-tree 的自然反压机制

场景3: Full Compaction
  - changelog-producer=full-compaction 模式
  - 每隔 deltaCommits 个 checkpoint,触发全局 Full Compaction
  - 合并所有 level 的文件,生成完整的 changelog
  - 用于下游消费完整的变更数据
```

### 有什么坑

**误区陷阱**:
1. **误以为 Compaction 是同步的**: 实际上是异步执行,写入和 compaction 并行
2. **忽略反压机制**: 当 sorted run 数量超过阈值时,写入会被阻塞
3. **Full Compaction 的性能开销**: 全局 Full Compaction 会合并所有文件,IO 开销巨大
4. **Compaction 线程数配置不当**: 过少导致 compaction 跟不上写入,过多导致 CPU 和 IO 竞争

**错误配置示例**:
```sql
-- 错误1: num-sorted-run.stop-trigger 过大
WITH ('num-sorted-run.stop-trigger' = '100')
-- 问题: Level 0 文件过多,读放大严重,且 compaction 压力大

-- 错误2: num-sorted-run.stop-trigger 过小
WITH ('num-sorted-run.stop-trigger' = '2')
-- 问题: 频繁触发反压,写入吞吐量低

-- 错误3: compaction.max-file-num 过大
WITH ('compaction.max-file-num' = '100')
-- 问题: 单次 compaction 合并过多文件,IO 和内存开销大

-- 错误4: Full Compaction 间隔过短
WITH ('full-compaction.delta-commits' = '1')
-- 问题: 每个 checkpoint 都触发 Full Compaction,性能极差
```

**生产环境注意事项**:
1. **监控 sorted run 数量**: 通过 metrics 监控,接近 stop-trigger 说明 compaction 跟不上
2. **Compaction 线程数**: 默认与 CPU 核数相关,高吞吐场景建议增加
3. **Full Compaction 间隔**: 建议至少 10 个 checkpoint,避免频繁触发
4. **Compaction 策略**: Universal Compaction 适合写多读少,Tiered Compaction 适合读多写少

**性能陷阱**:
1. **Compaction 追不上写入**: 写入速度过快,compaction 无法及时合并,导致反压
2. **Full Compaction 阻塞 Checkpoint**: waitCompaction=true 时,Checkpoint 会等待 compaction 完成
3. **Compaction IO 竞争**: Compaction 和写入共享磁盘 IO,可能相互影响

### 核心概念解释

**术语定义**:
- **Sorted Run**: LSM-tree 中的一组有序文件,Level 0 的每个文件是一个 sorted run
- **Compaction**: 合并多个 sorted run,生成更大的文件,减少文件数量
- **Full Compaction**: 合并所有 level 的文件,生成单个文件
- **Universal Compaction**: 基于文件大小的 compaction 策略,适合写多读少
- **Tiered Compaction**: 基于 level 的 compaction 策略,适合读多写少
- **反压 (Backpressure)**: 当 sorted run 数量过多时,阻塞写入,等待 compaction

**Compaction 触发条件**:
```
每次 flush 后检查:
  ├── sorted run 数量 >= numSortedRunCompactTrigger
  │     → 触发 compaction
  │
  ├── sorted run 数量 >= numSortedRunStopTrigger
  │     → 写入被阻塞,等待 compaction 完成
  │
  └── Full Compaction 条件:
        ├── changelog-producer = full-compaction
        ├── checkpointId % deltaCommits == 0
        └── 触发全局 Full Compaction
```

**Compaction 流程**:
```
MergeTreeWriter.flushWriteBuffer()
  ↓
compactManager.addNewFile(fileMeta)  (新文件注册到 levels)
  ↓
compactManager.triggerCompaction(forcedFullCompaction)
  ↓
strategy.pick(numberOfLevels, runs)  (选择要合并的文件)
  ↓
submitCompaction(unit)  (提交到线程池异步执行)
  ↓
CompactTask.call()  (读取旧文件 + 排序合并 + 写入新文件)
  ↓
trySyncLatestCompaction()  (获取 compaction 结果)
  ↓
levels.update(compactBefore, compactAfter)  (更新 levels)
```

**与其他系统对比**:
- **RocksDB**: 也使用 LSM-tree + 异步 compaction,策略类似
- **HBase**: 使用 Major Compaction 和 Minor Compaction,概念类似
- **Cassandra**: 使用 Size-Tiered Compaction 和 Leveled Compaction,策略更多样

### 设计理念

**为什么这样设计**:
1. **异步执行**: Compaction 在后台线程池执行,不阻塞写入
   - 问题: 如果同步执行,写入会被长时间阻塞
   - 解决: 异步执行,写入和 compaction 并行
   - 好处: 写入吞吐量高,延迟低

2. **反压机制**: 当 sorted run 过多时,阻塞写入
   - 问题: 如果不限制,Level 0 文件会无限增长
   - 解决: 通过 numSortedRunStopTrigger 做反压
   - 好处: 保证系统稳定,避免读放大和存储空间爆炸

3. **分层触发**: compactTrigger < stopTrigger < stopTrigger+1
   - compactTrigger: 开始 compaction,但不阻塞写入
   - stopTrigger: 阻塞写入,等待 compaction 完成
   - stopTrigger+1: checkpoint 也被阻塞,更严格的等待条件
   - 好处: 分层控制,避免过早或过晚触发

4. **Full Compaction 的全局同步**: 所有 Writer 在同一 checkpoint 触发
   - 问题: 如果每个 Writer 独立触发,生成的 changelog 不一致
   - 解决: 通过 GlobalFullCompactionSinkWrite 全局同步
   - 好处: 生成完整一致的 changelog

**权衡取舍**:
- **写入吞吐量 vs 读放大**: 异步 compaction 提高了写入吞吐量,但增加了读放大
- **反压 vs 稳定性**: 反压降低了写入吞吐量,但保证了系统稳定
- **Full Compaction vs 性能**: Full Compaction 生成完整 changelog,但 IO 开销大

**架构演进**:
1. **早期版本**: 同步 compaction,写入性能差
2. **引入异步 compaction**: 写入和 compaction 并行,性能提升
3. **引入反压机制**: 避免 Level 0 文件无限增长
4. **引入 Full Compaction**: 支持 changelog-producer=full-compaction

**业界对比**:
- **RocksDB**: 也使用异步 compaction + 反压,策略类似
- **HBase**: Major Compaction 通常是手动触发或定时触发,不如 Paimon 自动化
- **Cassandra**: 支持多种 compaction 策略,更灵活但更复杂

### 9.1 CompactManager 触发机制

**源码路径**: `paimon-core/src/main/java/org/apache/paimon/mergetree/compact/MergeTreeCompactManager.java`

```
MergeTreeCompactManager 继承 CompactFutureManager
  │
  ├── triggerCompaction(fullCompaction)
  │     │
  │     ├── fullCompaction = true:
  │     │     → CompactStrategy.pickFullCompaction()
  │     │     → 选择所有 level 的文件进行合并
  │     │
  │     └── fullCompaction = false (常规 compaction):
  │           → strategy.pick(numberOfLevels, runs)
  │           → 根据 UniversalCompaction / TieredCompaction 策略选择文件
  │           → 提交异步 CompactTask
  │
  ├── shouldWaitForLatestCompaction()
  │     → levels.numberOfSortedRuns() > numSortedRunStopTrigger
  │     → 当 sorted run 数量超过停止阈值时，写入必须等待 compaction 完成
  │     → 这是写入反压的核心机制
  │
  ├── shouldWaitForPreparingCheckpoint()
  │     → levels.numberOfSortedRuns() > numSortedRunStopTrigger + 1
  │     → checkpoint 准备阶段更严格的等待条件
  │
  └── getCompactionResult(blocking)
        → 从 Future 获取 compaction 结果
        → blocking=true 时同步等待
```

### 9.2 反压机制

```
                          写入速度 ──────────────────>
                          │
sorted run 数量  ─────────┤
                          │
  numSortedRunStopTrigger─┤─────────── 写入被阻塞,等待 compaction
                          │            (shouldWaitForLatestCompaction = true)
  numSortedRunStopTrigger+1│─────────── checkpoint 也被阻塞
                          │            (shouldWaitForPreparingCheckpoint = true)
                          │
  numSortedRunCompactTrigger─────────── 触发 compaction
                          │            (strategy.pick 返回 CompactUnit)
                          │
```

**为什么这样设计反压**: 如果不限制 sorted run 数量，Level 0 文件会无限增长，导致：
1. 读放大严重（每次读要合并更多文件）
2. compaction 追不上写入，形成恶性循环

通过 `numSortedRunStopTrigger` 做背压，当 compaction 跟不上写入速度时，自动降低写入速率，保证系统稳定。

### 9.3 异步执行模型

```java
// MergeTreeCompactManager.java:208-235
private void submitCompaction(CompactUnit unit, boolean dropDelete) {
    CompactTask task;
    if (unit.fileRewrite()) {
        task = new FileRewriteCompactTask(rewriter, unit, dropDelete, metricsReporter);
    } else {
        task = new MergeTreeCompactTask(
            keyComparator, compactionFileSize, rewriter, unit,
            dropDelete, levels.maxLevel(), metricsReporter,
            compactDfSupplier, recordLevelExpire, forceRewriteAllFiles);
    }
    // 提交到线程池异步执行
    taskFuture = executor.submit(task);
}
```

**为什么 Compaction 异步执行**: Compaction 涉及大量 IO（读取旧文件 + 排序合并 + 写入新文件），同步执行会严重阻塞写入路径。异步执行让写入和 compaction 并行进行，只在必要时（sorted run 过多）才同步等待。

### 9.4 flush 和 compaction 的协调

```java
// MergeTreeWriter.java:209-249 (flushWriteBuffer)
flushWriteBuffer(waitForLatestCompaction, forcedFullCompaction):
    // 1. 如果 sorted run 过多，强制等待
    if (compactManager.shouldWaitForLatestCompaction())
        waitForLatestCompaction = true;

    // 2. 将 WriteBuffer 数据写入 Level 0 文件
    writeBuffer.forEach(keyComparator, mergeFunction, changelogWriter, dataWriter);
    writeBuffer.clear();

    // 3. 新文件注册到 levels
    for (DataFileMeta fileMeta : dataWriter.result())
        compactManager.addNewFile(fileMeta);

    // 4. 尝试获取 compaction 结果
    trySyncLatestCompaction(waitForLatestCompaction);

    // 5. 触发新的 compaction
    compactManager.triggerCompaction(forcedFullCompaction);
```

---

## 10. CDC 写入场景

### 解决什么问题

**核心业务问题**: 如何正确处理 CDC 数据流中的 INSERT/UPDATE/DELETE 操作,保证数据的最终一致性?

**没有这个设计的后果**:
- 数据不一致: UPDATE_BEFORE 和 UPDATE_AFTER 处理不当,导致数据重复或丢失
- 无法撤回: AGGREGATE merge engine 需要 UPDATE_BEFORE 做撤回,否则聚合结果错误
- 性能浪费: 不必要的 UPDATE_BEFORE 传输,增加网络开销
- 语义错误: 不同 MergeEngine 对 RowKind 的处理不同,配置错误导致语义错误

**实际场景**:
```sql
-- 场景1: MySQL Binlog 同步 (deduplicate)
CREATE TABLE paimon_user (
    id BIGINT PRIMARY KEY NOT ENFORCED,
    name STRING,
    age INT
) WITH ('merge-engine' = 'deduplicate');

-- Flink CDC 产生的数据流:
-- +I[1, "Alice", 25]  (INSERT)
-- +U[1, "Alice", 26]  (UPDATE_AFTER, 无 UPDATE_BEFORE)
-- -D[1, "Alice", 26]  (DELETE)

-- Paimon 处理:
-- +I: 写入 KeyValue{key=[1], value=[1,"Alice",25], valueKind=INSERT}
-- +U: 写入 KeyValue{key=[1], value=[1,"Alice",26], valueKind=UPDATE_AFTER}
-- -D: 写入 KeyValue{key=[1], value=[1,"Alice",26], valueKind=DELETE}
-- Compaction 时,只保留最新的 DELETE 标记

-- 场景2: 聚合场景 (aggregate)
CREATE TABLE paimon_order_stats (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    total_amount DECIMAL(10,2)
) WITH (
    'merge-engine' = 'aggregate',
    'fields.total_amount.aggregate-function' = 'sum'
);

-- Flink 产生的数据流:
-- +I[1, 100.00]  (用户1首次下单)
-- -U[1, 100.00]  (UPDATE_BEFORE, 撤回旧值)
-- +U[1, 150.00]  (UPDATE_AFTER, 新值)

-- Paimon 处理:
-- +I: total_amount = 100.00
-- -U: total_amount = 100.00 - 100.00 = 0.00 (撤回)
-- +U: total_amount = 0.00 + 150.00 = 150.00 (累加)

-- 场景3: 部分更新 (partial-update)
CREATE TABLE paimon_user_profile (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    name STRING,
    age INT,
    city STRING
) WITH ('merge-engine' = 'partial-update');

-- 数据流:
-- +I[1, "Alice", 25, "Beijing"]  (完整记录)
-- +U[1, null, 26, null]          (只更新 age)

-- Paimon 处理:
-- +I: 写入完整记录
-- +U: 只更新 age=26,name 和 city 保持不变
```

### 有什么坑

**误区陷阱**:
1. **混淆 UPDATE_BEFORE 和 UPDATE_AFTER**: 
   - deduplicate: 只需要 UPDATE_AFTER,UPDATE_BEFORE 会被过滤
   - aggregate: 需要 UPDATE_BEFORE 做撤回,不能过滤
2. **RowKind 来源混淆**: 
   - 可以从 Flink RowData 的 RowKind 获取
   - 也可以从表配置的 rowkind-field 字段解析
3. **partial-update 的 null 语义**: null 表示不更新该字段,而不是更新为 null
4. **first-row 的 RowKind 过滤**: 只保留首次出现的行,UPDATE 和 DELETE 会被忽略

**错误配置示例**:
```sql
-- 错误1: aggregate 模式过滤 UPDATE_BEFORE
CREATE TABLE t (
    id INT PRIMARY KEY NOT ENFORCED,
    amount DECIMAL(10,2)
) WITH (
    'merge-engine' = 'aggregate',
    'fields.amount.aggregate-function' = 'sum',
    'changelog-mode' = 'upsert'  -- 错误!会过滤 UPDATE_BEFORE
);

-- 错误2: partial-update 期望更新为 null
INSERT INTO paimon_user_profile VALUES (1, null, 26, null);
-- 期望: 将 name 和 city 更新为 null
-- 实际: name 和 city 保持不变,只更新 age

-- 错误3: first-row 表接收 UPDATE
CREATE TABLE t (
    id INT PRIMARY KEY NOT ENFORCED,
    name STRING
) WITH ('merge-engine' = 'first-row');
-- UPDATE 操作会被忽略,只保留首次 INSERT 的值
```

**生产环境注意事项**:
1. **ChangelogMode 协商**: 确保 Flink Planner 和 Paimon Sink 的 ChangelogMode 一致
2. **CDC 格式**: 不同的 CDC 工具(Debezium/Canal/Maxwell)产生的 RowKind 可能不同
3. **rowkind-field 配置**: 如果 CDC 数据中 RowKind 存储在字段中,需要配置 rowkind-field
4. **聚合函数选择**: aggregate 模式支持 sum/max/min/last_value/last_non_null_value 等

**性能陷阱**:
1. **不必要的 UPDATE_BEFORE**: deduplicate 模式下,UPDATE_BEFORE 会被过滤,但仍然占用网络带宽
2. **频繁的撤回**: aggregate 模式下,频繁的 UPDATE_BEFORE 会增加计算开销
3. **partial-update 的读取开销**: 需要读取旧值才能合并,增加了读放大

### 核心概念解释

**术语定义**:
- **RowKind**: 行类型,描述数据的操作类型(INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE)
- **ChangelogMode**: 数据流包含哪些 RowKind 的描述
- **MergeEngine**: 定义如何合并同一 key 的多个版本
- **MergeFunction**: MergeEngine 的具体实现,定义 merge 逻辑
- **RowKindGenerator**: 从数据行中提取 RowKind 的工具
- **RowKindFilter**: 过滤特定 RowKind 的工具

**RowKind 映射**:
```
Flink RowKind          Paimon RowKind
  INSERT          →      INSERT
  UPDATE_BEFORE   →      UPDATE_BEFORE
  UPDATE_AFTER    →      UPDATE_AFTER
  DELETE          →      DELETE
```

**MergeEngine 对 RowKind 的处理**:
```
deduplicate (默认):
  INSERT:         写入新值
  UPDATE_BEFORE:  忽略 (Upsert 模式过滤)
  UPDATE_AFTER:   覆盖旧值
  DELETE:         删除记录

partial-update:
  INSERT:         写入新值
  UPDATE_BEFORE:  忽略
  UPDATE_AFTER:   部分字段更新 (null 表示不更新)
  DELETE:         可选支持 (配置 partial-update.remove-record-on-delete)

aggregate:
  INSERT:         聚合累加
  UPDATE_BEFORE:  撤回 (减去旧值)
  UPDATE_AFTER:   聚合累加
  DELETE:         撤回 (减去旧值)

first-row:
  INSERT:         写入 (仅首次)
  UPDATE_BEFORE:  忽略
  UPDATE_AFTER:   忽略
  DELETE:         忽略
```

**与其他系统对比**:
- **Hudi**: 支持 UPSERT/INSERT/DELETE,但没有 UPDATE_BEFORE 的概念
- **Iceberg**: 通过 Equality Delete 实现更新,不区分 UPDATE_BEFORE 和 UPDATE_AFTER
- **Delta Lake**: 支持 MERGE INTO 语法,但底层是 CoW,不区分 RowKind

### 设计理念

**为什么这样设计**:
1. **ChangelogMode 协商**: 让 Sink 告诉 Planner 自己能接受哪些 RowKind
   - 问题: 如果 Planner 产生 UPDATE_BEFORE,但 Sink 不需要,会浪费网络带宽
   - 解决: Sink 在 getChangelogMode 中声明只接受 INSERT/UPDATE_AFTER/DELETE
   - 好处: Planner 会自动过滤 UPDATE_BEFORE,减少数据传输

2. **多种 MergeEngine**: 不同场景有不同的 merge 语义
   - deduplicate: 最简单,只保留最新值,适合 CDC 同步
   - partial-update: 支持部分字段更新,适合宽表场景
   - aggregate: 支持聚合计算,适合实时统计
   - first-row: 只保留首次值,适合维度表

3. **RowKind 存储在 KeyValue 中**: valueKind 字段
   - 问题: 如何在 LSM-tree 中表示不同的操作类型?
   - 解决: KeyValue 包含 valueKind 字段,compaction 时根据 valueKind 决定如何 merge
   - 好处: 统一的存储格式,支持多种 merge 语义

4. **RowKindGenerator 的灵活性**: 支持从字段解析 RowKind
   - 问题: 某些 CDC 工具将 RowKind 存储在字段中(如 op_type='I'/'U'/'D')
   - 解决: 通过 rowkind-field 配置,从字段解析 RowKind
   - 好处: 兼容不同的 CDC 格式

**权衡取舍**:
- **灵活性 vs 复杂性**: 多种 MergeEngine 提供了灵活性,但增加了理解和配置的复杂性
- **性能 vs 功能**: partial-update 需要读取旧值,增加了读放大,但提供了部分更新功能
- **一致性 vs 性能**: aggregate 需要 UPDATE_BEFORE 做撤回,增加了网络开销,但保证了聚合结果的正确性

**架构演进**:
1. **早期版本**: 只支持 deduplicate,不支持 UPDATE_BEFORE
2. **引入 partial-update**: 支持部分字段更新
3. **引入 aggregate**: 支持聚合计算和撤回
4. **引入 RowKindGenerator**: 支持从字段解析 RowKind

**业界对比**:
- **Hudi 的 MergeOnRead**: 类似 Paimon 的 deduplicate,但不支持 aggregate
- **Iceberg 的 Equality Delete**: 通过 Delete File 实现更新,不区分 RowKind
- **Flink Table API 的 ChangelogMode**: Paimon 的 ChangelogMode 协商机制是 Flink 生态的标准做法

### 10.1 RowKind 的来源

Paimon 的 `RowKind` 有两种来源：

1. **Flink RowData 自带的 RowKind**: 通过 `FlinkRowWrapper.getRowKind()` → `fromFlinkRowKind()` 转换
2. **表配置的 rowkind-field**: 通过 `RowKindGenerator` 从某个字符串字段解析

```java
// RowKindGenerator.java:66-68
public static RowKind getRowKind(@Nullable RowKindGenerator rowKindGenerator, InternalRow row) {
    return rowKindGenerator == null
        ? row.getRowKind()                     // 直接从行获取
        : rowKindGenerator.generate(row);      // 从指定字段解析
}
```

### 10.2 不同 MergeEngine 对 RowKind 的处理

| MergeEngine | INSERT | UPDATE_BEFORE | UPDATE_AFTER | DELETE | 说明 |
|---|---|---|---|---|---|
| **deduplicate** (默认) | 写入 | 忽略 (Upsert 模式过滤) | 覆盖旧值 | 删除记录 | 主键表默认行为，去重保留最新值 |
| **partial-update** | 写入 | 忽略 | 部分字段更新 | 可选支持 | 只更新非 null 字段 |
| **aggregate** | 聚合累加 | 需要 (撤回) | 聚合累加 | 需要 (撤回) | 聚合函数处理，支持 retract |
| **first-row** | 写入（仅首次） | 忽略 | 忽略 | 忽略 | 只保留首次出现的行 |

### 10.3 Upsert 模式下 UPDATE_BEFORE 的过滤

```java
// FlinkTableSinkBase.java:71-99 (getChangelogMode)
// 对于主键表（非 INPUT/AGGREGATE），过滤 UPDATE_BEFORE
for (RowKind kind : requestedMode.getContainedKinds()) {
    if (kind != RowKind.UPDATE_BEFORE) {
        builder.addContainedKind(kind);
    }
}
```

**为什么过滤 UPDATE_BEFORE**: 在 deduplicate/partial-update 场景下，主键保证了唯一性，UPDATE_AFTER 足以确定最新值，不需要旧值（UPDATE_BEFORE）。过滤掉可以减少一半的更新数据传输。

### 10.4 RowKindFilter 机制

```java
// TableWriteImpl.java:187-189
if (rowKindFilter != null && !rowKindFilter.test(rowKind)) {
    return null;  // 被过滤，不写入
}
```

`RowKindFilter` 根据 `CoreOptions` 配置决定哪些 RowKind 需要写入。例如 `first-row` merge engine 可以过滤掉 UPDATE/DELETE。

### 10.5 KeyValue 中的 RowKind 存储

在 LSM-tree 存储层面，RowKind 被转换为 `KeyValue` 的 `valueKind`：

```
KeyValue = {
    key: BinaryRow (主键),
    sequenceNumber: long (单调递增),
    valueKind: RowKind (INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE),
    value: BinaryRow (完整行数据)
}
```

Compaction 时，`MergeFunction` 根据 `valueKind` 决定如何合并同一 key 的多条记录。

---

## 11. 批写入 vs 流写入的差异

### 解决什么问题

**核心业务问题**: 如何用统一的写入框架同时支持流式和批式两种执行模式,满足不同场景的需求?

**没有这个设计的后果**:
- 代码重复: 流式和批式需要维护两套代码,增加维护成本
- 行为不一致: 流式和批式的语义不同,容易出错
- 无法流批一体: 无法在同一张表上同时进行流式写入和批量回填
- 性能浪费: 批式场景无法利用批处理的优化(如 Clustering)

**实际场景**:
```sql
-- 场景1: 流式实时写入
SET 'execution.runtime-mode' = 'streaming';
SET 'execution.checkpointing.interval' = '1min';
INSERT INTO paimon_orders SELECT * FROM kafka_source;
-- 特点: 持续运行,通过 checkpoint 提交,支持 Exactly-Once

-- 场景2: 批量历史回填
SET 'execution.runtime-mode' = 'batch';
INSERT INTO paimon_orders 
SELECT * FROM hive_orders WHERE dt < '2026-01-01';
-- 特点: 一次性执行,通过 endInput 提交,支持幂等重试

-- 场景3: 批量覆盖分区
SET 'execution.runtime-mode' = 'batch';
INSERT OVERWRITE paimon_dim_user PARTITION (dt='2026-04-22')
SELECT * FROM hive_user_snapshot WHERE dt='2026-04-22';
-- 特点: 覆盖指定分区,ignorePreviousFiles=true

-- 场景4: 流批混合
-- 流式作业持续写入当天数据
INSERT INTO paimon_events SELECT * FROM kafka_stream;
-- 批作业每天凌晨回填历史数据
INSERT INTO paimon_events SELECT * FROM hive_history WHERE dt = '${yesterday}';
-- 特点: 同一张表,流批共存,互不影响
```

### 有什么坑

**误区陷阱**:
1. **流式作业使用 INSERT OVERWRITE**: 会直接抛异常,OVERWRITE 只支持批模式
2. **批作业未配置足够的内存**: 批作业通常数据量大,需要更多的 write-buffer-size
3. **批作业的 Clustering 配置错误**: Clustering 只在批模式 + BUCKET_UNAWARE 时生效
4. **批作业重启后重复提交**: 需要依赖幂等性检查,避免重复写入

**错误配置示例**:
```sql
-- 错误1: 流式作业使用 OVERWRITE
SET 'execution.runtime-mode' = 'streaming';
INSERT OVERWRITE paimon_table SELECT * FROM kafka_source;
-- 报错: Paimon doesn't support streaming INSERT OVERWRITE

-- 错误2: 批作业未开启 waitCompaction
SET 'execution.runtime-mode' = 'batch';
WITH ('prepare-commit.wait-compaction' = 'false')
-- 问题: 批作业结束时可能有大量未 compact 的小文件

-- 错误3: 批作业配置 checkpoint
SET 'execution.runtime-mode' = 'batch';
SET 'execution.checkpointing.interval' = '1min';
-- 无效配置,批作业不支持 checkpoint

-- 错误4: POSTPONE_MODE 批作业未启用 fixed-bucket 优化
WITH ('bucket' = '-2', 'postpone-batch-write-fixed-bucket' = 'false')
-- 问题: 批作业无法利用已知桶数的优化,性能较差
```

**生产环境注意事项**:
1. **批作业的内存配置**: write-buffer-size 建议设置为流式的 2-4 倍
2. **批作业的并行度**: 建议根据数据量动态调整,避免数据倾斜
3. **批作业的 Compaction**: 建议开启 waitCompaction,避免产生大量小文件
4. **流批混合场景**: 确保 commitUser 不冲突,避免互相覆盖

**性能陷阱**:
1. **批作业频繁 flush**: write-buffer-size 过小导致频繁 flush,性能差
2. **Clustering 排序开销**: clustering.columns 配置不当导致全局排序,严重影响性能
3. **批作业的 Checkpoint 开销**: 批作业不需要 checkpoint,配置了反而增加开销

### 核心概念解释

**术语定义**:
- **RuntimeExecutionMode**: Flink 的执行模式,STREAMING 或 BATCH
- **isBounded**: 数据流是否有界,流式为 false,批式为 true
- **endInput**: 批作业结束时的回调,用于触发最终提交
- **ignorePreviousFiles**: 是否忽略已有文件,INSERT OVERWRITE 时为 true
- **waitCompaction**: 是否等待 compaction 完成,批作业建议为 true
- **Clustering**: 批作业的数据排序优化,提升查询性能

**流式 vs 批式对比**:
```
维度                  流式 (Streaming)              批式 (Batch)
─────────────────────────────────────────────────────────────────
执行模式              RuntimeExecutionMode.STREAMING  RuntimeExecutionMode.BATCH
数据流                无界 (unbounded)                有界 (bounded)
Checkpoint            必须开启 (EXACTLY_ONCE)         不需要
提交触发              notifyCheckpointComplete        endInput
Compaction            异步持续执行                    commit 时 waitCompaction=true
ignorePreviousFiles   false                          true (OVERWRITE 时)
内存抢占              持续发生                        较少
Clustering            不支持                          支持 (BUCKET_UNAWARE)
POSTPONE_MODE         PostponeBucketSink             PostponeFixedBucketSink (已知桶数)
```

**提交路径对比**:
```
流式提交:
  prepareSnapshotPreBarrier(cpId)
    → emitCommittables(false, cpId)  (waitCompaction=false)
    → Writer flush 但不等 compaction
  notifyCheckpointComplete(cpId)
    → CommitterOperator.commitUpToCheckpoint(cpId)
    → StoreCommitter.commit()

批式提交:
  endInput()
    → PrepareCommitOperator.emitCommittables(true, Long.MAX_VALUE)  (waitCompaction=true)
    → Writer flush 并等 compaction 完成
  CommitterOperator.endInput()
    → if (!streamingCheckpointEnabled)
    →   commitUpToCheckpoint(END_INPUT_CHECKPOINT_ID)
    → StoreCommitter.filterAndCommit()  (幂等提交)
```

**与其他系统对比**:
- **Hive**: 只支持批式,无流式概念
- **Iceberg**: 支持流批一体,但批式优化较少
- **Hudi**: 支持流批一体,但流批行为差异较大

### 设计理念

**为什么这样设计**:
1. **统一的写入框架**: 流式和批式共享相同的 Sink 实现
   - 问题: 如果流批分开实现,代码重复,维护成本高
   - 解决: 通过 isBounded 和 streamingCheckpointEnabled 区分流批行为
   - 好处: 代码复用,行为一致,易于维护

2. **批式的 waitCompaction**: 批作业结束时等待 compaction 完成
   - 问题: 批作业产生大量小文件,影响后续读取性能
   - 解决: endInput 时 waitCompaction=true,等待 compaction 完成
   - 好处: 批作业结束后,文件已经合并,读取性能好

3. **批式的幂等提交**: 通过 filterAndCommit 检查 snapshot 是否已存在
   - 问题: Flink 新版本中,批作业失败后可能从中间算子重启,导致 endInput 被多次调用
   - 解决: 提交前检查 snapshot 是否已存在,避免重复提交
   - 好处: 支持批作业的故障恢复,保证 Exactly-Once

4. **Clustering 优化**: 批作业支持数据排序,提升查询性能
   - 问题: 追加表(BUCKET_UNAWARE)无法利用 bucket 做数据排序
   - 解决: 批模式下,通过 Clustering 对数据排序后写入
   - 好处: 查询时可以利用排序跳过不相关的文件,性能提升

**权衡取舍**:
- **统一 vs 优化**: 统一的框架保证了一致性,但某些批式优化(如 Clustering)只在特定模式下生效
- **waitCompaction vs 延迟**: 批式等待 compaction 增加了延迟,但避免了小文件问题
- **幂等 vs 性能**: 幂等检查增加了开销,但保证了批作业的正确性

**架构演进**:
1. **早期版本**: 流批分开实现,代码重复
2. **统一写入框架**: 流批共享相同的 Sink,通过 isBounded 区分
3. **引入 Clustering**: 支持批式数据排序优化
4. **引入幂等提交**: 支持批作业的故障恢复

**业界对比**:
- **Iceberg**: 也支持流批一体,但批式优化较少(无 Clustering)
- **Hudi**: 流批行为差异较大,批式使用 BulkInsert,流式使用 Upsert
- **Delta Lake**: 主要面向批处理,流式支持较弱

### 11.1 核心差异对比

| 维度 | 流写入 (Streaming) | 批写入 (Batch) |
|---|---|---|
| **执行模式** | `RuntimeExecutionMode.STREAMING` | `RuntimeExecutionMode.BATCH` |
| **Checkpoint** | 必须开启，EXACTLY_ONCE | 不需要 Checkpoint |
| **提交触发** | `notifyCheckpointComplete` | `endInput` |
| **Compaction** | 异步持续执行 | commit 时 waitCompaction=true |
| **ignorePreviousFiles** | false (需要恢复已有文件) | true (overwrite=true 时) |
| **内存抢占** | 持续发生 | 通常数据量较小，抢占较少 |
| **Clustering** | 不支持 | 支持 (BUCKET_UNAWARE 模式) |
| **POSTPONE_MODE 行为** | 使用 PostponeBucketSink | 可选使用 PostponeFixedBucketSink (已知桶数) |

### 11.2 提交路径差异

#### 流式提交

```
prepareSnapshotPreBarrier(cpId)
  → emitCommittables(false, cpId)
  → Writer 侧 flush 但不等 compaction (waitCompaction=false)

notifyCheckpointComplete(cpId)
  → CommitterOperator.commitUpToCheckpoint(cpId)
  → StoreCommitter.commit()
```

**流式模式的 streamingCheckpointEnabled 标志**:
- 当 Flink 启用了 checkpoint 时，`streamingCheckpointEnabled=true`
- 此时 CommitterOperator 在 `notifyCheckpointComplete` 时提交
- 如果 checkpoint 禁用，则在 `endInput` 时提交

#### 批式提交

```
endInput()
  → PrepareCommitOperator.emitCommittables(true, Long.MAX_VALUE)
  → Writer 侧 flush 并等 compaction 完成 (waitCompaction=true)

CommitterOperator.endInput()
  → if (!streamingCheckpointEnabled)
  →   commitUpToCheckpoint(END_INPUT_CHECKPOINT_ID)
  → StoreCommitter.filterAndCommit()  // 幂等提交，检查已提交的 snapshot
```

### 11.3 批写入幂等检查

```java
// CommitterOperator.java:205-213
if (checkpointId == END_INPUT_CHECKPOINT_ID) {
    // 批作业重启后可能再次 endInput，需要检查 snapshot 是否已存在
    committer.filterAndCommit(committables, false, true);
} else {
    committer.commit(committables);
}
```

**为什么批写入需要幂等检查**: Flink 新版本中，批作业失败后可能从中间算子重启（而不是从头），导致 `endInput()` 被多次调用。幂等提交通过检查 snapshot 是否已存在来避免重复写入。

### 11.4 AppendTableSink 的流式额外拓扑

```java
// AppendTableSink.java:98-131
if (enableCompaction && isStreamingMode) {
    // 流式追加表需要额外的 compaction 拓扑
    written.transform("Compact Coordinator: " + table.name(), ...)  // 单并行度协调器
           .forceNonParallel()
           .transform("Compact Worker: " + table.name(), ...);      // 多并行度 worker
}
```

**为什么追加表需要单独的 compaction 拓扑**: 追加表（BUCKET_UNAWARE）没有固定的 partition+bucket 到 writer 的映射，因此不能在 writer 内部做 compaction。需要一个独立的协调器来调度跨 writer 的 compaction 任务。

---

## 12. 与 Iceberg Flink Sink 写入流程的详细对比

### 解决什么问题

**核心业务问题**: 如何选择合适的 Lakehouse 方案(Paimon vs Iceberg),理解两者的设计差异和适用场景?

**没有这个设计的后果**:
- 选型错误: 选择了不适合业务场景的方案,导致性能差或功能不满足
- 迁移困难: 后期发现问题需要迁移,成本高昂
- 资源浪费: 不了解两者差异,无法充分利用各自的优势
- 运维复杂: 不了解底层机制,遇到问题无法快速定位

**实际场景**:
```
场景1: 高频实时更新 (推荐 Paimon)
  - 业务: 用户画像实时更新,每秒数万次更新
  - Paimon: LSM-tree 写入性能优异,原生支持 CDC
  - Iceberg: 需要写 Delete File,写入性能较差

场景2: 低频批量覆盖 (推荐 Iceberg CoW)
  - 业务: 每天凌晨全量覆盖维度表
  - Paimon: 需要 compaction 清理旧数据
  - Iceberg CoW: 直接重写文件,读取零开销

场景3: 跨分区 Upsert (推荐 Paimon)
  - 业务: 用户行为表,主键是 user_id,按日期分区
  - Paimon: KEY_DYNAMIC 模式支持跨分区 upsert
  - Iceberg: 不原生支持,需要手动处理

场景4: 已有 Iceberg 生态 (推荐 Iceberg)
  - 业务: 已有大量 Iceberg 表和工具链
  - Paimon: 需要迁移,成本高
  - Iceberg: 兼容性好,无需迁移
```

### 有什么坑

**误区陷阱**:
1. **误以为 Iceberg 不支持更新**: Iceberg 支持更新,但通过 Delete File 实现,性能不如 Paimon
2. **误以为 Paimon 不支持批处理**: Paimon 支持流批一体,批处理性能也很好
3. **忽略 Compaction 的差异**: 
   - Paimon: Writer 内异步 compaction,自动化
   - Iceberg: 需要独立的 compaction 作业或 maintenance API
4. **忽略小文件问题**: 
   - Paimon: 通过 LSM-tree 自动合并小文件
   - Iceberg: 需要手动触发 compaction

**错误配置示例**:
```sql
-- Paimon 错误配置: 期望 Iceberg 的 Delete File 行为
CREATE TABLE paimon_table (id INT PRIMARY KEY NOT ENFORCED, name STRING)
WITH ('merge-engine' = 'deduplicate');
-- 误解: Paimon 不会生成 Delete File,而是通过 LSM merge 处理删除

-- Iceberg 错误配置: 期望 Paimon 的自动 compaction
CREATE TABLE iceberg_table (id INT, name STRING)
TBLPROPERTIES ('write.merge-mode' = 'merge-on-read');
-- 问题: Iceberg 不会自动 compaction,需要手动触发

-- Paimon 错误配置: 期望 Iceberg 的 hidden partition
CREATE TABLE paimon_table (id INT, name STRING, dt STRING)
PARTITIONED BY (dt);
-- 误解: Paimon 的分区是显式的,不支持 hidden partition
```

**生产环境注意事项**:
1. **Paimon 的 Compaction 监控**: 监控 sorted run 数量,避免反压
2. **Iceberg 的 Compaction 调度**: 需要定期触发 compaction,避免小文件堆积
3. **Paimon 的内存配置**: write-buffer-size 需要根据并发 partition+bucket 数量调整
4. **Iceberg 的 Delete File 清理**: 需要定期清理过期的 Delete File

**性能陷阱**:
1. **Paimon 的读放大**: Level 0 文件过多导致读放大,需要及时 compaction
2. **Iceberg 的写放大**: 频繁更新导致大量 Delete File,写入性能差
3. **Paimon 的内存抢占**: 多个 partition+bucket 并发写入时,频繁抢占影响性能
4. **Iceberg 的小文件**: 高频写入导致大量小文件,需要频繁 compaction

### 核心概念解释

**术语定义**:
- **LSM-tree**: Log-Structured Merge-tree,Paimon 的存储引擎,写入优先
- **Copy-on-Write (CoW)**: Iceberg 的写入模式,更新时重写整个文件
- **Merge-on-Read (MoR)**: Iceberg 的写入模式,更新时写 Delete File,读取时合并
- **Delete File**: Iceberg 的删除文件,记录被删除或更新的行
- **Equality Delete**: Iceberg 的相等删除,通过主键删除
- **Position Delete**: Iceberg 的位置删除,通过文件路径+行号删除

**存储引擎对比**:
```
Paimon LSM-tree:
  Partition → Bucket → Levels → Sorted Runs → Data Files
  - 写入: 先进入 SortBuffer,flush 到 Level 0,后台 compaction
  - 更新: 写入新 KeyValue,compaction 时 merge
  - 删除: 写入 DELETE 标记,compaction 时清除

Iceberg CoW:
  Partition → Data Files
  - 写入: 直接写入新文件
  - 更新: 重写整个文件
  - 删除: 重写整个文件
  - 读取: 无额外开销

Iceberg MoR:
  Partition → Data Files + Delete Files
  - 写入: 直接写入新文件
  - 更新: 写入 Equality Delete + 新 Data File
  - 删除: 写入 Position Delete 或 Equality Delete
  - 读取: 需要 apply delete files
```

**Sink 算子拓扑对比**:
```
Paimon:
  RowData → mapToInternalRow → [LocalMerge] → [Partitioner] → Writer → Committer
                                                                  ↕
                                                          Compaction (异步)

Iceberg:
  RowData → IcebergStreamWriter → [Equality Delete Writer] → IcebergFilesCommitter
```

**与其他系统对比**:
- **Hudi**: 也支持 MoR 和 CoW,但架构更复杂,运维成本高
- **Delta Lake**: 主要面向 Spark,Flink 集成较弱
- **Hive**: 不支持更新,只能追加或覆盖

### 设计理念

**为什么 Paimon 选择 LSM-tree**:
1. **写入性能优先**: LSM-tree 的写入性能远超 CoW 和 MoR
   - 原因: 写入先进入内存,批量 flush,避免随机 IO
   - 适用场景: 高频实时更新,CDC 同步

2. **原生 CDC 支持**: LSM-tree 天然支持多版本,适合 CDC
   - 原因: 同一 key 的多个版本通过 sequenceNumber 排序,compaction 时 merge
   - 适用场景: 实时数仓,流式 ETL

3. **自动化 Compaction**: Writer 内异步 compaction,无需额外作业
   - 原因: Compaction 是 LSM-tree 的核心机制,与写入紧密耦合
   - 适用场景: 流式场景,持续写入

**为什么 Iceberg 选择 Delete File**:
1. **简单直接**: 写入路径简单,易于理解和调试
   - 原因: 数据直接写入文件,无额外缓冲层
   - 适用场景: 批处理,低频更新

2. **读取优化**: CoW 模式读取零开销
   - 原因: 更新时重写文件,读取时无需 merge
   - 适用场景: 读多写少,查询性能优先

3. **灵活性**: 支持 CoW 和 MoR 两种模式
   - 原因: 用户可以根据场景选择
   - 适用场景: 需要灵活配置的场景

**权衡取舍**:
```
维度              Paimon (LSM-tree)        Iceberg (Delete File)
─────────────────────────────────────────────────────────────────
写入性能          优秀 (内存写入)           中等 (直接写文件)
更新性能          优秀 (追加写入)           差 (CoW) / 中等 (MoR)
读取性能          中等 (需要 merge)         优秀 (CoW) / 中等 (MoR)
存储空间          中等 (需要 compaction)    差 (CoW) / 中等 (MoR)
运维复杂度        低 (自动 compaction)      中等 (需要手动 compaction)
流式支持          优秀 (原生支持)           中等 (需要额外配置)
批式支持          优秀 (流批一体)           优秀 (原生支持)
跨分区 Upsert     支持 (KEY_DYNAMIC)        不支持
小文件问题        自动解决 (compaction)     需要手动解决
```

**架构演进**:
- **Paimon**: 从 Flink Table Store 演进而来,专注于流批一体和实时更新
- **Iceberg**: 从 Netflix 开源,专注于批处理和数据湖标准化

**业界对比**:
- **RocksDB**: 也使用 LSM-tree,Paimon 借鉴了其设计
- **HBase**: 也使用 LSM-tree,但是分布式存储,架构更复杂
- **Cassandra**: 也使用 LSM-tree,但是 NoSQL 数据库,场景不同

### 12.1 架构层面对比

| 维度 | Paimon | Iceberg |
|---|---|---|
| **存储引擎** | LSM-tree (Merge-tree) | Copy-on-Write / Merge-on-Read |
| **数据组织** | Partition → Bucket → Sorted Runs → Data Files | Partition → Data Files (无 bucket 概念) |
| **更新机制** | LSM merge (写入+后台 compaction) | 行级 Delete File + Equality Delete 或 Position Delete |
| **文件格式** | ORC/Parquet/Avro (可配置) | Parquet/ORC/Avro |
| **元数据管理** | Snapshot → ManifestList → ManifestFile | Snapshot → ManifestList → ManifestFile (相似但不同实现) |

### 12.2 Sink 算子拓扑对比

#### Paimon 拓扑

```
RowData → mapToInternalRow → [LocalMerge] → [Partitioner] → Writer → Committer
                                                                ↕
                                                        Compaction (异步)
```

#### Iceberg 拓扑

```
RowData → IcebergStreamWriter → [Equality Delete Writer] → IcebergFilesCommitter
```

### 12.3 核心机制对比

| 机制 | Paimon | Iceberg |
|---|---|---|
| **数据分区 (Shuffle)** | 按 partition+bucket hash 分区；动态桶模式两次 shuffle | 通常按 partition 分区或不分区 |
| **写入缓冲** | SortBufferWriteBuffer (内存排序缓冲区) | 直接写入文件 (每个 partition 一个 writer) |
| **内存管理** | MemoryPoolFactory 统一管理 + 抢占机制 | 依赖 JVM 堆内存，无跨 writer 内存抢占 |
| **Compaction** | Writer 内异步执行，背压机制 | 独立的 compaction 作业或 maintenance API |
| **本地预合并** | LocalMergeOperator (可选) | 无类似机制 |
| **CDC 支持** | 原生支持 RowKind → MergeFunction | 通过 Equality Delete + Upsert 处理 |
| **Checkpoint 提交** | prepareSnapshotPreBarrier → notifyCheckpointComplete → atomic snapshot | prepareSnapshotPreBarrier → notifyCheckpointComplete → atomic commit |
| **Exactly-Once** | 通过 commitUser + checkpoint ID 保证 | 通过 Flink 快照协议 + commit 幂等保证 |

### 12.4 写入路径深度对比

#### Paimon 的 Write Path

```
InternalRow
  → KeyAndBucketExtractor 提取 partition/bucket/key
  → FileStoreWrite.write(partition, bucket, keyValue)
  → WriterContainer 路由到具体 MergeTreeWriter
  → SortBufferWriteBuffer.put() (内存排序)
  → flush: forEach(merge) → RollingFileWriter → DataFileMeta
  → 异步触发 Compaction
```

**特点**:
1. **写放大控制**: 数据先进入排序缓冲区，flush 时做预合并，减少 Level 0 文件大小
2. **按 bucket 隔离**: 每个 partition+bucket 有独立的 MergeTreeWriter 和 Levels
3. **流式友好**: LSM 结构天然支持流式写入，compaction 在后台异步进行

#### Iceberg 的 Write Path

```
RowData
  → PartitionedFanoutWriter / UnpartitionedWriter
  → 按 partition 路由到 BaseTaskWriter
  → Parquet/ORC AppendWriter 直接写文件
  → (如有主键) EqualityDeleteWriter 写 delete file
  → close() 时 flush 所有 writer → DataFile 列表
```

**特点**:
1. **简单直接**: 数据直接写入目标文件，无额外缓冲层
2. **写后不合并**: 每次写入产生新的数据文件，不在写入路径做 merge
3. **批式优先**: 设计哲学偏向批处理，流式需要额外的 maintenance 来清理小文件

### 12.5 更新操作对比

| 操作 | Paimon | Iceberg (MoR) | Iceberg (CoW) |
|---|---|---|---|
| **INSERT** | 写入 SortBuffer → flush 到 Level 0 | 写入新 Data File | 写入新 Data File |
| **UPDATE** | 写入新 KV (相同 key) → compaction 时 merge | 写入 Equality Delete + 新 Data File | 重写整个 Data File |
| **DELETE** | 写入 DELETE 标记的 KV → compaction 时清除 | 写入 Position Delete 或 Equality Delete | 重写整个 Data File |
| **读取代价** | 需要 merge 多个 sorted run | 需要 apply delete files | 无额外代价 |
| **写入代价** | 低（只追加） | 中（追加 + delete file） | 高（重写整个文件） |
| **Compaction 代价** | 后台异步，可调节 | 独立作业 | 写入时已完成 |

### 12.6 适用场景总结

| 场景 | 推荐方案 | 原因 |
|---|---|---|
| **高频实时更新** | Paimon | LSM-tree 写入性能优异，原生 CDC 支持 |
| **低频批量覆盖** | Iceberg | Copy-on-Write 模式读取零开销 |
| **流批一体 Lakehouse** | Paimon | 统一的流批读写路径，实时 changelog 生成 |
| **已有 Iceberg 生态** | Iceberg | 兼容性好，工具链成熟 |
| **大规模追加写入** | 两者均可 | 追加场景两者性能接近 |
| **跨分区 Upsert** | Paimon (KEY_DYNAMIC) | Iceberg 不原生支持跨分区 upsert |

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
