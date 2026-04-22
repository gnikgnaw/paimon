# Apache Paimon 源码深度分析

> 基于 Apache Paimon 1.5-SNAPSHOT 源码深度分析
> 分析时间: 2026-04-21
> 项目地址: https://github.com/apache/paimon

---

## 目录

- [一、核心架构篇](#一核心架构篇)
- [二、引擎集成篇](#二引擎集成篇)
- [三、进阶专题篇](#三进阶专题篇)
- [四、运维实战篇](#四运维实战篇)
- [阅读建议](#阅读建议)
- [与 Iceberg 的对比视角](#与-iceberg-的对比视角)

---

## 一、核心架构篇

| 序号 | 文档 | 主题 | 难度 |
|------|------|------|------|
| 01 | [核心存储引擎分析](01-核心存储引擎分析.md) | FileStore、LSM Merge-Tree、Levels、Compaction、Snapshot/Manifest | 高级 |
| 02 | [表读写路径分析](02-表读写路径分析.md) | Table API、Scan/Split/SplitRead、Write/Commit、谓词下推、系统表 | 高级 |
| 03 | [Catalog与元数据管理](03-Catalog与元数据管理.md) | Catalog体系、Schema演进、Snapshot管理、Manifest管理、Commit乐观锁 | 中级 |
| 04 | [DeletionVectors与文件索引](04-DeletionVectors与文件索引.md) | DV、Bloom Filter、Bitmap、BSI、Range Bitmap、Lookup机制 | 高级 |

## 二、引擎集成篇

| 序号 | 文档 | 主题 | 难度 |
|------|------|------|------|
| 05 | [Flink集成源码分析](05-Flink集成源码分析.md) | Source/Sink、CDC Sync、Checkpoint提交、Lookup Join | 高级 |
| 06 | [Spark集成源码分析](06-Spark集成源码分析.md) | DataSource V2、SQL Extensions、Procedures、Sort/ZOrder | 中级 |

## 三、进阶专题篇

| 序号 | 文档 | 主题 | 难度 |
|------|------|------|------|
| 08 | [Merge引擎与聚合函数](08-Merge引擎与聚合函数.md) | Deduplicate/PartialUpdate/Aggregation/FirstRow、Changelog产生 | 高级 |
| 09 | [Paimon面试题库](09-Paimon面试题库.md) | 从基础到专家级的源码面试题与深度解答 | 全部 |
| 12 | [查询优化与多维排序策略](12-查询优化与多维排序策略.md) | 多层过滤、文件索引、Z-Order/Hilbert排序、Lookup点查、Split优化 | 高级 |

## 四、运维实战篇

| 序号 | 文档 | 主题 | 难度 |
|------|------|------|------|
| 10 | [运维优化方案](10-运维优化方案.md) | Compaction调优、内存管理、Snapshot/Changelog管理、Bucket选择、监控指标 | 中级 |
| 11 | [小文件治理机制](11-小文件治理机制.md) | 小文件成因、Compaction治理、Manifest合并、过期清理、孤儿文件 | 高级 |

---

## 阅读建议

### 入门路线（1-2天）
1. 先读 **01-核心存储引擎分析** -- 理解 LSM-Tree 和 FileStore 的核心设计
2. 再读 **03-Catalog与元数据管理** -- 理解 Snapshot/Manifest 的层级关系
3. 最后读 **08-Merge引擎与聚合函数** -- 理解四种合并模式的差异

### 进阶路线（3-5天）
4. 读 **02-表读写路径分析** -- 理解完整的数据流转过程
5. 读 **04-DV与文件索引** -- 理解索引加速和行级删除
6. 读 **05-Flink集成** 或 **06-Spark集成** -- 根据你的技术栈选择

### 专家路线（持续）
7. 读 **12-查询优化与多维排序策略** -- 掌握查询性能优化的全链路
8. 读 **11-小文件治理** -- 理解生产环境的核心挑战
9. 读 **10-运维优化** -- 掌握调优方法论
10. 反复刷 **09-面试题库** -- 巩固源码级理解

---

## 与 Iceberg 的对比视角

| 维度 | Paimon | Iceberg | 为什么不同 |
|------|--------|---------|----------|
| 核心数据结构 | LSM Merge-Tree | 不可变文件 + Snapshot | Paimon 面向流式更新，LSM 天然支持 upsert |
| 更新策略 | 原生支持流式更新 | MOR (Equality/Position Delete) | Iceberg 优先保证读性能，更新是后期需求 |
| 主键支持 | 一等公民（主键表） | V2+ 通过 Equality Delete 实现 | Paimon 的核心场景就是主键表 |
| Compaction | 内置 LSM Compaction | 外部触发 RewriteDataFiles | Paimon 写入和 Compaction 紧耦合 |
| 流式写入 | 原生设计目标 | 后期集成 | Paimon 与 Flink 深度协同设计 |
| Lookup | 内置 Lookup 支持 | 需要外部服务 | 主键表的点查是高频需求 |
| Changelog | 原生 Changelog 产生 | 需要 CDC Scan | 流式消费需要原生 changelog |
| Manifest | base + delta 分离 | 单一 manifest list | Paimon 优化流式增量读取 |
| 小文件治理 | 内置自动 Compaction | 需要外部触发 | Paimon 的 LSM 天然包含 Compaction |
| 查询优化 | 多层过滤 + 4种文件索引 + Z-Order | manifest partition_summaries + Bloom | Paimon 索引类型更丰富 |
