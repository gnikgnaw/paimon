# Apache Paimon Changelog 机制全链路分析

> **版本**：1.5-SNAPSHOT　**源码模块**：`paimon-core`（配置/枚举/Snapshot 定义在 `paimon-api`）　**核对日期**：2026-06

**一句话定位**：Changelog 是 Paimon 把"只能读到合并后最终值"的 LSM 表，变成"能逐行回放 `+I/-U/+U/-D` 变更"的 CDC 源的机制——它把"推导变更"这件事从读端前移到写端/压缩端预计算，让流式消费零额外开销。

读完本文你应能回答：① 为什么 LSM 表天生"丢失变更语义"，Paimon 用什么补回来；② `none/input/full-compaction/lookup` 四种 producer 各自在哪个时机、用什么算法产生 changelog，代价分别是什么；③ `full-compaction` 与 `lookup` 都靠"比对旧值与新值"出 changelog，凭什么 lookup 延迟更低；④ 为什么只有"含 Level 0 记录"的合并才产 changelog，高层 compaction 为什么不产；⑤ `changelog-producer.row-deduplicate` 在解决什么、配错会怎样；⑥ changelog 文件如何与数据文件分离存储、`changelogManifestList` 为何可为 null；⑦ 流式消费时 `ChangelogFollowUpScanner` 如何选 snapshot、`ScanMode.CHANGELOG` 与 `DELTA` 的真实分工；⑧ changelog 过期为什么要和 snapshot 解耦，Consumer 位点如何保护未消费的 changelog。

> 阅读约定：本文每个机制按"① 要解决什么问题 → ② 设计原理与取舍 → ③ 关键源码（精选片段 + `路径:行号`）→ ④ 风险/陷阱/边界 → ⑤ 收益与代价"组织。源码行号以本次核对为准；与旧稿不符处用 `（已修正）` 标注。**本篇是「Changelog 产生机制」主讲文档**；增量读取/Consumer 消费位点的完整展开见 17 号文档，本篇点到即引用。Compaction/LSM 基础见 01、23 号。

---

## 目录

- [1. 快速理解（核心问题 / 概念 / 陷阱）](#1-快速理解核心问题--概念--陷阱)
  - [1.1 核心问题：LSM 的 Merge 吞噬了变更语义](#11-核心问题lsm-的-merge-吞噬了变更语义)
  - [1.2 核心概念速查表](#12-核心概念速查表)
  - [1.3 高频生产陷阱](#13-高频生产陷阱)
- [2. ChangelogProducer 四种模式总览](#2-changelogproducer-四种模式总览)
  - [2.1 枚举定义与配置入口](#21-枚举定义与配置入口)
  - [2.2 四种模式对比矩阵](#22-四种模式对比矩阵)
  - [2.3 一条变更如何流过四种模式（产生时机串讲）](#23-一条变更如何流过四种模式产生时机串讲)
  - [2.4 模式选择决策树](#24-模式选择决策树)
- [3. NONE 模式 — 不产生 Changelog](#3-none-模式--不产生-changelog)
- [4. INPUT 模式 — flush 时双写输入](#4-input-模式--flush-时双写输入)
  - [4.1 flushWriteBuffer 的双写实现](#41-flushwritebuffer-的双写实现)
  - [4.2 changelog 文件的独立存储与归宿](#42-changelog-文件的独立存储与归宿)
  - [4.3 风险与收益](#43-风险与收益)
- [5. FULL_COMPACTION 模式 — 全压缩比对旧值](#5-full_compaction-模式--全压缩比对旧值)
  - [5.1 topLevelKv 与 merged 的比对算法](#51-toplevelkv-与-merged-的比对算法)
  - [5.2 Changelog 产生决策矩阵](#52-changelog-产生决策矩阵)
  - [5.3 full-compaction.delta-commits 控制频率](#53-full-compactiondelta-commits-控制频率)
  - [5.4 风险与收益](#54-风险与收益)
- [6. LOOKUP 模式 — 查找式低延迟 Changelog](#6-lookup-模式--查找式低延迟-changelog)
  - [6.1 LookupChangelogMergeFunctionWrapper 核心逻辑](#61-lookupchangelogmergefunctionwrapper-核心逻辑)
  - [6.2 "只在含 Level 0 时产 changelog" 的不变式](#62-只在含-level-0-时产-changelog-的不变式)
  - [6.3 LookupStrategy 与 Deletion Vector 协同](#63-lookupstrategy-与-deletion-vector-协同)
  - [6.4 为什么延迟低于 FULL_COMPACTION](#64-为什么延迟低于-full_compaction)
  - [6.5 风险与收益](#65-风险与收益)
- [7. FirstRow Changelog — 首行去重的特例](#7-firstrow-changelog--首行去重的特例)
- [8. Changelog 的存储架构](#8-changelog-的存储架构)
  - [8.1 Snapshot 的 changelogManifestList 字段](#81-snapshot-的-changelogmanifestlist-字段)
  - [8.2 文件分离存储与两条产生路径](#82-文件分离存储与两条产生路径)
  - [8.3 Long-Lived Changelog 与 ChangelogManager](#83-long-lived-changelog-与-changelogmanager)
- [9. Changelog 的流式消费机制](#9-changelog-的流式消费机制)
  - [9.1 FollowUpScanner 体系与正确的扫描器映射](#91-followupscanner-体系与正确的扫描器映射)
  - [9.2 ChangelogFollowUpScanner：跳过无 changelog 的 snapshot](#92-changelogfollowupscanner跳过无-changelog-的-snapshot)
  - [9.3 ScanMode 三态与 readChanges 的真实分工](#93-scanmode-三态与-readchanges-的真实分工)
- [10. AuditLog 和 Binlog 系统表](#10-auditlog-和-binlog-系统表)
- [11. Changelog 过期机制](#11-changelog-过期机制)
  - [11.1 changelog 专属过期配置与降级](#111-changelog-专属过期配置与降级)
  - [11.2 changelogDecoupled 解耦设计](#112-changelogdecoupled-解耦设计)
  - [11.3 ChangelogDeletion 的两条清理路径](#113-changelogdeletion-的两条清理路径)
- [12. Consumer 位点与 Changelog 过期协同](#12-consumer-位点与-changelog-过期协同)
- [13. 增量读取（incremental-between）](#13-增量读取incremental-between)
- [14. 与 Iceberg CDC 能力的对比](#14-与-iceberg-cdc-能力的对比)
- [15. 设计决策总结](#15-设计决策总结)

---

## 1. 快速理解（核心问题 / 概念 / 陷阱）

### 1.1 核心问题：LSM 的 Merge 吞噬了变更语义

**① 要解决什么问题**

主键表用 LSM Merge-Tree 存储：同一主键的多次写入分散在不同 Level 的文件里，读取时由 `MergeFunction` 现场合并出"最新值"（详见 01 §3.4、08 号 Merge 引擎）。这套机制对"查最新状态"很高效，但它**只保留终态、丢弃了过程**——下游拿到合并结果时，无法分辨这一行是新插入、是更新（旧值是什么）、还是已被删除。

后果是致命的：① 流式消费者只能看到快照终态，做不了实时 CDC；② Paimon 表无法作为一等的变更数据源接入 Flink CDC 生态；③ 下游被迫自己维护全量历史状态去 diff，把存储引擎该干的活推给了每一个消费者。

**② 设计原理与取舍：预计算 vs 读时推导**

核心抉择是"在哪一端付出推导变更的成本"。

| 方案 | 变更在何时算出 | 读端开销 | 写端开销 | Paimon 取舍 |
|------|--------------|---------|---------|------------|
| 读时推导（Iceberg incremental） | 每次读时比对两 snapshot 文件 diff | 高（每次都 diff） | 0 | 不采用为默认 |
| 预计算 Changelog（Paimon） | 写入/压缩时一次性算好，落成独立文件 | ~0（直接读） | 看模式：双写 / 比对 / 查找 | **采用** |

**设计哲学一句话**：把"推导变更"从读端（高频、每个消费者各算一遍）前移到写端/压缩端（一次算好、所有消费者共享），用一次性的写放大或压缩开销，换流式读的零额外成本。

由此衍生三个关键设计：① changelog 落成**独立文件**（与数据文件分离，可有不同格式/压缩/过期）；② changelog 与 snapshot **原子提交**（一致性）；③ **四种 producer** 覆盖不同延迟/成本权衡，默认 `none`（不需要就零成本）。

**流表二象性**：同一套存储两种读法——表视角读 snapshot 全量（batch），流视角读 snapshot 之间的 changelog（stream）。`ScanMode`（`paimon-core/.../table/source/ScanMode.java:22`）三态正是这套二象性的落点：`ALL`（表视角全量）/`DELTA`（增量文件，无 RowKind 语义）/`CHANGELOG`（专门的 changelog 文件，含 RowKind 语义）。

**与 Iceberg 的根本差异**（§14 详展）：Iceberg incremental scan 是**文件级 diff**、读时推导，只能区分"加了哪些文件/删了哪些文件"，UPDATE 表现为 DELETE+INSERT；Paimon changelog 是**行级、预计算**的 CDC 流，写入/压缩时就钉死了每行的 RowKind。

### 1.2 核心概念速查表

| 概念 | 一句话定义 | 关键源码 |
|------|-----------|---------|
| **ChangelogProducer** | 控制 changelog 生成策略的枚举，四态 `none/input/full-compaction/lookup` | `CoreOptions.java:4051`（枚举）、`:913`（配置项，已修正：旧稿标 3948/902） |
| **RowKind** | 每条 changelog 的变更标记：`+I` INSERT、`-U` UPDATE_BEFORE、`+U` UPDATE_AFTER、`-D` DELETE | `data/RowKind.java` |
| **MergeFunctionWrapper** | 包在 `MergeFunction` 外、在合并的同时吐出 changelog 的装饰器 | `mergetree/compact/MergeFunctionWrapper.java` |
| **FullChangelogMergeFunctionWrapper** | full-compaction 模式：用 maxLevel 旧值 vs 合并新值比对产 changelog | `.../compact/FullChangelogMergeFunctionWrapper.java:40` |
| **LookupChangelogMergeFunctionWrapper** | lookup 模式：缺高层旧值时主动 lookup，再比对产 changelog | `.../compact/LookupChangelogMergeFunctionWrapper.java:54` |
| **FirstRowMergeFunctionWrapper** | first-row 引擎专用：只 `contains` 检查、只产 `+I` | `.../compact/FirstRowMergeFunctionWrapper.java:28` |
| **LookupStrategy** | 封装"是否需要 lookup"的四个布尔决策 | `paimon-api/.../lookup/LookupStrategy.java:22`，构造在 `CoreOptions.java:3193` |
| **changelogManifestList** | Snapshot 指向 changelog 文件清单的字段，可为 null | `paimon-api/.../Snapshot.java:106` |
| **ChangelogFollowUpScanner** | 流式消费 changelog 的扫描器，跳过无 changelog 的 snapshot | `.../snapshot/ChangelogFollowUpScanner.java:28`（已修正：旧稿误称 AllDeltaFollowUpScanner） |
| **changelogDecoupled** | changelog 保留时长 > snapshot 时自动置位，解耦两者生命周期 | `paimon-api/.../options/ExpireConfig.java:54` |
| **Long-Lived Changelog** | snapshot 过期后被"提升"到独立 `changelog/` 目录的长寿命 changelog | `utils/ChangelogManager.java:54` |

### 1.3 高频生产陷阱

**陷阱 1：误把 NONE 模式的流读当 CDC。** `none` 下流读走 `DeltaFollowUpScanner`，只扫 APPEND snapshot 的 delta 文件，**所有行都是 `+I`**，看不到 UPDATE/DELETE。主键表想做 CDC 必须显式开 `input/lookup/full-compaction`。

**陷阱 2：`full-compaction` 未配 `full-compaction.delta-commits`，changelog 长时间不出。** 该模式只在全压缩时产 changelog，全压缩频率默认由 size-amplification 等策略决定，可能数小时一次。需显式配 `full-compaction.delta-commits=N`（每 N 次提交强制全压缩，`CoreOptions.java:1367`）。低延迟场景应改用 `lookup`。

**陷阱 3：在 first-row 引擎上配 `changelog-producer=lookup` —— 会被忽略。** `LookupMergeFunction.wrap()`（`:130`）检测到 `FirstRowMergeFunction` 直接返回原 factory，first-row 走自己的 `FirstRowMergeFunctionWrapper`，只产 `+I`，永不产 `-U/+U/-D`。

**陷阱 4：`input` 模式当万能 CDC，但上游缺 `-U`。** `input` 是把输入原样双写成 changelog，**上游有什么 RowKind 就出什么**。若上游是 INSERT-only 流（或配了 deduplicate 引擎），changelog 里全是 `+I`，下游看不到更新语义。需要精确 UPDATE 就用 `lookup/full-compaction` 让引擎自己比对。

**陷阱 5：以为切换 producer 会补算历史 changelog。** 切模式只影响切换后产生的 snapshot，历史 snapshot 不会回填 changelog。

**陷阱 6：changelog 没配独立过期 / 没配 Consumer，被提前清理导致流读断流或回退。** 默认 changelog 生命周期跟 snapshot 绑定。流读消费慢时，需用 Consumer 位点（`ConsumerManager`）保护未消费的 changelog，或用 `changelog.*-retained` 拉长保留（触发 `changelogDecoupled`）。详见 §11、§12 与 17 号文档。

**陷阱 7：误以为 `lookup` 无额外开销。** lookup 每次涉及 L0 的压缩都要查旧值，键空间大时 lookup I/O 不可忽视；它还依赖 LookupLevels 本地索引（磁盘开销，详见 01 §6）。代价换来的是远低于全压缩的延迟。

**陷阱 8：`changelog-producer.row-deduplicate` 的适用范围误解。** 该项只对 `lookup`/`full-compaction` 生效（`CoreOptions.java:922` 描述明确），用于"新旧值完全相同就不产 `-U/+U`"。不开则任何一次合并命中都可能产冗余 UPDATE，下游空转。

---

## 2. ChangelogProducer 四种模式总览

**① 要解决什么问题**

不同业务对 changelog 的延迟、精确性、成本要求差异极大：实时大屏要秒级、可接受写放大；数仓同步要省成本、容忍分钟级；去重场景只要 `+I`。一种固定策略必然在某些场景过度付费或延迟不达标。Paimon 的解法是把"产生时机"做成可选维度——同一份合并逻辑，挂在不同的时机上触发。

**② 设计原理与取舍：四种时机，一条共享的比对内核**

四种 producer 本质是"在 LSM 生命周期的哪个点产 changelog"：

- `none` —— 不产，流读退化为读 delta 文件（全 `+I`）。
- `input` —— 在 **flush 写缓冲**时，把输入原样双写一份。最早、最快，但"输入是什么就出什么"。
- `full-compaction` —— 在**全压缩**时，用 maxLevel 旧值 vs 合并新值比对。最精确，延迟由全压缩频率决定。
- `lookup` —— 在**任何涉及 Level 0 的压缩**时，缺旧值就主动 lookup，再比对。精确且延迟远低于全压缩。

关键洞察：`full-compaction` 与 `lookup` 的 changelog 计算内核**几乎相同**（都是"旧值 vs 新值"四象限比对，见 §5.2 与 §6.1 的 `setChangelog`），区别只在"旧值从哪来、何时触发"——前者等全压缩把数据沉到 maxLevel 再读，后者不等、主动 lookup。这解释了为什么二者精确性一致、延迟却差一个数量级。

**③ 关键源码：枚举与配置入口**（见 §2.1）

**④ 风险/陷阱/边界**

- 切换 producer 不回填历史（陷阱 5）；`input` 依赖上游 CDC 质量（陷阱 4）；`full-compaction` 不配频率会长时间无 changelog（陷阱 2）；`lookup` 对 first-row 无效（陷阱 3）。
- `none` 默认值意味着"主键表默认也不产 changelog"——很多人误以为开了主键就有 CDC，必须显式配。

**⑤ 收益与代价**

收益：用一个配置项覆盖从"零成本无 CDC"到"秒级精确 CDC"的全谱系，且四种模式共享同一套存储与消费接口。代价：四种模式的运维心智不统一（延迟来源、文件大小、触发条件各异），选型错误会直接体现为延迟或成本异常。

### 2.1 枚举定义与配置入口

`ChangelogProducer` 枚举（`paimon-api/.../CoreOptions.java:4051`，已修正：旧稿标 3948）：

```java
public enum ChangelogProducer implements DescribedEnum {
    NONE("none", "No changelog file."),
    INPUT("input", "Double write to a changelog file when flushing memory table, the changelog is from input."),
    FULL_COMPACTION("full-compaction", "Generate changelog files with each full compaction."),
    LOOKUP("lookup", "Generate changelog files through 'lookup' compaction.");
}
```

配置项 `CHANGELOG_PRODUCER`（`CoreOptions.java:913`，已修正：旧稿标 902），**默认 `NONE`**——不需要 changelog 的场景零开销，用户按需显式开启。配套两个去重项：`changelog-producer.row-deduplicate`（`:922`，默认 false）与 `changelog-producer.row-deduplicate-ignore-fields`（`:929`），**仅对 `lookup`/`full-compaction` 生效**。

### 2.2 四种模式对比矩阵

| 维度 | NONE | INPUT | FULL_COMPACTION | LOOKUP |
|------|------|-------|-----------------|--------|
| **产生时机** | 不产生 | flush 写缓冲时 | 全压缩时 | 涉及 L0 的压缩时 |
| **changelog 来源** | 无 | 原始输入 | maxLevel 旧值 vs 合并新值 | lookup 旧值 vs 合并新值 |
| **延迟** | N/A | 最低（checkpoint 级） | 高（全压缩间隔） | 中（L0 压缩频率） |
| **RowKind 精确性** | 全 `+I`（流读 delta） | 取决于输入 | 精确 | 精确 |
| **写放大** | 无 | ~2x | 压缩时产生 | 压缩时产生 |
| **额外资源** | 无 | 双写 I/O | 全压缩 I/O | lookup I/O + 本地索引 |
| **changelog 归宿** | — | `DataIncrement` | `CompactIncrement` | `CompactIncrement` |
| **流读扫描器** | `DeltaFollowUpScanner` | `ChangelogFollowUpScanner` | `ChangelogFollowUpScanner` | `ChangelogFollowUpScanner` |
| **适用 MergeEngine** | 所有 | 所有（注意 deduplicate） | 所有主键表 | 除 first-row 外 |

> 流读扫描器一行已修正：旧稿对 input/full-compaction/lookup 全标 `AllDeltaFollowUpScanner`，实际是 `ChangelogFollowUpScanner`（详见 §9）。

### 2.3 一条变更如何流过四种模式（产生时机串讲）

以同一条 upsert `(pk=42)` 为例，看四种模式在 LSM 生命周期的哪一点动手：

```
写入 → WriteBuffer 排序合并 → flush 成 Level 0 文件 → compaction 逐级下沉 → 全压缩沉到 maxLevel
        │                        │                       │                      │
      input ────────────────────┘（flush 时双写，最早）  │                      │
      lookup ─────────────────────────────────────────────┘（任意含 L0 压缩，缺旧值就 lookup）
      full-compaction ─────────────────────────────────────────────────────────┘（仅全压缩，读 maxLevel 旧值）
      none：以上都不产，流读只能读 flush 出的 delta 文件，全 +I
```

要点：① 越早产（input）延迟越低但越依赖输入；② 越晚产（full-compaction）越精确但延迟越高；③ lookup 用"主动查旧值"打破了"必须等数据沉到高层"的限制，把精确性和低延迟同时拿到手。

### 2.4 模式选择决策树

```
需要流式 CDC 消费?
├── 否 → none（默认，零开销）
└── 是 → 上游自带完整 CDC 语义（如 MySQL CDC / Flink CDC）?
    ├── 是 → input（最低延迟，直接双写输入）
    └── 否 → first-row 去重引擎?
        ├── 是 → input 或 full-compaction（lookup 对 first-row 无效）
        └── 否 → 能接受分钟级以上延迟?
            ├── 是 → full-compaction（省 lookup 开销）
            └── 否 → lookup（推荐：精确且低延迟）
```

---

## 3. NONE 模式 — 不产生 Changelog

**① 要解决什么问题**

不是所有表都需要 CDC。append-only 日志表、纯批处理归档、临时表，产生 changelog 纯属浪费。`none` 是默认值，体现"不需要的功能不应有任何成本"。

**② 设计原理与取舍**

`none` 下写端完全不创建 changelog writer（见 §4.1 的判断条件），流读则走 `DeltaFollowUpScanner`：只扫 `CommitKind.APPEND` 的 snapshot、用 `ScanMode.DELTA` 读 delta 文件。注意这里仍能"流式读"，但读到的是**文件级增量**，所有行都是 `+I`——这与"CDC"有本质区别。

```java
// DeltaFollowUpScanner.java:34
public boolean shouldScanSnapshot(Snapshot snapshot) {
    return snapshot.commitKind() == Snapshot.CommitKind.APPEND;  // 跳过 COMPACT / OVERWRITE
}
// :47
public SnapshotReader.Plan scan(...) {
    return snapshotReader.withMode(ScanMode.DELTA).withSnapshot(snapshot).read();
}
```

**为什么只扫 APPEND**：`COMPACT` snapshot 只是文件重组、不引入新数据；`OVERWRITE` 默认不读（除非开 `streaming-read-overwrite`）。只认 APPEND 才能避免把同一份数据重复推给下游。

**③ 关键源码**：`paimon-core/.../table/source/snapshot/DeltaFollowUpScanner.java`（全文仅 ~50 行）。

**④ 风险/陷阱/边界**

- 在主键表上用 `none` 做流读是常见误用（陷阱 1）：下游看到的全是 `+I`，无法区分更新/删除，被迫自己维护状态去 diff。
- 后续切到其他 producer，历史 snapshot 不会补 changelog。

**⑤ 收益与代价**

收益：零存储、零计算开销。代价：无行级 CDC 能力。适用于 append-only、纯批、不需要变更语义的场景。

---

## 4. INPUT 模式 — flush 时双写输入

**① 要解决什么问题**

当上游已经是完整 CDC 流（MySQL Binlog、Flink CDC），最省事且延迟最低的做法是：把输入里携带的 RowKind 原样保留下来，不要重新推导。`input` 模式正是这个思路——在数据落盘的同一时刻把它再写一份到 changelog 文件。

**② 设计原理与取舍：为什么在 flush 双写**

`flush` 是 WriteBuffer 持久化的唯一出口。在这一点双写有三重好处：① changelog 与数据文件包含完全相同的记录、原子提交；② WriteBuffer 的排序+合并只做一次，结果同时喂给两个 writer，不重复排序；③ 不增加额外内存压力，只多一个输出目标。代价是 changelog writer 的输出基本等于一份数据，**写放大 ~2x**。

### 4.1 flushWriteBuffer 的双写实现

**源码**：`paimon-core/.../mergetree/MergeTreeWriter.java`，`flushWriteBuffer` 在 `:209`。

```java
// 仅当 changelogProducer == INPUT 时才创建 changelogWriter，否则为 null（其余模式零开销）
final RollingFileWriter<KeyValue, DataFileMeta> changelogWriter =
        changelogProducer == ChangelogProducer.INPUT
                ? writerFactory.createRollingChangelogFileWriter(0)   // :217-218
                : null;
final RollingFileWriter<KeyValue, DataFileMeta> dataWriter =
        writerFactory.createRollingMergeTreeFileWriter(0, FileSource.APPEND);

// forEach 在一次遍历里同时写 dataWriter 和 changelogWriter（write 回调二选一）
writeBuffer.forEach(keyComparator, mergeFunction,
        changelogWriter == null ? null : changelogWriter::write,
        dataWriter::write);
// ...
newFilesChangelog.addAll(changelogWriter.result());  // :238 收集 changelog 文件
```

判断条件极简——`changelogProducer == INPUT`，每次 flush 评估一次，确保其余模式不创建 changelog writer、零开销。`createRollingChangelogFileWriter(0)`：`Rolling` 表示按大小滚动，`0` 表示 Level 0，`Changelog` 前缀用 `changelog-file.prefix`（默认 `"changelog-"`，`CoreOptions.java:345`）与数据文件区分。

### 4.2 changelog 文件的独立存储与归宿

changelog 文件与数据文件**同目录、不同前缀**（详见 §8.2），且支持独立配置——`changelog-file.format`/`.compression`/`.stats-mode` 默认沿用数据文件，但可单独设。**为什么允许独立配置**：数据文件常做点查/列裁剪，偏好列存；changelog 多为顺序扫描，可选更利于顺序读的格式与压缩。

归宿上，`input` 产生的 changelog 进入 `DataIncrement.changelogFiles`（不是 `CompactIncrement`），因为它在写入阶段产生、与压缩无关。`MergeTreeWriter` 在 `:238` 收进 `newFilesChangelog`，最终由 `drainIncrement()`（`:280`）打包进 `DataIncrement`（见 §8.2 两条路径）。

### 4.3 风险与收益

**风险/陷阱**：① 写放大 ~2x，高吞吐场景要评估 I/O 容量（陷阱 4 提到的另一面）；② changelog 内容**完全取决于输入**——上游若缺 `-U`、或是 INSERT-only 流配 deduplicate 引擎，changelog 里就看不到更新语义；③ 频繁 flush 会产生大量小 changelog 文件，需配合 compaction。

**收益与代价**：延迟最低（checkpoint 级），逻辑最简单（无 lookup/比对）。代价是写放大与对上游 CDC 质量的强依赖。适用：上游有完整 CDC、要求最低延迟。

---

## 5. FULL_COMPACTION 模式 — 全压缩比对旧值

**① 要解决什么问题**

上游不带完整 CDC（如 INSERT-only 流 + deduplicate 引擎）时，要让 Paimon 自己推导出精确的 `+I/-U/+U/-D`。难点是"旧值在哪"——它散在各层文件里。`full-compaction` 的洞察：全压缩是**唯一能看到所有层级数据**的时刻，此时可以拿到"上次沉淀的旧值"与"本次合并的新值"做比对。

**② 设计原理与取舍：maxLevel = 旧值**

LSM 中只有全压缩（UniversalCompaction）会把数据写到最高层 maxLevel（见 `FullChangelogMergeTreeCompactRewriter.java:84` 的 `upgradeStrategy`：`outputLevel == maxLevel` 才走 `CHANGELOG_NO_REWRITE`）。因此 maxLevel 的记录就是"上次全压缩后的终值"=旧值，本次合并出的 merged=新值。包装器 `FullChangelogMergeFunctionWrapper`（`:40`）在合并的同时分离出 maxLevel 的 KV，最后比对产 changelog。它在 `FullChangelogMergeTreeCompactRewriter.createMergeWrapper()`（`:90`）被构造，`maxLevel` 来自 `numLevels-1`。

### 5.1 topLevelKv 与 merged 的比对算法

`add()`（`:74`）把来自 maxLevel 的 KV 单独存进 `topLevelKv`，其余进 mergeFunction：

```java
public void add(KeyValue kv) {
    if (maxLevel == kv.level()) {                       // :75
        Preconditions.checkState(topLevelKv == null, "Top level key-value already exists!...");
        topLevelKv = kv;                                // 记录旧值
    }
    if (initialKv == null) initialKv = kv;              // 处理"只有一条记录"的快路径
    else { if (!isInitialized) { merge(initialKv); isInitialized = true; } merge(kv); }
}
```

`getResult()`（`:96`）做四象限比对（核心逻辑，已核对）：

```java
KeyValue merged = mergeFunction.getResult();
if (topLevelKv == null) {                               // 无旧值
    if (merged.isAdd()) addChangelog(INSERT, merged);   // → +I
} else {                                                // 有旧值
    if (!merged.isAdd())                                // 合并成 retract
        addChangelog(DELETE, topLevelKv);               // → -D（用旧值）
    else if (valueEqualiser == null
             || !valueEqualiser.equals(topLevelKv.value(), merged.value()))  // 值变了
        addChangelog(UPDATE_BEFORE, topLevelKv)         // → -U + +U
            .addChangelog(UPDATE_AFTER, merged);
    // else 值相同 → 不产 changelog
}
```

### 5.2 Changelog 产生决策矩阵

| topLevelKv（旧值） | merged（新值） | 产出 | 含义 |
|---|---|---|---|
| null | isAdd | `+I` | 新键插入 |
| null | retract | 无 | 无旧值的删除，忽略 |
| 存在 | retract | `-D`（旧值） | 删除 |
| 存在 | isAdd，值不同 | `-U`+`+U` | 更新 |
| 存在 | isAdd，值相同 | 无 | 无实质变更 |

`valueEqualiser` 仅在 `changelog-producer.row-deduplicate=true` 时非 null。**它在解决什么**：避免"合并结果与旧值字节相同"时仍产 `-U/+U`（下游空转）；还能用 `row-deduplicate-ignore-fields` 忽略某些字段（如更新时间戳）再比对。注意该项**只对 lookup/full-compaction 生效**（`CoreOptions.java:922` 描述）。

### 5.3 full-compaction.delta-commits 控制频率

`full-compaction` 的 changelog 只在全压缩时出，全压缩频率不控制就可能很久才一次。`FULL_COMPACTION_DELTA_COMMITS`（`CoreOptions.java:1367`，已修正：旧稿标 1356）：

```java
key("full-compaction.delta-commits").intType().noDefaultValue()
    .withDescription("For streaming write, full compaction will be constantly triggered after delta commits. ...");
```

设 N 即"每 N 次增量提交强制一次全压缩"——延迟与压缩开销之间的旋钮：设 1=每次提交都全压缩（最低延迟、最高成本），设 10=每 10 次（更高延迟、更低成本）。**不配是常见生产事故（陷阱 2）**：changelog 可能数小时才出一批。

### 5.4 风险与收益

**风险/陷阱**：① 不配 delta-commits 导致 changelog 长时间不出；② 全压缩是重操作（读所有层），频繁触发 CPU/IO 飙升；③ 一次全压缩可能批量吐出大量 changelog，给下游造成尖峰；④ 没触发全压缩的 snapshot 没有 changelog（`changelogManifestList` 为 null），流读会跳过它（§9.2）。

**收益与代价**：changelog 精确、不依赖上游、flush 阶段无额外写放大。代价是延迟高（等全压缩）+ 全压缩本身的 I/O/CPU。适用：对延迟不敏感、上游无 CDC、批量导入/数据修复。

---

## 6. LOOKUP 模式 — 查找式低延迟 Changelog

**① 要解决什么问题**

`full-compaction` 精确但延迟高（等全压缩），`input` 低延迟但依赖上游 CDC。`lookup` 要同时拿到"精确"和"低延迟"：不等数据沉到 maxLevel，而是在**任意一次涉及 Level 0 的压缩**中，当合并里缺少高层旧值时**主动 lookup 查出旧值**，再走与 full-compaction 相同的比对。

**② 设计原理与取舍：用主动查找打破"必须等全压缩"**

L0 压缩远比全压缩频繁，所以 lookup 的 changelog 延迟远低于 full-compaction。代价是每次压缩可能要查旧值——靠 LookupLevels 本地索引（RocksDB/Hash，详见 01 §6）把 lookup I/O 压到可控。

### 6.1 LookupChangelogMergeFunctionWrapper 核心逻辑

**源码**：`paimon-core/.../mergetree/compact/LookupChangelogMergeFunctionWrapper.java:54`。`getResult()`（`:104`）分四步：

```java
// 1. 取出参与合并的最高层记录，并记录是否含 Level 0
KeyValue highLevel = mergeFunction.pickHighLevel();
boolean containLevel0 = mergeFunction.containLevel0();
// 2. 若没有高层旧值，主动 lookup
if (highLevel == null) {
    T lookupResult = lookup.apply(mergeFunction.key());
    if (lookupResult != null) {
        if (lookupStrategy.deletionVector) {           // DV 模式：拿到旧值所在文件+行位置
            ... deletionVectorsMaintainer.notifyNewDeletion(fileName, rowPosition);  // :126
        } else {
            highLevel = (KeyValue) lookupResult;
        }
        if (highLevel != null) mergeFunction.insertInto(highLevel, comparator);
    }
}
// 3. 计算合并结果
KeyValue result = mergeFunction.getResult();
// 4. 仅当含 Level 0 且需要产 changelog 时，比对 highLevel(旧) vs result(新)
reusedResult.reset();
if (containLevel0 && lookupStrategy.produceChangelog) {   // :141
    setChangelog(highLevel, result);
}
return reusedResult.setResult(result);
```

`setChangelog`（`:148`，旧稿未展示，已补）的四象限逻辑与 full-compaction 的 `getResult` **完全同构**——这印证了 §2 说的"共享比对内核"：

```java
private void setChangelog(@Nullable KeyValue before, KeyValue after) {
    if (before == null || !before.isAdd()) {        // 无有效旧值
        if (after.isAdd()) addChangelog(INSERT, after);              // → +I
    } else {                                         // 有旧值
        if (!after.isAdd()) addChangelog(DELETE, before);           // → -D
        else if (valueEqualiser == null
                 || !valueEqualiser.equals(before.value(), after.value()))
            addChangelog(UPDATE_BEFORE, before).addChangelog(UPDATE_AFTER, after);  // → -U +U
        // else 值相同 → 不产
    }
}
```

`LookupMergeFunction.pickHighLevel()`（`:76`）选"最小的正 Level"作为旧值——Level 越低越新，但 `level <= 0` 是未落盘/L0 的新数据要跳过，所以"最小正 Level"=距今最近的已确认历史值：

```java
public KeyValue pickHighLevel() {
    KeyValue highLevel = null;
    for (KeyValue kv : candidates) {
        if (kv.level() <= 0) continue;              // 跳过 L0 与 write-buffer 的 level -1
        if (highLevel == null || kv.level() < highLevel.level()) highLevel = kv;
    }
    return highLevel;
}
```

> 已修正：旧稿把 `containLevel0()` 写成"遍历 candidates"，实际 `containLevel0` 是在 `add()`（`:63`）里 `kv.level()==0` 时置位的布尔标志（`:66`），`containLevel0()`（`:71`）只返回该标志。

### 6.2 "只在含 Level 0 时产 changelog" 的不变式

`getResult` 第 4 步的守卫 `containLevel0 && produceChangelog` 是 lookup 模式的核心不变式。**为什么必须含 Level 0 才产**：

- Level 0 文件是 flush 产生的，承载真实的业务变更；
- 纯高层压缩（如 L1→L2）只是文件重组，没有新业务数据参与；
- 若高层重组也产 changelog，同一条变更会被**重复记录**。

这条不变式让"每条业务变更只产生一次 changelog"，且与 `input`（只在 flush 产）语义对齐。

### 6.3 LookupStrategy 与 Deletion Vector 协同

`LookupStrategy`（`paimon-api/.../lookup/LookupStrategy.java:22`）把"是否需要 lookup"收敛成四个布尔：

```java
this.needLookup = produceChangelog || deletionVector || isFirstRow || forceLookup;  // :40
```

构造点 `CoreOptions.lookupStrategy()`（`:3193`）一眼看清触发条件：

```java
return LookupStrategy.from(
        mergeEngine() == FIRST_ROW,                     // isFirstRow
        changelogProducer() == LOOKUP,                  // produceChangelog
        deletionVectorsEnabled(),                       // deletionVector
        options.get(FORCE_LOOKUP));                     // forceLookup
```

**DV 协同**：当 `deletionVector=true`，lookup 查到旧值后不写 retract 记录，而是 `deletionVectorsMaintainer.notifyNewDeletion(fileName, rowPosition)`（`:126`）在索引里把旧行标记为已删——MOW 表的更新就是"标删旧行 + 写新行"，lookup 在查旧值时顺手完成标删（DV 机制详见 01 §7.2、04 号文档）。把 lookup、changelog、DV 三者的触发统一进一个 strategy 对象，避免散落的 if-else。

### 6.4 为什么延迟低于 FULL_COMPACTION

| 维度 | FULL_COMPACTION | LOOKUP |
|---|---|---|
| 触发时机 | 仅全压缩 | 每次涉及 L0 的压缩 |
| 旧值来源 | 等数据沉到 maxLevel 直接读 | 主动 lookup（不等下沉） |
| 频率 | 由 delta-commits 控制（稀） | 跟随 L0 压缩（密） |
| I/O | 全量读写所有层 | 点查 + 部分读写 |

核心优势：不等全压缩，L0 一参与压缩就能出 changelog；L0 压缩频率远高于全压缩。代价：每次压缩要 lookup 查旧值，键空间大时 lookup I/O 显著（靠本地索引缓解）。`lookup-wait` 可控制 commit 是否等待 lookup 压缩完成。

### 6.5 风险与收益

**风险/陷阱**：① first-row 引擎无效（陷阱 3）；② 维护本地 lookup 索引有磁盘开销；③ 键空间极大时 lookup 成本上升；④ 与 DV 协同时多一份 DV 维护开销。

**收益与代价**：精确 CDC + 低延迟 + 不依赖上游 + 天然支持 DV，是推荐的通用方案。代价是 lookup I/O 与本地索引。适用：上游无 CDC、要求低延迟的实时数仓/流式 ETL。

---

## 7. FirstRow Changelog — 首行去重的特例

**① 要解决什么问题**

first-row 引擎的语义是"每个键只保留第一条记录"（用户首次行为、设备首次上线、幂等写入）。这种场景的 changelog 语义天然受限：只有"某键首次出现"是有意义的事件，永远不会有更新或删除。

**② 设计原理与取舍：只需"键是否存在"，不需旧值**

正因为只关心"是不是第一次"，first-row **不需要读取旧值做比对**，只要判断键是否已存在。这比 lookup/full-compaction 的"取旧值 + 四象限比对"轻得多。所以 first-row 有**专属包装器** `FirstRowMergeFunctionWrapper`（`:28`），不复用 `LookupChangelogMergeFunctionWrapper`。

`LookupMergeFunction.wrap()`（`:130`）显式短路：

```java
if (wrapped.create() instanceof FirstRowMergeFunction) {
    return wrapped;  // don't wrap first row, it is already OK
}
```

**这解释了陷阱 3**：在 first-row 表上配 `changelog-producer=lookup` 不会启用 lookup 比对——它直接走 first-row 自己的 wrapper。

**③ 关键源码**：`FirstRowMergeFunctionWrapper.getResult()`（`:56`）：

```java
KeyValue result = mergeFunction.getResult();
if (mergeFunction.containsHighLevel) {        // 合并中已有高层记录 → 键已存在
    reusedResult.setResult(result);
    return reusedResult;                      // 不产 changelog
}
if (contains.test(result.key())) {            // 通过 contains 过滤器查 LookupLevels：键已存在
    return reusedResult;                       // 空结果（重复键被吞）
}
return reusedResult.setResult(result).addChangelog(result);  // 新键 → 产 +I
```

`contains` 是 `Filter<InternalRow>`，通常接到 LookupLevels，判断该 key 是否已在更高层存在——这是"只 contains 检查、不取旧值"的体现。

**④ 风险/陷阱/边界**

- changelog 只含 `+I`，下游必须理解这个语义（别期待 `-U/+U/-D`）；
- 键已存在则新记录被吞、不产任何 changelog；
- `contains` 检查依赖 lookup 索引，键空间大时有开销。

**⑤ 收益与代价**

收益：逻辑最简、性能高、契合去重语义。代价：表达力受限（只有 INSERT）。适用：去重、幂等写入、首次行为记录。

---

## 8. Changelog 的存储架构

**① 要解决什么问题**

changelog 与数据文件有**不同的生命周期与读取模式**：流读可能要保留 24 小时 changelog 做故障恢复，但批读只要最近 1 小时的 snapshot；changelog 顺序扫描、数据文件点查列裁剪。把两者绑死会让"为了留 changelog 而留 snapshot"，浪费且不灵活。

**② 设计原理与取舍：清单层分离 + 可解耦的生命周期**

Paimon 的做法是"物理同目录、逻辑分清单、生命周期可解耦"：changelog 文件与数据文件在同一 bucket 目录下（仅前缀不同），但在 Snapshot 元数据里通过独立的 `changelogManifestList` 引用，且支持独立过期（§11）甚至"提升"为独立 `changelog/` 目录下的 Long-Lived Changelog。

### 8.1 Snapshot 的 changelogManifestList 字段

**源码**：`paimon-api/.../Snapshot.java`，字段在 `:106`（getter `:315`，已修正：旧稿标 58-59/106-115）。

```java
@JsonProperty(FIELD_CHANGELOG_MANIFEST_LIST)
@JsonInclude(JsonInclude.Include.NON_NULL)
@Nullable
protected final String changelogManifestList;   // 记录本 snapshot 产生的所有 changelog；无 changelog 则 null
```

Snapshot 的四类清单引用并列：`baseManifestList`（全量）/`deltaManifestList`（本次增量）/`changelogManifestList`（changelog，可 null）/`indexManifest`（索引）。

**`changelogManifestList` 为何可 null**：① `none` 模式恒 null；② 任何模式下，某次 commit 没有数据写入/没有产生变更时为 null；③ `full-compaction` 下未触发全压缩的 snapshot 为 null。这个"可 null"是 §9 流式消费跳过逻辑的根因。

### 8.2 文件分离存储与两条产生路径

物理上 changelog 与数据文件**同 bucket 目录、靠前缀区分**：数据 `data-{uuid}.parquet`（`data-file.prefix`），changelog `changelog-{uuid}.parquet`（`changelog-file.prefix`）。**为什么不独立目录**：同 bucket 下共享分区/桶目录结构，清单层已用文件类别区分，物理无需再分目录，简化路径管理与清理。

按 producer 不同，changelog 走两条收集路径，最终在 `MergeTreeWriter.drainIncrement()`（`:280`）汇总：

| 路径 | 模式 | 收集字段 | 归宿 |
|---|---|---|---|
| 路径 1（写入端） | INPUT | `newFilesChangelog`（`:238` addAll） | `DataIncrement.changelogFiles` |
| 路径 2（压缩端） | FULL_COMPACTION / LOOKUP / FIRST_ROW | `compactChangelog`（`:328` 从 `CompactResult.changelog()`） | `CompactIncrement.changelogFiles` |

```java
private CommitIncrement drainIncrement() {                      // :280
    DataIncrement dataIncrement = new DataIncrement(
            ..., new ArrayList<>(newFilesChangelog));            // INPUT 的 changelog
    CompactIncrement compactIncrement = new CompactIncrement(
            ..., new ArrayList<>(compactChangelog));             // COMPACT 类的 changelog
}
```

### 8.3 Long-Lived Changelog 与 ChangelogManager

当 changelog 保留配置 > snapshot（触发 `changelogDecoupled`，§11.2），snapshot 过期时其 changelog 不能随之删除，而是被"提升"到独立的 `changelog/` 目录长期保存。这由 `ChangelogManager`（`utils/ChangelogManager.java:54`）管理：

```java
public static final String CHANGELOG_PREFIX = "changelog-";        // :60
public Path longLivedChangelogPath(long snapshotId) {              // :114
    return new Path(branchPath(tablePath, branch) + "/changelog/" + CHANGELOG_PREFIX + snapshotId);
}
public void commitChangelog(Changelog changelog, long id) {       // :123
    fileIO.writeFile(longLivedChangelogPath(id), changelog.toJson(), true);
}
```

目录结构：`snapshot/`（含 `changelogManifestList`）、`changelog/`（独立元数据 JSON）、`manifest/`、`bucket-N/`（data 与 changelog 文件混放）。

**④ 风险/陷阱/边界**：① 误以为 changelog 在独立目录（实际同 bucket）；② `changelogManifestList` 可 null，消费/清理都要处理 null 分支；③ Long-Lived Changelog 的 JSON 元数据多了会成为元数据负担。

**⑤ 收益与代价**：灵活的独立生命周期与配置、批流分离。代价是额外存储（~一份数据）与更复杂的清理逻辑（两条路径，§11.3）。

---

## 9. Changelog 的流式消费机制

**① 要解决什么问题**

流式消费者要持续跟踪新 snapshot，并在不同 producer 模式下用正确的方式读出变更——`none` 读 delta、其余读 changelog——同时跨过那些"没有 changelog"的 snapshot 而不中断流。

**② 设计原理与取舍：按 producer 选扫描器**

入口是 `FollowUpScanner`（接口仅两方法：`shouldScanSnapshot` 决定要不要这个 snapshot、`scan` 决定怎么读）。`DataTableStreamScan.createFollowUpScanner()`（`:264`）按 producer 选实现——这里是旧稿的核心错误来源，下面给出**经核对的正确映射**。

### 9.1 FollowUpScanner 体系与正确的扫描器映射

```java
// DataTableStreamScan.java:264
switch (changelogProducer) {
    case NONE:            followUpScanner = new DeltaFollowUpScanner();     break;  // :276
    case INPUT:
    case FULL_COMPACTION:
    case LOOKUP:          followUpScanner = new ChangelogFollowUpScanner(); break;  // :281
}
```

| ChangelogProducer | FollowUpScanner | shouldScanSnapshot | scan |
|---|---|---|---|
| NONE | `DeltaFollowUpScanner` | 仅 `CommitKind.APPEND` | `ScanMode.DELTA` + `read()` |
| INPUT / FULL_COMPACTION / LOOKUP | `ChangelogFollowUpScanner` | 仅 `changelogManifestList != null` | `ScanMode.CHANGELOG` + `read()` |

> **重大修正**：旧稿全篇称 input/full-compaction/lookup 用 `AllDeltaFollowUpScanner` + `ScanMode.DELTA` + `readChanges()` 并"降级为 DELTA"。实际流读用的是 `ChangelogFollowUpScanner`：它**只接受有 changelog 的 snapshot**（无 changelog 的直接跳过、不降级），并用 `ScanMode.CHANGELOG` + `read()` 直读 changelog 文件。`AllDeltaFollowUpScanner` 仅用于 `FILE_MONITOR` 扫描模式（专用 compaction 监控作业），它才用 `readChanges()`。

### 9.2 ChangelogFollowUpScanner：跳过无 changelog 的 snapshot

**源码**：`paimon-core/.../table/source/snapshot/ChangelogFollowUpScanner.java:28`（全文 ~46 行）。

```java
public boolean shouldScanSnapshot(Snapshot snapshot) {
    if (snapshot.changelogManifestList() != null) return true;  // :34
    LOG.debug("Next snapshot id {} has no changelog, check next one.", snapshot.id());
    return false;                                               // 无 changelog → 跳过这个 snapshot
}
public SnapshotReader.Plan scan(Snapshot snapshot, SnapshotReader snapshotReader) {
    return snapshotReader.withMode(ScanMode.CHANGELOG).withSnapshot(snapshot).read();  // :44 直读 changelog
}
```

这正是为什么 `full-compaction` 模式下"某些 snapshot 没 changelog"会被**跳过**（陷阱在 §5.4）——不是降级成 `+I`，而是流读直接略过它、消费下一个有 changelog 的 snapshot。所以 `full-compaction` 的流读延迟取决于"下一个带 changelog 的 snapshot 何时产生"。这也是低延迟应选 `lookup`（几乎每个 snapshot 都带 changelog）的根本原因。

> 关于流式首个 plan 的细节：`lookup` 首读会用 `withLevelFilter(level -> level > 0)`（`DataTableStreamScan.java:160`，过滤掉尚未压缩的 L0，等其后续压缩产 changelog），`full-compaction` 首读则只读 maxLevel（`:162`）。即起始就与各自的"changelog 产生层"对齐。

### 9.3 ScanMode 三态与 readChanges 的真实分工

`ScanMode`（`ScanMode.java:22`）三态：`ALL`（全量数据，表视角）、`DELTA`（增量数据文件，无 RowKind）、`CHANGELOG`（专门的 changelog 文件，含 RowKind）。

两条读路径，别混淆：

- **正常流读（`ChangelogFollowUpScanner`）**：`ScanMode.CHANGELOG` + `read()`，直读 changelog manifest 引用的文件。
- **`readChanges()`（仅 `AllDeltaFollowUpScanner` / FILE_MONITOR 用）**：`SnapshotReaderImpl.readChanges()`（`:473`，已修正：旧稿标 466）先 `withMode(ScanMode.DELTA)`，再按 `FileKind.DELETE`/`ADD` 分组生成 before/after 增量计划——这是给 compaction 监控/文件级 diff 用的，不是普通 CDC 消费路径。

```java
// SnapshotReaderImpl.java:473
public Plan readChanges() {
    withMode(ScanMode.DELTA);
    FileStoreScan.Plan plan = scan.plan();
    Map<...> beforeFiles = groupByPartFiles(plan.files(FileKind.DELETE));
    Map<...> afterFiles  = groupByPartFiles(plan.files(FileKind.ADD));
    return toIncrementalPlan(true, beforeSnapshot, beforeFiles, plan.snapshot(), plan.watermark(), afterFiles);
}
```

**④ 风险/陷阱/边界**：① 误以为有"DELTA 降级"——CDC 流读不降级，无 changelog 的 snapshot 被跳过；② `none` 与有 changelog 的模式读路径完全不同，切 producer 后下游看到的 RowKind 语义会变；③ changelog 被提前清理会让流读断点找不到文件（用 Consumer 保护，§12）。

**⑤ 收益与代价**：扫描器按 producer 分派，逻辑清晰、读端零推导。代价是 `full-compaction` 跳过无 changelog snapshot 带来的延迟不确定性。增量读取/消费位点的完整展开见 17 号文档。

---

## 10. AuditLog 和 Binlog 系统表

**① 要解决什么问题**

下游对 CDC 格式需求不同：Flink CDC 管道要标准的"每行一个 RowKind"格式；审计/分析场景希望一行同时看到 UPDATE 前后值。Paimon 用两张只读系统表覆盖这两种形态，它们读的是**同一份 changelog**，只是呈现方式不同。

**② 设计原理与取舍：包装同一份 changelog，呈现两种 schema**

- `AuditLogTable`（`table/system/AuditLogTable.java:90`）：包装底层 `FileStoreTable`，在每行前加 `_ROW_KIND` 字段（值为 `"+I"/"-U"/"+U"/"-D"`），可选加 `_SEQUENCE_NUMBER`。schema = `[_ROW_KIND, 原列...]`。一次 UPDATE 仍是两行（`-U`+`+U`），与 Flink `RowKind` 对齐。
- `BinlogTable`（`table/system/BinlogTable.java`，继承 AuditLog）：把每列变成 ARRAY，一行就能装下 before/after。schema = `[_ROW_KIND, [col1], [col2]...]`。

```java
// AuditLogTable 构造：加特殊字段
specialFields.add(SpecialFields.ROW_KIND);                                   // :101
if (CoreOptions.fromMap(wrapped.options()).tableReadSequenceNumberEnabled()) // :104
    specialFields.add(SpecialFields.SEQUENCE_NUMBER);                        // :108
```

**Binlog 数组格式**：INSERT/DELETE 每列是长度 1 的数组（`[+I, [v1], [v2]]`），UPDATE 是长度 2（`[+U, [v1_before, v1_after], ...]`），类似 MySQL Binlog 的 before/after image。

**③ 关键源码**：`AuditLogTable.java:90`（`AUDIT_LOG = "audit_log"`，`:92`）、`BinlogTable.java`（`BINLOG = "binlog"`）。访问方式：`表名$audit_log` / `表名$binlog`。

**④ 风险/陷阱/边界**：① 两表是同一份数据的不同视图，不是两套数据；② `_ROW_KIND` 是字符串字段、与内部 `RowKind` 枚举不是一回事；③ Binlog 数组长度随变更类型变化，下游需按 `_ROW_KIND` 解析；④ AuditLog 的 UPDATE 记录数是 Binlog 的 2 倍；⑤ 系统表只读。

**⑤ 收益与代价**：用两种 schema 满足"流处理标准格式"与"审计紧凑格式"两类需求，零额外存储（复用 changelog）。代价是 Binlog 数组格式需下游特殊处理、AuditLog 的 UPDATE 行数翻倍。

---

## 11. Changelog 过期机制

**① 要解决什么问题**

changelog 会持续增长，必须有过期策略，但它的保留需求常与 snapshot 不同——流读要长保留以支持故障恢复，批读只要短保留。难点是：既要能独立配置 changelog 过期，又要保护"正在被消费但已过保留期"的 changelog 不被误删。

**② 设计原理与取舍：默认降级到 snapshot，超出则自动解耦**

设计哲学是"约定优于配置"：changelog 专属过期项默认 `noDefaultValue`——不配就沿用 snapshot 的过期策略（多数场景两者一致）；一旦 changelog 保留期 > snapshot，就自动进入 `changelogDecoupled` 模式，changelog 走自己的独立生命周期。

### 11.1 changelog 专属过期配置与降级

`CoreOptions.java`：`changelog.num-retained.min`（`:516`）、`.max`（`:523`）、`.time-retained`（`:530`），全部 `noDefaultValue`。降级在 `ExpireConfig.Builder.build()`（`:175` 一带）实现：

```java
changelogTimeRetain == null ? snapshotTimeRetain : changelogTimeRetain   // 不配则沿用 snapshot
```

**为什么用降级而非给默认值**：大多数表的 changelog 和 snapshot 过期需求相同，降级让用户"不操心 changelog 过期"，只有真正要拉长 changelog 保留时才显式配——配了就顺带触发解耦。

### 11.2 changelogDecoupled 解耦设计

**源码**：`paimon-api/.../options/ExpireConfig.java:54`。

```java
this.changelogDecoupled =
        changelogRetainMax > snapshotRetainMax
                || changelogRetainMin > snapshotRetainMin
                || changelogTimeRetain.compareTo(snapshotTimeRetain) > 0;   // :54-57
```

解耦后，snapshot 过期不会删它的 changelog，而是把该 snapshot 的 changelog "提升"为 Long-Lived Changelog（`ChangelogManager.commitChangelog()`，存 `changelog/` 目录），按 changelog 自己的策略独立清理。**典型场景**：批作业只留 1 小时 snapshot，但流消费者要 24 小时 changelog 做故障恢复。

### 11.3 ChangelogDeletion 的两条清理路径

**源码**：`paimon-core/.../operation/ChangelogDeletion.java`，`cleanUnusedDataFiles` 按 changelog 是否有独立清单分两路（对应 §8.2 两条产生路径）：

```java
if (changelog.changelogManifestList() != null) {
    deleteAddedDataFiles(changelog.changelogManifestList());      // 有独立清单：删 changelog 专属文件
} else if (manifestList.exists(changelog.deltaManifestList())) {
    cleanUnusedDataFiles(changelog.deltaManifestList(), skipper); // 无独立清单：从 delta 清单清理
}
```

`cleanUnusedManifests` 同理双路。这两路正反映了 changelog 的两种存在形态：独立 changelog 文件（input/lookup/full-compaction）vs 信息内嵌在 delta（如 none 的增量）。

**④ 风险/陷阱/边界**：① 不配 changelog 过期就跟 snapshot 同寿命，流读慢时易被清掉（用 Consumer 保护，§12）；② 保留过长存储爆，过短消费者恢复失败；③ Long-Lived Changelog 的 JSON 元数据多了成负担。

**⑤ 收益与代价**：批流分离的独立过期 + Consumer 保护。代价是更复杂的清理逻辑与额外的 Long-Lived Changelog 存储。

---

## 12. Consumer 位点与 Changelog 过期协同

> Consumer 机制由 **17 号文档主讲**，本节只讲与 changelog 过期保护直接相关的部分。

**① 要解决什么问题**

流读消费慢时，已过保留期的 changelog 若被清理，消费者就找不到断点文件、流读失败。需要一个"消费位点"机制，让过期逻辑知道"哪些 snapshot/changelog 还有人在用、不能删"。

**② 设计原理与取舍：记录 nextSnapshot，取全局最小做下界**

`Consumer`（`consumer/Consumer.java`）记录的是 `nextSnapshot`（下一个要消费的 id），而非"最后消费的"。**为什么**：`minNextSnapshot() - 1` 直接就是"所有消费者都已消费完"的最新 snapshot，该 id 之前的 snapshot/changelog 才可安全过期，省一次换算。

`ConsumerManager`（`consumer/ConsumerManager.java`）取所有 consumer 的最小 nextSnapshot 作为过期下界：

```java
public OptionalLong minNextSnapshot() {                              // 遍历 consumer/ 目录
    return listOriginalVersionedFiles(...).map(this::consumer)
            .filter(Optional::isPresent).map(Optional::get)
            .mapToLong(Consumer::nextSnapshot).reduce(Math::min);
}
```

存储：`consumer/consumer-{id}`（JSON `{"nextSnapshot": N}`）。

**③ 关键源码与配置**

`consumer.changelog-only`（`CoreOptions.java:1414`，已修正：旧稿标 1403）：

```java
key("consumer.changelog-only").booleanType().defaultValue(false)
    .withDescription("If true, consumer will only affect changelog expiration "
            + "and will not prevent snapshot from being expired.");
```

`true` 时 Consumer 位点**只挡 changelog 过期、不挡 snapshot 过期**（`changelogOnly()` 在 `CoreOptions.java:3501` 还要求 `changelogLifecycleDecoupled()`）——适合只做流读、不做批回溯的场景：snapshot 照常按策略过期，changelog 一直保到最慢的 consumer 消费完。

`EXACTLY_ONCE` vs `AT_LEAST_ONCE`（`ConsumerMode`，`CoreOptions.java:4371`，已修正：旧稿标 4268）：前者所有 reader 精确消费同一 snapshot 后才推进位点（默认，精确一次）；后者记录最慢 reader 进度（容忍重复、换吞吐）。

**④ 风险/陷阱/边界**：① 没配 Consumer 又流读慢 → changelog 被按保留策略清掉、流读断流；② Consumer 文件残留（作业删了没清 consumer）会永久挡住过期、撑爆存储；③ `consumer.changelog-only=true` 需配合解耦才有意义。

**⑤ 收益与代价**：以"全局最小位点"为下界精确保护在用数据，分离批/流过期需求。代价是要管理 consumer 文件生命周期（残留即泄漏）。位点推进、reset、消费语义细节见 17 号文档。

---

## 13. 增量读取（incremental-between）

> 增量读取由 **17 号文档主讲**，本节只点出它与 changelog 的关系。

**① 要解决什么问题**

除了"持续流读"，用户还需要"读两个 snapshot 之间的变更"（批式增量）。changelog 的有无，决定了能否读到精确 RowKind 还是只能读文件级 diff。

**② 设计原理与取舍：四种子模式，AUTO 优先用 changelog**

`IncrementalBetweenScanMode`（`CoreOptions.java:4139`，已修正：旧稿标 4036）：

| 模式 | 行为 | 与 changelog 的关系 |
|---|---|---|
| `AUTO`（默认） | 有 changelog 就读 changelog，否则读 delta | 自动复用 changelog 的精确 RowKind |
| `DELTA` | 只读两 snapshot 间新增/删除的数据文件 | 不需要 RowKind，文件级 |
| `CHANGELOG` | 只读 changelog 文件 | 强制要精确 CDC，无 changelog 则读不到 |
| `DIFF` | 比对起止 snapshot 的**全量数据**算差 | 不依赖 changelog，但要读两份全量 |

`AUTO` 的设计哲学：能用预计算的 changelog 就用（便宜且精确），没有才退而求其次。`DIFF` 是 `none` 模式下"事后计算变更"的兜底——代价是读两个 snapshot 的全量数据做行级 diff。

**③ 关键源码**：`IncrementalSplit`（`table/source/IncrementalSplit.java`）同时持有 `beforeFiles`/`afterFiles`（及各自 DeletionFile），供 `DIFF` 模式在 reader 层做行级比对；`SnapshotReaderImpl.readChanges()`（`:473`，§9.3）是 `DELTA`/`AUTO` 的底层计划生成。

**④ 风险/陷阱/边界**：`CHANGELOG` 子模式在 `none` 表或无 changelog 区间读不到数据；`DIFF` 对大表代价高（两份全量）。

**⑤ 收益与代价**：四子模式覆盖"精确 CDC / 文件增量 / 全量 diff"三类增量需求。详见 17 号文档。

---

## 14. 与 Iceberg CDC 能力的对比

**架构理念差异**

| 维度 | Paimon | Iceberg |
|------|--------|---------|
| **CDC 实现层次** | 存储引擎原生预计算 | 文件级 diff 推导 |
| **变更语义** | 行级 RowKind（`+I/-U/+U/-D`） | 文件级（新增/删除文件） |
| **产生时机** | 写入/压缩时 | 读取时推导 |
| **存储开销** | 额外 changelog 文件 | 无 |
| **一致性** | 与 snapshot 原子提交 | 依赖 snapshot 隔离 |

Paimon 优势：行级精确 CDC，下游零推导，特别契合 UPSERT（同主键的 INSERT 与 UPDATE 语义不同）。Iceberg 优势：零额外存储、写入无额外负担，适合分析场景偶尔看增量。

**延迟/精确性/开销对比**

| 维度 | Paimon INPUT | Paimon LOOKUP | Paimon FULL_COMPACTION | Iceberg Incremental |
|------|-------------|---------------|----------------------|---------------------|
| 延迟 | checkpoint 级 | L0 压缩级 | 全压缩级 | 读时计算 |
| RowKind 精确性 | 取决于输入 | 精确 | 精确 | 只有 INSERT/DELETE |
| UPDATE 语义 | 需输入提供 | 自动 `-U/+U` | 自动 `-U/+U` | 表现为 DELETE+INSERT |
| 写入开销 | ~2x | lookup I/O | 全压缩 I/O | 无 |
| 读取开销 | 直接读 | 直接读 | 直接读 | 需 diff 计算 |

**选型建议**

- 选 Paimon Changelog：要实时 CDC（< 分钟级）、要精确 UPDATE 语义、有 Flink 流下游、主键表频繁 UPSERT。
- 选 Iceberg incremental：偶尔看增量（小时/天级）、只要"哪些文件变了"、不需区分 UPDATE/INSERT、不想要额外 changelog 存储。

---

## 15. 设计决策总结

| 决策点 | 选择 | 取舍 / 代价 | 收益 |
|--------|------|------------|------|
| 变更在哪端推导 | 写端/压缩端**预计算** changelog | 写放大 / 压缩开销 / 额外存储 | 流式读零额外开销、行级精确 CDC |
| 提供几种 producer | 四种（`none/input/full-compaction/lookup`） | 运维心智不统一、选型门槛 | 覆盖零成本到秒级精确 CDC 全谱系 |
| 默认 producer | `none` | 主键表默认也无 CDC，需显式开 | 不需要就零开销 |
| INPUT 在哪产 | flush 时**双写**输入 | 写放大 ~2x、依赖上游 CDC 质量 | 延迟最低、逻辑最简 |
| FULL/LOOKUP 的旧值来源 | full 读 maxLevel；lookup 主动查 | full 延迟高；lookup 有 lookup I/O + 本地索引 | 不依赖上游、精确 `-U/+U/-D` |
| 何时产 changelog | 仅"含 Level 0"的合并 | 高层重组不产（需理解不变式） | 每条业务变更只产一次、不重复 |
| first-row 特例 | 专属 wrapper、只 `contains` | 只能产 `+I` | 去重场景逻辑最轻 |
| row-deduplicate | 可选，仅 lookup/full 生效 | 多一次值比较 | 值未变不产 `-U/+U`，下游不空转 |
| changelog 存储 | 同目录前缀区分 + 独立清单 + 可解耦 | 额外存储、清理两条路径 | 独立格式/压缩/过期、批流分离 |
| 流读扫描器 | 按 producer 分派（Delta / Changelog） | full 跳过无 changelog 的 snapshot → 延迟不定 | 读端零推导、语义清晰 |
| changelog 过期 | 默认降级到 snapshot，超出则解耦 | 解耦逻辑复杂 + Long-Lived 存储 | 约定优于配置、批流分离 |
| 消费保护 | Consumer 记 nextSnapshot，取全局最小 | 需管理 consumer 文件生命周期 | 精确保护在用数据不被误删 |
| CDC 呈现 | AuditLog（标准）/ Binlog（数组） | Binlog 需特殊解析、AuditLog UPDATE 翻倍 | 一份 changelog 两种消费形态 |

---

## 附录：关键源码路径索引

| 类 / 文件 | 路径:行号 | 核心职责 |
|---|---|---|
| `ChangelogProducer`（枚举） | `paimon-api/.../CoreOptions.java:4051` | 四种模式 |
| `CHANGELOG_PRODUCER`（配置） | `CoreOptions.java:913` | producer 配置项，默认 NONE |
| `CHANGELOG_PRODUCER_ROW_DEDUPLICATE` | `CoreOptions.java:922` | 去重，仅 lookup/full 生效 |
| `FULL_COMPACTION_DELTA_COMMITS` | `CoreOptions.java:1367` | 全压缩频率 |
| `CoreOptions.lookupStrategy()` | `CoreOptions.java:3193` | lookup 触发四条件 |
| `ConsumerMode` | `CoreOptions.java:4371` | 一致性模式 |
| `IncrementalBetweenScanMode` | `CoreOptions.java:4139` | 增量子模式 |
| `MergeTreeWriter.flushWriteBuffer` | `mergetree/MergeTreeWriter.java:209` | INPUT 双写 |
| `MergeTreeWriter.drainIncrement` | `mergetree/MergeTreeWriter.java:280` | 两条 changelog 收集汇总 |
| `FullChangelogMergeFunctionWrapper` | `.../compact/FullChangelogMergeFunctionWrapper.java:40` | full 比对，getResult `:96` |
| `LookupChangelogMergeFunctionWrapper` | `.../compact/LookupChangelogMergeFunctionWrapper.java:54` | lookup 比对，getResult `:104`、setChangelog `:148` |
| `LookupMergeFunction` | `.../compact/LookupMergeFunction.java:37` | pickHighLevel `:76`、wrap `:130` |
| `FirstRowMergeFunctionWrapper` | `.../compact/FirstRowMergeFunctionWrapper.java:28` | first-row，只产 +I |
| `LookupStrategy` | `paimon-api/.../lookup/LookupStrategy.java:22` | lookup 决策封装 |
| `Snapshot.changelogManifestList` | `paimon-api/.../Snapshot.java:106` | changelog 清单引用 |
| `ScanMode` | `.../table/source/ScanMode.java:22` | ALL/DELTA/CHANGELOG |
| `DataTableStreamScan.createFollowUpScanner` | `.../table/source/DataTableStreamScan.java:264` | 按 producer 选扫描器 |
| `ChangelogFollowUpScanner` | `.../snapshot/ChangelogFollowUpScanner.java:28` | 流读 changelog（跳无 changelog snapshot） |
| `DeltaFollowUpScanner` | `.../snapshot/DeltaFollowUpScanner.java:29` | NONE 模式流读 delta |
| `AllDeltaFollowUpScanner` | `.../snapshot/AllDeltaFollowUpScanner.java:25` | 仅 FILE_MONITOR 用，readChanges |
| `SnapshotReaderImpl.readChanges` | `.../snapshot/SnapshotReaderImpl.java:473` | 增量计划生成 |
| `ChangelogManager` | `utils/ChangelogManager.java:54` | Long-Lived Changelog |
| `ConsumerManager` / `Consumer` | `consumer/ConsumerManager.java`、`consumer/Consumer.java` | 位点管理与实体 |
| `ChangelogDeletion` | `operation/ChangelogDeletion.java` | changelog 清理（两路径） |
| `ExpireConfig.changelogDecoupled` | `paimon-api/.../options/ExpireConfig.java:54` | 生命周期解耦 |
| `AuditLogTable` / `BinlogTable` | `table/system/AuditLogTable.java:90`、`BinlogTable.java` | CDC 系统表 |
