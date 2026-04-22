# Apache Paimon Deletion Vectors 与文件索引深度分析

> 基于 Paimon 1.5-SNAPSHOT 源码分析，commit: 55f4fd175
> 分析日期: 2026-04-21

---

## 目录

- [1. DeletionVector 接口体系](#1-deletionvector-接口体系)
  - [1.1 继承层次总览](#11-继承层次总览)
  - [1.2 DeletionVector 接口定义](#12-deletionvector-接口定义)
  - [1.3 BitmapDeletionVector (V1)](#13-bitmapdeletionvector-v1)
  - [1.4 Bitmap64DeletionVector (V2)](#14-bitmap64deletionvector-v2)
  - [1.5 V1/V2 序列化格式与 Magic Number](#15-v1v2-序列化格式与-magic-number)
  - [1.6 反序列化分派机制](#16-反序列化分派机制)
  - [1.7 Factory 模式与延迟加载](#17-factory-模式与延迟加载)
- [2. DV 文件组织](#2-dv-文件组织)
  - [2.1 DeletionVectorsIndexFile 索引文件](#21-deletionvectorsindexfile-索引文件)
  - [2.2 DeletionFileWriter 写入器](#22-deletionfilewriter-写入器)
  - [2.3 DeletionVectorMeta 元数据](#23-deletionvectormeta-元数据)
  - [2.4 DeletionVectorIndexFileWriter 与 Rolling 写入](#24-deletionvectorindexfilewriter-与-rolling-写入)
  - [2.5 DV 索引文件物理布局](#25-dv-索引文件物理布局)
- [3. BucketedDvMaintainer 维护机制](#3-bucketeddvmaintainer-维护机制)
  - [3.1 核心职责与数据结构](#31-核心职责与数据结构)
  - [3.2 notifyNewDeletion 增量删除通知](#32-notifynewdeletion-增量删除通知)
  - [3.3 mergeNewDeletion 合并删除向量](#33-mergenewdeletion-合并删除向量)
  - [3.4 removeDeletionVectorOf 移除操作](#34-removedeletionvectorof-移除操作)
  - [3.5 Factory 模式与状态恢复](#35-factory-模式与状态恢复)
  - [3.6 写入输出与 modified 跟踪](#36-写入输出与-modified-跟踪)
- [4. DV 在读路径中的应用](#4-dv-在读路径中的应用)
  - [4.1 ApplyDeletionVectorReader 行级过滤](#41-applydeletionvectorreader-行级过滤)
  - [4.2 ApplyDeletionFileRecordIterator 迭代过滤](#42-applydeletionfilerecorditerator-迭代过滤)
  - [4.3 读路径全链路分析](#43-读路径全链路分析)
- [5. DV 在写路径中的应用](#5-dv-在写路径中的应用)
  - [5.1 Lookup 模式下标记旧行删除](#51-lookup-模式下标记旧行删除)
  - [5.2 Compaction 时 DV 合并](#52-compaction-时-dv-合并)
  - [5.3 DV 生命周期总结](#53-dv-生命周期总结)
- [6. Append 表的 DV](#6-append-表的-dv)
  - [6.1 BaseAppendDeleteFileMaintainer 接口](#61-baseappenddeletefilemaintainer-接口)
  - [6.2 AppendDeleteFileMaintainer (Unaware Bucket)](#62-appenddeletefilemaintainer-unaware-bucket)
  - [6.3 BucketedAppendDeleteFileMaintainer (Bucketed)](#63-bucketedappenddeletefilemaintainer-bucketed)
  - [6.4 Append 表与 PK 表 DV 差异对比](#64-append-表与-pk-表-dv-差异对比)
- [7. 文件索引 SPI 体系](#7-文件索引-spi-体系)
  - [7.1 FileIndexer/FileIndexerFactory 接口设计](#71-fileindexerfileindexerfactory-接口设计)
  - [7.2 SPI 加载机制](#72-spi-加载机制)
  - [7.3 FileIndexFormat 存储格式 (Header + Body)](#73-fileindexformat-存储格式-header--body)
  - [7.4 FileIndexWriter 写入接口](#74-fileindexwriter-写入接口)
  - [7.5 FileIndexReader 读取接口](#75-fileindexreader-读取接口)
- [8. Bloom Filter 索引](#8-bloom-filter-索引)
  - [8.1 BloomFilterFileIndex 实现](#81-bloomfilterfileindex-实现)
  - [8.2 BloomFilter64 底层结构](#82-bloomfilter64-底层结构)
  - [8.3 FastHash 哈希策略](#83-fasthash-哈希策略)
  - [8.4 序列化格式](#84-序列化格式)
  - [8.5 仅支持 Equal 的设计决策](#85-仅支持-equal-的设计决策)
  - [8.6 参数调优 (items/fpp)](#86-参数调优-itemsfpp)
- [9. Bitmap 倒排索引](#9-bitmap-倒排索引)
  - [9.1 BitmapFileIndex 核心思想](#91-bitmapfileindex-核心思想)
  - [9.2 V1 格式 (BitmapFileIndexMeta)](#92-v1-格式-bitmapfileindexmeta)
  - [9.3 V2 格式 (BitmapFileIndexMetaV2) 二级索引优化](#93-v2-格式-bitmapfileindexmetav2-二级索引优化)
  - [9.4 支持的谓词类型](#94-支持的谓词类型)
  - [9.5 单值优化](#95-单值优化)
  - [9.6 ApplyBitmapIndexRecordReader 行级下推](#96-applybitmapindexrecordreader-行级下推)
- [10. BSI (Bit-Sliced Index)](#10-bsi-bit-sliced-index)
  - [10.1 BitSliceIndexBitmapFileIndex 核心原理](#101-bitsliceindexbitmapfileindex-核心原理)
  - [10.2 正负数分组策略](#102-正负数分组策略)
  - [10.3 范围查询 O(bits) 复杂度](#103-范围查询-obits-复杂度)
  - [10.4 值映射 (Value Mapper)](#104-值映射-value-mapper)
  - [10.5 支持的数据类型与谓词](#105-支持的数据类型与谓词)
- [11. Range Bitmap 索引](#11-range-bitmap-索引)
  - [11.1 RangeBitmapFileIndex 架构](#111-rangebitmapfileindex-架构)
  - [11.2 ChunkedDictionary 分块字典](#112-chunkeddictionary-分块字典)
  - [11.3 BitSliceIndexBitmap 内部 BSI](#113-bitsliceindexbitmap-内部-bsi)
  - [11.4 TopN 查询加速](#114-topn-查询加速)
  - [11.5 与 BSI 索引的差异](#115-与-bsi-索引的差异)
- [12. FileIndexResult 三态逻辑](#12-fileindexresult-三态逻辑)
  - [12.1 REMAIN/SKIP/BitmapIndexResult 三种状态](#121-remainskipbitmapindexresult-三种状态)
  - [12.2 AND/OR 运算语义](#122-andor-运算语义)
  - [12.3 BitmapIndexResult 的延迟计算](#123-bitmapindexresult-的延迟计算)
- [13. FileIndexPredicate 评估流程](#13-fileindexpredicate-评估流程)
  - [13.1 谓词评估入口](#131-谓词评估入口)
  - [13.2 FileIndexPredicateTest 访问者模式](#132-fileindexpredicatetest-访问者模式)
  - [13.3 TopN 评估](#133-topn-评估)
- [14. 全局索引](#14-全局索引)
  - [14.1 GlobalIndexMeta 架构](#141-globalindexmeta-架构)
  - [14.2 BTree 全局索引](#142-btree-全局索引)
  - [14.3 用途: 跨分区更新](#143-用途-跨分区更新)
- [15. Predicate 体系](#15-predicate-体系)
  - [15.1 LeafPredicate 叶子谓词](#151-leafpredicate-叶子谓词)
  - [15.2 CompoundPredicate 组合谓词](#152-compoundpredicate-组合谓词)
  - [15.3 FunctionVisitor 双重分派](#153-functionvisitor-双重分派)
  - [15.4 Predicate 多层下推](#154-predicate-多层下推)
- [16. Lookup 机制](#16-lookup-机制)
  - [16.1 StateFactory 接口](#161-statefactory-接口)
  - [16.2 RocksDB 状态后端](#162-rocksdb-状态后端)
  - [16.3 InMemory 状态后端](#163-inmemory-状态后端)
  - [16.4 LRU Cache 策略](#164-lru-cache-策略)
- [17. 与 Iceberg DV 对比](#17-与-iceberg-dv-对比)

---

## 1. DeletionVector 接口体系

### 1.1 继承层次总览

```
DeletionVectorJudger (接口, 仅提供 isDeleted(long) 判定能力)
  |
  +-- DeletionVector (接口, 完整 DV 操作能力)
        |
        +-- BitmapDeletionVector     (V1, 基于 RoaringBitmap32, 最大 2^31-1 行)
        |   源码: paimon-core/.../deletionvectors/BitmapDeletionVector.java
        |
        +-- Bitmap64DeletionVector   (V2, 基于 OptimizedRoaringBitmap64, 最大 2^63 行)
            源码: paimon-core/.../deletionvectors/Bitmap64DeletionVector.java
```

**为什么采用两层接口设计**: `DeletionVectorJudger` 仅暴露 `isDeleted(long)` 方法，这是读路径唯一需要的能力。`DeletionVector` 继承 `DeletionVectorJudger` 并添加了写入能力 (`delete`, `merge`, `serializeTo`)。这种分离遵循接口隔离原则，读路径的组件不需要感知 DV 的写入细节。

### 1.2 DeletionVector 接口定义

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/DeletionVector.java` (L44-L196)

```java
public interface DeletionVector extends DeletionVectorJudger {
    // 写入操作
    void delete(long position);                    // 标记指定行位置为已删除
    void merge(DeletionVector deletionVector);     // 合并另一个 DV (OR 语义)
    boolean checkedDelete(long position);           // 标记删除并返回是否为新增删除 (L66-L73)
    
    // 状态查询
    boolean isEmpty();                              // 是否没有任何删除标记
    long getCardinality();                          // 已删除的行数
    
    // 序列化
    int serializeTo(DataOutputStream out);          // 序列化到输出流，返回写入字节数
    
    // 继承自 DeletionVectorJudger
    boolean isDeleted(long position);               // 判断某行是否被标记为删除
}
```

接口还提供了一系列静态工厂方法:

- `read(FileIO, DeletionFile)` (L88-L95): 从文件系统读取 DV，通过 `DeletionFile` 中的 path/offset/length 定位
- `read(DataInputStream, Long)` (L97-L146): 核心反序列化方法，通过 Magic Number 判断 V1/V2 格式
- `factory(BucketedDvMaintainer)` (L152-L157): 创建 DV 工厂，委托给 `dvMaintainer::deletionVectorOf`
- `serializeToBytes` / `deserializeFromBytes` (L171-L190): 便捷的序列化/反序列化工具方法

`checkedDelete` 方法 (L66-L73) 的默认实现值得关注: 它先检查 `isDeleted`，如果已存在则返回 `false`（不是新删除），否则调用 `delete` 并返回 `true`。`BitmapDeletionVector` 覆盖了这个方法 (L65-L68)，直接调用底层 `RoaringBitmap32.checkedAdd`，原子地完成检查与添加，避免了两次查找。

**为什么 checkedDelete 需要返回 boolean**: 调用方（如 `BucketedDvMaintainer.notifyNewDeletion`）通过返回值判断是否产生了新的修改，从而设置 `modified` 标记。如果一个 position 已经被删除过，重复删除不应该触发新的 DV 文件写入。

### 1.3 BitmapDeletionVector (V1)

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/BitmapDeletionVector.java` (L34-L147)

核心实现:
- 底层数据结构: `RoaringBitmap32` -- Roaring Bitmap 的 32 位版本
- 最大行数限制: `RoaringBitmap32.MAX_VALUE` (2^31 - 1 = 2,147,483,647)
- Magic Number: `1581511376` (L36)

```java
public class BitmapDeletionVector implements DeletionVector {
    public static final int MAGIC_NUMBER = 1581511376;
    private final RoaringBitmap32 roaringBitmap;
    
    public void delete(long position) {
        checkPosition(position);              // 校验不超过 Integer.MAX_VALUE
        roaringBitmap.add((int) position);    // 强转为 int 存储
    }
    
    public boolean isDeleted(long position) {
        checkPosition(position);
        return roaringBitmap.contains((int) position);
    }
    
    public void merge(DeletionVector deletionVector) {
        if (deletionVector instanceof BitmapDeletionVector) {
            roaringBitmap.or(((BitmapDeletionVector) deletionVector).roaringBitmap);
        } else {
            throw new RuntimeException("Only instance with the same class type can be merged.");
        }
    }
}
```

序列化格式 (`serializeTo` 方法, L87-L101):
1. 先将 `Magic Number + Bitmap 数据` 写入临时缓冲区
2. 写入缓冲区长度 (4 字节 int)
3. 写入缓冲区内容
4. 写入 CRC32 校验和 (4 字节 int)

**为什么 V1 中 position 要 checkPosition**: V1 使用 32 位 Roaring Bitmap，position 被强转为 int。如果文件行数超过 2^31-1 但使用了 V1 格式，会导致溢出。`checkPosition` (L118-L123) 在这种情况下抛出 `IllegalArgumentException`，快速失败而非静默数据错误。

### 1.4 Bitmap64DeletionVector (V2)

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/Bitmap64DeletionVector.java` (L38-L189)

核心实现:
- 底层数据结构: `OptimizedRoaringBitmap64` -- Roaring Bitmap 的 64 位版本
- 最大行数: `OptimizedRoaringBitmap64.MAX_VALUE` (理论支持 2^63 行)
- Magic Number: `1681511377` (L40)
- 注释: "Mostly copied from iceberg" (L37) -- 与 Iceberg V3 DV 格式兼容

```java
public class Bitmap64DeletionVector implements DeletionVector {
    public static final int MAGIC_NUMBER = 1681511377;
    private final OptimizedRoaringBitmap64 roaringBitmap;
    
    public void delete(long position) {
        roaringBitmap.add(position);   // 直接使用 long，无需类型转换
    }
    
    public int serializeTo(DataOutputStream out) throws IOException {
        roaringBitmap.runLengthEncode();  // 序列化前做 RLE 编码压缩
        // ... 写入 [length][magic+bitmap_data][crc]
    }
}
```

关键设计差异:
1. **字节序**: V2 使用 **Little-Endian** 字节序 (L125-L135)，与 Iceberg DV 兼容; V1 使用 Java 默认的 **Big-Endian**
2. **无需 checkPosition**: `long` 类型直接存储，无溢出风险
3. **RLE 编码**: 序列化前调用 `runLengthEncode()` (L94)，对连续的删除位置做游程编码压缩
4. **类型转换**: 提供 `fromBitmapDeletionVector` (L56-L61) 方法，支持从 V1 升级到 V2

### 1.5 V1/V2 序列化格式与 Magic Number

**V1 格式 (BitmapDeletionVector):**

```
+------------------+--------------------+--------------------+------------+
| Bitmap Length    | Magic Number        | Roaring Bitmap Data | CRC32     |
| (4 bytes, BE)    | (4 bytes, BE)       | (变长)              | (4 bytes) |
| = Magic+Data长度 | 1581511376          |                     |           |
+------------------+--------------------+--------------------+------------+
                   |<--- Bitmap Length 覆盖的范围 --->|
```

- `Bitmap Length`: Magic Number + Bitmap 数据的总长度
- CRC32 校验范围: Magic Number + Bitmap 数据

**V2 格式 (Bitmap64DeletionVector):**

```
+------------------+--------------------+--------------------+------------+
| Bitmap Length    | Magic Number        | Bitmap64 Data      | CRC32     |
| (4 bytes, BE)    | (4 bytes, LE!)      | (变长, LE)         | (4 bytes) |
| = Magic+Data长度 | 1681511377          |                     |           |
+------------------+--------------------+--------------------+------------+
                   |<--- Bitmap Length 覆盖的范围 --->|
```

- `Bitmap Length`: 注意这里记录的是 `Magic + Bitmap Data` 的长度 (不含 Length 字段和 CRC 字段本身)
- Magic Number 使用 Little-Endian 写入
- CRC32 校验范围: Magic Number + Bitmap64 数据

**为什么选择不同的字节序**: V2 设计参考了 Iceberg 的 DV 格式，Iceberg 使用 Little-Endian（与 Roaring Bitmap 的原生序列化一致）。V1 使用 Java 标准的 Big-Endian。反序列化时通过 `toLittleEndianInt()` (L168-L171) 方法做字节序转换来识别 Magic Number。

### 1.6 反序列化分派机制

`DeletionVector.read(DataInputStream, Long)` (L97-L146) 是 DV 反序列化的核心入口:

```java
static DeletionVector read(DataInputStream dis, @Nullable Long length) throws IOException {
    int bitmapLength = dis.readInt();      // 读取 bitmap 数据长度
    int magicNumber = dis.readInt();       // 读取 Magic Number
    
    if (magicNumber == BitmapDeletionVector.MAGIC_NUMBER) {
        // V1: Big-Endian Magic, 直接匹配
        // 校验 length 一致性...
        byte[] bytes = new byte[bitmapLength - MAGIC_NUMBER_SIZE_BYTES];
        dis.readFully(bytes);
        dis.skipBytes(4);  // 跳过 CRC
        return BitmapDeletionVector.deserializeFromByteBuffer(ByteBuffer.wrap(bytes));
    } else if (toLittleEndianInt(magicNumber) == Bitmap64DeletionVector.MAGIC_NUMBER) {
        // V2: Little-Endian Magic, 需要字节序转换后匹配
        // 校验 length 一致性 (扣除 LENGTH_SIZE_BYTES 和 CRC_SIZE_BYTES)...
        byte[] bytes = new byte[bitmapLength - MAGIC_NUMBER_SIZE_BYTES];
        dis.readFully(bytes);
        dis.skipBytes(4);  // 跳过 CRC
        return Bitmap64DeletionVector.deserializeFromBitmapDataBytes(bytes);
    } else {
        throw new RuntimeException("Invalid magic number: " + magicNumber);
    }
}
```

**为什么通过 Magic Number 而非版本号区分格式**: DV 数据可能存储在不同版本的文件中混合存在。Magic Number 自包含在每个 DV 数据块内，使得每个 DV 都能独立反序列化，无需外部元数据辅助。注意 V2 的 `bitmapLength` 含义与 V1 不同: V2 中 `length = bitmapLength + LENGTH_SIZE_BYTES + CRC_SIZE_BYTES`。

### 1.7 Factory 模式与延迟加载

接口提供了三种 Factory 模式 (L148-L169):

```java
// 1. 空工厂 - 永远返回 Optional.empty()
static Factory emptyFactory() { return fileName -> Optional.empty(); }

// 2. 基于 DvMaintainer 的工厂 - 从内存中的 map 查找
static Factory factory(@Nullable BucketedDvMaintainer dvMaintainer) {
    if (dvMaintainer == null) return emptyFactory();
    return dvMaintainer::deletionVectorOf;   // 方法引用
}

// 3. 基于文件的工厂 - 按需从文件系统读取 DV
static Factory factory(FileIO fileIO, List<DataFileMeta> files, 
                        @Nullable List<DeletionFile> deletionFiles) {
    DeletionFile.Factory factory = DeletionFile.factory(files, deletionFiles);
    return fileName -> {
        Optional<DeletionFile> deletionFile = factory.create(fileName);
        if (deletionFile.isPresent()) {
            return Optional.of(DeletionVector.read(fileIO, deletionFile.get()));
        }
        return Optional.empty();
    };
}
```

**为什么需要 Factory 模式**: 读路径中一个 Split 可能包含多个数据文件，每个文件可能有或没有 DV。Factory 模式允许调用方只在需要时才加载 DV，避免一次性加载所有 DV 到内存。方案 3 的延迟加载特性尤其重要: DV 文件可能很大，按需读取可以减少内存压力。

---

## 2. DV 文件组织

### 2.1 DeletionVectorsIndexFile 索引文件

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/DeletionVectorsIndexFile.java` (L44-L173)

这是 DV 索引文件的核心抽象，继承自 `IndexFile`，负责读写 DV 索引文件。

关键常量:
```java
public static final String DELETION_VECTORS_INDEX = "DELETION_VECTORS";  // 索引类型标识
public static final byte VERSION_ID_V1 = 1;                              // 文件版本号
```

构造参数:
```java
public DeletionVectorsIndexFile(
    FileIO fileIO,
    IndexPathFactory pathFactory,
    MemorySize targetSizePerIndexFile,   // Rolling 写入时的目标文件大小
    boolean bitmap64                      // 是否使用 V2 (64位) 格式
)
```

**读取方法**:

1. `readAllDeletionVectors(IndexFileMeta)` (L73-L96): 顺序读取一个索引文件中的所有 DV
   - 从 `IndexFileMeta.dvRanges()` 获取每个数据文件的 DV 偏移量信息
   - 打开索引文件，先校验版本号 (1字节)
   - 按照 dvRanges 中的顺序依次反序列化 DV
   
2. `readDeletionVector(Map<String, DeletionFile>)` (L105-L127): 按需读取指定数据文件的 DV
   - 所有 DeletionFile 必须属于同一个索引文件
   - 通过 `seek` 跳转到指定偏移量读取

3. `readDeletionVector(DeletionFile)` (L129-L140): 读取单个 DV

**写入方法**:

1. `writeSingleFile(Map<String, DeletionVector>)` (L142-L148): 将所有 DV 写入单个文件
2. `writeWithRolling(Map<String, DeletionVector>)` (L150-L155): 按目标大小分文件写入

**版本校验**: `checkVersion` (L163-L172) 读取文件的第一个字节，验证必须是 `VERSION_ID_V1 = 1`。

**为什么 DV 索引文件需要版本号**: 区别于 DV 数据本身的 V1/V2 (通过 Magic Number 区分)，索引文件级别的版本号用于未来扩展索引文件的整体结构（如增加压缩、增加全局元数据等）。

### 2.2 DeletionFileWriter 写入器

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/DeletionFileWriter.java` (L36-L76)

`DeletionFileWriter` 负责将多个 DV 写入单个物理文件:

```java
public class DeletionFileWriter implements Closeable {
    private final Path path;
    private final boolean isExternalPath;
    private final DataOutputStream out;
    private final LinkedHashMap<String, DeletionVectorMeta> dvMetas;
    
    public DeletionFileWriter(IndexPathFactory pathFactory, FileIO fileIO) throws IOException {
        this.path = pathFactory.newPath();
        this.out = new DataOutputStream(fileIO.newOutputStream(path, true));
        out.writeByte(VERSION_ID_V1);     // 写入文件版本号 (1 字节)
        this.dvMetas = new LinkedHashMap<>();
    }
    
    public void write(String key, DeletionVector deletionVector) throws IOException {
        int start = out.size();           // 记录当前写入位置
        int length = deletionVector.serializeTo(out);   // 序列化 DV
        dvMetas.put(key, new DeletionVectorMeta(key, start, length, 
                        deletionVector.getCardinality()));
    }
    
    public IndexFileMeta result() {
        return new IndexFileMeta(
            DELETION_VECTORS_INDEX,
            path.getName(),
            getPos(),               // 文件总大小
            dvMetas.size(),         // DV 数量
            dvMetas,                // 数据文件到 DV 位置的映射
            isExternalPath ? path.toString() : null);
    }
}
```

**关键设计决策**:

1. **LinkedHashMap 保序**: `dvMetas` 使用 `LinkedHashMap` 确保按写入顺序记录元数据。这很重要，因为 `readAllDeletionVectors` 方法 (DeletionVectorsIndexFile L82-L86) 按照 dvRanges 中的顺序**顺序**读取，保序可以保证顺序 I/O。

2. **offset 记录**: `out.size()` 返回的是从文件开头到当前位置的偏移量。注意第一个 DV 的 offset 不是 0，而是 1（因为文件头有 1 字节版本号）。

3. **isExternalPath 标志**: 如果路径是外部路径（不在标准索引目录下），则在 `IndexFileMeta` 中额外记录完整路径。

### 2.3 DeletionVectorMeta 元数据

源码路径: `paimon-core/src/main/java/org/apache/paimon/index/DeletionVectorMeta.java` (L33-L103)

每个数据文件对应一条 DV 元数据:

```java
public class DeletionVectorMeta {
    public static final RowType SCHEMA = RowType.of(
        new DataField(0, "f0", newStringType(false)),        // 数据文件名
        new DataField(1, "f1", new IntType(false)),          // 偏移量 (offset)
        new DataField(2, "f2", new IntType(false)),          // 长度 (length)
        new DataField(3, "_CARDINALITY", new BigIntType(true)));  // 删除行数 (可选)
    
    private final String dataFileName;     // 关联的数据文件名
    private final int offset;              // DV 数据在索引文件中的偏移量
    private final int length;              // DV 数据的字节长度
    @Nullable private final Long cardinality;  // 已删除的行数
}
```

**为什么需要 cardinality 字段**: `cardinality` 记录了该 DV 中已删除的行数。这个信息在查询计划阶段非常有用: 查询引擎可以通过 `totalRowCount - cardinality` 估算有效行数，用于代价估算和优化决策。`cardinality` 被标记为 `@Nullable` 和 `BigIntType(true)`，**这是后向兼容设计，旧版本数据可能没有此字段，读取时需要容错处理**。

### 2.4 DeletionVectorIndexFileWriter 与 Rolling 写入

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/DeletionVectorIndexFileWriter.java` (L34-L95)

这是 DV 索引文件写入的高层封装，支持两种写入模式:

**1. 单文件写入 (`writeSingleFile`)**:

```java
public IndexFileMeta writeSingleFile(Map<String, DeletionVector> input) throws IOException {
    DeletionFileWriter writer = new DeletionFileWriter(indexPathFactory, fileIO);
    try {
        for (Map.Entry<String, DeletionVector> entry : input.entrySet()) {
            writer.write(entry.getKey(), entry.getValue());
        }
    } finally {
        writer.close();
    }
    return writer.result();
}
```

适用场景: Bucketed 表（每个 bucket 一个 DV 文件），由 `BucketedDvMaintainer.writeDeletionVectorsIndex()` 调用。

**2. Rolling 写入 (`writeWithRolling`)**:

```java
public List<IndexFileMeta> writeWithRolling(Map<String, DeletionVector> input) throws IOException {
    List<IndexFileMeta> result = new ArrayList<>();
    Iterator<Map.Entry<String, DeletionVector>> iterator = input.entrySet().iterator();
    while (iterator.hasNext()) {
        result.add(tryWriter(iterator));
    }
    return result;
}

private IndexFileMeta tryWriter(Iterator<...> iterator) throws IOException {
    DeletionFileWriter writer = new DeletionFileWriter(indexPathFactory, fileIO);
    while (iterator.hasNext()) {
        Map.Entry<String, DeletionVector> entry = iterator.next();
        writer.write(entry.getKey(), entry.getValue());
        if (writer.getPos() > targetSizeInBytes) {
            break;    // 当前文件超过目标大小，开始新文件
        }
    }
    writer.close();
    return writer.result();
}
```

**为什么需要 Rolling 写入**: Unaware Bucket (无桶) 的 Append 表没有 bucket 分区概念，一个分区内可能有大量数据文件。如果所有 DV 都写入同一个索引文件，该文件可能非常大。Rolling 写入按 `targetSizePerIndexFile`（可配置）拆分为多个索引文件，平衡单文件大小和文件数量。

**注意**: Rolling 写入策略是"写完一个 DV 后检查大小"，因此实际文件大小可能略超 `targetSizeInBytes`（最多超出一个 DV 的大小）。

### 2.5 DV 索引文件物理布局

```
DV Index File (一个 .index 文件):
+------------------------------------------+
| Version (1 byte, 固定值 1)                |
+------------------------------------------+
| DV Data 1 (DataFile-A 的 DeletionVector) |
|   [bitmap_length][magic][bitmap_data][crc]|
+------------------------------------------+
| DV Data 2 (DataFile-B 的 DeletionVector) |
|   [bitmap_length][magic][bitmap_data][crc]|
+------------------------------------------+
| DV Data 3 (DataFile-C 的 DeletionVector) |
+------------------------------------------+
| ...                                      |
+------------------------------------------+
```

每个 DV 的位置通过 `IndexFileMeta.dvRanges()` 中的 `DeletionVectorMeta(dataFileName, offset, length)` 来定位。`offset` 指向该 DV 在文件中的字节偏移量，`length` 是该 DV 序列化后的总字节长度（包含 bitmap_length + magic + bitmap_data + crc）。

---

## 3. BucketedDvMaintainer 维护机制

### 3.1 核心职责与数据结构

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/BucketedDvMaintainer.java` (L35-L179)

`BucketedDvMaintainer` 管理单个 `(partition, bucket)` 内所有数据文件的 DV 状态:

```java
public class BucketedDvMaintainer {
    private final DeletionVectorsIndexFile dvIndexFile;
    private final Map<String, DeletionVector> deletionVectors;  // 文件名 -> DV
    protected final boolean bitmap64;                            // 使用 V1 还是 V2
    private boolean modified;                                    // 是否有变更
}
```

**为什么要在内存中维护完整的 DV 映射**: Bucketed 表的一个 bucket 通常只有一个 DV 索引文件。写入时可能对多个数据文件产生删除标记（例如对不同文件中的旧记录标记删除），因此需要维护一个完整的 `Map<String, DeletionVector>` 来追踪所有文件的 DV 状态。在 `prepareCommit` 时，将整个 map 写入新的 DV 索引文件。

### 3.2 notifyNewDeletion 增量删除通知

```java
// 通知单行删除 (L61-L67)
public void notifyNewDeletion(String fileName, long position) {
    DeletionVector deletionVector =
            deletionVectors.computeIfAbsent(fileName, k -> createNewDeletionVector());
    if (deletionVector.checkedDelete(position)) {
        modified = true;     // 只有真正新增的删除才标记 modified
    }
}

// 通知整个 DV 替换 (L75-L78)
public void notifyNewDeletion(String fileName, DeletionVector deletionVector) {
    deletionVectors.put(fileName, deletionVector);
    modified = true;
}
```

**为什么有两个重载方法**: 
- 单行版本 (`fileName, position`): 用于 Lookup 写入场景，每次找到一个旧记录就通知一次。使用 `computeIfAbsent` 确保 DV 延迟创建。
- 整体替换版本 (`fileName, DeletionVector`): 用于批量场景（如 Compaction 合并 DV）。

**`createNewDeletionVector()` (L50-L52)**: 根据 `bitmap64` 配置创建 V1 或 V2 的 DV:
```java
private DeletionVector createNewDeletionVector() {
    return bitmap64 ? new Bitmap64DeletionVector() : new BitmapDeletionVector();
}
```

### 3.3 mergeNewDeletion 合并删除向量

```java
public void mergeNewDeletion(String fileName, DeletionVector deletionVector) {
    DeletionVector old = deletionVectors.get(fileName);
    if (old != null) {
        deletionVector.merge(old);    // 新 DV 吸收旧 DV (OR 语义)
    }
    deletionVectors.put(fileName, deletionVector);
    modified = true;
}
```

**为什么是 `deletionVector.merge(old)` 而非 `old.merge(deletionVector)`**: 因为调用方传入的 `deletionVector` 是新产生的 DV，`old` 是已有的 DV。merge 操作是 OR 语义（取并集），方向无关紧要，但这里选择让新 DV 吸收旧 DV，然后替换。这种模式在 `BucketedAppendDeleteFileMaintainer` (L55-L57) 中被使用。

### 3.4 removeDeletionVectorOf 移除操作

```java
public void removeDeletionVectorOf(String fileName) {
    if (deletionVectors.containsKey(fileName)) {
        deletionVectors.remove(fileName);
        modified = true;
    }
}
```

**使用场景**: Compaction 时，旧文件被合并为新文件。旧文件的 DV 不再需要（因为旧文件本身将被删除），需要从 DV 映射中移除。注意只有当确实存在该文件的 DV 时才标记 `modified`。

### 3.5 Factory 模式与状态恢复

```java
public static class Factory {
    private final IndexFileHandler handler;
    
    // 从已恢复的 IndexFileMeta 列表创建 (L164-L172)
    public BucketedDvMaintainer create(
            BinaryRow partition, int bucket, @Nullable List<IndexFileMeta> restoredFiles) {
        if (restoredFiles == null) {
            restoredFiles = Collections.emptyList();
        }
        Map<String, DeletionVector> deletionVectors =
                new HashMap<>(handler.readAllDeletionVectors(partition, bucket, restoredFiles));
        return create(partition, bucket, deletionVectors);
    }
    
    // 从已有的 DV 映射创建 (L174-L177)
    public BucketedDvMaintainer create(
            BinaryRow partition, int bucket, Map<String, DeletionVector> deletionVectors) {
        return new BucketedDvMaintainer(handler.dvIndex(partition, bucket), deletionVectors);
    }
}
```

**恢复流程解析**:
1. Checkpoint 恢复时，Flink/Spark 的 Writer 持有已提交的 `IndexFileMeta` 列表
2. Factory 通过 `handler.readAllDeletionVectors()` 从这些索引文件中读取所有 DV 数据
3. 将 DV 数据装入 `HashMap`，构建新的 `BucketedDvMaintainer`
4. `handler.dvIndex(partition, bucket)` 创建对应 (partition, bucket) 的 `DeletionVectorsIndexFile`，用于后续写入

**为什么需要从 restoredFiles 恢复而非从最新 Snapshot 读取**: Checkpoint 恢复时，最新 Snapshot 可能不包含当前 Writer 的数据（可能有其他并行 Writer 的提交）。只有从 Writer 自身保存的 `restoredFiles` 恢复才能保证一致性。

### 3.6 写入输出与 modified 跟踪

```java
public Optional<IndexFileMeta> writeDeletionVectorsIndex() {
    if (modified) {
        modified = false;
        return Optional.of(dvIndexFile.writeSingleFile(deletionVectors));
    }
    return Optional.empty();
}
```

**为什么返回 `Optional`**: 如果没有任何修改 (`modified = false`)，不需要写入新的 DV 索引文件，返回空。这个设计避免了在没有删除操作时产生不必要的小文件。

**modified 重置**: 写入后将 `modified` 重置为 `false`，使下一轮提交只在有新变更时才写入。

---

## 4. DV 在读路径中的应用

### 4.1 ApplyDeletionVectorReader 行级过滤

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/ApplyDeletionVectorReader.java` (L31-L67)

这是一个装饰器模式的 `FileRecordReader`，包装底层 Reader 并应用 DV 过滤:

```java
public class ApplyDeletionVectorReader implements FileRecordReader<InternalRow> {
    private final FileRecordReader<InternalRow> reader;
    private final DeletionVector deletionVector;
    
    @Nullable
    public FileRecordIterator<InternalRow> readBatch() throws IOException {
        FileRecordIterator<InternalRow> batch = reader.readBatch();
        if (batch == null) return null;
        return new ApplyDeletionFileRecordIterator(batch, deletionVector);
    }
}
```

**设计模式**: 经典的**装饰器模式**。`ApplyDeletionVectorReader` 不改变 Reader 的读取逻辑，只在每个 batch 上包装一层过滤迭代器。这使得 DV 过滤可以透明地与任何 `FileRecordReader` 组合。

### 4.2 ApplyDeletionFileRecordIterator 迭代过滤

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/ApplyDeletionFileRecordIterator.java` (L30-L80)

```java
public class ApplyDeletionFileRecordIterator 
        implements FileRecordIterator<InternalRow>, DeletionFileRecordIterator {
    
    private final FileRecordIterator<InternalRow> iterator;
    private final DeletionVector deletionVector;
    
    public InternalRow next() throws IOException {
        while (true) {
            InternalRow next = iterator.next();
            if (next == null) return null;
            if (!deletionVector.isDeleted(returnedPosition())) {
                return next;   // 未被删除的行才返回
            }
            // 被删除的行直接跳过，继续读取下一行
        }
    }
    
    public long returnedPosition() {
        return iterator.returnedPosition();    // 委托给底层迭代器获取行位置
    }
}
```

**关键设计**: 
1. `returnedPosition()` 返回的是**文件内的行号**（从 0 开始），与 DV 中记录的 position 对应
2. 使用 `while(true)` 循环跳过连续的已删除行，直到找到有效行或到达文件末尾
3. 实现了 `DeletionFileRecordIterator` 接口，使得上层可以获取到 DV 引用

**为什么在迭代器层面而非 Reader 层面过滤**: Reader 以 batch 为粒度工作（一个 batch 可能包含多行），而 DV 过滤需要逐行检查。在迭代器层面可以精确地按行过滤，且不需要修改 batch 的内部结构。

### 4.3 读路径全链路分析

```
TableRead (入口)
  |
  v
DataSplit.rawConvertible = true (DV 启用后的优化标志)
  |
  v  
RawFileSplitRead.createReader(split)
  |
  +-- 获取 split 中的 DeletionFile 列表
  |
  +-- 对每个数据文件:
  |     FileRecordReader reader = 底层文件格式 Reader (ORC/Parquet)
  |     Optional<DeletionVector> dv = dvFactory.create(fileName)
  |     if (dv.isPresent()) {
  |         reader = new ApplyDeletionVectorReader(reader, dv.get())
  |     }
  |     -> 返回带 DV 过滤的 Reader
  |
  v
ApplyDeletionVectorReader.readBatch()
  -> ApplyDeletionFileRecordIterator.next()
     -> 对每行检查 dv.isDeleted(position)
     -> 跳过已删除行
```

**DV 启用后的关键优化**: 当 `rawConvertible = true` 时，读取路径跳过了 LSM-tree 的 merge 过程（无需将多个 level 的数据合并），直接逐文件读取并应用 DV 过滤。这使得读取性能接近 Append-Only 表。

---

## 5. DV 在写路径中的应用

### 5.1 Lookup 模式下标记旧行删除

在主键表 (PK Table) 的写入路径中，DV 通过 Lookup 机制生成:

```
TableWriteImpl.write(row)
  |
  v
KeyValueFileStoreWrite.write(partition, bucket, kv)
  |
  v (如果启用了 DV, 即 needLookup = true)
MergeTreeWriter.write(kv)
  |
  +-- LookupLevels.lookup(key)
  |     |
  |     +-- 在 Level-0 (内存 buffer) 中查找
  |     +-- 在 Level-1..N (磁盘文件) 中查找
  |     +-- 如果找到旧记录:
  |           返回 LookupResult(fileName, position, ...)
  |
  +-- 如果 LookupResult 有效:
  |     dvMaintainer.notifyNewDeletion(
  |         lookupResult.fileName(), lookupResult.position())
  |     // 标记旧文件中的旧行为已删除
  |
  +-- 写入新记录到 Level-0 (内存 buffer)
```

**为什么 Lookup 是 DV 的核心前提**: DV 的本质是"标记旧行为已删除"。要知道旧行在哪个文件的哪个位置，必须先查找旧记录。Lookup 机制（基于 RocksDB 或内存）维护了 `key -> (fileName, position)` 的映射，使得写入时可以精确定位旧记录。

### 5.2 Compaction 时 DV 合并

Compaction 过程中的 DV 处理:

```
Compaction 触发
  |
  v
选择参与 compaction 的文件列表 (beforeFiles)
  |
  v
合并读取 beforeFiles 的数据 + 应用 DV 过滤
  |
  +-- 对于每个 beforeFile:
  |     removeDeletionVectorOf(beforeFileName)  // 移除旧文件的 DV
  |
  v
写入合并后的新文件 (afterFiles)
  |
  +-- 新文件不需要 DV (数据已是最新的)
  |
  v
提交: 
  - 旧的 DV 索引文件标记为删除 (IndexManifestEntry DELETE)
  - 新的 DV 索引文件（如果有修改）标记为新增 (IndexManifestEntry ADD)
```

**关键点**: Compaction 的一个重要作用是"消除" DV。参与 compaction 的旧文件中被 DV 标记为删除的行不会被写入新文件，因此新文件不需要 DV。这实质上是将 merge-on-read 的开销转移到了后台的 compaction 中。

### 5.3 DV 生命周期总结

```
┌─────────────────────────────────────────────────────┐
│ 1. 初始化: Factory.create() 从 IndexManifest 恢复    │
│    -> 读取已有的 DV 索引文件到内存 Map               │
├─────────────────────────────────────────────────────┤
│ 2. 写入期间: Lookup 发现旧记录                        │
│    -> notifyNewDeletion(oldFile, oldPosition)        │
│    -> 在内存 Map 中更新 DV                           │
├─────────────────────────────────────────────────────┤
│ 3. Compaction 期间: 旧文件被合并                      │
│    -> removeDeletionVectorOf(oldFileName)            │
│    -> 从内存 Map 中移除对应 DV                       │
├─────────────────────────────────────────────────────┤
│ 4. prepareCommit(): 生成新的 DV 索引文件             │
│    -> writeDeletionVectorsIndex()                    │
│    -> 生成 IndexFileMeta 加入 IndexIncrement         │
├─────────────────────────────────────────────────────┤
│ 5. commit(): 提交到 Snapshot                         │
│    -> 旧 DV 索引文件 -> DELETE entry                 │
│    -> 新 DV 索引文件 -> ADD entry                    │
└─────────────────────────────────────────────────────┘
```

---

## 6. Append 表的 DV

### 6.1 BaseAppendDeleteFileMaintainer 接口

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/append/BaseAppendDeleteFileMaintainer.java` (L52-L103)

```java
public interface BaseAppendDeleteFileMaintainer {
    BinaryRow getPartition();
    int getBucket();
    void notifyNewDeletionVector(String dataFile, DeletionVector deletionVector);
    List<IndexManifestEntry> persist();
}
```

**为什么 Append 表也需要 DV**: Append 表虽然不支持 UPDATE，但支持 DELETE 操作（如 `DELETE FROM append_table WHERE id = 1`）。DV 允许在不重写整个数据文件的情况下标记被删除的行。

接口提供两个工厂方法:

1. `forBucketedAppend(...)` (L62-L75): 为有桶的 Append 表创建维护器
2. `forUnawareAppend(...)` (L77-L102): 为无桶的 Append 表创建维护器

### 6.2 AppendDeleteFileMaintainer (Unaware Bucket)

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/append/AppendDeleteFileMaintainer.java` (L41-L169)

这是无桶 Append 表的 DV 维护器，复杂度最高:

**核心数据结构**:
```java
private final Map<String, DeletionFile> dataFileToDeletionFile;  // 数据文件 -> DV文件位置
private final Map<String, IndexManifestEntry> indexNameToEntry;  // 索引文件 -> Manifest 条目
private final Map<String, Map<String, DeletionFile>> indexFileToDeletionFiles; // 索引文件 -> (数据文件 -> DV位置)
private final Map<String, String> dataFileToIndexFile;           // 数据文件 -> 索引文件名
private final Set<String> touchedIndexFiles;                     // 被修改的索引文件
private final Map<String, DeletionVector> deletionVectors;       // 新增的 DV
```

**关键方法 `notifyNewDeletionVector`** (L119-L125):

```java
public void notifyNewDeletionVector(String dataFile, DeletionVector deletionVector) {
    DeletionFile previous = notifyRemovedDeletionVector(dataFile);
    if (previous != null) {
        deletionVector.merge(dvIndexFile.readDeletionVector(previous));
    }
    deletionVectors.put(dataFile, deletionVector);
}
```

**为什么需要 merge 已有的 DV**: 同一个数据文件可能在多次操作中被删除不同的行。新的 DV 必须包含之前所有的删除信息，否则之前被删除的行会"复活"。

**`persist()` 方法** (L128-L134) 的两步写入:

1. **writeUnchangedDeletionVector** (L146-L168): 处理被"触碰"的索引文件
   - 如果一个索引文件中只有部分 DV 被修改，需要将未修改的 DV 重写到新文件
   - 旧的索引文件标记为 DELETE
   - 这是因为无桶表的索引文件通过 Rolling 写入，一个索引文件包含多个数据文件的 DV

2. **writeWithRolling(deletionVectors)**: 将新增的 DV 写入新的索引文件

**为什么这比 BucketedDvMaintainer 复杂**: Bucketed 表每个 bucket 只有一个 DV 索引文件，修改时整体重写即可。无桶表的 DV 分散在多个 Rolling 索引文件中，部分更新需要处理"读取旧索引文件中未修改的 DV -> 写入新索引文件 -> 删除旧索引文件"的复杂逻辑。

### 6.3 BucketedAppendDeleteFileMaintainer (Bucketed)

源码路径: `paimon-core/src/main/java/org/apache/paimon/deletionvectors/append/BucketedAppendDeleteFileMaintainer.java` (L31-L68)

这是有桶 Append 表的 DV 维护器，内部直接委托给 `BucketedDvMaintainer`:

```java
public class BucketedAppendDeleteFileMaintainer implements BaseAppendDeleteFileMaintainer {
    private final BinaryRow partition;
    private final int bucket;
    private final BucketedDvMaintainer maintainer;   // 核心委托对象
    
    public void notifyNewDeletionVector(String dataFile, DeletionVector deletionVector) {
        maintainer.mergeNewDeletion(dataFile, deletionVector);   // 使用 merge 语义
    }
    
    public List<IndexManifestEntry> persist() {
        List<IndexManifestEntry> result = new ArrayList<>();
        maintainer.writeDeletionVectorsIndex()
                .map(fileMeta -> new IndexManifestEntry(FileKind.ADD, partition, bucket, fileMeta))
                .ifPresent(result::add);
        return result;
    }
}
```

**为什么使用 `mergeNewDeletion` 而非 `notifyNewDeletion`**: Append 表的 DELETE 操作可能一次性传入一个完整的 DV（表示这次操作要删除的所有行），需要与已有的 DV 合并，而不是逐行通知。

### 6.4 Append 表与 PK 表 DV 差异对比

| 维度 | PK 表 DV | Append 表 DV |
|------|----------|-------------|
| DV 产生方式 | Lookup 发现旧记录时自动生成 | 用户执行 DELETE 语句时生成 |
| 维护器 | `BucketedDvMaintainer` | `AppendDeleteFileMaintainer` 或 `BucketedAppendDeleteFileMaintainer` |
| Compaction 消除 | 是，Compaction 后旧文件 DV 被移除 | 否，除非被 Compact 的文件包含 DV |
| 写入模式 | 单文件 (`writeSingleFile`) | Rolling (`writeWithRolling`) 或单文件 |
| 一个索引文件含 DV 数 | 整个 bucket 的所有 DV | 取决于 Rolling 策略 |

---

## 7. 文件索引 SPI 体系

### 7.1 FileIndexer/FileIndexerFactory 接口设计

**FileIndexerFactory** (SPI 入口):

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/FileIndexerFactory.java` (L24-L30)

```java
public interface FileIndexerFactory {
    String identifier();                           // SPI 标识符，如 "bloom-filter", "bitmap"
    FileIndexer create(DataType type, Options options);  // 创建索引器
}
```

**FileIndexer** (索引器):

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/FileIndexer.java` (L29-L41)

```java
public interface FileIndexer {
    FileIndexWriter createWriter();                                      // 创建写入器
    FileIndexReader createReader(SeekableInputStream inputStream, int start, int length);  // 创建读取器
    
    static FileIndexer create(String type, DataType dataType, Options options) {
        FileIndexerFactory factory = FileIndexerFactoryUtils.load(type);
        return factory.create(dataType, options);
    }
}
```

**为什么采用 SPI 架构**: 文件索引是可扩展的。通过 Java SPI (`ServiceLoader`)，用户可以实现自己的索引类型并通过 classpath 注册，无需修改 Paimon 核心代码。目前内置了 4 种索引类型:

| SPI identifier | 工厂类 | 索引实现类 |
|---------------|--------|-----------|
| `bloom-filter` | `BloomFilterFileIndexFactory` | `BloomFilterFileIndex` |
| `bitmap` | `BitmapFileIndexFactory` | `BitmapFileIndex` |
| `bsi` | `BitSliceIndexBitmapFileIndexFactory` | `BitSliceIndexBitmapFileIndex` |
| `range-bitmap` | `RangeBitmapFileIndexFactory` | `RangeBitmapFileIndex` |

### 7.2 SPI 加载机制

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/FileIndexerFactoryUtils.java` (L29-L56)

```java
public class FileIndexerFactoryUtils {
    private static final Map<String, FileIndexerFactory> factories = new HashMap<>();
    
    static {
        ServiceLoader<FileIndexerFactory> serviceLoader =
                ServiceLoader.load(FileIndexerFactory.class);
        for (FileIndexerFactory indexerFactory : serviceLoader) {
            if (factories.put(indexerFactory.identifier(), indexerFactory) != null) {
                LOG.warn("Found multiple FileIndexer for type: " + indexerFactory.identifier());
            }
        }
    }
    
    static FileIndexerFactory load(String type) {
        FileIndexerFactory factory = factories.get(type);
        if (factory == null) {
            throw new RuntimeException("Can't find file index for type: " + type);
        }
        return factory;
    }
}
```

**实现细节**: 使用静态初始化块加载所有 SPI 实现，存入 `HashMap`。重复注册同一 `identifier` 会打印警告但不报错（后注册的覆盖先注册的）。

### 7.3 FileIndexFormat 存储格式 (Header + Body)

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/FileIndexFormat.java` (L99-L367)

文件索引以附加文件 (`.index`) 的形式存储，格式如下:

```
 ______________________________________    _____________________
|     magic (8 bytes long)              |
|   = 1493475289347502                  |
|---------------------------------------|
|   version (4 bytes int) = 1          |
|---------------------------------------|
|   head length (4 bytes int)          |
|---------------------------------------|
|   column number (4 bytes int)        |
|---------------------------------------|
|   column 1 name ｜ index count       |         HEAD
|---------------------------------------|
|   index type 1 ｜ start pos ｜length |
|---------------------------------------|
|   index type 2 ｜ start pos ｜length |
|---------------------------------------|
|   column 2 name ｜ index count       |
|---------------------------------------|
|   index type 1 ｜ start pos ｜length |
|---------------------------------------|
|   redundant length (4 bytes) = 0     |
|---------------------------------------|    ---------------------
|            BODY (索引数据)             |         BODY
|______________________________________|    _____________________
```

**Header 结构解析**:
- `magic` (8 bytes): 固定值 `1493475289347502L`，用于识别文件格式
- `version` (4 bytes): 当前版本 `1`
- `head length` (4 bytes): header 的总字节数（包含 magic、version、head length 本身）
- `column number` (4 bytes): 有索引的列数
- 每列: `column_name (UTF) + index_count (4 bytes)`
- 每个索引: `index_type (UTF) + start_pos (4 bytes, 含 head offset) + length (4 bytes)`
- `redundant length` (4 bytes): 预留扩展字段，当前为 0

**为什么 start_pos 需要加上 headLength**: Body 数据紧跟在 Header 之后。`start_pos` 在 Body 构建时是相对于 Body 起始的偏移，写入 Header 时需要加上 `headLength` 转换为文件绝对偏移。这使得 Reader 可以直接 `seek(start_pos)` 读取索引数据。

**EMPTY_INDEX_FLAG (-1)**: 如果某个索引的数据为 null（写入器返回 null 字节数组），其 `start_pos` 被设为 `-1`，Reader 遇到此标记返回 `EmptyFileIndexReader.INSTANCE`。

### 7.4 FileIndexWriter 写入接口

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/FileIndexWriter.java` (L22-L39)

```java
public abstract class FileIndexWriter {
    private boolean empty = true;
    
    public void writeRecord(Object key) {
        empty = false;
        write(key);
    }
    
    public abstract void write(Object key);        // 子类实现：写入一个值
    public abstract byte[] serializedBytes();       // 子类实现：序列化为字节数组
    
    public boolean empty() { return empty; }
}
```

**为什么需要 `empty` 标记**: 如果一列的所有值都是 null，索引可能不需要序列化。`writeRecord` 方法在调用 `write` 前设置 `empty = false`，上层可以通过 `empty()` 判断是否需要序列化。

### 7.5 FileIndexReader 读取接口

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/FileIndexReader.java` (L34-L148)

`FileIndexReader` 是抽象类，实现了 `FunctionVisitor<FileIndexResult>` 接口。所有 visit 方法的默认返回值都是 `REMAIN`（即"不过滤"），子类只需覆盖支持的操作:

```java
public abstract class FileIndexReader implements FunctionVisitor<FileIndexResult> {
    // 默认实现: 所有操作都返回 REMAIN (保留)
    public FileIndexResult visitEqual(FieldRef fieldRef, Object literal) { return REMAIN; }
    public FileIndexResult visitLessThan(FieldRef fieldRef, Object literal) { return REMAIN; }
    public FileIndexResult visitIn(FieldRef fieldRef, List<Object> literals) {
        // 默认: 对每个 literal 调用 visitEqual，结果做 OR
        FileIndexResult result = null;
        for (Object key : literals) {
            result = result == null ? visitEqual(fieldRef, key) : result.or(visitEqual(fieldRef, key));
        }
        return result;
    }
    // ... 其他操作同理
}
```

**为什么默认返回 REMAIN 而非 SKIP**: 安全性考虑。如果某种索引不支持某个操作（如 Bloom Filter 不支持范围查询），返回 REMAIN 意味着"不确定，保留该文件"。这是一种**保守策略**，避免错误过滤导致数据丢失。

---

## 8. Bloom Filter 索引

### 8.1 BloomFilterFileIndex 实现

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/bloomfilter/BloomFilterFileIndex.java` (L48-L136)

```java
public class BloomFilterFileIndex implements FileIndexer {
    private static final int DEFAULT_ITEMS = 1_000_000;    // 默认预期元素数
    private static final double DEFAULT_FPP = 0.1;         // 默认误判率 10%
    
    private final DataType dataType;
    private final int items;
    private final double fpp;
}
```

**Writer** (L83-L112):
```java
private static class Writer extends FileIndexWriter {
    private final BloomFilter64 filter;
    private final FastHash hashFunction;
    
    public Writer(DataType type, int items, double fpp) {
        this.filter = new BloomFilter64(items, fpp);        // 根据预期数量和误判率初始化
        this.hashFunction = FastHash.getHashFunction(type);  // 按类型获取哈希函数
    }
    
    public void write(Object key) {
        if (key != null) {
            filter.addHash(hashFunction.hash(key));    // null 值不加入 BloomFilter
        }
    }
    
    public byte[] serializedBytes() {
        // [4字节 numHashFunctions][BitSet 字节数组]
        int numHashFunctions = filter.getNumHashFunctions();
        byte[] serialized = new byte[filter.getBitSet().bitSize() / 8 + 4];
        // Big-Endian 写入 numHashFunctions
        serialized[0] = (byte) ((numHashFunctions >>> 24) & 0xFF);
        // ... 
        filter.getBitSet().toByteArray(serialized, 4, serialized.length - 4);
        return serialized;
    }
}
```

**Reader** (L114-L135):
```java
private static class Reader extends FileIndexReader {
    private final BloomFilter64 filter;
    private final FastHash hashFunction;
    
    public FileIndexResult visitEqual(FieldRef fieldRef, Object key) {
        return key == null || filter.testHash(hashFunction.hash(key)) ? REMAIN : SKIP;
    }
    // 注意: 只覆盖了 visitEqual，其他操作使用父类默认的 REMAIN
}
```

### 8.2 BloomFilter64 底层结构

BloomFilter64 是一个 64 位哈希的 Bloom Filter 实现:
- 使用 `BitSet` 存储位数组
- 根据 `items` 和 `fpp` 自动计算最优的位数组大小和哈希函数个数
- 使用 `numHashFunctions` 个哈希函数，每个哈希值通过对 bitSize 取模来确定位位置

### 8.3 FastHash 哈希策略

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/bloomfilter/FastHash.java` (L52-L217)

```java
public interface FastHash {
    long hash(Object o);
    
    static FastHash getHashFunction(DataType type) {
        return type.accept(FastHashVisitor.INSTANCE);
    }
}
```

`FastHashVisitor` 为不同数据类型提供不同的哈希策略:

| 数据类型 | 哈希方式 |
|---------|---------|
| `CHAR/VARCHAR` | `LongHashFunction.xx().hashBytes(bytes)` (xxHash) |
| `BINARY/VARBINARY` | `LongHashFunction.xx().hashBytes(bytes)` (xxHash) |
| `TINYINT/SMALLINT/INT/BIGINT` | Thomas Wang 的整数哈希函数 |
| `FLOAT` | `getLongHash(Float.floatToIntBits(value))` |
| `DOUBLE` | `getLongHash(Double.doubleToLongBits(value))` |
| `DATE/TIME` | 作为 int 用 Thomas Wang 哈希 |
| `TIMESTAMP` | precision <= 3 用毫秒，否则用微秒 |
| `BOOLEAN/DECIMAL/ARRAY/MAP/ROW` | **不支持**，抛出 `UnsupportedOperationException` |

**为什么字符串用 xxHash 而数值用 Thomas Wang 哈希**: xxHash 是一种高性能的通用哈希函数，适合字节数组。Thomas Wang 的整数哈希 (L202-L211) 是一种位混洗（bit-mixing）算法，对整数值有更好的分布性且无需序列化为字节数组，性能更高。

### 8.4 序列化格式

```
+----------------------------+----------------------------+
| numHashFunctions (4 bytes) | BitSet 数据 (变长)          |
| Big-Endian int             |                            |
+----------------------------+----------------------------+
```

### 8.5 仅支持 Equal 的设计决策

Bloom Filter Reader **仅覆盖了 `visitEqual`**，其他操作（`visitLessThan`, `visitGreaterThan` 等）使用父类默认的 `REMAIN`。

**为什么 Bloom Filter 不支持范围查询**: Bloom Filter 的数学原理决定了它只能回答"某个元素可能存在"或"一定不存在"的问题，无法回答"是否存在大于 X 的元素"。范围查询需要值的有序性信息，而 Bloom Filter 通过哈希打散了值的顺序。

**但 `visitIn` 可以工作**: 因为 `IN` 等价于多个 `EQUAL` 的 OR。父类 `FileIndexReader` 的 `visitIn` 默认实现就是对每个值调用 `visitEqual` 后做 OR 运算 (L97-L106)。

### 8.6 参数调优 (items/fpp)

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `items` | 1,000,000 | 预期的不同值数量 |
| `fpp` | 0.1 (10%) | 误判率 (False Positive Probability) |

配置方式:
```sql
CREATE TABLE t (...) WITH (
  'file-index.bloom-filter.columns' = 'col1',
  'file-index.bloom-filter.col1.items' = '500000',
  'file-index.bloom-filter.col1.fpp' = '0.01'
);
```

**调优建议**:
- `items` 应接近每个文件中该列的不同值数量。过大浪费空间，过小增加误判率
- `fpp` 越小精度越高，但 BitSet 占用空间越大。`0.01` (1%) 是常用的平衡点
- Bloom Filter 的空间公式: `bits = -items * ln(fpp) / (ln(2))^2`

---

## 9. Bitmap 倒排索引

### 9.1 BitmapFileIndex 核心思想

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/bitmap/BitmapFileIndex.java` (L49-L389)

为每个不同的字段值维护一个 RoaringBitmap32，记录包含该值的行号集合:

```
字段 "city" (值 -> 包含该值的行号):
  "Beijing"  -> RoaringBitmap32 {0, 3, 7, 12}
  "Shanghai" -> RoaringBitmap32 {1, 4, 8}
  "Shenzhen" -> RoaringBitmap32 {2, 5, 9, 10, 11}
  null       -> RoaringBitmap32 {6}
```

**Writer 核心逻辑** (L80-L191):
```java
private static class Writer extends FileIndexWriter {
    private final int version;
    private final Map<Object, RoaringBitmap32> id2bitmap = new HashMap<>();
    private final RoaringBitmap32 nullBitmap = new RoaringBitmap32();
    private int rowNumber;
    
    public void write(Object key) {
        if (key == null) {
            nullBitmap.add(rowNumber++);
        } else {
            id2bitmap.computeIfAbsent(valueMapper.apply(key), k -> new RoaringBitmap32())
                    .add(rowNumber++);
        }
    }
}
```

**为什么需要 `valueMapper`**: 某些数据类型（如 `Timestamp`）的内部表示不适合直接作为 HashMap 的 key。`getValueMapper()` (L316-L388) 将 Timestamp 转换为 Long 值（毫秒或微秒），BinaryString 做深拷贝（避免对象复用导致的 key 冲突）。

### 9.2 V1 格式 (BitmapFileIndexMeta)

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/bitmap/BitmapFileIndexMeta.java` (L74-L360)

```
Bitmap File Index V1:
+-------------------------------------------------+
| version (1 byte) = 1                           |
+-------------------------------------------------+
| row count (4 bytes int)                         |
+-------------------------------------------------+
| non-null value bitmap number (4 bytes int)      |
+-------------------------------------------------+
| has null value (1 byte)                         |
+-------------------------------------------------+
| null value offset (4 bytes, 仅 hasNull 时存在)   |    HEAD
+-------------------------------------------------+
| value 1 | offset 1                              |
+-------------------------------------------------+
| value 2 | offset 2                              |
+-------------------------------------------------+
| ...                                             |
+-------------------------------------------------+-----------
| serialized bitmap 1                             |
+-------------------------------------------------+
| serialized bitmap 2                             |    BODY
+-------------------------------------------------+
| ...                                             |
+-------------------------------------------------+
```

**offset 的特殊编码**: 当一个值只出现在一行时 (cardinality == 1)，offset 被编码为 `-1 - rowNumber`。这样读取时不需要读取序列化的 Bitmap，直接从 offset 中解码出行号。

### 9.3 V2 格式 (BitmapFileIndexMetaV2) 二级索引优化

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/bitmap/BitmapFileIndexMetaV2.java` (L100-L409)

**设计动机**: 当列的基数（cardinality）很高时，V1 格式需要在读取时遍历整个字典来查找目标值。V2 在字典上建立**二级索引块**（Index Blocks），支持二分查找，显著减少读取开销。

```
Bitmap File Index V2:
+-------------------------------------------------+
| version (1 byte) = 2                           |
+-------------------------------------------------+
| row count | non-null number | has null         |
+-------------------------------------------------+
| null offset | null bitmap length               |    HEAD
+-------------------------------------------------+
| bitmap index block number (4 bytes)             |
+-------------------------------------------------+
| block key 1 | block offset 1                   |  (二级索引入口)
| block key 2 | block offset 2                   |
| ...                                             |
+-------------------------------------------------+
| bitmap body offset (4 bytes)                    |
+-------------------------------------------------+-----------
| Index Block 1:                                  |
|   entry number | value | offset | length | ...  |  INDEX BLOCKS
| Index Block 2:                                  |
|   entry number | value | offset | length | ...  |
| ...                                             |
+-------------------------------------------------+-----------
| serialized bitmap 1                             |
| serialized bitmap 2                             |  BITMAP BLOCKS
| ...                                             |
+-------------------------------------------------+
```

**查找流程**:
1. 在 HEAD 中的二级索引入口做二分查找，定位到 `BitmapIndexBlock`
2. 在 `BitmapIndexBlock` 内部做二分查找，定位到具体的 bitmap entry
3. 按 offset + length 读取 bitmap 数据

```java
public Entry findEntry(Object bitmapId) {
    BitmapIndexBlock block = findBlock(bitmapId);     // 二分查找定位 Block
    if (block != null) {
        return block.findEntry(bitmapId);             // Block 内部二分查找
    }
    return null;
}
```

**Block 大小限制**: 通过 `index-block-size` 参数控制（默认 16KB），确保每个 Block 不会太大。

**为什么 V2 比 V1 更快**: V1 查找一个值需要遍历所有字典条目 O(n)，V2 通过二级索引将复杂度降低到 O(log(n/block_size))。对于百万级基数的列，差异非常显著。

### 9.4 支持的谓词类型

| 谓词 | 支持 | 返回类型 |
|------|------|---------|
| `Equal` | 是 | `BitmapIndexResult` |
| `NotEqual` | 是 | `BitmapIndexResult` (通过 flip 实现) |
| `In` | 是 | `BitmapIndexResult` (多个 bitmap 的 OR) |
| `NotIn` | 是 | `BitmapIndexResult` (In 结果 flip) |
| `IsNull` | 是 | `BitmapIndexResult` (null bitmap) |
| `IsNotNull` | 是 | `BitmapIndexResult` (null bitmap flip) |
| `LessThan/GreaterThan` 等 | 否 | 默认 `REMAIN` |

### 9.5 单值优化

当某个值只出现在一行时:
```java
if (v.getCardinality() == 1) {
    bitmapOffsets.put(k, -1 - v.iterator().next());  // 编码: offset = -1 - rowNumber
}
```

读取时:
```java
int offset = entry.offset;
if (offset < 0) {
    return RoaringBitmap32.bitmapOf(-1 - offset);    // 解码: rowNumber = -1 - offset
}
```

**为什么需要这个优化**: 对于高基数列（如 ID 列），大量值只出现一次。为每个值存储完整的 Bitmap 序列化数据非常浪费空间。单值优化将 Bitmap 压缩为一个 int 值（行号），节省了大量存储空间。

### 9.6 ApplyBitmapIndexRecordReader 行级下推

源码路径:
- `paimon-common/.../fileindex/bitmap/ApplyBitmapIndexRecordReader.java` (L31-L58)
- `paimon-common/.../fileindex/bitmap/ApplyBitmapIndexFileRecordIterator.java` (L33-L78)

Bitmap 索引不仅可以做文件级过滤（SKIP/REMAIN），还可以做**行级过滤**:

```java
public class ApplyBitmapIndexFileRecordIterator implements FileRecordIterator<InternalRow> {
    private final FileRecordIterator<InternalRow> iterator;
    private final RoaringBitmap32 bitmap;   // 匹配行号集合
    private final int last;                 // bitmap 中的最大行号
    
    public InternalRow next() throws IOException {
        while (true) {
            InternalRow next = iterator.next();
            if (next == null) return null;
            int position = (int) returnedPosition();
            if (position > last) return null;          // 超过最大匹配行号，提前终止
            if (bitmap.contains(position)) return next; // 在匹配集合中
        }
    }
}
```

**关键优化 `position > last`**: 当读取位置超过 Bitmap 中的最大行号时，后续不可能有匹配行，直接返回 null 终止读取。这避免了读取文件末尾的大量无用数据。

**为什么 Bitmap 索引可以做行级下推而 Bloom Filter 不行**: Bloom Filter 只能判断"值可能存在于文件中"，无法告诉你具体在哪些行。Bitmap 索引直接记录了匹配值对应的行号集合，因此可以精确到行级过滤。

---

## 10. BSI (Bit-Sliced Index)

### 10.1 BitSliceIndexBitmapFileIndex 核心原理

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/bsi/BitSliceIndexBitmapFileIndex.java` (L55-L412)

BSI 将整数值的每个二进制位拆开，为每个位维护一个 RoaringBitmap32（称为 Slice）:

```
假设有 4 行数据，值分别为 5(101), 3(011), 7(111), 2(010):

bit-0 (最低位): RoaringBitmap32 {0, 1, 2}    (值 5=1, 3=1, 7=1, 2=0)
bit-1:          RoaringBitmap32 {1, 2, 3}    (值 5=0, 3=1, 7=1, 2=1)
bit-2 (最高位): RoaringBitmap32 {0, 2}       (值 5=1, 3=0, 7=1, 2=0)
```

**为什么 BSI 比 Bitmap 倒排更适合范围查询**: Bitmap 倒排为每个值维护一个 Bitmap。范围查询 `age >= 5` 需要找到所有值 >= 5 的 Bitmap 并做 OR，开销与基数成正比。BSI 只需要对位切片做固定次数的位运算（次数 = 值的位数），时间复杂度 O(bits)。

### 10.2 正负数分组策略

BSI 将值分为正数组和负数组分别处理:

```java
// Writer 中的分组逻辑 (L115-L139)
for (int i = 0; i < collector.values.size(); i++) {
    Long value = collector.values.get(i);
    if (value != null) {
        if (value < 0) {
            negative.append(i, Math.abs(value));   // 负数取绝对值存储
        } else {
            positive.append(i, value);
        }
    }
}
```

**为什么要分组**: BSI 的底层 `BitSliceIndexRoaringBitmap` 只能处理非负整数。对负数取绝对值后存入独立的 BSI，查询时分别在正数和负数 BSI 上操作:

```java
// 查询 `x < value` 的逻辑 (L279-L289)
public FileIndexResult visitLessThan(FieldRef fieldRef, Object literal) {
    return new BitmapIndexResult(() -> {
        Long value = valueMapper.apply(literal);
        if (value < 0) {
            return negative.gt(Math.abs(value));   // x < -3 等价于 |x| > 3 (负数域)
        } else {
            return RoaringBitmap32.or(positive.lt(value), negative.isNotNull());
            // x < 5 等价于 (正数 < 5) OR (所有负数)
        }
    });
}
```

**负数查询语义转换**:
- `x < -3` -> 在负数域中 `|x| > 3`
- `x >= -3` -> 在负数域中 `|x| <= 3` OR 所有正数
- `x = -3` -> 在负数域中 `|x| = 3`

### 10.3 范围查询 O(bits) 复杂度

以 `gt(code)` (大于查询) 为例:

```java
// BitSliceIndexRoaringBitmap.gt 核心算法:
public RoaringBitmap32 gt(long code) {
    RoaringBitmap32 state = null;
    int start = Long.numberOfTrailingZeros(~code); // 优化: 跳过末尾连续1位
    
    for (int i = start; i < slices.length; i++) {
        if (state == null) {
            state = getSlice(i).clone();
            continue;
        }
        long bit = (code >> i) & 1;
        if (bit == 1) {
            state.and(getSlice(i));    // 该位为1: 保留该位也为1的行
        } else {
            state.or(getSlice(i));     // 该位为0: 加入该位为1的行
        }
    }
    state.and(foundSet);  // 与有效行集合取交集
    return state;
}
```

**时间复杂度**: O(bits)，对于 long 类型最多 64 次 Bitmap 运算，与数据量和基数无关。

### 10.4 值映射 (Value Mapper)

```java
public static Function<Object, Long> getValueMapper(DataType dataType) {
    return dataType.accept(new DataTypeDefaultVisitor<>() {
        // TinyInt/SmallInt/Int -> long
        // BigInt -> 直接使用
        // Date/Time -> int -> long
        // Timestamp -> 毫秒或微秒 (取决于精度)
        // Decimal -> unscaledLong
    });
}
```

### 10.5 支持的数据类型与谓词

| 数据类型 | 映射方式 |
|---------|---------|
| `TINYINT/SMALLINT/INT/BIGINT` | 直接转 long |
| `DATE/TIME` | 天数/毫秒数 转 long |
| `TIMESTAMP` | 精度 <= 3 用毫秒, 否则微秒 |
| `DECIMAL` | `unscaledLong` |
| 字符串/浮点/复杂类型 | **不支持** |

| 谓词 | 支持 | 说明 |
|------|------|------|
| `Equal/NotEqual` | 是 | 精确匹配 |
| `In/NotIn` | 是 | 多值匹配 |
| `LessThan/LessOrEqual` | 是 | 范围查询 |
| `GreaterThan/GreaterOrEqual` | 是 | 范围查询 |
| `Between` | 是 | `gte AND lte` |
| `IsNull/IsNotNull` | 是 | 通过 EBM (Existence Bitmap) |

---

## 11. Range Bitmap 索引

### 11.1 RangeBitmapFileIndex 架构

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/rangebitmap/RangeBitmapFileIndex.java` (L43-L185)

Range Bitmap 是最复杂的索引类型，组合了三个核心组件:

```
RangeBitmap (核心)
  |
  +-- ChunkedDictionary (字典编码: 值 <-> 整数码)
  |     |
  |     +-- Chunk[] (分块存储)
  |           |
  |           +-- FixedLengthChunk / VariableLengthChunk
  |
  +-- BitSliceIndexBitmap (BSI: 整数码的位切片索引)
        |
        +-- RoaringBitmap32[] slices (位切片)
        +-- RoaringBitmap32 ebm (Existence Bitmap)
```

**整体查询流程**:
1. 将查询值通过 Dictionary 转换为整数码 (code)
2. 在 BSI 上执行位切片运算，得到匹配的行号集合
3. 返回 `BitmapIndexResult`

**为什么要在 BSI 上加一层 Dictionary**: BSI 要求输入为非负整数。Range Bitmap 通过字典编码将任意类型（包括字符串、浮点数）映射为连续的非负整数，从而使 BSI 可以处理所有类型。字典按值排序，因此码值的大小关系与原始值一致，保证了范围查询的正确性。

### 11.2 ChunkedDictionary 分块字典

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/rangebitmap/dictionary/chunked/ChunkedDictionary.java` (L33-L277)

字典采用分块存储策略:

```
ChunkedDictionary:
  +-- Chunk[0]: key="apple",  code=0,  内含 keys: apple, banana, cherry
  +-- Chunk[1]: key="date",   code=3,  内含 keys: date, elderberry, fig
  +-- Chunk[2]: key="grape",  code=6,  内含 keys: grape, honey, ice
```

**二分查找流程 (`find` 方法, L75-L95)**:
1. 先在 Chunk 数组上做二分查找，定位到目标 Chunk
2. 在 Chunk 内部查找具体的码值
3. 如果值不存在，返回 `-(insertion_point) - 1` (类似 `Arrays.binarySearch`)

```java
public int find(Object key) {
    int low = 0, high = size - 1;
    while (low <= high) {
        int mid = (low + high) >>> 1;
        Chunk found = get(mid);
        int result = comparator.compare(found.key(), key);
        if (result > 0) high = mid - 1;
        else if (result < 0) low = mid + 1;
        else return found.code();           // 精确匹配 Chunk 的 key
    }
    if (low == 0) return -(low + 1);        // 比最小值还小
    return get(low - 1).find(key);           // 在前一个 Chunk 内查找
}
```

**为什么使用分块而非扁平字典**: 分块字典支持**增量读取**。查找时只需要反序列化命中的 Chunk，不需要读取整个字典。对于大字典（百万级基数），这显著减少了 I/O 和内存开销。分块大小通过 `chunk-size` 参数控制。

### 11.3 BitSliceIndexBitmap 内部 BSI

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/rangebitmap/BitSliceIndexBitmap.java` (L35-L431)

这是 Range Bitmap 专用的 BSI 实现（与 `paimon-common/.../bsi` 中的 BSI 不同），支持:
- `eq(code)`: 等值查询
- `gt(code)` / `gte(code)`: 大于/大于等于
- `topK(k, foundSet, strict)`: 找 Top-K 最大值
- `bottomK(k, foundSet, strict)`: 找 Top-K 最小值

**延迟加载优化**: BSI 的 slices 数组在构造时不立即加载，而是在首次查询时才从文件中读取 (L287-L322)。`ebm` (Existence Bitmap) 也是延迟加载的 (L272-L285)。

### 11.4 TopN 查询加速

Range Bitmap 是唯一支持 TopN 查询的索引类型。`visitTopN` (L167-L183) 的实现:

```java
public FileIndexResult visitTopN(TopN topN, FileIndexResult result) {
    RoaringBitmap32 foundSet = result instanceof BitmapIndexResult 
            ? ((BitmapIndexResult) result).get() : null;
    int limit = topN.limit();
    SortValue sort = orders.get(0);
    boolean strict = orders.size() == 1;
    
    if (ASCENDING.equals(sort.direction())) {
        return new BitmapIndexResult(
                () -> bitmap.bottomK(limit, nullOrdering, foundSet, strict));
    } else {
        return new BitmapIndexResult(
                () -> bitmap.topK(limit, nullOrdering, foundSet, strict));
    }
}
```

**TopK 算法 (BitSliceIndexBitmap.topK, L165-L210)**:

参考论文: *Bit-Sliced Index Arithmetic* (O'Neil & Quass, SIGMOD 1997) Algorithm 4.1

```
算法: 找 K 个最大值
G = {} (guaranteed set, 一定在 top-K 中)
E = isNotNull(foundSet) (eligible set, 候选集)

从最高位到最低位遍历:
  X = G OR (E AND slice[i])
  if |X| > K:
    E = E AND slice[i]      // 保留该位为 1 的行
  else if |X| < K:
    G = X                    // 确认 X 中的行
    E = E AND NOT slice[i]   // 继续看更低位
  else:
    E = E AND slice[i]
    break
结果 = G OR E
```

**时间复杂度**: O(slices * bitmap_op)，其中 bitmap_op 是 Bitmap 位运算的复杂度。这比排序整个数据集 O(n*log(n)) 要高效得多。

**NullOrdering 处理**: `fillNulls` 方法 (L303-L329) 处理 `NULLS FIRST` 和 `NULLS LAST` 语义:
- `NULLS_LAST`: 先从非 null 值中找 top-K，不够再用 null 行补充
- `NULLS_FIRST`: 先取所有 null 行，不够再从非 null 值中补充

### 11.5 与 BSI 索引的差异

| 维度 | BSI (bsi) | Range Bitmap (range-bitmap) |
|------|-----------|---------------------------|
| 数据类型 | 仅数值型 | 任意类型（通过字典编码） |
| 负数处理 | 正/负分组，取绝对值 | 字典编码为非负整数 |
| 存储结构 | BitSliceIndexRoaringBitmap | ChunkedDictionary + BitSliceIndexBitmap |
| TopN 支持 | 不支持 | 支持 |
| 内存开销 | Writer 需要缓存所有值 (StatsCollectList) | Writer 通过 TreeMap 排序 |
| 适用场景 | 纯数值列的范围查询 | 任意类型的范围查询 + TopN |

---

## 12. FileIndexResult 三态逻辑

### 12.1 REMAIN/SKIP/BitmapIndexResult 三种状态

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/FileIndexResult.java` (L22-L77)

```java
public interface FileIndexResult {
    // 状态1: REMAIN - "不确定，可能包含匹配数据"
    FileIndexResult REMAIN = new FileIndexResult() {
        boolean remain() { return true; }
        FileIndexResult and(other) { return other; }      // REMAIN AND x = x
        FileIndexResult or(other) { return this; }        // REMAIN OR x = REMAIN
    };
    
    // 状态2: SKIP - "一定不包含匹配数据"
    FileIndexResult SKIP = new FileIndexResult() {
        boolean remain() { return false; }
        FileIndexResult and(other) { return this; }       // SKIP AND x = SKIP
        FileIndexResult or(other) { return other; }       // SKIP OR x = x
    };
    
    // 状态3: BitmapIndexResult - "精确匹配这些行"
    // 见 BitmapIndexResult 类
}
```

**为什么是"三态"而非布尔**: Bloom Filter 无法告诉你具体哪些行匹配，只能说"可能匹配" (REMAIN) 或"一定不匹配" (SKIP)。Bitmap 索引则可以返回精确的行号集合 (BitmapIndexResult)。三态设计允许不同精度的索引在同一个框架中协同工作。

### 12.2 AND/OR 运算语义

**AND 运算 (交集语义)**:

| AND | REMAIN | SKIP | Bitmap{1,3,5} |
|-----|--------|------|---------------|
| REMAIN | REMAIN | SKIP | Bitmap{1,3,5} |
| SKIP | SKIP | SKIP | SKIP |
| Bitmap{2,3,4} | Bitmap{2,3,4} | SKIP | Bitmap{3} |

**OR 运算 (并集语义)**:

| OR | REMAIN | SKIP | Bitmap{1,3,5} |
|----|--------|------|---------------|
| REMAIN | REMAIN | REMAIN | REMAIN |
| SKIP | REMAIN | SKIP | Bitmap{1,3,5} |
| Bitmap{2,3,4} | REMAIN | Bitmap{2,3,4} | Bitmap{1,2,3,4,5} |

**REMAIN 在 AND 中的特殊行为**: `REMAIN AND x = x` 而非 `REMAIN AND x = REMAIN`。这是因为 REMAIN 表示"不确定"，与一个确定结果做 AND 时，应该保留确定结果。例如: Bloom Filter 返回 REMAIN (可能包含 "Beijing")，Bitmap 返回 Bitmap{1,3,5} (行 1,3,5 是 "Beijing")，AND 后应该使用 Bitmap{1,3,5} 做行级过滤。

### 12.3 BitmapIndexResult 的延迟计算

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/bitmap/BitmapIndexResult.java` (L29-L77)

```java
public class BitmapIndexResult extends LazyField<RoaringBitmap32> implements FileIndexResult {
    public BitmapIndexResult(Supplier<RoaringBitmap32> supplier) {
        super(supplier);    // 延迟计算: 只有在 get() 被调用时才执行 supplier
    }
    
    public boolean remain() {
        return !get().isEmpty();    // 触发计算
    }
    
    public FileIndexResult and(FileIndexResult other) {
        if (other instanceof BitmapIndexResult) {
            return new BitmapIndexResult(
                    () -> RoaringBitmap32.and(get(), ((BitmapIndexResult) other).get()));
        }
        return FileIndexResult.super.and(other);
    }
}
```

**为什么使用延迟计算**: Bitmap 运算（特别是大 Bitmap 的 AND/OR）可能很耗时。如果最终结果因为某个 SKIP 而不需要 Bitmap 结果，延迟计算可以避免不必要的运算。例如: `BitmapResult AND SKIP = SKIP`，此时 BitmapResult 的 Supplier 永远不会被执行。

**额外方法**:
- `andNot(RoaringBitmap32 deletion)`: 从 Bitmap 中移除 DV 标记的行
- `limit(int limit)`: 限制结果行数（用于 TopN）

---

## 13. FileIndexPredicate 评估流程

### 13.1 谓词评估入口

源码路径: `paimon-common/src/main/java/org/apache/paimon/fileindex/FileIndexPredicate.java` (L54-L206)

```java
public class FileIndexPredicate implements Closeable {
    private final FileIndexFormat.Reader reader;
    
    public FileIndexResult evaluate(@Nullable Predicate predicate) {
        if (predicate == null) return REMAIN;
        
        // 1. 提取谓词涉及的列名
        Set<String> requiredFieldNames = getRequiredNames(predicate);
        
        // 2. 加载这些列的索引 Reader
        Map<String, Collection<FileIndexReader>> indexReaders = new HashMap<>();
        requiredFieldNames.forEach(name -> indexReaders.put(name, reader.readColumnIndex(name)));
        
        // 3. 用 FileIndexPredicateTest 访问者评估谓词
        FileIndexResult result = new FileIndexPredicateTest(indexReaders).test(predicate);
        
        if (!result.remain()) {
            LOG.debug("One file has been filtered: " + path);
        }
        return result;
    }
}
```

### 13.2 FileIndexPredicateTest 访问者模式

```java
private static class FileIndexPredicateTest implements PredicateVisitor<FileIndexResult> {
    private final Map<String, Collection<FileIndexReader>> columnIndexReaders;
    
    // 叶子谓词评估
    public FileIndexResult visit(LeafPredicate predicate) {
        FieldRef fieldRef = predicate.fieldRefOptional().orElse(null);
        if (fieldRef == null) return REMAIN;
        
        FileIndexResult compoundResult = REMAIN;
        for (FileIndexReader reader : columnIndexReaders.get(fieldRef.name())) {
            compoundResult = compoundResult.and(
                    predicate.function().visit(reader, fieldRef, predicate.literals()));
            if (!compoundResult.remain()) return compoundResult;  // 短路优化
        }
        return compoundResult;
    }
    
    // 组合谓词评估
    public FileIndexResult visit(CompoundPredicate predicate) {
        if (predicate.function() instanceof Or) {
            // OR: 对所有子谓词做 or 运算
            FileIndexResult result = null;
            for (Predicate child : predicate.children()) {
                result = result == null ? child.visit(this) : result.or(child.visit(this));
            }
            return result == null ? REMAIN : result;
        } else {
            // AND: 对所有子谓词做 and 运算，支持短路
            FileIndexResult result = null;
            for (Predicate child : predicate.children()) {
                result = result == null ? child.visit(this) : result.and(child.visit(this));
                if (!result.remain()) return result;  // 短路: SKIP 后不再评估
            }
            return result == null ? REMAIN : result;
        }
    }
}
```

**关键设计**: 一列可能有**多个索引**（如同时有 Bloom Filter 和 Bitmap），对同一列的多个索引结果做 AND 运算。这意味着 Bloom Filter 的文件级判断可以与 Bitmap 的行级判断组合: 如果 Bloom Filter 判断 SKIP，则直接跳过；否则使用 Bitmap 的行级结果。

### 13.3 TopN 评估

```java
public FileIndexResult evaluateTopN(@Nullable TopN topN, FileIndexResult result) {
    if (topN == null || !result.remain()) return result;
    
    // 如果已有 BitmapIndexResult 且行数 <= K，无需 TopN 过滤
    if (result instanceof BitmapIndexResult) {
        long cardinality = ((BitmapIndexResult) result).get().getCardinality();
        if (cardinality <= k) return result;
    }
    
    // 在第一个排序列的索引上执行 TopN
    String requiredName = orders.get(0).field().name();
    Set<FileIndexReader> readers = reader.readColumnIndex(requiredName);
    for (FileIndexReader reader : readers) {
        FileIndexResult ret = reader.visitTopN(topN, result);
        if (!REMAIN.equals(ret)) return ret;
    }
    return result;
}
```

**为什么只看第一个排序列**: 当前实现仅支持单列排序的 TopN 优化。`orders.size() == 1` 时 `strict = true`（精确 K 行），多列时 `strict = false`（可能返回多于 K 行）。

---

## 14. 全局索引

### 14.1 GlobalIndexMeta 架构

源码路径: `paimon-core/src/main/java/org/apache/paimon/index/GlobalIndexMeta.java` (L33-L60)

```java
public class GlobalIndexMeta {
    public static final RowType SCHEMA = new RowType(true, Arrays.asList(
        new DataField(0, "_ROW_RANGE_START", new BigIntType(false)),    // 行范围起始
        new DataField(1, "_ROW_RANGE_END", new BigIntType(false)),      // 行范围结束
        new DataField(2, "_INDEX_FIELD_ID", new IntType(false)),        // 索引字段 ID
        new DataField(3, "_EXTRA_FIELD_IDS", DataTypes.ARRAY(IntType)), // 附加字段 ID
        new DataField(4, "_INDEX_META", DataTypes.BYTES())));           // 索引元数据
    
    private final long rowRangeStart;
    private final long rowRangeEnd;
    private final int indexFieldId;
    @Nullable private final int[] extraFieldIds;
    @Nullable private final byte[] indexMeta;
}
```

### 14.2 BTree 全局索引

源码路径: `paimon-core/src/main/java/org/apache/paimon/globalindex/btree/BTreeGlobalIndexBuilder.java`

全局索引使用 B-Tree 结构存储在独立的索引文件中:
- 通过 `GlobalIndexFileReadWrite` 进行文件读写
- 索引文件以 `global-index-{uuid}.index` 命名
- 每个索引条目记录 key 到 (partition, bucket) 的映射

### 14.3 用途: 跨分区更新

源码路径: `paimon-core/src/main/java/org/apache/paimon/crosspartition/GlobalIndexAssigner.java`

当表启用了跨分区更新时，全局索引维护主键到 (partition, bucket) 的映射:

1. **启动时**: 通过 `IndexBootstrap` 扫描全表，在 RocksDB 中建立完整索引
2. **写入时**: 查找旧记录所在的 partition + bucket
3. **跨分区更新**: 如果旧记录在不同分区，先在旧分区产生 DELETE（通过 DV），再在新分区产生 INSERT

**与本地 Lookup 的区别**: 本地 Lookup 只在同一 (partition, bucket) 内查找旧记录，全局索引可以跨分区查找。

---

## 15. Predicate 体系

### 15.1 LeafPredicate 叶子谓词

源码路径: `paimon-common/src/main/java/org/apache/paimon/predicate/LeafPredicate.java` (L46-L283)

```java
public class LeafPredicate implements Predicate {
    private final Transform transform;       // 字段引用或表达式
    private final LeafFunction function;     // 比较函数 (Equal, LessThan, In 等)
    private transient List<Object> literals; // 比较字面量
    
    // 多层求值:
    // 1. 行级求值 (L136-L139)
    public boolean test(InternalRow row) {
        Object value = transform.transform(row);
        return function.test(transform.outputType(), value, literals);
    }
    
    // 2. 统计级求值 (L142-L164) - 用于文件级过滤
    public boolean test(long rowCount, InternalRow minValues, InternalRow maxValues, 
                        InternalArray nullCounts) {
        // 基于 min/max/nullCount 统计信息判断
    }
}
```

**LeafFunction 实现**: Equal, NotEqual, LessThan, GreaterThan, LessOrEqual, GreaterOrEqual, IsNull, IsNotNull, In, NotIn, StartsWith, EndsWith, Contains, Like, Between, AlwaysFalse 等。

### 15.2 CompoundPredicate 组合谓词

源码路径: `paimon-common/src/main/java/org/apache/paimon/predicate/CompoundPredicate.java` (L35-L103)

```java
public class CompoundPredicate implements Predicate {
    private final CompoundFunction function;   // And 或 Or
    private final List<Predicate> children;    // 子谓词列表
}
```

### 15.3 FunctionVisitor 双重分派

源码路径: `paimon-common/src/main/java/org/apache/paimon/predicate/FunctionVisitor.java` (L27-L100)

```java
public interface FunctionVisitor<T> extends PredicateVisitor<T> {
    default T visit(LeafPredicate predicate) {
        // 双重分派: predicate.function().visit(this, fieldRef, literals)
        return predicate.function().visit(this, fieldRef.get(), predicate.literals());
    }
    
    // 子类只需实现具体的 visit 方法:
    T visitEqual(FieldRef fieldRef, Object literal);
    T visitLessThan(FieldRef fieldRef, Object literal);
    T visitIn(FieldRef fieldRef, List<Object> literals);
    // ... 等
}
```

**为什么使用双重分派**: Predicate 的求值取决于两个维度: (1) 函数类型 (Equal, LessThan 等)，(2) 求值上下文 (行级测试、统计测试、索引测试)。双重分派允许在不修改 LeafFunction 的情况下，通过不同的 Visitor 实现不同的求值逻辑。`FileIndexReader` 就是 `FunctionVisitor<FileIndexResult>` 的实现。

### 15.4 Predicate 多层下推

Predicate 在 Paimon 中被层层下推:

```
                     Predicate
                        |
        +---------------+---------------+
        |               |               |
   Manifest 层      DataFile 层      FileIndex 层       Row 层
   (分区统计)     (min/max 统计)   (Bloom/Bitmap/BSI)  (逐行过滤)
   过滤 Manifest  过滤数据文件     过滤文件/行           精确过滤
```

每一层的过滤是**递进的**: 上层过滤后的结果传给下层继续过滤。越靠前的层过滤成本越低但精度也越低。

---

## 16. Lookup 机制

### 16.1 StateFactory 接口

源码路径: `paimon-core/src/main/java/org/apache/paimon/lookup/StateFactory.java` (L27-L51)

```java
public interface StateFactory extends Closeable {
    <K, V> ValueState<K, V> valueState(String name, Serializer<K> keySerializer, 
            Serializer<V> valueSerializer, long lruCacheSize) throws IOException;
    
    <K, V> SetState<K, V> setState(String name, Serializer<K> keySerializer, 
            Serializer<V> valueSerializer, long lruCacheSize) throws IOException;
    
    <K, V> ListState<K, V> listState(String name, Serializer<K> keySerializer, 
            Serializer<V> valueSerializer, long lruCacheSize) throws IOException;
    
    boolean preferBulkLoad();    // 是否偏好批量加载
}
```

### 16.2 RocksDB 状态后端

源码路径: `paimon-core/src/main/java/org/apache/paimon/lookup/rocksdb/RocksDBStateFactory.java` (L45)

- 数据持久化到本地磁盘，适合大数据量
- 支持 TTL (通过 `TtlDB`)
- 支持 merge operator (字符串追加)
- `preferBulkLoad() = true`: RocksDB 的 SST 文件导入比逐条 put 更高效

**为什么默认使用 RocksDB**: Lookup 需要存储 `key -> (fileName, position)` 的映射。对于大表，这个映射可能有数十亿条目，放不进内存。RocksDB 作为嵌入式 KV 存储，通过 LSM-tree 结构可以高效地在磁盘上存储和查询海量数据。

### 16.3 InMemory 状态后端

源码路径: `paimon-core/src/main/java/org/apache/paimon/lookup/memory/InMemoryStateFactory.java` (L30)

- 全内存存储，适合小数据量
- `preferBulkLoad() = false`: 内存操作不需要批量优化
- 提供 `InMemoryValueState`, `InMemorySetState`, `InMemoryListState`

### 16.4 LRU Cache 策略

所有 State 接口都接受 `lruCacheSize` 参数:
- RocksDB 状态: LRU Cache 缓存最近访问的 KV 对，减少磁盘读取
- InMemory 状态: LRU Cache 限制内存使用上限

**为什么需要 LRU Cache**: Lookup 操作在写入每条记录时都会触发。热点 key (频繁更新的记录) 通过 LRU Cache 可以避免重复的 RocksDB 查询。Cache size 通过 `lookup.cache-rows` 等参数配置。

---

## 17. 与 Iceberg DV 对比

| 维度 | Paimon DV | Iceberg DV |
|------|-----------|------------|
| **引入版本** | Paimon 0.7+ | Iceberg V3 (format-version=3) |
| **位图实现** | RoaringBitmap32 (V1) / OptimizedRoaringBitmap64 (V2) | Roaring64Bitmap (Puffin blob) |
| **存储方式** | 独立的 DV 索引文件 (`DELETION_VECTORS` 类型) | Puffin 文件格式 (`.puffin`) |
| **关联方式** | 通过 IndexManifest 中的 IndexFileMeta | 通过 Manifest 中的 content_offset/content_length |
| **粒度** | 每个 bucket 一个 DV 文件 (PK 表) 或 Rolling 多个文件 (Append 表) | 每个 data file 一个 DV blob |
| **更新方式** | 增量合并 (merge + writeSingleFile 整体重写) | Copy-on-Write (重写整个 Puffin 文件) |
| **产生方式** | 写入时自动通过 Lookup 生成 | 需要显式执行 DELETE 或 rewrite |
| **用途** | 消除 merge-on-read 开销，使 PK 表读取接近 Append 表 | 替代 position delete 文件 |
| **与 Compaction 关系** | Compaction 消除 DV (旧文件 DV 被移除) | 没有直接关系，需要单独的 rewrite 操作 |
| **格式兼容性** | V2 格式参考 Iceberg DV 格式 (Little-Endian, 64-bit Roaring) | Iceberg 标准格式 |
| **Checkpoint 恢复** | 从 Writer 保存的 restoredFiles 恢复 | 从 Iceberg Snapshot 恢复 |
| **实时性** | 实时写入时产生 DV | 通常由 Spark/Trino 等批处理引擎产生 |

**Paimon DV 的独特优势**:

1. **与 LSM-tree 深度集成**: Paimon 的 DV 通过 Lookup 机制在写入时**自动**生成，无需额外操作。用户写入一条更新记录时，系统自动查找旧记录并标记删除。

2. **Compaction 自动清理**: LSM-tree 的 Compaction 过程天然会消除 DV，不需要像 Iceberg 那样手动执行 `rewrite` 操作。

3. **读取优化**: DV 启用后，PK 表的读取跳过 LSM merge，性能接近 Append 表。这是因为每个数据文件中有效的行（未被 DV 标记删除的行）就是最新版本，无需与其他文件合并。

4. **Append 表也支持**: Paimon 允许在 Append 表上使用 DV 实现行级删除（`DELETE FROM` 语句），而 Iceberg 的 DV 仅用于替代 position delete。
