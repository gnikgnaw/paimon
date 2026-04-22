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

### 解决什么问题

**核心业务问题**：
- **Paimon**：解决流式数据高频写入并快速可见的问题。在实时数仓场景中，数据从 Kafka、MySQL Binlog 等源头持续流入，需要秒级到分钟级可见，同时支持高频更新（同一主键的数据可能每秒更新多次）
- **Iceberg**：解决大规模数据湖的元数据管理和表语义问题。Netflix 在使用 Hive 时遇到元数据瓶颈（单个表的分区数达到数万时 Hive Metastore 性能崩溃），需要一个可扩展的表格式

**没有这个设计的后果**：
- **没有 Paimon 的 LSM-Tree**：流式更新场景下，每次更新都需要重写整个数据文件（Copy-on-Write），或者产生大量 Delete File（Merge-on-Read），导致写入吞吐量低、小文件泛滥
- **没有 Iceberg 的不可变文件**：在对象存储上实现表语义时，并发写入可能导致文件损坏，需要复杂的锁机制；文件修改需要重新上传整个文件，成本高昂

**实际场景**：
- **Paimon 典型场景**：电商订单表实时同步（订单状态频繁变更）、用户画像实时更新（多个数据源更新同一用户的不同字段）、实时指标聚合（PV/UV 实时累加）
- **Iceberg 典型场景**：日志数据归档（每天追加一次）、数据仓库 ETL（每小时批量更新一次）、多引擎分析（Spark 写入、Trino/Presto/Athena 查询）

### 有什么坑

**Paimon 的坑**：
1. **Compaction 参数调优陷阱**：`numSortedRunStopTrigger` 设置过小会导致写入频繁阻塞，设置过大会导致读放大严重。生产环境需要根据写入速率和查询延迟要求反复调优
2. **内存配置误区**：`write-buffer-size` 设置过大会导致 OOM，设置过小会产生大量小文件。需要根据并行度和堆内存大小精确计算
3. **Bucket 数量陷阱**：固定桶模式下，桶数设置过少会导致单桶数据倾斜，设置过多会导致小文件问题。动态桶模式虽然自动扩展，但会引入全局索引的维护开销
4. **Lookup 模式性能陷阱**：`changelog.producer = lookup` 模式下，每次写入都需要查找旧值，如果数据未缓存会导致大量远程读取，严重拖慢写入速度

**Iceberg 的坑**：
1. **Delete File 累积陷阱**：高频更新场景下，如果不及时运行 `RewriteDataFiles`，Delete File 会快速累积，导致查询性能指数级下降（每个数据文件都需要关联检查所有 Delete File）
2. **Compaction 调度缺失**：Iceberg 不内置 Compaction，生产环境必须自己搭建调度系统（如 Airflow），否则小文件和 Delete File 会失控
3. **Equality Delete 性能陷阱**：Equality Delete 需要逐行匹配，性能远低于 Position Delete。但很多场景下（如 CDC 同步）只能使用 Equality Delete
4. **Snapshot 过期配置陷阱**：`expire_snapshots` 如果配置不当，可能删除正在被查询的 Snapshot，导致查询失败

### 核心概念解释

**LSM-Tree（Log-Structured Merge-Tree）**：
- 一种将随机写转化为顺序写的数据结构
- 数据先写入内存（MemTable），达到阈值后 flush 为磁盘上的有序文件（SSTable）
- 磁盘文件分层组织（Level-0、Level-1...），后台异步合并（Compaction）
- 代价是读放大（查询时可能需要合并多层文件）和写放大（Compaction 需要重写文件）
- 典型应用：RocksDB、LevelDB、HBase、Cassandra

**不可变文件（Immutable File）**：
- 文件一旦写入就不再修改，修改通过追加新文件实现
- 优点：避免并发写入冲突、利于缓存、支持时间旅行
- 代价：更新操作需要额外的 Delete File 或重写整个文件
- 典型应用：Iceberg、Delta Lake、Hudi（MOR 模式）

**Merge-on-Read vs Copy-on-Write**：
- **Merge-on-Read（MOR）**：更新时只追加变更记录，查询时合并。写快读慢
- **Copy-on-Write（COW）**：更新时重写整个文件。写慢读快
- Paimon 主键表是 MOR（LSM-Tree 本质是 MOR），Iceberg 默认是 COW + Delete File（混合模式）

**主键表 vs 追加表**：
- **主键表**：有主键约束，支持 Upsert/Delete，需要去重和合并逻辑
- **追加表**：无主键，只支持 Append，不做去重
- Paimon 两者都支持（`KeyValueFileStore` vs `AppendOnlyFileStore`），Iceberg 只有追加表（主键通过 Upsert 语义模拟）

### 设计理念

**Paimon 为什么选择 LSM-Tree**：
1. **流式场景的本质需求**：Flink 的 Checkpoint 间隔通常是秒级到分钟级，在这个时间窗口内，一个热点主键可能被更新数十次甚至上百次。如果每次更新都重写文件（COW），写入吞吐量会崩溃
2. **顺序写的性能优势**：对象存储（S3/OSS）的顺序写吞吐量远高于随机写。LSM-Tree 将所有写入转化为顺序追加，充分利用存储带宽
3. **内存吸收突发流量**：WriteBuffer 在内存中吸收短时间内的大量更新，批量 flush 为文件，减少 I/O 次数
4. **Compaction 异步化**：写入线程不参与 Compaction，由独立线程池异步执行，写入延迟低且稳定

**Iceberg 为什么选择不可变文件**：
1. **对象存储的特性匹配**：S3/OSS 等对象存储不支持原地修改（没有 POSIX 的 seek/write 语义），只能整个文件上传。不可变文件天然适配这种存储
2. **并发安全性**：不可变文件避免了并发写入时的文件损坏问题。多个写入者可以同时写入不同的文件，通过 OCC（乐观并发控制）在提交时检测冲突
3. **时间旅行的简洁实现**：每个 Snapshot 指向一组不可变文件，回溯到历史版本只需切换 Snapshot 指针，不需要重建数据
4. **引擎无关性**：不可变文件模型不依赖任何特定的计算引擎或常驻进程，任何引擎都可以读写

**权衡取舍**：
- **Paimon 的取舍**：用更高的系统复杂度（LSM-Tree 的 Compaction 策略、反压机制）换取流式场景的高性能
- **Iceberg 的取舍**：用更新性能的牺牲（Delete File 机制）换取引擎无关性和批查询的简洁性

**业界对比**：
- **Delta Lake**：与 Iceberg 类似，也是不可变文件 + Delete File，但与 Spark 耦合更紧密
- **Hudi**：提供 MOR 和 COW 两种模式，MOR 模式类似 Paimon 的 LSM-Tree，但实现复杂度更高
- **Kudu**：也使用 LSM-Tree，但是独立的存储引擎（非文件格式），需要常驻进程

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

### 解决什么问题

**核心业务问题**：
- **Paimon 的 Bucket + LSM-Tree**：解决主键表的高效更新和查询问题。同一主键的所有版本必须在同一个存储单元中，才能高效地进行 Merge-on-Read
- **Iceberg 的扁平文件 + Delete File**：解决在不修改原始数据文件的前提下实现更新和删除的问题

**没有这个设计的后果**：
- **没有 Bucket 分桶**：主键数据散落在所有文件中，Merge-on-Read 时需要扫描全表所有文件，性能不可接受
- **没有 LSM 分层**：所有更新都写入同一层，文件无序，查询时需要全量扫描并排序
- **没有 Delete File**：更新操作必须重写整个数据文件，在大文件场景下（如 1GB 的 Parquet 文件）成本极高

**实际场景**：
- **Paimon Bucket 场景**：用户表按 user_id 哈希分桶，同一用户的所有更新落在同一个 Bucket，查询单个用户时只需扫描一个 Bucket 的 LSM-Tree
- **Iceberg Delete File 场景**：订单表每天追加新订单，偶尔需要更新订单状态。更新时只追加一个小的 Position Delete 文件，不需要重写整个数据文件

### 有什么坑

**Paimon 的坑**：
1. **固定桶数陷阱**：`HASH_FIXED` 模式下，桶数一旦确定无法修改。如果数据量增长超出预期，单桶数据过大会导致 Compaction 时间过长、查询性能下降
2. **动态桶的全局索引开销**：`HASH_DYNAMIC` 模式下，需要维护全局的 Hash Index（主键到 Bucket 的映射），这个索引本身可能变得很大（数亿主键时可达 GB 级别），影响写入性能
3. **Level-0 文件累积**：如果 Compaction 速度跟不上写入速度，Level-0 文件会快速累积，导致读放大严重（查询时需要合并几十个甚至上百个文件）
4. **跨 Bucket 查询性能**：范围查询或全表扫描时，需要扫描所有 Bucket 的所有 Level，I/O 放大严重

**Iceberg 的坑**：
1. **Delete File 与 Data File 的关联成本**：每个 Data File 都需要检查是否有对应的 Delete File。当 Delete File 数量很多时（如数千个），这个关联操作本身就成为性能瓶颈
2. **Equality Delete 的性能陷阱**：Equality Delete 需要逐行匹配删除条件，性能极差。在 CDC 同步场景下，如果上游没有提供行位置信息，只能使用 Equality Delete
3. **Position Delete 的脆弱性**：Position Delete 依赖文件路径和行号，如果数据文件被 Compaction 重写，Position Delete 会失效（需要同步重写）
4. **Deletion Vector 的兼容性**：Deletion Vector 是 V3 格式新增的特性，老版本引擎无法识别，可能导致数据不一致

### 核心概念解释

**Bucket（桶）**：
- Paimon 中用于组织主键数据的逻辑单元
- 同一主键的所有版本必定落在同一个 Bucket 中（通过哈希或范围分区）
- 每个 Bucket 维护独立的 LSM-Tree，互不干扰
- 类比：HBase 的 Region、Cassandra 的 Partition

**LSM-Tree 的 Level（层级）**：
- **Level-0**：无序文件，直接从内存 flush 产生，文件之间的 Key 范围可能重叠
- **Level-1 及以上**：有序文件，同一层内的文件 Key 范围不重叠，通过 Compaction 从下层合并产生
- **Level-max**：最高层，包含最老的数据，文件最大

**SortedRun（有序运行）**：
- LSM-Tree 中的一个有序文件序列
- Level-0 中每个文件是一个 SortedRun（因为文件之间无序）
- Level-1 及以上每层是一个 SortedRun（因为层内文件有序且不重叠）
- 查询时需要合并所有 SortedRun 的数据

**Delete File 的三种类型**：
- **Position Delete**：记录 `(file_path, row_position)` 对，精确删除某个文件的某一行。读取时可以高效跳过
- **Equality Delete**：记录删除条件（如 `id = 123`），读取时需要逐行匹配。性能差但灵活
- **Deletion Vector**：用 Bitmap 标记被删除的行号，是 Position Delete 的优化版本（V3 格式）

**写放大 / 读放大 / 空间放大**：
- **写放大**：写入 1MB 数据，实际磁盘写入 > 1MB（因为 Compaction 需要重写文件）
- **读放大**：读取 1MB 数据，实际磁盘读取 > 1MB（因为需要合并多个文件或应用 Delete File）
- **空间放大**：存储 1MB 有效数据，实际占用 > 1MB（因为有过期数据未清理或 Delete File 占用空间）

### 设计理念

**Paimon 为什么需要 Bucket**：
1. **限制 Merge-on-Read 的范围**：如果没有 Bucket，查询一个主键需要扫描全表所有文件。有了 Bucket，只需扫描一个 Bucket 的文件（通常是几十个到几百个文件）
2. **并行写入的隔离**：每个 Writer 负责若干个 Bucket，互不干扰。Flink 的并行度可以大于 Bucket 数（多个 Writer 共享 Bucket），也可以小于 Bucket 数（一个 Writer 负责多个 Bucket）
3. **Compaction 的并行化**：每个 Bucket 独立做 Compaction，可以并行执行，不会互相阻塞

**Paimon 为什么需要 Level 分层**：
1. **控制写放大**：如果所有文件都在一层，每次 Compaction 都需要合并全部文件，写放大极高。分层后，大部分 Compaction 只涉及相邻两层的部分文件
2. **控制读放大**：分层后，高层文件更大、更有序，查询时优先从高层读取，减少需要合并的文件数量
3. **支持 TTL 和过期**：老数据沉淀到高层，过期时可以直接删除整个高层文件，不需要逐行扫描

**Iceberg 为什么不需要 Bucket**：
1. **不可变文件模型**：数据文件一旦写入不再修改，不需要按主键组织。更新通过全局的 Delete File 标记，与文件组织方式无关
2. **分区已经提供了数据组织**：Iceberg 的分区（Partition）本身就是一种数据组织方式，按时间或其他维度分区后，查询可以跳过无关分区
3. **简化模型**：不引入 Bucket 概念，降低用户理解成本

**Iceberg 为什么选择 Delete File**：
1. **避免重写大文件**：数据文件可能很大（1GB+），重写成本高。Delete File 通常很小（几 MB），追加成本低
2. **支持事务性删除**：Delete File 可以在一个事务中原子提交，保证删除操作的一致性
3. **延迟合并**：Delete File 可以累积一段时间后再通过 Compaction 合并到数据文件中，平衡写入和查询性能

**权衡取舍**：
- **Paimon 的 Bucket + LSM**：用更复杂的存储结构换取主键表的高性能，代价是需要精心设计 Bucket 数量和 Compaction 策略
- **Iceberg 的扁平文件 + Delete File**：用更简单的模型换取引擎无关性，代价是更新性能较差且需要外部 Compaction

**业界对比**：
- **HBase/Cassandra**：也使用 LSM-Tree + 分区（Region/Partition），但它们是独立的存储系统，不是文件格式
- **Delta Lake**：使用 Delete File（称为 Deletion Vector），但没有 LSM-Tree 的分层结构
- **Hudi**：MOR 模式下使用 Log File（类似 Delete File），但组织方式更复杂（FileGroup + FileSlice）

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

### 解决什么问题

**核心业务问题**：
- **Paimon 的 base/delta 分离**：解决流式增量读取的性能问题。流式消费者只关心自上次消费以来的新增数据，不需要扫描全量元数据
- **Iceberg 的统一 manifest list**：解决批量查询的简洁性问题。批查询需要看到某个时间点的完整表状态，统一的 manifest list 提供了这个视图

**没有这个设计的后果**：
- **没有 delta manifest**：流式消费者每次都需要扫描全量 manifest list，对比两个 Snapshot 的差异。当表有数百万个文件时，这个对比操作本身就需要数秒甚至数十秒
- **没有 base manifest**：全量查询时需要从头开始累积所有增量变更，性能低下
- **没有 manifest 分层**：所有文件元数据混在一起，无法做针对性的优化（如增量读取、过期清理）

**实际场景**：
- **Paimon 增量读取场景**：Flink 流式作业消费 Paimon 表，每个 Checkpoint 间隔（如 1 分钟）读取一次新数据。通过 `deltaManifestList` 可以在毫秒级发现新文件，无需扫描全量元数据
- **Iceberg 批量查询场景**：Spark 批作业查询某个时间点的表快照，直接读取该 Snapshot 的 manifest list，获取所有有效文件

### 有什么坑

**Paimon 的坑**：
1. **base/delta 不一致陷阱**：如果 Manifest Compaction 失败（如进程崩溃），可能导致 base 和 delta 不一致，查询结果错误。需要通过 `RepairProcedure` 修复
2. **delta manifest 膨胀**：如果长时间不做 Manifest Compaction，delta manifest 会累积大量小文件，反而拖慢增量读取性能
3. **Manifest Full Compaction 的时机**：`MANIFEST_FULL_COMPACTION_FILE_SIZE` 设置过小会导致频繁的全量合并（影响写入性能），设置过大会导致 delta 膨胀
4. **Snapshot 过期时的 manifest 清理**：过期 Snapshot 时需要同时清理其引用的 manifest 文件，如果清理逻辑有 bug，可能导致 manifest 泄漏（占用大量存储空间）

**Iceberg 的坑**：
1. **增量读取的性能陷阱**：`IncrementalAppendScan` 需要对比两个 Snapshot 的 manifest list，当 manifest 数量很多时（如数千个），这个对比操作本身就很慢
2. **Manifest 文件碎片化**：每次 Commit 都会产生新的 manifest 文件，如果不定期运行 `RewriteManifests`，manifest 文件会碎片化（数量多、单个文件小），拖慢查询规划
3. **metadata.json 的单点瓶颈**：所有元数据都在 `metadata.json` 中，这个文件会随着 Schema 变更、Snapshot 累积而变大。当达到 MB 级别时，读取和解析成本显著
4. **Snapshot 引用的传递性**：一个 Snapshot 可能引用多个 manifest，每个 manifest 又引用多个数据文件。过期 Snapshot 时需要递归检查引用关系，逻辑复杂且容易出错

### 核心概念解释

**Snapshot（快照）**：
- 表在某个时间点的完整状态
- 包含该时刻所有有效数据文件的引用（通过 manifest list）
- 不可变——一旦创建就不再修改
- 支持时间旅行——可以查询历史 Snapshot

**Manifest List（清单列表）**：
- Snapshot 的直接子元数据，记录该 Snapshot 包含哪些 Manifest 文件
- Paimon 分为 base 和 delta 两个 list，Iceberg 只有一个 list
- 通常是 Avro 格式（Paimon 也支持 JSON）

**Manifest File（清单文件）**：
- 记录一组数据文件的元数据（路径、大小、行数、统计信息等）
- 每个 Manifest 文件通常包含数百到数千个数据文件的元数据
- Manifest 文件本身也是不可变的

**Manifest Entry（清单条目）**：
- Manifest 文件中的一条记录，对应一个数据文件
- 包含文件的详细元数据：路径、格式、分区、行数、列统计、Null 值统计等
- Paimon 还包含 `_KIND`（ADD/DELETE）、`_BUCKET` 等字段

**base/delta 分离的原理**：
- **base manifest list**：包含该 Snapshot 的全量文件列表（累积视图）
- **delta manifest list**：只包含自上次 Snapshot 以来的增量变更（增量视图）
- 全量查询读 base，增量读取读 delta，各取所需

**Manifest Compaction（清单压缩）**：
- 将多个小 Manifest 文件合并为一个大 Manifest 文件
- 减少 Manifest 文件数量，加速查询规划
- Paimon 在提交时自动触发，Iceberg 需要手动调用 `RewriteManifests`

### 设计理念

**Paimon 为什么要做 base/delta 分离**：
1. **流式消费是一等场景**：Paimon 的核心用户是 Flink 流式作业，增量读取的性能直接影响端到端延迟。base/delta 分离使得增量读取的时间复杂度从 O(全量文件数) 降低到 O(增量文件数)
2. **Snapshot 过期的效率**：过期 Snapshot 时，只需要处理 delta 部分引用的文件，不需要扫描全量 base manifest。这在文件数量达到百万级时非常关键
3. **Manifest Compaction 的灵活性**：base 和 delta 可以独立做 Compaction。delta 可以频繁合并（因为小），base 可以低频合并（因为大）

**Iceberg 为什么选择统一 manifest list**：
1. **批查询的简洁性**：批查询只需要读取一个 manifest list，就能获取完整的文件列表。不需要理解 base/delta 的概念
2. **引擎无关性**：统一的 manifest list 更容易被各种引擎理解和实现。base/delta 分离需要引擎理解增量读取的语义
3. **模型简单**：只有一个 manifest list，降低了元数据管理的复杂度

**Paimon 为什么选择 JSON 格式的 Snapshot**：
1. **人类可读**：运维人员可以直接 `cat` Snapshot 文件查看内容，无需专门的工具
2. **调试友好**：出现问题时可以快速定位（如哪个 manifest list 损坏）
3. **灵活性**：JSON 易于扩展新字段，不需要修改 Schema

**Iceberg 为什么选择 Avro 格式的 Snapshot**：
1. **紧凑性**：Avro 的二进制格式比 JSON 更紧凑，节省存储空间
2. **Schema 演进**：Avro 内置 Schema 演进支持，可以安全地添加/删除字段
3. **性能**：Avro 的解析性能优于 JSON（尤其是大文件）

**权衡取舍**：
- **Paimon 的 base/delta 分离**：用更高的元数据管理复杂度换取流式增量读取的极致性能
- **Iceberg 的统一 manifest list**：用增量读取的性能牺牲换取模型的简洁性和引擎无关性

**业界对比**：
- **Delta Lake**：也使用 JSON 格式的 Commit Log，但没有 base/delta 分离，增量读取需要扫描多个 Commit
- **Hudi**：使用 Timeline 机制管理元数据，类似 Paimon 的 Snapshot 序列，但元数据格式更复杂

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

### 解决什么问题

**核心业务问题**：
- **Paimon 的 MergeFunction**：解决复杂的数据合并语义问题。在实时场景中，同一主键的数据可能来自多个源头，需要灵活的合并策略（去重、部分更新、聚合等）
- **Iceberg 的 Delete File**：解决在不修改原始数据文件的前提下实现更新和删除的问题

**没有这个设计的后果**：
- **没有 MergeFunction**：所有更新都只能是简单的覆盖（Last Write Wins），无法实现复杂的业务逻辑（如字段级合并、实时聚合）
- **没有 Delete File**：每次更新都需要重写整个数据文件，在大文件场景下（如 1GB 的 Parquet）成本极高，吞吐量低
- **没有 Lookup 机制**：无法生成完整的 CDC 流（需要知道旧值才能产出 UPDATE_BEFORE 事件）

**实际场景**：
- **Paimon Partial Update 场景**：用户画像表，用户行为日志更新 `last_login_time`，订单系统更新 `total_amount`，推荐系统更新 `interest_tags`。三个系统并发写入同一用户的不同字段，Partial Update 自动合并
- **Paimon Aggregation 场景**：实时 PV/UV 统计表，每条日志到达时对 `pv` 字段做 `sum` 聚合，对 `uv` 字段做 `count(distinct)` 聚合（通过 HyperLogLog）
- **Iceberg Position Delete 场景**：订单表每天追加新订单，偶尔需要取消订单（删除）。通过 Position Delete 标记被删除的行，不需要重写数据文件

### 有什么坑

**Paimon 的坑**：
1. **Partial Update 的字段冲突**：如果两个写入者同时更新同一字段，后到达的会覆盖先到达的（Last Write Wins）。需要通过 `sequence.field` 指定版本字段来控制合并顺序
2. **Aggregation 的数据类型限制**：不是所有数据类型都支持所有聚合函数。如 `sum` 只支持数值类型，`listagg` 只支持字符串类型。配置错误会导致运行时异常
3. **Lookup 模式的性能陷阱**：`changelog.producer = lookup` 模式下，每次写入都需要查找旧值。如果旧值不在缓存中，需要远程读取文件，严重拖慢写入速度。需要合理配置 `lookup.cache-file-retention` 和 `lookup.cache-max-memory`
4. **MergeFunction 的状态膨胀**：某些 MergeFunction（如 `collect`、`listagg`）会累积所有历史值，导致单行数据膨胀。需要配合 TTL 或定期清理

**Iceberg 的坑**：
1. **Delete File 累积的性能雪崩**：高频更新场景下，Delete File 会快速累积。当一个 Data File 关联了数百个 Delete File 时，读取性能会指数级下降（需要逐个应用 Delete File）
2. **Equality Delete 的全表扫描**：Equality Delete 需要逐行匹配删除条件，无法利用索引。在大表上使用 Equality Delete 会导致查询变成全表扫描
3. **Position Delete 的脆弱性**：Position Delete 依赖文件路径和行号。如果数据文件被 Compaction 重写（路径变化），Position Delete 会失效。需要在 Compaction 时同步重写 Position Delete
4. **Deletion Vector 的兼容性**：Deletion Vector 是 V3 格式新增的特性，老版本引擎（如 Spark 3.3）无法识别，可能导致数据不一致（删除的行仍然可见）

### 核心概念解释

**MergeFunction（合并函数）**：
- 定义同一主键的多条记录如何合并为一条记录
- 在 Merge-on-Read 时调用（查询时或 Compaction 时）
- Paimon 提供 4 种内置合并引擎 + 20+ 种字段聚合器

**Deduplicate（去重）**：
- 最简单的合并策略，保留最后一条记录（或第一条记录）
- 适用于标准的 Upsert 场景
- 实现：`DeduplicateMergeFunction`

**Partial Update（部分更新）**：
- 合并多条记录的非 null 字段
- 适用于多源写入同一行的不同列
- 实现：`PartialUpdateMergeFunction`
- 示例：记录 A 有 `{id=1, name="Alice", age=null}`，记录 B 有 `{id=1, name=null, age=30}`，合并后 `{id=1, name="Alice", age=30}`

**Aggregation（聚合）**：
- 对字段应用聚合函数（sum、max、min、count 等）
- 适用于实时指标计算
- 实现：`AggregateMergeFunction` + `FieldAggregator`
- 示例：PV 字段配置 `sum` 聚合，每条记录的 PV 值累加

**Lookup（查找）**：
- 在写入时查找旧值，与新值合并，生成完整的 CDC 流
- 适用于需要 UPDATE_BEFORE 事件的场景
- 实现：`LookupMergeFunction` 包装其他 MergeFunction
- 性能优化：通过 `LookupLevels` 缓存高层文件到本地

**Position Delete（位置删除）**：
- 记录 `(file_path, row_position)` 对，精确删除某个文件的某一行
- 读取时可以高效跳过（通过 Bitmap 或跳表）
- 适用于已知行位置的删除场景（如 CDC 同步）

**Equality Delete（等值删除）**：
- 记录删除条件（如 `id = 123`），读取时逐行匹配
- 性能差但灵活，适用于不知道行位置的删除场景
- 实现：读取时需要构建删除条件的 Hash Set，逐行检查

**Deletion Vector（删除向量）**：
- 用 Bitmap 标记被删除的行号，是 Position Delete 的优化版本
- 读取时通过 Bitmap 过滤，性能优于逐行检查
- Iceberg V3 格式新增，存储在 Puffin 文件中

### 设计理念

**Paimon 为什么需要丰富的 MergeFunction**：
1. **实时场景的复杂性**：实时数仓不是简单的 Upsert，而是有各种复杂的业务逻辑。如用户画像需要部分更新，实时指标需要聚合，CDC 下游需要完整的变更流
2. **多源写入的协调**：在微服务架构下，同一张表可能被多个服务并发写入。MergeFunction 提供了一种声明式的协调机制，避免写入冲突
3. **性能优化**：将合并逻辑下推到存储层，避免在计算层做 Join 或聚合。如实时 PV/UV 统计，如果在 Flink 中做状态聚合，状态会非常大；下推到 Paimon 后，状态只保留在存储中

**Paimon 为什么需要 Lookup 机制**：
1. **CDC 语义的完整性**：下游消费者（如 Elasticsearch、MySQL）需要完整的 CDC 流（包括 UPDATE_BEFORE）才能正确处理更新。没有 Lookup，只能产出 INSERT 和 UPDATE_AFTER
2. **流批一体**：Lookup 机制使得流式写入可以产出与批量查询一致的结果。批量查询看到的是合并后的最终值，流式消费看到的是合并前后的变化

**Iceberg 为什么选择 Delete File**：
1. **不可变文件的约束**：Iceberg 的数据文件是不可变的，无法原地修改。Delete File 是在不修改原文件的前提下实现删除的唯一方式
2. **事务性**：Delete File 可以在一个事务中原子提交，保证删除操作的一致性
3. **延迟合并**：Delete File 可以累积一段时间后再通过 Compaction 合并到数据文件中，平衡写入和查询性能

**Iceberg 为什么提供三种删除方式**：
1. **Position Delete**：性能最好，但需要知道行位置（通常来自 CDC 源或之前的查询）
2. **Equality Delete**：最灵活，但性能最差。适用于无法获取行位置的场景（如 SQL DELETE 语句）
3. **Deletion Vector**：性能和灵活性的折中，是 Position Delete 的优化版本

**权衡取舍**：
- **Paimon 的 MergeFunction**：用更高的系统复杂度（需要实现和维护 20+ 种聚合器）换取业务逻辑的灵活性和性能
- **Iceberg 的 Delete File**：用更新性能的牺牲（Delete File 累积后严重影响读性能）换取模型的简洁性和事务性

**业界对比**：
- **Delta Lake**：也使用 Delete File（称为 Deletion Vector），但没有 Paimon 的 MergeFunction 体系
- **Hudi**：提供 Payload 机制（类似 MergeFunction），但实现复杂度更高，性能不如 Paimon
- **ClickHouse**：提供 ReplacingMergeTree、SummingMergeTree 等多种 MergeTree 变体，思路与 Paimon 类似

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

### 解决什么问题

**核心业务问题**：
- **Paimon 的内置 Compaction**：解决 LSM-Tree 的读放大问题。如果不做 Compaction，Level-0 文件会无限累积，查询时需要合并几十甚至上百个文件，性能不可接受
- **Iceberg 的外部 Compaction**：解决小文件和 Delete File 累积问题。高频写入会产生大量小文件，Delete File 累积会严重拖慢查询

**没有这个设计的后果**：
- **没有 Compaction**：Paimon 的查询性能会快速退化（SortedRun 累积导致读放大），最终系统不可用。Iceberg 的查询性能也会退化（小文件和 Delete File 累积），但不会完全不可用
- **没有反压机制**：写入速度超过 Compaction 速度时，系统会失控（内存溢出或磁盘爆满）
- **没有自动触发**：依赖人工或外部调度触发 Compaction，运维成本高，容易遗漏

**实际场景**：
- **Paimon 自动 Compaction 场景**：Flink 流式作业持续写入订单表，每分钟产生数百个 Level-0 文件。后台 Compaction 线程自动将 Level-0 文件合并到 Level-1，再合并到 Level-2，保持查询性能稳定
- **Iceberg 手动 Compaction 场景**：Spark 批作业每小时写入一次日志表，每次产生数千个小文件。运维人员配置 Airflow 定时任务，每天凌晨运行 `RewriteDataFiles`，将小文件合并为大文件

### 有什么坑

**Paimon 的坑**：
1. **Compaction 阻塞写入**：当 SortedRun 数量超过 `numSortedRunStopTrigger` 时，写入线程会被阻塞，等待 Compaction 完成。如果 Compaction 速度慢（如文件很大或 I/O 慢），会导致写入延迟飙升甚至超时
2. **Compaction 线程池配置**：`compaction.max-parallel-compactions` 设置过小会导致 Compaction 速度跟不上写入速度，设置过大会占用过多 CPU 和内存。需要根据机器配置和写入速率精确调优
3. **Full Compaction 的时机**：`EarlyFullCompaction` 可以定期触发全量合并，但如果触发过于频繁，会导致写放大严重；触发过于稀疏，会导致读放大严重
4. **Compaction 与 Checkpoint 的交互**：Compaction 结果在 Checkpoint 时提交。如果 Checkpoint 失败，Compaction 结果会丢失，需要重新执行。频繁的 Checkpoint 失败会导致 Compaction 无效功

**Iceberg 的坑**：
1. **Compaction 调度缺失**：Iceberg 不内置 Compaction，生产环境必须自己搭建调度系统（如 Airflow、Oozie）。如果调度系统故障或配置错误，Compaction 会停止，导致性能退化
2. **RewriteDataFiles 的成本**：`RewriteDataFiles` 需要重写整个数据文件，I/O 成本极高。在大表上（如 TB 级别），一次 Compaction 可能需要数小时，占用大量计算资源
3. **Compaction 与查询的冲突**：Compaction 期间会产生大量 I/O，可能影响正在运行的查询。需要在非高峰时段运行 Compaction
4. **Delete File 的 Compaction 策略**：`RewriteDataFiles` 会应用 Delete File 并重写数据文件，但如果 Delete File 很少（如只删除了 1% 的行），重写整个文件的成本不划算。需要根据删除比例决定是否 Compaction

### 核心概念解释

**Universal Compaction（通用压缩）**：
- RocksDB 风格的 Compaction 策略，Paimon 采用
- 核心思想：根据文件大小和数量动态选择合并策略
- 四种触发条件：空间放大、大小比例、文件数量、定时全量合并

**SortedRun（有序运行）**：
- LSM-Tree 中的一个有序文件序列
- Level-0 中每个文件是一个 SortedRun（因为文件之间无序）
- Level-1 及以上每层是一个 SortedRun（因为层内文件有序且不重叠）
- `numSortedRunStopTrigger` 控制 SortedRun 数量上限

**空间放大（Space Amplification）**：
- 存储 1MB 有效数据，实际占用 > 1MB（因为有过期数据未清理）
- `maxSizeAmp` 参数控制空间放大上限（默认 200%，即最多占用 3MB）
- 当空间放大超过阈值时，触发全量 Compaction

**大小比例（Size Ratio）**：
- 相邻 SortedRun 的大小比例
- 如果 `size(run[i]) / size(run[i+1]) < sizeRatio`，触发合并
- 目的：保持各层大小平衡，避免某一层过大

**反压机制（Backpressure）**：
- 当 SortedRun 数量超过 `numSortedRunStopTrigger` 时，写入线程阻塞
- 目的：防止写入速度超过 Compaction 速度，导致系统失控
- 类似 Flink 的反压机制

**RewriteDataFiles（重写数据文件）**：
- Iceberg 的 Compaction 操作，重写数据文件以合并小文件或应用 Delete File
- 参数：`target-file-size-bytes`（目标文件大小）、`min-file-size-bytes`（最小文件大小）
- 策略：将小于 `min-file-size-bytes` 的文件合并为接近 `target-file-size-bytes` 的文件

**RewriteManifests（重写清单）**：
- Iceberg 的 Manifest Compaction 操作，合并小 Manifest 文件
- 目的：减少 Manifest 文件数量，加速查询规划

### 设计理念

**Paimon 为什么选择内置 Compaction**：
1. **LSM-Tree 的必要性**：LSM-Tree 如果不做 Compaction，读性能会快速退化到不可用。Compaction 不是可选的优化，而是系统正确运行的前提
2. **流式场景的实时性**：流式写入是持续的，不能等到批量 Compaction。内置 Compaction 可以在写入的同时异步执行，保持系统稳定
3. **简化运维**：用户只需配置几个参数（如 `numSortedRunStopTrigger`、`maxSizeAmp`），系统自动管理 Compaction，无需搭建外部调度系统

**Paimon 为什么选择 Universal Compaction**：
1. **写放大可控**：Universal Compaction 的写放大由 `maxSizeAmp` 控制，可以根据业务需求调整（如对写入敏感的场景可以设置更大的 `maxSizeAmp`）
2. **适应性强**：Universal Compaction 可以根据数据分布动态调整策略，适应不同的写入模式（如高频小批次、低频大批次）
3. **实现简单**：相比 Leveled Compaction（RocksDB 的另一种策略），Universal Compaction 的实现更简单，代码更易维护

**Iceberg 为什么选择外部 Compaction**：
1. **引擎无关性**：Iceberg 作为开放表格式，不绑定任何计算引擎。内置 Compaction 意味着需要一个常驻的计算进程，这与 Iceberg 的设计哲学矛盾
2. **灵活性**：外部 Compaction 可以使用任何计算引擎（Spark、Flink、Trino），用户可以根据自己的技术栈选择
3. **成本控制**：外部 Compaction 可以在非高峰时段运行，充分利用闲置资源，降低成本

**Iceberg 为什么提供多种 Compaction Action**：
1. **不同的优化目标**：`RewriteDataFiles` 优化查询性能，`RewriteManifests` 优化元数据性能，`RemoveDanglingDeleteFiles` 优化存储空间
2. **灵活组合**：用户可以根据表的特点选择合适的 Action 组合。如追加表只需要 `RewriteDataFiles`，更新表还需要 `RewritePositionDeleteFiles`

**权衡取舍**：
- **Paimon 的内置 Compaction**：用更高的系统复杂度（需要实现 Compaction 策略、反压机制）和运行时开销（占用 CPU 和内存）换取运维的简洁性和实时性
- **Iceberg 的外部 Compaction**：用更高的运维成本（需要搭建调度系统）和更低的实时性（Compaction 延迟）换取引擎无关性和灵活性

**业界对比**：
- **Delta Lake**：也提供外部 Compaction（`OPTIMIZE` 命令），但与 Spark 耦合更紧密
- **Hudi**：提供内置 Compaction（Async Compaction），类似 Paimon，但实现复杂度更高
- **ClickHouse**：MergeTree 的 Compaction 是内置的，策略类似 Paimon 的 Universal Compaction

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

### 解决什么问题

**核心业务问题**：
- **Paimon 的原生 Changelog**：解决流式消费下游需要完整 CDC 流的问题。下游系统（如 Elasticsearch、MySQL）需要知道每条数据是新增、更新还是删除，以及更新前后的值
- **Iceberg 的增量扫描**：解决批量增量读取的问题。定期（如每小时）读取新增数据，避免全量扫描

**没有这个设计的后果**：
- **没有 Changelog**：下游只能看到最终状态，无法知道数据如何变化。如用户余额从 100 变为 200，下游不知道是充值了 100 还是先消费了 50 再充值了 150
- **没有增量读取**：每次都需要全量扫描表，在大表上（如 TB 级别）性能不可接受
- **没有 Watermark**：流式消费无法与事件时间语义对齐，无法处理乱序数据或触发窗口计算

**实际场景**：
- **Paimon Changelog 场景**：订单表的变更实时同步到 Elasticsearch。订单创建时产出 INSERT 事件，订单状态变更时产出 UPDATE_BEFORE 和 UPDATE_AFTER 事件，订单取消时产出 DELETE 事件。Elasticsearch 根据事件类型执行相应操作
- **Iceberg 增量扫描场景**：日志表每小时追加一次新数据。下游 Spark 作业每小时运行一次，通过 `IncrementalAppendScan` 只读取最近一小时的新数据，避免全量扫描

### 有什么坑

**Paimon 的坑**：
1. **Lookup 模式的性能陷阱**：`changelog.producer = lookup` 模式下，每次写入都需要查找旧值。如果缓存配置不当（`lookup.cache-max-memory` 过小），会导致大量远程读取，严重拖慢写入速度
2. **Full Compaction 模式的延迟**：`changelog.producer = full-compaction` 模式下，Changelog 只在 Full Compaction 时生成。如果 Full Compaction 间隔很长（如数小时），下游消费延迟会很高
3. **Changelog 文件的生命周期**：Changelog 文件有独立的 TTL（`changelog.num-retained.min`），如果配置过短，下游消费者可能读取不到历史 Changelog
4. **Consumer 进度丢失**：如果 Consumer 没有正确提交进度（调用 `ConsumerManager.commitConsumer`），重启后会从头消费，导致重复处理

**Iceberg 的坑**：
1. **IncrementalAppendScan 的局限性**：只能看到追加操作（APPEND），看不到更新和删除。如果表有更新操作，增量扫描会漏掉这些变更
2. **IncrementalChangelogScan 的性能**：需要对比两个 Snapshot 的 Manifest 差异，并应用 Delete File。当 Manifest 和 Delete File 很多时，性能很差
3. **Snapshot 过期导致增量读取失败**：如果消费者的起始 Snapshot 已经被过期清理，增量读取会失败。需要配置足够长的 Snapshot 保留时间
4. **无 Watermark 支持**：Iceberg 的 Snapshot 没有 Watermark 字段，无法与 Flink 的事件时间语义对齐。需要在应用层自己管理 Watermark

### 核心概念解释

**Changelog（变更日志）**：
- 记录数据的变更历史，包括新增、更新、删除
- 每条 Changelog 记录包含操作类型（INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE）和数据内容
- 下游消费者根据操作类型执行相应操作

**ChangelogProducer（变更日志生成器）**：
- Paimon 的 Changelog 生成模式，有 4 种：NONE、INPUT、FULL_COMPACTION、LOOKUP
- 不同模式的性能和完整性不同，需要根据业务需求选择

**Watermark（水位线）**：
- 事件时间语义中的时间进度标记
- 表示早于 Watermark 的数据已经全部到达，可以触发窗口计算
- Paimon 的 Snapshot 内嵌 Watermark，与 Flink 的 Watermark 机制天然对齐

**Consumer（消费者）**：
- 流式消费 Paimon 表的客户端
- Paimon 的 `ConsumerManager` 记录每个消费者的消费位点（`nextSnapshot`）
- 消费者重启后可以从上次位点继续消费，避免重复处理

**IncrementalAppendScan（增量追加扫描）**：
- Iceberg 的增量读取接口，只扫描两个 Snapshot 之间新增（APPEND）的数据文件
- 适用于追加表（Append-Only Table）
- 局限性：看不到更新和删除

**IncrementalChangelogScan（增量变更扫描）**：
- Iceberg 的增量变更读取接口，扫描两个 Snapshot 之间的所有变更
- 产出 `ChangelogScanTask`，包含操作类型（INSERT/DELETE/UPDATE_BEFORE/UPDATE_AFTER）
- 性能较差，因为需要对比 Manifest 差异并应用 Delete File

**deltaManifestList（增量清单列表）**：
- Paimon 的增量元数据，只包含自上次 Snapshot 以来的新增/删除文件
- 流式消费者只需读取 deltaManifestList，无需扫描全量 baseManifestList
- 这是 Paimon 增量读取性能优于 Iceberg 的关键

### 设计理念

**Paimon 为什么提供 4 种 Changelog 模式**：
1. **NONE**：不需要 Changelog 的场景（如只做批量查询），避免额外开销
2. **INPUT**：输入流本身就是完整的 CDC 流（如 MySQL Binlog），直接使用，无需额外计算
3. **FULL_COMPACTION**：对延迟不敏感但需要 Changelog 的场景，在 Full Compaction 时生成，开销低
4. **LOOKUP**：需要低延迟的完整 CDC 输出，实时查找旧值生成 Changelog，开销高但延迟低

**Paimon 为什么内嵌 Watermark**：
1. **流批一体**：Watermark 是流式计算的核心概念，内嵌到 Snapshot 中使得流式消费和批量查询可以共享同一套元数据
2. **事件时间语义**：Paimon 的 Snapshot 按事件时间（而非处理时间）组织，Watermark 标记了事件时间的进度
3. **与 Flink 对齐**：Flink 的 Watermark 机制与 Paimon 的 Watermark 天然对齐，无需额外转换

**Paimon 为什么需要 ConsumerManager**：
1. **进度管理**：流式消费者需要记录消费位点，重启后从上次位点继续消费
2. **Snapshot 保留**：被 Consumer 引用的 Snapshot 不会被过期清理，保证消费者可以回溯
3. **多消费者协调**：多个消费者可以独立消费同一张表，互不干扰

**Iceberg 为什么不提供原生 Changelog**：
1. **批处理为主**：Iceberg 的核心场景是批量分析，不需要完整的 CDC 流
2. **引擎无关性**：Changelog 的生成依赖计算引擎（如 Flink），内置 Changelog 会绑定特定引擎
3. **模型简洁**：不可变文件模型天然不包含变更历史，要生成 Changelog 需要额外的计算

**Iceberg 为什么提供两种增量扫描**：
1. **IncrementalAppendScan**：性能好，适用于追加表（大多数日志类数据）
2. **IncrementalChangelogScan**：功能完整，适用于有更新的表，但性能较差

**权衡取舍**：
- **Paimon 的原生 Changelog**：用更高的系统复杂度（需要实现 Lookup、Compaction 时生成 Changelog）和运行时开销（Lookup 需要查找旧值）换取流式消费的完整性和低延迟
- **Iceberg 的增量扫描**：用功能的局限性（无完整 CDC 流）和性能的牺牲（需要对比 Manifest 差异）换取模型的简洁性和引擎无关性

**业界对比**：
- **Delta Lake**：提供 Change Data Feed（CDF），类似 Paimon 的 Changelog，但需要显式开启且性能不如 Paimon
- **Hudi**：提供 Incremental Query，类似 Iceberg 的增量扫描，但实现复杂度更高
- **Debezium**：专门的 CDC 工具，可以从数据库生成 Changelog，但不是表格式的一部分

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

### 解决什么问题

**核心业务问题**：
- **Paimon 的文件索引**：解决等值查询和范围查询的文件过滤问题。在有数百万个文件的大表上，如果没有索引，查询需要扫描所有文件
- **Paimon 的 Lookup 索引**：解决主键点查的性能问题。在实时场景中，需要根据主键快速查找单条记录（如用户查询、订单查询）
- **Iceberg 的列统计**：解决批量查询的文件过滤问题。通过 min/max 统计信息跳过不相关的文件

**没有这个设计的后果**：
- **没有文件索引**：等值查询（如 `WHERE user_id = 123`）需要扫描所有文件，性能极差。在千万级文件的表上，查询可能需要数分钟甚至数小时
- **没有 Lookup 索引**：主键点查需要扫描整个 Bucket 的所有 Level 文件，性能不可接受（可能需要数秒）
- **没有列统计**：范围查询（如 `WHERE create_time > '2024-01-01'`）无法跳过不相关的文件，导致全表扫描

**实际场景**：
- **Paimon Bloom Filter 场景**：用户表有 10 亿条记录，分布在 10 万个文件中。查询 `WHERE user_id = 123` 时，通过 Bloom Filter 快速过滤掉 99.99% 的文件，只需扫描 1-2 个文件
- **Paimon Lookup 场景**：订单表需要支持根据订单号实时查询订单详情。通过 Lookup 机制，查询延迟可以控制在毫秒级（内存命中）到百毫秒级（磁盘命中）
- **Iceberg 列统计场景**：日志表按天分区，每天有数千个文件。查询 `WHERE timestamp > '2024-01-01 12:00:00'` 时，通过列统计的 min/max 值跳过早于该时间的文件

### 有什么坑

**Paimon 的坑**：
1. **Bloom Filter 的假阳性**：Bloom Filter 是概率型数据结构，存在假阳性（False Positive）。如果 FPP（假阳性率）配置过高，会导致过滤效果差；配置过低，会导致索引文件过大
2. **Bitmap 索引的基数限制**：Bitmap 索引适用于低基数列（如性别、状态）。如果用在高基数列（如用户 ID），索引文件会非常大，甚至超过数据文件本身
3. **Lookup 缓存的内存开销**：`lookup.cache-max-memory` 配置过大会导致 OOM，配置过小会导致缓存命中率低，频繁远程读取
4. **全局索引的维护成本**：Dynamic Bucket 模式下的全局 Hash Index 需要在每次写入时更新。当主键数量达到数亿时，索引更新本身就成为性能瓶颈

**Iceberg 的坑**：
1. **列统计的粒度限制**：列统计只有 min/max/null_count，无法支持等值查询的精确过滤。如查询 `WHERE user_id = 123`，即使文件的 min/max 范围包含 123，也不知道文件中是否真的有这条记录
2. **Bloom Filter 的配置复杂性**：Iceberg V3 的 Bloom Filter 需要手动配置（通过 `write.parquet.bloom-filter-enabled.column.xxx`），且只支持 Parquet 格式
3. **Puffin 文件的兼容性**：Bloom Filter 存储在 Puffin 文件中，老版本引擎无法识别，可能导致查询结果不一致
4. **列统计的更新成本**：每次写入都需要计算列统计（min/max/null_count），对于宽表（如数百列）成本较高

### 核心概念解释

**Bloom Filter（布隆过滤器）**：
- 概率型数据结构，用于判断元素是否可能存在
- 特点：无假阴性（如果返回不存在，则一定不存在），有假阳性（如果返回存在，可能不存在）
- 参数：FPP（假阳性率，默认 0.1）、预期元素数量
- 适用场景：等值查询过滤（如 `WHERE id = 123`）

**Bitmap Index（位图索引）**：
- 每个值一个 Bitmap，标记哪些行包含该值
- 特点：精确匹配，无假阳性
- 适用场景：低基数列的等值/IN 查询（如 `WHERE status IN ('PENDING', 'PROCESSING')`）
- 局限性：高基数列的索引文件会非常大

**BSI（Bit-Sliced Index，位切片索引）**：
- 将数值按位切片，每个位一个 Bitmap
- 特点：支持范围查询（如 `WHERE age > 18`）
- 原理：通过位运算快速计算范围内的行
- 适用场景：数值列的范围查询

**Range Bitmap（范围位图）**：
- 将数值范围划分为多个区间，每个区间一个 Bitmap
- 特点：支持区间过滤，比 BSI 更灵活
- 适用场景：数值列的区间过滤（如 `WHERE price BETWEEN 100 AND 200`）

**Lookup（查找）**：
- Paimon 的主键点查机制，通过内存/磁盘 Hash 表实现 O(1) 查找
- 原理：缓存高层文件到本地，构建 Hash 索引，查询时先查缓存，再查远程文件
- 优化：Bloom Filter 加速（快速判断主键是否存在）、远程文件直接查找（避免下载整个文件）

**LookupLevels（查找层级）**：
- Paimon 的 Lookup 实现，维护多层 Hash 表
- Level-max 的文件缓存到本地（因为最稳定），Level-0 的文件直接查找（因为变化快）
- 查询时从高层到低层依次查找，找到即返回

**列统计（Column Statistics）**：
- 每个数据文件的列级统计信息：min、max、null_count、value_count
- 存储位置：Iceberg 在 manifest entry 中，Paimon 在数据文件的 Footer 中
- 用途：查询规划时跳过不相关的文件（如 `WHERE age > 18` 可以跳过 max < 18 的文件）

**Puffin 文件**：
- Iceberg V3 格式引入的索引文件格式
- 可以存储 Bloom Filter、Sketch（如 HyperLogLog）等索引
- 与数据文件分离，独立管理

### 设计理念

**Paimon 为什么提供 4 种文件索引**：
1. **不同查询模式的需求**：等值查询用 Bloom Filter，低基数列用 Bitmap，范围查询用 BSI/Range Bitmap
2. **性能与空间的权衡**：Bloom Filter 空间小但有假阳性，Bitmap 精确但空间大，BSI 支持范围但计算复杂
3. **灵活组合**：用户可以根据查询模式为不同列配置不同索引（如 `user_id` 用 Bloom Filter，`status` 用 Bitmap，`age` 用 BSI）

**Paimon 为什么需要 Lookup 索引**：
1. **主键表的核心需求**：主键点查是主键表的基本操作（如根据订单号查订单、根据用户 ID 查用户）
2. **实时性要求**：实时场景下，点查延迟需要控制在毫秒级到百毫秒级。全表扫描或 Bucket 扫描都无法满足
3. **Lookup 合并的前提**：`changelog.producer = lookup` 模式下，写入时需要查找旧值。Lookup 索引是这个模式的性能保证

**Paimon 为什么将索引嵌入数据文件**：
1. **减少 I/O**：查询时只需读取一个文件（数据 + 索引），而不是两个文件（数据文件 + 索引文件）
2. **简化管理**：索引与数据的生命周期一致，不需要单独管理索引文件的过期和清理
3. **原子性**：数据和索引在同一个文件中，写入是原子的，不会出现数据和索引不一致的情况

**Iceberg 为什么只提供列统计**：
1. **批处理为主**：批量查询通常是范围查询或聚合查询，列统计的 min/max 足以满足大部分过滤需求
2. **引擎无关性**：列统计是标准的元数据，所有引擎都支持。复杂的索引（如 Bloom Filter）需要引擎特殊支持
3. **模型简洁**：列统计的计算和存储都很简单，不增加系统复杂度

**Iceberg 为什么在 V3 引入 Bloom Filter**：
1. **等值查询的需求**：随着 Iceberg 在实时场景的应用增多，等值查询的需求也增加
2. **Puffin 文件的灵活性**：Puffin 文件可以存储各种索引，未来可以扩展更多索引类型（如 Sketch、倒排索引）
3. **向后兼容**：Puffin 文件是可选的，老版本引擎可以忽略，不影响正确性（只是性能差一些）

**权衡取舍**：
- **Paimon 的丰富索引**：用更高的系统复杂度（需要实现和维护多种索引）和存储开销（索引占用空间）换取查询性能的提升
- **Iceberg 的简单索引**：用查询性能的牺牲（无法精确过滤）换取模型的简洁性和引擎无关性

**业界对比**：
- **Delta Lake**：提供 Data Skipping（类似列统计）和 Z-Ordering（数据排序优化），但没有 Bloom Filter 等索引
- **Hudi**：提供 Bloom Filter 索引，但实现复杂度高，性能不如 Paimon
- **ClickHouse**：提供丰富的索引（Primary Key、Skip Index、Bloom Filter、MinMax），思路与 Paimon 类似

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

### 解决什么问题

**核心业务问题**：
- **小文件问题的根源**：流式写入和高频更新场景下，每次 Checkpoint 或 Commit 都会产生新文件。如果不治理，文件数量会快速增长到数百万甚至数千万
- **小文件的危害**：
  - 元数据膨胀：每个文件都需要元数据（路径、大小、统计信息），数百万文件的元数据可能达到 GB 级别
  - 查询性能下降：查询规划需要扫描所有文件的元数据，文件越多规划越慢
  - 存储效率低：小文件的元数据开销占比高，存储利用率低
  - 对象存储成本高：S3/OSS 等对象存储按请求次数收费，小文件导致请求次数激增

**没有这个设计的后果**：
- **没有自动 Compaction**：Paimon 的查询性能会快速退化（SortedRun 累积），最终系统不可用。Iceberg 的查询规划时间会从秒级增长到分钟级甚至小时级
- **没有 Manifest Compaction**：Manifest 文件碎片化，查询规划需要读取数千个小 Manifest 文件，I/O 成本高
- **没有 Snapshot 过期**：历史 Snapshot 和数据文件无限累积，存储成本失控

**实际场景**：
- **Paimon 自动治理场景**：Flink 流式作业每分钟 Checkpoint 一次，每次产生 100 个小文件（每个并行度一个文件）。一天下来会产生 14.4 万个文件。通过内置 Compaction，文件数量控制在数千个
- **Iceberg 手动治理场景**：Spark 批作业每小时写入一次，每次产生 1000 个小文件。如果不治理，一个月会累积 72 万个文件。需要配置定时任务每天运行 `RewriteDataFiles`，将小文件合并为大文件

### 有什么坑

**Paimon 的坑**：
1. **Compaction 与写入的资源竞争**：Compaction 和写入共享 CPU 和内存。如果 Compaction 配置过于激进（如 `compaction.max-parallel-compactions` 过大），会影响写入性能
2. **Snapshot 过期的时机**：`snapshot.num-retained.min` 配置过小会导致正在被查询的 Snapshot 被过期，查询失败；配置过大会导致历史数据累积，存储成本高
3. **Manifest Compaction 的触发条件**：`MANIFEST_FULL_COMPACTION_FILE_SIZE` 配置不当会导致 Manifest 文件过多或过少。过多会拖慢查询规划，过少会导致频繁的 Manifest Compaction（影响写入性能）
4. **孤儿文件的产生**：如果写入失败（如进程崩溃），可能产生孤儿文件（已写入但未被任何 Snapshot 引用）。需要定期运行 `RemoveOrphanFilesProcedure` 清理

**Iceberg 的坑**：
1. **Compaction 调度的复杂性**：需要搭建外部调度系统（如 Airflow），配置触发条件（如文件数量阈值、文件大小阈值）。调度系统故障会导致小文件失控
2. **RewriteDataFiles 的成本**：重写数据文件的 I/O 成本极高。在大表上（如 TB 级别），一次 Compaction 可能需要数小时，占用大量计算资源
3. **Compaction 与写入的冲突**：Compaction 期间会产生大量 I/O，可能影响正在运行的写入和查询。需要在非高峰时段运行
4. **Delete File 的治理**：高频更新场景下，Delete File 也会快速累积。需要单独运行 `RewritePositionDeleteFiles` 治理，增加运维复杂度

### 核心概念解释

**小文件（Small File）**：
- 通常指小于目标文件大小（如 128MB）的数据文件
- 产生原因：流式写入、高频 Commit、并行度高
- 危害：元数据膨胀、查询性能下降、存储效率低

**Compaction（压缩）**：
- 将多个小文件合并为一个大文件
- 目的：减少文件数量，提高查询性能
- 代价：I/O 成本高，占用计算资源

**Manifest Compaction（清单压缩）**：
- 将多个小 Manifest 文件合并为一个大 Manifest 文件
- 目的：减少 Manifest 文件数量，加速查询规划
- Paimon 在提交时自动触发，Iceberg 需要手动调用 `RewriteManifests`

**Snapshot 过期（Snapshot Expiration）**：
- 删除过期的 Snapshot 及其关联的数据文件和 Manifest 文件
- 目的：释放存储空间，控制元数据大小
- 策略：保留最近 N 个 Snapshot，或保留最近 N 天的 Snapshot

**孤儿文件（Orphan File）**：
- 已写入但未被任何 Snapshot 引用的文件
- 产生原因：写入失败（如进程崩溃）、Compaction 失败
- 危害：占用存储空间，无法被查询到
- 治理：定期运行 `RemoveOrphanFiles` 清理

**反压机制（Backpressure）**：
- 当 Compaction 速度跟不上写入速度时，阻塞写入
- 目的：防止文件数量失控，保证系统稳定
- Paimon 通过 `numSortedRunStopTrigger` 实现，Iceberg 无反压机制

### 设计理念

**Paimon 为什么选择自动治理**：
1. **流式场景的必要性**：流式写入是持续的，不能等到批量治理。自动治理可以在写入的同时异步执行，保持系统稳定
2. **简化运维**：用户只需配置几个参数（如 `target-file-size`、`numSortedRunStopTrigger`），系统自动管理小文件，无需搭建外部调度系统
3. **反压保护**：内置反压机制防止写入速度超过 Compaction 速度，避免系统失控

**Paimon 为什么自动做 Manifest Compaction**：
1. **元数据性能的关键**：Manifest 文件碎片化会严重拖慢查询规划。自动 Compaction 保证 Manifest 文件数量可控
2. **与数据 Compaction 协同**：Manifest Compaction 在提交时触发，与数据 Compaction 协同工作，保证元数据和数据的一致性
3. **增量读取的性能**：Manifest Compaction 将 delta manifest 合并到 base manifest，保证增量读取的性能

**Iceberg 为什么选择外部治理**：
1. **引擎无关性**：Iceberg 作为开放表格式，不绑定任何计算引擎。自动治理需要常驻进程，与引擎无关性矛盾
2. **灵活性**：外部治理可以使用任何计算引擎（Spark、Flink、Trino），用户可以根据自己的技术栈选择
3. **成本控制**：外部治理可以在非高峰时段运行，充分利用闲置资源，降低成本

**Iceberg 为什么提供多种治理 Action**：
1. **不同的治理目标**：`RewriteDataFiles` 治理数据文件，`RewriteManifests` 治理 Manifest 文件，`RemoveOrphanFiles` 治理孤儿文件，`ExpireSnapshots` 治理历史 Snapshot
2. **灵活组合**：用户可以根据表的特点选择合适的 Action 组合。如追加表只需要 `RewriteDataFiles`，更新表还需要 `RewritePositionDeleteFiles`
3. **精细控制**：每个 Action 都有丰富的参数（如 `target-file-size-bytes`、`min-file-size-bytes`），用户可以精细控制治理策略

**权衡取舍**：
- **Paimon 的自动治理**：用更高的系统复杂度（需要实现 Compaction 策略、反压机制）和运行时开销（占用 CPU 和内存）换取运维的简洁性和实时性
- **Iceberg 的外部治理**：用更高的运维成本（需要搭建调度系统）和更低的实时性（治理延迟）换取引擎无关性和灵活性

**业界对比**：
- **Delta Lake**：提供 `OPTIMIZE` 命令（类似 Iceberg 的 `RewriteDataFiles`）和 `VACUUM` 命令（类似 `ExpireSnapshots`），但需要手动触发
- **Hudi**：提供内置 Compaction（Async Compaction），类似 Paimon，但实现复杂度更高
- **ClickHouse**：MergeTree 的 Compaction 是内置的，自动合并小 Part（类似 Paimon 的文件）

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

### 解决什么问题

**核心业务问题**：
- **技术选型的困境**：面对 Paimon 和 Iceberg 两种表格式，如何根据业务场景、技术栈、团队能力做出正确的选择
- **避免选型错误的代价**：选错表格式可能导致性能不达标、运维成本高、迁移成本高等问题
- **长期演进的考虑**：表格式的选择不仅影响当前，还影响未来的技术演进路径

**没有正确选型的后果**：
- **选错 Paimon**：如果业务以批处理为主、需要多引擎查询，Paimon 的优势无法发挥，反而增加系统复杂度
- **选错 Iceberg**：如果业务是高频实时更新，Iceberg 的 Delete File 机制会导致性能雪崩，运维成本失控
- **盲目跟风**：看到别人用 Paimon/Iceberg 就跟着用，不考虑自己的实际场景，导致水土不服

**实际场景**：
- **选 Paimon 的典型场景**：电商公司的实时数仓，订单表、用户表需要秒级更新并实时查询，下游有多个流式消费者（如实时大屏、推荐系统）
- **选 Iceberg 的典型场景**：数据分析公司的数据湖，日志表每天追加一次，需要支持 Spark、Trino、Presto、Athena 等多种引擎查询
- **两者结合的场景**：金融公司的实时风控系统，用 Paimon 做实时写入和流式消费，通过 Iceberg 兼容层对接下游的 BI 工具（如 Tableau、PowerBI）

### 有什么坑

**选型决策的坑**：
1. **只看性能不看运维**：Paimon 的写入性能优于 Iceberg，但需要精心调优 Compaction 参数。如果团队缺乏 LSM-Tree 的运维经验，可能导致系统不稳定
2. **只看生态不看场景**：Iceberg 的引擎生态广泛，但如果业务是高频实时更新，生态再广也无法弥补性能短板
3. **忽视迁移成本**：从 Hive 迁移到 Paimon/Iceberg，或从 Paimon 迁移到 Iceberg（反之亦然），都有较高的迁移成本（数据重写、作业改造、运维流程调整）
4. **过度设计**：为了"未来可能的需求"选择更复杂的方案，导致当前的开发和运维成本增加

**Paimon 选型的坑**：
1. **低估 Compaction 调优的难度**：Compaction 参数（如 `numSortedRunStopTrigger`、`maxSizeAmp`、`sizeRatio`）的调优需要深入理解 LSM-Tree 原理，不是简单的试错可以解决的
2. **高估团队的 Flink 能力**：Paimon 与 Flink 深度集成，如果团队对 Flink 不熟悉（如只用过 Spark），学习曲线会很陡峭
3. **忽视 Bucket 数量的规划**：固定桶模式下，桶数一旦确定无法修改。如果初期规划不当，后期只能重建表
4. **低估流式消费的复杂性**：流式消费需要管理 Consumer 进度、处理 Checkpoint 失败、应对数据倾斜等问题，比批量查询复杂得多

**Iceberg 选型的坑**：
1. **低估 Compaction 调度的复杂性**：搭建 Compaction 调度系统（如 Airflow DAG）、配置触发条件、监控执行状态，都需要额外的开发和运维工作
2. **高估 Delete File 的性能**：在 POC 阶段，Delete File 数量少，性能看起来还可以。但在生产环境，Delete File 累积后性能会急剧下降
3. **忽视云厂商的支持差异**：在国际公有云（AWS/Azure/GCP）上 Iceberg 支持很好，但在国内云厂商（如阿里云）上可能缺乏原生集成
4. **低估流式场景的局限性**：Iceberg 的流式能力有限（无完整 CDC 流、增量读取性能差），如果业务有流式需求，需要额外的开发工作

### 核心概念解释

**技术选型的维度**：
- **性能维度**：写入性能、查询性能、更新性能、点查性能
- **功能维度**：流式能力、索引能力、Schema Evolution、Time Travel
- **生态维度**：引擎支持、云厂商支持、社区活跃度
- **运维维度**：自动化程度、调优难度、监控能力、故障恢复
- **成本维度**：计算成本、存储成本、人力成本

**流批一体（Stream-Batch Unification）**：
- 用同一套存储支持流式处理和批量处理
- Paimon 的核心设计目标，通过 LSM-Tree + Changelog 实现
- Iceberg 主要面向批处理，流式能力有限

**引擎无关性（Engine Agnostic）**：
- 表格式不绑定任何特定的计算引擎
- Iceberg 的核心设计目标，通过开放规范和 REST Catalog 实现
- Paimon 虽然支持多引擎，但与 Flink 耦合更紧密

**运维成本（Operational Cost）**：
- 包括调优成本、监控成本、故障处理成本、人力成本
- Paimon 的自动化程度高，运维成本低，但需要 LSM-Tree 的专业知识
- Iceberg 的运维成本高（需要搭建 Compaction 调度系统），但模型简单易理解

**迁移成本（Migration Cost）**：
- 从现有系统（如 Hive）迁移到新表格式的成本
- 包括数据重写、作业改造、运维流程调整、团队培训
- 需要在选型时充分评估

### 设计理念

**为什么需要选型建议**：
1. **避免盲目跟风**：技术选型不能只看热度和宣传，需要结合实际场景和团队能力
2. **降低试错成本**：表格式的选择影响深远，一旦选错，迁移成本极高
3. **提供决策框架**：通过系统化的对比和分析，帮助用户建立决策框架

**选型的核心原则**：
1. **场景优先**：根据业务场景（流式 vs 批量、高频更新 vs 低频追加）选择合适的表格式
2. **能力匹配**：根据团队的技术栈和能力（Flink vs Spark、LSM-Tree 经验）选择合适的表格式
3. **长期演进**：考虑未来的技术演进路径（如从批处理演进到流批一体）
4. **成本平衡**：在性能、功能、运维成本之间找到平衡点

**Paimon 的适用边界**：
1. **核心优势场景**：实时数据入湖、CDC 同步、高频 Upsert、流式消费
2. **不适用场景**：纯批处理、多引擎混合查询（特别是非 Flink/Spark 引擎）、低频更新
3. **团队要求**：熟悉 Flink、理解 LSM-Tree、有流式处理经验

**Iceberg 的适用边界**：
1. **核心优势场景**：批量分析、多引擎查询、低频更新、国际公有云环境
2. **不适用场景**：高频实时更新、流式消费为主、运维精力有限
3. **团队要求**：熟悉批处理、有调度系统经验、理解分布式存储

**两者结合的可行性**：
1. **技术可行性**：Paimon 的 `paimon-iceberg` 模块支持将 Paimon 表以 Iceberg REST 协议对外暴露
2. **架构模式**：写入侧用 Paimon（高性能流式写入），读取侧用 Iceberg 兼容层（广泛的引擎支持）
3. **适用场景**：需要同时满足实时写入和多引擎查询的场景

**权衡取舍**：
- **Paimon**：用更高的系统复杂度和学习曲线换取流式场景的极致性能和低运维成本
- **Iceberg**：用更新性能的牺牲和更高的运维成本换取引擎无关性和生态广度

**业界实践**：
- **阿里巴巴**：大规模使用 Paimon（前身 Flink Table Store）构建实时数仓
- **Netflix**：Iceberg 的发源地，用于构建 PB 级数据湖
- **Uber**：使用 Hudi（与 Paimon 类似的 MOR 模式）构建实时数据平台
- **Databricks**：推广 Delta Lake（与 Iceberg 类似的不可变文件模式）

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
