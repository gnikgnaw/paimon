# Flink 写入 Paimon 源码流程深度分析

> 基于 Apache Paimon 1.5-SNAPSHOT (commit: 7c93bd720)
> 分析日期: 2026-04-15

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
        throw new UnsupportedOperationException("...");
    }
    // 2. 创建 PaimonDataStreamSinkProvider，Lambda 内构建 FlinkSinkBuilder
    return new PaimonDataStreamSinkProvider(
        (dataStream) -> {
            FlinkSinkBuilder builder = createSinkBuilder();
            builder.forRowData(dataStream);
            // 3. 可选：Clustering 排序优化（仅批模式 + BUCKET_UNAWARE）
            builder.clusteringIfPossible(...);
            // 4. 可选：INSERT OVERWRITE
            if (overwrite) builder.overwrite(staticPartitions);
            // 5. 可选：自定义并行度
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
        // 主键表默认 Upsert 模式：过滤掉 UPDATE_BEFORE
        // 特殊情况：INPUT changelog producer / AGGREGATE merge engine 保留全部 RowKind
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

### 3.1 核心分发代码

```java
// FlinkSinkBuilder.java:207-243
public DataStreamSink<?> build() {
    // Step 1: RowData → InternalRow
    DataStream<InternalRow> input = mapToInternalRow(this.input, table.rowType(), ...);

    // Step 2: 可选 LocalMerge 算子（主键表 + local-merge-buffer-size > 0）
    if (table.coreOptions().localMergeEnabled() && table.schema().primaryKeys().size() > 0) {
        input = input.forward().transform("local merge", ..., new LocalMergeOperator.Factory(...));
    }

    // Step 3: 根据 BucketMode 路由到不同 Sink
    BucketMode bucketMode = table.bucketMode();
    switch (bucketMode) {
        case POSTPONE_MODE:   return buildPostponeBucketSink(input);
        case HASH_FIXED:      return buildForFixedBucket(input);
        case HASH_DYNAMIC:    return buildDynamicBucketSink(input, false);
        case KEY_DYNAMIC:     return buildDynamicBucketSink(input, true);
        case BUCKET_UNAWARE:  return buildUnawareBucketSink(input);
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

#### BUCKET_UNAWARE 模式

```java
// FlinkSinkBuilder.java:317-332
private DataStreamSink<?> buildUnawareBucketSink(DataStream<InternalRow> input) {
    checkArgument(table.primaryKeys().isEmpty(), "Unaware bucket mode only works with append-only table.");
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
```

**为什么需要 LocalMerge**: 解决主键数据倾斜问题。在 shuffle 之前先在本地做预合并，减少网络传输量。例如一条记录被连续更新 100 次，LocalMerge 只向下游发送最终值。

**好处**:
1. 显著减少 shuffle 数据量，降低网络开销
2. 减轻下游 writer 的写放大
3. 支持两种 LocalMerger 实现：`HashMapLocalMerger`（固定长度值字段）和 `SortBufferLocalMerger`（变长字段），自动选择

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
public List<Committable> prepareCommit(boolean waitCompaction, long checkpointId) {
    checkSuccessfulFullCompaction();
    // 记录本 checkpoint 期间修改的 bucket
    if (!currentWrittenBuckets.isEmpty()) {
        writtenBuckets.computeIfAbsent(checkpointId, k -> new HashSet<>())
                      .addAll(currentWrittenBuckets);
        currentWrittenBuckets.clear();
    }
    // 判断是否到达 full compaction 触发点
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
// CommitterOperator.java:190-193
public void notifyCheckpointComplete(long checkpointId) throws Exception {
    super.notifyCheckpointComplete(checkpointId);
    commitUpToCheckpoint(endInput ? END_INPUT_CHECKPOINT_ID : checkpointId);
}

// CommitterOperator.java:195-218
private void commitUpToCheckpoint(long checkpointId) throws Exception {
    NavigableMap<Long, GlobalCommitT> headMap =
        committablesPerCheckpoint.headMap(checkpointId, true);
    List<GlobalCommitT> committables = committables(headMap);
    // ...
    committer.commit(committables);  // → StoreCommitter.commit()
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

### 7.4 FileStoreCommitImpl 的原子提交

**源码路径**: `paimon-core/src/main/java/org/apache/paimon/operation/FileStoreCommitImpl.java`

提交流程概述：
1. **冲突检查**: 检查要删除的文件是否仍然存在，新文件的 key 范围是否与已有文件冲突
2. **写入 Manifest**: 将 `ManifestEntry` 写入 `ManifestFile`，汇总为 `ManifestFileMeta`
3. **写入 ManifestList**: 将新旧 `ManifestFileMeta` 汇总写入 `ManifestList`
4. **写入 Snapshot**: 原子性地创建新的 Snapshot 文件（通过文件系统的 atomic rename 或外部 `SnapshotCommit`）
5. **重试机制**: 如果冲突失败，从最新 snapshot 重新扫描并重试

**为什么 Committer 必须单并行度**: Snapshot 的创建需要全局有序——每个 snapshot ID 严格递增，只有单点才能保证这一点。

---

## 8. 内存管理

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
