# Apache Paimon Deletion Vector 机制源码深度分析

> **版本**：1.5-SNAPSHOT　**源码模块**：`paimon-core`（`org.apache.paimon.deletionvectors`，配置项在 `paimon-api` 的 `CoreOptions`）　**核对日期**：2026-06

**一句话定位**：Deletion Vector（DV）用一个 **per-data-file 的位图** 把"哪些行已失效"记下来，让主键表读取时**跳过 LSM merge、按行过滤**直接读最新文件——这就是 Paimon 的 **MOW（merge-on-write）**模式，把 merge-on-read 的代价从"每次查询"转移到"后台一次 compaction"。

读完本文你应能回答：① DV 到底是文件级的还是行级的、行号是相对谁的偏移；② MOW 模式下 DV 是在 WriteBuffer flush 时生成、还是在 **lookup compaction** 时生成（这是旧稿讲糊的关键）；③ DV 与 LSM 各 level 的关系——为什么"Level 0 在 compaction 前对读不可见"；④ 一个 bucket 的 DV 为什么整体重写而不是追加，`modified` 标记省了什么；⑤ 读路径上 DV 如何按行过滤、与文件索引返回的 selection 如何叠加；⑥ Append 表的 DELETE 为什么比 PK 表的 DV 维护更复杂（Rolling + touched 索引文件重写）；⑦ "行级标删（MOW）"对比"整文件重写（COW）"各自的代价边界，什么时候 DV 反而变成累赘；⑧ V1/V2 两种位图格式的差异与 Iceberg 兼容点。

> 阅读约定：本文每个机制按"① 要解决什么问题 → ② 设计原理与取舍 → ③ 关键源码（精选片段 + `路径:行号`）→ ④ 风险/陷阱/边界 → ⑤ 收益与代价"组织。源码行号以本次核对为准；与旧稿不符处用"（已修正）"标注。文件索引（Bloom/Bitmap/BSI/Range Bitmap）的内部结构由 **13 号文档主讲**，本文只在读路径交汇处点到并交叉引用，不重复展开。

---

## 目录

- [1. 快速理解（核心问题 / 概念速查 / 高频陷阱）](#1-快速理解核心问题--概念速查--高频陷阱)
  - [1.1 核心问题：DV 把 merge 代价从"读"挪到"写"](#11-核心问题dv-把-merge-代价从读挪到写)
  - [1.2 核心概念速查表](#12-核心概念速查表)
  - [1.3 高频生产陷阱](#13-高频生产陷阱)
- [2. DV 的数据结构：BitmapDeletionVector(V1) 与 Bitmap64DeletionVector(V2)](#2-dv-的数据结构bitmapdeletionvectorv1-与-bitmap64deletionvectorv2)
  - [2.1 接口分层与 checkedDelete](#21-接口分层与-checkeddelete)
  - [2.2 V1/V2 序列化格式与 Magic Number 分派](#22-v1v2-序列化格式与-magic-number-分派)
- [3. DV 文件组织：索引文件 + dvRanges 元数据](#3-dv-文件组织索引文件--dvranges-元数据)
  - [3.1 一个 bucket 一个索引文件：DeletionVectorsIndexFile / DeletionFileWriter](#31-一个-bucket-一个索引文件deletionvectorsindexfile--deletionfilewriter)
  - [3.2 单文件写入 vs Rolling 写入](#32-单文件写入-vs-rolling-写入)
- [4. BucketedDvMaintainer：DV 的内存维护与增量提交](#4-bucketeddvmaintainerdv-的内存维护与增量提交)
- [5. MOW 写路径：DV 在 lookup compaction 时生成（核心）](#5-mow-写路径dv-在-lookup-compaction-时生成核心)
  - [5.1 DV 不是在 flush 时生成，而是在 compaction 时](#51-dv-不是在-flush-时生成而是在-compaction-时)
  - [5.2 DV 与 LSM 各 level 的关系](#52-dv-与-lsm-各-level-的关系)
  - [5.3 compaction 如何"消除"旧 DV](#53-compaction-如何消除旧-dv)
  - [5.4 DV 生命周期串讲](#54-dv-生命周期串讲)
- [6. DV 读路径：按行过滤与装饰器](#6-dv-读路径按行过滤与装饰器)
- [7. Append 表的 DV：行级 DELETE 与 Rolling 维护](#7-append-表的-dv行级-delete-与-rolling-维护)
- [8. MOW vs COW vs MOR：行级标删的取舍边界](#8-mow-vs-cow-vs-mor行级标删的取舍边界)
- [9. 文件索引交汇点（详见 13 号文档）](#9-文件索引交汇点详见-13-号文档)
- [10. 与 Iceberg DV 对比](#10-与-iceberg-dv-对比)
- [11. 设计决策总结](#11-设计决策总结)

---

## 1. 快速理解（核心问题 / 概念速查 / 高频陷阱）

### 1.1 核心问题：DV 把 merge 代价从"读"挪到"写"

**① 要解决什么问题**

主键表用 LSM merge-tree 存储，同一个 key 的多个版本散落在 Level 0 到 Level N 的不同文件里。没有任何额外机制时，读取必须把命中的多层文件做归并（merge-on-read，MOR），按 sequenceNumber 取最新版本、丢弃 DELETE——这条归并链路（`SortMergeReader` → `DropDeleteReader`，详见 [01 §5.3](01-核心存储引擎分析.md)）是主键表读取慢于 Append 表 3–10 倍的根因。

**② 设计原理与取舍**

DV 的思路：既然旧版本注定要被丢弃，何不在**写入侧**就把"某文件第 N 行已被新版本顶替"记下来？读取时直接逐文件顺序读、用位图按行号跳过失效行，**完全不做跨文件归并**。这就把 merge 的 CPU/内存代价从"每次查询都付"变成"后台 compaction 付一次"。

| 方案 | merge 时机 | 读取代价 | 写入代价 | 适用 |
|------|-----------|---------|---------|------|
| **MOR**（无 DV） | 每次读都归并多层 | 高（读放大随 run 数线性增长） | 低（纯追加） | 写多读少、changelog 流读 |
| **MOW**（DV） | compaction 时归并并生成 DV | 低（顺序读 + 位图跳行，接近 Append） | 中（compaction 时做 lookup） | 读多、点查/OLAP、batch 读 |
| **COW**（整文件重写） | 写入时重写整个数据文件 | 最低（文件即最新） | 极高（改一行重写整文件） | Paimon 不用于主键表 |

一句话设计哲学：**用一次性的 compaction 写放大，换取每次查询都省掉的 merge 读放大。** DV 不是"删除数据"，而是"标记某物理行在逻辑上已失效"。

**关键澄清（旧稿最大的坑）**：DV 的本质是**行级**标记，但它**不是在 `write()`/flush 时生成的**。Level 0 文件刚 flush 出来时还没有 DV、对读不可见；DV 是在 **lookup compaction** 把 Level 0 推向高层时，发现某个 key 在更高 level 已有旧版本，才调用 `notifyNewDeletion(旧文件, 旧行号)` 生成的（见 §5）。理解这一点是读懂 MOW 的钥匙。

### 1.2 核心概念速查表

| 概念 | 一句话定义 | 关键源码 |
|------|-----------|---------|
| **DeletionVector** | 记录某数据文件中已失效行号的位图接口；读侧只需 `isDeleted(long)` | `DeletionVector.java:44` |
| **BitmapDeletionVector** | V1 实现，基于 `RoaringBitmap32`，行号上限 2^31-1 | `BitmapDeletionVector.java:34` |
| **Bitmap64DeletionVector** | V2 实现，基于 `OptimizedRoaringBitmap64`，与 Iceberg 兼容 | `Bitmap64DeletionVector.java:38` |
| **DeletionVectorsIndexFile** | 一个物理 `.index` 文件，顺序存放多个数据文件的 DV | `DeletionVectorsIndexFile.java:44` |
| **DeletionVectorMeta / dvRanges** | `(dataFileName, offset, length, cardinality)`，定位 DV 在索引文件中的位置 | `DeletionVectorMeta.java:33` |
| **BucketedDvMaintainer** | 在内存里维护单个 (partition,bucket) 的 `文件名→DV` 映射，提交时整体重写 | `BucketedDvMaintainer.java:35` |
| **notifyNewDeletion** | 标记"某文件某行已删"，靠 `checkedDelete` 去重并置 `modified` | `BucketedDvMaintainer.java:61` |
| **LookupChangelogMergeFunctionWrapper** | compaction 时 lookup 旧版本并触发 `notifyNewDeletion` 的地方 | `LookupChangelogMergeFunctionWrapper.java:126` |
| **ApplyDeletionVectorReader** | 读路径装饰器，逐行用 DV 过滤 | `ApplyDeletionVectorReader.java:31` |
| **BaseAppendDeleteFileMaintainer** | Append 表 DELETE 的 DV 维护器（分桶/无桶两实现） | `BaseAppendDeleteFileMaintainer.java:52` |

### 1.3 高频生产陷阱

**陷阱 1：以为开 DV 后"写完立刻能查到/查不到旧值"。** MOW 下 Level 0 文件在 compaction 生成 DV 之前对读不可见（官方 table-mode 明确 "files with level 0 will only be visible after compaction"）。所以 MOW 表的 compaction 必须**同步**执行；若被改成纯异步 compaction，新写数据会出现可见延迟。排障口诀：MOW 表"刚写的数据查不到/旧数据没删干净"，先查 compaction 是不是被异步化了。

**陷阱 2：超大文件（行数 > 2^31-1）却用了 V1。** `BitmapDeletionVector.delete()` 会把 `long position` 强转 int，`checkPosition`（`BitmapDeletionVector.java:118`）在越界时抛 `IllegalArgumentException` 快速失败。单文件可能逼近 21 亿行时需开 `deletion-vectors.bitmap64=true` 用 V2；且 **只有 V2 才与 Iceberg DV 格式兼容**。

**陷阱 3：DV 不会自动消失，只能靠 compaction 收敛。** DV 累积越多，读时被标删的行越多（白读 I/O），写时 lookup 命中越频繁。若更新极频繁而 compaction 跟不上，会出现"读 1000 行实际有效 200 行"的浪费。监控 `dvRanges` 里 `cardinality` 之和占 `rowCount` 的比例，过高说明 compaction 滞后。

**陷阱 4：DV 索引文件目标大小默认 2MB，无桶 Append 表尤其敏感。** `deletion-vector.index-file.target-size` 默认 `2 MB`（`CoreOptions.java:1875`，已修正旧稿"默认值可能不适合"的含糊表述）。Rolling 写入是"写完一个 DV 再判断是否超限"，故单文件实际可能略超目标值（最多超出一个 DV 的大小）。

**陷阱 5：误以为 DV 在每次提交都重写。** `BucketedDvMaintainer` 用 `modified` 标记，只有真正产生新增/移除时才在 `writeDeletionVectorsIndex()` 写新文件（`BucketedDvMaintainer.java:115`）。`checkedDelete` 对已删行返回 `false`、不置 `modified`，所以重复标记同一行不会触发多余的小文件。

**陷阱 6：把 Append 表 DELETE 的代价当成和 PK 表一样。** 无桶 Append 表的 DV 分散在多个 Rolling 索引文件里，删一个数据文件的若干行，要把它所在索引文件中**其它未改动的 DV 也重写到新文件**、旧索引文件整体 DELETE（`AppendDeleteFileMaintainer.writeUnchangedDeletionVector`，`:146`）。频繁小批 DELETE 会放大索引文件写。

**陷阱 7：DV 与数据文件必须同一次提交。** DV 的 `IndexFileMeta` 打进 `IndexIncrement`，与数据文件的增量在同一个 Snapshot 原子提交。若分两次提交，中间崩溃会出现"新版本可见但旧版本未标删"的数据重复——这由提交层保证（详见 [01 §13 提交流程](01-核心存储引擎分析.md)）。

---

## 2. DV 的数据结构：BitmapDeletionVector(V1) 与 Bitmap64DeletionVector(V2)

### 2.1 接口分层与 checkedDelete

DV 接口刻意分两层：读路径只依赖 `DeletionVectorJudger.isDeleted(long)`，写路径才需要 `DeletionVector`（`delete`/`merge`/`serializeTo`/`checkedDelete`）。这是接口隔离——读侧组件拿到一个 DV 只为按行判定，无需感知写入细节。

```
DeletionVectorJudger          isDeleted(long)          ← 读路径唯一所需
  └─ DeletionVector           delete / merge / serializeTo / checkedDelete
       ├─ BitmapDeletionVector       (V1, RoaringBitmap32, 行号 ≤ 2^31-1)
       └─ Bitmap64DeletionVector     (V2, OptimizedRoaringBitmap64, 与 Iceberg 兼容)
```

`checkedDelete` 是去重的关键（`DeletionVector.java:66`）。接口默认实现是"先 `isDeleted` 再 `delete`"，但 V1 覆盖为底层 `RoaringBitmap32.checkedAdd` 一次原子完成（`BitmapDeletionVector.java:64-68`），少一次查找：

```java
// BitmapDeletionVector.java:64
@Override
public boolean checkedDelete(long position) {
    checkPosition(position);                       // 校验不超过 Integer.MAX_VALUE
    return roaringBitmap.checkedAdd((int) position); // 新增返回 true，已存在返回 false
}
```

返回值是 `modified` 标记的来源：`BucketedDvMaintainer.notifyNewDeletion` 仅当 `checkedDelete` 返回 `true` 才把 `modified` 置真（见 §4）。这样重复标记同一行不会触发多余的 DV 文件写入。

V1 与 V2 的差异只在底层位图和序列化字节序，行为接口完全一致：

| 维度 | V1 `BitmapDeletionVector` | V2 `Bitmap64DeletionVector` |
|------|---------------------------|------------------------------|
| 底层位图 | `RoaringBitmap32` | `OptimizedRoaringBitmap64`（注释 "Mostly copied from iceberg"，`:36`） |
| 行号上限 | `RoaringBitmap32.MAX_VALUE` = 2^31-1 | 理论 2^63 |
| position 处理 | 强转 int，`checkPosition` 越界即抛 | 直接存 long，无溢出风险 |
| Magic Number | `1581511376`（`:36`） | `1681511377`（`:40`） |
| 序列化字节序 | Big-Endian（Java 默认） | Little-Endian（与 Iceberg/Roaring 原生一致） |
| 额外处理 | 无 | `serializeTo` 前 `runLengthEncode()` 做游程编码（`:94`） |
| 升级 | — | `fromBitmapDeletionVector()` 支持 V1→V2（`:56`） |

**为什么需要两个版本**：V1 设计时未预见单文件逾 21 亿行的场景；V2 为兼容 Iceberg V3 DV 格式而引入（Little-Endian + 64 位 Roaring）。两者通过 Magic Number 自动识别，DV 数据块自包含、无需外部版本号，因此**新旧格式可在同一张表里混存**。

### 2.2 V1/V2 序列化格式与 Magic Number 分派

反序列化入口 `DeletionVector.read(DataInputStream, Long)`（`DeletionVector.java:97`）通过 Magic Number 自动分派 V1/V2。两种格式都是 `[bitmapLength][magic][bitmap_data][crc]`，关键差别在 magic 的字节序，以及 `bitmapLength` 的口径：

```
V1: [bitmapLength(BE)][magic=1581511376(BE)][roaring_data][crc]
        bitmapLength = magic + 数据 的长度（= length 字段值本身）
V2: [bitmapLength(BE)][magic=1681511377(LE)][bitmap64_data(LE)][crc]
        bitmapLength = magic + 数据；总长度 = bitmapLength + LENGTH_SIZE(4) + CRC_SIZE(4)
```

```java
// DeletionVector.java:97  —— 行号已核对
static DeletionVector read(DataInputStream dis, @Nullable Long length) throws IOException {
    int bitmapLength = dis.readInt();
    int magicNumber  = dis.readInt();
    if (magicNumber == BitmapDeletionVector.MAGIC_NUMBER) {          // V1：BE magic 直接命中
        byte[] bytes = new byte[bitmapLength - BitmapDeletionVector.MAGIC_NUMBER_SIZE_BYTES];
        dis.readFully(bytes); dis.skipBytes(4);                      // 跳过 CRC
        return BitmapDeletionVector.deserializeFromByteBuffer(ByteBuffer.wrap(bytes));
    } else if (toLittleEndianInt(magicNumber) == Bitmap64DeletionVector.MAGIC_NUMBER) { // V2：转 LE 再比
        byte[] bytes = new byte[bitmapLength - Bitmap64DeletionVector.MAGIC_NUMBER_SIZE_BYTES];
        dis.readFully(bytes); dis.skipBytes(4);
        return Bitmap64DeletionVector.deserializeFromBitmapDataBytes(bytes);
    }
    throw new RuntimeException("Invalid magic number: " + magicNumber);
}
```

**为什么用 Magic Number 而不是外部版本号**：DV 数据块自包含，每个 DV 独立可解析，因此**同一张表、甚至同一个索引文件**里可以混存 V1/V2，无需任何全局元数据协调升级。`length` 参数（来自 `DeletionVectorMeta.length`）用于做一致性校验，注意 V2 的 `bitmapLength` 不含 length/crc 字段，校验时要先扣除（`DeletionVector.java:118-130`）。

**Factory 的三种来源**（`DeletionVector.java:148-169`）决定读侧 DV 从哪来：① `emptyFactory()` 恒返回空（无 DV 场景）；② `factory(BucketedDvMaintainer)` 从内存 map 取（写进程内复用，方法引用 `dvMaintainer::deletionVectorOf`）；③ `factory(FileIO, files, deletionFiles)` 按需从文件读，**懒加载**——一个 Split 多个数据文件，只有真正要读某文件时才打开它的 DV，避免把整 bucket 的 DV 全拉进内存。

**⑤ 收益与代价**：用位图而非独立 delete 文件，`isDeleted(pos)` 是 O(1) 内存判定、无需 join；Roaring 对稀疏/密集删除都极致压缩；OR 合并天然支持增量。代价是位图必须整块加载到内存才能判定（不能像 B+Tree 那样部分加载），所以单个 DV 不宜过大——这也是 §3 要把 DV 切成"每数据文件一个、索引文件按 size 滚动"的原因。

---

## 3. DV 文件组织：索引文件 + dvRanges 元数据

### 3.1 一个 bucket 一个索引文件：DeletionVectorsIndexFile / DeletionFileWriter

**① 要解决什么问题**

一个 bucket 可能有成百上千个数据文件，其中相当一部分有 DV。两种朴素方案都不行：每个数据文件一个 DV 文件 → 对象存储小文件爆炸；所有 DV 塞一个文件又没索引 → 读单个 DV 要扫全文件。

**② 设计原理与取舍**

Paimon 选"**一个物理索引文件顺序存放多个 DV + 一张 offset 表**"：DV 顺序追加进 `.index` 文件，每个 DV 在文件中的位置由 `DeletionVectorMeta(dataFileName, offset, length, cardinality)` 记录，这张表（`IndexFileMeta.dvRanges()`）随 `IndexFileMeta` 进 manifest。读时按 offset/length 精确 seek，写时纯顺序追加。

物理布局（`DeletionVectorsIndexFile.java:44`，类型标识 `"DELETION_VECTORS"`，文件首字节版本号 `VERSION_ID_V1 = 1`）：

```
.index 文件:  [version(1B)=1][DV_A: len|magic|data|crc][DV_B: ...][DV_C: ...] ...
dvRanges:     A→(offset, length, cardinality)  B→(...)  C→(...)
```

`DeletionFileWriter` 是顺序追加写的本体（`DeletionFileWriter.java:36`）：

```java
// DeletionFileWriter.java:43 / :55  —— 行号已核对
public DeletionFileWriter(IndexPathFactory pathFactory, FileIO fileIO) throws IOException {
    this.out = new DataOutputStream(fileIO.newOutputStream(path, true));
    out.writeByte(VERSION_ID_V1);                    // 文件头 1 字节版本号
    this.dvMetas = new LinkedHashMap<>();            // 保序：读侧顺序 I/O 依赖此顺序
}
public void write(String key, DeletionVector dv) throws IOException {
    int start  = out.size();                         // 第一个 DV 的 offset 是 1（版本号占了第 0 字节）
    int length = dv.serializeTo(out);
    dvMetas.put(key, new DeletionVectorMeta(key, start, length, dv.getCardinality()));
}
```

两个易忽视的细节：① `dvMetas` 用 `LinkedHashMap` 保序，因为读侧 `readAllDeletionVectors` 按 `dvRanges` 顺序逐个反序列化，保序才能保证顺序 I/O（`DeletionVectorsIndexFile.java:82`）；② **第一个 DV 的 offset 是 1 不是 0**，因为第 0 字节是版本号——手写解析时极易踩坑。

**为什么记 `cardinality`**：它是该 DV 已删行数，标 `@Nullable BigIntType(true)`（`DeletionVectorMeta.java:35-40`，后向兼容：旧数据可能没有）。查询计划用 `rowCount - Σcardinality` 估有效行数做代价估算；运维侧用它判断 compaction 是否滞后（见 §1.3 陷阱 3）。

**读取的三个入口**（`DeletionVectorsIndexFile.java`）：`readAllDeletionVectors(IndexFileMeta)`（`:73`，恢复/扫描时一次读全部，先 `checkVersion` 校验首字节）、`readDeletionVector(Map)`（`:105`，要求同属一个索引文件、批量 seek）、`readDeletionVector(DeletionFile)`（`:129`，读单个）。

### 3.2 单文件写入 vs Rolling 写入

**① 要解决什么问题**：PK 表（分桶）每个 bucket 只一个 DV 索引文件，修改时整体重写即可；但无桶（unaware）Append 表一个分区可能有上万个数据文件，所有 DV 挤一个索引文件会让它无限膨胀。

**② 设计原理与取舍**：`DeletionVectorsIndexFile` 暴露两种写法（`DeletionVectorsIndexFile.java:142/150`），按表形态二选一：

- `writeSingleFile(Map)`：所有 DV 写进一个文件，给 PK 表 / 分桶 Append 表用（`BucketedDvMaintainer` 调它）。
- `writeWithRolling(Map)`：边写边按 `targetSizePerIndexFile`（默认 2MB）滚动切文件，给无桶 Append 表用。

Rolling 的实现是"写完一个 DV 再判断是否超限"（`DeletionVectorIndexFileWriter`）：

```java
// DeletionVectorIndexFileWriter.tryWriter —— 行号已核对（DeletionVectorIndexFileWriter.java）
while (iterator.hasNext()) {
    Map.Entry<String, DeletionVector> entry = iterator.next();
    writer.write(entry.getKey(), entry.getValue());
    if (writer.getPos() > targetSizeInBytes) break;   // 已经超了才 break
}
```

**④ 风险/边界**：因为"先写后判断"，单个 Rolling 文件实际大小最多超出 `targetSize` 一个 DV 的体量。`targetSizePerIndexFile` 配小 → 文件数过多；配大 → 读单个 DV 时 seek 范围内白读 I/O。读侧每次 `readAllDeletionVectors` 都先 `checkVersion`（读首字节校验 = 1），版本不符直接抛异常，避免静默读到脏数据。

**⑤ 收益与代价**：索引文件 + dvRanges 这套组织把"N 个数据文件的 DV"压成"O(N/每文件DV数) 个索引文件 + 一张 offset 表"，既不爆小文件、又支持精确 seek。代价是修改任一 DV 都要重写它所在的整个索引文件（对象存储不能原地改）——这对 PK 表无所谓（反正整 bucket 一个文件、本来就整体重写），但对无桶 Append 表就引出了 §7 的"未改动 DV 也要搬家"复杂度。

---

## 4. BucketedDvMaintainer：DV 的内存维护与增量提交

**① 要解决什么问题**

一次 checkpoint 周期内会产生大量删除标记（compaction 时每命中一个旧版本就标一行）。如果每个标记都落一次盘，会小文件爆炸；如果不知道"这一轮有没有真改动"，又会每次提交都重写一遍 DV 文件。`BucketedDvMaintainer` 在内存里维护单个 `(partition, bucket)` 的 `文件名 → DV` 全量映射，把"高频增量标记"和"低频批量落盘"解耦。

**② 设计原理与取舍**

核心状态只有四个字段（`BucketedDvMaintainer.java:35-40`）：`dvIndexFile`（落盘器）、`deletionVectors`（内存 map）、`bitmap64`（V1/V2 开关）、`modified`（脏标记）。三个写入口对应三种语义：

```java
// BucketedDvMaintainer.java:61 / :87 / :102  —— 行号已核对
// (a) 单行标删：lookup compaction 时逐行调用，computeIfAbsent 懒建 DV，checkedDelete 去重
public void notifyNewDeletion(String fileName, long position) {
    DeletionVector dv = deletionVectors.computeIfAbsent(fileName, k -> createNewDeletionVector());
    if (dv.checkedDelete(position)) { modified = true; }   // 只有真新增才置脏
}
// (b) 整 DV 合并：Append 表批量 DELETE 用，与已有 DV 取并集
public void mergeNewDeletion(String fileName, DeletionVector dv) {
    DeletionVector old = deletionVectors.get(fileName);
    if (old != null) { dv.merge(old); }                    // OR 语义，新 DV 吸收旧 DV
    deletionVectors.put(fileName, dv); modified = true;
}
// (c) 移除：compaction 时旧文件被合并掉，它的 DV 不再需要
public void removeDeletionVectorOf(String fileName) {
    if (deletionVectors.containsKey(fileName)) { deletionVectors.remove(fileName); modified = true; }
}
```

`createNewDeletionVector()`（`:50`）按 `bitmap64` 配置 new 出 V1 或 V2，所以一张表的 V1/V2 选择是在维护器层统一定的。

**提交时落盘只看 `modified`**（`BucketedDvMaintainer.java:115`）：

```java
public Optional<IndexFileMeta> writeDeletionVectorsIndex() {
    if (modified) { modified = false; return Optional.of(dvIndexFile.writeSingleFile(deletionVectors)); }
    return Optional.empty();   // 这一轮没改动，不产生任何 DV 文件
}
```

返回 `Optional` 而非强制写文件，是为了**在没有删除发生的提交里不产生空/重复的 DV 小文件**——这是 `modified` 标记的全部价值。写完置 `false`，下一轮只在又有新改动时才写。

**③ 状态恢复（容错的关键）**：`Factory.create(partition, bucket, restoredFiles)`（`BucketedDvMaintainer.java:164`）从 Writer 自己保存的 `restoredFiles`（已提交的 `IndexFileMeta` 列表）重建内存 map：

```java
public BucketedDvMaintainer create(BinaryRow partition, int bucket, @Nullable List<IndexFileMeta> restoredFiles) {
    if (restoredFiles == null) { restoredFiles = Collections.emptyList(); }   // 容 null
    Map<String, DeletionVector> dvs =
        new HashMap<>(handler.readAllDeletionVectors(partition, bucket, restoredFiles));
    return create(partition, bucket, dvs);
}
```

**为什么从 `restoredFiles` 恢复而不是从最新 Snapshot 读**：failover 时最新 Snapshot 可能已包含其它并行 Writer 的提交，从全局快照恢复会把别人的 DV 也吃进来。只有从本 Writer 自身 checkpoint 的 `restoredFiles` 恢复，才能保证状态一致。

**④ 风险/陷阱**：① 直接 `deletionVectors.put(...)` 绕过 `notifyNewDeletion` 会漏置 `modified`，导致删除丢失——务必走维护器方法；② `mergeNewDeletion` 是 `dv.merge(old)`（让新 DV 吸收旧 DV 再替换），方向选择无关正确性（OR 可交换），但要求两个 DV 同为 V1 或同为 V2，否则 `merge` 抛 `Only instance with the same class type can be merged`（`BitmapDeletionVector.java:60`）。

**⑤ 收益与代价**：内存 map 让高频标记 O(1)、批量落盘减小文件、`modified` 免无谓写、Factory 撑 failover。代价是要把整 bucket 的 DV 常驻内存——对分桶表 DV 数量可控、没问题，但这也是 DV 只适合"桶内文件数受控"的隐含前提。

---

## 5. MOW 写路径：DV 在 lookup compaction 时生成（核心）

### 5.1 DV 不是在 flush 时生成，而是在 compaction 时

**① 要解决什么问题 / 旧稿的根本误解**

很多资料（包括本文旧稿）把 DV 描述成"写入一条更新就 lookup 旧记录、立即标删"。这在 Paimon 里**不成立**：写入侧 `MergeTreeWriter.write(kv)` 只是把 KeyValue 排序进 WriteBuffer，buffer 满了 flush 成一个 **Level 0** 文件——此时**完全没有 DV，也没有任何 lookup**。Level 0 文件 key 范围互相重叠，是否要标删别人、标删谁都还不知道。

DV 真正生成的地方是 **lookup compaction**：当 compaction 把 Level 0 文件向高层推时，对参与的每个 key 去更高 level 做一次 lookup，**若发现该 key 在更高 level 已存在旧版本**，就把"旧版本所在文件的那一行"标记为删除。

**② 设计原理与取舍**

入口在 `LookupChangelogMergeFunctionWrapper.getResult()`（`LookupChangelogMergeFunctionWrapper.java:104`）。它先在参与 merge 的 section 里找最高层版本，没找到才去 lookup 更高层：

```java
// LookupChangelogMergeFunctionWrapper.java:104  —— 行号已核对
public ChangelogResult getResult() {
    KeyValue highLevel = mergeFunction.pickHighLevel();
    boolean containLevel0 = mergeFunction.containLevel0();
    if (highLevel == null) {                       // 本次 section 内没有更高层版本
        T lookupResult = lookup.apply(mergeFunction.key());   // 去 outputLevel+1 及以上做点查
        if (lookupResult != null) {
            if (lookupStrategy.deletionVector) {              // DV 模式：拿到旧版本的文件名+行号
                // PositionedKeyValue / FilePosition 都携带 (fileName, rowPosition)
                deletionVectorsMaintainer.notifyNewDeletion(fileName, rowPosition); // ← DV 在这里生成
            } else {
                highLevel = (KeyValue) lookupResult;          // 非 DV 模式（如 lookup changelog）
            }
        }
    }
    // ... 计算最终值、产出 changelog
}
```

注意 lookup 目标是 `lookupLevels.lookup(key, outputLevel + 1)`（`LookupMergeTreeCompactRewriter.java:212`）——只查**输出层以上**的层。这意味着 DV 标记的永远是"被本次 compaction 输出的新版本所顶替的、位于更高 level 的旧行"。

而触发这条路径的前提，是 `LookupMergeTreeCompactRewriter.upgradeStrategy`（`:133`）决定对 Level 0 文件**强制 rewrite**：

```java
// LookupMergeTreeCompactRewriter.java:133  —— 行号已核对
protected UpgradeStrategy upgradeStrategy(int outputLevel, DataFileMeta file) {
    if (file.level() != 0) { return NO_CHANGELOG_NO_REWRITE; }      // 非 L0 文件不重写
    // DV 模式下，只要文件有删除行就必须 rewrite（因为要 drop delete）
    if (dvMaintainer != null && file.deleteRowCount().map(c -> c > 0).orElse(true)) {
        return CHANGELOG_WITH_REWRITE;
    }
    // ...
}
```

**一句话**：DV 模式下 Level 0 文件必须经过一次 compaction rewrite 才会"落定"，rewrite 过程顺带 lookup 高层、生成 DV。这就是为什么 MOW 表 Level 0 在 compaction 前对读不可见。

### 5.2 DV 与 LSM 各 level 的关系

把上面拼起来，DV 与 level 的对应关系是理解 MOW 的核心：

- **Level 0 文件**：刚 flush 出来、key 重叠、还没 DV。它们是"新写入的最新版本"，对读不可见，等待 compaction。
- **更高 level 文件（L1..Ln）**：存量数据，可能被 Level 0 里同 key 的新版本顶替。DV 标记的就是**这些高层文件里被顶替的行**。
- **compaction 时**：被 rewrite 的 Level 0 数据写进新文件（成为新的高层文件），同时为"被它们顶替的旧高层行"生成/更新 DV。

所以 DV 始终是"高层旧文件 + 它的失效行位图"。读取时，每个数据文件配它自己的 DV，逐文件顺序读、按行跳过——不再需要跨 level 归并。这正是 MOW 让 PK 表读取接近 Append 表的根因（详见 §6）。

> Lookup 本身（`key → (fileName, rowPosition)` 的点查能力）由 `LookupLevels` + 状态后端（RocksDB / 内存）提供，是把列式文件转成本地 KV 索引的点查加速器。它的内部结构、远程 lookup 文件、RocksDB 调优等不在本文展开，详见 [01 §6 LookupLevels](01-核心存储引擎分析.md)。

### 5.3 compaction 如何"消除"旧 DV

compaction 不只是生成 DV，也是**回收 DV** 的唯一途径。`notifyRewriteCompactBefore`（`LookupMergeTreeCompactRewriter.java:103`）在 rewrite 开始前，把所有参与 compaction 的旧文件的 DV 从维护器里移除：

```java
// LookupMergeTreeCompactRewriter.java:103  —— 行号已核对
protected void notifyRewriteCompactBefore(List<DataFileMeta> files) {
    if (dvMaintainer != null) {
        files.forEach(file -> dvMaintainer.removeDeletionVectorOf(file.fileName()));
    }
}
```

逻辑闭环：被 compact 的旧文件中，被 DV 标删的行不会写进新文件（drop delete），所以**新文件里全是有效行、不需要 DV**；旧文件连同它的 DV 一起退场。提交时，旧 DV 索引文件标 `IndexManifestEntry DELETE`、新 DV 索引文件（若有）标 `ADD`，与数据文件增量一起进同一个 Snapshot。这把"读时 merge"的代价彻底物化成了"compaction 时一次性付清"。

### 5.4 DV 生命周期串讲

以一条 `(pk=42)` 的更新在 MOW 表里的旅程为例：

```
1. write(新版本v2)        → 排序进 WriteBuffer，flush 成 Level 0 文件 F_new（无 DV，对读不可见）
2. compaction 触发        → upgradeStrategy 对 L0 文件 F_new 判定 CHANGELOG_WITH_REWRITE
3. rewrite 中 lookup      → 对 pk=42 查 outputLevel+1 以上，发现旧版本 v1 在高层文件 F_old 第 N 行
4. notifyNewDeletion      → dvMaintainer 标记 F_old 第 N 行失效（checkedDelete 去重、置 modified）
5. notifyRewriteCompactBefore → 若 F_old 自己也参与了本次 compaction，则其 DV 被 removeDeletionVectorOf 回收
6. prepareCommit          → writeDeletionVectorsIndex() 把 F_old 的 DV 写成新索引文件（modified 才写）
7. commit                 → 数据文件增量 + IndexIncrement(新 DV ADD / 旧 DV DELETE) 原子进同一 Snapshot
8. 读取 pk=42             → 顺序读 F_old，DV 跳过第 N 行；读 F_new 拿到 v2；无需归并（§6）
```

**④ 风险/陷阱**：① MOW 表的 compaction 不能纯异步，否则新数据可见延迟（§1.3 陷阱 1）；② lookup 在写期是真实开销（要从高层文件/索引点查），更新越频繁、命中越多，写吞吐越受 lookup 性能影响——这是 MOW 相对纯 MOR 多付的写代价；③ `deletion-vectors.enabled` 默认 `false`（`CoreOptions.java:1860`），不开就是 MOR。

**⑤ 收益与代价**：把 merge 从"每次读"降为"compaction 一次"，读取接近 Append。代价是写期多一次 lookup + compaction 必须同步 + DV 索引文件的额外写。适合"读多写少、点查/OLAP、batch 读"，不适合"超高频更新且几乎不读"的纯摄入场景（那种情况 MOR + 异步 compaction 更划算）。

---

## 6. DV 读路径：按行过滤与装饰器

**① 要解决什么问题**

DV 已经在写期把"哪些行失效"记好了，读期要做的是：跳过 merge、逐文件顺序读、按行号扔掉失效行，且这个过滤必须**对文件格式透明**、能和文件索引返回的行选择叠加。

**② 设计原理与取舍：装饰器 + 迭代器层过滤**

入口在 `RawFileSplitRead`（`RawFileSplitRead.java:179`）：为 split 里的每个数据文件建一个懒加载 DV factory，读到某文件时拿它的 DV，**非空则用 `ApplyDeletionVectorReader` 把底层 reader 包一层**：

```java
// RawFileSplitRead.java:262 / :308  —— 行号已核对
DeletionVector deletionVector = dvFactory == null ? null : dvFactory.get();
// ...（先做文件索引：FileIndexEvaluator 可能返回 BitmapIndexResult 作为行 selection）
FileRecordReader<InternalRow> fileRecordReader = new DataFileRecordReader(...);
if (fileIndexResult instanceof BitmapIndexResult) {                 // 文件索引的行级 selection
    fileRecordReader = new ApplyBitmapIndexRecordReader(fileRecordReader, (BitmapIndexResult) fileIndexResult);
}
if (deletionVector != null && !deletionVector.isEmpty()) {          // DV 行级过滤叠加在最外层
    return new ApplyDeletionVectorReader(fileRecordReader, deletionVector);
}
return fileRecordReader;
```

注意两层行级过滤的**叠加顺序**：文件索引（Bitmap/BSI，详见 [13 号文档](13-索引机制深度分析.md)）先在 `DataFileRecordReader` 上挑出"谓词命中的行"，DV 再在更外层剔除"逻辑失效的行"。两者都基于"文件内行号"，互不冲突。

`ApplyDeletionVectorReader` 是纯装饰器（`ApplyDeletionVectorReader.java:31`），不改底层读逻辑，只在每个 batch 外包一层过滤迭代器：

```java
// ApplyDeletionVectorReader.java:53 + ApplyDeletionFileRecordIterator —— 行号已核对
public FileRecordIterator<InternalRow> readBatch() throws IOException {
    FileRecordIterator<InternalRow> batch = reader.readBatch();
    return batch == null ? null : new ApplyDeletionFileRecordIterator(batch, deletionVector);
}
// 迭代器 next()：用 while 跳过连续失效行，靠底层 returnedPosition() 拿"文件内行号"
public InternalRow next() throws IOException {
    while (true) {
        InternalRow next = iterator.next();
        if (next == null) return null;
        if (!deletionVector.isDeleted(returnedPosition())) return next;  // 未删则返回
        // 已删则跳过，继续读下一行
    }
}
```

**为什么在迭代器层而不是 batch/Reader 层过滤**：Reader 以 batch 为粒度（一批多行），改 batch 内部结构复杂；迭代器逐行检查、`returnedPosition()` 直接给文件内行号（与 DV 记录的 position 同源），简单且 O(1) 每行。

**③ 为什么读期能不 merge**：当 split 标记为 `rawConvertible` 时，走 `RawFileSplitRead` 而非 `MergeFileSplitRead`——即"逐文件直读 + DV 过滤"，跳过 `SortMergeReader` 归并。`MergeFileSplitRead` 路径下也会构造 DV factory（`MergeFileSplitRead.java:257`）用于 drop 已删行，但真正让 PK 表读取接近 Append 的，是 `rawConvertible` 让大部分文件走免归并的 raw 读。`rawConvertible` 的判定属于 scan 层（详见 [01 §7.1/§12.4](01-核心存储引擎分析.md)）。

**④ 风险/陷阱**：① DV 行号是**文件内 0-based 偏移**，迭代器若 `returnedPosition()` 返回"已读总行数"而非"文件内位置"，过滤就会错乱——这也是为什么 DV 严格绑定 `(数据文件, 行号)`、而非全局行号；② DV 命中率过高（一个文件大半被标删）说明 compaction 滞后，读时白读 I/O 多，应触发 compaction；③ 读期还要为有 DV 的文件多读一次 DV 索引文件（小额 I/O）。

**⑤ 收益与代价**：读取从"跨多层归并"降为"顺序读 + O(1) 位图判定"，逼近 Append 表吞吐；代价是每个有 DV 的文件多一次 DV 加载、且整块位图须驻内存。与 Iceberg position-delete 需要"data 文件 join delete 文件"相比，DV 嵌入读路径、零 join。

---

## 7. Append 表的 DV：行级 DELETE 与 Rolling 维护

**① 要解决什么问题**

Append 表不支持 UPDATE，但支持 `DELETE FROM append_table WHERE ...`。若按 COW 整文件重写来删几行，代价极高。DV 让 Append 表也能"标记某文件某行失效"而不重写数据文件——但 Append 表的删除是**用户 DELETE 语句一次性给出"这次要删的整个 DV"**，与 PK 表"compaction 时逐行通知"语义不同，因此维护器接口也不同。

**② 设计原理与取舍**

接口 `BaseAppendDeleteFileMaintainer`（`:52`）只两个动作：`notifyNewDeletionVector(dataFile, dv)`（标删）、`persist()`（落盘出 `IndexManifestEntry`）。两个工厂按桶形态分流（`:62/:77`）：

- 分桶 Append → `BucketedAppendDeleteFileMaintainer`，内部直接委托 `BucketedDvMaintainer`（和 PK 表共用维护器，单文件写）。
- 无桶（unaware）Append → `AppendDeleteFileMaintainer`，**复杂度最高**，Rolling 写。

**分桶版本极简**（`BucketedAppendDeleteFileMaintainer.java:55`）——`notifyNewDeletionVector` 转 `mergeNewDeletion`（整 DV 合并而非逐行），`persist` 转 `writeDeletionVectorsIndex` 包成 `FileKind.ADD` 条目。为什么用 `mergeNewDeletion`：DELETE 一次给一个完整 DV，要与该文件已有 DV 取并集，否则上次删的行会"复活"。

**无桶版本为何复杂**：无桶表的 DV 分散在多个 Rolling 索引文件里，一个索引文件含多个数据文件的 DV。要改其中一个数据文件的 DV，就得动它所在的整个索引文件。核心是 `notifyNewDeletionVector` + `persist` 两步（`AppendDeleteFileMaintainer.java:119/128`）：

```java
// AppendDeleteFileMaintainer.java:119  —— 行号已核对
public void notifyNewDeletionVector(String dataFile, DeletionVector dv) {
    DeletionFile previous = notifyRemovedDeletionVector(dataFile); // 把该文件从旧索引文件"摘除"，并标记 touched
    if (previous != null) { dv.merge(dvIndexFile.readDeletionVector(previous)); } // 合并旧 DV，防复活
    deletionVectors.put(dataFile, dv);
}
public List<IndexManifestEntry> persist() {
    List<IndexManifestEntry> result = writeUnchangedDeletionVector();      // ① 搬家：未改动的 DV
    dvIndexFile.writeWithRolling(deletionVectors).stream()                 // ② 新增/改动的 DV 滚动写
            .map(this::toAddEntry).forEach(result::add);
    return result;
}
```

关键在 `writeUnchangedDeletionVector`（`:146`）：被 touched 的索引文件里，那些**没改动的 DV 也必须重写到新文件**、旧索引文件整体标 `toDeleteEntry()`。因为对象存储不能原地改，一个索引文件只要被动了一处，就得整体重生。它维护一组辅助 map（`dataFileToIndexFile` / `indexFileToDeletionFiles` / `touchedIndexFiles` 等，`:46-51`）来追踪"哪个数据文件的 DV 在哪个索引文件、哪些索引文件被碰过"。

**③ Append 与 PK 表 DV 的差异对照**

| 维度 | PK 表 DV | Append 表 DV |
|------|----------|--------------|
| 产生时机 | lookup compaction 时逐行 `notifyNewDeletion` 自动生成 | 用户 `DELETE` 语句一次性给整个 DV |
| 维护器 | `BucketedDvMaintainer` | 分桶→`BucketedAppendDeleteFileMaintainer`（委托前者）；无桶→`AppendDeleteFileMaintainer` |
| 写入模式 | 单文件 `writeSingleFile` | 分桶单文件；无桶 `writeWithRolling` |
| compaction 消除 | 是，旧文件 DV 随 compaction 自动回收 | 否（除非被 compact 的文件恰好含 DV） |
| 索引文件粒度 | 整个 bucket 一个 | 无桶按 size Rolling 多个，部分修改要"搬家" |

**④ 风险/陷阱**：无桶 Append 表频繁小批 DELETE 会反复触发"未改动 DV 搬家 + 旧索引文件 DELETE"，放大索引文件写与 manifest 条目数；`notifyNewDeletionVector` 必须先 merge 旧 DV，漏 merge 会让历史删除复活。

**⑤ 收益与代价**：Append 表获得"不重写数据文件即可行级删除"的能力；代价是无桶形态下索引文件维护远比 PK 表重。若 DELETE 频繁，分桶 Append 表比无桶更可控。

---

## 8. MOW vs COW vs MOR：行级标删的取舍边界

**① 要解决什么问题**：同样是"主键表能 update/delete"，业界有三条路。理解它们的代价边界，才知道 DV（MOW）什么时候是收益、什么时候是负担。

**② 三种范式的本质对比**

| | MOR（无 DV） | **MOW（DV）** | COW（整文件重写） |
|---|---|---|---|
| 更新一行做什么 | 追加新版本，旧版本留着 | 追加新版本 + compaction 时标删旧行 | 读出整个旧文件、改一行、写新整文件 |
| 写放大 | 最低（纯追加） | 中（lookup + DV 写 + compaction） | 最高（GB 级文件改一行也重写） |
| 读放大 | 最高（每次跨层归并） | 低（顺序读 + 位图跳行） | 最低（文件即最新） |
| 旧数据何时真正消失 | compaction | compaction（标删 + drop delete） | 立即（被新文件替代） |
| 空间放大 | 中（旧版本 + changelog） | 中（旧文件 + DV 共存到 compaction） | 低 |
| Paimon 用否 | 用（默认，`deletion-vectors.enabled=false`） | 用（开 DV） | **不用于主键表** |

**③ 为什么 Paimon 主键表不走 COW**：COW 改一行重写整文件，在对象存储上写放大灾难性（详见 [01 §1.1 LSM vs B+Tree](01-核心存储引擎分析.md) 对 Delta/Iceberg CoW 的对比）。Paimon 用 LSM 追加 + 后台 compaction，本质上把"何时重写"推迟并批量化。DV 是在这套 LSM 之上叠的一层：**让"重写聚合后的文件"和"标记被聚合掉的旧行"同时在 compaction 完成**，从而读取免归并。

**④ DV 什么时候反而是负担**：
- 纯摄入、几乎不查的场景：DV 的读收益用不上，却白付 lookup + compaction 同步的写代价——这种场景 MOR + 异步 compaction 更划算。
- 更新极频繁且 compaction 跟不上：DV 大量累积，读时白读被标删行、写时 lookup 命中率高，双向变慢（§1.3 陷阱 3）。
- 桶内文件数失控：DV 维护器要把整 bucket 的 DV 常驻内存（§4），桶过大时内存压力上升。

**⑤ 结论**：DV/MOW 是"**读多写少、点查/OLAP/batch 读**"的最优解；"**超高频写、极少读**"仍应留在 MOR。这也是 `deletion-vectors.enabled` 默认 `false`、需显式开启的原因。

---

## 9. 文件索引交汇点（详见 13 号文档）

> 文件索引（Bloom Filter / Bitmap / BSI / Range Bitmap / FileIndexFormat / FileIndexResult 三态 / FileIndexPredicate / Predicate 体系 / 全局索引 / Lookup 状态后端）的**完整原理与实现由 [13 号文档](13-索引机制深度分析.md) 主讲**，本文不重复展开。这里只说清它与 DV 在读路径上的交汇关系。

**① 二者解决的问题不同，但都作用在"文件内行号"上**：

- **文件索引**：读前过滤。Bloom Filter 答"这个值可能在/一定不在本文件"（文件级 SKIP/REMAIN）；Bitmap/BSI/Range Bitmap 能进一步给出"哪些行命中谓词"的行号位图（行级 `BitmapIndexResult`）。
- **DV**：读时剔除逻辑失效行。

**② 在读路径如何叠加**（见 §6 与 `RawFileSplitRead.java:262-309`）：先跑文件索引（`FileIndexEvaluator`）——若整体 SKIP 直接返回空 reader；若返回行级 `BitmapIndexResult`，用 `ApplyBitmapIndexRecordReader` 在数据 reader 上挑出"谓词命中行"；DV 再在最外层用 `ApplyDeletionVectorReader` 剔除"失效行"。两者都基于文件内 0-based 行号，互不冲突、可叠加。

```
DataFileRecordReader（读数据文件）
  → ApplyBitmapIndexRecordReader（文件索引行选择：留下谓词命中行）   ← 详见 13 号
  → ApplyDeletionVectorReader（DV：剔除失效行）                      ← 本文 §6
```

**③ 一个值得知道的细节**：`FileIndexEvaluator.evaluate` 接收 `deletionVector` 参数（`RawFileSplitRead.java:273`），表示文件索引的行选择结果与 DV 可以在评估阶段就协同（如 BitmapIndexResult 提供 `andNot(deletion)` 从命中集合里再去掉已删行）。但索引内部的三态运算、延迟计算、TopN 等机制属于 13 号文档范畴。

**与本文的边界**：凡涉及"索引怎么建、怎么编码、怎么求值、Predicate 怎么下推、Lookup 状态后端（RocksDB/InMemory）怎么调优"——一律见 [13 号文档](13-索引机制深度分析.md)；本文只保证你理解"DV 在读路径上和文件索引各管一段、如何叠加"。

---

## 10. 与 Iceberg DV 对比

| 维度 | Paimon DV | Iceberg DV |
|------|-----------|------------|
| **引入版本** | Paimon 0.7+ | Iceberg V3 (format-version=3) |
| **位图实现** | RoaringBitmap32 (V1) / OptimizedRoaringBitmap64 (V2) | Roaring64Bitmap (Puffin blob) |
| **存储方式** | 独立的 DV 索引文件 (`DELETION_VECTORS` 类型) | Puffin 文件格式 (`.puffin`) |
| **关联方式** | 通过 IndexManifest 中的 IndexFileMeta | 通过 Manifest 中的 content_offset/content_length |
| **粒度** | 每个 bucket 一个 DV 文件 (PK 表) 或 Rolling 多个文件 (Append 表) | 每个 data file 一个 DV blob |
| **更新方式** | 增量合并 (merge + writeSingleFile 整体重写) | Copy-on-Write (重写整个 Puffin 文件) |
| **产生方式** | PK 表在 **lookup compaction** 时自动生成；Append 表由 DELETE 语句生成 | 需要显式执行 DELETE 或 rewrite |
| **用途** | 消除 merge-on-read 开销，使 PK 表读取接近 Append 表 | 替代 position delete 文件 |
| **与 Compaction 关系** | Compaction 既生成 DV、也消除 DV (旧文件 DV 被移除) | 没有直接关系，需要单独的 rewrite 操作 |
| **格式兼容性** | V2 格式参考 Iceberg DV 格式 (Little-Endian, 64-bit Roaring) | Iceberg 标准格式 |
| **Checkpoint 恢复** | 从 Writer 保存的 restoredFiles 恢复 | 从 Iceberg Snapshot 恢复 |
| **实时性** | 实时写入 + 同步 compaction 时产生 DV | 通常由 Spark/Trino 等批处理引擎产生 |

**Paimon DV 的独特优势**:

1. **与 LSM-tree 深度集成**: Paimon 的 DV 在 **lookup compaction** 时自动生成——compaction 把 Level 0 推向高层、对每个 key lookup 更高 level 的旧版本并标删，无需用户额外操作。

2. **Compaction 既产又消**: LSM compaction 一边为"被新版本顶替的旧行"生成 DV，一边把"参与本次 compaction 的旧文件的 DV"回收（`notifyRewriteCompactBefore`），不需要像 Iceberg 那样手动 `rewrite`。

3. **读取优化**: DV 启用后 PK 表读取跳过 LSM merge，逐文件顺序读 + 位图跳行，性能接近 Append 表。

4. **Append 表也支持**: Paimon 允许在 Append 表上用 DV 实现行级 `DELETE`，而 Iceberg 的 DV 仅用于替代 position delete。

---

## 11. 设计决策总结

| 决策点 | 选择 | 取舍/代价 | 收益 |
|--------|------|-----------|------|
| 删除标记的形态 | per-data-file 的 **Roaring 位图**（DV），而非独立 delete 文件 | 整块位图须驻内存判定；不能部分加载 | `isDeleted` O(1)、零 join、稀疏/密集都极致压缩、支持 OR 合并 |
| merge 时机 | **MOW**：compaction 时归并并生成 DV，读时免归并 | 写期多一次 lookup + compaction 必须同步 | 读取从"每查必归并"降到接近 Append 表 |
| 不走 COW | 主键表用 LSM 追加 + 后台 compaction + DV | 旧文件与 DV 共存到下次 compaction（空间放大） | 避免"改一行重写整文件"的写放大灾难 |
| DV 生成位置 | lookup compaction（`LookupChangelogMergeFunctionWrapper`），**非 flush** | Level 0 在 compaction 前对读不可见 | 只标记"被聚合掉的旧高层行"，逻辑闭环、可被 compaction 回收 |
| V1/V2 双格式 | Magic Number 自动分派，可同表混存 | 两套序列化（BE/LE）维护成本 | V1 省（2^31 行内）、V2 兼容 Iceberg 且支持超大文件 + RLE |
| 索引文件组织 | 一个物理 `.index` 顺序存多 DV + `dvRanges` offset 表 | 改任一 DV 须重写整个索引文件 | 不爆小文件、精确 seek、顺序写 |
| 写入模式分流 | 分桶/PK 单文件重写；无桶 Append 按 size Rolling | 无桶部分修改要"未改动 DV 搬家" | 适配两种桶形态的文件数规模 |
| 内存维护 + `modified` | `BucketedDvMaintainer` 内存 map + 脏标记，提交时整体写 | 整 bucket DV 常驻内存 | 高频标记 O(1)、无改动不写文件、支持 failover 恢复 |
| 与文件索引叠加 | DV 在最外层、文件索引行选择在内层，同基于文件内行号 | 两层装饰器 | 谓词命中行 + 失效行剔除互不冲突、可组合 |
| 默认关闭 | `deletion-vectors.enabled` 默认 `false` | 用户需显式开启 | 超高频写/极少读场景仍可留在低写放大的 MOR |
