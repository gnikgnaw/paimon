# Apache Paimon Flink 集成源码深度分析

> **版本**：1.5-SNAPSHOT　**源码模块**：`paimon-flink`（`paimon-flink-common` / `paimon-flink-cdc` / `paimon-flink-action` 及各版本适配层）　**核对日期**：2026-06

**一句话定位**：Paimon-Flink 集成把 Paimon 的「LSM 存储引擎 + ACID 快照」接到 Flink 的「批流统一 Source API + Checkpoint 两阶段提交」上，让一张表既能被 Flink 流式 upsert、又能被批量回填、还能做维表 Lookup Join 和 CDC 入湖——核心价值是用 Flink 的 Checkpoint 对齐换来端到端 Exactly-Once。

读完本文你应能回答：
1. 一条记录从 Flink 算子 `write()` 到生成 Paimon Snapshot，中间经过哪些算子、哪个 Checkpoint 回调触发了什么；
2. 为什么 Committer 并行度必须强制为 1、为什么不支持 Unaligned Checkpoint、为什么必须 EXACTLY_ONCE；
3. 流式 Source 的增量扫描如何只读新 Snapshot、Enumerator 用哪两把闸做反压、为什么有主键表必须有序读；
4. Split 为什么要按 bucket 分配给固定 task（Bucket 感知），配错会导致什么；
5. `BucketMode` 的五种值各自走哪条 Sink 链路、`write-only` + Dedicated Compaction 解决了什么；
6. Lookup Join 的 Full / Partial / Remote 三种缓存各自的边界，`refreshFullThreshold` 和黑名单解决什么；
7. CDC Sync 的表级/库级入口怎么走、Schema 自动演进靠什么机制热替换 Writer 而不重启；
8. `commitUser` 为什么必须跨重启一致、配成随机值会造成什么后果。

> 阅读约定：本文每个机制按"① 要解决什么问题 → ② 设计原理与取舍 → ③ 关键源码（精选片段 + `路径:行号`）→ ④ 风险/陷阱/边界 → ⑤ 收益与代价"组织。源码行号以本次核对（commit 基于 master 2026-06）为准；与旧稿不符处用 `（已修正）` 标注。
>
> **交叉引用**：本篇是 Paimon-Flink 集成的**总览主讲**——讲整体拓扑、连接器架构、两阶段提交与 Checkpoint 对齐的语义。**Flink 写入的算子级内部细节**（StoreSinkWrite 内部如何驱动 `TableWriteImpl`、Committer 的冲突重试、内存池抢占）由 **15 号文档**主讲，本篇点到为止；**CDC 数据集成的解析/类型映射/Schema 演进细节**由 **14 号文档**主讲，本篇只讲 Flink CDC 的 Action 入口与拓扑。LSM/Compaction 原理见 **01 号**，Lookup 点查底层见 **01 号 §6**。

---

## 目录

- [1. 快速理解（核心问题 / 概念速查 / 高频陷阱）](#1-快速理解核心问题--概念速查--高频陷阱)
  - [1.1 核心问题：Flink 集成到底要解决什么](#11-核心问题flink-集成到底要解决什么)
  - [1.2 核心概念速查表](#12-核心概念速查表)
  - [1.3 高频生产陷阱](#13-高频生产陷阱)
- [2. 总体架构：拓扑与模块分层](#2-总体架构拓扑与模块分层)
  - [2.1 端到端拓扑：从 write 到 Snapshot](#21-端到端拓扑从-write-到-snapshot)
  - [2.2 模块分层与版本适配](#22-模块分层与版本适配)
- [3. Flink Source 连接器体系](#3-flink-source-连接器体系)
  - [3.1 继承层次与 Static/Continuous 分叉](#31-继承层次与-staticcontinuous-分叉)
  - [3.2 StaticFileStoreSource 批量读取](#32-staticfilestoresource-批量读取)
  - [3.3 ContinuousFileStoreSource 流式读取与有序性](#33-continuousfilestoresource-流式读取与有序性)
  - [3.4 ContinuousFileSplitEnumerator：增量扫描与反压](#34-continuousfilesplitenumerator增量扫描与反压)
  - [3.5 Bucket 感知的 Split 分配](#35-bucket-感知的-split-分配)
  - [3.6 FlinkSourceBuilder 构建决策树](#36-flinksourcebuilder-构建决策树)
- [4. Flink Sink 连接器体系](#4-flink-sink-连接器体系)
  - [4.1 算子拓扑与职责](#41-算子拓扑与职责)
  - [4.2 FlinkSink 基类：doWrite / doCommit 两步走](#42-flinksink-基类dowrite--docommit-两步走)
  - [4.3 FlinkSinkBuilder 按 BucketMode 分发](#43-flinksinkbuilder-按-bucketmode-分发)
  - [4.4 Writer 算子继承链与提交触发](#44-writer-算子继承链与提交触发)
  - [4.5 StoreSinkWrite 三种写入策略](#45-storesinkwrite-三种写入策略)
- [5. Checkpoint 两阶段提交与 Exactly-Once](#5-checkpoint-两阶段提交与-exactly-once)
  - [5.1 两阶段提交的完整时序](#51-两阶段提交的完整时序)
  - [5.2 CommitterOperator 状态管理](#52-committeroperator-状态管理)
  - [5.3 StoreCommitter 实际提交](#53-storecommitter-实际提交)
  - [5.4 三条硬约束及其根因](#54-三条硬约束及其根因)
  - [5.5 状态恢复：commitUser 与 Writer State](#55-状态恢复commituser-与-writer-state)
- [6. CDC 同步入口（Flink 侧）](#6-cdc-同步入口flink-侧)
  - [6.1 Action 体系与构建流程](#61-action-体系与构建流程)
  - [6.2 表级与库级同步](#62-表级与库级同步)
  - [6.3 Schema 自动演进与拓扑](#63-schema-自动演进与拓扑)
- [7. Flink SQL 集成（Catalog / Factory / Procedure）](#7-flink-sql-集成catalog--factory--procedure)
- [8. Dedicated Compaction](#8-dedicated-compaction)
- [9. Lookup Join 机制](#9-lookup-join-机制)
  - [9.1 FileStoreLookupFunction 与缓存选择](#91-filestorelookupfunction-与缓存选择)
  - [9.2 三种缓存实现的边界](#92-三种缓存实现的边界)
  - [9.3 缓存刷新与 Bucket 感知](#93-缓存刷新与-bucket-感知)
- [10. 与 Iceberg Flink Sink 对比](#10-与-iceberg-flink-sink-对比)
- [11. 设计决策总结](#11-设计决策总结)

---

## 1. 快速理解（核心问题 / 概念速查 / 高频陷阱）

### 1.1 核心问题：Flink 集成到底要解决什么

**① 要解决什么问题**

Paimon Core 提供的是"单机视角"的存储能力：`TableWrite.write()` 写本地 LSM、`TableCommit.commit()` 原子生成 Snapshot、`TableScan` + `TableRead` 读快照。但生产中数据是分布式、持续流入、随时可能 failover 的。Flink 集成层要把这套单机 API 嫁接到 Flink 的分布式运行时上，同时满足四个互相牵制的需求：

1. **批流统一**：同一张表，批模式读历史、流模式读增量，用户只学一套 API（FLIP-27 Source）。
2. **端到端 Exactly-Once**：多个并行 Writer 分布式写入，故障重启不重不漏，要求把 Paimon 的原子提交对齐到 Flink 的 Checkpoint 边界。
3. **有序性**：主键表的 CDC changelog（UPDATE_BEFORE/UPDATE_AFTER）必须按 Snapshot 顺序到达同一下游 task，否则数据错乱。
4. **资源弹性**：写入、Compaction、Lookup 各有不同资源画像，能拆分、能隔离。

**② 设计哲学一句话**：**Writer 负责"写"和"flush"，Committer 负责"提交"，二者由 Flink Checkpoint 的两个回调（`prepareSnapshotPreBarrier` 和 `notifyCheckpointComplete`）精确编排——把 Paimon 的两阶段提交"折叠"进 Flink 的 Checkpoint 生命周期。**

这条主线贯穿全文：Source 用 Enumerator/Reader 分离做增量扫描和反压（§3）；Sink 用 Writer/Committer 分离做两阶段提交（§4、§5）；Lookup 复用 Paimon 的 LSM 点查能力做维表关联（§6）；CDC 复用整套 Sink 链路再叠加 Schema 演进（§7）。

### 1.2 核心概念速查表

| 概念 | 一句话定义 | 关键源码 |
|------|-----------|---------|
| **FlinkSource** | 实现 Flink `Source` 接口的抽象基类，统一 `createReader`；派生 Static/Continuous | `FlinkSource.java:42` |
| **ContinuousFileSplitEnumerator** | 运行在 JobManager，周期扫新 Snapshot 生成 Split、做反压、按 bucket 分配 | `ContinuousFileSplitEnumerator.java:231`（scanNextSnapshot） |
| **FileStoreSourceReader** | 运行在 TaskManager，单线程复用读多个 Split，上报消费进度 | `FileStoreSourceReader.java` |
| **FlinkSink** | Sink 基类，`sinkFrom` = `doWrite`（写）+ `doCommit`（提交） | `FlinkSink.java:97` |
| **PrepareCommitOperator** | Writer 算子抽象基类，在 `prepareSnapshotPreBarrier` 触发 flush+提交 | `PrepareCommitOperator.java:93` |
| **StoreSinkWrite** | 封装 Paimon `TableWriteImpl` 的写入策略，三种实现 | `StoreSinkWrite.java:49`、`createWriteProvider:100` |
| **CommitterOperator** | 单并行算子，按 checkpointId 聚合 Committable，Checkpoint 完成后提交 | `CommitterOperator.java:190`（notifyCheckpointComplete） |
| **StoreCommitter** | `Committer` 接口实现，最终调用 `TableCommitImpl.commitMultiple` | `StoreCommitter.java:99` |
| **Committable / ManifestCommittable** | Writer 产出的提交元数据 → 按 checkpoint 合并后的全局提交单元 | `StoreCommitter.java:79` |
| **commitUser** | 标识提交者的稳定 ID，跨重启不变，用于识别/清理未完成提交 | `CommitterOperator.java:131` |
| **BucketMode** | 数据物理分组的五种模式，决定 Sink 走哪条链路 | `FlinkSinkBuilder.java:236` |
| **FileStoreLookupFunction** | Lookup Join 的 Flink 函数，按配置选 Full/Partial/Remote 缓存 | `FileStoreLookupFunction.java:167` |
| **SynchronizationActionBase** | CDC Sync Action 抽象基类，构建 source→parse→sink 链路 | `SynchronizationActionBase.java:131` |

### 1.3 高频生产陷阱

**陷阱 1：commitUser 配成随机值（如 `UUID.randomUUID()`）。** commitUser 是 Paimon 识别"哪些未提交文件属于本作业"的依据。每次启动用新值，重启后新 Committer 认不出旧的未完成提交，旧文件成为孤儿、可能被清理，且无法续提。必须用稳定值（默认由 Paimon 生成并存入 Flink state，从 Checkpoint 恢复，见 `CommitterOperator.java:131`）。**切勿用 Job ID**——从 Savepoint 恢复时 Job ID 会变。

**陷阱 2：以为 Checkpoint 成功 = 数据已可查。** Checkpoint 成功只意味着 Writer 已 flush 文件到磁盘，Snapshot 尚未生成；真正提交发生在其后的 `notifyCheckpointComplete`（§5.2）。两者之间窗口内查表看不到这批数据。排障时"刚写完查不到"先确认是否还没到提交回调。

**陷阱 3：开启 Unaligned Checkpoint 或非 EXACTLY_ONCE。** Paimon Sink 在 `doCommit` 启动时硬校验（`FlinkSink.java:264` assertStreamingConfiguration），不满足直接抛异常。根因：Unaligned 允许 Barrier 越过数据，会把未 flush 的数据混进提交，破坏两阶段提交一致性（§5.4）。

**陷阱 4：有主键表误开无序读取。** `bucket-append-ordered=false` 只对 append-only 表合法。有主键表强制有序（`FlinkSourceBuilder.unordered():101` 判定），否则下游可能先收到 UPDATE_AFTER 再收 UPDATE_BEFORE，主键状态错乱（§3.3）。

**陷阱 5：流式 Source 扫描间隔远小于 Checkpoint 间隔。** `continuous.discovery-interval`（默认 10s）若设成 1s，会每秒生成 Split 但要等 Checkpoint 才提交消费位点，导致在途 Snapshot 积压。经验：扫描间隔 ≈ Checkpoint 间隔的同量级或更大。Enumerator 有 `scan.max-splits-per-task` 和 `scan.max-snapshot-count` 两把反压闸（§3.4），但配置不当仍会积压。

**陷阱 6：bucket 数与并行度严重失配。** 每个 bucket 是一棵独立 LSM，Writer/Reader/Compactor 都按 bucket 路由。bucket 远多于并行度 → 单 task 扛多桶、内存压力大；bucket 远少于并行度 → 大量 task 空闲。FlinkSinkBuilder 对非分区固定桶表会自动把 Writer 并行度压到 bucket 数（`buildForFixedBucket:292`，**已修正：旧稿标 L272-287**）。

**陷阱 7：写入作业未配 `write-only=true` 却又跑 Dedicated Compaction。** 此时写入作业内的 Writer 仍会触发 Compaction，与独立 Compaction 作业重复压缩、互相竞争甚至冲突。剥离 Compaction 时写入侧必须 `write-only=true`（§8）。

**陷阱 8：Lookup Join 用 join key ≠ primary key 又不建索引。** 走 `SecondaryIndexLookupTable`，无文件索引时退化为全表扫描；大维表会拖垮查询。应在 join 列上配 `file-index.bloom-filter.columns`（§6.2）。另：大维表用 `lookup.cache-mode=full` 可能 OOM，应让 join key=primary key 走 Partial 缓存或显式 none。

---

## 2. 总体架构：拓扑与模块分层

### 2.1 端到端拓扑：从 write 到 Snapshot

理解 Flink 集成最快的路径是跟一条记录走完全程。以主键表流式 upsert 为例：

```
Source 侧（读）                              Sink 侧（写）
─────────────                              ─────────────
Snapshot                                   DataStream<RowData>
  │ 周期扫描(discovery-interval)              │ [可选] LocalMergeOperator 本地预合并同 key
ContinuousFileSplitEnumerator (JM,单点)      │
  │ 增量 plan() → Split，按 bucket 分配        ▼
  │ 反压：splitMaxNum / maxSnapshotCount     RowDataStoreWriteOperator (TM,多并行)
  ▼                                          │ processElement → write.write(row)  写本地 LSM
FileStoreSourceReader (TM,单线程复用)         │ prepareSnapshotPreBarrier → flush → CommitMessage
  │ 读 Split，完成后上报 consume progress      │
  ▼                                          │ DataStream<Committable>
RowData → 下游算子                           │ [可选] Changelog Compact 算子链
                                            ▼
                                           CommitterOperator (并行度=1)
                                            │ snapshotState: 按 checkpointId 聚合 → state
                                            │ notifyCheckpointComplete: commitMultiple → Snapshot
                                            ▼
                                           DiscardingSink (终结拓扑)
```

关键时序锚点（贯穿 §4、§5）：

1. 数据流经 Writer，`write()` 进 Paimon Core 的本地 LSM WriteBuffer（写路径内部见 **01 号 §4**）；
2. Checkpoint Barrier 到达前，Flink 回调 `prepareSnapshotPreBarrier(cpId)` → Writer flush 内存、产出 `CommitMessage`（Phase 1，prepare）；
3. Committer 在 `snapshotState` 把本次 Committable 按 checkpointId 聚合存入 Flink state（容错点）；
4. Checkpoint **整体成功**后，Flink 回调 `notifyCheckpointComplete(cpId)` → Committer 提交 → 生成 Snapshot（Phase 2，commit）。

这套编排的精妙处：Phase 1 与 Flink 的"Barrier 到达即数据已持久化"语义对齐，Phase 2 与"Checkpoint 成功即状态已落盘"语义对齐——Paimon 不需要自己造分布式锁或事务协调器，**整个两阶段提交"借用"了 Flink 的 Checkpoint 协调能力**。

### 2.2 模块分层与版本适配

**① 要解决什么问题**：Flink 从 1.16 到 2.2 跨多个大版本，API、Java（8 vs 11）、Scala（2.12 vs 2.13）都有差异。若每个版本一套完整代码，8 个版本意味着一个 bug 改 8 遍。

**② 设计取舍**：核心逻辑只写一次放在 `paimon-flink-common`，版本差异隔离在薄适配层。`paimon-flink-common` 依赖变量 `${paimon-flinkx-common}`，由 Maven Profile 解析为 `paimon-flink1-common`（编译于 Flink 1.20.1，1.x 的 API 超集）或 `paimon-flink2-common`（编译于 2.x）。低版本（如 1.16）通过适配层桥接缺失 API。

模块结构（`paimon-flink/pom.xml`）：

```
paimon-flink/
  ├ paimon-flink1-common/   Flink 1.x 公共代码（编译于 1.20.1）
  ├ paimon-flink2-common/   Flink 2.x 公共代码（编译于 2.2.0，仅 flink2 Profile）
  ├ paimon-flink-common/    版本无关核心：Source/Sink/Catalog/Lookup/Procedure
  ├ paimon-flink-cdc/       CDC 同步（依赖 flink-common）
  ├ paimon-flink-action/    命令行 Action（依赖 flink-common）
  └ paimon-flink-1.16 … 2.2 各版本适配 + Bundle 打包
```

| Profile | Flink 版本 | 公共模块 | Java | Scala | 默认激活 |
|---|---|---|---|---|---|
| `flink1` | 1.16–1.20 | `paimon-flink1-common` | 8+ | 2.12 | 是 |
| `flink2` | 2.0–2.2 | `paimon-flink2-common` | 11+ | 2.13 | 否 |

**④ 风险/陷阱**：
- 部署时 Bundle 必须与集群 Flink 版本匹配，否则 `NoSuchMethodError`/`ClassNotFoundException`；
- `flink2` 因需 Java 11、Scala 2.13，不默认激活，与 Paimon 整体 JDK 8 基线不同；
- 本地编译别全量 `mvn install`（会编所有版本模块），用 `-pl paimon-flink/paimon-flink-1.18 -am` 只编需要的。

**⑤ 收益与代价**：代码复用极高、新增 Flink 版本只需薄适配层；代价是公共层必须谨慎只用 1.x/2.x 公共子集 API，跨版本行为差异要靠适配层兜住。

---

## 3. Flink Source 连接器体系

**① 要解决什么问题**：用同一套 API 既批量读历史又流式读增量；流式时高效发现新 Snapshot 而非全表重扫；下游慢时不让 Split/Snapshot 无限积压撑爆 JM；主键表保证同 key 的 CDC 事件有序到达同一 task。

**② 设计原理**：基于 Flink FLIP-27 新 Source API，用 **Enumerator（JM 单点，扫描+分配）/ Reader（TM 多并行，读取）分离**架构。批流通过 `Boundedness` 区分但复用 `createReader`。流式靠 `StreamTableScan` 维护 `nextSnapshotId` 做增量 `plan()`，每次只扫新 Snapshot。

### 3.1 继承层次与 Static/Continuous 分叉

```
Source<RowData, FileStoreSourceSplit, PendingSplitsCheckpoint>   (Flink API)
  └ FlinkSource (abstract)                         createReader + 序列化器
      ├ StaticFileStoreSource                      Boundedness.BOUNDED，一次性全量 Split
      ├ ContinuousFileStoreSource                  CONTINUOUS_UNBOUNDED，周期增量 Split
      └ AlignedContinuousFileStoreSource           对齐 Checkpoint 的流式读取
```

`FlinkSource`（`FlinkSource.java:42`）只做两件事：`createReader`（L64-83，组装 `IOManager` + `FlinkMetricRegistry` + `readBuilder.newRead()` → `FileStoreSourceReader`）和定义 Split/Checkpoint 序列化器。它持有可序列化的 `ReadBuilder`（封装 filter/projection/limit），因此能安全分发到 TaskManager。

**为什么 Static 与 Continuous 分开**：两者的 SplitEnumerator 行为完全不同——批一次扫完即结束，流要无限周期扫描并维护增量位点，强行合并会让分支逻辑爆炸。

### 3.2 StaticFileStoreSource 批量读取

`StaticFileStoreSource`（`StaticFileStoreSource.java`）固定 `getBoundedness()=BOUNDED`（L76）。它在 `restoreEnumerator` 时通过 `getSplits()`（L92）执行一次性 `scan.plan()` 拿到当前 Snapshot 的全部 `DataSplit`，再交给 `createSplitAssigner`（L103）按模式分配：

| 模式 (`scan.split-assigner-mode`) | 实现类 | 行为 | 适用场景 |
|------|--------|------|---------|
| `FAIR`（默认） | `PreAssignSplitAssigner` | 预先按 task 均匀分配 | 数据分布均匀，防倾斜 |
| `PREEMPTIVE` | `FIFOSplitAssigner` | 先到先得（FIFO） | 数据倾斜时让快 task 多处理 |

**为什么默认 FAIR**：预分配让各 TaskManager 工作量大致均等，避免部分 task 空闲、部分过载。确认有倾斜（如某 bucket 1GB、其余 10MB）再切 PREEMPTIVE。

### 3.3 ContinuousFileStoreSource 流式读取与有序性

`ContinuousFileStoreSource`（`ContinuousFileStoreSource.java:42`）的有界性是**动态**的：配了 `scan.bounded.watermark` 则 BOUNDED，否则 CONTINUOUS_UNBOUNDED（`getBoundedness():74`）——这让"流式读到某 watermark 停止"成为可能。

`restoreEnumerator()`（L79-96，**已修正：旧稿标 L80-96**）的恢复逻辑：从 checkpoint 取出 `nextSnapshotId` 与残留 splits → `readBuilder.newStreamScan()` 建流式扫描器 → `scan.restore(nextSnapshotId)` 恢复扫描位点 → 构建 `ContinuousFileSplitEnumerator`。源并行度上界也在此定（`buildEnumerator:108`）：固定桶取 bucket 数，动态桶取 `MAX_PARALLELISM_OF_SOURCE=32768`。

**有序 vs 无序读取**：`unordered` 由 `FlinkSourceBuilder.unordered()`（L101，**已修正：旧稿标 L101-118 行号范围**）判定——只有 append-only 表才可能无序，有主键表恒为有序。无序时用 `FIFOSplitAssigner`，有序时用 `PreAssignSplitAssigner`（见 `ContinuousFileSplitEnumerator.createSplitAssigner:367`）。

**为什么有主键表必须有序**：Paimon 的 changelog 语义依赖 Snapshot 提交顺序。若乱序，下游可能先收到某 key 的 UPDATE_AFTER 再收 UPDATE_BEFORE，主键状态机错乱、聚合结果错误。这是一条不可违反的硬约束（CDC 一致性见 **14 号**）。
### 3.4 ContinuousFileSplitEnumerator：增量扫描与反压

`ContinuousFileSplitEnumerator`（`ContinuousFileSplitEnumerator.java`）是流式读取的中枢，跑在 JobManager 上，**单点协调**所有 Reader。核心状态：`scan`（StreamTableScan）、`splitAssigner`、`readersAwaitingSplit`（等 Split 的 Reader 集合）、`consumerProgressCalculator`（消费进度）、`nextSnapshotId`、`handledSnapshotCount`。

工作循环：`start()` 注册周期任务（每 `discovery-interval`）→ 在 worker 线程调 `scanNextSnapshot()` 增量扫描 → 在 coordinator 线程 `processDiscoveredSplits()` 把新 Split 喂给 assigner → Reader 发 `handleSplitRequest` 时 `assignSplits()` 分配。

**两把反压闸**（`scanNextSnapshot():231-252`，行号准确）：

```java
protected synchronized Optional<PlanWithNextSnapshotId> scanNextSnapshot() {
    if (splitAssigner.numberOfRemainingSplits() >= splitMaxNum) {  // 闸1：积压 Split 超限
        return Optional.empty();
    }
    if (maxSnapshotCount > 0 && handledSnapshotCount >= maxSnapshotCount) {  // 闸2：在途快照超限
        return Optional.empty();
    }
    TableScan.Plan plan = scan.plan();          // 真正的增量扫描，只读新 Snapshot
    Long nextSnapshotId = scan.checkpoint();
    ...
}
```

- 闸 1 `splitMaxNum = currentParallelism() * scan.max-splits-per-task`（默认每 task 10），控制总积压量；
- 闸 2 `scan.max-snapshot-count` 控制一次未提交期间最多消化几个 Snapshot，避免扫描节奏失控；
- 两闸任一触发，本周期不扫描——**这就是 Source 的反压**：下游消费慢→Split 不被领走→积压触顶→停止扫描，向上游 Snapshot 自然回压。

**消费进度与 Checkpoint**：`snapshotState()`（L202-216，**已修正：旧稿标 L203-216**）记录各 Reader 进度；`notifyCheckpointComplete()`（L218-224）把最小进度回写给 `StreamTableScan`，落成 Consumer 位点——这是流式作业重启后从正确位置续读的依据（Consumer 机制见 **17 号**）。

读取端 `FileStoreSourceReader` 继承 Flink 的 `SingleThreadMultiplexSourceReaderBase`：单线程复用读多个 Split，省去每 Split 一线程的开销，且 Split 间顺序切换天然保证同 task 内有序。完成一个 Split 后通过 `ReaderConsumeProgressEvent` 上报最大 snapshotId 给 Enumerator。

### 3.5 Bucket 感知的 Split 分配

`assignSuggestedTask(DataSplit)`（`ContinuousFileSplitEnumerator.java:336-353`，**已修正：旧稿标 L328-365 且漏了 POSTPONE_BUCKET 分支**）决定一个 Split 该发给哪个 Reader subtask：

```java
protected int assignSuggestedTask(DataSplit split) {
    int parallelism = context.currentParallelism();
    int bucketId;
    if (split.bucket() == BucketMode.POSTPONE_BUCKET) {        // 延迟桶：用文件名里的 writeId 取模
        bucketId = PostponeBucketFileStoreWrite.getWriteId(split.dataFiles().get(0).fileName())
                        % parallelism;
    } else {
        bucketId = split.bucket();
    }
    if (shuffleBucketWithPartition) {
        return ChannelComputer.select(split.partition(), bucketId, parallelism);  // 分区+桶联合 hash
    } else {
        return ChannelComputer.select(bucketId, parallelism);                     // 仅按桶 hash
    }
}
```

**为什么按 bucket 路由**：保证同一 bucket 的 Split 始终去同一 Reader task。对有主键表是硬要求——同一 key 落同一 bucket，其 INSERT/UPDATE/DELETE 必须由同一 task 顺序处理。`read.shuffle-bucket-with-partition` 决定是否把分区也纳入 hash：分区多、单分区桶少时打开更均匀，否则同一 bucket 跨分区会挤到同一 task 造成倾斜。

### 3.6 FlinkSourceBuilder 构建决策树

`FlinkSourceBuilder.build()`（`FlinkSourceBuilder.java:308`，行号准确）按配置在五条路径间分流：

```
build():
  if (sourceBounded):                                # 批模式
      if SCAN_DEDICATED_SPLIT_GENERATION → buildDedicatedSplitGenSource(true)  # 独立 Split 生成
      else                               → buildStaticFileSource()             # 标准批量
  streamingReadingValidate(table)                    # 流模式前置校验
  if SOURCE_CHECKPOINT_ALIGN_ENABLED   → buildAlignedContinuousFileSource()    # 对齐 Checkpoint
  if CONSUMER_ID && EXACTLY_ONCE       → buildDedicatedSplitGenSource(false)   # Consumer 精确一次
  else                                 → buildContinuousFileSource()           # 标准流式
```

- `buildDedicatedSplitGenSource()`（L339）用 MonitorSource 模式把 Split 生成与读取拆成两个算子，服务于"批模式动态 Split 生成"和"Consumer 精确一次"两类更严格的需求；
- `AlignedContinuousFileStoreSource` 让 Source 的 Snapshot 推进与 Checkpoint 对齐，适合需要 Source/Sink 端到端一致快照的场景，但要求严格的 Checkpoint 配置。

---

## 4. Flink Sink 连接器体系

**① 要解决什么问题**：把多个并行 Writer 的分布式写入，原子地、Exactly-Once 地落成 Paimon Snapshot；同时让固定桶/动态桶/无桶/跨分区/延迟桶五种表结构各走最优的数据分发路径；写入缓冲与 Flink 内存管理集成，避免 OOM。

**② 设计原理**：**Writer/Committer 分离 + 两阶段提交**。Writer 多并行写本地 LSM，Committer 单并行做全局原子提交。提交时机由 Flink Checkpoint 的两个回调精确驱动（详见 §5）。Writer 写入算子内部如何驱动 `TableWriteImpl`、内存池如何在算子间抢占——属算子级细节，由 **15 号文档**主讲，本篇只讲拓扑装配与策略选择。

### 4.1 算子拓扑与职责

```
DataStream<T>
   │ [可选] LocalMergeOperator         有主键 + local-merge-enabled 时本地预合并同 key
   ▼
RowDataStoreWriteOperator (并行度=上游)  写本地 LSM；prepareSnapshotPreBarrier 时 flush → CommitMessage
   │ DataStream<Committable>
   │ [可选] ChangelogCompact 算子链      有主键 + PRECOMMIT_COMPACT 时压缩 changelog 小文件
   ▼
CommitterOperator (并行度=1)            聚合所有 Writer 的 Committable，notifyCheckpointComplete 时原子提交
   ▼
DiscardingSink                          终结拓扑
```

源码：`FlinkSink.java`。两个"为什么"值得记住：

- **Committer 并行度强制为 1**（`setParallelism(1).setMaxParallelism(1)`，`FlinkSink.java:228-229`，行号准确）：每个 Checkpoint 只能产生一个全局唯一的 Snapshot，多 Committer 并发提交会撞 Snapshot ID、需要分布式锁。单并行换取原子性与简单性，代价是 Committer 可能成为大并行度下的瓶颈（可单独配 Slot Sharing Group 加资源）。
- **末端 `DiscardingSink`**（`FlinkSink.java:243`）：`CommitterOperator` 是 `OneInputStreamOperator` 而非 Sink，但 Flink 要求 DAG 以 Sink 收尾，故补一个空 Sink；Committable 已在 Committer 内被消费。

### 4.2 FlinkSink 基类：doWrite / doCommit 两步走

`FlinkSink.sinkFrom()`（`FlinkSink.java:97`，行号准确）就是"先写后提交"：

```java
public DataStreamSink<?> sinkFrom(DataStream<T> input, String initialCommitUser) {
    DataStream<Committable> written = doWrite(input, initialCommitUser, null);  // 写，不产生 Snapshot
    return doCommit(written, initialCommitUser);                                // 提交，生成 Snapshot
}
```

- `doWrite()`（L125）：用 `StoreSinkWrite.createWriteProvider(...)` 选写入策略 → 子类 `createWriteOperatorFactory()` 造 Writer 算子 → 配 Managed Memory（`SINK_USE_MANAGED_MEMORY`）与 Slot Sharing Group → 有主键且 `PRECOMMIT_COMPACT` 时追加 Changelog Compact 算子链（L163）。
- `doCommit()`（L189）：流模式且开了 Checkpoint 先做硬校验 `assertStreamingConfiguration`（§5.4）→ 造 `CommitterOperatorFactory` → 可选包 `AutoTagForSavepointCommitterOperatorFactory`（Savepoint 自动打 Tag）/ `BatchWriteGeneratorTagOperatorFactory`（批模式打 Tag）→ 并行度锁 1 → 可选 `startNewChain()` 断链。

Writer 与 Committer 可分别配 `sink.writer-cpu/memory`、`sink.committer-cpu/memory`（`configureSlotSharingGroup` L246），实现资源隔离。

### 4.3 FlinkSinkBuilder 按 BucketMode 分发

`FlinkSinkBuilder.build()`（`FlinkSinkBuilder.java:236` 起 switch，**已修正：旧稿标 build 在 L207-243，实际 build() 在 L211、switch 在 L236**）按 `BucketMode` 分到五条链路：

| BucketMode | Sink 实现 | 数据分发 | 适用场景 |
|------------|----------|---------|---------|
| `HASH_FIXED` | `FixedBucketSink` | `hash(partition,bucket)%parallelism` | 固定桶，最稳定（默认） |
| `HASH_DYNAMIC` | `RowDynamicBucketSink` | `HashBucketAssignerOperator` 动态分桶 | 数据量波动大、桶自动扩缩 |
| `KEY_DYNAMIC` | `GlobalDynamicBucketSink` | `GlobalIndexAssigner` 全局索引 | 跨分区 upsert（主键不含分区列） |
| `BUCKET_UNAWARE` | `RowAppendTableSink` | 无 bucket | append-only 表 |
| `POSTPONE_MODE` | `PostponeBucketSink` | 延迟分桶 | 新增模式，写时不定桶 |

（五种 BucketMode 的存储语义、动态桶索引维护见 **01 号 §10.3**。）

- **非分区固定桶的并行度优化**（`buildForFixedBucket:292`，**已修正：旧稿标 L272-287**）：bucket 数 < 输入并行度时，自动把 Writer 并行度压到 bucket 数，省掉空闲 task。
- **Local Merge**（`build()` 内 L225-231，**已修正：旧稿标 L217-226**）：仅当 `local-merge-enabled` 且**有主键**才插 `LocalMergeOperator`，本地合并同 key 后再 shuffle，减少跨网络数据量；append-only 表无效。

### 4.4 Writer 算子继承链与提交触发

```
AbstractStreamOperator
  └ PrepareCommitOperator       内存池 + prepareSnapshotPreBarrier 触发 emitCommittables
      └ TableWriteOperator      初始化 StoreSinkWrite、状态恢复、commitUser
          └ RowDataStoreWriteOperator   processElement → write.write(row)
```

提交触发的关键回调（`PrepareCommitOperator.java:93`，行号准确）：

```java
public void prepareSnapshotPreBarrier(long checkpointId) throws Exception {
    if (!endOfInput) {
        emitCommittables(false, checkpointId);   // flush 内存 → 产出 CommitMessage 发往下游
    }
}
```

**为什么挑这个回调**：`prepareSnapshotPreBarrier` 是 Checkpoint Barrier 到达本算子**之前**的最后一个钩子，此刻 flush 能保证"Barrier 之前的数据已全部落盘"，与 Flink Checkpoint 语义严丝合缝（这就是 Phase 1）。

- **内存**（`setup():68`）：Managed Memory（Flink `MemoryManager` 统管）或 Heap（按 `write-buffer-size`/`page-size`）。
- **状态恢复**（`TableWriteOperator.initializeState():75`，行号准确）：用 `StateValueFilter`（基于 `ChannelComputer.select` 算哪些 state 属于本 subtask）过滤、经 `StoreSinkWrite.Provider` 建实例、装 `ConfigRefresher` 支持配置热更新。`processElement` 见 `RowDataStoreWriteOperator.java:54`。

### 4.5 StoreSinkWrite 三种写入策略

`StoreSinkWrite`（`StoreSinkWrite.java:49`）封装 Paimon Core 的 `TableWriteImpl`，`createWriteProvider()`（L100，行号准确）按 changelog 配置选实现：

```
createWriteProvider():
  changelogProducer = coreOptions.changelogProducer()              # L118
  if changelogProducer==FULL_COMPACTION || deltaCommits>=0:        # L138
      → GlobalFullCompactionSinkWrite(deltaCommits)                # L143
  if coreOptions.needLookup():                                     # L157
      → LookupSinkWrite                                            # L160
  else:
      → StoreSinkWriteImpl   (默认；writeOnly 时 waitCompaction=false)
```

| 实现 | 何时用 | 关键行为 |
|------|--------|---------|
| `StoreSinkWriteImpl` | 默认 / write-only | `write()`→`TableWriteImpl.writeAndReturn`；`prepareCommit()`→`prepareCommit(waitCompaction,cpId)` |
| `GlobalFullCompactionSinkWrite` | `changelog-producer=full-compaction` | 追踪 `writtenBuckets`，每 `deltaCommits` 个 checkpoint 对写过的桶做全量 compaction 产 changelog |
| `LookupSinkWrite` | `changelog-producer=lookup` | 恢复时对 `activeBuckets` 触发 compact 重建 lookup 缓存（lookup changelog 依赖本地 LSM 点查产旧值+新值） |

`StoreSinkWriteImpl.replace()`（`StoreSinkWriteImpl.java:173`，行号准确）支持 Schema 变更时热替换 Writer：`write.checkpoint()` 取状态 → `write.close()` → 用新表建 Writer → `restore(states)`。这是 CDC Schema 演进不重启作业的基础（§7.3）。changelog producer 各模式的产出机制详见 **24 号**；compaction 触发策略详见 **01 号 §8**。

---

## 5. Checkpoint 两阶段提交与 Exactly-Once

**① 要解决什么问题**：多个并行 Writer 分布式写入，要在 failover/重启时保证不重不漏（端到端 Exactly-Once），并让 Flink 的 Checkpoint 状态与 Paimon 的 Snapshot 严格一致——Checkpoint 成功了 Snapshot 也成功、Checkpoint 回滚了未提交数据也不可见。

**② 设计哲学**：**把 Paimon 的两阶段提交折叠进 Flink Checkpoint 生命周期**，不自造分布式事务协调器。Phase 1（prepare）= Writer 在 Barrier 前 flush；Phase 2（commit）= Committer 在 Checkpoint 成功后提交。容错点在于：Committable 在 Phase 1 和 Phase 2 之间存进 Flink state，任何环节失败都能从 state 重放未完成提交。

### 5.1 两阶段提交的完整时序

```
Phase 1（prepare，Barrier 到达前）            Phase 2（commit，Checkpoint 成功后）
─────────────────────────────              ──────────────────────────────
Writer.prepareSnapshotPreBarrier(cpId)      Committer.notifyCheckpointComplete(cpId)
  → StoreSinkWrite.prepareCommit()            → commitUpToCheckpoint(cpId)
  → flush 内存 buffer 到数据文件               → StoreCommitter.commit(committables)
  → 生成 CommitMessage                        → TableCommitImpl.commitMultiple()
  → emit Committable(cpId, CommitMessage)     → 原子写入新 Snapshot
        │                                            ▲
        ▼                                            │
Committer.snapshotState():  按 cpId 聚合 Committable → 存入 Flink state（容错点）
```

注意两个回调的先后：`snapshotState`（存 state）发生在 Checkpoint 进行中，`notifyCheckpointComplete`（真正提交）发生在 Checkpoint **整体成功之后**。这解释了"Checkpoint 成功 ≠ 数据可查"（§1.3 陷阱 2）：提交还要等下一个回调。


### 5.2 CommitterOperator 状态管理

源码路径: `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/sink/CommitterOperator.java`（以下行号均经核对）

**核心状态结构**:

```java
// 按 checkpointId 分组的 GlobalCommitT (ManifestCommittable)
protected final NavigableMap<Long, GlobalCommitT> committablesPerCheckpoint;
// 输入缓冲区
private final Deque<CommitT> inputs = new ArrayDeque<>();
```

**`processElement()` (L221)**: 只做两件事——转发元素和缓存输入。

```java
@Override
public void processElement(StreamRecord<CommitT> element) {
    output.collect(element);      // 转发给下游 (DiscardingSink)
    this.inputs.add(element.getValue());  // 缓存到输入队列
}
```

**`snapshotState()` (L164)**: Checkpoint Barrier 到达时:

1. `pollInputs()` —— 将输入队列中的 Committable 按 checkpointId 分组，合并为 `ManifestCommittable`
2. 通过 `committableStateManager.snapshotState()` 持久化到 state

**`pollInputs()` (L240)**: 用 `committer.groupByCheckpoint(inputs)` 分组、`committer.combine()` 合并为 `GlobalCommitT`。对 `END_INPUT_CHECKPOINT_ID`（`Long.MAX_VALUE`，L49）特殊处理——允许合并不报错，因为多个有界输入可能在不同时刻 endInput。

**`notifyCheckpointComplete()` (L190)**: Checkpoint 完成时触发实际提交:

```java
public void notifyCheckpointComplete(long checkpointId) throws Exception {
    super.notifyCheckpointComplete(checkpointId);
    commitUpToCheckpoint(endInput ? END_INPUT_CHECKPOINT_ID : checkpointId);
}
```

**`commitUpToCheckpoint()` (L195)**: 提取 `<= checkpointId` 的所有 committables，调用 `committer.commit()` 或 `committer.filterAndCommit()`。

### 5.3 StoreCommitter 实际提交

源码路径: `paimon-flink/paimon-flink-common/src/main/java/org/apache/paimon/flink/sink/StoreCommitter.java`

```java
public class StoreCommitter implements Committer<Committable, ManifestCommittable> {
    private final TableCommitImpl commit;           // Paimon Core 的提交器
    @Nullable private final CommitterMetrics committerMetrics;  // 提交指标
    private final CommitListeners commitListeners;   // 提交监听器
}
```

**`combine()` (L79)**: 将多个 `Committable` 合并为一个 `ManifestCommittable`（把各 Writer 的 `commitMessage` 收进同一个 checkpoint 的提交单元）。

**`commit()` (L99)**: 调用 `commit.commitMultiple(committables, false)`，最终走 Paimon Core 的 `FileStoreCommit` 执行原子 Snapshot 提交（提交时的冲突检测与重试见 **01 号 §13** 与 **15 号**）。

**`filterAndCommit()` (L107)**: 用于 `endInput` 场景（批作业结束），先检查 Snapshot 是否已存在以避免重复提交（幂等）。

### 5.4 三条硬约束及其根因

`doCommit` 启动时调 `FlinkSink.assertStreamingConfiguration()`（`FlinkSink.java:264`，行号准确）做前两条校验，不满足直接抛异常拒启动：

| 约束 | 根因（为什么不这样会出错） |
|------|------|
| **禁用 Unaligned Checkpoint** | Unaligned 允许 Barrier 越过尚未处理的数据。若 Writer 在 Barrier 时还没 flush 这部分数据，它们就不在本次 CommitMessage 里，却已被算作"本 Checkpoint 处理过"——提交内容与 Checkpoint 范围不一致，破坏 Exactly-Once。 |
| **必须 EXACTLY_ONCE** | AT_LEAST_ONCE 下 Barrier 可被数据超越，同一条记录可能落进相邻两个 Checkpoint 的 Committable，重启重放时重复提交。 |
| **Committer 并行度 = 1** | 每个 Checkpoint 只能产出一个全局 Snapshot。多 Committer 并发提交会撞 Snapshot ID、并发改 Manifest，需引入分布式锁。单点是最简单的正确实现。 |

### 5.5 状态恢复：commitUser 与 Writer State

**commitUser 持久化**（`CommitterOperator.java:131`，**已修正：旧稿标 L129-131**；Writer 侧 `TableWriteOperator.java:118`）：

```java
commitUser = StateUtils.getSingleValueFromState(
    context, "commit_user_state", String.class, initialCommitUser);
```

`initialCommitUser` 只对全新作业有效；一旦作业跑起来，commitUser 就被记进 write/commit 算子的 state，重启从 state 恢复、传入值被忽略。**为什么必须稳定**：Paimon 用 commitUser 识别"哪些未完成提交属于本作业"。变了，新 Committer 认不出旧的未完成提交，无法续提、且这些文件可能被当孤儿清理。源码注释明确：**不能用 Job ID**——Savepoint 恢复时 Job ID 会变。

**Writer 状态热替换**：`StoreSinkWriteImpl.replace()`（`StoreSinkWriteImpl.java:173`，行号准确）—— `write.checkpoint()` 取出内存状态 → `close()` → 用新表建 Writer → `restore(states)`。CDC Schema 演进靠它在不重启作业、不丢内存数据的前提下切到新 Schema（§7.3）。


---

## 6. CDC 同步入口（Flink 侧）

**① 要解决什么问题**：把 MySQL/PostgreSQL/MongoDB/Kafka/Pulsar 的变更实时入湖到 Paimon，且源表加字段时自动演进 Paimon 表无需停机，整库几百张表能共用一个作业。

**② 设计取舍**：内置 Action 命令直接集成 CDC 源，复用 §4/§5 整套 Sink + 两阶段提交链路，再叠加两件 CDC 专属能力：Schema 自动演进（靠 §5.5 的 `replace()` 热替换）与库级 COMBINED 多表复用。

> **交叉引用**：CDC 数据的解析（Debezium/Canal/Maxwell 格式）、类型映射规则、Schema 演进的细粒度策略由 **14 号文档**主讲。本节只讲 **Flink 侧的 Action 入口、构建流程与拓扑**——即"作业是怎么搭起来的"，不展开"数据是怎么解析的"。

### 6.1 Action 体系与构建流程

CDC 同步以命令行 Action 启动（`paimon-flink-action.jar` + 子命令如 `mysql-sync-table`/`mysql-sync-database`）。类层次（`paimon-flink-cdc/.../action/cdc/`）：

```
ActionBase
  └ SynchronizationActionBase           构建 CDC 流、建库、配 Watermark
      ├ SyncTableActionBase             单表：建/校验目标表，CdcSinkBuilder 造 Sink
      │   └ MySqlSyncTableAction / KafkaSyncTableAction / ...
      └ SyncDatabaseActionBase          库级：表名映射/过滤，FlinkCdcSyncDatabaseSinkBuilder
          └ MySqlSyncDatabaseAction / KafkaSyncDatabaseAction / ...
```

支持的源（`SynchronizationActionBase` 子类）：

| 源 | 表级 Action | 库级 Action | 格式 |
|----|------------|------------|------|
| MySQL | `MySqlSyncTableAction` | `MySqlSyncDatabaseAction` | Debezium |
| PostgreSQL | `PostgresSyncTableAction` | — | Debezium |
| MongoDB | `MongoDBSyncTableAction` | `MongoDBSyncDatabaseAction` | ChangeStream |
| Kafka | `KafkaSyncTableAction` | `KafkaSyncDatabaseAction` | Canal/Debezium JSON |
| Pulsar | `PulsarSyncTableAction` | `PulsarSyncDatabaseAction` | Canal/Debezium JSON |

`SynchronizationActionBase.build()`（`SynchronizationActionBase.java:131`，行号准确）三步搭链：

```java
public void build() throws Exception {
    syncJobHandler.checkRequiredOption();            // 校验源配置
    catalog.createDatabase(database, true);          // 建目标库
    beforeBuildingSourceSink();                      // 子类钩子：建/校验目标表
    DataStream<RichCdcMultiplexRecord> input =
        buildDataStreamSource(buildSource())         // ① CDC 源
            .flatMap(recordParse()).name("Parse");   // ② 解析为 RichCdcMultiplexRecord（细节见 14 号）
    EventParser.Factory<...> parserFactory = buildEventParserFactory();
    buildSink(input, parserFactory);                 // ③ 复用 §4 的 Paimon Sink
}
```

`run()`（L238）会自动开 Checkpoint（默认 3 分钟），再 `build()` + `execute()`——所以 CDC 作业天然满足 §5 的 Checkpoint 前置要求。

### 6.2 表级与库级同步

**表级**（`SyncTableActionBase.java`）的核心是 `beforeBuildingSourceSink()`（L115，行号准确）准备目标表：表存在则 `alterTableOptions` + `retrieveSchema` + `assertSchemaCompatible` 校验兼容；不存在则 `retrieveSchema` + `buildPaimonSchema` + `catalog.createTable` 自动建表。随后 `buildSink()`（L163）用 `CdcSinkBuilder` 接上目标 `FileStoreTable`、`typeMapping`、`catalogLoader`（Schema 演进需要它去 alterTable）。

> 注意：若源表无主键，自动建出的是 append-only 表，后续无法 UPDATE/DELETE——需 `--primary-keys` 显式指定。

**库级**（`SyncDatabaseActionBase.java`）用 `FlinkCdcSyncDatabaseSinkBuilder`（`buildSink` L242），关键配置：`including/excludingTables`（正则过滤表）、`including/excludingDbs`（多库）、`tablePrefix/Suffix`、`tableMapping`、`mergeShards`（合并分库分表）、`mode=COMBINED`。

**为什么库级用 COMBINED 而非每表一个 Sink**：COMBINED 下所有表共享一套 Writer + Committer，靠 `RichCdcMultiplexRecord` 里的表名路由，运行时新增表能动态接入。100 张表只占一个作业的算子，大幅省资源；代价是大表会和小表抢同一套 Writer，写入量极不均时宜把大表单独拆出。

### 6.3 Schema 自动演进与拓扑

演进触发链（解析与判定细节见 **14 号**）：CDC 记录出现未知字段/类型变更 → 生成 `SchemaChange` → `SchemaEvolutionClient` 调 `catalog.alterTable()` 原子改 Schema → `StoreSinkWrite.replace(newTable)` 热替换 Writer。支持 **ADD COLUMN** 与 **类型提升**（INT→BIGINT、FLOAT→DOUBLE）；**不支持** DROP/RENAME COLUMN、类型降级。

**为什么热替换而非重启**：CDC 作业长期运行，靠 §5.5 的 `TableWriteImpl.checkpoint()` + `restore()` 在不丢内存状态下平滑切到新 Schema。多 Writer 同时检测到新字段时，Paimon 用乐观锁保证只有一个 alterTable 成功、其余重试，可能引入秒级写入抖动。

拓扑（表级 / 库级 COMBINED）：

```
表级:    CDC Source → FlatMap(RecordParse) → CdcRecordStoreWriteOperator → CommitterOperator → DiscardingSink
库级:    CDC Source → FlatMap(RecordParse) → MultiplexCdcRecordWriteOperator(多表复用) → CommitterOperator → DiscardingSink
```

可见 CDC 拓扑与 §4 普通 Sink 同构，仅把 Writer 换成带 Schema 演进能力的 `CdcRecordStoreWriteOperator`（库级再换成多表复用版）。

---

## 7. Flink SQL 集成（Catalog / Factory / Procedure）

**① 要解决什么问题**：让用户用标准 Flink SQL（`CREATE CATALOG/TABLE`、`INSERT`、`SELECT`、`CALL`）访问 Paimon，复用 Flink SQL 生态（SQL Client、Hive Metastore），运维操作也能用 SQL 完成而非写 Java。

**② 设计原理**：实现 Flink 的三套 SPI——`Catalog`（FlinkCatalog 桥接 DDL）、`DynamicTableSource/SinkFactory`（FlinkTableFactory 造 §3 Source / §4 Sink）、`Procedure`（运维操作）。SQL 写入/读取最终都落到前面讲的 FlinkSource/FlinkSink，SQL 层只是薄薄的协议适配。

> 边界提示：Paimon 是存储而非计算引擎，OVER 窗口、MATCH_RECOGNIZE 等纯计算语义由 Flink 引擎承担、不在 Paimon 侧；FlinkCatalog 内部有缓存，外部（如 Spark）改了表后 Flink Session 可能看到旧元数据，需重建 Session。

### 7.1 FlinkTableFactory 体系

```
DynamicTableSourceFactory / DynamicTableSinkFactory   (Flink SPI)
  └ AbstractFlinkTableFactory     造 DataTableSource / FlinkTableSink、加载 Paimon Table
      └ FlinkTableFactory         额外支持 auto-create 自动建表
```

`createDynamicTableSource()`（`AbstractFlinkTableFactory.java:85`）按流/批决定 `unbounded`，系统表（`$snapshots` 等）走 `SystemTableSource`，普通表走 `DataTableSource`（内部封装 §3 的 FlinkSource，支持分区裁剪/列裁剪/Filter/Limit 下推）。`auto-create`（`FlinkTableFactory.java:64`）让 SQL 里 CREATE 后立即可用——仅适合开发环境，生产应显式建表。

### 7.2 FlinkCatalog 桥接

`FlinkCatalog`（`paimon-flink-common`）把 Flink Catalog API 一一映射到 Paimon Catalog：

| Flink Catalog API | Paimon Catalog API |
|---|---|
| `createDatabase` / `createTable` / `alterTable` | 同名 `catalog.*` |
| `getTable` | `catalog.getTable()` → 包装为 Flink `CatalogTable` |
| `listTables` | `catalog.listTables()` |

后端可为 Hive Metastore / Filesystem / JDBC / REST（Catalog 体系详见 **03 号**）。

### 7.3 Procedure 体系

Paimon 提供 30+ Flink SQL Procedure（`CALL sys.xxx(...)`），按域分类（功能详见各主讲文档）：

| 域 | 代表 Procedure | 主讲 |
|----|---------------|------|
| Snapshot | `expire_snapshots`、`rollback_to`、`rollback_to_timestamp`、`expire_changelogs` | 01/17 |
| Tag/Branch | `create_tag`、`expire_tags`、`create_branch`、`fast_forward` | 17/41 |
| Compaction | `compact`、`compact_database`、`compact_manifest` | 本篇 §8、11/23 |
| 数据管理 | `merge_into`、`drop_partition`、`remove_orphan_files`、`remove_unexisting_files` | 11 |
| 迁移/克隆 | `migrate_table`、`clone`、`copy_files` | — |
| 其他 | `query_service`、`rescale`、`rewrite_file_index`、`vector_search` | 21/13/28 |

每个 Procedure 是一个独立类（如 `CompactProcedure`），位于 `paimon-flink-common/.../flink/procedure/`。注意多数 Procedure 非事务性（如 `compact`），中途失败可能留下中间态，宜低峰执行；重负载 compaction 应走 §8 Dedicated Compaction 而非阻塞式 `CALL compact`。

---

## 8. Dedicated Compaction

**① 要解决什么问题**：默认 Writer 内置 Compaction（写时顺带压缩），但 Compaction 是 CPU/IO 密集型，会和写入抢资源、造成写入抖动，也无法独立伸缩或单独重启。

**② 设计取舍**：把 Compaction 从写入作业剥离成**独立作业**——写入作业配 `write-only=true` 只写不压，另起一个 Compaction 作业专做压缩，两者共享同一张 Paimon 表、各自独立资源/并行度/容错。

```
写入作业 (write-only=true):  数据流 → Writer(只写) → Committer → Snapshot
                                          │ 共享 Paimon 表
Compaction 作业:            CompactorSource → CompactorSink(StoreCompactOperator) → Committer → Compact Snapshot
```

启动方式：`CompactAction`（命令行 `compact` 子命令，可加 `--continuous` 转流式、`--partition`/`--partition-idle-time` 过滤）或 SQL `INSERT INTO t SELECT * FROM t$compact_buckets`。

**CompactorSourceBuilder**（`CompactorSourceBuilder.java`）用系统表 `CompactBucketsTable` 扫描"待压缩的 bucket"，按模式分叉：

| 模式 | 配置 | 行为 |
|------|------|------|
| 批 | `scan.mode=latest-full, batch-scan-mode=compact` | 一次性扫全部待压文件，跑完即止（幂等，失败重跑即可） |
| 流 | `stream-scan-mode=compact-bucket-table` | 持续监控新 Snapshot，增量发现待压 bucket（需开 Checkpoint 才能 failover 续跑） |

`withPartitionIdleTime()`（`CompactorSourceBuilder.java:83`，**已修正：旧稿标 L128-143**）：批模式按分区空闲时间过滤，只压"冷"分区，避免压正在写的分区做无用功。

**CompactorSinkBuilder**（`CompactorSinkBuilder.java`）只支持 `HASH_FIXED`/`HASH_DYNAMIC`（`build()` 内 switch），`buildForBucketAware()`（L63）用 `BucketsRowChannelComputer` 按 partition+bucket 分发给 `CompactorSink`（继承 §4 的 `FlinkSink`），内部用 `StoreCompactOperator` 执行压缩，仍经 §5 的 `CommitterOperator` 提交结果——可见 Compaction 作业复用了整套 Sink + 两阶段提交基建。

**④ 关键陷阱**：① 写入作业忘配 `write-only=true` → 双方重复压缩、互相冲突（§1.3 陷阱 7）；② Compaction 并行度应 ≈ bucket 数，过高大量 task 空闲；③ 流模式必须开 Checkpoint 才能 failover 续跑；④ Compaction 策略由表配置决定，作业无需额外设。

**⑤ 收益与代价**：写入性能稳定不受压缩干扰、可独立弹性伸缩、可用低成本资源跑压缩；代价是多一个作业要部署运维。Compaction 策略本身（UniversalCompaction、触发条件）详见 **01 号 §8**，小文件治理详见 **11 号**。

---

## 9. Lookup Join 机制

**① 要解决什么问题**：流式计算关联维表（如订单流关联用户表），既要低延迟（不能每条都读存储），又要能装下大维表（不能全塞内存），还要维表更新后缓存及时刷新。

**② 设计原理**：Paimon 复用自身 LSM 点查能力做维表 Lookup，提供三档缓存策略按需取舍延迟/内存/启动速度——全量缓存（FullCache，小表延迟最低）、部分缓存（PrimaryKeyPartial，大表按需加载）、远程（Remote，走查询服务）。`lookup.cache-mode=auto` 时自动选：join key=主键走 Partial，否则走 Full。

> 适用边界：Lookup Join 只适合"大流 join 小/中维表"，大表 join 大表会 OOM。Lookup 点查底层（LookupLevels、本地 KV 索引构建）见 **01 号 §6**；远程查询服务见 **21 号**。

### 9.1 FileStoreLookupFunction 与缓存选择

`FileStoreLookupFunction`（`FileStoreLookupFunction.java:167`，**已修正：旧稿把 open 标为 L182-234，实际 open 在 L167，选择逻辑 L199-232**）是 Lookup Join 的 Flink 函数。`open()` 选缓存：

```
if (lookup.cache-mode == AUTO && joinKeys == primaryKeys):   # L199
    远程服务可用 → PrimaryKeyPartialLookupTable.createRemoteTable()   # L203
    否则 try    → PrimaryKeyPartialLookupTable.createLocalTable()    # L209（本地 LSM 点查）
                catch 不支持 → 继续降级
if lookupTable == null:                                       # L231
    → FullCacheLookupTable.create(context, lookup.cache-rows)  # L232（全量缓存）
```

`lookup()`（L278，行号准确）：先 `tryRefresh()` 看是否要刷新缓存，再查；有分区维表则遍历所有活跃分区分别查找后合并。

### 9.2 三种缓存实现的边界

`LookupTable` 体系：

```
LookupTable
  ├ FullCacheLookupTable（全量加载到本地缓存）
  │   ├ PrimaryKeyLookupTable        有主键，joinKey==主键
  │   ├ SecondaryIndexLookupTable    有主键，joinKey≠主键（需文件索引，否则全表扫）
  │   └ NoPrimaryKeyLookupTable      无主键（append-only）
  └ PrimaryKeyPartialLookupTable（按需加载，joinKey==主键）
      ├ LocalTable    本地 LocalQueryExecutor 查 LSM
      └ RemoteTable   走 paimon-service 远程查询
```

`FullCacheLookupTable.create()`（`FullCacheLookupTable.java:366`，行号准确）按表特征选上述三个子类。存储后端（`createStateFactory:171`）：`lookup.cache-mode=memory` 用 `InMemoryStateFactory`（最快、受堆内存限制），默认用 `RocksDBStateFactory`（本地 RocksDB，可装下超内存的大表）。`bootstrap()`（L181）全量加载时，RocksDB 后端走 `BinaryExternalSortBuffer` 排序 + `RocksDBBulkLoader` 批量灌入加速。

| 实现 | 何时用 | 内存/启动 |
|------|--------|----------|
| FullCache | 小表，或 joinKey≠主键 | 全量装内存/RocksDB，启动慢 |
| PrimaryKeyPartial（Local/Remote） | 大表 + joinKey=主键 | 按需加载，启动快、内存省 |

**为什么 Partial 要求 joinKey=主键**：按需加载需要靠主键直接在 LSM 中定位记录；joinKey 不是主键时无法定位，只能退回 FullCache 的 `SecondaryIndexLookupTable`（且需在 join 列上配 `file-index.bloom-filter.columns`，否则全表扫描，§1.3 陷阱 8）。

### 9.3 缓存刷新与 Bucket 感知

`tryRefresh()`（`FileStoreLookupFunction.java:339`，行号准确）四步：① 查刷新黑名单（高峰期禁刷避免 IO 抖动，配 `lookup.refresh-time-periods-blacklist`）；② 异步分区刷新完成则原子切换 LookupTable；③ 刷新动态分区（`max_pt()` 只加载最新分区）；④ 到达 `refresh-interval` 则 `lookupTable.refresh()`（增量）或重 `open()`（全量）。

`shouldDoFullLoad()`（L401，行号准确）：积压快照数超 `refreshFullThreshold` 时，改全量重载而非增量刷新——因为积压太多时逐快照增量比一次全量更慢。`refresh()` 支持同步/异步（`lookup.refresh-async` + 线程数），异步刷新查询不阻塞但可能读到旧值。

**Bucket 感知**（`getRequireCachedBucketIds():493`，行号准确）：Flink Planner 按 join key 把数据 shuffle 到固定 subtask 后，每个 subtask 只需缓存自己负责的 bucket，内存降到 1/N（N=并行度）。`strategy==null` 时返回 null（缓存所有 bucket）；非空时 FullCache 在 bootstrap/refresh 只加载指定 bucket。**注意**：启用 Bucket 感知时并行度必须 ≥ bucket 数，否则部分 bucket 无人负责、查询出错。

---

## 10. 与 Iceberg Flink Sink 对比

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| **Committer 并行度** | 强制为 1 (FlinkSink L228-L229) | 强制为 1 (IcebergFilesCommitter) |
| **提交触发** | `notifyCheckpointComplete(cpId)` (CommitterOperator L190) | `notifyCheckpointComplete(cpId)` |
| **数据 flush 时机** | `prepareSnapshotPreBarrier` (PrepareCommitOperator L93) | `prepareSnapshotPreBarrier` |
| **提交内容** | `ManifestCommittable` (CommitMessage 列表) | `WriteResult` (DataFile + DeleteFile) |
| **Unaligned Checkpoint** | 不支持 (FlinkSink L266) | 不支持 |
| **Checkpoint 模式** | 仅 EXACTLY_ONCE (FlinkSink L271) | 仅 EXACTLY_ONCE |
| **Compaction** | Writer 内置 LSM Compaction + 可选 Dedicated Compaction | 独立的 Compaction 作业或触发机制 |
| **Changelog 生成** | 内置多种 changelog-producer (input/lookup/full-compaction) | 不支持原生 changelog |
| **Streaming 写入** | 原生支持 (核心设计目标) | 支持但不是主要场景 |
| **Lookup Join** | 原生支持 (FullCache/PartialCache/Remote) | 不支持 |
| **Schema Evolution (CDC)** | 内置自动 Schema 演进 (ADD COLUMN / Type Promotion) | 需要外部工具 |
| **Bucket 模式** | 5 种模式 (Fixed/Dynamic/KeyDynamic/Unaware/Postpone) | 无 Bucket 概念 (分区内文件) |
| **状态恢复** | commitUser 持久化 + Writer State 恢复 | Committer State 恢复 |

**核心差异总结**:

1. **实时性**: Paimon 以实时湖仓为核心设计目标，提供了丰富的流式能力 (changelog producer, lookup join, CDC sync)。Iceberg 更侧重于批处理和 ACID 事务。

2. **Compaction 策略**: Paimon 的 LSM 结构使得 Compaction 可以与写入同时进行 (Writer 内置)，也可以拆分为独立作业。Iceberg 的 Compaction 通常是独立的后台操作。

3. **数据分发**: Paimon 通过 BucketMode 精确控制数据如何分布到文件中，而 Iceberg 依赖分区和排序。Bucket 机制使得 Paimon 可以实现更高效的点查和 Lookup Join。

4. **CDC 能力**: Paimon 内置了完整的 CDC 同步链路 (多种源、Schema 演进、多表同步)，这在 Iceberg 生态中需要借助额外工具（如 Flink CDC + Iceberg Connector 组合）。

---

## 11. 设计决策总结

| 决策点 | 选择 | 取舍/代价 | 收益 |
|--------|------|-----------|------|
| 提交编排 | 把两阶段提交折叠进 Flink Checkpoint（prepareSnapshotPreBarrier + notifyCheckpointComplete） | 提交延迟 ≈ 一个 Checkpoint 周期 | 无需自造分布式事务协调器，端到端 Exactly-Once 与 Flink 语义天然对齐 |
| Committer 并行度 | 强制单并行 | 大并行度下可能成瓶颈（可单独配资源缓解） | 每 Checkpoint 唯一 Snapshot，免分布式锁、免 Manifest 写冲突 |
| Checkpoint 模式 | 仅 EXACTLY_ONCE + 禁 Unaligned（启动硬校验） | 不能用 AT_LEAST_ONCE/Unaligned 提吞吐 | 提交内容与 Checkpoint 范围严格一致，杜绝重复/错配 |
| Source 架构 | Enumerator(JM 单点) / Reader(TM) 分离 + 增量 plan | Enumerator 是单点（靠 Checkpoint 容错） | 扫描集中、易反压、只读新 Snapshot 省 CPU/IO |
| Source 反压 | splitMaxNum + maxSnapshotCount 两闸 | 配置不当仍会积压 | 下游慢时自然回压、防 JM OOM |
| 有序性 | 主键表强制有序读 | 牺牲部分吞吐 | 保证 CDC changelog 顺序、主键状态正确 |
| 数据分发 | 按 bucket 路由（Bucket 感知） | bucket 数需与并行度匹配 | 同 key 落同 task 有序、点查/Lookup 高效 |
| commitUser | 稳定 ID 存入 Flink state | 不能用 Job ID、不能随机 | 重启续提未完成提交、不误清孤儿文件 |
| Schema 演进 | replace() 热替换 Writer | 多 Writer 并发 alterTable 有秒级抖动 | CDC 加字段不停机 |
| Compaction 部署 | Writer 内置 + 可选 Dedicated（write-only） | Dedicated 需多部署一个作业 | 写入性能稳定、可独立伸缩 |
| Lookup 缓存 | Full / Partial / Remote 三档 + Bucket 感知 | 各有 OOM/未命中延迟/网络开销边界 | 按维表规模与 join key 取最优 |
| 版本适配 | Common + 薄适配层（Maven Profile） | 公共层只能用版本公共 API | 复用率高、新增 Flink 版本成本低 |

---

> 本文档基于 Paimon 1.5-SNAPSHOT 源码分析，如有更新请以最新代码为准。
