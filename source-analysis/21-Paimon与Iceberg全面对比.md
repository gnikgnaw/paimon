# Apache Paimon vs Apache Iceberg 全面深度对比

> **版本**：Paimon 1.5-SNAPSHOT；Iceberg 以 v3 规范（2025 年 1.8–1.10 系列落地）为现状基准　**源码模块**：`paimon-core` / `paimon-common` / `paimon-iceberg`　**核对日期**：2026-06

**一句话定位**：这是一篇**选型对比文档**，帮你回答"在我的场景下该选 Paimon 还是 Iceberg"——核心是把两者在架构、主键更新、流式/CDC、Compaction、快照与时间旅行、索引、生态成熟度等维度上**各自怎么做、取舍是什么、谁更适合什么场景**讲清楚，而不是单独讲透某一个机制。

读完本文你应能回答：① Paimon 的 LSM Merge-Tree 与 Iceberg 的"不可变文件 + Delete/DV"在**更新这件事上**的根因差异是什么；② 为什么 Paimon 需要 Bucket 而 Iceberg 不需要；③ 流式 CDC / 增量消费场景下两者的能力边界各在哪；④ Compaction 到底是"内置自动"还是"外部触发"——以及 Iceberg 在 2025 年后这条结论变了多少；⑤ 索引、快照、时间旅行、Catalog 生态各自强在哪；⑥ 一张决策表面前，给定我的场景该怎么选，以及"Paimon 写 + Iceberg 读"的混合架构是否可行。

> 阅读约定：Paimon 侧的机制论断均已用 `Grep`/`Read` 对照 1.5-SNAPSHOT 源码核对（标 `路径:行号`，与旧稿不符处标 `（已修正）`）；涉及的内部细节点到为止，深入展开见交叉引用（LSM→01、Compaction→23、快照/时间旅行→17、索引→13、DeletionVector→04）。Iceberg 侧基于其公开规范（区分 v2/v3 能力，注意 v3 是 2025 年才成熟落地的现状），拿不准的标注"需视版本/实现"，不臆测。

---

## 目录

- [1. 快速理解（总览对比表 / 选型速查 / 高频误区）](#1-快速理解总览对比表--选型速查--高频误区)
  - [1.1 一张总览对比表](#11-一张总览对比表)
  - [1.2 选型决策树](#12-选型决策树)
  - [1.3 核心概念速查表](#13-核心概念速查表)
  - [1.4 五个最常见的选型误区](#14-五个最常见的选型误区)
- [2. 架构与设计哲学：流优先 vs 格式标准](#2-架构与设计哲学流优先-vs-格式标准)
- [3. 存储模型：LSM + Bucket vs 扁平文件 + Delete/DV](#3-存储模型lsm--bucket-vs-扁平文件--deletedv)
- [4. 元数据管理：base/delta 分离 vs 统一 manifest list](#4-元数据管理basedelta-分离-vs-统一-manifest-list)
- [5. 主键更新：MergeFunction vs Delete File / DV](#5-主键更新mergefunction-vs-delete-file--dv)
- [6. Compaction：内置自动 vs 外部/原生维护任务](#6-compaction内置自动-vs-外部原生维护任务)
- [7. 流式与 CDC：原生 Changelog vs 增量扫描](#7-流式与-cdc原生-changelog-vs-增量扫描)
- [8. 引擎集成：Flink / Spark 谁更深](#8-引擎集成flink--spark-谁更深)
- [9. 索引能力：多种文件索引 + Lookup vs 列统计 + Puffin](#9-索引能力多种文件索引--lookup-vs-列统计--puffin)
- [10. 快照、时间旅行与版本管理](#10-快照时间旅行与版本管理)
- [11. 生态与成熟度：Catalog / 引擎 / 云](#11-生态与成熟度catalog--引擎--云)
- [12. 选型决策总结](#12-选型决策总结)

---

## 1. 快速理解（总览对比表 / 选型速查 / 高频误区）

一句话先把两者的"根因差异"钉死：**Paimon 把流式更新的能力做进了存储引擎本身（LSM Merge-Tree + 内置 Compaction + 原生 Changelog），Iceberg 把自己定位成引擎无关的表格式规范（不可变文件 + 元数据快照），更新/维护交给外部或可插拔的引擎能力。** 这一条决定了后面所有维度的取舍走向。

### 1.1 一张总览对比表

| 维度 | Paimon | Iceberg | 谁更强 / 取舍 |
|------|--------|---------|--------------|
| 一句话定位 | 面向流的实时湖仓格式，存储引擎自带 LSM | 引擎无关的开放表格式规范 | 看场景：流式 vs 通用分析 |
| 诞生背景 | 由 Flink Table Store 演化，天生为流式设计 | Netflix 为解决 Hive 元数据瓶颈而生 | — |
| 主键/更新核心结构 | LSM Merge-Tree（主键表）+ Append（追加表），主键是一等公民 | 不可变数据文件 + Delete File / Deletion Vector，主键非核心 | 高频更新 → Paimon |
| 更新写入代价 | 追加一条新版本，O(1) 入内存缓冲 | 写 Delete/DV + 数据文件，需先定位旧行 | 高频更新 → Paimon |
| 更新读取代价 | Merge-on-Read 归并多层（Compaction 后收敛） | 数据文件关联 Delete/DV 过滤 | 看 Compaction/治理是否跟上 |
| Compaction | 写入路径**内置自动**触发 + 反压 | 不在写路径内自动跑；靠 Spark/Flink 维护任务触发（Flink 1.7+ 已可不依赖 Spark） | Paimon 省运维；Iceberg 更灵活但需调度 |
| 流式 Changelog | 原生四档 Changelog（NONE/INPUT/FULL_COMPACTION/LOOKUP） | 无原生完整 CDC，靠增量扫描重建 | 流式消费 → Paimon |
| CDC 整库入湖 | 内置 `paimon-flink-cdc`，自动建表 + Schema 演进 | 无内置工具；靠 Flink CDC + Dynamic Sink 组合 | Paimon 开箱即用 |
| 索引 | Bloom/Bitmap/BSI/RangeBitmap 文件索引 + Lookup 点查 + 动态桶全局索引 | 列统计 + Partition Summary + Puffin 内 Bloom（需配置） | 点查/细粒度过滤 → Paimon |
| 快照格式 | Snapshot 为 JSON（version=3），可读易运维 | `metadata.json`（JSON）+ manifest-list（Avro），format-version v1/v2/v3 | 各有取舍 |
| 时间旅行 / 分支 | Tag/Branch/Consumer，支持自动打 Tag、Watermark 回滚 | SnapshotRef 统一管 Tag/Branch，支持 cherry-pick | 流式语义 → Paimon；分支语义成熟度 → Iceberg |
| Catalog / 引擎生态 | Filesystem/Hive/JDBC/REST；深耕 Flink、Spark、国内云 | Filesystem/Hive/JDBC/REST/Glue/Nessie/Polaris/Unity；引擎覆盖最广 | 生态广度 → Iceberg |
| 社区成熟度 | 较新（毕业 Apache 顶级项目，国内落地多） | 成熟，事实标准之一 | Iceberg |

> 这张表是全篇的索引：每一行在后续对应章节会展开"为什么 + 取舍 + 适用场景"。

### 1.2 选型决策树

把选型简化成几个先后判断（从上往下，命中即停）：

```
1. 业务以「高频 Upsert / CDC 实时入湖 / 流式消费下游」为主？
      是 → Paimon（LSM + 原生 Changelog 是硬优势，Iceberg 难追平）
      否 → 进入 2
2. 需要「多引擎（Trino/Presto/Snowflake/Doris/StarRocks…）混合读」且以批分析为主？
      是 → Iceberg（引擎与 Catalog 生态覆盖最广）
      否 → 进入 3
3. 更新频率：每天/每小时级的低频批量更新，且追加为主？
      是 → 两者都行，按团队技术栈选（Spark 重 → Iceberg；Flink 重 → Paimon）
      否（分钟级以上的更新） → 偏向 Paimon
4. 既要实时写、又要广引擎批量读？
      → 考虑「Paimon 写 + paimon-iceberg 暴露 Iceberg REST 元数据给下游读」的混合架构（§11.4）
```

判据排序的依据：**更新频率 > 流式需求 > 引擎生态 > 团队技术栈**。前两条是 Paimon/Iceberg 的根因分水岭，后两条是同等可行时的权衡。

### 1.3 核心概念速查表

| 概念 | 一句话定义 | 关键源码 / 出处 |
|------|-----------|----------------|
| **LSM Merge-Tree** | 把随机写转顺序写：内存缓冲 → flush 成 Level-0 → 后台 Compaction 逐级合并 | Paimon `KeyValueFileStore`；详见 01 |
| **Bucket** | Paimon 中主键数据的物理分组单元，每桶一棵独立 LSM 树 | `BucketMode.java:30`（共 5 种模式） |
| **MergeFunction** | 同主键多条记录如何合并：去重/部分更新/聚合/取首条 | `CoreOptions.MergeEngine`（`CoreOptions.java:3910`） |
| **ChangelogProducer** | Paimon 产生 CDC 流的四种模式 | `CoreOptions.ChangelogProducer`（`CoreOptions.java:4051`） |
| **不可变文件** | Iceberg 数据文件写后不改，更新靠追加 Delete/DV 标记删除 | Iceberg 规范 |
| **Delete File** | Iceberg v2 的删除标记：Position Delete / Equality Delete | Iceberg v2 规范 |
| **Deletion Vector (DV)** | v3 的二进制位图删除向量，存于 Puffin，**取代** v2 的 Position Delete | Iceberg v3 规范（2025） |
| **Snapshot** | 表在某时刻的完整状态。Paimon 为 JSON，Iceberg 为元数据 + manifest 链 | Paimon `Snapshot.java`（version=3）；详见 17 |
| **base/delta manifest** | Paimon 把全量基线与增量变更分两个 manifest list，加速增量读 | `Snapshot.java:54-59`（已修正路径） |

### 1.4 五个最常见的选型误区

**误区 1：「Iceberg 没有 Compaction，一定要自己搭 Airflow」——已部分过时。** 旧资料常说 Iceberg 必须靠外部 Spark Action 做维护。现状是：Iceberg 自 1.7 起在 Flink runtime 内置了 `TableMaintenance`（compaction / expire / orphan 清理），1.10 进一步把 Flink/Spark 的 compaction 逻辑统一，可在 Flink 作业内原生跑维护而**不再强依赖 Spark 集群**。关键差异仍在：Iceberg 的 compaction **不是写入路径内自动触发**的，要由维护任务调度；Paimon 的 compaction 在写入时按策略自动触发并带反压（§6）。

**误区 2：「Iceberg 更新就是写 Delete File，永远慢」——要分 v2/v3。** v2 的 Position/Equality Delete 累积后确实拖慢读；v3 用 Deletion Vector（Puffin 二进制位图）替代 Position Delete，删除/更新更快、compaction 成本更低。对比时务必说清楚是 v2 还是 v3。

**误区 3：「Paimon 不支持多引擎」。** Paimon 支持 Flink/Spark/Hive/Trino 等，且 `paimon-iceberg` 模块能把元数据以 Iceberg REST 协议暴露给下游读（§11.4）。准确说法是：Paimon 引擎生态广度不如 Iceberg，但并非锁死单引擎。

**误区 4：「主键表 = Paimon 专属，Iceberg 不能做 Upsert」。** Iceberg 通过 MERGE/Upsert + Delete/DV 也能做更新，只是没有 Paimon 的 MergeFunction（部分更新、聚合）这类下推到存储层的合并语义，且高频小批量更新代价更高（§5）。

**误区 5：「选 Paimon 运维一定更省心」。** Paimon 把 Compaction 内置了，但 Bucket 数、`num-sorted-run.*`、`write-buffer-size` 等参数调优需要懂 LSM 原理，调错会写阻塞或读放大（§6 陷阱）。省的是"搭调度系统"，不省的是"懂 LSM"。

---

## 2. 架构与设计哲学：流优先 vs 格式标准

**① 两者到底在解决什么不同的问题**

- **Paimon** 要解决的是"流式数据高频写入并快速可见"。数据从 Kafka、MySQL Binlog 持续流入，一个热点主键在一个 Flink Checkpoint 间隔（秒级到分钟级）内可能被更新几十上百次。如果每次更新都重写文件，写入吞吐会崩溃。
- **Iceberg** 要解决的是"在对象存储上为大规模数据湖提供可靠、引擎无关的表语义"。Netflix 在 Hive 上遇到元数据瓶颈（分区数到数万时 Hive Metastore 崩溃），需要一个可扩展、不依赖任何常驻进程、任何引擎都能读写的表格式。

这两个出发点导出了两条根本不同的设计路线，是后面所有维度差异的源头。

**② 设计原理与取舍**

| 设计选择 | Paimon：LSM Merge-Tree | Iceberg：不可变文件 + 元数据快照 |
|----------|------------------------|----------------------------------|
| 写入哲学 | 一切写入皆顺序追加；更新 = 写新版本 + 后台合并 | 文件写后永不改；更新 = 追加 Delete/DV 标记 |
| 对象存储适配 | 优（纯顺序写，吻合 S3/OSS 只追加特性） | 优（写一次读多次，天然适配） |
| 是否依赖常驻进程 | 写入端通常常驻（Flink 作业内做 Compaction） | 否，格式即规范，任何引擎可独立读写 |
| 更新延迟 | 秒级（内存吸收 + Checkpoint 提交） | 取决于写入作业 Commit 间隔 |
| 并发安全 | 桶级 OCC 冲突检测 + 原子重命名提交 | OCC 乐观并发，提交时检测冲突 |
| 复杂度落点 | 复杂度在存储引擎内部（Compaction/反压/内存抢占） | 复杂度在用户侧（维护任务调度、更新性能管理） |

**一句话设计哲学**：Paimon 把"流式更新好用"做进了存储引擎；Iceberg 把"任何引擎都能可靠读写"做成了格式规范。前者用更高的系统复杂度换流式性能与低运维，后者用更新性能与维护职责的外移换引擎无关性与生态广度。

**③ 各自的好处与代价（落到可验证事实）**

- Paimon 好处：流式写入秒级可见、更新吞吐高（顺序写）、内置 Changelog 利于 CDC 下游、主键点查有 Lookup 加速。代价：Merge-on-Read 读放大需靠 Compaction 收敛、LSM 参数调优是运维核心、引擎生态广度不及 Iceberg。
- Iceberg 好处：引擎无关、批查询友好、Catalog/引擎生态最广、Schema/Partition Evolution 成熟。代价：v2 下高频更新 Delete File 累积拖慢读（v3 DV 已显著改善）、原生流式 CDC 能力弱、维护任务需自行调度。

**④ 与同类格式的横向定位**（帮助理解坐标，非本文重点）

- **Delta Lake**：不可变文件 + Deletion Vector，与 Spark 耦合更紧；定位接近 Iceberg。
- **Hudi**：提供 MOR/COW，MOR 用 Log File，思路接近 Paimon 的"增量 + 合并"，但实现更重。
- **Kudu/HBase**：同样 LSM，但是独立存储系统而非文件格式，需常驻服务进程。

Paimon 与 Iceberg 不是简单竞品，而是"流优先存储引擎"与"通用表格式标准"两个生态位，很多企业实际是组合使用（§11.4）。

---

## 3. 存储模型：LSM + Bucket vs 扁平文件 + Delete/DV

**① 要解决什么问题**

主键表的核心难题是：如何在不重写大文件的前提下高效更新，并在查询时快速合并出最新值。两者答案不同——Paimon 用 Bucket 限定 Merge 范围 + LSM 分层控制读写放大；Iceberg 用全局 Delete/DV 标记，避免重写不可变数据文件。

**② 模型架构对照**

```
Paimon 主键表 (KeyValueFileStore)              Iceberg 表 (v2/v3)
┌─────────────────────────────┐                ┌──────────────────────────────┐
│  Partition                  │                │  Partition                   │
│  ├── Bucket 0               │                │  ├── data-00001.parquet      │
│  │   ├── Level-0 (无序文件)  │                │  ├── data-00002.parquet      │
│  │   ├── Level-1 (有序文件)  │                │  ├── delete-00001.parquet    │ (v2 Position Delete)
│  │   ├── Level-2 ...        │                │  ├── eq-delete-001.parquet   │ (v2 Equality Delete)
│  │   └── Level-max          │                │  └── dv-00001.puffin         │ (v3 Deletion Vector)
│  ├── Bucket 1 ...           │                └──────────────────────────────┘
│  └── ...                    │                 注：v3 中 Position Delete 文件不再新增，
└─────────────────────────────┘                     由 DV 取代；Equality Delete 仍可用。
```

**③ 为什么 Paimon 需要 Bucket、Iceberg 不需要**

Paimon 的 `Bucket` 是 LSM 树的组织单元，五种模式见 `BucketMode.java:30`（**已修正：旧稿 §2.3 只列了 3 种且把 POSTPONE 写成"延迟分桶"含糊带过**）：

| BucketMode | 触发条件 | 作用 | 适用 |
|------------|----------|------|------|
| `HASH_FIXED` | `bucket=N`（默认） | 主键哈希取模分桶，桶数离线才能改 | 数据量可预估 |
| `HASH_DYNAMIC` | `bucket=-1` 且主键含全部分区字段 | 维护主键 hash→桶映射，自适应桶数 | 桶数难预估、单写入 |
| `KEY_DYNAMIC` | `bucket=-1` 且跨分区更新（主键不含全部分区字段） | 维护主键→(分区,桶) 映射，启动读全表主键建索引 | 跨分区 upsert |
| `BUCKET_UNAWARE` | 追加表无桶 | 忽略桶概念，读写并行不受限 | 追加表 |
| `POSTPONE_MODE` | `bucket=-2` | 桶数后台自适应调整 | 主键表桶数难定 |

判定逻辑见 `KeyValueFileStore.java:91-101`：`-2`→POSTPONE，`-1`→按是否跨分区更新选 KEY_DYNAMIC/HASH_DYNAMIC，其余→HASH_FIXED（已核对）。

Bucket 的本质是**限制 Merge-on-Read 的扫描范围**：同一主键所有版本必落在同一桶，合并只在桶内进行；每桶一棵独立 LSM，可并行写入与 Compaction。Iceberg 数据文件不可变、更新靠全局 Delete/DV 标记，无 LSM 分层，自然不需要按桶组织——它用 Partition 做粗粒度数据组织，靠列统计跳过无关文件。

**④ 三种放大的取舍（核心对比）**

| 放大类型 | Paimon (LSM) | Iceberg (不可变 + Delete/DV) |
|----------|--------------|------------------------------|
| **写放大** | 中。Compaction 重写文件，受 `maxSizeAmp`（默认 200%）约束；但写入本身顺序、无随机写 | 低（更新只追加 Delete/DV）；但 `RewriteDataFiles` 维护时写放大高（重写整文件） |
| **读放大** | 低到中。Compaction 后读单层即可；未压实时归并多 SortedRun。Lookup 点查 O(1) | 中到高（v2）：每个数据文件需关联 Delete 过滤，Equality Delete 逐行匹配最慢。v3 DV 用位图过滤显著降低 |
| **空间放大** | 低到中，过期数据在 Compaction 后清理，受 `maxSizeAmp` 控制 | 中到高：Delete/DV 标记删除不立即物理回收，compaction 前原数据与删除标记并存 |

**⑤ 追加表：Paimon 也能像 Iceberg**

Paimon 两种 FileStore：`KeyValueFileStore`（主键表，LSM）与 `AppendOnlyFileStore`（追加表，无主键、文件直接追加）。追加表行为接近 Iceberg——文件写后不改。但即便追加表，Paimon 仍内置自动 Compaction（`BucketedAppendFileStoreWrite` / `AppendFileStoreWrite` 含合并逻辑，已核对存在），而 Iceberg 的追加写不在写路径内自动合并（见 §6）。小文件治理细节见 11 号文档。

**陷阱速记**：
- HASH_FIXED 桶数定死，数据涨超预期会让单桶过大、Compaction 变慢——预估不准用 `POSTPONE_MODE`。
- 动态桶的全局索引在亿级主键时可达 GB 级，维护成本计入写入开销。
- Level-0 堆积（Compaction 跟不上写入）→ 读放大飙升，靠反压与参数调优控制（§6）。

---

## 4. 元数据管理：base/delta 分离 vs 统一 manifest list

**① 要解决什么问题**

元数据设计的分水岭是"流式增量读取要不要做成一等场景"。Paimon 把"流式消费者只关心增量"做进了元数据（base/delta 分离）；Iceberg 把"批查询要看到某时刻完整状态"做成统一 manifest list。

**② 元数据层级对照**

```
Paimon 元数据层级                              Iceberg 元数据层级
┌─────────────────┐                            ┌──────────────────────┐
│   snapshot-N    │ (JSON, version=3)          │   metadata.json      │ (JSON)
│   ├── baseManifestList   全量基线            │   (format-version v1/v2/v3)
│   ├── deltaManifestList  增量变更            │   ├── schemas[] / partition-specs[]
│   ├── changelogManifestList (可选)           │   ├── snapshots[] → snap-N.avro
│   └── indexManifest                          │   └── refs{} (tag/branch)
└────────┬────────┘                            └──────────┬───────────┘
┌────────▼────────┐                            ┌──────────▼───────────┐
│  ManifestList   │ (Avro)                      │   manifest-list.avro │ (Avro)
│  └── ManifestFileMeta[]                       │   └── ManifestFile[]
└────────┬────────┘                            └──────────┬───────────┘
┌────────▼────────┐                            ┌──────────▼───────────┐
│  ManifestFile   │ (Avro)                      │   manifest-N.avro    │ (Avro)
│  └── ManifestEntry[]                          │   └── ManifestEntry[]
│      _KIND(ADD/DELETE)/_PARTITION/_BUCKET/_FILE │   status(ADDED/DELETED/EXISTING)/data_file
└─────────────────┘                            └──────────────────────┘
```

**③ Paimon 为什么做 base/delta 分离**

`Snapshot.java:54-59`（**已修正：旧稿凭记忆给的字段名与行号未核对，现以源码 `FIELD_*` 常量为准**）定义了三个 manifest list 字段：`baseManifestList`（全量基线）、`deltaManifestList`（增量变更）、`changelogManifestList`（可选 changelog）。设计收益：

1. **增量读取从 O(全量文件) 降到 O(增量文件)**：流式消费者只读 delta 即可发现新增/删除文件，无需扫全量 base。
2. **快照过期更快**：过期时主要处理 delta 引用的文件。
3. **manifest 膨胀可控**：base 可独立做全量合并，对应 `MANIFEST_FULL_COMPACTION_FILE_SIZE` 配置项（`CoreOptions.java:457`，已核对存在）。

Iceberg 不分 base/delta，而在 manifest entry 上打 `status`（ADDED/DELETED/EXISTING），增量读取需 diff 两个 Snapshot 的 manifest list。

**④ 取舍对照**

| 维度 | Paimon (base/delta 分离) | Iceberg (统一 manifest list) |
|------|--------------------------|------------------------------|
| 增量读取效率 | O(delta)，快 | O(全量 manifest)，需 diff |
| 全量读取 | 合并 base + delta | 直接读一个 manifest list |
| 元数据复杂度 | 更高（两套） | 更低（单一抽象） |
| 快照过期 | 更高效（主处理 delta） | 需扫全量 manifest |
| manifest 治理 | 提交时自动触发合并 | 需手动 `RewriteManifests`（见 §6） |

**⑤ 快照格式差异**

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| Snapshot 表达 | 单个 JSON 文件（`Snapshot` version=3，`Snapshot.java:49`） | `metadata.json`（JSON）总入口 + 每快照一个 `snap-{id}-{uuid}.avro` |
| 入口提示 | `snapshot/` 下 LATEST/EARLIEST hint 文件 | `version-hint.text` 或 catalog 指针 |
| 版本号 | Snapshot 自身 version 字段（当前 3） | TableMetadata format-version（v1/v2/v3，v4 在设计中） |
| Schema 管理 | 独立 SchemaManager，Schema 存为独立 JSON | Schema 内嵌 `metadata.json` |
| 额外元数据 | watermark、commitUser、commitIdentifier、statistics、nextRowId（`Snapshot.java:54-204` 已核对） | operation、summary map、（v3）row lineage 字段 |

Paimon Snapshot 用 JSON 而非 Avro 的实际好处：人类可读、`cat` 即可排障；代价是文件略大于 Avro 二进制。**注意常见说法纠偏**：Iceberg 的"元数据入口" `metadata.json` 本身是 JSON，只是 manifest-list 与 manifest 用 Avro，不能笼统说"Iceberg Snapshot 是 Avro"（旧稿 §3.3 有此含糊，已修正）。快照与时间旅行的完整机制见 17 号文档。

---

## 5. 主键更新：MergeFunction vs Delete File / DV

这是两者**最根本的差异点**，也是选型的第一判据。

**① 要解决什么问题**

同一主键来自多源、被高频更新，如何低成本写入、又能查到正确的合并值？Paimon 的答案是把"合并语义"下推到存储层（MergeFunction）；Iceberg 的答案是"先删后加"——写 Delete/DV 标记旧行 + 写新数据行。

**② Paimon 的 MergeFunction 体系**

4 种合并引擎（`CoreOptions.MergeEngine`，`CoreOptions.java:3910`，已核对）：

| 引擎 (config value) | 实现类 | 行为 | 适用 |
|---------------------|--------|------|------|
| `deduplicate`（默认） | `DeduplicateMergeFunction` | 保留最后一条 | 通用 Upsert |
| `partial-update` | `PartialUpdateMergeFunction` | 合并非 null 字段 | 多源写同一行不同列 |
| `aggregation` | `AggregateMergeFunction` | 按字段聚合 | 实时指标 |
| `first-row` | `FirstRowMergeFunction` | 保留第一条 | 去重保留最早 |

字段聚合器（`FieldAggregator` 子类，已逐一核对存在，约 20+ 种）：`FieldSumAgg`、`FieldMinAgg`、`FieldMaxAgg`、`FieldProductAgg`、`FieldPrimaryKeyAgg`、`FieldFirstValueAgg`、`FieldLastValueAgg`、`FieldFirstNonNullValueAgg`、`FieldLastNonNullValueAgg`、`FieldListaggAgg`、`FieldCollectAgg`、`FieldBoolAndAgg`、`FieldBoolOrAgg`、`FieldMergeMapAgg`、`FieldMergeMapWithKeyTimeAgg`、`FieldRoaringBitmap32Agg`、`FieldRoaringBitmap64Agg`、`FieldNestedUpdateAgg`、`FieldNestedPartialUpdateAgg`、`FieldHllSketchAgg`、`FieldThetaSketchAgg`、`FieldIgnoreRetractAgg`。

**`LookupMergeFunction`** 是特殊包装器：写入时通过 Lookup 查旧值与新值合并，是 `changelog.producer=lookup` 的基础（与 §7 衔接）。PartialUpdate/Aggregation 的语义细节见 08 号文档，Lookup/LSM 点查见 01/13。

**③ Iceberg 的删除/更新机制**

Iceberg 更新本质是"先删后加"，删除有三类（注意 v2/v3 之别）：

| 方式 | 版本 | 性能 | 适用 |
|------|------|------|------|
| Position Delete | v2 | 按位跳过，较快 | 已知行位置（v3 起不再新增此类文件，由 DV 取代） |
| Equality Delete | v2/v3 | 逐行匹配删除条件，最慢 | 不知行位置（如纯 SQL DELETE、部分 CDC） |
| Deletion Vector (DV) | v3 | Puffin 二进制位图过滤，快 | v3 删除/更新的主路径 |

**关键纠偏（已修正旧稿）**：DV 不是"Position Delete 的优化版"那么简单——在 v3 规范里 DV **替代**了 Position Delete（v3 表不再新增 Position Delete 文件），并与 Delta Lake 的 DV 采用兼容的二进制编码。Iceberg 的 DV 与 Paimon 的 Deletion Vector 思路一致（位图标记删除行，merge-on-read 时过滤），Paimon DV 机制详见 04 号文档。

**④ 性能差异根因（核心对比）**

写入：

| 场景 | Paimon | Iceberg |
|------|--------|---------|
| 单条 Upsert | 入内存 buffer，O(1)，无需读旧值 | 需定位旧行写 Delete/DV + 写新数据 |
| 高频小批量更新 | 内存吸收，异步 Compaction 合并 | 易产生大量小 Delete/数据文件，碎片化（v3 DV 缓解但不消除） |

读取：

| 场景 | Paimon | Iceberg |
|------|--------|---------|
| 全表扫描（已 Compact） | 读 Level-max 即可 | 直接读数据文件 |
| 全表扫描（未压实/未维护） | Merge-on-Read 归并多层 | 关联应用 Delete/DV |
| 主键点查 | Lookup + Hash 索引，接近 O(1) | 无专门点查优化，扫数据文件 + 应用删除 |

**根因一句话**：Paimon 写入时就把数据排序组织好了，后续合并是高效归并；Iceberg 的删除是"事后补丁"，v2 下补丁累积越多读越慢，v3 用 DV 把"补丁"变成紧凑位图，显著改善但仍需维护任务定期合并（§6）。

**⑤ 各自适用场景**

- Paimon 更适合：高频更新（CDC、实时聚合）、需要复杂合并语义（部分更新、聚合）、流式更新 + 实时查询。
- Iceberg 更适合：低频更新（每天几次批量）、追加为主的日志类、跨引擎通用分析；若用 v3 DV，更新性能短板已明显收窄。

**陷阱速记**：
- Paimon Partial Update 同字段并发写是 Last-Write-Wins，需 `sequence.field` 控顺序。
- Paimon 聚合器有类型约束（`sum` 仅数值、`listagg` 仅字符串），配错运行时报错。
- Paimon `collect`/`listagg` 会累积历史值致单行膨胀，需配 TTL。
- Iceberg v2 Equality Delete 无法用索引、近似全表扫描；高频更新务必上 v3 DV 或勤跑维护。

---

## 6. Compaction：内置自动 vs 外部/原生维护任务

**① 要解决什么问题**

两者都会产生小文件/删除标记累积，都需要合并治理。差异在**触发时机与职责归属**：Paimon 把 Compaction 做进写入路径自动触发并带反压；Iceberg 把它做成独立维护任务（早期只能 Spark Action，现已可在 Flink 内原生运行）。

**② Paimon 的内置 Universal Compaction**

`UniversalCompaction.java:42`（已核对）实现 RocksDB 风格策略，选择优先级（文字流程）：
EarlyFullCompaction（定时全量）→ pickForSizeAmp（空间放大超阈）→ pickForSizeRatio（相邻 run 大小比例失衡）→ pickForFileNum（run 数超阈）。

关键参数（已核对）：

| 参数 | 含义 | 默认 |
|------|------|------|
| `maxSizeAmp`（`compaction.max-size-amplification-percent`） | 最大空间放大百分比 | 200% |
| `sizeRatio`（`compaction.size-ratio`） | 触发合并的大小比例 | 1 |
| `num-sorted-run.compaction-trigger` | 触发 compaction 的 SortedRun 数 | 5（`CoreOptions.java:756`） |
| `num-sorted-run.stop-trigger` | 阻塞写入（反压）的 SortedRun 数 | compaction-trigger **+ 3** = 8（`CoreOptions.java:764,3068`，**已修正：旧稿 §5.1 误写为 +1**） |

执行方式：Compaction 在独立 `ExecutorService` 异步跑（`MergeTreeCompactManager`），写入线程在 SortedRun 超过 stop-trigger 时被阻塞（反压）。额外策略：`ForceUpLevel0`、off-peak hours、`EarlyFullCompaction`。Compaction 全链路深度展开见 23 号文档，反压细节见 01。

**③ Iceberg 的维护任务（现状已变，重点纠偏）**

旧资料常说"Iceberg 不内置 Compaction，必须自己搭 Airflow"。现状是：

- **Spark Actions**：`RewriteDataFiles`、`RewriteManifests`、`RemoveDanglingDeleteFiles`、`RewritePositionDeleteFiles` 等仍是常用维护手段。
- **Flink TableMaintenance API**：自 Iceberg 1.7 起，flink-iceberg-runtime 内置维护任务（compaction / expire snapshots / orphan 清理），可嵌入流式作业或作为独立 Flink 作业运行，**不再强依赖 Spark 集群**；1.10 进一步统一了 Flink/Spark 的 compaction 逻辑并新增 `max-files-to-rewrite` 等控制项。协调用 `TriggerLockFactory`（JDBC/ZooKeeper 锁）防并发维护冲突。

所以准确的差异不是"有没有 Compaction"，而是**触发模型**：Paimon 在写入路径内按策略自动触发 + 反压保护；Iceberg 需要由维护任务（定时/独立作业）触发，写路径本身不自动合并。

**④ 自动化与运维成本对照**

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 触发方式 | 写入时自动触发，内置策略 | 维护任务触发（Spark Action 或 Flink TableMaintenance） |
| 写路径内自动合并 | 是 | 否 |
| 反压机制 | 内置（SortedRun 过多阻塞写入） | 无（删除标记/小文件可累积，靠维护频率兜底） |
| Manifest Compaction | 提交时自动触发 | `RewriteManifests`（手动/任务） |
| 是否依赖 Spark 集群做维护 | 否 | 不再强制（Flink 1.7+ 可原生维护） |
| 运维心智 | 调好 LSM 参数即可，但要懂 LSM | 配维护任务调度 + 触发条件 + 分布式锁 |

**⑤ 取舍**

- Paimon 内置 Compaction：用更高的引擎复杂度（策略/反压/内存抢占）和写入端 CPU/内存占用，换"无需搭调度系统"与实时性；代价是 LSM 参数调优是硬运维，配错会写阻塞或读放大。
- Iceberg 维护任务：用维护调度的运维成本，换引擎无关与灵活（可挑非高峰跑、可选引擎）；v3 DV + Flink 原生维护后，这条路径的实时性与易用性已明显提升。

**陷阱速记**：
- Paimon `num-sorted-run.stop-trigger` 过小 → 频繁写阻塞；过大 → Level-0 堆积读放大。
- Paimon Compaction 结果随 Checkpoint 提交，Checkpoint 频繁失败会让 Compaction 做无用功。
- Iceberg 流式写入最常见事故是"只部署了写、没部署维护任务"——几天内查询性能就退化。
- Iceberg v2 高频更新不跑 `RewriteDataFiles`/不上 DV，Delete File 累积致读性能雪崩。

---

## 7. 流式与 CDC：原生 Changelog vs 增量扫描

**① 要解决什么问题**

下游（ES/MySQL/实时大屏）需要完整 CDC 流（INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE）才能正确处理更新；增量消费需要"只读新数据"避免全量扫描。这是 Paimon 与 Iceberg 差距最大的维度。

**② Paimon 的原生 Changelog**

四种 Changelog 模式（`CoreOptions.ChangelogProducer`，`CoreOptions.java:4051`，已核对）：

| 模式 | 机制 | 延迟/开销 | 适用 |
|------|------|-----------|------|
| `NONE` | 不产 changelog | 无开销 | 只批量查询 |
| `INPUT` | flush 时双写 changelog（直接用输入流） | 低 | 输入本身就是完整 CDC（如 Binlog） |
| `FULL_COMPACTION` | 全量 compaction 时对比新旧值产出 | 延迟随 compaction 间隔 | 对延迟不敏感但要 changelog |
| `LOOKUP` | 写入时 Lookup 旧值产出完整 changelog | 开销高、延迟低 | 需低延迟完整 CDC |

增量发现机制：流式读取靠 `deltaManifestList` 快速发现新文件（`ContinuousFileSplitEnumerator` 周期扫新 Snapshot 的 delta manifest），只读增量、不扫全量元数据。Changelog 数据存于独立 `changelogManifestList`，与主数据分离，可有独立 TTL，消费不影响主数据读写。Changelog 产生机制全链路见 24 号文档，Flink 写入链路见 15。

**③ Iceberg 的增量读取**

- `IncrementalAppendScan`：只扫两个 Snapshot 间新增（APPEND）的数据文件，看不到更新/删除，适合追加表。
- `IncrementalChangelogScan`：扫描两 Snapshot 间变更，产出 `ChangelogScanTask`（含 INSERT/DELETE/UPDATE_BEFORE/UPDATE_AFTER），但底层需 diff manifest + 应用 Delete，性能不如原生 changelog。
- Snapshot 无 Watermark 字段，无法与 Flink 事件时间天然对齐。
- 流式写入侧：Iceberg 1.10 提供 Flink **Dynamic Sink**（自动 schema 演进、fan-out 多表），增量"写入"能力提升，但"读出完整 CDC"仍弱于 Paimon。

**④ 实时性对照**

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 最小可见延迟 | Flink Checkpoint 间隔（秒级） | 写入作业 Commit 间隔 |
| Changelog 完整性 | 原生完整 CDC 四类事件 | 靠 IncrementalChangelogScan 重建，依赖 Delete diff |
| Watermark | 原生内嵌 Snapshot | 无原生支持 |
| Consumer 进度管理 | 内置 `ConsumerManager`，记录每消费者位点 | 无内置，靠外部记录 |
| 流式背压 | 通过 SortedRun 控写入速度 | 无 |

**⑤ 根因**

Paimon Snapshot 的 `commitIdentifier` 保证有序（源码注释："snapshot A 的 commitIdentifier 小于 B，则 A 必先于 B 提交"），`deltaManifestList` 提供高效增量发现，`changelogManifestList` 提供完整变更——三者组合成完整流式消费协议。Iceberg Snapshot 记录的是"某时刻状态"而非"状态间变更"，要还原 CDC 需 diff + 应用 Delete，且 Delete 语义使"一次更新对应哪些行变化"变得需要额外计算。Tag/Branch/Consumer 详见 17 号文档。

**陷阱速记**：
- Paimon `LOOKUP` 模式缓存配小（`lookup.cache-max-memory`）→ 大量远程读拖慢写入。
- Paimon `FULL_COMPACTION` 模式若 compaction 间隔长 → 下游消费延迟高。
- Paimon Consumer 未正确提交位点 → 重启从头消费、重复处理。
- Iceberg `IncrementalAppendScan` 漏掉更新/删除；起始 Snapshot 被过期则增量读失败。

---

## 8. 引擎集成：Flink / Spark 谁更深

**① Flink 集成对照**

Paimon 与 Flink 是"亲生"关系（前身即 Flink Table Store）；Iceberg 的 Flink 集成在 2025 年后已大幅成熟（1.10 全面支持 Flink 2.0、Dynamic Sink、原生 TableMaintenance）。

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 写入提交内容 | `ManifestCommittable`（DataIncrement + CompactIncrement + IndexIncrement，一事务内提交数据与 compaction 结果） | DataFile + DeleteFile/DV 列表 |
| 原子性 | `FileStoreCommitImpl` 文件系统原子重命名 | Iceberg `Transaction` API |
| 冲突处理 | 内置 OCC（`ConflictDetection`）自动重试 | 内置 OCC 自动重试 |
| Savepoint Tag | `AutoTagForSavepointCommitterOperator` 自动打 Tag | 无内置 Savepoint Tag |
| Exactly-once | 靠 `commitIdentifier` 去重 | 靠 Snapshot 幂等 |
| 流式源 | `ContinuousFileStoreSource` 持续监听新 Snapshot（delta manifest） | IcebergSource 流模式周期扫新 Snapshot（monitor-interval） |
| 维护任务 | 写路径内自动 Compaction | Flink TableMaintenance（1.7+）原生跑 compaction/expire/orphan |
| Dynamic Sink | — | 1.10 提供（自动 schema 演进、fan-out 建表） |

关键差异：Paimon 的 Committer 把数据写入与 Compaction 结果**在同一事务提交**（`CommitMessageImpl` 含 `DataIncrement`/`CompactIncrement`），保证一致性；这也是其内置 Compaction 能与写入协同的基础。Flink 写入链路全展开见 15 号文档。

**② CDC 整库入湖（Paimon 的独特优势）**

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 内置 CDC 同步 | `paimon-flink-cdc` 提供完整管道（如 `FlinkCdcSyncDatabaseSinkBuilder`） | 无内置工具 |
| 支持源 | MySQL/Kafka/MongoDB 等 | 靠 Flink CDC Connector + Dynamic Sink 组合 |
| 整库同步 | 支持，自动建表 + Schema 演进 | 需逐表配置（Dynamic Sink 缓解部分） |

Paimon `paimon-flink-cdc` 可从 MySQL 等一站式整库同步到 Paimon（自动建表、Schema 演进、并行多表），是"流式湖仓"定位的直接体现；Iceberg 实现类似能力需组合多个组件，1.10 的 Dynamic Sink 缩小了差距但仍非一站式整库工具。CDC 机制主讲见 14 号文档。

**③ Spark 集成对照**

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 实现入口 | `SparkTable`（Scala），支持 `SupportsRowLevelOperations` | `SparkTable`，`SupportsRead`/`SupportsWrite` |
| Spark 版本 | 3.2–4.0（版本特定子模块） | 3.x–4.0（随版本演进） |
| 行级更新 | MERGE INTO（SQL Extensions）+ CoW | 原生 MERGE INTO，MoR/CoW 可选 |
| 流式读 | Structured Streaming 支持 | Structured Streaming 支持 |

Procedure 丰富度（Paimon 约 30 个 vs Iceberg 约 15–20 个）：Paimon 在 Branch 管理、Consumer 管理、文件索引、CDC 迁移上更全（如 `CreateBranchProcedure`/`ResetConsumerProcedure`/`RewriteFileIndexProcedure`/`MigrateTableProcedure`）；Iceberg 在 `rewrite_data_files`/`expire_snapshots`/`cherrypick_snapshot` 等通用维护上成熟。这与各自定位一致：Paimon 偏流式运维，Iceberg 偏批维护。Spark 集成细节见 06 号文档。

**④ 取舍**

- 以 Flink 为主力、需要整库 CDC 入湖 → Paimon 集成最深、开箱即用。
- 以 Spark 为主力、需要广引擎读 → Iceberg 更顺，且 Spark 行级更新（MERGE）更原生成熟。
- 两者的 Flink/Spark 都能用，差异在"哪种生态是你团队主场 + 是否需要流式 CDC 一站式"。

---

## 9. 索引能力：多种文件索引 + Lookup vs 列统计 + Puffin

**① 要解决什么问题**

在百万级文件的大表上，等值/范围/点查若无索引就退化成全表扫描。Paimon 的主键表内在需要高效点查（支撑 Lookup 合并、动态桶路由），所以索引体系更重；Iceberg 作为分析型格式主要靠列统计做粗粒度跳过。

**② Paimon 的索引体系（已核对实现类存在）**

4 种文件级索引（`fileindex` 包，已核对）：

| 索引 | 实现类 | 原理 | 适用 |
|------|--------|------|------|
| Bloom Filter | `BloomFilterFileIndex` | 概率型，判存在性 | 等值过滤 |
| Bitmap | `BitmapFileIndex` | 每值一位图，精确 | 低基数列等值/IN |
| BSI | `BitSliceIndexBitmapFileIndex` | 位切片索引 | 数值范围查询 |
| Range Bitmap | `RangeBitmapFileIndex` | 范围位图 | 数值区间过滤 |

文件索引**嵌入数据文件**（`FileIndexFormat` 定义序列化格式），查询时一个文件读完数据 + 索引，减少 I/O、生命周期与数据一致、写入原子。

全局索引：`HashIndexFile` + `DynamicBucketIndexMaintainer` 维护主键→桶映射，服务于动态桶路由。

Lookup 点查：`LookupLevels` 把高层文件转本地 KV 索引实现接近 O(1) 点查，支持 Bloom 加速（`LOOKUP_CACHE_BLOOM_FILTER_ENABLED`）与远程 lookup 文件（`LOOKUP_REMOTE_FILE_ENABLED`）。索引机制深度展开见 13 号文档，Lookup 与 LSM 点查见 01。

**③ Iceberg 的索引能力**

| 能力 | 出处 | 原理 |
|------|------|------|
| Partition Summary | manifest 的 `partitions` 字段 | 记录每 manifest 覆盖的分区范围 |
| 列统计 | manifest entry 的 `lower_bounds`/`upper_bounds`/`null_value_counts` 等 | 每文件 min/max/null_count |
| Bloom Filter | Parquet 内 / Puffin（v3） | 需配置；v3 起 Puffin 可存更多索引类型 |

**④ 对比**

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 文件索引种类 | 4 种 + 全局索引 + Lookup | 列统计 + Partition Summary + Bloom（需配置） |
| 等值过滤 | Bloom + Bitmap + Hash Lookup | 列统计 + Bloom（v3） |
| 范围过滤 | BSI + Range Bitmap | 列统计 min/max |
| 点查 | 接近 O(1) Lookup | 无专门点查优化 |
| 索引存储 | 嵌入数据文件 | 列统计在 manifest；Bloom 在 Parquet/Puffin |
| 全局索引 | 支持（动态桶） | 不支持 |

**根因**：Paimon 主键表需要高效点查支撑 Lookup 合并与动态桶路由，这是 LSM 模型内在要求；Iceberg 偏分析、靠列统计粗过滤即可满足多数批查询，对细粒度索引需求弱。

**取舍**：Paimon 索引丰富换来查询性能但增加存储/维护开销；Iceberg 索引简单换来引擎无关与低复杂度，代价是等值/点查无法精确过滤。

---

## 10. 快照、时间旅行与版本管理

> 小文件与过期治理的自动化程度，本质就是 §6 Compaction 触发模型的延伸，这里只给一张对照表（治理机制主讲见 11 号文档，不重复展开）：
>
> | 治理项 | Paimon | Iceberg |
> |--------|--------|---------|
> | 小文件合并 | 写入路径自动（异步 Compaction + 反压） | 维护任务触发（`RewriteDataFiles` / Flink TableMaintenance） |
> | Manifest 合并 | 提交时自动（`FileStoreCommitImpl`） | `RewriteManifests`（手动/任务） |
> | 过期 Snapshot 清理 | 可配置自动过期 | `ExpireSnapshots`（任务） |
> | 孤儿文件清理 | `RemoveOrphanFilesProcedure` | `RemoveOrphanFiles` Action |
> | Delete/DV 治理 | DV 在 compaction 时合并 | `RemoveDanglingDeleteFiles` / `RewritePositionDeleteFiles` |
>
> 一句话：Paimon 把"小文件不失控"作为写入路径的硬约束（反压）；Iceberg 把治理交给维护任务，灵活但需保证调度不缺位。

下面进入版本管理本体。两者都支持时间旅行、Tag、Branch，差异在**流式语义**（Paimon 强）与**分支语义成熟度**（Iceberg 强）。

**① Paimon 的 Tag / Branch / Consumer**

- **Tag**（`Tag` 继承 `Snapshot`）：命名的快照；支持按时间周期**自动创建**（`TagAutoCreation`）、按时间**自动过期**（`TagTimeExpire`）；被 Tag 引用的 Snapshot 及其数据不会被过期清理（锚点）。
- **Branch**（`BranchManager`，含 `FileSystemBranchManager` / `CatalogBranchManager`）：从 Tag/Snapshot 创建独立 Snapshot 序列，支持 Fast-Forward 合回主干。
- **Consumer**（`ConsumerManager`）：记录每个流式消费者位点（`nextSnapshot`），被引用的 Snapshot 不被清理；支持重置/清理。

**② Iceberg 的 SnapshotRef（统一 Tag/Branch）**

- `SnapshotRef` 统一管理：`TAG`（不可变引用）、`BRANCH`（可变、可独立演进），都记在 `metadata.json` 的 `refs{}`。
- `ManageSnapshots` API：`createBranch`/`createTag`/`rollbackTo`/`cherrypick`/`setMaxSnapshotAgeMs`/`setMinSnapshotsToKeep`。
- v3 还引入 row lineage（`_row_id` / `_last_updated_sequence_number`），用于增量识别变更行。

**③ 对比**

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| Tag 自动创建 | 支持（`TagAutoCreation`，按小时/天） | 不支持（手动/外部触发） |
| Tag 自动过期 | 支持（`TagTimeExpire`） | 支持（`maxRefAgeMs`） |
| Branch 独立性 | 独立 Snapshot 序列 | 共享 TableMetadata（refs 统一管理） |
| Consumer 跟踪 | 内置 `ConsumerManager` | 无内置 |
| 引用锚定 | Tag/Consumer 引用的 Snapshot 自动保留 | 需配 min-snapshots-to-keep |
| Cherry-pick | 不支持 | 支持 |
| 回滚 | Snapshot / Timestamp / Watermark | Snapshot / Timestamp |
| 时间旅行 SQL（Spark） | `VERSION AS OF 1` / `TIMESTAMP AS OF` / `VERSION AS OF 'tag'` | `TIMESTAMP AS OF` / `t.tag_xxx` |

**④ 取舍**

- Paimon 独特能力：自动打 Tag（生产很实用）、Consumer 进度管理（流式刚需）、Watermark 回滚（流式独有，从 Snapshot 的 `watermark` 字段回滚）。
- Iceberg 独特能力：cherry-pick、refs 统一管理的成熟分支语义、v3 row lineage。

快照与时间旅行的完整机制（含 Tag/Branch/Consumer 锚定与过期）见 17 号文档。

---

## 11. 生态与成熟度：Catalog / 引擎 / 云

这是 **Iceberg 的主场**。Iceberg 作为事实标准之一，引擎与 Catalog 生态覆盖最广；Paimon 深耕 Flink/Spark 与国内云，并通过 `paimon-iceberg` 借力 Iceberg 生态。

### 11.1 Catalog 支持

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

### 11.2 计算引擎支持

| 引擎 | Paimon | Iceberg |
|------|--------|---------|
| Flink | 原生深度集成（1.16-2.2） | 1.10 起全面支持（含 Flink 2.0、Dynamic Sink、原生 TableMaintenance） |
| Spark | DataSource V2（3.2-4.0） | 原生深度集成（随 Spark 版本演进至 4.0） |
| Hive | SerDe 连接器（2.1-3.1） | SerDe 连接器 |
| Trino/Presto | 连接器 | 原生连接器 |
| Doris / StarRocks | 连接器 | 原生连接器 |
| Impala / Dremio / DuckDB | 不支持/有限 | 原生连接器 |
| Snowflake / BigQuery | 不支持/有限 | 原生支持（External / Managed Table） |

**关键差异**：Iceberg 的引擎覆盖面远大于 Paimon——几乎所有主流分析引擎都内置 Iceberg 支持；Paimon 集中在 Flink/Spark（其中 Flink 集成最深）。注意 Iceberg 的 Flink 集成在 2025 年后已非"社区轻量集成"而是一等支持（已修正旧稿"社区集成"的轻描淡写）。

### 11.3 云厂商支持

| 云厂商 | Paimon | Iceberg |
|--------|--------|---------|
| AWS | S3 FileSystem | 原生（EMR、Glue、Athena、S3 Tables） |
| Azure | Azure Blob | 原生（Synapse、Databricks） |
| GCP | GCS | 原生（BigQuery、Dataproc） |
| 阿里云 | 深度支持（OSS、EMR、Hologres） | 基本支持 |
| 华为云/腾讯云 | OBS / COS 支持 | 基本支持 |

**关键差异**：国际公有云（AWS/Azure/GCP）Iceberg 原生集成度更高；国内云（尤其阿里云）Paimon 支持更深入。

### 11.4 Paimon 的 Iceberg 兼容模块（混合架构的关键）

`paimon-iceberg` 模块（已核对：`IcebergRestMetadataCommitter`、`IcebergHiveMetadataCommitter`、核心 `IcebergMetadataCommitter`）可把 Paimon 表的元数据以 Iceberg 协议（REST / Hive）对外暴露。意义：**用 Paimon 写入（高性能流式 + 内置 Compaction），用 Iceberg 兼容元数据让下游广引擎读取**，在一定程度上兼取两者之长（见 §12 混合架构）。这也直接反驳了"Paimon 锁死单引擎"的误区。

### 11.5 社区与成熟度

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| 项目阶段 | Apache 顶级项目，较新，迭代快 | 成熟，事实标准之一，规范稳定（v3 已落地） |
| 标杆用户 | 阿里巴巴等（前身 Flink Table Store） | Netflix（发源）、Apple、Adobe 等 PB 级数据湖 |
| 规范开放度 | 自有格式，跨格式互通靠兼容模块 | 开放规范 + REST 标准，被 Polaris/Unity/Gravitino 等广泛采纳 |

成熟度上 Iceberg 占优；流式实时方向 Paimon 迭代更聚焦。

---

## 12. 选型决策总结

### 12.1 选 Paimon 的场景

| 场景 | 原因 |
|------|------|
| 实时数据入湖 | LSM 高写入吞吐 + 秒级可见 |
| CDC 整库同步 + 增量 | `paimon-flink-cdc` 一站式 + 原生 Changelog（独有） |
| 高频 Upsert（秒/分钟级） | 追加新版本 + 异步合并，远优于反复写 Delete |
| 实时聚合 / 部分更新 | Aggregation / PartialUpdate 合并引擎（独有） |
| Flink 为主力 | 集成最深 |
| 流式消费下游 | 原生 Changelog + Consumer 进度管理 |
| 主键点查多 | Lookup + 文件索引接近 O(1) |

### 12.2 选 Iceberg 的场景

| 场景 | 原因 |
|------|------|
| 多引擎混合查询 | 引擎/Catalog 生态最广 |
| 批分析为主的数据仓库 | 不可变文件利于列存优化 |
| 国际公有云环境 | AWS/Azure/GCP 原生集成最高 |
| 数据湖标准化 | REST Catalog 被 Polaris/Unity/Gravitino 广泛采纳 |
| 低频更新、高频批查询 | 无写路径内 Compaction 开销，查询路径简洁；v3 DV 让更新性能短板收窄 |
| 已有 Iceberg 生态投资 | 迁移成本低 |

### 12.3 混合架构：Paimon 写 + Iceberg 读

```
           实时写入                          广引擎批量读
  CDC源 ─→ Flink ─→ Paimon ─→ (paimon-iceberg 暴露 Iceberg 元数据) ─→ Trino / Snowflake / Dremio …
                      │
                      ├─ Flink 流式消费（原生 Changelog）
                      └─ Spark 交互式查询
```

适用于"既要实时写 + 流式消费、又要广引擎批量读"的场景：写入侧吃 Paimon 的流式性能与内置 Compaction，读取侧借 `paimon-iceberg` 的 Iceberg 兼容元数据对接广泛分析引擎。注意这是元数据兼容层，并非把 Paimon 数据完全转成原生 Iceberg，具体能力与一致性需视版本与下游引擎验证。

### 12.4 设计决策总结表

| 决策点 | Paimon 的选择 | Iceberg 的选择 | 取舍 / 代价 | 谁的收益更大 |
|--------|---------------|----------------|-------------|--------------|
| 更新模型 | LSM 追加新版本 + 异步合并 | 不可变文件 + Delete/DV 标记 | Paimon 写快但读需归并；Iceberg 写避免重写但 v2 删除累积拖读（v3 DV 缓解） | 高频更新 → Paimon |
| 数据组织 | Bucket + Level 分层 | Partition + 扁平文件 | Paimon 限定 merge 范围但需规划桶数；Iceberg 简单但无桶级隔离 | 主键场景 → Paimon |
| 元数据 | base/delta 分离（JSON Snapshot） | 统一 manifest list（JSON 入口 + Avro 链） | Paimon 增量读 O(delta) 但维护两套；Iceberg 简洁但增量需 diff | 流式增量 → Paimon |
| Compaction | 写路径内自动 + 反压 | 维护任务触发（Spark/Flink TableMaintenance） | Paimon 省调度但要懂 LSM；Iceberg 灵活但需保证调度不缺位 | 省运维 → Paimon；灵活 → Iceberg |
| Changelog/CDC | 原生四档 + Consumer | 增量扫描重建，无原生完整 CDC | Paimon 复杂度高但流式完整；Iceberg 简洁但流式弱 | 流式 → Paimon |
| 索引 | Bloom/Bitmap/BSI/RangeBitmap + Lookup | 列统计 + Puffin Bloom | Paimon 强但增存储；Iceberg 简单但点查弱 | 点查/细过滤 → Paimon |
| 版本管理 | Tag/Branch/Consumer，自动打 Tag、Watermark 回滚 | SnapshotRef 统一，cherry-pick、row lineage(v3) | 各有独特能力 | 流式语义 → Paimon；分支成熟度 → Iceberg |
| 生态 | Flink/Spark + 国内云 + iceberg 兼容层 | 全引擎 + 全云 + REST 标准 | Paimon 聚焦；Iceberg 广 | 生态广度 → Iceberg |

### 12.5 一句话收束

Paimon 把**流式处理的理念深植进存储引擎**（LSM + MergeFunction + 原生 Changelog + 内置 Compaction），是实时湖仓的最佳选择，代价是更高的系统复杂度与相对有限的引擎生态。Iceberg 把**引擎无关的开放表格式**做成了事实标准（不可变文件 + 丰富 Catalog/引擎生态 + 成熟 Schema/Partition Evolution），是批分析与多引擎环境的首选，v3（DV + row lineage）已显著补强其更新与增量能力。二者并非简单竞品，而是面向不同核心场景的互补方案——很多企业的实际答案是"Paimon 写、Iceberg 读"的组合。

**选型一句话**：高频更新 / 流式 CDC / 流式消费 → Paimon；多引擎批分析 / 广生态 / 低频更新 → Iceberg；两者都要 → 混合架构。
