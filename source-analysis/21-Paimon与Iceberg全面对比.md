# Apache Paimon vs Apache Iceberg 全面深度对比

> **分析基础**: Paimon 1.5-SNAPSHOT 源码 (commit: 55f4fd175)，Iceberg 1.4.x 源码及公开设计文档
>
> **分析日期**: 2026-04-21

---

## 目录

- [1. 设计哲学对比](#1-设计哲学对比)
- [2. 存储模型对比](#2-存储模型对比)
- [3. 元数据管理对比](#3-元数据管理对比)
- [4. 主键更新对比](#4-主键更新对比)
- [5. Compaction 对比](#5-compaction-对比)
- [6. 流式能力对比](#6-流式能力对比)
- [7. Flink 集成对比](#7-flink-集成对比)
- [8. Spark 集成对比](#8-spark-集成对比)
- [9. 索引能力对比](#9-索引能力对比)
- [10. 小文件治理对比](#10-小文件治理对比)
- [11. 版本管理对比](#11-版本管理对比)
- [12. 生态系统对比](#12-生态系统对比)
- [13. 选型建议](#13-选型建议)

---

## 1. 设计哲学对比

### 1.1 核心定位

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 一句话定位 | 面向流的 LSM-Tree 实时湖仓格式 | 面向批的不可变开放表格式 |
| 诞生背景 | 从 Flink Table Store 演化而来，天然为流式场景设计 | Netflix 为解决 Hive 元数据瓶颈而设计的通用表格式 |
| 核心数据结构 | LSM Merge-Tree（主键表）+ Append-Only（追加表） | 不可变数据文件 + Delete File（删除文件） |
| 设计出发点 | "数据怎样被高频更新" | "数据怎样被可靠地组织和查询" |
| 主键地位 | 一等公民——主键决定存储引擎选择 | 非核心——主键仅作为写入去重的辅助手段 |

### 1.2 为什么设计不同

**Paimon 选择 LSM-Tree 的根本原因**：它要解决的核心问题是"流式数据高频写入并快速可见"。在 Flink 的 Checkpoint 间隔（秒级到分钟级）内，一个 Bucket 可能接收到大量更新。LSM-Tree 将随机写转化为顺序写，天然适合这种高写入吞吐场景。这一点从 `MergeTreeWriter` 源码可以清楚看到：数据先写入内存中的 `WriteBuffer`，达到阈值后 flush 为有序文件到 Level-0，再由后台 `MergeTreeCompactManager` 异步合并到更高层级。这个设计使得写入路径几乎没有随机 I/O。

**Iceberg 选择不可变文件的根本原因**：它要解决的核心问题是"在分布式存储上实现可靠的表语义"。Iceberg 的哲学是"一旦写入，永不修改"——每个数据文件是不可变的，修改通过追加 Delete File（标记哪些行被删除）来实现。这种设计最大限度地利用了对象存储的特性（写入一次、读取多次、不支持原地更新），并且天然避免了并发写入时的文件损坏问题。

### 1.3 各自的好处与代价

**Paimon 的好处**：
- 流式写入延迟低（秒级到分钟级可见）
- 更新操作的吞吐量高（LSM-Tree 将随机写转顺序写）
- 内置 Changelog 生成能力，天然支持 CDC 下游消费
- 主键表的点查效率高（Lookup 机制）

**Paimon 的代价**：
- 读放大：查询时可能需要合并多个 Level 的文件（Merge-on-Read）
- 系统复杂度高：Compaction 策略调优是运维核心负担
- 与引擎耦合度较高：虽然支持 Spark/Hive，但与 Flink 的集成最为紧密

**Iceberg 的好处**：
- 引擎无关性强：不依赖任何特定计算引擎
- 批查询性能优秀：数据文件不可变，利于列存优化和缓存
- 生态广泛：几乎所有主流引擎都有 Iceberg 连接器
- Schema/Partition Evolution 设计成熟

**Iceberg 的代价**：
- 更新性能差：每次更新需要写入 Delete File，累积后严重影响读性能
- 实时性不足：缺乏原生流式 Changelog，增量读取能力有限
- Compaction 依赖外部触发，运维成本转嫁到用户

---

## 2. 存储模型对比

### 2.1 模型架构

```
Paimon 主键表 (KeyValueFileStore)              Iceberg 表
┌─────────────────────────────┐                ┌──────────────────────────────┐
│  Partition                  │                │  Partition                   │
│  ├── Bucket 0               │                │  ├── data-00001.parquet      │
│  │   ├── Level-0 (无序文件)  │                │  ├── data-00002.parquet      │
│  │   ├── Level-1 (有序文件)  │                │  ├── delete-00001.parquet    │
│  │   ├── Level-2 ...        │                │  │   (Position Delete)       │
│  │   └── Level-max          │                │  ├── eq-delete-001.parquet   │
│  ├── Bucket 1               │                │  │   (Equality Delete)       │
│  │   └── ...                │                │  └── dv-00001.puffin         │
│  └── ...                    │                │      (Deletion Vector)       │
└─────────────────────────────┘                └──────────────────────────────┘
```

### 2.2 写放大/读放大/空间放大的三角权衡

| 放大类型 | Paimon (LSM-Tree) | Iceberg (不可变文件 + Delete File) |
|----------|-------------------|-----------------------------------|
| **写放大** | 中等偏高。每次 Compaction 需要重写文件，Universal Compaction 的写放大约为 `size_ratio` 的函数（源码 `UniversalCompaction` 中 `maxSizeAmp` 默认 200%）。但写入本身是顺序的，无随机写 | 低。数据文件一旦写入不再修改，更新只追加 Delete File。但 Compaction（RewriteDataFiles）时写放大极高——需要重写整个数据文件 |
| **读放大** | 低到中等。Compaction 完成后读取单层即可；未 Compact 时需要合并多个 SortedRun（从 `Levels` 源码看 `numberOfSortedRuns()` 控制）。Lookup 模式下通过缓存和 Hash 索引加速点查 | 中等到高。每个 Data File 都需要关联检查 Delete File，特别是 Position Delete 和 Equality Delete 累积后会严重拖慢读取（需要逐行匹配或合并） |
| **空间放大** | 低到中等。LSM-Tree 的过期数据在 Compaction 后被清理，`maxSizeAmp` 参数控制空间放大上限 | 高。Delete File 不会物理删除数据，只是标记删除。在 Compaction 之前，原数据文件和 Delete File 同时占用空间 |

### 2.3 为什么 Paimon 需要 Bucket 而 Iceberg 不需要

Paimon 的 `Bucket` 是 LSM-Tree 的组织单元。从 `KeyValueFileStore.bucketMode()` 源码可以看到三种模式：
- `HASH_FIXED`：固定桶数，主键哈希取模分配
- `HASH_DYNAMIC` / `KEY_DYNAMIC`：动态桶，根据数据量自动扩展
- `POSTPONE_MODE`：延迟分桶模式

Bucket 的本质目的是**限制 Merge-on-Read 的扫描范围**。每个 Bucket 维护独立的 LSM-Tree，同一主键的所有版本必定落在同一个 Bucket 中，这样合并操作只需在一个 Bucket 内进行。

Iceberg 不需要 Bucket 是因为其数据文件是不可变的，更新通过全局的 Delete File 标记。没有 LSM-Tree 的层级结构，自然不需要按桶组织。

### 2.4 Paimon 追加表 vs 主键表

Paimon 提供两种 FileStore 实现（从源码可见）：

- **`KeyValueFileStore`**：主键表，使用 LSM Merge-Tree，支持更新、删除、聚合
- **`AppendOnlyFileStore`**：追加表，无主键，文件直接追加，不做合并

追加表模式下 Paimon 的行为更接近 Iceberg——数据文件一旦写入不再修改。但即使在这种模式下，Paimon 仍然提供了内置的自动 Compaction（从 `AppendOnlyFileStore.newWrite()` 源码中可以看到 `BucketedAppendFileStoreWrite` 和 `AppendFileStoreWrite` 均包含 Compaction 逻辑），而 Iceberg 的追加写入完全没有自动 Compaction。

---

## 3. 元数据管理对比

### 3.1 元数据层级结构

```
Paimon 元数据层级                              Iceberg 元数据层级
┌─────────────────┐                            ┌──────────────────────┐
│   snapshot-N    │ (JSON文件)                  │   metadata.json      │ (JSON文件)
│   ├── baseManifestList                       │   (version=v1/v2/v3/v4)
│   ├── deltaManifestList                      │   ├── schemas[]
│   ├── changelogManifestList                  │   ├── partition-specs[]
│   └── indexManifest                          │   ├── sort-orders[]
└────────┬────────┘                            │   ├── snapshots[]
         │                                     │   │   └── snap-N.avro
┌────────▼────────┐                            │   ├── snapshot-log[]
│  ManifestList   │ (Avro文件)                  │   └── refs{}
│  └── ManifestFileMeta[]                      └──────────┬───────────┘
│      ├── fileName                                       │
│      ├── fileSize                            ┌──────────▼───────────┐
│      ├── numAddedFiles                       │   manifest-list.avro │ (Avro文件)
│      ├── numDeletedFiles                     │   └── ManifestFile[]
│      └── partitionStats                      │       ├── path
└────────┬────────┘                            │       ├── length
         │                                     │       ├── partition_spec_id
┌────────▼────────┐                            │       └── partitions (summary)
│  ManifestFile   │ (Avro文件)                  └──────────┬───────────┘
│  └── ManifestEntry[]                                     │
│      ├── _KIND (ADD/DELETE)                  ┌──────────▼───────────┐
│      ├── _PARTITION                          │   manifest-N.avro    │ (Avro文件)
│      ├── _BUCKET                             │   └── ManifestEntry[]
│      └── _FILE (DataFileMeta)                │       ├── status (ADDED/DELETED/EXISTING)
└─────────────────┘                            │       ├── data_file
                                               │       └── partition
                                               └──────────────────────┘
```

### 3.2 Paimon 为什么要做 base/delta 分离

这是 Paimon 元数据设计中最关键的创新之一。从 `Snapshot.java` 源码可以看到三个 manifest list 字段：

```java
protected final String baseManifestList;    // 全量基线
protected final String deltaManifestList;   // 增量变更
protected final String changelogManifestList; // changelog (可选)
```

**设计动机**：

1. **加速增量读取**：流式消费场景下，消费者只需读取 `deltaManifestList` 即可获取自上次快照以来的所有新增/删除文件，无需扫描整个 `baseManifestList`。从 `ManifestList.readDeltaManifests()` 源码可以看到这一快速路径。

2. **加速快照过期**：Snapshot 过期时只需处理 delta 部分引用的文件，不需要扫描全量 base manifest。

3. **控制 Manifest 膨胀**：base manifest list 可以独立做合并（类似 Compaction），将多次增量变更合并为一个紧凑的全量视图。对应 `MANIFEST_FULL_COMPACTION_FILE_SIZE` 配置项。

**Iceberg 的做法**：Iceberg 的每个 Snapshot 指向一个 manifest-list，其中包含该快照时刻所有有效的 manifest 文件。Iceberg 不区分 base/delta，而是在 manifest entry 上标记 status（ADDED/DELETED/EXISTING）。增量读取时需要比较两个 Snapshot 的 manifest list 差异。

**各自的权衡**：

| 维度 | Paimon (base/delta 分离) | Iceberg (统一 manifest list) |
|------|--------------------------|------------------------------|
| 增量读取效率 | O(delta 大小)，极快 | O(全量 manifest 大小)，需要 diff |
| 全量读取效率 | 需要合并 base + delta | 直接读取一个 manifest list |
| 元数据复杂度 | 更高，需要维护两套 | 更简单，单一抽象 |
| Snapshot 过期效率 | 更高，只需处理 delta | 需要扫描全量 manifest |

### 3.3 Snapshot 格式差异

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| Snapshot 格式 | JSON 文件 (`snap-{id}`) | Avro 文件 (`snap-{id}-{uuid}.avro`) |
| 元数据入口 | `snapshot/` 目录下的 LATEST/EARLIEST hint 文件 | `metadata/` 目录下的 `version-hint.text` 或 `metadata.json` |
| 版本化 | Snapshot 自身有 version 字段 (当前 v3) | TableMetadata 有 format-version (v1/v2/v3/v4) |
| Schema 管理 | 独立的 SchemaManager，Schema 存为独立 JSON 文件 | Schema 内嵌在 `metadata.json` 中 |
| 额外元数据 | watermark、commitUser、commitIdentifier、statistics、properties、nextRowId | operation、summary map |

Paimon 的 Snapshot 使用 JSON 而非 Avro 有一个实际好处：人类可读，便于调试和运维。但代价是文件可能略大于 Avro 的二进制格式。

---

## 4. 主键更新对比

### 4.1 Paimon 的 MergeFunction 体系

Paimon 的主键更新机制是其核心竞争力。从源码中可以梳理出完整的 MergeFunction 体系：

**4 种合并引擎** (`CoreOptions.MergeEngine`)：

| 引擎 | 实现类 | 行为 | 适用场景 |
|------|--------|------|----------|
| `deduplicate` | `DeduplicateMergeFunction` | 保留最后一条记录 | 通用 Upsert |
| `partial-update` | `PartialUpdateMergeFunction` | 合并非 null 字段 | 多源写入同一行的不同列 |
| `aggregation` | `AggregateMergeFunction` | 按字段聚合 | 实时指标聚合 |
| `first-row` | `FirstRowMergeFunction` | 保留第一条记录 | 去重保留最早到达的数据 |

**20+ 种字段聚合器** (`FieldAggregator` 子类)：

从源码发现的完整列表：`FieldSumAgg`、`FieldMinAgg`、`FieldMaxAgg`、`FieldProductAgg`、`FieldCountAgg`（通过 `FieldPrimaryKeyAgg`）、`FieldFirstValueAgg`、`FieldLastValueAgg`、`FieldFirstNonNullValueAgg`、`FieldLastNonNullValueAgg`、`FieldListaggAgg`、`FieldCollectAgg`、`FieldBoolAndAgg`、`FieldBoolOrAgg`、`FieldMergeMapAgg`、`FieldMergeMapWithKeyTimeAgg`、`FieldRoaringBitmap32Agg`、`FieldRoaringBitmap64Agg`、`FieldNestedUpdateAgg`、`FieldNestedPartialUpdateAgg`、`FieldHllSketchAgg`、`FieldThetaSketchAgg`、`FieldIgnoreRetractAgg`。

**LookupMergeFunction**：特殊的包装器，通过 Lookup 机制从已有数据中查找旧值，与新值合并，实现高效的增量更新。这是 `changelog.producer = lookup` 模式的基础。

### 4.2 Iceberg 的删除机制

Iceberg 的更新本质上是"先删后加"。从 `FileContent` 枚举可以看到：

```java
public enum FileContent {
    DATA(0),              // 数据文件
    POSITION_DELETES(1),  // 按位置删除
    EQUALITY_DELETES(2),  // 按等值条件删除
    DATA_MANIFEST(3),     // 数据 manifest
    DELETE_MANIFEST(4);   // 删除 manifest
}
```

**三种删除方式**：

| 方式 | 实现 | 性能 | 适用场景 |
|------|------|------|----------|
| Position Delete | 记录 (file_path, row_position) 对 | 读取时高效（按位跳过） | 精确已知位置的行删除 |
| Equality Delete | 记录需要删除的行的等值条件 | 读取时低效（需要逐行匹配） | 不知道具体位置的行删除 |
| Deletion Vector | Bitmap 标记被删除的行号 | 读取时较高效（Bitmap 过滤） | V3 格式新增的优化机制 |

### 4.3 性能差异深度分析

**写入性能**：

| 场景 | Paimon | Iceberg |
|------|--------|---------|
| 单条 Upsert | 写入内存 Buffer，O(1) | 需要写一个 Delete File + 一个 Data File（或追加到现有文件） |
| 批量 Upsert | 内存 Buffer 批量 flush，极高效 | 每批次产生一组 Delete File + Data File |
| 高频小批次更新 | 内存吸收，Compaction 异步合并 | 产生大量小 Delete File，严重碎片化 |

**读取性能**：

| 场景 | Paimon | Iceberg |
|------|--------|---------|
| 全表扫描（Compaction 后）| 读取 Level-max 即可 | 读取数据文件，无需额外处理 |
| 全表扫描（未 Compact）| 需要 Merge-on-Read 合并多层 | 需要 Apply Delete File |
| 点查 | Lookup + Hash 索引，O(1) | 需要扫描数据文件 + Delete File |
| 范围查询 | LSM-Tree 有序性支持高效范围扫描 | 依赖 Parquet/ORC 的统计信息 |

**核心差异根因**：Paimon 的 LSM-Tree 在写入时就对数据做了排序和组织，使得后续的合并是高效的归并操作。Iceberg 的 Delete File 是"事后补丁"——每次更新都在数据上贴一个补丁，补丁累积越多，读取越慢。

### 4.4 各自的适用场景

- **Paimon 更适合**：高频更新（CDC 同步、实时聚合）、需要复杂合并语义（Partial Update、Aggregation）、流式更新+实时查询的场景
- **Iceberg 更适合**：低频更新（每天几次批量更新）、以追加为主的日志类数据、需要跨引擎读写的通用分析场景

---

## 5. Compaction 对比

### 5.1 Paimon 的内置 Universal Compaction

从 `UniversalCompaction.java` 源码可以看到，Paimon 实现了 RocksDB 风格的 Universal Compaction：

```
选择策略优先级：
1. EarlyFullCompaction（定时触发全量合并）
2. pickForSizeAmp（空间放大超阈值时触发）
3. pickForSizeRatio（相邻 SortedRun 大小比例失衡时触发）
4. pickForFileNum（SortedRun 数量超阈值时触发）
```

**关键参数**（从源码提取）：

| 参数 | 含义 | 默认值 |
|------|------|--------|
| `maxSizeAmp` | 最大空间放大百分比 | 200% |
| `sizeRatio` | 触发合并的大小比例 | 1 |
| `numRunCompactionTrigger` | 触发合并的 SortedRun 数 | 5 |
| `numSortedRunStopTrigger` | 阻塞写入的 SortedRun 数 | numRunCompactionTrigger + 1 |

**Compaction 执行方式**：从 `MergeTreeCompactManager` 源码可以看到，Compaction 在独立的 `ExecutorService` 线程中异步执行，写入线程在 SortedRun 数量超过 `numSortedRunStopTrigger` 时会被阻塞（反压机制）。

**额外的 Compaction 策略**：
- `ForceUpLevel0Compaction`：强制将 Level-0 文件合并到上层
- Off-peak hours Compaction：支持配置非高峰时段执行全量 Compaction
- `EarlyFullCompaction`：支持按间隔触发全量 Compaction

### 5.2 Iceberg 的外部 Compaction

Iceberg 不内置 Compaction，而是通过 `Actions` API 提供外部触发的重写操作：

- **`RewriteDataFiles`**：重写数据文件，合并小文件、应用 Delete File、重新排序
- **`RewriteManifests`**：重写 Manifest 文件，合并小 Manifest
- **`RemoveDanglingDeleteFiles`**：清理无用的 Delete File
- **`RewritePositionDeleteFiles`**：重写 Position Delete 文件
- **`ConvertEqualityDeleteFiles`**：将 Equality Delete 转换为更高效的格式

### 5.3 自动化程度与运维成本对比

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 触发方式 | 写入时自动触发，内置策略 | 需要外部调度触发（Spark Action/定时任务） |
| 配置复杂度 | 通过 CoreOptions 配置参数即可 | 需要编写 Action 调用代码或配置调度系统 |
| 反压机制 | 内置——SortedRun 过多时阻塞写入 | 无——Delete File 可以无限累积 |
| Manifest Compaction | 内置——提交时自动触发 | 需要单独调用 `RewriteManifests` |
| 运维监控 | 内置 `CompactionMetrics` 监控 | 需要外部监控 Delete File 数量和大小 |
| 成本 | 占用写入节点的 CPU 和内存 | 需要额外的计算资源（Spark 任务） |

**为什么 Iceberg 选择外部 Compaction**：

Iceberg 作为开放表格式，不绑定任何计算引擎。内置 Compaction 意味着需要一个常驻的计算进程，这与 Iceberg "格式即规范"的设计哲学矛盾。代价是用户需要自己管理 Compaction 调度，这在生产环境中是一个非平凡的运维负担。

**为什么 Paimon 选择内置 Compaction**：

Paimon 的 LSM-Tree 如果不做 Compaction，读性能会快速退化（SortedRun 累积）。因此 Compaction 不是可选的优化，而是系统正确运行的前提。从 `shouldWaitForLatestCompaction()` 可以看到，当 SortedRun 数量超过阈值时写入会被阻塞，这是一个硬性约束。

---

## 6. 流式能力对比

### 6.1 Paimon 的原生 Changelog 体系

Paimon 提供 4 种 Changelog 生成模式（`ChangelogProducer` 枚举）：

| 模式 | 含义 | 实现机制 | 适用场景 |
|------|------|----------|----------|
| `NONE` | 不生成 Changelog | 无额外开销 | 只需要全量查询 |
| `INPUT` | 直接使用输入数据作为 Changelog | 在 `MergeTreeWriter.newFilesChangelog` 中记录 | 输入流本身就是完整的 CDC 流 |
| `FULL_COMPACTION` | Full Compaction 时生成 Changelog | `FullChangelogMergeFunctionWrapper` 对比新旧值 | 对延迟不敏感但需要 Changelog |
| `LOOKUP` | 通过 Lookup 旧值生成完整 Changelog | `LookupMergeFunction` 在写入时查找旧值 | 需要低延迟的完整 CDC 输出 |

**增量读取机制**：Paimon 的流式读取通过 `deltaManifestList` 实现快速增量发现。`ContinuousFileSplitEnumerator` 周期性扫描新 Snapshot 的 delta manifest，将新文件分配给 Reader。这个过程只读取增量数据，不需要扫描全量元数据。

**Changelog 存储**：Changelog 数据存储在独立的 `changelogManifestList` 指向的文件中，与主数据分离。这意味着：
- Changelog 可以有独立的生命周期（TTL）
- 消费 Changelog 不影响主数据的读写性能
- Changelog 文件格式与主数据文件相同

### 6.2 Iceberg 的增量读取能力

Iceberg 提供两种增量扫描接口：

- **`IncrementalAppendScan`**：只扫描两个 Snapshot 之间新增（APPEND）的数据文件
- **`IncrementalChangelogScan`**：扫描两个 Snapshot 之间的变更，产出 `ChangelogScanTask`（含 `ChangelogOperation`: INSERT/DELETE/UPDATE_BEFORE/UPDATE_AFTER）

**局限性**：
1. `IncrementalAppendScan` 只能看到追加操作，看不到更新和删除
2. `IncrementalChangelogScan` 虽然支持变更，但底层实现依赖 Delete File 的对比，性能不如 Paimon 的原生 Changelog
3. 没有内置的 Watermark 机制——Paimon 的 Snapshot 中有 `watermark` 字段，可以与 Flink 的事件时间语义天然对齐

### 6.3 实时性对比

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 最小可见延迟 | Flink Checkpoint 间隔（秒级） | Spark/Flink 作业的 Commit 间隔 |
| Changelog 完整性 | 完整的 CDC 流（INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE） | 依赖计算引擎重建 |
| Watermark 支持 | 原生内嵌在 Snapshot 中 | 无原生支持 |
| Consumer 进度管理 | 内置 `ConsumerManager`，记录每个消费者的消费位点 | 无内置机制，依赖外部记录 |
| 流式消费背压 | 通过 SortedRun 数量控制写入速度 | 无背压机制 |

**为什么 Paimon 的流式能力更强**：Paimon 从设计之初就把流式消费作为一等场景。Snapshot 中的 `commitIdentifier` 保证了有序性（源码注释："snapshot A has a smaller commitIdentifier than snapshot B, then snapshot A must be committed before snapshot B"），`deltaManifestList` 提供了高效的增量发现，`changelogManifestList` 提供了完整的变更记录。这三者组合形成了完整的流式消费协议。

**为什么 Iceberg 的流式能力有限**：Iceberg 的 Snapshot 记录的是某个时间点的表状态，而不是状态之间的变更。要重建变更，需要对比两个 Snapshot 的 Manifest 差异，这是一个 O(全量 manifest) 的操作。更重要的是，Iceberg 的 Delete File 机制使得"变更"的语义变得模糊——一个 Position Delete 可能对应多个原始行的更新，需要额外的计算才能还原出 CDC 流。

---

## 7. Flink 集成对比

### 7.1 架构对比

**Paimon 的 Flink 架构**：

```
Paimon Flink 写入管道：
  DataStream<T>
    → Writer Operator (每个并行度维护 MergeTreeWriter 或 AppendWriter)
       ├── 每个 Checkpoint 间隔 flush WriteBuffer → Level-0 文件
       └── 异步 Compaction (在 Writer 线程池中)
    → Committer Operator (单并行度，协调全局提交)
       ├── 收集所有 Writer 的 CommitMessage
       ├── 调用 FileStoreCommitImpl.commit()
       └── 原子创建新 Snapshot

Paimon Flink 读取管道：
  Source (SplitEnumerator + SourceReader)
    ├── StaticFileStoreSource (批模式)
    │   └── StaticFileStoreSplitEnumerator 一次性枚举所有 Split
    └── ContinuousFileStoreSource (流模式)
        └── ContinuousFileSplitEnumerator 持续监听新 Snapshot
```

**Iceberg 的 Flink 架构** (概要)：

```
Iceberg Flink 写入管道：
  DataStream<RowData>
    → IcebergStreamWriter (写数据文件 + Delete File)
    → IcebergFilesCommitter (单并行度，调用 Iceberg Transaction API 提交)

Iceberg Flink 读取管道：
  Source (SplitEnumerator + SourceReader)
    ├── 批模式：一次性计划所有 FileScanTask
    └── 流模式：周期性扫描新 Snapshot（需要配置 monitor-interval）
```

### 7.2 Checkpoint 提交机制差异

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 提交触发 | Flink Checkpoint 完成时 | Flink Checkpoint 完成时 |
| 提交内容 | `ManifestCommittable` (包含 DataIncrement + CompactIncrement + IndexIncrement) | DataFile + DeleteFile 列表 |
| 原子性保证 | `FileStoreCommitImpl` 通过文件系统原子重命名 (`rename`) | Iceberg `Transaction` API |
| 提交冲突处理 | 内置 OCC 冲突检测（`ConflictDetection`），自动重试 | Iceberg 内置 OCC，自动重试 |
| Savepoint 支持 | `AutoTagForSavepointCommitterOperator` 自动创建 Tag | 无内置 Savepoint Tag 机制 |
| Exactly-once 保证 | 通过 `commitIdentifier` 去重 | 通过 Snapshot 幂等性 |

**关键差异**：Paimon 的 Committer 不仅提交数据文件，还同时提交 Compaction 结果和索引更新。从 `CommitMessage`/`CommitMessageImpl` 可以看到它包含 `DataIncrement`（新数据文件）、`CompactIncrement`（Compaction 结果）。这意味着 Compaction 结果和数据写入在同一个事务中提交，保证了一致性。

### 7.3 CDC 同步能力

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 内置 CDC 同步 | 提供 `FlinkCdcSyncDatabaseSinkBuilder` 等完整的 CDC 管道 | 无内置 CDC 同步工具 |
| 支持的 CDC 源 | MySQL、Kafka、MongoDB 等（通过 paimon-flink-cdc 模块） | 需要通过外部工具（Flink CDC Connector）写入 |
| 库级别同步 | 支持整库同步，自动建表、Schema Evolution | 不支持，需要逐表配置 |
| Schema Evolution | CDC 同步时自动感知上游 Schema 变更 | 需要手动处理 |

**Paimon CDC 同步的独特优势**：Paimon 的 `paimon-flink-cdc` 模块提供了一站式的 CDC 同步能力，可以从 MySQL 等源头直接同步整个数据库到 Paimon，包括自动建表、Schema Evolution、并行同步多表。这是 Paimon 作为"流式湖仓"定位的直接体现。Iceberg 要实现类似功能需要组合多个外部组件。

---

## 8. Spark 集成对比

### 8.1 DataSource V2 实现

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 实现入口 | `SparkTable` (Scala) 实现 `SupportsRowLevelOperations` | `SparkTable` 实现 `SupportsRead`/`SupportsWrite` |
| 读取实现 | `PaimonScan` → `PaimonScanBuilder` | `SparkBatchScan` + `SparkScanBuilder` |
| 写入实现 | V2 Write (Copy-on-Write: `PaimonSparkCopyOnWriteOperation`) | V2 Write (MergeOnRead + CopyOnWrite) |
| Spark 版本 | 3.2 - 4.0 (版本特定子模块) | 3.3 - 3.5 |
| 流式读取 | 通过 Spark Structured Streaming 支持 | 通过 Spark Structured Streaming 支持 |

### 8.2 Procedure 丰富度

**Paimon Spark Procedures**（从源码统计约 30 个）：
- **Compaction 类**: `CompactProcedure`、`CompactDatabaseProcedure`、`CompactManifestProcedure`
- **Tag/Branch 类**: `CreateTagProcedure`、`DeleteTagProcedure`、`ReplaceTagProcedure`、`RenameTagProcedure`、`CreateBranchProcedure`、`DeleteBranchProcedure`、`RenameBranchProcedure`、`FastForwardProcedure`
- **版本管理类**: `RollbackProcedure`、`RollbackToTimestampProcedure`、`RollbackToWatermarkProcedure`
- **过期清理类**: `ExpireSnapshotsProcedure`、`ExpirePartitionsProcedure`、`ExpireTagsProcedure`、`RemoveOrphanFilesProcedure`、`PurgeFilesProcedure`
- **迁移类**: `MigrateTableProcedure`、`MigrateDatabaseProcedure`
- **索引类**: `RewriteFileIndexProcedure`、`CreateGlobalIndexProcedure`、`DropGlobalIndexProcedure`
- **Consumer 类**: `ResetConsumerProcedure`、`ClearConsumersProcedure`
- **其他**: `RepairProcedure`、`RescaleProcedure`、`MarkPartitionDoneProcedure`、`CopyFilesProcedure`

**Iceberg Spark Procedures**：
- `rewrite_data_files`、`rewrite_manifests`、`expire_snapshots`、`remove_orphan_files`
- `rollback_to_snapshot`、`rollback_to_timestamp`、`set_current_snapshot`
- `create_tag`、`drop_tag`、`fast_forward`、`cherrypick_snapshot`
- `rewrite_position_delete_files`、`register_table`
- 约 15-20 个

**对比结论**：Paimon 的 Procedure 数量约为 Iceberg 的 2-3 倍，尤其在 Branch 管理、Consumer 管理、文件索引、CDC 迁移等方面更加丰富。

### 8.3 SQL Extensions

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| Time Travel | `SELECT * FROM t VERSION AS OF 1` / `TIMESTAMP AS OF '...'` | `SELECT * FROM t TIMESTAMP AS OF '...'` |
| Tag 查询 | `SELECT * FROM t VERSION AS OF 'tag-name'` | `SELECT * FROM t.tag_xxx` |
| System Tables | `$snapshots`、`$schemas`、`$manifests`、`$files`、`$options`、`$branches`、`$tags`、`$consumers` 等 | `history`、`snapshots`、`files`、`manifests`、`partitions` 等 |
| MERGE INTO | 通过 Spark SQL Extensions 支持 | 原生支持 |
| Branch 操作 | 通过 Procedure | 通过 SQL (ALTER TABLE ... CREATE BRANCH) |

---

## 9. 索引能力对比

### 9.1 Paimon 的索引体系

从源码分析，Paimon 提供了极其丰富的索引能力：

**4 种文件级索引** (`FileIndexer` 实现)：

| 索引类型 | 实现类 | 原理 | 适用场景 |
|----------|--------|------|----------|
| Bloom Filter | `BloomFilterFileIndex` | 概率型数据结构，判断元素是否可能存在 | 等值查询过滤 |
| Bitmap | `BitmapFileIndex` | 每个值一个 Bitmap，精确匹配 | 低基数列的等值/IN 查询 |
| BSI (Bit-Sliced Index) | `BitSliceIndexBitmapFileIndex` | 位切片索引，支持范围查询 | 数值列的范围查询 |
| Range Bitmap | `RangeBitmapFileIndex` | 范围 Bitmap，支持区间过滤 | 数值列的区间过滤 |

**全局索引**：
- `HashIndexFile`：全局哈希索引，用于 Dynamic Bucket 模式下快速定位主键所在的 Bucket
- `DynamicBucketIndexMaintainer`：维护主键到 Bucket 的映射关系
- `GlobalIndexMeta`：全局索引元数据

**Lookup 索引**：
- LSM-Tree 的 `LookupLevels` 机制：缓存高层文件到本地，通过内存/磁盘 Hash 表实现 O(1) 点查
- 支持 Bloom Filter 加速 Lookup（`LOOKUP_CACHE_BLOOM_FILTER_ENABLED`）
- 支持远程文件直接 Lookup（`LOOKUP_REMOTE_FILE_ENABLED`）

**文件索引的嵌入方式**：Paimon 的文件索引直接嵌入在数据文件中（而非独立文件），这减少了 I/O 次数。通过 `FileIndexFormat` 定义了索引在文件中的序列化格式。

### 9.2 Iceberg 的索引能力

| 索引类型 | 实现 | 原理 |
|----------|------|------|
| Partition Summary | manifest 中的 `partitions` 字段 | 记录每个 manifest 文件覆盖的分区范围 |
| Column Statistics | manifest entry 中的 `column_sizes`、`value_counts`、`null_value_counts`、`lower_bounds`、`upper_bounds` | 每个数据文件的列级统计信息（min/max/null_count） |
| Bloom Filter | Puffin 文件格式中可存储 | V3 格式开始支持，需要额外配置 |

### 9.3 对比分析

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 索引类型数量 | 4 种文件索引 + 全局索引 + Lookup 索引 | 列统计 + Partition Summary + Bloom Filter |
| 等值查询优化 | Bloom Filter + Bitmap + Hash Lookup | 列统计 (min/max) + Bloom Filter (V3) |
| 范围查询优化 | BSI + Range Bitmap | 列统计 (min/max) |
| 点查能力 | O(1) Lookup（内存/磁盘 Hash 表） | 无专门点查优化 |
| 索引存储位置 | 嵌入数据文件 | 列统计在 manifest 中；Bloom Filter 在 Puffin 文件中 |
| 全局索引 | 支持（Hash Index 用于 Dynamic Bucket） | 不支持 |
| 索引自动维护 | 写入时自动构建 | 列统计自动构建；Bloom Filter 需配置 |

**为什么 Paimon 的索引更丰富**：Paimon 的主键表需要高效的点查来支持 Lookup 合并、Dynamic Bucket 路由等操作。这些都是 LSM-Tree 存储模型内在要求的能力。Iceberg 作为分析型表格式，主要依赖列统计做粗粒度过滤，对细粒度索引的需求不那么强烈。

---

## 10. 小文件治理对比

### 10.1 Paimon 的内置自动治理

**主键表**：
- 写入时 Level-0 文件由内存 WriteBuffer flush 产生，大小受 `write-buffer-size` 控制
- Compaction 自动合并小文件为更大的文件，`target-file-size` 控制目标文件大小
- `numSortedRunStopTrigger` 限制未合并文件的数量上限（反压）

**追加表**：
- `BucketedAppendFileStoreWrite` 和 `AppendFileStoreWrite` 内置 Compaction
- 无桶模式下自动合并小文件（`BUCKET_UNAWARE` 模式）
- 支持 `auto-compaction` 开关控制

**Manifest 治理**：
- `MANIFEST_FULL_COMPACTION_FILE_SIZE` 控制 Manifest 文件合并阈值
- 提交时自动触发 Manifest Compaction（在 `FileStoreCommitImpl` 中）

**Snapshot 过期**：
- 内置过期机制，自动清理过期 Snapshot 及其关联的数据文件和 Manifest 文件
- `ExpireSnapshotsProcedure` / `ExpirePartitionsProcedure` 提供手动触发入口

### 10.2 Iceberg 的外部工具集

| 操作 | Iceberg 实现方式 | 触发方式 |
|------|-----------------|----------|
| 合并小数据文件 | `RewriteDataFiles` Action | 手动触发（Spark/Flink Action） |
| 合并小 Manifest | `RewriteManifests` Action | 手动触发 |
| 清理过期 Snapshot | `ExpireSnapshots` Action | 手动触发 |
| 清理孤儿文件 | `RemoveOrphanFiles` Action | 手动触发 |
| 清理 Delete File | `RemoveDanglingDeleteFiles` + `RewritePositionDeleteFiles` | 手动触发 |

### 10.3 自动化程度对比

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 小文件合并 | 自动（写入时异步 Compaction） | 手动（需要调度 Action） |
| Manifest 合并 | 自动（提交时检查） | 手动（RewriteManifests） |
| 过期数据清理 | 半自动（可配置自动过期） | 手动（ExpireSnapshots） |
| 孤儿文件清理 | 提供 Procedure | 提供 Action |
| 运维成本 | 低——配好参数即可 | 高——需要搭建调度系统 |

**Iceberg 小文件问题在生产中的严重性**：由于 Iceberg 不内置 Compaction，在高频写入场景下（如每分钟一次 Commit），每次 Commit 产生一组小数据文件和 Delete File。一天下来可能产生数千个小文件，严重影响查询性能。用户必须设置定时任务（如每小时运行 `RewriteDataFiles`）来治理。这在 Paimon 中通过内置 Compaction 自动解决。

---

## 11. 版本管理对比

### 11.1 Paimon 的 Tag / Branch / Consumer

**Tag**（`Tag.java`，继承自 `Snapshot`）：
- Tag 本质是一个命名的 Snapshot，包含 Snapshot 的所有信息加上额外的 Tag 元数据
- 支持自动创建（`TagAutoCreation`）：按时间周期（小时/天/自定义）自动打 Tag
- 支持过期策略（`TagTimeExpire`）：按时间自动清理过期 Tag
- 被 Tag 引用的 Snapshot 及其数据文件不会被过期清理（锚点机制）
- 支持 Tag Preview（`TagPreview`）：预览 Tag 关联的数据

**Branch**（`BranchManager`）：
- 从某个 Tag 或 Snapshot 创建分支
- 两种实现：`FileSystemBranchManager`（基于文件系统）和 `CatalogBranchManager`（基于 Catalog）
- 分支拥有独立的 Snapshot 序列，与主干互不影响
- 支持 Fast-Forward：将分支合并回主干

**Consumer**（`ConsumerManager`）：
- 记录每个流式消费者的消费位点（`nextSnapshot`）
- 被 Consumer 引用的 Snapshot 不会被过期清理
- 支持重置和清理 Consumer

### 11.2 Iceberg 的 Ref / Branch / Tag

**SnapshotRef**：Iceberg 通过 `SnapshotRef` 统一管理 Tag 和 Branch：
- `TAG`：不可变引用，指向特定 Snapshot
- `BRANCH`：可变引用，可以独立演进

**ManageSnapshots API**：
- `createBranch(name, snapshotId)`：创建分支
- `createTag(name, snapshotId)`：创建 Tag
- `rollbackTo(snapshotId)`：回滚到指定 Snapshot
- `cherrypick(snapshotId)`：Cherry-pick 特定 Snapshot
- `setMaxSnapshotAgeMs` / `setMinSnapshotsToKeep`：配置过期策略

### 11.3 对比分析

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| Tag 自动创建 | 支持（按时间周期自动创建，`TagAutoCreation`） | 不支持（需手动或外部触发） |
| Tag 自动过期 | 支持（`TagTimeExpire`） | 支持（通过 `maxRefAgeMs`） |
| Branch 独立性 | 完全独立的 Snapshot 序列 | 共享 Table Metadata |
| Consumer 跟踪 | 内置 ConsumerManager | 无内置机制 |
| Snapshot 保留 | Tag/Consumer 引用的 Snapshot 自动保留 | 需要配置 min-snapshots-to-keep |
| Cherry-pick | 不支持 | 支持 |
| Branch Fast-Forward | 支持 | 支持 |
| 回滚 | 支持回滚到 Snapshot/Timestamp/Watermark | 支持回滚到 Snapshot/Timestamp |

**Paimon 的独特能力**：
1. **自动 Tag 创建**：生产环境中非常实用，可以自动按小时/天创建检查点
2. **Consumer 管理**：流式消费的核心需求，确保消费者的进度不丢失
3. **Watermark 回滚**：从 Snapshot 的 `watermark` 字段回滚，这是流式场景独有的需求

**Iceberg 的独特能力**：
1. **Cherry-pick**：可以选择性地应用某个 Snapshot 的变更
2. **成熟的 Branch 语义**：在 metadata.json 中统一管理 refs，语义清晰

---

## 12. 生态系统对比

### 12.1 Catalog 支持

| Catalog 类型 | Paimon | Iceberg |
|-------------|--------|---------|
| Filesystem | FileSystemCatalog（默认） | HadoopCatalog |
| Hive Metastore | 支持 | HiveCatalog |
| JDBC | 支持 | JdbcCatalog |
| REST | RESTCatalog | RESTCatalog |
| AWS Glue | 通过 REST | GlueCatalog |
| Nessie | 不支持 | NessieCatalog |
| Polaris | 不支持 | 通过 REST |
| Unity Catalog | 不支持 | 通过 REST |

**关键差异**：Iceberg 的 Catalog 生态远比 Paimon 丰富。特别是 REST Catalog 标准使得 Iceberg 可以接入任何实现了 REST 协议的 Catalog 服务（如 Polaris、Unity Catalog、Gravitino）。Paimon 虽然也支持 REST Catalog，但生态接入度不如 Iceberg。

### 12.2 计算引擎支持

| 引擎 | Paimon | Iceberg |
|------|--------|---------|
| Flink | 原生深度集成（1.16-2.2） | 社区集成 |
| Spark | DataSource V2（3.2-4.0） | 原生深度集成（3.3-3.5） |
| Hive | SerDe 连接器（2.1-3.1） | SerDe 连接器 |
| Trino/Presto | 连接器 | 原生连接器 |
| Doris | 连接器 | 原生连接器 |
| StarRocks | 连接器 | 原生连接器 |
| Impala | 不支持 | 原生连接器 |
| Dremio | 不支持 | 原生连接器 |
| DuckDB | 不支持 | 原生连接器 |
| Snowflake | 不支持 | 原生支持（External Table） |

**关键差异**：Iceberg 的引擎覆盖面远大于 Paimon。几乎所有主流分析引擎都内置了 Iceberg 支持，而 Paimon 的主要支持集中在 Flink 和 Spark。

### 12.3 云厂商支持

| 云厂商 | Paimon | Iceberg |
|--------|--------|---------|
| AWS | S3 FileSystem 支持 | 原生支持（EMR、Glue、Athena） |
| Azure | Azure Blob 支持 | 原生支持（Synapse、Databricks） |
| GCP | GCS 支持 | 原生支持（BigQuery、Dataproc） |
| 阿里云 | 深度支持（OSS、EMR、Hologres） | 基本支持 |
| 华为云 | OBS 支持 | 基本支持 |
| 腾讯云 | COS 支持 | 基本支持 |

**关键差异**：在国际公有云（AWS/Azure/GCP）上 Iceberg 的集成度更高，有大量原生支持。在国内云厂商（特别是阿里云）上 Paimon 的支持更深入。

### 12.4 Paimon 的 Iceberg 兼容模块

值得注意的是，Paimon 提供了 `paimon-iceberg` 模块（从源码可见 `IcebergRestMetadataCommitter`），支持将 Paimon 表的元数据以 Iceberg REST 协议对外暴露。这意味着用户可以用 Paimon 写入，用 Iceberg 读取，在一定程度上结合了两者的优势。

---

## 13. 选型建议

### 13.1 选 Paimon 的场景

| 场景 | 原因 |
|------|------|
| **实时数据入湖** | LSM-Tree 的高写入吞吐和低延迟是核心优势 |
| **CDC 全量同步 + 增量同步** | 内置的 CDC 同步管道和 Changelog 生成能力是独有的 |
| **高频 Upsert（秒级/分钟级）** | MergeFunction 体系远优于 Delete File 机制 |
| **实时聚合/部分更新** | Aggregation/PartialUpdate 合并引擎是独有的 |
| **Flink 为主力引擎** | 与 Flink 的深度集成是其他格式无法比拟的 |
| **流式消费下游** | 原生 Changelog + Consumer 管理 |
| **运维精力有限** | 内置 Compaction 和自动 Tag 减轻运维负担 |

### 13.2 选 Iceberg 的场景

| 场景 | 原因 |
|------|------|
| **多引擎混合查询** | 引擎生态覆盖最广 |
| **以批处理为主的数据仓库** | 不可变文件模型利于批查询优化 |
| **国际公有云环境** | AWS/Azure/GCP 的原生集成度最高 |
| **Schema/Partition Evolution** | 设计成熟，语义清晰 |
| **数据湖标准化** | REST Catalog 标准被广泛采用 |
| **低频更新、高频查询** | 无 Compaction 开销，查询路径简洁 |
| **已有 Iceberg 生态投资** | 迁移成本低 |

### 13.3 两者结合的场景

```
           实时写入                    批量分析
  CDC源 ──→ Flink ──→ Paimon ──→ (paimon-iceberg) ──→ Iceberg 兼容读取
                        │                                    │
                        │                                    ├── Trino 查询
                        │                                    ├── Snowflake 查询
                        ├── Flink 流式消费                    └── Dremio 查询
                        └── Spark 交互式查询
```

利用 Paimon 的 `paimon-iceberg` 兼容层，可以实现：
- 写入侧使用 Paimon 的高性能流式写入和内置 Compaction
- 读取侧通过 Iceberg REST 协议对接广泛的分析引擎
- 同一份数据同时支持流式消费（通过 Paimon）和批量分析（通过 Iceberg 兼容读取）

### 13.4 决策矩阵

| 决策因素 | 权重 | Paimon 评分 | Iceberg 评分 |
|----------|------|-------------|-------------|
| 流式写入性能 | 高 | 5 | 2 |
| 批量查询性能 | 高 | 4 | 5 |
| 更新/删除性能 | 高 | 5 | 3 |
| 引擎生态广度 | 中 | 3 | 5 |
| 云厂商支持 | 中 | 3 (国内5) | 5 (国际) |
| 运维复杂度 | 中 | 4 (低运维) | 2 (高运维) |
| 流式消费能力 | 中 | 5 | 2 |
| Schema Evolution | 低 | 4 | 5 |
| 社区成熟度 | 低 | 3 | 5 |
| 索引能力 | 低 | 5 | 3 |

> 评分说明：5 = 优秀，4 = 良好，3 = 中等，2 = 较弱，1 = 不支持

### 13.5 总结

**Paimon 和 Iceberg 不是简单的竞争关系，而是面向不同核心场景的互补方案。**

Paimon 的核心价值在于**将流式处理的理念深植于存储格式中**。LSM-Tree、MergeFunction、原生 Changelog、内置 Compaction 等特性使它成为实时湖仓场景的最佳选择。它的代价是更高的系统复杂度和相对有限的引擎生态。

Iceberg 的核心价值在于**为数据湖提供了通用的、引擎无关的表格式标准**。不可变文件、丰富的 Catalog 生态、广泛的引擎支持使它成为批处理分析和多引擎环境的首选。它的代价是更新性能不足和流式能力有限。

在实际生产中，许多企业正在探索两者结合的架构——用 Paimon 做实时入湖和流式消费，通过 Iceberg 兼容层对接下游分析引擎，兼取两者之长。
