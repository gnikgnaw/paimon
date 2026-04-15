# Apache Paimon Merge 引擎与聚合函数体系 -- 源码深度分析

> **基于版本**: Apache Paimon 1.5-SNAPSHOT (commit 7c93bd720)
> **分析日期**: 2026-04-15
> **核心源码路径**: `paimon-core/src/main/java/org/apache/paimon/mergetree/compact/`

---

## 目录

- [1. 概述与设计哲学](#1-概述与设计哲学)
- [2. MergeFunction 接口体系](#2-mergefunction-接口体系)
  - [2.1 核心接口定义](#21-核心接口定义)
  - [2.2 MergeFunctionFactory 工厂接口](#22-mergefunctionfactory-工厂接口)
  - [2.3 MergeFunctionWrapper 包装器接口](#23-mergefunctionwrapper-包装器接口)
  - [2.4 接口关系全景图](#24-接口关系全景图)
- [3. 四种 MergeEngine 实现](#3-四种-mergeengine-实现)
  - [3.1 Deduplicate 去重引擎](#31-deduplicate-去重引擎)
  - [3.2 PartialUpdate 部分更新引擎](#32-partialupdate-部分更新引擎)
  - [3.3 Aggregation 聚合引擎](#33-aggregation-聚合引擎)
  - [3.4 FirstRow 首行引擎](#34-firstrow-首行引擎)
  - [3.5 四种引擎对比总览](#35-四种引擎对比总览)
- [4. MergeEngine 的选择与创建流程](#4-mergeengine-的选择与创建流程)
- [5. LookupMergeFunction 包装机制](#5-lookupmergefunction-包装机制)
  - [5.1 设计动机](#51-设计动机)
  - [5.2 核心实现分析](#52-核心实现分析)
  - [5.3 LookupStrategy 策略](#53-lookupstrategy-策略)
- [6. 内置聚合函数完整列表](#6-内置聚合函数完整列表)
  - [6.1 FieldAggregator 抽象基类](#61-fieldaggregator-抽象基类)
  - [6.2 所有内置聚合函数详解](#62-所有内置聚合函数详解)
  - [6.3 FieldIgnoreRetractAgg 装饰器](#63-fieldignoreretractagg-装饰器)
- [7. FieldAggregator SPI 注册机制](#7-fieldaggregator-spi-注册机制)
  - [7.1 Factory 接口设计](#71-factory-接口设计)
  - [7.2 SPI 服务文件注册](#72-spi-服务文件注册)
  - [7.3 工厂发现与创建流程](#73-工厂发现与创建流程)
- [8. PartialUpdate Sequence Group 详解](#8-partialupdate-sequence-group-详解)
  - [8.1 设计动机与问题背景](#81-设计动机与问题背景)
  - [8.2 配置方式与解析流程](#82-配置方式与解析流程)
  - [8.3 更新逻辑深度分析](#83-更新逻辑深度分析)
  - [8.4 回撤逻辑 retractWithSequenceGroup](#84-回撤逻辑-retractwithsequencegroup)
  - [8.5 Sequence Group 中的聚合函数](#85-sequence-group-中的聚合函数)
- [9. Changelog 四种模式](#9-changelog-四种模式)
  - [9.1 NONE 模式](#91-none-模式)
  - [9.2 INPUT 模式](#92-input-模式)
  - [9.3 FULL_COMPACTION 模式](#93-full_compaction-模式)
  - [9.4 LOOKUP 模式](#94-lookup-模式)
  - [9.5 四种模式对比](#95-四种模式对比)
- [10. MergeFunctionWrapper 体系](#10-mergefunctionwrapper-体系)
  - [10.1 ReducerMergeFunctionWrapper](#101-reducermergefunctionwrapper)
  - [10.2 FullChangelogMergeFunctionWrapper](#102-fullchangelogmergefunctionwrapper)
  - [10.3 LookupChangelogMergeFunctionWrapper](#103-lookupchangelogmergefunctionwrapper)
  - [10.4 FirstRowMergeFunctionWrapper](#104-firstrowmergefunctionwrapper)
- [11. CompactRewriter 体系](#11-compactrewriter-体系)
  - [11.1 继承层次](#111-继承层次)
  - [11.2 ChangelogMergeTreeRewriter](#112-changelogmergetreerewriter)
  - [11.3 FullChangelogMergeTreeCompactRewriter](#113-fullchangelogmergetreecompactrewriter)
  - [11.4 LookupMergeTreeCompactRewriter](#114-lookupmergetreecompactrewriter)
- [12. 跨分区更新](#12-跨分区更新)
  - [12.1 判定条件](#121-判定条件)
  - [12.2 GlobalIndexAssigner 机制](#122-globalindexassigner-机制)
- [13. SortMergeReader 体系](#13-sortmergereader-体系)
  - [13.1 MinHeap 实现](#131-minheap-实现)
  - [13.2 LoserTree 实现](#132-losertree-实现)
  - [13.3 选择策略与性能对比](#133-选择策略与性能对比)
- [14. 设计决策总结](#14-设计决策总结)

---

## 1. 概述与设计哲学

Apache Paimon 的 Merge 引擎体系是其主键表(Primary Key Table)的核心，负责定义当多条拥有相同主键的记录在 LSM-Tree 合并时，如何产生最终结果。这一体系直接决定了 Paimon 在实时数据湖场景下的语义正确性和性能表现。

**为什么需要 Merge 引擎?**

Paimon 采用 LSM-Tree 存储结构，写入时数据追加到 Level-0 层的有序文件中，相同主键的不同版本可能分布在不同 Level 的不同文件中。在 Compaction(合并压缩)和 Read(读取)时，需要将同一主键的多条记录合并为一条最终结果。不同的业务场景对"合并"有不同的语义需求:

- **CDC 数据入湖**: 需要去重，保留最新值 --> Deduplicate
- **多源宽表拼接**: 每个源只更新部分列 --> PartialUpdate
- **实时指标预聚合**: 对数值型字段做 SUM/COUNT --> Aggregation
- **去重保留首条**: 只关心第一次出现的值 --> FirstRow

**设计好处**: 通过将合并策略抽象为可插拔的 `MergeFunction` 接口，Paimon 在一套 LSM-Tree 存储引擎之上支持了多种业务语义，避免了为每种场景维护独立的存储引擎。

**核心源码包结构**:

```
paimon-core/.../mergetree/compact/
├── MergeFunction.java              // 核心接口
├── MergeFunctionFactory.java       // 工厂接口
├── MergeFunctionWrapper.java       // 包装器接口
├── DeduplicateMergeFunction.java   // 去重合并
├── PartialUpdateMergeFunction.java // 部分更新合并
├── FirstRowMergeFunction.java      // 首行合并
├── LookupMergeFunction.java        // Lookup包装
├── ReducerMergeFunctionWrapper.java
├── FullChangelogMergeFunctionWrapper.java
├── LookupChangelogMergeFunctionWrapper.java
├── FirstRowMergeFunctionWrapper.java
├── SortMergeReader.java            // 排序合并读取器接口
├── SortMergeReaderWithMinHeap.java // 最小堆实现
├── SortMergeReaderWithLoserTree.java // 败者树实现
├── ChangelogResult.java
├── MergeTreeCompactRewriter.java   // 合并压缩重写器
├── ChangelogMergeTreeRewriter.java // Changelog 压缩重写器
├── FullChangelogMergeTreeCompactRewriter.java
├── LookupMergeTreeCompactRewriter.java
└── aggregate/                      // 聚合函数子包
    ├── AggregateMergeFunction.java
    ├── FieldAggregator.java        // 聚合器抽象基类
    ├── FieldSumAgg.java            // 20+ 聚合实现
    ├── ...
    └── factory/                    // SPI 工厂子包
        ├── FieldAggregatorFactory.java
        ├── FieldSumAggFactory.java
        └── ...
```

---

## 2. MergeFunction 接口体系

### 2.1 核心接口定义

**源码路径**: `paimon-core/.../mergetree/compact/MergeFunction.java`

```java
public interface MergeFunction<T> {
    void reset();           // 重置到初始状态
    void add(KeyValue kv);  // 添加一条 KeyValue 记录
    T getResult();          // 获取合并结果
    boolean requireCopy();  // 是否需要复制输入 KV
}
```

**为什么泛型参数是 `<T>` 而不是固定返回 `KeyValue`?**

虽然所有当前实现的 `T` 都是 `KeyValue`，但泛型设计为将来可能的扩展预留了空间。更重要的是，这使得 `MergeFunctionWrapper` 可以将返回类型改为 `ChangelogResult`，从而在合并过程中同时产生数据结果和 Changelog 记录。

**关于对象复用的重要约束** (源码注释):
- 不要保存 `KeyValue` 和 `InternalRow` 的引用到 List 中
- 前两个 add 的 KeyValue 对象及其内部 InternalRow 是安全的
- 第三个对象的引用可能覆盖第一个对象
- 可以安全保存字段级引用，字段不会复用对象

**好处**: 这种对象复用策略大幅减少了 GC 压力，在高吞吐量场景下性能提升显著。

### 2.2 MergeFunctionFactory 工厂接口

**源码路径**: `paimon-core/.../mergetree/compact/MergeFunctionFactory.java`

```java
@FunctionalInterface
public interface MergeFunctionFactory<T> extends Serializable {
    default MergeFunction<T> create() { return create(null); }
    MergeFunction<T> create(@Nullable RowType readType);
    default RowType adjustReadType(RowType readType) { return readType; }
}
```

**为什么 Factory 需要 `readType` 参数?**

当进行列裁剪(Column Pruning)时，读取的列可能只是表全部列的子集。`create(readType)` 允许 MergeFunction 根据实际读取的列类型来调整内部结构，例如重新映射字段索引。`adjustReadType()` 则允许 MergeFunction 声明它需要额外的列(如 PartialUpdate 需要 Sequence Group 中的序列字段)。

**好处**: 支持列裁剪优化的同时保证语义正确性，避免因缺少必要字段导致合并结果错误。

### 2.3 MergeFunctionWrapper 包装器接口

**源码路径**: `paimon-core/.../mergetree/compact/MergeFunctionWrapper.java`

```java
public interface MergeFunctionWrapper<T> {
    void reset();
    void add(KeyValue kv);
    @Nullable T getResult();
}
```

**为什么需要 Wrapper 而不是直接使用 MergeFunction?**

Wrapper 的泛型 `<T>` 可以不同于 MergeFunction 的返回类型。关键场景:
- `ReducerMergeFunctionWrapper` 返回 `KeyValue`，包装后只有一条记录时跳过合并
- `FullChangelogMergeFunctionWrapper` 返回 `ChangelogResult`，包含数据结果 + changelog 记录
- `LookupChangelogMergeFunctionWrapper` 返回 `ChangelogResult`，通过 lookup 生成 changelog

**好处**: 通过装饰器模式，将数据合并和 changelog 生成正交分离，任何 MergeFunction 都可以组合任何 Wrapper 来获得不同的 changelog 行为。

### 2.4 接口关系全景图

```mermaid
classDiagram
    class MergeFunction~T~ {
        <<interface>>
        +reset()
        +add(KeyValue kv)
        +getResult() T
        +requireCopy() boolean
    }

    class MergeFunctionFactory~T~ {
        <<interface>>
        +create() MergeFunction~T~
        +create(RowType readType) MergeFunction~T~
        +adjustReadType(RowType) RowType
    }

    class MergeFunctionWrapper~T~ {
        <<interface>>
        +reset()
        +add(KeyValue kv)
        +getResult() T
    }

    MergeFunction <|.. DeduplicateMergeFunction
    MergeFunction <|.. PartialUpdateMergeFunction
    MergeFunction <|.. AggregateMergeFunction
    MergeFunction <|.. FirstRowMergeFunction
    MergeFunction <|.. LookupMergeFunction

    MergeFunctionWrapper <|.. ReducerMergeFunctionWrapper
    MergeFunctionWrapper <|.. FullChangelogMergeFunctionWrapper
    MergeFunctionWrapper <|.. LookupChangelogMergeFunctionWrapper
    MergeFunctionWrapper <|.. FirstRowMergeFunctionWrapper

    MergeFunctionFactory --> MergeFunction : creates

    ReducerMergeFunctionWrapper --> MergeFunction : wraps
    FullChangelogMergeFunctionWrapper --> MergeFunction : wraps
    LookupChangelogMergeFunctionWrapper --> LookupMergeFunction : wraps
    FirstRowMergeFunctionWrapper --> FirstRowMergeFunction : wraps
    LookupMergeFunction --> MergeFunction : delegates
```

---

## 3. 四种 MergeEngine 实现

MergeEngine 在 `CoreOptions` 中定义为枚举:

**源码路径**: `paimon-api/.../CoreOptions.java` (第 3807 行)

```java
public enum MergeEngine implements DescribedEnum {
    DEDUPLICATE("deduplicate", "De-duplicate and keep the last row."),
    PARTIAL_UPDATE("partial-update", "Partial update non-null fields."),
    AGGREGATE("aggregation", "Aggregate fields with same primary key."),
    FIRST_ROW("first-row", "De-duplicate and keep the first row.");
}
```

### 3.1 Deduplicate 去重引擎

**源码路径**: `paimon-core/.../mergetree/compact/DeduplicateMergeFunction.java`

**核心逻辑**: 保留最后一条记录(按 sequence number 排序后最大的)。

```java
public class DeduplicateMergeFunction implements MergeFunction<KeyValue> {
    private final boolean ignoreDelete;
    private KeyValue latestKv;

    @Override
    public void add(KeyValue kv) {
        if (ignoreDelete && kv.valueKind().isRetract()) {
            return;  // 配置忽略删除时跳过回撤记录
        }
        latestKv = kv;  // 始终保留最后一条
    }

    @Override
    public KeyValue getResult() { return latestKv; }

    @Override
    public boolean requireCopy() { return false; }
}
```

**为什么 `requireCopy()` 返回 false?** Deduplicate 只保存 `latestKv` 引用，不需要拷贝，因为它只关心最后添加的那条记录，之前的记录可以被安全覆盖。

**输入输出示例**:

| 操作 | Key | Value | SequenceNumber | Level |
|------|-----|-------|----------------|-------|
| add | pk=1 | {name="A", age=20} | 1 | 2 |
| add | pk=1 | {name="B", age=25} | 3 | 0 |
| add | pk=1 | {name="C", age=null} | 5 | 0 |
| **getResult** | pk=1 | **{name="C", age=null}** | **5** | **0** |

**配置参数**:
- `ignore-delete`: 是否忽略 DELETE/UPDATE_BEFORE 记录 (默认 false)

### 3.2 PartialUpdate 部分更新引擎

**源码路径**: `paimon-core/.../mergetree/compact/PartialUpdateMergeFunction.java`

**核心逻辑**: 对每个字段取最新的非 null 值，适用于多源宽表拼接场景。

**关键字段**:
```java
private final InternalRow.FieldGetter[] getters;         // 字段提取器
private final boolean ignoreDelete;                       // 是否忽略删除
private final List<WrapperWithFieldIndex<FieldsComparator>> fieldSeqComparators; // Sequence Group 比较器
private final List<WrapperWithFieldIndex<FieldAggregator>> fieldAggregators;     // 字段聚合器
private final boolean removeRecordOnDelete;               // 是否在删除时移除整行
private final Set<Integer> sequenceGroupPartialDelete;    // 基于 Sequence Group 的部分删除
```

**基本更新逻辑 (无 Sequence Group)**:
```java
private void updateNonNullFields(KeyValue kv) {
    for (int i = 0; i < getters.length; i++) {
        Object field = getters[i].getFieldOrNull(kv.value());
        if (field != null) {
            row.setField(i, field);  // 非 null 字段覆盖旧值
        }
    }
}
```

**输入输出示例 (基本模式)**:

| 操作 | Key | name | age | city |
|------|-----|------|-----|------|
| add(源1) | pk=1 | "Alice" | null | null |
| add(源2) | pk=1 | null | 25 | null |
| add(源3) | pk=1 | null | null | "Beijing" |
| **getResult** | pk=1 | **"Alice"** | **25** | **"Beijing"** |

**删除处理的三种策略**:

1. **`ignore-delete=true`**: 忽略所有回撤记录
2. **`partial-update.remove-record-on-delete=true`**: 收到 DELETE 时删除整行
3. **`sequence-group` + `partial-update.remove-record-on-sequence-group`**: 基于 Sequence Group 的精细化删除

**为什么提供三种删除策略?** 不同业务场景对删除语义的需求不同。CDC 场景通常需要精确处理删除；维表拼接场景可以忽略删除；而多源独立更新场景则需要按 Sequence Group 粒度控制删除行为。

**好处**: 灵活的删除策略使 PartialUpdate 能适应从简单到复杂的各种业务场景。

### 3.3 Aggregation 聚合引擎

**源码路径**: `paimon-core/.../mergetree/compact/aggregate/AggregateMergeFunction.java`

**核心逻辑**: 每个字段使用独立的聚合函数进行预聚合。

```java
public class AggregateMergeFunction implements MergeFunction<KeyValue> {
    private final FieldAggregator[] aggregators;

    @Override
    public void add(KeyValue kv) {
        boolean isRetract = kv.valueKind().isRetract();
        for (int i = 0; i < getters.length; i++) {
            Object accumulator = getters[i].getFieldOrNull(row);
            Object inputField = getters[i].getFieldOrNull(kv.value());
            Object mergedField = isRetract
                    ? fieldAggregator.retract(accumulator, inputField)
                    : fieldAggregator.agg(accumulator, inputField);
            row.setField(i, mergedField);
        }
    }
}
```

**字段聚合函数选择逻辑** (源码路径: `AggregateMergeFunction.getAggFuncName()`):

```
1. sequence 字段 --> last_value (仅覆盖，不聚合)
2. 主键字段 --> primary-key (直接取最新值)
3. 用户通过 fields.${name}.aggregate-function 指定 --> 用户配置
4. fields.default-aggregate-function 指定 --> 默认聚合
5. 以上都未配置 --> last_non_null_value (最终默认)
```

**为什么默认是 `last_non_null_value` 而不是 `last_value`?** 因为 `last_non_null_value` 在聚合引擎中更安全，可以避免 null 值意外覆盖有效的聚合结果。

**输入输出示例**:

假设配置: `fields.amount.aggregate-function = sum`, `fields.count.aggregate-function = sum`

| 操作 | Key | amount (sum) | count (sum) | name (last_non_null_value) |
|------|-----|-------------|-------------|---------------------------|
| add(INSERT) | pk=1 | 100 | 1 | "order_A" |
| add(INSERT) | pk=1 | 200 | 1 | null |
| add(INSERT) | pk=1 | 50 | 1 | "order_C" |
| **getResult** | pk=1 | **350** | **3** | **"order_C"** |

**删除处理**: Aggregation 引擎同样支持 `aggregation.remove-record-on-delete` 配置。当收到 DELETE 记录时可以选择删除整行。

### 3.4 FirstRow 首行引擎

**源码路径**: `paimon-core/.../mergetree/compact/FirstRowMergeFunction.java`

**核心逻辑**: 仅保留第一条记录，忽略后续的所有更新。

```java
public class FirstRowMergeFunction implements MergeFunction<KeyValue> {
    private KeyValue first;
    public boolean containsHighLevel;  // 重要标记: 是否包含高层记录

    @Override
    public void add(KeyValue kv) {
        if (kv.valueKind().isRetract()) {
            if (ignoreDelete) return;
            else throw new IllegalArgumentException(...);  // 默认不接受删除
        }
        if (first == null) {
            this.first = kv;  // 只保存第一条
        }
        if (kv.level() > 0) {
            containsHighLevel = true;  // 标记是否有高层记录
        }
    }
}
```

**为什么 `requireCopy()` 返回 true?** FirstRow 需要保留第一条记录的引用，而后续的 `add()` 调用不会更新它。由于对象复用机制，如果不拷贝，第一条记录的数据可能被后续记录覆盖。

**为什么需要 `containsHighLevel` 标记?** 这个标记用于 `FirstRowMergeFunctionWrapper` 中的 Changelog 生成: 如果合并中包含高层(已持久化)记录，说明这个 key 已经存在于存储中，不需要生成 INSERT changelog; 只有全是 Level-0 的记录且 lookup 未找到历史记录时，才生成 INSERT changelog。

**好处**: FirstRow 引擎特别适合数据去重场景(如日志去重)，配合 lookup changelog 模式，可以高效地做到"仅在首次出现时触发下游"。

**输入输出示例**:

| 操作 | Key | Value | SequenceNumber | Level |
|------|-----|-------|----------------|-------|
| add | pk=1 | {name="A", age=20} | 1 | 2 |
| add | pk=1 | {name="B", age=25} | 3 | 0 |
| add | pk=1 | {name="C", age=30} | 5 | 0 |
| **getResult** | pk=1 | **{name="A", age=20}** | **1** | **2** |

### 3.5 四种引擎对比总览

| 特性 | Deduplicate | PartialUpdate | Aggregation | FirstRow |
|------|-------------|---------------|-------------|----------|
| **配置值** | `deduplicate` | `partial-update` | `aggregation` | `first-row` |
| **合并语义** | 保留最新一条 | 按字段取最新非 null 值 | 按字段聚合 | 保留第一条 |
| **requireCopy** | false | false | false | true |
| **支持删除** | 原生支持 | 需配置策略 | 需配置策略 | 默认不支持 |
| **支持回撤** | 原生支持 | 部分(需 seq group) | 聚合函数各异 | 默认不支持 |
| **Sequence Group** | 不适用 | 支持 | 不适用 | 不适用 |
| **典型场景** | CDC 数据入湖 | 多源宽表拼接 | 实时指标聚合 | 日志/事件去重 |

---

## 4. MergeEngine 的选择与创建流程

**源码路径**: `paimon-core/.../table/PrimaryKeyTableUtils.java` (第 58 行)

```java
public static MergeFunctionFactory<KeyValue> createMergeFunctionFactory(TableSchema tableSchema) {
    Options conf = Options.fromMap(tableSchema.options());
    CoreOptions options = new CoreOptions(conf);
    CoreOptions.MergeEngine mergeEngine = options.mergeEngine();

    switch (mergeEngine) {
        case DEDUPLICATE: return DeduplicateMergeFunction.factory(conf);
        case PARTIAL_UPDATE: return PartialUpdateMergeFunction.factory(conf, rowType, primaryKeys);
        case AGGREGATE: return AggregateMergeFunction.factory(conf, rowType, primaryKeys);
        case FIRST_ROW: return FirstRowMergeFunction.factory(conf);
        default: throw new UnsupportedOperationException(...);
    }
}
```

**整体创建链路**:

```mermaid
sequenceDiagram
    participant Table as PrimaryKeyFileStoreTable
    participant Utils as PrimaryKeyTableUtils
    participant Factory as MergeFunctionFactory
    participant LookupWrap as LookupMergeFunction
    participant Store as KeyValueFileStore

    Table->>Utils: createMergeFunctionFactory(tableSchema)
    Utils->>Utils: 读取 merge-engine 配置
    Utils->>Factory: 创建对应的 Factory

    alt needLookup() == true
        Table->>LookupWrap: LookupMergeFunction.wrap(factory, ...)
        Note over LookupWrap: 除 FirstRow 外都包装
    end

    Table->>Store: new KeyValueFileStore(..., mfFactory, ...)
```

**源码路径**: `paimon-core/.../table/PrimaryKeyFileStoreTable.java` (第 82-101 行)

```java
MergeFunctionFactory<KeyValue> mfFactory = PrimaryKeyTableUtils.createMergeFunctionFactory(tableSchema);
if (options.needLookup()) {
    mfFactory = LookupMergeFunction.wrap(mfFactory, options, keyType, rowType);
}
lazyStore = new KeyValueFileStore(..., mfFactory, ...);
```

**为什么 `needLookup()` 时要额外包装?** Lookup 模式下，Compaction 需要查询历史数据来正确合并。LookupMergeFunction 作为包装器，负责管理候选记录的缓冲和高层记录的查找逻辑。

---

## 5. LookupMergeFunction 包装机制

### 5.1 设计动机

在标准的 LSM-Tree Compaction 中，SortMergeReader 会按 key 有序读取所有参与合并的文件，相同 key 的所有版本都会参与合并。但在 Lookup Compaction 中，只有 Level-0 和少量高层文件参与合并，更高层的历史数据需要通过 lookup 查询获取。

**为什么不简单查询所有层?** 因为全层合并(Full Compaction)成本极高，Lookup Compaction 只合并涉及 Level-0 的部分层，用 lookup 补齐缺失的历史记录，兼顾了性能和正确性。

### 5.2 核心实现分析

**源码路径**: `paimon-core/.../mergetree/compact/LookupMergeFunction.java`

```java
public class LookupMergeFunction implements MergeFunction<KeyValue> {
    private final MergeFunction<KeyValue> mergeFunction;  // 被包装的实际合并函数
    private final KeyValueBuffer candidates;               // 候选记录缓冲区
    private boolean containLevel0;

    @Override
    public void add(KeyValue kv) {
        currentKey = kv.key();
        if (kv.level() == 0) {
            containLevel0 = true;
        }
        candidates.put(kv);  // 所有记录先缓冲
    }

    public KeyValue pickHighLevel() {
        // 从候选中找到最低高层(level > 0)的记录
        // 因为最低高层的记录是最近一次合并的结果
    }

    @Override
    public KeyValue getResult() {
        mergeFunction.reset();
        KeyValue highLevel = pickHighLevel();
        // 只将 level-0 记录和选中的高层记录传给实际 MergeFunction
        try (CloseableIterator<KeyValue> iterator = candidates.iterator()) {
            while (iterator.hasNext()) {
                KeyValue kv = iterator.next();
                if (kv.level() <= 0 || kv == highLevel) {
                    mergeFunction.add(kv);
                }
            }
        }
        return mergeFunction.getResult();
    }
}
```

**关键设计决策**: `pickHighLevel()` 选择 level 最低的高层记录(而非最高层)。

**为什么?** 因为在 LSM-Tree 中，level 越低的数据越新。高层记录代表的是之前 compaction 的合并结果，level 最低的高层记录是最近一次合并的结果，是最新的"已确认"状态。

**特殊处理 FirstRow**: LookupMergeFunction.wrap() 中明确排除了 FirstRow:
```java
public static MergeFunctionFactory<KeyValue> wrap(...) {
    if (wrapped.create() instanceof FirstRowMergeFunction) {
        return wrapped;  // FirstRow 不需要包装
    }
    return new Factory(wrapped, options, keyType, valueType);
}
```

**为什么 FirstRow 不需要 Lookup 包装?** FirstRow 只保留第一条记录，不需要与历史记录合并。它有自己专门的 `FirstRowMergeFunctionWrapper` 来处理 changelog 生成。

**KeyValueBuffer 缓冲机制**: LookupMergeFunction 使用 `KeyValueBuffer` (HybridBuffer) 缓冲候选记录。当记录数量少时使用内存 List，超过阈值时自动切换到基于磁盘的 Binary 缓冲区，避免 OOM。

### 5.3 LookupStrategy 策略

**源码路径**: `paimon-api/.../lookup/LookupStrategy.java`

```java
public class LookupStrategy {
    public final boolean needLookup;
    public final boolean isFirstRow;
    public final boolean produceChangelog;
    public final boolean deletionVector;

    // needLookup = produceChangelog || deletionVector || isFirstRow || forceLookup
}
```

触发 Lookup 的四种条件:
1. **`changelog-producer = lookup`**: 需要通过 lookup 生成 changelog
2. **`deletion-vectors.enabled = true`**: 需要通过 lookup 维护删除向量
3. **`merge-engine = first-row`**: FirstRow 引擎需要 lookup 判断 key 是否已存在
4. **`force-lookup = true`**: 强制开启 lookup

**创建代码** (源码路径: `CoreOptions.java` 第 3112 行):
```java
public LookupStrategy lookupStrategy() {
    return LookupStrategy.from(
            mergeEngine().equals(MergeEngine.FIRST_ROW),
            changelogProducer().equals(ChangelogProducer.LOOKUP),
            deletionVectorsEnabled(),
            options.get(FORCE_LOOKUP));
}
```

---

## 6. 内置聚合函数完整列表

### 6.1 FieldAggregator 抽象基类

**源码路径**: `paimon-core/.../mergetree/compact/aggregate/FieldAggregator.java`

```java
public abstract class FieldAggregator implements Serializable {
    protected final DataType fieldType;  // 字段数据类型
    protected final String name;         // 聚合函数标识名

    // 正向聚合: 合并 accumulator 和 inputField
    public abstract Object agg(Object accumulator, Object inputField);

    // 反向聚合: 用于 PartialUpdate 中旧序列号的处理
    public Object aggReversed(Object accumulator, Object inputField) {
        return agg(inputField, accumulator);  // 默认交换参数顺序
    }

    // 重置状态 (有状态聚合函数需要重写)
    public void reset() {}

    // 回撤: 从 accumulator 中撤回 retractField 的贡献
    public Object retract(Object accumulator, Object retractField) {
        throw new UnsupportedOperationException(...);  // 默认不支持
    }
}
```

**为什么需要 `aggReversed`?** 在 PartialUpdate + Sequence Group 场景中，当新记录的序列号小于旧记录时，需要"反向"聚合: 将新值作为 accumulator，旧值作为 input。对于 SUM 这类满足交换律的函数无影响，但对于 `collect` 等非交换函数则有区别。`FieldCollectAgg` 覆写了 `aggReversed` 使其直接调用 `agg`，因为 accumulator 已经去重过，跳过重新去重可以提升性能。

### 6.2 所有内置聚合函数详解

以下是 Paimon 内置的 **22** 种聚合函数完整列表:

| # | 标识名 | 实现类 | 数据类型约束 | 支持回撤 | 说明 |
|---|--------|--------|-------------|---------|------|
| 1 | `sum` | `FieldSumAgg` | 数值类型(Numeric) | 是(减法) | 累加求和 |
| 2 | `product` | `FieldProductAgg` | 数值类型(Numeric) | 是(除法) | 累乘求积 |
| 3 | `max` | `FieldMaxAgg` | 可比较类型 | 否 | 取最大值 |
| 4 | `min` | `FieldMinAgg` | 可比较类型 | 否 | 取最小值 |
| 5 | `last_value` | `FieldLastValueAgg` | 任意类型 | 是(置null) | 取最后一个值(含null) |
| 6 | `last_non_null_value` | `FieldLastNonNullValueAgg` | 任意类型 | 是(条件置null) | 取最后一个非null值 |
| 7 | `first_value` | `FieldFirstValueAgg` | 任意类型 | 否 | 取第一个值(有状态) |
| 8 | `first_non_null_value` | `FieldFirstNonNullValueAgg` | 任意类型 | 否 | 取第一个非null值(有状态) |
| 9 | `first_not_null_value` | (Legacy) `FieldFirstNonNullValueAgg` | 任意类型 | 否 | `first_non_null_value` 的旧名兼容 |
| 10 | `listagg` | `FieldListaggAgg` | `VARCHAR(MAX)` | 否 | 字符串拼接(可配分隔符/去重) |
| 11 | `bool_and` | `FieldBoolAndAgg` | `BOOLEAN` | 否 | 布尔与 |
| 12 | `bool_or` | `FieldBoolOrAgg` | `BOOLEAN` | 否 | 布尔或 |
| 13 | `collect` | `FieldCollectAgg` | `ARRAY<T>` | 是(移除元素) | 收集元素到数组(可去重) |
| 14 | `merge_map` | `FieldMergeMapAgg` | `MAP<K,V>` | 是(按key移除) | 合并两个Map |
| 15 | `merge_map_with_keytime` | `FieldMergeMapWithKeyTimeAgg` | `MAP<K,ROW>` | 否 | 按时间戳字段合并Map |
| 16 | `nested_update` | `FieldNestedUpdateAgg` | `ARRAY<ROW>` | 是(按key/全匹配) | 嵌套表更新 |
| 17 | `nested_partial_update` | `FieldNestedPartialUpdateAgg` | `ARRAY<ROW>` | 否 | 嵌套表部分更新 |
| 18 | `rbm32` | `FieldRoaringBitmap32Agg` | `VARBINARY` | 否 | 32位RoaringBitmap OR合并 |
| 19 | `rbm64` | `FieldRoaringBitmap64Agg` | `VARBINARY` | 否 | 64位RoaringBitmap OR合并 |
| 20 | `hll_sketch` | `FieldHllSketchAgg` | `VARBINARY` | 否 | HyperLogLog Sketch Union |
| 21 | `theta_sketch` | `FieldThetaSketchAgg` | `VARBINARY` | 否 | Theta Sketch Union |
| 22 | `primary-key` | `FieldPrimaryKeyAgg` | 任意类型 | 是(直接取新值) | 主键字段专用(总取最新值) |

**各函数详细分析**:

#### sum (FieldSumAgg)

**源码路径**: `paimon-core/.../aggregate/FieldSumAgg.java`

支持的类型: `DECIMAL`, `TINYINT`, `SMALLINT`, `INTEGER`, `BIGINT`, `FLOAT`, `DOUBLE`

```java
// 聚合: accumulator + inputField
public Object agg(Object accumulator, Object inputField) {
    if (accumulator == null || inputField == null) {
        return accumulator == null ? inputField : accumulator;  // null 安全处理
    }
    // 按类型分别处理加法...
}

// 回撤: accumulator - inputField
public Object retract(Object accumulator, Object inputField) {
    if (accumulator == null || inputField == null) {
        return accumulator == null ? negative(inputField) : accumulator;
    }
    // 按类型分别处理减法...
}
```

**为什么 null 处理逻辑是"返回非 null 的那个"?** 这遵循 SQL 的 NULL 语义: NULL 不参与聚合。当 accumulator 为 null(初始状态)时，结果就是第一个非 null 的输入值。

**回撤时 `negative()` 的作用**: 当 accumulator 为 null 但 inputField 不为 null 时，回撤的语义是"减去一个值"，此时结果应为 inputField 的负值。

#### product (FieldProductAgg)

**源码路径**: `paimon-core/.../aggregate/FieldProductAgg.java`

累乘聚合，回撤通过除法实现。支持与 sum 相同的数值类型。

**注意**: 除法回撤在 inputField 为 0 时会抛出算术异常(除零错误)。这是该聚合函数的已知限制。

#### last_value / last_non_null_value

**源码路径**: `paimon-core/.../aggregate/FieldLastValueAgg.java`, `FieldLastNonNullValueAgg.java`

```java
// FieldLastValueAgg
public Object agg(Object accumulator, Object inputField) { return inputField; }
public Object retract(Object accumulator, Object retractField) { return null; }

// FieldLastNonNullValueAgg
public Object agg(Object accumulator, Object inputField) {
    return (inputField == null) ? accumulator : inputField;
}
public Object retract(Object accumulator, Object retractField) {
    return retractField != null ? null : accumulator;
}
```

**两者的关键区别**: `last_value` 会让 null 覆盖有效值，而 `last_non_null_value` 忽略 null 输入。

#### first_value / first_non_null_value

**源码路径**: `paimon-core/.../aggregate/FieldFirstValueAgg.java`, `FieldFirstNonNullValueAgg.java`

这两个聚合器是**有状态**的，内部维护 `boolean initialized` 标记:

```java
// FieldFirstValueAgg
public Object agg(Object accumulator, Object inputField) {
    if (!initialized) {
        initialized = true;
        return inputField;     // 第一次调用: 返回 inputField
    } else {
        return accumulator;    // 后续调用: 忽略 inputField
    }
}

public void reset() { this.initialized = false; }
```

**为什么需要 `reset()` 方法?** 有状态聚合器在处理不同 key 时需要重置状态。MergeFunction 的 `reset()` 会触发所有 aggregator 的 `reset()`。

#### collect (FieldCollectAgg)

**源码路径**: `paimon-core/.../aggregate/FieldCollectAgg.java`

支持 `distinct` 模式。对于复合类型(如 ROW)使用 `RecordEqualiser`(代码生成) 进行等值比较:

```java
public Object agg(Object accumulator, Object inputField) {
    if (equaliser != null) {
        List<Object> collection = new ArrayList<>();
        collect(collection, accumulator);           // accumulator 已去重，直接添加
        collectWithEqualiser(collection, inputField); // 输入需要与已有元素去重比较
        return new GenericArray(collection.toArray());
    } else {
        Collection<Object> collection = distinct ? new HashSet<>() : new ArrayList<>();
        collect(collection, accumulator);
        collect(collection, inputField);
        return new GenericArray(collection.toArray());
    }
}
```

支持回撤: 遍历 retract 数组，从 accumulator 中移除匹配的元素。

#### merge_map (FieldMergeMapAgg)

**源码路径**: `paimon-core/.../aggregate/FieldMergeMapAgg.java`

合并两个 Map，相同 key 时后者覆盖前者。支持回撤(按 key 移除)。

#### merge_map_with_keytime (FieldMergeMapWithKeyTimeAgg)

**源码路径**: `paimon-core/.../aggregate/FieldMergeMapWithKeyTimeAgg.java`

在 merge_map 基础上增加了时间戳比较: Map 的 value 类型必须是 ROW，其中指定的字段作为时间戳，相同 key 时只保留时间戳更大的值。如果 value 为 null 则删除该 key。

#### nested_update (FieldNestedUpdateAgg)

**源码路径**: `paimon-core/.../aggregate/FieldNestedUpdateAgg.java`

用于嵌套表(`ARRAY<ROW>`)场景。可配置:
- `nested-key`: 指定嵌套表的主键列，按 key 更新行
- `count-limit`: 限制数组最大长度，防止 OOM

#### nested_partial_update (FieldNestedPartialUpdateAgg)

**源码路径**: `paimon-core/.../aggregate/FieldNestedPartialUpdateAgg.java`

与 `nested_update` 类似但对相同 key 的行执行**部分更新**(非 null 字段覆盖):

```java
private void partialUpdate(GenericRow toUpdate, InternalRow input) {
    for (int i = 0; i < fieldGetters.length; i++) {
        Object field = fieldGetter.getFieldOrNull(input);
        if (field != null) {
            toUpdate.setField(i, field);  // 非 null 覆盖
        }
    }
}
```

#### rbm32 / rbm64 (RoaringBitmap)

**源码路径**: `paimon-core/.../aggregate/FieldRoaringBitmap32Agg.java`, `FieldRoaringBitmap64Agg.java`

对 RoaringBitmap 的序列化字节数组执行 OR 合并。内部复用 bitmap 对象避免频繁创建:
```java
roaringBitmapAcc.deserialize(ByteBuffer.wrap((byte[]) accumulator));
roaringBitmapInput.deserialize(ByteBuffer.wrap((byte[]) inputField));
roaringBitmapAcc.or(roaringBitmapInput);
return roaringBitmapAcc.serialize();
```

#### hll_sketch / theta_sketch (近似去重)

**源码路径**: `paimon-core/.../aggregate/FieldHllSketchAgg.java`, `FieldThetaSketchAgg.java`

调用 `HllSketchUtil.union()` / `ThetaSketch.union()` 合并两个 sketch 的序列化字节数组。用于近似基数(distinct count)估算。

### 6.3 FieldIgnoreRetractAgg 装饰器

**源码路径**: `paimon-core/.../aggregate/FieldIgnoreRetractAgg.java`

```java
public class FieldIgnoreRetractAgg extends FieldAggregator {
    private final FieldAggregator aggregator;

    @Override
    public Object agg(Object accumulator, Object inputField) {
        return aggregator.agg(accumulator, inputField);  // 委托给内部聚合器
    }

    @Override
    public Object retract(Object accumulator, Object retractField) {
        return accumulator;  // 忽略回撤，直接返回 accumulator
    }
}
```

**为什么需要装饰器模式?** 不是所有聚合函数都支持回撤(如 max/min 无法精确回撤)。通过装饰器，用户可以为任何聚合函数配置 `fields.${field_name}.ignore-retract=true` 来跳过回撤处理，而无需修改聚合函数本身。

**好处**: 正交组合，任何聚合函数与是否忽略回撤，两个维度可以自由组合。

---

## 7. FieldAggregator SPI 注册机制

### 7.1 Factory 接口设计

**源码路径**: `paimon-core/.../aggregate/factory/FieldAggregatorFactory.java`

```java
public interface FieldAggregatorFactory extends Factory {
    FieldAggregator create(DataType fieldType, CoreOptions options, String field);
    String identifier();  // 继承自 Factory 接口
}
```

每个具体工厂通过 `identifier()` 方法返回唯一标识名(如 "sum", "max"), 通过 `create()` 方法创建对应的聚合器实例。工厂的 `create()` 方法接收三个参数:
- `fieldType`: 字段数据类型，工厂内部进行类型校验
- `options`: 全局配置，用于获取字段级配置(如 listagg 的分隔符)
- `field`: 字段名，用于读取字段级配置

### 7.2 SPI 服务文件注册

**源码路径**: `paimon-core/src/main/resources/META-INF/services/org.apache.paimon.factories.Factory`

所有 22 个工厂类通过 Java SPI 机制注册(与 CatalogFactory 等共享同一个 Factory 服务文件):

```
org.apache.paimon.mergetree.compact.aggregate.factory.FieldBoolAndAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldBoolOrAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldCollectAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldFirstNonNullValueAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldFirstNonNullValueAggLegacyFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldFirstValueAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldHllSketchAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldLastNonNullValueAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldLastValueAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldListaggAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldMaxAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldMergeMapAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldMergeMapWithKeyTimeAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldMinAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldNestedUpdateAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldNestedPartialUpdateAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldPrimaryKeyAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldProductAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldRoaringBitmap32AggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldRoaringBitmap64AggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldSumAggFactory
org.apache.paimon.mergetree.compact.aggregate.factory.FieldThetaSketchAggFactory
```

### 7.3 工厂发现与创建流程

**源码路径**: `FieldAggregatorFactory.create()` 静态方法 (第 39 行)

```mermaid
sequenceDiagram
    participant Caller as AggregateMergeFunction / PartialUpdate
    participant FacStatic as FieldAggregatorFactory (static)
    participant FactoryUtil as FactoryUtil.discoverFactory
    participant SPI as ServiceLoader
    participant FacImpl as 具体 Factory 实现

    Caller->>FacStatic: create(fieldType, fieldName, aggFuncName, options)
    FacStatic->>FactoryUtil: discoverFactory(classLoader, FieldAggregatorFactory.class, aggFuncName)
    FactoryUtil->>SPI: ServiceLoader.load(Factory.class)
    SPI-->>FactoryUtil: 返回所有注册的 Factory
    FactoryUtil->>FactoryUtil: 按 identifier() 匹配 aggFuncName
    FactoryUtil-->>FacStatic: 返回匹配的 FieldAggregatorFactory
    FacStatic->>FacImpl: factory.create(fieldType, options, fieldName)
    FacImpl-->>FacStatic: 返回 FieldAggregator 实例

    alt fieldIgnoreRetract == true
        FacStatic->>FacStatic: new FieldIgnoreRetractAgg(aggregator)
    end

    FacStatic-->>Caller: 返回最终 FieldAggregator
```

完整的静态 `create()` 方法:

```java
static FieldAggregator create(DataType fieldType, String fieldName, String aggFuncName, CoreOptions options) {
    FieldAggregatorFactory factory = FactoryUtil.discoverFactory(
            FieldAggregator.class.getClassLoader(), FieldAggregatorFactory.class, aggFuncName);
    if (factory == null) {
        throw new RuntimeException("Use unsupported aggregation: " + aggFuncName + " ...");
    }

    boolean removeRecordOnRetract = options.aggregationRemoveRecordOnDelete();
    boolean fieldIgnoreRetract = options.fieldAggIgnoreRetract(fieldName);
    // 两个选项不能同时启用
    Preconditions.checkState(!(removeRecordOnRetract && fieldIgnoreRetract), ...);

    FieldAggregator aggregator = factory.create(fieldType, options, fieldName);
    return fieldIgnoreRetract ? new FieldIgnoreRetractAgg(aggregator) : aggregator;
}
```

**为什么使用 SPI 而不是 if-else 或 Map?** SPI 机制允许用户在不修改 Paimon 源码的情况下，通过引入自定义 JAR 包来扩展聚合函数。只需实现 `FieldAggregatorFactory` 接口并在 META-INF/services 中注册即可。

**好处**: 开放-封闭原则(OCP)，对扩展开放，对修改封闭。

---

## 8. PartialUpdate Sequence Group 详解

### 8.1 设计动机与问题背景

在多源宽表拼接场景中，不同数据源可能以不同的频率和时序更新同一行的不同列。简单的"取最新非 null"策略可能导致数据不一致:

```
时刻1: 源A 发送 {pk=1, name="Alice", age=null,  version_a=3}
时刻2: 源B 发送 {pk=1, name=null,    age=25,    version_b=5}
时刻3: 源A 发送 {pk=1, name="Bob",   age=null,  version_a=2}  <-- 乱序! version_a < 之前的
```

如果不区分序列号，时刻3 的 name="Bob" 会覆盖 name="Alice"，但实际上 version_a=2 < version_a=3，应该保留 "Alice"。

**Sequence Group 的解决方案**: 将列分组，每组有独立的序列字段，只有当组内序列字段更大时才更新该组的列。

### 8.2 配置方式与解析流程

配置格式:
```sql
-- fields.${sequence_fields}.sequence-group = ${field1},${field2},...
'fields.version_a.sequence-group' = 'name,address'
'fields.version_b.sequence-group' = 'age,salary'
```

**解析流程** (源码路径: `PartialUpdateMergeFunction.Factory` 构造函数, 第 392-492 行):

```mermaid
flowchart TD
    A[遍历 options.toMap] --> B{key 以 fields. 开头且以 sequence-group 结尾?}
    B -->|是| C[解析序列字段名 -> sequenceFields int数组]
    C --> D[解析被保护字段名 -> field int 列表]
    D --> E[为每个被保护字段创建 FieldsComparator]
    E --> F[为序列字段自身也注册 FieldsComparator]
    B -->|否| G[跳过]
    F --> H[检查冲突: 同一字段不能属于多个 Group]
    H --> I[为聚合字段创建 FieldAggregator]
    I --> J[校验: 除 last_non_null_value 外,聚合函数必须在 sequence group 中]
```

**关键校验逻辑**:
1. 同一字段不能同时属于多个 Sequence Group (第 428 行)
2. `removeRecordOnDelete` 和 `ignoreDelete` 不能同时启用 (第 454 行)
3. `removeRecordOnDelete` 和 `sequenceGroup` 不能同时启用 (第 473 行)
4. `partial-update.remove-record-on-sequence-group` 中指定的字段必须是序列字段 (第 482 行)
5. 使用聚合函数的字段必须在 Sequence Group 保护下(除了 `last_non_null_value`)

### 8.3 更新逻辑深度分析

**源码路径**: `PartialUpdateMergeFunction.updateWithSequenceGroup()` (第 190 行)

核心算法按字段索引从小到大遍历:

```java
for (int i = 0; i < getters.length; i++) {
    // 1. 获取当前字段的 seqComparator 和 aggregator
    FieldsComparator seqComparator = ...; // 按 fieldIndex 排序的迭代器取出
    FieldAggregator aggregator = ...;

    if (seqComparator == null) {
        // 无序列组保护的字段: 非null覆盖 或 聚合
        if (aggregator != null) {
            row.setField(i, aggregator.agg(accumulator, field));
        } else if (field != null) {
            row.setField(i, field);
        }
    } else {
        // 有序列组保护的字段
        if (isEmptySequenceGroup(kv, seqComparator, isEmptySequenceGroup)) {
            continue;  // 序列字段全为null,跳过整个组
        }
        if (seqComparator.compare(kv.value(), row) >= 0) {
            // 新记录序列号 >= 当前值: 更新
            if (是序列字段本身) {
                // 序列字段全部更新
                for (int fieldIndex : seqComparator.compareFields()) {
                    row.setField(fieldIndex, getters[fieldIndex].getFieldOrNull(kv.value()));
                }
            } else {
                // 普通受保护字段
                row.setField(i, aggregator == null ? field : aggregator.agg(accumulator, field));
            }
        } else if (aggregator != null) {
            // 新记录序列号 < 当前值但有聚合器: 反向聚合
            row.setField(i, aggregator.aggReversed(accumulator, field));
        }
    }
}
```

**为什么需要 `isEmptySequenceGroup` 检查?** 当序列字段全为 null 时，说明这条记录不属于该 Sequence Group 的源系统(该源系统没有发送数据)，应该跳过整个组的更新，避免用 null 覆盖有效数据。

**为什么序列号小时还要调用 `aggReversed`?** 对于聚合字段(如 SUM)，即使序列号更小，聚合值仍然应该被累加。`aggReversed` 以反向参数顺序调用聚合函数，确保对于非交换聚合函数(如 `collect`)也能正确处理顺序。

**`isEmptySequenceGroup` 的缓存优化**: 使用 `boolean[] isEmptySequenceGroup` 数组避免对同一个序列组重复检查:
```java
private boolean isEmptySequenceGroup(KeyValue kv, FieldsComparator comparator, boolean[] isEmptySequenceGroup) {
    if (isEmptySequenceGroup[comparator.compareFields()[0]]) return true; // 已判定过
    for (int fieldIndex : comparator.compareFields()) {
        if (getters[fieldIndex].getFieldOrNull(kv.value()) != null) return false; // 有非null序列字段
    }
    for (int fieldIndex : comparator.compareFields()) {
        isEmptySequenceGroup[fieldIndex] = true; // 标记所有序列字段
    }
    return true;
}
```

### 8.4 回撤逻辑 retractWithSequenceGroup

**源码路径**: `PartialUpdateMergeFunction.retractWithSequenceGroup()` (第 271 行)

回撤在 Sequence Group 下的行为:

```java
if (seqComparator.compare(kv.value(), row) >= 0) {
    if (是序列字段) {
        if (kv.valueKind() == RowKind.DELETE && sequenceGroupPartialDelete.contains(field)) {
            // 触发基于 Sequence Group 的整行删除
            currentDeleteRow = true;
            row = new GenericRow(getters.length);
            initRow(row, kv.value());
            return;
        }
        // 更新序列字段
    } else {
        if (aggregator == null) {
            row.setField(i, null);  // 普通字段置null
        } else {
            // 聚合字段回撤
            row.setField(i, aggregator.retract(accumulator, inputField));
        }
    }
} else if (aggregator != null) {
    // 序列号较小但有聚合器时也回撤
    row.setField(i, aggregator.retract(accumulator, inputField));
}
```

**`sequenceGroupPartialDelete` 的作用**: 当配置了 `partial-update.remove-record-on-sequence-group` 时，指定哪些 Sequence Group 的 DELETE 消息应触发整行删除。这允许精细控制: 只有特定源系统的删除操作才能删除整行。

### 8.5 Sequence Group 中的聚合函数

**源码路径**: `PartialUpdateMergeFunction.getAggFuncName()` (第 660 行)

```java
public static String getAggFuncName(String fieldName, CoreOptions options,
        List<String> primaryKeys, List<String> sequenceFields,
        List<String> fieldsProtectedBySequenceGroup) {

    if (sequenceFields.contains(fieldName)) return null;     // 序列字段: 无聚合
    if (primaryKeys.contains(fieldName)) return "primary-key"; // 主键: 总取新值
    String aggFuncName = options.fieldAggFunc(fieldName);       // 用户配置
    if (aggFuncName == null) aggFuncName = options.fieldsDefaultFunc(); // 默认聚合

    if (aggFuncName != null) {
        // 聚合函数必须在 Sequence Group 保护下(除 last_non_null_value 外)
        checkArgument(
            aggFuncName.equals("last_non_null_value")
                || fieldsProtectedBySequenceGroup.contains(fieldName),
            "Must use sequence group for aggregation functions...");
    }
    return aggFuncName;
}
```

**为什么 `last_non_null_value` 例外?** `last_non_null_value` 本质上就是 PartialUpdate 的默认行为(取最新非 null 值)，不依赖序列号排序，因此不需要 Sequence Group 保护。

**典型配置示例**:
```sql
CREATE TABLE wide_table (
    pk INT PRIMARY KEY NOT ENFORCED,
    -- 源A 的列
    version_a INT,
    name STRING,
    address STRING,
    -- 源B 的列
    version_b INT,
    age INT,
    total_amount DECIMAL(10,2)
) WITH (
    'merge-engine' = 'partial-update',
    'fields.version_a.sequence-group' = 'name,address',
    'fields.version_b.sequence-group' = 'age,total_amount',
    'fields.total_amount.aggregate-function' = 'sum'
);
```

在这个配置中:
- `name`, `address` 受 `version_a` 保护，只有 version_a 更大时才更新
- `age`, `total_amount` 受 `version_b` 保护
- `total_amount` 在 version_b 保护下还使用 SUM 聚合

---

## 9. Changelog 四种模式

**源码路径**: `paimon-api/.../CoreOptions.java` (第 3948 行)

```java
public enum ChangelogProducer {
    NONE("none", "No changelog file."),
    INPUT("input", "Double write to a changelog file when flushing memory table."),
    FULL_COMPACTION("full-compaction", "Generate changelog files with each full compaction."),
    LOOKUP("lookup", "Generate changelog files through 'lookup' compaction.");
}
```

### 9.1 NONE 模式

**行为**: 不产生任何 changelog 文件。流式读取只能消费最新快照的差异(delta 文件)。

**为什么是默认模式?** 产生 changelog 有额外的存储和计算开销。对于只需要批量查询或不需要精确 CDC 语义的场景，NONE 模式最经济。

**适用场景**: 纯批处理查询、数据仓库分层。

### 9.2 INPUT 模式

**行为**: 在刷写内存表(MemTable flush)时，将输入的 changelog 记录直接双写到 changelog 文件。

**源码关系**: 由 `MergeTreeWriter` 在 flush 时直接写入，不涉及 Compaction 阶段的 MergeFunction。

**为什么不推荐用于非 Deduplicate 引擎?** INPUT 模式记录的是原始输入，而非合并后的结果。对于 PartialUpdate/Aggregation，原始输入不等于最终状态变更，下游消费者看到的 changelog 可能不准确。

**适用场景**: Deduplicate 引擎 + 精确 CDC 语义(如从 MySQL binlog 直接入湖)。

### 9.3 FULL_COMPACTION 模式

**行为**: 仅在 Full Compaction(合并到最高层)时产生 changelog。

**源码路径**: `paimon-core/.../compact/FullChangelogMergeTreeCompactRewriter.java`

判断条件:
```java
@Override
protected boolean rewriteChangelog(int outputLevel, boolean dropDelete, ...) {
    boolean changelog = outputLevel == maxLevel;  // 仅当输出到最高层
    if (changelog) {
        Preconditions.checkArgument(dropDelete, ...);  // Full Compaction 必须 drop delete
    }
    return changelog;
}
```

使用 `FullChangelogMergeFunctionWrapper` 生成 changelog:
```java
@Override
protected MergeFunctionWrapper<ChangelogResult> createMergeWrapper(int outputLevel) {
    return new FullChangelogMergeFunctionWrapper(mfFactory.create(), maxLevel, valueEqualiser);
}
```

**为什么 Full Compaction 模式的 changelog 有延迟?** changelog 只在 Full Compaction 时产生，而 Full Compaction 的触发频率远低于增量 Compaction，因此 changelog 的产生有较大延迟。

**好处**: 比 NONE 多了准确的 changelog，但计算开销远小于 LOOKUP。是准确性和性能的折中。

**适用场景**: 对延迟不敏感的流式消费场景，如准实时报表。

### 9.4 LOOKUP 模式

**行为**: 在涉及 Level-0 文件的 Compaction 中，通过查找(lookup)历史值来生成精确的 changelog。

**源码路径**: `paimon-core/.../compact/LookupMergeTreeCompactRewriter.java`

判断条件:
```java
@Override
protected boolean rewriteChangelog(int outputLevel, boolean dropDelete, ...) {
    return rewriteLookupChangelog(outputLevel, sections);
    // 即: outputLevel > 0 且 sections 中包含 Level-0 文件
}
```

changelog 生成逻辑由 `LookupChangelogMergeFunctionWrapper` 实现:

```mermaid
flowchart TD
    A[获取合并候选记录] --> B{包含 Level-0 记录?}
    B -->|否| C[无 changelog]
    B -->|是| D{有高层记录?}
    D -->|是| E[高层记录作为 BEFORE]
    D -->|否| F[lookup 查询更高层]
    F --> G{lookup 找到?}
    G -->|是| H[lookup 结果作为 BEFORE]
    G -->|否| I[无 BEFORE]
    E --> J[合并计算 AFTER]
    H --> J
    I --> J
    J --> K{BEFORE == null 且 AFTER.isAdd?}
    K -->|是| L[生成 INSERT changelog]
    K -->|否| M{BEFORE.isAdd 且 !AFTER.isAdd?}
    M -->|是| N[生成 DELETE changelog]
    M -->|否| O{值不等?}
    O -->|是| P[生成 UPDATE_BEFORE + UPDATE_AFTER]
    O -->|否| Q[无 changelog 值未变]
```

**为什么 Lookup 模式能产生即时 changelog?** 因为每次涉及 Level-0 的 Compaction 都会触发 changelog 生成，而 Level-0 文件是新写入的数据，Compaction 频率远高于 Full Compaction。

**好处**: 低延迟、高准确性的 changelog，是 Paimon 在流式场景中推荐的模式。

**代价**: 需要维护 lookup 索引(如 hash index 文件或 RocksDB)，有额外的内存和 I/O 开销。

**与 DeletionVector 的协同**: 当同时启用 DV 和 lookup changelog 时，lookup 到旧记录后除了用于 changelog 对比，还会通过 `deletionVectorsMaintainer.notifyNewDeletion()` 标记旧记录的物理位置为已删除。

### 9.5 四种模式对比

| 特性 | NONE | INPUT | FULL_COMPACTION | LOOKUP |
|------|------|-------|-----------------|--------|
| **Changelog 来源** | 无 | 原始输入双写 | Full Compaction 时计算 | 每次 L0 Compaction lookup 计算 |
| **延迟** | N/A | 最低(flush时) | 高(依赖全量合并) | 低(增量合并) |
| **准确性** | N/A | 仅 Deduplicate 准确 | 所有引擎准确 | 所有引擎准确 |
| **额外开销** | 无 | 双写存储 | Full Compaction 计算 | Lookup 索引内存/IO |
| **适合引擎** | 所有 | Deduplicate | 所有 | 所有(推荐) |
| **配置值** | `none` | `input` | `full-compaction` | `lookup` |
| **row-deduplicate** | N/A | N/A | 支持 | 支持 |

**changelog-producer.row-deduplicate**: 当设置为 true 时，如果合并前后值相同，则不生成 UPDATE_BEFORE/UPDATE_AFTER changelog。通过 `RecordEqualiser` 比较值的相等性。可配合 `changelog-producer.row-deduplicate-ignore-fields` 忽略某些字段的比较。

---

## 10. MergeFunctionWrapper 体系

### 10.1 ReducerMergeFunctionWrapper

**源码路径**: `paimon-core/.../compact/ReducerMergeFunctionWrapper.java`

**设计目的**: 优化只有一条记录时的合并性能。

```java
public class ReducerMergeFunctionWrapper implements MergeFunctionWrapper<KeyValue> {
    private KeyValue initialKv;
    private boolean isInitialized;

    @Override
    public void add(KeyValue kv) {
        if (initialKv == null) {
            initialKv = kv;  // 第一条: 暂存
        } else {
            if (!isInitialized) {
                merge(initialKv);  // 第二条到来时才初始化合并
                isInitialized = true;
            }
            merge(kv);
        }
    }

    @Override
    public KeyValue getResult() {
        return isInitialized ? mergeFunction.getResult() : initialKv;
        // 只有一条记录时直接返回，跳过 MergeFunction
    }
}
```

**为什么这个优化重要?** 在 LSM-Tree 中，大部分 key 在单次 Compaction 中只出现一次(来自一个 SortedRun)。跳过 MergeFunction 的 reset/add/getResult 调用链可以显著减少方法调用和对象创建。

**使用场景**: 在 `MergeTreeCompactRewriter.rewriteCompaction()` 中被默认使用(第 87 行):
```java
reader = readerForMergeTree(sections, new ReducerMergeFunctionWrapper(mfFactory.create()));
```

### 10.2 FullChangelogMergeFunctionWrapper

**源码路径**: `paimon-core/.../compact/FullChangelogMergeFunctionWrapper.java`

**设计目的**: 在 Full Compaction 时同时产生合并数据和 changelog。

核心逻辑:
```java
@Override
public ChangelogResult getResult() {
    reusedResult.reset();
    if (isInitialized) {  // 多条记录
        KeyValue merged = mergeFunction.getResult();
        if (topLevelKv == null) {
            // 无最高层历史记录且合并结果是 ADD -> INSERT
            if (merged.isAdd()) reusedResult.addChangelog(replace(reusedAfter, INSERT, merged));
        } else {
            if (!merged.isAdd()) {
                // 有历史 + 合并结果是删除 -> DELETE
                reusedResult.addChangelog(replace(reusedBefore, DELETE, topLevelKv));
            } else if (valueEqualiser == null || !valueEqualiser.equals(topLevelKv.value(), merged.value())) {
                // 有历史 + 值变化 -> UPDATE_BEFORE + UPDATE_AFTER
                reusedResult.addChangelog(replace(reusedBefore, UPDATE_BEFORE, topLevelKv));
                reusedResult.addChangelog(replace(reusedAfter, UPDATE_AFTER, merged));
            }
        }
        return reusedResult.setResultIfNotRetract(merged);
    } else {  // 单条记录
        if (topLevelKv == null && initialKv.isAdd()) {
            reusedResult.addChangelog(replace(reusedAfter, INSERT, initialKv));
        }
        return reusedResult.setResultIfNotRetract(initialKv);
    }
}
```

**`topLevelKv` 的含义**: 来自 maxLevel(最高层)的记录，代表上一次 Full Compaction 的结果。只有该层级的记录被视为"旧值"。

**`setResultIfNotRetract` 的作用**: Full Compaction 时 dropDelete 总是 true，但 MergeFunction 可能返回 DELETE 类型的结果。此方法确保 DELETE 结果不被作为数据输出(只作为 changelog 的参考)。

### 10.3 LookupChangelogMergeFunctionWrapper

**源码路径**: `paimon-core/.../compact/LookupChangelogMergeFunctionWrapper.java`

**设计目的**: 通过 lookup 历史数据来实时生成 changelog。

关键方法:
```java
@Override
public ChangelogResult getResult() {
    // 1. 从 LookupMergeFunction 中获取高层记录
    KeyValue highLevel = mergeFunction.pickHighLevel();
    boolean containLevel0 = mergeFunction.containLevel0();

    // 2. 如果没有高层记录，通过 lookup 函数查询
    if (highLevel == null) {
        T lookupResult = lookup.apply(mergeFunction.key());
        if (lookupResult != null) {
            if (lookupStrategy.deletionVector) {
                // DV 模式: 获取文件名和行位置，标记删除
                deletionVectorsMaintainer.notifyNewDeletion(fileName, rowPosition);
            }
            highLevel = extractKeyValue(lookupResult);
            mergeFunction.insertInto(highLevel, comparator);  // 插入到候选中参与合并
        }
    }

    // 3. 计算合并结果
    KeyValue result = mergeFunction.getResult();

    // 4. 只在有 Level-0 记录且配置产生 changelog 时生成
    reusedResult.reset();
    if (containLevel0 && lookupStrategy.produceChangelog) {
        setChangelog(highLevel, result);
    }
    return reusedResult.setResult(result);
}
```

**`setChangelog` 逻辑**:
```java
private void setChangelog(@Nullable KeyValue before, KeyValue after) {
    if (before == null || !before.isAdd()) {
        if (after.isAdd()) {
            reusedResult.addChangelog(replaceAfter(INSERT, after));     // 新增
        }
    } else {
        if (!after.isAdd()) {
            reusedResult.addChangelog(replaceBefore(DELETE, before));   // 删除
        } else if (valueEqualiser == null || !valueEqualiser.equals(before.value(), after.value())) {
            reusedResult.addChangelog(replaceBefore(UPDATE_BEFORE, before)); // 更新前
            reusedResult.addChangelog(replaceAfter(UPDATE_AFTER, after));    // 更新后
        }
    }
}
```

**序列号比较器的构造**: Lookup 模式中需要按正确的顺序将 lookup 到的记录插入候选列表:
```java
private Comparator<KeyValue> createSequenceComparator(@Nullable FieldsComparator userDefinedSeqComparator) {
    if (userDefinedSeqComparator == null) {
        return Comparator.comparingLong(KeyValue::sequenceNumber);
    }
    return (o1, o2) -> {
        int result = userDefinedSeqComparator.compare(o1.value(), o2.value());
        if (result != 0) return result;
        return Long.compare(o1.sequenceNumber(), o2.sequenceNumber());
    };
}
```

### 10.4 FirstRowMergeFunctionWrapper

**源码路径**: `paimon-core/.../compact/FirstRowMergeFunctionWrapper.java`

**设计目的**: 为 FirstRow 引擎生成 changelog (仅 INSERT)。

```java
@Override
public ChangelogResult getResult() {
    reusedResult.reset();
    KeyValue result = mergeFunction.getResult();

    if (mergeFunction.containsHighLevel) {
        // 有高层记录说明 key 已存在 -> 不生成 changelog
        return reusedResult.setResult(result);
    }

    if (contains.test(result.key())) {
        // lookup 发现更高层已有此 key -> 不生成 changelog，且不输出结果
        return reusedResult;  // 空结果(null result)
    }

    // 全新的 key -> 生成 INSERT changelog
    return reusedResult.setResult(result).addChangelog(result);
}
```

**`contains` 函数**: 通过 `LookupLevels.lookup()` 检查更高层是否存在此 key。返回 `Boolean`(而非 `KeyValue`)，因为 FirstRow 只关心 key 是否存在，不关心具体值。这大大减少了 lookup 的数据传输量。

---

## 11. CompactRewriter 体系

### 11.1 继承层次

```mermaid
classDiagram
    class AbstractCompactRewriter {
        +rewrite(outputLevel, dropDelete, sections)
        +upgrade(outputLevel, file)
    }

    class MergeTreeCompactRewriter {
        #readerFactory
        #writerFactory
        #keyComparator
        #mfFactory
        #mergeSorter
        +rewrite()
        #rewriteCompaction()
        #readerForMergeTree()
    }

    class ChangelogMergeTreeRewriter {
        #maxLevel
        #mergeEngine
        #produceChangelog
        +rewrite()
        +upgrade()
        #rewriteChangelog()*
        #upgradeStrategy()*
        #createMergeWrapper()*
    }

    class FullChangelogMergeTreeCompactRewriter {
        +rewriteChangelog()
        +upgradeStrategy()
        +createMergeWrapper()
    }

    class LookupMergeTreeCompactRewriter {
        -lookupLevels
        -wrapperFactory
        -dvMaintainer
        +rewriteChangelog()
        +upgradeStrategy()
        +createMergeWrapper()
    }

    AbstractCompactRewriter <|-- MergeTreeCompactRewriter
    MergeTreeCompactRewriter <|-- ChangelogMergeTreeRewriter
    ChangelogMergeTreeRewriter <|-- FullChangelogMergeTreeCompactRewriter
    ChangelogMergeTreeRewriter <|-- LookupMergeTreeCompactRewriter
```

### 11.2 ChangelogMergeTreeRewriter

**源码路径**: `paimon-core/.../compact/ChangelogMergeTreeRewriter.java`

模板方法模式，定义了 Compaction + Changelog 的骨架流程:

```java
@Override
public CompactResult rewrite(int outputLevel, boolean dropDelete, List<List<SortedRun>> sections) {
    if (rewriteChangelog(outputLevel, dropDelete, sections)) {
        return rewriteOrProduceChangelog(outputLevel, sections, dropDelete, true);
    } else {
        return rewriteCompaction(outputLevel, dropDelete, sections);
    }
}
```

`rewriteOrProduceChangelog` 的核心流程:
```java
private CompactResult rewriteOrProduceChangelog(...) {
    iterator = readerForMergeTree(sections, createMergeWrapper(outputLevel)).toCloseableIterator();
    compactFileWriter = writerFactory.createRollingMergeTreeFileWriter(outputLevel, FileSource.COMPACT);
    changelogFileWriter = writerFactory.createRollingChangelogFileWriter(outputLevel);

    while (iterator.hasNext()) {
        ChangelogResult result = iterator.next();
        // 数据文件写入
        if (compactFileWriter != null && result.result() != null && (!dropDelete || result.result().isAdd())) {
            compactFileWriter.write(result.result());
        }
        // changelog 文件写入
        if (produceChangelog) {
            for (KeyValue kv : result.changelogs()) {
                changelogFileWriter.write(kv);
            }
        }
    }
    return new CompactResult(before, after, changelogFiles);
}
```

**三个抽象方法由子类实现**:
- `rewriteChangelog()`: 判断当前 Compaction 是否需要产生 changelog
- `upgradeStrategy()`: 判断 Level 提升时的策略
- `createMergeWrapper()`: 创建对应的 MergeFunctionWrapper

**UpgradeStrategy 枚举**:
```java
protected enum UpgradeStrategy {
    NO_CHANGELOG_NO_REWRITE(false, false),     // 直接升级，不重写不产生changelog
    CHANGELOG_NO_REWRITE(true, false),          // 产生changelog但不重写文件
    CHANGELOG_WITH_REWRITE(true, true);         // 产生changelog并重写文件
}
```

### 11.3 FullChangelogMergeTreeCompactRewriter

**源码路径**: `paimon-core/.../compact/FullChangelogMergeTreeCompactRewriter.java`

简单直接的实现:
```java
protected boolean rewriteChangelog(...) { return outputLevel == maxLevel; }
protected UpgradeStrategy upgradeStrategy(int outputLevel, DataFileMeta file) {
    return outputLevel == maxLevel ? CHANGELOG_NO_REWRITE : NO_CHANGELOG_NO_REWRITE;
}
protected MergeFunctionWrapper<ChangelogResult> createMergeWrapper(int outputLevel) {
    return new FullChangelogMergeFunctionWrapper(mfFactory.create(), maxLevel, valueEqualiser);
}
```

### 11.4 LookupMergeTreeCompactRewriter

**源码路径**: `paimon-core/.../compact/LookupMergeTreeCompactRewriter.java`

升级策略比 Full Compaction 更复杂:

```java
@Override
protected UpgradeStrategy upgradeStrategy(int outputLevel, DataFileMeta file) {
    if (file.level() != 0) return NO_CHANGELOG_NO_REWRITE;

    // Level-0 文件格式与目标层不同 -> 必须重写
    if (!level2FileFormat.apply(file.level()).equals(level2FileFormat.apply(outputLevel)))
        return CHANGELOG_WITH_REWRITE;

    // DV 模式下有删除行 -> 必须重写(drop delete)
    if (dvMaintainer != null && file.deleteRowCount().map(cnt -> cnt > 0).orElse(true))
        return CHANGELOG_WITH_REWRITE;

    // 输出到最高层 -> changelog 但不重写
    if (outputLevel == maxLevel) return CHANGELOG_NO_REWRITE;

    // Deduplicate 无序列字段 -> changelog 但不重写
    if (mergeEngine == MergeEngine.DEDUPLICATE && noSequenceField)
        return CHANGELOG_NO_REWRITE;

    // 其他引擎 -> 必须重写(因为 lookup 可能改变合并结果)
    return CHANGELOG_WITH_REWRITE;
}
```

**为什么 Deduplicate + 无序列字段可以不重写?** Deduplicate 只保留最新记录，如果没有用户定义的序列字段，则按系统 sequence number 排序。Level 升级不会改变"最新"的判定结果，因此不需要重新执行合并计算。

**与 DeletionVector 的集成**:
```java
@Override
protected void notifyRewriteCompactBefore(List<DataFileMeta> files) {
    if (dvMaintainer != null) {
        files.forEach(file -> dvMaintainer.removeDeletionVectorOf(file.fileName()));
    }
}

@Override
protected List<DataFileMeta> notifyRewriteCompactAfter(List<DataFileMeta> files) {
    if (remoteLookupFileManager != null) {
        // 为新文件生成远程 lookup 文件
        return files.stream().map(f -> remoteLookupFileManager.genRemoteLookupFile(f)).collect(...);
    }
    return files;
}
```

**MergeFunctionWrapperFactory 体系**: LookupMergeTreeCompactRewriter 内部定义了两个 Factory:

1. **`LookupMergeFunctionWrapperFactory`**: 用于 Deduplicate/PartialUpdate/Aggregation 引擎
   - 创建 `LookupChangelogMergeFunctionWrapper`
   - lookup 返回 `KeyValue` 或 `PositionedKeyValue`(带文件位置)

2. **`FirstRowMergeFunctionWrapperFactory`**: 专用于 FirstRow 引擎
   - 创建 `FirstRowMergeFunctionWrapper`
   - lookup 仅返回 `Boolean`(key 是否存在)

---

## 12. 跨分区更新

### 12.1 判定条件

**源码路径**: `paimon-api/.../schema/TableSchema.java` (第 200 行)

```java
public boolean crossPartitionUpdate() {
    if (primaryKeys.isEmpty() || partitionKeys.isEmpty()) {
        return false;
    }
    return notContainsAll(primaryKeys, partitionKeys);
    // 当主键不完全包含分区键时，同一主键可能出现在不同分区
}
```

**为什么这意味着跨分区更新?** 如果主键包含所有分区键，则相同主键的记录一定在同一分区内。反之，相同主键的记录可能因为分区字段值不同而分布在不同分区中。当记录的分区字段值变化时(如用户搬家导致城市字段变化)，需要先删除旧分区中的记录，再在新分区中插入。

**对 BucketMode 的影响** (源码路径: `KeyValueFileStore.java` 第 97 行):
```java
if (crossPartitionUpdate) {
    return BucketMode.KEY_DYNAMIC;  // 跨分区更新使用 KEY_DYNAMIC 模式
} else {
    return BucketMode.HASH_DYNAMIC; // 普通动态分桶
}
```

### 12.2 GlobalIndexAssigner 机制

**源码路径**: `paimon-core/.../crosspartition/GlobalIndexAssigner.java`

核心职责: 为跨分区更新的记录分配正确的 bucket，并生成 UPDATE_BEFORE 记录。

```
全局索引 (基于 RocksDB):
   primary_key --> (partition, bucket)

处理流程:
1. 新记录到来，提取主键
2. 查询全局索引: 此主键之前在哪个 (partition, bucket)?
3. 如果之前存在且分区变化:
   a. 生成 DELETE 记录写入旧 (partition, bucket)
   b. 更新索引指向新 (partition, bucket)
   c. 生成 INSERT 记录写入新 (partition, bucket)
4. 如果之前存在且分区未变化:
   a. 直接写入原 (partition, bucket)
5. 如果之前不存在:
   a. 在索引中注册新 (partition, bucket)
   b. 生成 INSERT 记录写入新 (partition, bucket)
```

**为什么使用 RocksDB 而非内存 Map?** 全局索引可能非常大(全表所有主键的映射)，内存放不下。RocksDB 提供了磁盘 + 缓存的混合存储，支持大规模索引。

**配置参数**:
- `cross-partition-upsert.index-ttl`: 索引条目的过期时间，防止索引无限增长 (源码路径: `CoreOptions.java` 第 1800 行)
- `cross-partition-upsert.bootstrap-parallelism`: 启动时初始化索引的并行度 (源码路径: `CoreOptions.java` 第 1809 行)

**Schema 校验约束** (源码路径: `SchemaValidation.java` 第 563 行):
```java
if (schema.crossPartitionUpdate()) {
    // 跨分区更新场景不能使用 sequence.field
    // 因为全局索引无法保证不同分区间的序列号有序性
}
```

**好处**: 使 Paimon 能够支持分区变更(如用户从城市 A 搬到城市 B)等业务场景，这是许多其他数据湖格式不支持的能力。代价是需要维护额外的全局索引，启动时需要 bootstrap(从已有数据构建索引)。

---

## 13. SortMergeReader 体系

### 13.1 MinHeap 实现

**源码路径**: `paimon-core/.../compact/SortMergeReaderWithMinHeap.java`

使用 `PriorityQueue` (最小堆) 进行多路归并:

```java
// 堆中元素的比较规则 (第 56 行):
(e1, e2) -> {
    int result = userKeyComparator.compare(e1.kv.key(), e2.kv.key());
    if (result != 0) return result;
    if (userDefinedSeqComparator != null) {
        result = userDefinedSeqComparator.compare(e1.kv.value(), e2.kv.value());
        if (result != 0) return result;
    }
    return Long.compare(e1.kv.sequenceNumber(), e2.kv.sequenceNumber());
}
```

**排序规则**: 先按用户 key -> 再按用户定义序列号 -> 最后按系统 sequence number。这保证了相同 key 的记录按时间顺序排列，MergeFunction 可以按序处理。

**批处理机制**: MinHeap 实现将读取分为多个 batch。核心在 `SortMergeIterator.nextImpl()`:

```java
// 将之前 poll 出的元素放回或标记为待重新获取
for (Element element : polled) {
    if (element.update()) {     // 尝试获取下一条记录
        minHeap.offer(element); // 还有数据，放回堆
    } else {
        element.iterator.releaseBatch(); // batch 耗尽，释放资源
        nextBatchReaders.add(element.reader); // 等待下一轮 readBatch
    }
}
if (!nextBatchReaders.isEmpty()) {
    return false;  // 有 reader 需要重新获取 batch，结束当前迭代
}

// 从堆顶取出所有相同 key 的元素
mergeFunctionWrapper.reset();
InternalRow key = minHeap.peek().kv.key();
while (!minHeap.isEmpty() && userKeyComparator.compare(key, minHeap.peek().kv.key()) == 0) {
    Element element = minHeap.poll();
    mergeFunctionWrapper.add(element.kv);
    polled.add(element);
}
```

### 13.2 LoserTree 实现

**源码路径**: `paimon-core/.../compact/SortMergeReaderWithLoserTree.java`

使用败者树(Loser Tree)进行多路归并:

```java
// 注意比较器方向是反的 (e2, e1)，因为败者树内部需要 "倒序" 比较
this.loserTree = new LoserTree<>(
    readers,
    (e1, e2) -> userKeyComparator.compare(e2.key(), e1.key()),
    createSequenceComparator(userDefinedSeqComparator)
);
```

**与 MinHeap 的关键区别**: LoserTree 只产生一个 batch，所有数据在一次迭代中完成:

```java
/** Compared with heapsort, LoserTree will only produce one batch. */
@Nullable
@Override
public RecordIterator<T> readBatch() throws IOException {
    loserTree.initializeIfNeeded();
    return loserTree.peekWinner() == null ? null : new SortMergeIterator();
}
```

LoserTree 的 SortMergeIterator 合并逻辑:
```java
@Nullable
@Override
public T next() throws IOException {
    while (true) {
        loserTree.adjustForNextLoop();
        KeyValue winner = loserTree.popWinner();
        if (winner == null) return null;

        mergeFunctionWrapper.reset();
        mergeFunctionWrapper.add(winner);

        // 继续 pop 所有相同 key 的 winner
        while (loserTree.peekWinner() != null) {
            mergeFunctionWrapper.add(loserTree.popWinner());
        }
        T result = mergeFunctionWrapper.getResult();
        if (result != null) return result;
    }
}
```

### 13.3 选择策略与性能对比

**源码路径**: `paimon-core/.../compact/SortMergeReader.java`

```java
static <T> SortMergeReader<T> createSortMergeReader(..., SortEngine sortEngine) {
    switch (sortEngine) {
        case MIN_HEAP:  return new SortMergeReaderWithMinHeap<>(...);
        case LOSER_TREE: return new SortMergeReaderWithLoserTree<>(...);
    }
}
```

通过 `sort-engine` 配置:

| 特性 | MinHeap | LoserTree |
|------|---------|-----------|
| **数据结构** | PriorityQueue(动态数组堆) | 败者树(固定数组) |
| **比较次数** | O(log n) per pop | O(log n) per pop |
| **Cache 友好性** | 较差(对象引用跳转) | 较好(数组连续内存) |
| **批处理** | 多 batch(可渐进释放内存) | 单 batch(一次性处理) |
| **内存占用** | 动态，可渐进释放 | 固定，一次性分配 |
| **默认** | 否 | **是(默认)** |

**为什么默认选择 LoserTree?** 败者树的数组结构对 CPU Cache 更友好，在多路归并场景下有更稳定的性能表现。Paimon 作为数据湖格式，Compaction 是性能关键路径，选择更高效的归并算法有显著收益。

**MergeSorter 的角色** (源码路径: `paimon-core/.../mergetree/MergeSorter.java`):

MergeSorter 在 SortMergeReader 之上增加了一层: 当参与合并的 SortedRun 数据量超过内存阈值时，先将数据 spill 到磁盘的临时排序文件，再从磁盘文件进行归并。这防止了大规模 Compaction 时的 OOM。

---

## 14. 设计决策总结

### 14.1 策略模式 + 装饰器模式的双重运用

**决策**: MergeFunction 用策略模式，MergeFunctionWrapper 用装饰器模式。

**为什么**: 合并语义(Deduplicate/PartialUpdate/Aggregation/FirstRow)和变更日志生成(None/Full/Lookup)是两个正交维度。策略模式让合并语义可替换，装饰器模式让 changelog 行为可叠加。

**好处**: 4 种 MergeEngine x 4 种 Changelog 模式的组合，只需 4+4 个类实现而非 4x4=16 个类。代码量大幅减少，扩展性极强。

### 14.2 对象复用与零拷贝

**决策**: MergeFunction 内部大量复用对象(如 `KeyValue reused`, `GenericRow row`)。

**为什么**: Compaction 是高吞吐量操作，处理数百万甚至数十亿条记录。每条记录创建新对象会导致严重的 GC 压力。

**好处**: 显著降低 GC 开销，提高 Compaction 吞吐量。代价是代码复杂度增加(需要注意对象引用安全性，参见 MergeFunction 的 JavaDoc 注释)。

### 14.3 SPI 机制的聚合函数扩展

**决策**: 聚合函数通过 Java SPI (`FieldAggregatorFactory`) 注册，而非硬编码。

**为什么**: Paimon 作为通用数据湖格式，不同用户有不同的聚合需求。内置函数无法覆盖所有场景(如特定业务的自定义统计量)。

**好处**: 用户可以实现自定义聚合函数，打成 JAR 包放入 classpath 即可使用，无需修改 Paimon 源码。这遵循了开放-封闭原则(OCP)。

### 14.4 Sequence Group 的精细化控制

**决策**: PartialUpdate 引擎支持 Sequence Group，每组列有独立的序列号。

**为什么**: 多源宽表拼接场景中，不同源的更新频率和时序不同，全局序列号无法解决乱序问题。例如源 A 每秒更新一次，源 B 每分钟更新一次，如果用全局序列号，源 A 的旧数据可能覆盖源 B 的新数据。

**好处**: 每个源系统用自己的序列号控制自己负责的列，互不干扰。配合聚合函数(如 SUM)还能在 Partial Update 中实现部分字段的预聚合。

### 14.5 Lookup vs Full Compaction 的权衡

**决策**: 提供两种 changelog 生成策略: Full Compaction(高延迟低成本)和 Lookup(低延迟高成本)。

**为什么**: 不同业务对延迟和成本的容忍度不同。金融场景需要秒级 changelog，而离线报表可以容忍分钟级延迟。

**好处**: 用户可以根据业务需求在延迟和成本之间做取舍。生产环境中，大多数流式场景选择 Lookup 模式以获得低延迟 changelog。

### 14.6 LoserTree 作为默认排序引擎

**决策**: 默认使用 LoserTree 而非 MinHeap 进行多路归并。

**为什么**: 败者树的数组结构对现代 CPU 的缓存层次更友好，在路数较多时性能优势明显。LSM-Tree 的 Compaction 通常涉及多个 SortedRun 的归并，路数可达数十甚至上百。

**好处**: Compaction 性能提升(取决于路数和数据分布)。代价是实现复杂度略高于 MinHeap。

### 14.7 跨分区更新的全局索引

**决策**: 通过基于 RocksDB 的全局索引实现跨分区 Upsert。

**为什么**: 当主键不包含分区键时，同一主键可能出现在不同分区。没有全局索引就无法知道记录之前在哪个分区，也就无法生成正确的 DELETE 记录来清理旧分区的数据。

**好处**: 使 Paimon 能够支持分区变更(如用户从城市 A 搬到城市 B)等业务场景。代价是需要维护额外的全局索引，启动时需要从已有数据 bootstrap 构建索引，有一定的启动延迟。

### 14.8 模板方法模式在 CompactRewriter 中的应用

**决策**: `ChangelogMergeTreeRewriter` 使用模板方法模式定义 Compaction 骨架。

**为什么**: Full Compaction changelog 和 Lookup changelog 的整体流程相同(读取 -> 合并 -> 写数据文件 + changelog 文件)，但判断条件和具体 Wrapper 不同。

**好处**: 避免代码重复，子类只需关注差异化逻辑。新增 changelog 模式只需继承 `ChangelogMergeTreeRewriter` 并实现三个抽象方法。
