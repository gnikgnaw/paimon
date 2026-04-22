# Apache Paimon 详细介绍文档

## 一、项目概述

Apache Paimon 是一个用于构建湖仓架构（Lakehouse Architecture）的湖格式（Lake Format），支持流式和批处理操作。Paimon 为分析提供大规模数据湖存储，通过 LSM（Log-Structured Merge-tree）结构支持实时流式更新，并为 AI 工作负载提供多模态数据管理能力 —— 所有这些都在一个统一的格式中实现。

**当前版本**: 1.5-SNAPSHOT

## 二、核心特性

### 2.1 大规模数据湖

Paimon 专为海量分析数据集而构建。单个表可以包含数十 PB 的数据，即使是如此庞大的表也可以在没有分布式 SQL 引擎的情况下高效读取。

**核心能力**：

- **时间旅行（Time Travel）**：支持可重现的查询，使用完全相同的表快照，或让用户轻松检查变更历史。版本回滚允许用户通过将表重置为良好状态来快速纠正问题。

- **快速扫描规划**：使用表元数据通过分区和列级统计信息对数据文件进行剪枝。文件索引（BloomFilter、Bitmap、Range Bitmap）和聚合下推进一步加速查询。

- **Schema 演进**：支持添加、删除、更新或重命名列，且没有副作用。

- **丰富的生态系统**：将表添加到计算引擎，包括 Flink、Spark、Hive、Trino、Presto、StarRocks 和 Doris，就像使用 SQL 表一样工作。

- **增量聚簇（Incremental Clustering）**：使用 z-order/hilbert/order 排序以低成本优化数据布局。

### 2.2 实时数据湖

Paimon 的主键表（Primary Key Table）将实时流式更新引入湖架构，由 LSM（Log-Structured Merge-tree）结构提供支持。

**核心能力**：

- **大规模流式更新**：具有非常高的性能，通常通过 Flink Streaming 实现。

- **多种合并引擎（Merge Engines）**：
  - **Deduplicate**：去重保留最后一行
  - **Partial Update**：部分更新以逐步完成记录
  - **Aggregation**：聚合值
  - **First Row**：保留最早的记录
  
  可以按照您喜欢的方式更新记录。

- **多种表模式**：
  - **Merge On Read (MOR)**：读时合并
  - **Copy On Write (COW)**：写时复制
  - **Merge On Write (MOW) with Deletion Vectors**：使用删除向量的写时合并
  
  提供灵活的读/写权衡。

- **Changelog 生产者**：None、Input、Lookup、Full Compaction 四种模式为合并引擎生成正确且完整的 changelog，简化流式分析。

- **CDC 数据摄取**：支持从 MySQL、Kafka、MongoDB、Pulsar、PostgreSQL 和 Flink CDC 摄取数据，并支持 schema 演进。

### 2.3 多模态数据湖

Paimon 是面向 AI 的多模态湖仓。将多模态数据、元数据和嵌入保存在同一张表中，并通过向量搜索、全文搜索或 SQL 查询它们。

**核心能力**：

- **数据演进（Data Evolution）**：高效的行级更新和部分列更改，无需重写整个文件 —— 随着应用程序的发展添加新特征（列），无需复制现有数据。

- **Blob 表**：用于存储多模态数据（图像、视频、音频、文档、模型权重），具有分离的存储布局 —— blob 数据存储在专用的 `.blob` 文件中，而元数据保留在标准列式文件中。

- **全局索引（Global Index）**：
  - **BTree Index**：用于高性能标量查找
  - **Vector Index (DiskANN)**：用于近似最近邻搜索

- **PyPaimon**：原生 Python SDK，无需 JDK 依赖，与 Python AI 生态系统（包括 Ray、PyTorch、Pandas 和 PyArrow）无缝集成，用于数据加载、训练和推理工作流。

## 三、架构设计

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    计算引擎层                                 │
│  Flink | Spark | Hive | Trino | Presto | StarRocks | Doris  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    Paimon 表抽象层                            │
│  批处理模式：类似 Hive 表，查询最新快照                        │
│  流式模式：类似消息队列，查询流式 changelog                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    存储引擎层                                 │
│  主键表：LSM Merge-Tree | Append-Only 表：追加式存储          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                文件系统/对象存储                               │
│  HDFS | S3 | OSS | Azure | GCS | COS | OBS | 本地文件系统    │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 文件布局

Paimon 采用分层式文件组织结构，所有文件存储在一个基础目录下：

```
table_path/
├── snapshot/              # 快照文件目录
│   ├── snapshot-1
│   ├── snapshot-2
│   └── ...
├── manifest/              # 清单文件目录
│   ├── manifest-list-xxx
│   ├── manifest-xxx-0
│   └── ...
├── schema/                # Schema 文件目录
│   ├── schema-0
│   └── ...
└── bucket-0/              # 数据文件目录（按分区和桶组织）
    ├── data-xxx.parquet
    └── ...
```

**核心组件**：

1. **Snapshot（快照）**：
   - 捕获表在某个时间点的状态
   - 包含使用的 schema 文件信息
   - 包含此快照所有变更的 manifest list
   - 用户可以通过最新快照访问表的最新数据
   - 通过时间旅行可以访问表的历史状态

2. **Manifest Files（清单文件）**：
   - **Manifest List**：manifest 文件名列表
   - **Manifest File**：包含 LSM 数据文件和 changelog 文件的变更信息
   - 记录哪些 LSM 数据文件被创建，哪些文件被删除

3. **Data Files（数据文件）**：
   - 按分区分组
   - 支持 Parquet（默认）、ORC 和 Avro 格式

4. **Partition（分区）**：
   - 采用与 Apache Hive 相同的分区概念
   - 基于特定列（如日期、城市、部门）的值将表划分为相关部分
   - 每个表可以有一个或多个分区键来标识特定分区

### 3.3 读写路径

**读取路径**：
```
Table.newReadBuilder() 
  → ReadBuilder 
  → TableScan (生成 Split[]) 
  → TableRead (读取 splits)
```

**写入路径**：
```
Table.newBatchWriteBuilder() / newStreamWriteBuilder() 
  → TableWrite (写入数据) 
  → TableCommit (原子提交)
```

## 四、表类型

### 4.1 主键表（Primary Key Table）

主键表使用 LSM Merge-Tree 结构，支持高性能的流式更新。

**创建示例**：
```sql
CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    PRIMARY KEY (dt, user_id, item_id) NOT ENFORCED
) PARTITIONED BY (dt);
```

**合并引擎**：

1. **Deduplicate（默认）**：只保留最新记录，丢弃具有相同主键的其他记录
2. **Partial Update**：部分更新，逐步完成记录
3. **Aggregation**：聚合相同主键的记录
4. **First Row**：保留最早的记录

**表模式**：

- **MOR (Merge On Read)**：读时合并，写入快，查询时需要合并
- **COW (Copy On Write)**：写时复制，写入慢，查询快
- **MOW (Merge On Write) with Deletion Vectors**：使用删除向量，平衡读写性能

### 4.2 Append-Only 表

如果表没有定义主键，则为 append-only 表。与主键表相比，它不能直接接收 changelog，不能通过 upsert 直接更新数据，只能接收追加数据。

**创建示例**：
```sql
CREATE TABLE my_table (
    product_id BIGINT,
    price DOUBLE,
    sales BIGINT
);
```

**适用场景**：

- 批量写入和批量读取的典型应用场景
- 类似于常规的 Hive 分区表
- 流式读写，像队列一样使用
- 支持 DELETE / UPDATE / MERGE INTO 的低成本行级操作

**优势**：

1. 时间旅行和版本回滚
2. 快速扫描规划（分区和列级统计信息剪枝）
3. Schema 演进
4. 丰富的生态系统支持
5. 增量聚簇优化数据布局
6. 流式读写支持

## 五、CDC 数据摄取

Paimon 支持多种方式将数据摄取到 Paimon 表中，并支持 schema 演进。

### 5.1 支持的数据源

1. **MySQL 同步**：
   - 同步单表或多表到一个 Paimon 表
   - 同步整个 MySQL 数据库到一个 Paimon 数据库

2. **Kafka 同步**：
   - 同步一个 Kafka topic 的表到一个 Paimon 表
   - 同步包含多个表的一个 topic 或每个表一个 topic 到一个 Paimon 数据库

3. **MongoDB 同步**：
   - 同步一个 Collection 到一个 Paimon 表
   - 同步整个 MongoDB 数据库到一个 Paimon 数据库

4. **Pulsar 同步**：
   - 同步一个 Pulsar topic 的表到一个 Paimon 表
   - 同步包含多个表的一个 topic 或每个表一个 topic 到一个 Paimon 数据库

5. **PostgreSQL 同步**

6. **Flink CDC 同步**

7. **程序 API 同步**：同步自定义 DataStream 输入到一个 Paimon 表

### 5.2 Schema 演进

**什么是 Schema 演进**：

在传统的 Flink SQL 中，如果在数据摄取后更改 MySQL 表的 schema，表 schema 更改不会同步到 Paimon。

使用 Paimon 的 CDC 同步（如 MySqlSyncTableAction），如果在数据摄取后更改 MySQL 表的 schema，表 schema 更改将同步到 Paimon，新添加的字段数据也将同步到 Paimon。

**支持的 Schema 变更**：

1. **添加列**
2. **修改列类型**（有限制）：
   - 字符串类型到更长的字符串类型
   - 非字符串类型到字符串类型
   - 二进制类型到更长的二进制类型
   - 整数类型到更宽范围的整数类型
   - 浮点类型到更宽范围的浮点类型

**不支持的操作**：

- 重命名表（会被忽略）
- 删除列（会被忽略）
- 重命名列（会添加新列）

### 5.3 计算函数

**时间函数**：用于将日期和 epoch 时间转换为另一种形式，常见用例是生成分区值。

支持的函数包括：
- `year(date_col)`
- `month(date_col)`
- `day(date_col)`
- `hour(timestamp_col)`
- `date_format(temporal_col, format_string)`
- 等等

**其他函数**：用于数据转换和处理。

## 六、生态系统集成

### 6.1 兼容性矩阵

| 引擎 | 版本 | 批读 | 批写 | 创建表 | 修改表 | 流写 | 流读 | 批覆盖 | DELETE & UPDATE | MERGE INTO | 时间旅行 |
|------|------|------|------|--------|--------|------|------|--------|-----------------|------------|----------|
| Flink | 1.16-1.20 | ✅ | ✅ | ✅ | ✅(1.17+) | ✅ | ✅ | ✅ | ✅(1.17+) | ❌ | ✅ |
| Spark | 3.2-4.0 | ✅ | ✅ | ✅ | ✅ | ✅(3.3+) | ✅(3.3+) | ✅ | ✅ | ✅ | ✅(3.3+) |
| Hive | 2.1-3.1 | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Trino | 420-440 | ✅ | ✅(427+) | ✅(427+) | ✅(427+) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Presto | 0.236-0.280 | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| StarRocks | 3.1+ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Doris | 2.0.6+ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

### 6.2 推荐引擎

**流式引擎**：
- **Flink Streaming**：最全面的流计算引擎，广泛用于数据 CDC 摄取和流式管道构建（推荐版本：Flink 1.17.2）
- **Spark Streaming**：也可以使用 Spark Streaming 构建流式管道，schema 演进能力更好，但必须接受 mini-batch 机制

**批处理引擎**：
- **Spark Batch**：最广泛使用的批处理计算引擎（推荐版本：Spark 3.5.8）
- **Flink Batch**：也可用，可以使流式和批处理统一，使管道更加集成

**OLAP 引擎**：
- **StarRocks**：最推荐的 OLAP 引擎，具有最先进的集成（推荐版本：StarRocks 3.2.6）
- **其他 OLAP**：也可以使用 Doris、Trino 和 Presto，或者直接使用 Spark、Flink 和 Hive 查询 Paimon 表

## 七、一致性保证

Paimon 写入器使用两阶段提交协议原子地将一批记录提交到表中。每次提交在提交时最多产生两个快照：

1. 如果仅执行增量写入而不触发压缩操作，则只会创建一个增量快照
2. 如果触发压缩操作，则会创建一个增量快照和一个压缩快照

对于同时修改表的任意两个写入器：
- 只要它们不修改同一分区，它们的提交可以并行发生
- 如果它们修改同一分区，则只保证快照隔离
- 最终表状态可能是两次提交的混合，但不会丢失任何更改

## 八、数据跳过与优化

### 8.1 按顺序跳过数据

Paimon 默认在 manifest 文件中记录每个字段的最大值和最小值。

在查询中，根据查询的 WHERE 条件，结合 manifest 中的统计信息，可以执行文件过滤。如果过滤效果好，原本需要几分钟的查询可以加速到毫秒级完成。

可以通过排序压缩来优化数据分布，提高过滤效果。

### 8.2 文件索引跳过数据

Paimon 支持文件索引，通过在读取端建立索引来过滤文件。

**支持的索引类型**：

1. **BloomFilter**：适用于点查询场景
2. **Bitmap**：可能消耗更多空间但可以获得更高的准确性
3. **Range Bitmap**：范围位图索引

**配置方式**：
- `file-index.bloom-filter.columns`
- `file-index.bitmap.columns`
- `file-index.range-bitmap.columns`

### 8.3 聚合下推

Append 表支持聚合下推：

```sql
SELECT COUNT(*) FROM TABLE WHERE DT = '20230101';
```

此查询可以在编译期间加速并非常快速地返回。

对于 Spark SQL，还支持：

```sql
SELECT MIN(a), MAX(b) FROM TABLE WHERE DT = '20230101';
SELECT * FROM TABLE ORDER BY a LIMIT 1;
```

Min、Max、TopN 查询也可以在编译期间加速。

## 九、行级操作

### 9.1 DELETE 操作

```sql
DELETE FROM my_table WHERE currency = 'UNKNOWN';
```

### 9.2 UPDATE 操作

更新 append 表有两种模式：

1. **COW (Copy on Write)**：搜索命中的文件，然后重写每个文件以从文件中删除需要删除的数据。此操作成本高。

2. **MOW (Merge on Write)**：通过指定 `'deletion-vectors.enabled' = 'true'`，可以启用 Deletion Vectors 模式。仅标记相应文件的某些记录为删除并写入删除文件，而不重写整个文件。

### 9.3 MERGE INTO 操作

Spark SQL 支持 MERGE INTO 操作，可以实现复杂的 upsert 逻辑。

## 十、统一存储

对于像 Apache Flink 这样的流式引擎，通常有三种类型的连接器：

1. **消息队列**（如 Apache Kafka）：用于源和中间阶段，保证延迟在秒级
2. **OLAP 系统**（如 ClickHouse）：以流式方式接收处理后的数据并服务用户的即席查询
3. **批存储**（如 Apache Hive）：支持传统批处理的各种操作，包括 INSERT OVERWRITE

Paimon 提供表抽象，使用方式与传统数据库无异：

- 在**批处理执行模式**下，它像 Hive 表一样工作，支持各种批处理 SQL 操作。查询它可以看到最新快照。
- 在**流式执行模式**下，它像消息队列一样工作。查询它就像从消息队列查询流式 changelog，其中历史数据永不过期。

## 十一、快速开始

### 11.1 使用 Flink

参考 Flink 快速开始指南，它提供了 API 的逐步介绍，并指导您完成实际应用。

### 11.2 使用 Spark

参考 Spark 快速开始指南，了解如何在 Spark 中使用 Paimon。

## 十二、获取帮助

如果遇到问题，可以：

1. 订阅用户邮件列表：user-subscribe@paimon.apache.org
2. 在 GitHub 上创建 issue
3. 提交 pull request 贡献代码

## 十三、总结

Apache Paimon 是一个功能强大的湖格式，它：

1. **统一了流批处理**：在一个统一的格式中支持流式和批处理操作
2. **提供实时更新能力**：通过 LSM 结构支持高性能的流式更新
3. **支持多模态数据**：为 AI 工作负载提供多模态数据管理能力
4. **生态系统丰富**：与 Flink、Spark、Hive、Trino、StarRocks、Doris 等多个计算引擎集成
5. **性能优异**：通过文件索引、聚合下推、增量聚簇等技术优化查询性能
6. **易于使用**：提供简单的 SQL 接口和丰富的 CDC 数据摄取能力

Paimon 适用于构建实时数仓、CDC 数据同步、流批一体、AI 数据湖等多种场景，是构建现代数据架构的理想选择。
