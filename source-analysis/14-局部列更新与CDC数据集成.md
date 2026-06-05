# Apache Paimon 局部列更新与 CDC 数据集成深度分析

> **版本**：1.5-SNAPSHOT　**源码模块**：`paimon-flink/paimon-flink-cdc`（CDC 链路）、`paimon-core`（partial-update 应用、跨分区索引）　**核对日期**：2026-06

**一句话定位**：本文讲两件强绑定的事——**用 partial-update merge engine 把"多源、乱序、部分列"的更新在写入侧自动合并成宽表**，以及 **CDC ingestion 链路如何把上游 binlog 的数据 + schema 变更持续灌入 Paimon 并自动演进**。它们合起来回答的是"如何把一张实时宽表喂饱"。

读完本文你应能回答：① partial-update 在生产里到底怎么配（sequence-group、聚合、删除策略），配错会丢哪些数据；② CDC 同步作业 `build()` 后在 Flink 里展开成什么拓扑、schema 变更走哪条边、为什么必须并行度 1；③ `RichCdcMultiplexRecord → CdcRecord` 这条数据流为什么用 `Map<String,String>` 当中间态、库级同步靠什么把多表路由到各自的 Paimon 表；④ CDC 自动 schema 演进只允许哪些类型变更、`canConvert` 的三态（CONVERT/IGNORE/EXCEPTION）各意味着什么；⑤ 分库分表（sharding）如何被合并成一张表、新表如何被动态发现；⑥ 跨分区 upsert 为什么必须动态 bucket + 单写入，不同 merge engine 的迁移策略差在哪。

> 阅读约定：本文每个机制按"① 要解决什么问题 → ② 设计原理与取舍 → ③ 关键源码（精选片段 + `路径:行号`）→ ④ 风险/陷阱/边界 → ⑤ 收益与代价"组织。源码行号以本次核对为准；与旧稿不符处用 `（已修正）` 标注。
>
> **交叉引用约定**：partial-update / aggregation / sequence-group / first-row 的 **merge engine 内部实现由 `08` 主讲**，本文只讲它们的应用、配置与 CDC 链路，原理详见 `08`；**Changelog 四种 producer 的产生机制由 `24` 主讲**（merge engine 视角另见 `08` §8），本文只点到选型；**Schema 演进的 SchemaChange/SchemaManager 乐观锁机制由 `22` 主讲**，本文只讲 CDC 场景特有的自动演进；Flink 写入链路详见 `15`，本文是 `05`/`15` 引用的 CDC 主讲文档。

---

## 目录

- [1. 快速理解（核心问题 / 概念速查 / 高频陷阱）](#1-快速理解核心问题--概念速查--高频陷阱)
  - [1.1 核心问题：把"喂宽表"这件事下沉到存储层](#11-核心问题把喂宽表这件事下沉到存储层)
  - [1.2 核心概念速查表](#12-核心概念速查表)
  - [1.3 高频生产陷阱](#13-高频生产陷阱)
- [2. Partial Update 局部列更新（应用视角）](#2-partial-update-局部列更新应用视角)
  - [2.1 默认语义：非空字段覆盖](#21-默认语义非空字段覆盖)
  - [2.2 Sequence Group：字段级乱序容忍](#22-sequence-group字段级乱序容忍)
  - [2.3 Partial Update + 聚合函数混用](#23-partial-update--聚合函数混用)
  - [2.4 删除处理的三种模式与互斥约束](#24-删除处理的三种模式与互斥约束)
  - [2.5 建表实战与配置清单](#25-建表实战与配置清单)
- [3. CDC 集成链路（主讲）](#3-cdc-集成链路主讲)
  - [3.1 模块全景与数据流](#31-模块全景与数据流)
  - [3.2 支持的 CDC 源与中转格式](#32-支持的-cdc-源与中转格式)
  - [3.3 同步作业的构建流程：从 Action 到 Flink 拓扑](#33-同步作业的构建流程从-action-到-flink-拓扑)
  - [3.4 CDC 数据模型：RichCdcMultiplexRecord 与 CdcRecord](#34-cdc-数据模型richcdcmultiplexrecord-与-cdcrecord)
  - [3.5 CDC Sink 拓扑：schema 变更为什么走旁路](#35-cdc-sink-拓扑schema-变更为什么走旁路)
- [4. CDC 自动 Schema 演进](#4-cdc-自动-schema-演进)
  - [4.1 UpdatedDataFieldsProcessFunction 处理流程](#41-updateddatafieldsprocessfunction-处理流程)
  - [4.2 类型拓宽规则：canConvert 的三态](#42-类型拓宽规则canconvert-的三态)
  - [4.3 容错：AddColumn 重复与并发提交](#43-容错addcolumn-重复与并发提交)
- [5. 库级同步：分库分表合并与动态发现](#5-库级同步分库分表合并与动态发现)
  - [5.1 COMBINED vs DIVIDED 两种 Sink 模式](#51-combined-vs-divided-两种-sink-模式)
  - [5.2 分库分表合并与表过滤](#52-分库分表合并与表过滤)
  - [5.3 新表动态发现](#53-新表动态发现)
  - [5.4 Changelog 选型（CDC 视角，详见 24）](#54-changelog-选型cdc-视角详见-24)
- [6. 跨分区更新（cross-partition upsert）](#6-跨分区更新cross-partition-upsert)
  - [6.1 问题与触发条件](#61-问题与触发条件)
  - [6.2 GlobalIndexAssigner 与 RocksDB 全局索引](#62-globalindexassigner-与-rocksdb-全局索引)
  - [6.3 ExistingProcessor：不同 merge engine 的迁移策略](#63-existingprocessor不同-merge-engine-的迁移策略)
  - [6.4 Bootstrap 索引重建](#64-bootstrap-索引重建)
- [7. CDC 同步实战配置](#7-cdc-同步实战配置)
  - [7.1 表级同步](#71-表级同步)
  - [7.2 库级同步（分库分表合并）](#72-库级同步分库分表合并)
  - [7.3 跨分区更新建表](#73-跨分区更新建表)
- [8. 设计决策总结](#8-设计决策总结)

---

## 1. 快速理解（核心问题 / 概念速查 / 高频陷阱）

### 1.1 核心问题：把"喂宽表"这件事下沉到存储层

**① 要解决什么问题**

一张实时宽表，它的列往往来自多个数据源、多个时间点：订单创建系统写状态、支付系统写金额、物流系统写运单号。传统做法是在 Flink 里用 `FULL OUTER JOIN` + 状态把这些源拼起来再覆盖写湖——JOIN 状态巨大、乱序难处理、加一列要改作业。

Paimon 的思路是**把"拼宽表"和"消费 binlog"都下沉到存储层**：

- **partial-update merge engine** 让每个源只写自己负责的列（其余留 null），存储层在合并同主键的多条版本时"只更新非空字段"，自动拼出宽表；
- **CDC ingestion 链路**把上游数据库的 binlog（含 schema 变更）持续灌进来，新增列自动 `ALTER TABLE`，分库分表自动合并成一张表。

**② 设计哲学一句话**：**把多源合并、乱序容忍、schema 演进这些"脏活"从计算层搬到写入/合并时刻，用声明式配置替代有状态作业。** 代价是只能用 Paimon 内置的合并策略、调试是"存储层黑盒"，收益是架构简化、Flink 无状态、加列零改动。

**③ 两条主线如何咬合**（贯穿全文）：

```
上游 MySQL/Postgres/Kafka(Debezium...)
   │  binlog / 消息
   ▼
[CDC Source] → CdcSourceRecord
   │  RecordParser.flatMap（§3.1）
   ▼
RichCdcMultiplexRecord（携带 库名+表名+schema+数据，§3.4）
   │  CdcParsingProcessFunction：数据走主流，schema 变更走旁路（§3.5）
   ├──旁路──▶ UpdatedDataFieldsProcessFunction（并行度=1）→ 自动 ALTER TABLE（§4）
   ▼
CdcRecord（纯数据 Map<String,String>）
   │  按 bucketMode 路由 + 类型转换
   ▼
Paimon Writer ── 写入主键表，merge-engine=partial-update（§2）
   │  同一主键多版本，合并时"只更非空列" + sequence-group 控乱序
   ▼
宽表（读出来已是合并后的最终值）
```

partial-update 是"合并语义"，CDC 是"喂数据 + 维护 schema"；二者最常见的组合就是 `mysql_sync_database ... --table-conf merge-engine=partial-update`（§7）。

### 1.2 核心概念速查表

| 概念 | 一句话定义 | 关键源码 |
|------|-----------|---------|
| **partial-update** | merge engine 之一，合并同主键多版本时只更新非空字段；内部实现详见 08 | `PartialUpdateMergeFunction.java:65` |
| **sequence-group** | 为一组字段指定独立的版本号字段，实现字段级乱序容忍；详见 08 §7 | `PartialUpdateMergeFunction.java:67`（`SEQUENCE_GROUP`）|
| **FieldAggregator** | 字段级聚合器（sum/max/collect…），partial-update 可混用；详见 08 §6 | `aggregate/FieldAggregator.java` |
| **CdcRecord** | CDC 单条数据变更：`RowKind + Map<String,String>`（字段名→字符串值） | `sink/cdc/CdcRecord.java:32` |
| **RichCdcMultiplexRecord** | 携带 库名/表名/CdcSchema 的多表 CDC 记录，库级同步路由用 | `sink/cdc/RichCdcMultiplexRecord.java:30` |
| **SynchronizationActionBase** | 所有 sync 作业的基类，`build()` 拼出 Flink 拓扑 | `action/cdc/SynchronizationActionBase.java:60` |
| **CdcSinkBuilder** | 把解析后的流接成 Paimon Sink，并挂上 schema 演进旁路 | `sink/cdc/CdcSinkBuilder.java:47` |
| **UpdatedDataFieldsProcessFunction** | 消费 schema 变更事件、自动 ALTER TABLE（必须并行度 1） | `sink/cdc/UpdatedDataFieldsProcessFunction.java:43` |
| **canConvert** | 判断 CDC 类型变更能否安全应用，返回 CONVERT/IGNORE/EXCEPTION | `sink/cdc/UpdatedDataFieldsProcessFunctionBase.java:173` |
| **GlobalIndexAssigner** | 跨分区 upsert 的全局主键→(分区,bucket) 索引，基于 RocksDB | `crosspartition/GlobalIndexAssigner.java` |
| **ExistingProcessor** | 主键已存在于别的分区时的处理策略，按 merge engine 分流 | `crosspartition/ExistingProcessor.java:59` |

> **路径修正**：`PartialUpdateMergeFunction` 共 717 行（旧稿标 720，已修正）；`ChangelogProducer` 枚举现位于 `CoreOptions.java:4051`（旧稿标 3948，已修正）；CDC 源类型枚举 `SyncJobHandler.SourceType` 在 `SyncJobHandler.java:253`（旧稿标 254，已修正）。

### 1.3 高频生产陷阱

| # | 陷阱 | 后果 | 规避 |
|---|------|------|------|
| 1 | partial-update 不配删除策略就收到 DELETE | 直接抛 `IllegalArgumentException` | 必须三选一：`ignore-delete` / `partial-update.remove-record-on-delete` / `partial-update.remove-record-on-sequence-group`（§2.4）|
| 2 | 聚合函数不配 sequence-group | 乱序数据导致重复 / 错误累加 | 除 `last_non_null_value` 外都必须在 sequence-group 保护下（§2.3，原理详见 08）|
| 3 | 非主键列设成 NOT NULL | 输入该列为 null 时抛异常 | partial-update 下非主键列尽量 nullable |
| 4 | 有数据的表加 NOT NULL 列 | 旧文件无此列、读时填 null 失败 | 新列必须 nullable（§4，约束详见 22）|
| 5 | CDC 同步保留源库 NOT NULL | 后续 schema 演进 / 主键场景报错 | 加 `--type-mapping to-nullable`（§7）|
| 6 | 跨分区 upsert 用固定 bucket / 多作业并发写 | 不生效 / 索引错乱 | 必须 `bucket=-1` 且单写入（§6.1）|
| 7 | CDC schema 演进算子并行度 >1 | 并发改 schema 冲突、频繁乐观锁重试 | 框架强制 `setParallelism(1)`，不要绕过（§3.5/§4）|
| 8 | sequence-group 的序列号字段全为 null | 该 group 所有字段被跳过、看似"没更新" | 保证序列号字段有值；理解此跳过逻辑（§2.2）|
| 9 | 误把 CDC schema 自动演进当作"任意类型可改" | 缩窄类型被静默 IGNORE，或不兼容直接 EXCEPTION 让作业失败 | 理解 `canConvert` 三态（§4.2）|

---

## 2. Partial Update 局部列更新（应用视角）

> 本节聚焦 **怎么配、配错会怎样**。`PartialUpdateMergeFunction` 的 `updateNonNullFields` / `updateWithSequenceGroup` / `retractWithSequenceGroup` 等内部合并逻辑、聚合函数清单与 retract 支持，**merge engine 实现原理由 08 主讲**（§3.2、§6、§7），此处不重复展开，只给应用结论与配置。

### 2.1 默认语义：非空字段覆盖

**① 要解决什么问题**：多源各写一部分列，不能让后到的源把别人的列覆盖成 null。

**② 设计原理**：merge engine 配 `'merge-engine' = 'partial-update'` 后，合并同主键的多条记录时遵循一条简单规则——**新记录里非空的字段覆盖旧值，为 null 的字段保留旧值**（内部即 `PartialUpdateMergeFunction.updateNonNullFields`，源码详见 08 §3.2）。这是无 sequence-group 时的默认分支，依赖记录按全局 sequenceNumber 有序到达，没有字段级乱序保护。

```sql
CREATE TABLE user_profile (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    name STRING, email STRING, phone STRING, address STRING
) WITH ('merge-engine' = 'partial-update', 'ignore-delete' = 'true');

INSERT INTO user_profile VALUES (1, 'Alice', 'alice@x.com', NULL, NULL);  -- 注册系统
INSERT INTO user_profile VALUES (1, NULL, NULL, '138...', NULL);          -- 手机绑定
-- 合并结果: (1, 'Alice', 'alice@x.com', '138...', NULL)  ← name/email 保留
```

**④ 风险**：① 非主键列若声明 NOT NULL，输入该列为 null 会抛异常（陷阱 3）；② 无 sequence-group 时只靠全局 sequenceNumber 排序，跨源乱序无法纠正（陷阱 8 的反面——见 §2.2）。

**⑤ 收益与代价**：零额外配置、语义直观；代价是无乱序保护、必须保证非主键列 nullable。

### 2.2 Sequence Group：字段级乱序容忍

**① 要解决什么问题**：无 sequence-group 时所有字段共享一个全局序列号，无法表达"支付字段用 payment_ts 排序、物流字段用 shipping_ts 排序"这种字段级版本控制。一个源的乱序回放会污染另一个源的列。

**② 设计原理与取舍**：`fields.<seq_field>.sequence-group = <f1>,<f2>,...` 为一组 value 字段绑定一个序列号字段。合并时**只有当新记录的 seq_field ≥ 当前行的 seq_field，该 group 的字段才更新**；更小则忽略（乱序丢弃）。多个 group 各自独立比较，互不影响（内部 `updateWithSequenceGroup`，详见 08 §7.3）。

| 维度 | 无 sequence-group | 有 sequence-group |
|------|------------------|-------------------|
| 版本控制粒度 | 全表一个全局 seq | 每个字段组一个 seq 字段 |
| 多源乱序 | 不能区分，后到覆盖 | 各组独立按各自 seq 取舍 |
| 配置成本 | 零 | 每组一行 `sequence-group` |
| 适用 | 单源、有序 | 多源、乱序（CDC 多表汇聚典型） |

```sql
WITH (
  'merge-engine' = 'partial-update',
  'fields.payment_ts.sequence-group'  = 'order_status,payment_amount',
  'fields.shipping_ts.sequence-group' = 'shipping_address,tracking_no'
)
-- payment_ts=100 写入支付；shipping_ts=200 写入物流；payment_ts=50 的乱序补偿被忽略
```

**④ 风险**：**某 group 的序列号字段全为 null 时，整组字段被跳过不更新**（陷阱 8）——这是为了避免用 null 序列号覆盖有效数据，但容易被误读成"没生效"。务必保证序列号字段有值。

**⑤ 收益与代价**：精确的字段级乱序容忍，多源可安全并行写；代价是配置变复杂、需要为每组维护一个单调递增的序列号字段。

### 2.3 Partial Update + 聚合函数混用

**① 要解决什么问题**：partial-update 默认是"覆盖非空"，但有些列要的是"累加 / 取最大 / 收集成数组"——例如总消费金额、最大单笔、标签集合。

**② 设计原理**：partial-update 允许**对部分字段配聚合函数、其余字段仍走非空覆盖**——这是它与 aggregation 引擎的关键区别（aggregation 要求所有非主键列都配聚合函数）。字段是否聚合由 `getAggFuncName` 按优先级判定（主键→`primary-key`、序列字段→不聚合、用户配置 `fields.<f>.aggregate-function`、默认 `fields.default-aggregate-function`），详见 08 §6.2。

**关键约束**：**除 `last_non_null_value` 外，所有聚合函数都必须在 sequence-group 保护下使用**，否则建表报错。原因：乱序到达时，无版本控制的累加（如 sum）无法区分"正常增量"与"乱序重复"，会算错；而 `last_non_null_value` 本质就是 partial-update 默认行为，天然适配乱序。

聚合函数的完整清单、哪些支持 retract（sum 减、collect 移除、max/min 不支持）、`ignore-retract` 装饰器——**详见 08 §6.2 / §6.3**，本文不重复列举。

```sql
WITH (
  'merge-engine' = 'partial-update',
  'fields.action_ts.sequence-group' = 'latest_action,action_count,total_amount,max_single',
  'fields.action_count.aggregate-function' = 'sum',
  'fields.total_amount.aggregate-function' = 'sum',
  'fields.max_single.aggregate-function'   = 'max'
)  -- latest_action 走非空覆盖，其余三列分别 sum/sum/max
```

**④ 风险**：① 聚合列漏配 sequence-group → 建表/运行报错；② 用了不支持 retract 的聚合函数（如 max/min）又收到 DELETE → 报错，需配 `fields.<f>.ignore-retract=true`（详见 08 §6.3）。

**⑤ 收益与代价**：一张表里"宽表拼接 + 预聚合"一次写完；代价是聚合不可逆函数遇 retract 需额外处理，调试黑盒。

### 2.4 删除处理的三种模式与互斥约束

**① 要解决什么问题**：partial-update 默认不知道"收到一条 DELETE 该怎么办"——是丢弃？删整行？还是只清某些列？为避免静默丢数据，**默认收到 retract（DELETE / UPDATE_BEFORE）直接抛异常**，强制用户显式选策略。

**② 三种模式对比**：

| 模式 | 配置 | 行为 | 适用 |
|------|------|------|------|
| 忽略删除 | `ignore-delete = true` | 丢弃 retract 记录 | 上游软删除、不需要物理删 |
| 整行删除 | `partial-update.remove-record-on-delete = true` | 收到 `RowKind.DELETE` 删整行（`UPDATE_BEFORE` 不触发）| 需要精确删除整行 |
| 按序列组删除 | `partial-update.remove-record-on-sequence-group = <seq_f1>,...` | 仅指定序列组的 DELETE 才删整行 | 多源中只有某主源的删除应删整行 |

**互斥约束**（建表时校验，详见 08 §7.2）：
- `ignore-delete` 与 `remove-record-on-delete` 不能同时启用；
- `ignore-delete` 与 `remove-record-on-sequence-group` 不能同时启用；
- `remove-record-on-delete` 与 `sequence-group` 不能同时使用（后者需要更细粒度删除控制）；
- `remove-record-on-sequence-group` 引用的字段必须是某个 sequence-group 的序列号字段。

> 这些配置项的 key（`ignore-delete`、`partial-update.remove-record-on-delete`、`partial-update.remove-record-on-sequence-group`）均已在 `CoreOptions.java` 核对（分别约 585 / 978 / 986 行）。

**④ 风险**：CDC 场景最常踩——MySQL 删一行，partial-update 表若没配任何删除策略，作业直接挂。库级同步多表共用配置时尤其要注意。

**⑤ 收益与代价**：删除语义显式可控、不会静默丢数据；代价是必须理解三种模式差异，配错（如同时配互斥项）建表即失败。

### 2.5 建表实战与配置清单

多源汇聚用户画像表（三源、三 sequence-group、混合聚合、主源删除触发整行删）：

```sql
CREATE TABLE user_profile (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    name STRING, gender STRING, register_ts BIGINT,           -- 注册源
    total_spent DECIMAL(10,2), order_count INT, payment_ts BIGINT,  -- 支付源
    last_login TIMESTAMP, login_count INT, behavior_ts BIGINT       -- 行为源
) WITH (
    'merge-engine' = 'partial-update',
    'fields.register_ts.sequence-group' = 'name,gender',
    'fields.payment_ts.sequence-group'  = 'total_spent,order_count',
    'fields.behavior_ts.sequence-group' = 'last_login,login_count',
    'fields.total_spent.aggregate-function' = 'sum',
    'fields.order_count.aggregate-function' = 'sum',
    'fields.login_count.aggregate-function' = 'sum',
    'partial-update.remove-record-on-sequence-group' = 'register_ts'
);
```

配置速查：

| 配置项 | 作用 |
|--------|------|
| `merge-engine=partial-update` | 启用局部列更新 |
| `fields.<seq>.sequence-group=<cols>` | 为一组列绑定版本字段 |
| `fields.<col>.aggregate-function=<fn>` | 对某列做聚合（需 sequence-group 保护）|
| `fields.default-aggregate-function=<fn>` | 全表默认聚合策略 |
| `ignore-delete` / `partial-update.remove-record-on-delete` / `partial-update.remove-record-on-sequence-group` | 三选一的删除策略 |

## 3. CDC 集成链路（主讲）

### 3.1 模块全景与数据流

**① 要解决什么问题**：把上游 OLTP 数据库的持续变更（INSERT/UPDATE/DELETE + 偶发的 schema 变更）实时、自动、可演进地灌进 Paimon——不写一行解析代码、不为加一列重启作业、分库分表自动归一。

**② 模块与继承结构**：CDC 集成代码位于 `paimon-flink/paimon-flink-cdc`，入口是一组 Action（命令行子命令）：

```
SynchronizationActionBase（基类，build() 拼 Flink 拓扑）
  ├── SyncTableActionBase（表级：一个源表 → 一张 Paimon 表）
  │     └── MySql / Postgres / Kafka / Pulsar / MongoDB SyncTableAction
  └── SyncDatabaseActionBase（库级：整库/分库分表 → 多张 Paimon 表）
        └── MySql / Postgres / Kafka / Pulsar / MongoDB SyncDatabaseAction
```

**③ 数据流转链**（贯穿 §3.3–§3.5）：

```
CDC Source ─▶ CdcSourceRecord ─(RecordParser.flatMap)─▶ RichCdcMultiplexRecord
                                                          │ 携带 库名+表名+CdcSchema+CdcRecord
        ┌─────────────────────────────────────────────────┘
        ▼
  CdcParsingProcessFunction（表级）/ CdcDynamicTableParsingProcessFunction（库级）
        ├── 主流：CdcRecord（纯数据）────────▶ 类型转换 ─▶ Paimon Writer
        └── 旁路：CdcSchema（schema 变更）───▶ UpdatedDataFieldsProcessFunction（并行度=1）─▶ ALTER TABLE
```

**④ 设计要点**：解析（RecordParser）与下沉（Sink）解耦，schema 变更与数据走**两条边**（§3.5）；多表用同一条 DataStream 复用（`RichCdcMultiplexRecord` 携带表名做路由）。

### 3.2 支持的 CDC 源与中转格式

**源码**：`SyncJobHandler.SourceType` 枚举（`SyncJobHandler.java:253`，已修正——旧稿标 254）：

| CDC 源 | SourceType | 必需参数（节选）| 解析器 |
|--------|-----------|----------------|--------|
| MySQL | `MYSQL` | hostname, username, password, database-name | `MySqlRecordParser` |
| PostgreSQL | `POSTGRES` | hostname, username, password, database-name, schema-name, slot-name | `PostgresRecordParser` |
| Kafka | `KAFKA` | value.format, properties.bootstrap.servers, topic/topic-pattern | `DataFormat.createParser()` |
| Pulsar | `PULSAR` | value.format, pulsar.service-url, pulsar.subscription.name, topic/topic-pattern | `DataFormat.createParser()` |
| MongoDB | `MONGODB` | hosts, database | `MongoDBRecordParser` |

**为什么支持 Kafka/Pulsar 这种"非 CDC 源"？** 它们本身不产生 binlog，而是作为 CDC 的**中转**：Debezium/Canal/Maxwell 把数据库变更写进 Kafka，Paimon 再消费。通过 `value.format`（`debezium-json` / `canal-json` / `maxwell-json` / `ogg-json` / `dms-json` / `aliyun-json` 等，对应 `format/` 包下各 `*RecordParser`）指定中转格式。

**收益**：采集与下沉解耦——采集端可独立扩缩容、多个下游共享同一份 CDC 流；代价是多一跳消息队列、端到端延迟略增。

### 3.3 同步作业的构建流程：从 Action 到 Flink 拓扑

**① 统一骨架**（`SynchronizationActionBase.build()`，`SynchronizationActionBase.java:130`）：

```
build():
  1. syncJobHandler.checkRequiredOption()      // 校验源必需参数
  2. catalog.createDatabase(database, true)     // 建目标库（幂等）
  3. beforeBuildingSourceSink()                 // 子类钩子：准备表/schema
  4. buildDataStreamSource(buildSource())        // 构建 CDC Source（带 watermark 策略）
       .flatMap(recordParse())                   // 解析为 RichCdcMultiplexRecord
  5. buildSink(input, buildEventParserFactory()) // 子类构建 Paimon Sink
```

`run()` 还会在用户没开 checkpoint 时兜底 `enableCheckpointing(3min)`（`SynchronizationActionBase.java:239`）。

**② 表级 `beforeBuildingSourceSink`**（`SyncTableActionBase.java:114`）——核心是"表在不在"的分流：

```
try 取已存在的表:
  成功 → alterTableOptions() 合并动态选项
       → 能取到源 schema：assertSchemaCompatible() 校验兼容
       → 取不到（SchemaRetrievalException）：用已有表 schema 构建计算列 + checkConstraints()
失败（TableNotExistException）:
  → retrieveSchema() 从 CDC 源拉 schema
  → buildPaimonSchema() + catalog.createTable()  // 首次自动建表
```

**为什么要先判断表是否存在？** 同一条命令要同时支持两种场景：**首次同步**（用源 schema 自动建表）与**增量/重启同步**（复用已有表，只校验兼容性）。`alterTableOptions` 会移除 immutable 选项与值未变的选项，只对真正变化的动态选项发 `SchemaChange.setOption`（`SynchronizationActionBase.java:200`）。

**③ 表级 Sink** 由 `CdcSinkBuilder` 构建（`SyncTableActionBase.java:169`）；库级 Sink 由 `FlinkCdcSyncDatabaseSinkBuilder` 构建（§5）。

**④ 风险**：`assertSchemaCompatible` 不通过会让作业起不来——常因源库改了不兼容的类型，或 `--partition-keys`/`--primary-keys` 与已有表不一致（`checkConstraints` 会报"not equal to the existed table"）。

### 3.4 CDC 数据模型：RichCdcMultiplexRecord 与 CdcRecord

**① CdcRecord**（`sink/cdc/CdcRecord.java:32`）——最小数据变更单元：

```java
public class CdcRecord implements Serializable {
    private RowKind kind;                    // INSERT / UPDATE_BEFORE / UPDATE_AFTER / DELETE
    private final Map<String, String> data;  // 字段名 → 字符串值
}
```

**为什么用 `Map<String,String>` 而非强类型行？** CDC 数据来自十几种格式（Debezium/Canal/Maxwell…），字段集合还会随 schema 演进变化。用"字段名→字符串"的松散表示能**统一所有格式、容忍字段增减**，把真正的类型转换推迟到写入 Paimon 表那一刻（按目标表 schema 转）。代价是中间态无类型检查、字符串化有微小开销。

**② RichCdcMultiplexRecord**（`sink/cdc/RichCdcMultiplexRecord.java:30`）——给 `CdcRecord` 套上"我是谁"的元信息：

```java
public class RichCdcMultiplexRecord implements Serializable {
    @Nullable private final String databaseName;  // 来源库
    @Nullable private final String tableName;      // 来源表
    private final CdcSchema cdcSchema;             // 字段列表 + 主键 + 注释
    private final CdcRecord cdcRecord;             // 数据本身
}
```

**为什么需要它？** 库级同步时一条 DataStream 里混着多张表的记录，下游必须靠 `databaseName/tableName` 把每条路由到正确的 Paimon 表，靠 `cdcSchema` 触发对应表的 schema 演进。`buildSchema()` 能直接据此建表，`toRichCdcRecord()` 可降级为单表记录。

**③ 数据流（带类型）**：`CdcSourceRecord → RichCdcMultiplexRecord（含 schema）→ CdcRecord（纯数据）→ 写入时按目标 schema 类型转换`。

### 3.5 CDC Sink 拓扑：schema 变更为什么走旁路

`CdcSinkBuilder.build()`（`sink/cdc/CdcSinkBuilder.java:99`）把输入流接成下面的拓扑——这是理解 CDC 自动演进的关键：

```
input ──forward──▶ CdcParsingProcessFunction
                    ├── 主输出：CdcRecord（数据）
                    └── 旁路 SideOutput(SCHEMA_CHANGE_OUTPUT_TAG)：CdcSchema（schema 变更）
                                          │
                                          ▼
                         UpdatedDataFieldsProcessFunction
                         （setParallelism(1) + setMaxParallelism(1)）→ ALTER TABLE
   主输出 CdcRecord ─▶ CaseSensitiveUtils.cdcRecordConvert ─▶ 按 bucketMode 分发：
        HASH_FIXED  → partition + CdcFixedBucketSink
        HASH_DYNAMIC→ CdcDynamicBucketSink
        POSTPONE    → CdcFixedBucketSink（postpone channel computer）
        UNAWARE     → CdcAppendTableSink（rebalance 以保证 schema change 不死循环）
```

**① 为什么 schema 变更要单独拉一条旁路？** schema 变更（DDL）和数据变更（DML）频率、并行度需求完全不同：数据要高并行吞吐，DDL 必须**串行**（否则两个并行 task 同时给同一张表加列会冲突）。Flink 的 SideOutput 正好把两类事件拆到两条边，各自设并行度。

**② 为什么 `UpdatedDataFieldsProcessFunction` 强制并行度 1？** 见代码 `schemaChangeProcessFunction.getTransformation().setParallelism(1)`（`CdcSinkBuilder.java:129`）。schema 变更必须单线程串行执行——`SchemaManager` 内部虽有乐观锁重试兜底（详见 22），但并行度 1 从源头消除并发冲突、避免无意义的重试风暴。这也是陷阱 7 的根因：绕过它（强行调并行度）会引发频繁冲突。

**③ Append 表为什么要 `rebalance()`？** 注释明确："rebalance it to make sure schema change work to avoid infinite loop"（`CdcSinkBuilder.java:172`）——unaware-bucket 的 append 表若不 rebalance，schema 变更与写入的反馈可能形成死循环。

**`RichCdcSinkBuilder`**（`sink/cdc/RichCdcSinkBuilder.java:37`，`@Public` / `@since 0.8`）是给用户直接用 DataStream API 接 `RichCdcRecord` 的薄封装，内部仍委托 `CdcSinkBuilder`，只是把 parserFactory 固定成 `RichEventParser::new`。

**④ 收益与代价**：DML/DDL 分离让"数据高吞吐 + DDL 强串行"同时成立；代价是 schema 变更算子成为单点（并行度 1），DDL 极频繁时它可能成瓶颈——但 DDL 本就罕见，可接受。

---

## 4. CDC 自动 Schema 演进

> SchemaChange 的 12 种类型、`SchemaManager` 的乐观锁循环、各类约束（分区键/主键不可改、新列必须 nullable 等）由 **22 主讲**。本节只讲 **CDC 链路特有的"自动"演进**——即 `UpdatedDataFieldsProcessFunction` 如何从 CDC 流里识别 schema 变化、转成 `SchemaChange` 并自动应用。

### 4.1 UpdatedDataFieldsProcessFunction 处理流程

**① 要解决什么问题**：上游数据库随时可能 `ALTER TABLE`（加列、改类型、改注释），CDC 流里会夹带新的 `CdcSchema`。下游必须自动把这些变化翻译成 Paimon 的 `SchemaChange` 并应用——否则新列的数据无处落、作业报错。

**② 处理流程**（`UpdatedDataFieldsProcessFunction.processElement`，`UpdatedDataFieldsProcessFunction.java:63`）：

```
收到一个 CdcSchema（CDC 源当前 schema）:
  1. actualUpdatedDataFields(): 过滤掉"已存在且未变"的字段，得到真正变化的字段
     （靠本地缓存 latestFields 做差集，UpdatedDataFieldsProcessFunctionBase.java:412）
  2. 若无变化且无注释变更 → 直接 return（绝大多数记录走这里，零开销）
  3. extractSchemaChanges(): 把差异转成 SchemaChange 列表（Base.java:348）
        - 新字段        → SchemaChange.addColumn
        - 同名但类型不同 → 生成（嵌套）类型更新 + 可选注释更新
        - 注释变化      → updateColumnComment / updateComment
  4. applySchemaChange(): 逐个调用 catalog.alterTable() 应用
  5. updateLatestFields(): 用提交后的最新 schema 刷新本地缓存
```

**关键设计**：第 1 步用本地 `latestFields` 缓存挡掉绝大多数"schema 没变"的记录——CDC 流里每条记录都带 schema，但 99.99% 都没变，差集判断让这个算子几乎零成本。注意第 5 步**不能用 `actualUpdatedDataFields` 来刷新缓存**（源码注释明确），因为存在非 AddColumn 场景，否则已存在字段会无法再被修改。

### 4.2 类型拓宽规则：canConvert 的三态

**① 要解决什么问题**：上游把 `INT` 改成 `BIGINT` 应该自动跟进；但把 `BIGINT` 改成 `INT`（缩窄）会丢数据，绝不能跟。需要一套"只拓宽不缩窄"的判定。

**② 设计原理**：`canConvert(oldType, newType, typeMapping)`（`UpdatedDataFieldsProcessFunctionBase.java:173`，已修正——旧稿标 173-264）返回**三态枚举** `ConvertAction`，这是旧稿漏掉的关键细节：

| 返回值 | 含义 | 后续动作 |
|--------|------|---------|
| `CONVERT` | 新类型是旧类型的安全拓宽 | 执行 `catalog.alterTable` |
| `IGNORE` | 同类型族但新类型精度/长度更小（缩窄）| **静默忽略**，不改 schema、不报错 |
| `EXCEPTION` | 跨不兼容类型族 | 抛 `UnsupportedOperationException`，作业失败 |

类型族内的拓宽方向（源码 `STRING_TYPES`/`INTEGER_TYPES`… 顺序，`Base.java:67`）：

| 类型族 | 拓宽方向 | 缩窄时 |
|--------|---------|--------|
| STRING | CHAR → VARCHAR（长度只增）| 长度变小 → IGNORE |
| BINARY | BINARY → VARBINARY（长度只增）| IGNORE |
| INTEGER | TINYINT→SMALLINT→INTEGER→BIGINT | 反向 → IGNORE |
| FLOATING | FLOAT → DOUBLE | IGNORE |
| DECIMAL | precision/scale 只增 | 都更小 → IGNORE |
| TIMESTAMP | precision 只增 | IGNORE |
| 跨族转 STRING | 任意 → STRING（需 `to-string` TypeMapping）| —— |

ARRAY/MAP/MULTISET/ROW 递归判定（map 的 key 类型不允许演进，因为 hashcode 会变，`Base.java:284`）。

**③ 风险（旧稿误导处）**：缩窄类型不是"报错"而是 **IGNORE（静默不变）**——这意味着上游缩窄了类型，Paimon 表的列类型**不会同步**，但作业照常跑。运维时若发现"上游改了类型 Paimon 没跟上"，多半是命中了 IGNORE 分支，属预期行为，不是 bug。真正会让作业挂的是 `EXCEPTION`（跨族不兼容，如 STRING→INT）。

**④ 为什么只拓宽不缩窄？** 缩窄会丢数据/溢出（BIGINT→INT 溢出、长 VARCHAR→短 VARCHAR 截断），存储层不能替用户做有损决策；拓宽则总是安全的。

### 4.3 容错：AddColumn 重复与并发提交

`applySchemaChange`（`UpdatedDataFieldsProcessFunctionBase.java:106`，已修正——旧稿标 106-161）的容错：

- **AddColumn 捕获 `ColumnAlreadyExistException`**（`Base.java:115`）：分库分表合并时，`order_0`/`order_1`/… 多个源表会**各自**发"加同一列"的事件，但它们指向同一张目标 Paimon 表，第二个之后必然"列已存在"。这里捕获并仅 debug 日志，不让作业失败——这是分库分表场景能正常工作的关键容错。
- **UpdateColumnType 走 `canConvert` 三态**（`Base.java:138`）：CONVERT 才 alter，EXCEPTION 抛错，IGNORE 跳过。
- 其余只接受 `UpdateColumnComment` / `UpdateComment`，未知类型直接抛 `UnsupportedOperationException`——CDC 自动演进**只支持加列、拓宽类型、改注释**，不会自动删列/改主键（那些有损或破坏性，留给人工）。

**库级**用 `MultiTableUpdatedDataFieldsProcessFunction`（同包），逻辑相同但按 `Identifier` 分表维护各自的 `latestFields`。

---

## 5. 库级同步：分库分表合并与动态发现

### 5.1 COMBINED vs DIVIDED 两种 Sink 模式

**① 要解决什么问题**：库级同步要把整库（可能上百张表）灌进 Paimon。是给每张表起一条独立 sink 链路（简单但表多时算子爆炸、无法发现新表），还是所有表复用一条多路复用链路（复杂但能动态扩表）？

**② 两种模式**（`SyncDatabaseActionBase.buildSink` → `FlinkCdcSyncDatabaseSinkBuilder`，`SyncDatabaseActionBase.java:241`；模式判定 `FlinkCdcSyncDatabaseSinkBuilder.java:156`）：

| 维度 | DIVIDED | COMBINED |
|------|---------|----------|
| 拓扑 | 每张表一条独立 sink | 所有表共用一条多路复用 sink |
| 新表发现 | 不支持（需重启作业）| **支持**（运行中自动接管新表）|
| 算子数 | 随表数线性增长 | 固定 |
| 适用 | 表数少且稳定 | 表多 / 会动态增表 / 分库分表 |

库级的 `EventParserFactory` 是 `RichCdcMultiplexRecordEventParser`（`SyncDatabaseActionBase.java:228`），它内部持有 include/exclude 正则、`TableNameConverter`、已建表集合 `createdTables`，负责"这条记录属于哪张目标表、要不要同步"。

### 5.2 分库分表合并与表过滤

**① 分库分表合并**（`--merge-shards true`，默认）：业务把数据按 hash 散到 `db_0..db_n`、`order_0..order_99`，数据湖侧要归一成一张 `order` 表分析。`TableNameConverter`（`SyncDatabaseActionBase.java:213` 构造）负责表名归一：去尾部数字后缀合并同名表、套 `--table-prefix/--table-suffix`、`--db-prefix/--db-suffix`、自定义 `--table-mapping`。合并后多个分表的 schema 演进汇聚到同一张目标表——这正是 §4.3 "AddColumn 容忍重复列"存在的原因。

**② 表/库过滤**（正则）：

```
--including-tables 'order.*|user.*'   --excluding-tables 'tmp_.*'
--including-dbs    'db_prod_.*'        --excluding-dbs    'db_test_.*'
```

正则化的 include/exclude 在 `SyncDatabaseActionBase.java:208` 编译成 `Pattern`，交给 EventParser 在运行时逐表匹配——灵活匹配命名规范的库表。

### 5.3 新表动态发现

COMBINED 模式下，`CdcDynamicTableParsingProcessFunction`（`sink/cdc/CdcDynamicTableParsingProcessFunction.java:46`）处理"运行中冒出的新表"，它有**两个 side output**：

- `DYNAMIC_OUTPUT_TAG`（`CdcMultiplexRecord` 数据）→ 路由到多路复用 sink；
- `DYNAMIC_SCHEMA_CHANGE_OUTPUT_TAG`（`Tuple2<Identifier, CdcSchema>`）→ 路由到 `MultiTableUpdatedDataFieldsProcessFunction` 做 schema 演进（含建新表）。

**流程**：已知表数据直接路由到对应 sink；命中过滤规则的新表，其数据走 `DYNAMIC_OUTPUT_TAG`、其 schema 走 `DYNAMIC_SCHEMA_CHANGE_OUTPUT_TAG` 触发自动建表，之后该表就被纳入正常同步。**收益**：上游加一张表无需重启 Flink 作业；**代价**：COMBINED 拓扑更复杂、多路复用 sink 是共享算子，单表热点可能相互影响。

---

### 5.4 Changelog 选型（CDC 视角，详见 24）

CDC 表通常需要向下游再吐 changelog。`changelog-producer` 的四种模式（`none`/`input`/`full-compaction`/`lookup`）及 first-row 的特殊机制，**产生原理由 24 主讲、merge engine 视角由 08 §8 主讲**，此处只给 CDC 选型结论：

| 场景 | 推荐 producer | 理由 |
|------|--------------|------|
| deduplicate 表、上游已是完整 CDC | `input` | 零计算、最低延迟（直接双写输入变更）|
| partial-update / aggregation 表 | `lookup` | input 不适用（输入≠最终变更）；lookup 比 full-compaction 延迟低 |
| 复杂引擎 + 下游低频消费、省资源 | `full-compaction` | 只在全量压缩时产 changelog，延迟高但开销集中 |
| 不需要下游消费 | `none` | 不产 changelog，省存储 |

> `ChangelogProducer` 枚举定义在 `CoreOptions.java:4051`（已修正——旧稿标 3948-3976）。注意 first-row 是 **merge engine** 不是 changelog-producer，它经 `FirstRowMergeFunctionWrapper` 仅产 INSERT changelog。

---

## 6. 跨分区更新（cross-partition upsert）

### 6.1 问题与触发条件

**① 要解决什么问题**：分区表里一条记录的**分区键变了**（订单 dt 从 `01-01` 改到 `01-02`），普通写入会在新分区凭空多一条、旧分区残留一条，造成主键重复。跨分区 upsert 要保证"同一主键全局唯一"，自动处理旧分区的清理或在旧位置原地更新。

**② 触发条件**：分区表 + **动态 bucket（`bucket=-1`）** + 主键**不包含全部分区字段**时，写入路径自动启用 `GlobalIndexAssigner`（判定逻辑与 BucketMode 选择详见 01 §10.3 / 08 §11.1）。约束：

- **必须 `bucket=-1`**（动态 bucket）——固定 bucket 模式不走 GlobalIndexAssigner；
- **必须单写入**——索引是各 task 的本地 RocksDB 状态，多作业并发写会各持一份不一致的索引（陷阱 6）；
- 索引可设 TTL（`cross-partition-upsert.index-ttl`），过期 key 再现按新记录处理。

> 本节聚焦"CDC/upsert 应用怎么用"。`GlobalIndexAssigner` 的内部机制（RocksDB 状态、bootstrap、SortMerge）**08 §11.2 亦有覆盖**，此处给应用结论 + 关键源码定位，两篇互为补充。

### 6.2 GlobalIndexAssigner 与 RocksDB 全局索引

**核心状态**（`GlobalIndexAssigner.java`）：一个 RocksDB value state 把主键映射到 (分区,bucket)：

```java
RocksDBValueState<InternalRow, PositiveIntInt> keyIndex;
// key: 主键(InternalRow)  →  value: (partitionId, bucketId)
```

`open` 时（`GlobalIndexAssigner.java:113`，已修正——旧稿标 113-180）创建带 TTL 的 `RocksDBStateFactory`。**为什么用 RocksDB 而非 Flink 堆状态？** 全局主键索引可能远超内存——RocksDB 落盘、按 key 高效查找、支持 bulk load 与 TTL，正合适。代价是引入本地磁盘状态、bootstrap 成本（§6.4）。

**记录处理流程**（`processInput`，`GlobalIndexAssigner.java:243`，已修正——旧稿标 243-272）：

```
每条输入:
  提取 partition + 主键 → partId = partMapping.index(partition)
  keyIndex.get(key):
    命中 (previousPartId, previousBucket):
      previousPartId == partId → 同分区，直接写旧 bucket（无跨分区）
      否则 → 跨分区：existingProcessor.processExists(...)，返回 true 才 processNewRecord
    未命中 → 新主键，processNewRecord 分配新 bucket
```

跨分区的真正分歧点在 `existingProcessor`——不同 merge engine 处理"旧分区残留"的策略完全不同（§6.3）。

### 6.3 ExistingProcessor：不同 merge engine 的迁移策略

**① 要解决什么问题**：主键 X 原本在分区 A，现在来了一条属于分区 B 的记录。旧分区 A 的那条怎么办？删掉？还是把新数据写回 A 不迁移？答案取决于 merge engine 的语义。

**② 策略分派**（`ExistingProcessor.create`，`ExistingProcessor.java:59`）：

| MergeEngine | ExistingProcessor | 行为 | 为什么 |
|-------------|------------------|------|--------|
| `DEDUPLICATE` | `DeleteExistingProcessor` | 向旧分区发一条 DELETE，再把新记录写新分区（真迁移）| 去重语义：同主键只留一条，旧的必须删 |
| `PARTIAL_UPDATE` / `AGGREGATE` | `UseOldExistingProcessor` | 把新记录的分区改回旧分区、写旧位置（**不迁移**）| 这两种引擎要在同一位置累积宽表/聚合状态，迁移会丢累积值 |
| `FIRST_ROW` | `SkipNewExistingProcessor` | 直接丢弃新记录 | first-row 只保留首条 |

`DeleteExistingProcessor` 还会 `bucketAssigner.decrement(previousPart, previousBucket)` 维护旧分区的桶计数后 `return true`（继续写新分区）；`UseOldExistingProcessor`/`SkipNewExistingProcessor` `return false`（不处理新分区）。

**④ 风险（陷阱 6 的本质）**：partial-update/aggregate 下"跨分区更新"其实**不迁移分区**——数据留在旧分区原地更新。如果业务以为"改了 dt 数据就搬到新分区了"，按新 dt 去查会查不到。这是设计使然（保聚合状态），不是 bug，但极易误解。

### 6.4 Bootstrap 索引重建

**① 为什么需要**：RocksDB 索引是本地状态，作业首启/重启/扩缩容时本地没有索引，必须从 Paimon 表历史数据重建。

**② 流程**（`GlobalIndexAssigner`）：

```
open(): bootstrap=true，建排序缓冲 bootstrapKeys + 记录缓冲 bootstrapRecords
bootstrapKey(value)（:182）: 从 IndexBootstrap 读历史主键→(分区,bucket)，序列化进排序缓冲
processInput(value)（bootstrap 期）: 新输入先缓存进 bootstrapRecords
endBoostrap(isEndInput)（:198）:
  a. bootstrapKeys 排序后用 RocksDB Bulk Load 批量灌（比逐条 put 快一个量级；失败提示"可能有重复主键"）
  b. isEndInput 且 RocksDB 空 → bulkLoadBootstrapRecords() 优化：跳过查找直接排序分配 bucket
  c. 否则逐条回放缓存记录走正常 processInput
```

`IndexBootstrap`（`crosspartition/IndexBootstrap.java`）扫历史主键时 `withProjection(keyProjection)` 只读主键列减 I/O，`withBucketFilter(bucket -> bucket % numAssigners == assignId)` 按 assigner 分片——多并行度下每个实例只加载自己负责的 bucket。

**⑤ 收益与代价**：跨分区 upsert 让上游"改分区键"无感、湖侧自动维护全局唯一；代价是 bootstrap（大表重启时扫全表主键，可能很慢，`cross-partition-upsert.bootstrap-parallelism` 调并行度）+ 单写入限制 + 本地 RocksDB 状态运维。

## 7. CDC 同步实战配置

### 7.1 表级同步

把单张 MySQL 表同步为一张 partial-update 主键表（`mysql_sync_table`）：

```bash
<FLINK_HOME>/bin/flink run \
    -c org.apache.paimon.flink.action.FlinkActions paimon-flink-action.jar \
    mysql_sync_table \
    --warehouse hdfs:///paimon/warehouse \
    --database paimon_db --table orders \
    --primary-keys order_id --partition-keys dt \
    --mysql-conf hostname=mysql-host --mysql-conf port=3306 \
    --mysql-conf username=root --mysql-conf password=secret \
    --mysql-conf database-name=source_db --mysql-conf table-name='orders' \
    --type-mapping to-nullable \
    --table-conf merge-engine=partial-update \
    --table-conf changelog-producer=lookup \
    --table-conf bucket=4 \
    --table-conf snapshot.time-retained=24h
```

要点：`--type-mapping to-nullable` 把源库 NOT NULL 列转 nullable（陷阱 5）；partial-update 表配 `changelog-producer=lookup`（§5.4 选型）；表不存在时按源 schema 自动建表（§3.3）。

对应的 Paimon 表若手动建（多源汇聚画像），配置如 §2.5 所示，三 sequence-group + 混合聚合 + 主源删除触发整行删。

### 7.2 库级同步（分库分表合并）

把 `source_db_0..source_db_n` 整库（含分表）合并同步（`mysql_sync_database`）：

```bash
<FLINK_HOME>/bin/flink run \
    -c org.apache.paimon.flink.action.FlinkActions paimon-flink-action.jar \
    mysql_sync_database \
    --warehouse hdfs:///paimon/warehouse --database paimon_db \
    --mysql-conf hostname=mysql-host --mysql-conf port=3306 \
    --mysql-conf username=root --mysql-conf password=secret \
    --mysql-conf database-name='source_db_\d+' \
    --merge-shards true \
    --including-tables 'order.*|user.*' --excluding-tables 'tmp_.*' \
    --table-prefix '' --table-suffix '' \
    --table-conf changelog-producer=input \
    --table-conf bucket=-1 \
    --type-mapping to-nullable,to-string
```

要点：`database-name='source_db_\d+'` 正则匹配多个分库；`--merge-shards true` 把 `order_0/order_1/…` 归一成 `order`（§5.2）；`--including/--excluding-tables` 正则过滤；`--type-mapping to-string` 把不支持的类型兜底转 STRING（对应 §4.2 跨族转 STRING 分支）。COMBINED 模式下新加的表会被自动发现接管（§5.3）。

### 7.3 跨分区更新建表

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY NOT ENFORCED,
    order_status STRING, amount DECIMAL(10,2), dt STRING
) PARTITIONED BY (dt) WITH (
    'merge-engine' = 'deduplicate',
    'bucket' = '-1',                            -- 必须动态 bucket（§6.1）
    'changelog-producer' = 'lookup',
    'cross-partition-upsert.index-ttl' = '7d'   -- 全局索引 TTL
);
```

注意事项（详见 §6）：① 必须 `bucket=-1`，否则不走 `GlobalIndexAssigner`；② 只能单写入（本地 RocksDB 状态）；③ merge engine 决定迁移策略——deduplicate 删旧写新（真迁移）、partial-update/aggregation 原地更新旧分区（不迁移）、first-row 丢弃新记录；④ `cross-partition-upsert.index-ttl` 控制索引过期，过期 key 再现按新记录处理；⑤ 大表重启 bootstrap 慢，调 `cross-partition-upsert.bootstrap-parallelism`；⑥ 索引默认存 `<table>/index/`，可用 `global-index.external-path` 外置。

---

## 8. 设计决策总结

| 决策点 | 选择 | 取舍 / 代价 | 收益 |
|--------|------|------------|------|
| 多源宽表如何拼 | partial-update merge engine（写入侧合并非空字段）| 只能用内置合并策略、合并是黑盒 | Flink 无状态、加列零改动、声明式 |
| 字段级乱序如何控 | sequence-group（每组绑定 seq 字段）| 配置变复杂、需维护单调 seq 字段 | 多源各自独立取舍、互不污染 |
| 部分列聚合 | partial-update 混用 FieldAggregator | 不可逆函数遇 retract 需特殊处理 | 一张表内"拼接 + 预聚合"一次写完 |
| 删除语义 | 默认抛异常，强制三选一显式策略 | 用户必须理解三种模式与互斥约束 | 不会静默丢数据 |
| CDC 中间数据态 | `Map<String,String>`（CdcRecord）| 中间态无类型检查、字符串化开销 | 统一十几种格式、容忍字段增减 |
| 多表路由 | RichCdcMultiplexRecord 携带库名/表名/schema | 记录体积略大 | 一条流承载整库、动态扩表 |
| DDL 与 DML | SideOutput 拆两条边，DDL 并行度强制 1 | schema 算子单点 | 数据高吞吐 + DDL 强串行并存 |
| CDC 类型演进 | canConvert 三态：只拓宽 / 缩窄静默 IGNORE / 跨族 EXCEPTION | 缩窄被静默不同步、易误解 | 数据安全（不丢/不溢出），自动跟进拓宽 |
| 分库分表合并 | TableNameConverter 归一 + AddColumn 容忍重复列 | 多表 schema 必须可兼容合并 | 一张目标表分析、加列自动汇聚 |
| 库级 Sink 模式 | COMBINED（多路复用，支持动态发现）/ DIVIDED | COMBINED 拓扑复杂、共享算子热点 | 表多/动态增表场景免重启 |
| 跨分区唯一性 | GlobalIndexAssigner + RocksDB 全局索引 | 单写入限制、bootstrap 成本、本地状态运维 | 改分区键无感、全局主键唯一 |
| 跨分区迁移策略 | 按 merge engine 分派 ExistingProcessor | partial-update/agg 不迁移易误解 | 各引擎语义正确（去重删旧、聚合保状态）|

---

> **文档版本**：基于 Paimon 1.5-SNAPSHOT 源码　**核对日期**：2026-06
> 交叉引用：merge engine 内部实现见 08；Changelog 产生机制见 24；Schema 演进机制见 22；Flink 写入链路见 15；BucketMode/动态桶见 01 §10。
