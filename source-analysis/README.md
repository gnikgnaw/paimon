# Apache Paimon 源码深度分析

> 基于 Apache Paimon 1.5-SNAPSHOT 源码深度分析
> 分析时间: 2026-04-22
> 项目地址: https://github.com/apache/paimon
> 代码版本: commit 55f4fd175

---

## 目录

- [一、核心架构篇](#一核心架构篇)
- [二、引擎集成篇](#二引擎集成篇)
- [三、进阶专题篇](#三进阶专题篇)
- [四、运维实战篇](#四运维实战篇)
- [五、深度专题篇](#五深度专题篇)
- [六、对标分析篇](#六对标分析篇)
- [阅读建议](#阅读建议)

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

## 五、深度专题篇

| 序号 | 文档 | 主题 | 难度 |
|------|------|------|------|
| 13 | [索引机制深度分析](13-索引机制深度分析.md) | Bloom Filter、Bitmap Index、BSI、Range Bitmap、索引查询优化 | 高级 |
| 14 | [局部列更新与CDC数据集成](14-局部列更新与CDC数据集成.md) | PartialUpdate、CDC集成、数据一致性、增量同步 | 高级 |
| 15 | [Flink写入Paimon源码流程分析](15-Flink写入Paimon源码流程分析.md) | 写入流程、Checkpoint、事务性、性能优化 | 高级 |
| 16 | [分桶机制原理与实践](16-分桶机制原理与实践.md) | Bucket设计、分桶策略、并发写入、性能影响 | 中级 |
| 17 | [时间旅行与版本管理机制深度分析](17-时间旅行与版本管理.md) | Time Travel、Snapshot管理、版本回溯、时间戳查询 | 中级 |
| 18 | [内存管理与溢写机制深度分析](18-内存管理与溢写机制.md) | 内存分配、缓冲管理、溢写策略、GC优化 | 高级 |
| 19 | [数据类型系统与行存储深度分析](19-数据类型系统与行存储.md) | 类型系统、InternalRow、BinaryRow、序列化 | 中级 |
| 20 | [文件格式与IO层源码分析](20-文件格式与IO层分析.md) | Parquet/ORC/Avro、IO优化、压缩、编码 | 高级 |
| 23 | [Compaction全链路深度分析](23-Compaction全链路深度分析.md) | Compaction策略、执行流程、性能调优、工程细节 | 高级 |
| 24 | [Changelog机制全链路分析](24-Changelog机制全链路分析.md) | Changelog产生、消费、增量读取、流式处理 | 高级 |

## 六、对标分析篇

| 序号 | 文档 | 主题 | 难度 |
|------|------|------|------|
| 21 | [Paimon与Iceberg全面深度对比](21-Paimon与Iceberg全面对比.md) | 架构对比、性能对比、功能对比、适用场景 | 中级 |
| 22 | [Schema演进机制深度分析](22-Schema演进机制深度分析.md) | Schema演进、兼容性、版本管理、迁移策略 | 中级 |

---

## 阅读建议

### 入门路线（1-2天）
1. 先读 **01-核心存储引擎分析** -- 理解 LSM-Tree 和 FileStore 的核心设计
2. 再读 **03-Catalog与元数据管理** -- 理解 Snapshot/Manifest 的层级关系
3. 最后读 **08-Merge引擎与聚合函数** -- 理解四种合并模式的差异

### 进阶路线（3-5天）
4. 读 **02-表读写路径分析** -- 理解完整的数据流转过程
5. 读 **04-DeletionVectors与文件索引** -- 理解索引加速和行级删除
6. 读 **05-Flink集成** 或 **06-Spark集成** -- 根据你的技术栈选择

### 专家路线（持续）
7. 读 **12-查询优化与多维排序策略** -- 掌握查询性能优化的全链路
8. 读 **11-小文件治理** -- 理解生产环境的核心挑战
9. 读 **10-运维优化** -- 掌握调优方法论
10. 读 **23-Compaction全链路深度分析** -- 深入理解Compaction机制
11. 读 **24-Changelog机制全链路分析** -- 掌握流式消费的核心
12. 反复刷 **09-面试题库** -- 巩固源码级理解

### 对标学习路线
- 读 **21-Paimon与Iceberg全面深度对比** -- 理解两个系统的设计差异
- 读 **22-Schema演进机制深度分析** -- 掌握Schema管理最佳实践

### 特定场景深入
- **Flink用户**: 05 → 15 → 24 → 14
- **Spark用户**: 06 → 12 → 23
- **性能优化**: 10 → 11 → 18 → 20 → 23
- **数据集成**: 14 → 15 → 24
- **系统设计**: 01 → 02 → 03 → 04 → 21

---

## 文档质量说明

- 所有文档基于 Paimon 1.5-SNAPSHOT (commit: 55f4fd175) 源码分析
- 每份文档都包含源码位置引用，便于对照验证
- 文档涵盖架构设计、实现细节、性能优化、最佳实践
- 持续更新中，欢迎反馈和补充
