# Apache Paimon 局部列更新与 CDC 数据集成深度分析

> 基于 Paimon 1.5-SNAPSHOT 源码分析，commit: 55f4fd175

---

## 目录

- [1. 局部列更新 (Partial Update) 深度分析](#1-局部列更新-partial-update-深度分析)
  - [1.1 整体架构与核心类](#11-整体架构与核心类)
  - [1.2 无 Sequence Group 时的 updateNonNullFields 逻辑](#12-无-sequence-group-时的-updatenonnullfields-逻辑)
  - [1.3 有 Sequence Group 时的 updateWithSequenceGroup 逻辑](#13-有-sequence-group-时的-updatewithsequencegroup-逻辑)
  - [1.4 Sequence Group 配置解析和字段映射](#14-sequence-group-配置解析和字段映射)
  - [1.5 Partial Update + Aggregation 混合使用](#15-partial-update--aggregation-混合使用)
  - [1.6 删除处理的三种模式](#16-删除处理的三种模式)
  - [1.7 retractWithSequenceGroup 的实现](#17-retractwithsequencegroup-的实现)
  - [1.8 使用示例](#18-使用示例)
- [2. 聚合表 (Aggregation) 深度分析](#2-聚合表-aggregation-深度分析)
  - [2.1 AggregateMergeFunction 核心机制](#21-aggregatemergefunction-核心机制)
  - [2.2 getAggFuncName 的字段分配逻辑](#22-getaggfuncname-的字段分配逻辑)
  - [2.3 内置聚合函数完整列表](#23-内置聚合函数完整列表)
  - [2.4 Retract 支持的聚合函数](#24-retract-支持的聚合函数)
  - [2.5 自定义聚合函数的 SPI 扩展方法](#25-自定义聚合函数的-spi-扩展方法)
- [3. CDC 数据集成能力](#3-cdc-数据集成能力)
  - [3.1 CDC 模块架构](#31-cdc-模块架构)
  - [3.2 支持的 CDC 源](#32-支持的-cdc-源)
  - [3.3 SyncTableActionBase / SyncDatabaseActionBase 的构建流程](#33-synctableactionbase--syncdatabaseactionbase-的构建流程)
  - [3.4 CDC 数据解析：RichCdcMultiplexRecord 到 CdcRecord](#34-cdc-数据解析richcdcmultiplexrecord-到-cdcrecord)
  - [3.5 Schema 自动演进](#35-schema-自动演进)
  - [3.6 库级同步](#36-库级同步)
- [4. Changelog 产生机制详解](#4-changelog-产生机制详解)
  - [4.1 ChangelogProducer 枚举与 3 种模式 + FirstRow 特殊机制](#41-changelogproducer-枚举与-3-种模式--firstrow-特殊机制)
  - [4.2 INPUT 模式](#42-input-模式)
  - [4.3 FULL_COMPACTION 模式](#43-full_compaction-模式)
  - [4.4 LOOKUP 模式](#44-lookup-模式)
  - [4.5 FirstRow 模式](#45-firstrow-模式)
  - [4.6 各模式的权衡对比](#46-各模式的权衡对比)
- [5. 跨分区更新](#5-跨分区更新)
  - [5.1 GlobalIndexAssigner 的全局索引维护](#51-globalindexassigner-的全局索引维护)
  - [5.2 RocksDB 存储的 key 到 partition-bucket 映射](#52-rocksdb-存储的-key-到-partition-bucket-映射)
  - [5.3 ExistingProcessor 的不同策略](#53-existingprocessor-的不同策略)
  - [5.4 Bootstrap 流程](#54-bootstrap-流程)
- [6. Schema Evolution 详解](#6-schema-evolution-详解)
  - [6.1 SchemaManager.commitChanges 的完整流程](#61-schemamanagercommitchanges-的完整流程)
  - [6.2 所有 SchemaChange 类型](#62-所有-schemachange-类型)
  - [6.3 CDC 场景下的自动 Schema 演进](#63-cdc-场景下的自动-schema-演进)
- [7. 实战示例](#7-实战示例)
  - [7.1 Partial Update 建表和使用示例](#71-partial-update-建表和使用示例)
  - [7.2 Aggregation 建表和使用示例](#72-aggregation-建表和使用示例)
  - [7.3 MySQL CDC 同步到 Paimon 的完整配置](#73-mysql-cdc-同步到-paimon-的完整配置)
  - [7.4 跨分区更新的配置和注意事项](#74-跨分区更新的配置和注意事项)

---

## 1. 局部列更新 (Partial Update) 深度分析

### 1.1 整体架构与核心类

**核心类**: `PartialUpdateMergeFunction`
**源码路径**: `paimon-core/src/main/java/org/apache/paimon/mergetree/compact/PartialUpdateMergeFunction.java` (720行)

**为什么需要 Partial Update?**
在实际业务中，同一行数据的不同列往往来自不同数据源或不同时间点的更新。例如订单系统中，创建时只有基本信息，支付时更新支付信息，发货时更新物流信息。Partial Update 允许每次只更新非空字段，而非全量覆盖整行。

**好处**:
- 避免多源数据相互覆盖导致字段丢失
- 简化 ETL 逻辑，无需在上游进行全量合并
- 支持字段级别的版本控制（通过 Sequence Group）

**继承关系**:
```
MergeFunction<KeyValue>         -- 接口: reset(), add(kv), getResult()
    └── PartialUpdateMergeFunction  -- 实现: 非空字段更新逻辑
```

**核心字段** (第69-83行):

| 字段 | 类型 | 作用 |
|------|------|------|
| `getters` | `InternalRow.FieldGetter[]` | 字段值提取器数组 |
| `ignoreDelete` | `boolean` | 是否忽略删除记录 |
| `fieldSeqComparators` | `List<WrapperWithFieldIndex<FieldsComparator>>` | 每个字段的序列号比较器（按字段索引排序） |
| `fieldSequenceEnabled` | `boolean` | 是否启用了 Sequence Group |
| `fieldAggregators` | `List<WrapperWithFieldIndex<FieldAggregator>>` | 每个字段的聚合器（按字段索引排序） |
| `removeRecordOnDelete` | `boolean` | 收到 DELETE 记录时是否删除整行 |
| `sequenceGroupPartialDelete` | `Set<Integer>` | 指定哪些 sequence field 收到 DELETE 时删除整行 |
| `nullables` | `boolean[]` | 每个字段是否允许为空 |
| `row` | `GenericRow` | 当前合并结果行 |
| `meetInsert` | `boolean` | 是否遇到过 INSERT 记录 |

### 1.2 无 Sequence Group 时的 updateNonNullFields 逻辑

**源码位置**: 第177-188行

```java
private void updateNonNullFields(KeyValue kv) {
    for (int i = 0; i < getters.length; i++) {
        Object field = getters[i].getFieldOrNull(kv.value());
        if (field != null) {
            row.setField(i, field);          // 非空则更新
        } else {
            if (!nullables[i]) {
                throw new IllegalArgumentException("Field " + i + " can not be null");
            }
            // 空值则保留旧值（不做任何操作）
        }
    }
}
```

**为什么这样设计?**
最简单的语义：新记录中的非空字段覆盖旧值，空字段保留旧值。这是 partial-update 的核心语义——"只更新有值的列"。

**好处**: 实现简洁、语义清晰、无需额外配置。

**触发条件** (第168行): 当 `fieldSeqComparators` 为空时（即未配置 sequence-group），走此分支。

**关键细节**:
- 如果字段 `nullables[i]` 为 `false`（NOT NULL 约束），而输入值为 null，会直接抛出异常
- 该逻辑没有任何乱序保护，依赖记录按 sequenceNumber 排序到达

### 1.3 有 Sequence Group 时的 updateWithSequenceGroup 逻辑

**源码位置**: 第190-247行

**为什么需要 Sequence Group?**
在无 Sequence Group 模式下，所有字段共享同一个全局序列号，后到的记录覆盖先到的。但在实际场景中，不同字段可能由不同数据源更新，各自有独立的版本控制。例如：
- 支付系统用 `payment_ts` 控制支付字段的版本
- 物流系统用 `shipping_ts` 控制物流字段的版本

Sequence Group 允许为不同字段组指定独立的序列号字段，实现字段级别的乱序容忍。

**核心逻辑**:

```java
private void updateWithSequenceGroup(KeyValue kv) {
    // 使用两个迭代器同步遍历（都按字段索引排序）
    Iterator<WrapperWithFieldIndex<FieldsComparator>> comparatorIter = ...;
    Iterator<WrapperWithFieldIndex<FieldAggregator>> aggIter = ...;

    for (int i = 0; i < getters.length; i++) {
        // 获取当前字段的 seqComparator 和 aggregator
        FieldsComparator seqComparator = ...;
        FieldAggregator aggregator = ...;

        if (seqComparator == null) {
            // 不属于任何 sequence group 的字段
            Object field = getters[i].getFieldOrNull(kv.value());
            if (aggregator != null) {
                row.setField(i, aggregator.agg(accumulator, field));  // 有聚合器则聚合
            } else if (field != null) {
                row.setField(i, field);  // 无聚合器则非空更新
            }
        } else {
            // 属于某个 sequence group 的字段
            if (isEmptySequenceGroup(kv, seqComparator, ...)) {
                continue;  // 跳过序列号字段全为 null 的 group
            }
            if (seqComparator.compare(kv.value(), row) >= 0) {
                // 新记录的序列号 >= 旧记录的序列号 → 更新
                // 特殊处理：sequence 字段本身需要一次性更新所有
                if (isSequenceField(index)) {
                    for (int fieldIndex : seqComparator.compareFields()) {
                        row.setField(fieldIndex, getters[fieldIndex].getFieldOrNull(kv.value()));
                    }
                } else {
                    row.setField(i, aggregator == null ? field : aggregator.agg(accumulator, field));
                }
            } else if (aggregator != null) {
                // 旧记录的序列号更大，但有聚合器 → 反向聚合
                row.setField(i, aggregator.aggReversed(accumulator, field));
            }
        }
    }
}
```

**好处**:
- 字段级别的乱序容忍，不同数据源可以安全地并行更新不同字段组
- 与聚合函数结合时，即使收到乱序数据也能正确计算（通过 `aggReversed`）

**关键设计决策**:
1. **isEmptySequenceGroup 检查** (第249-269行): 如果 sequence 字段全为 null，表示该 group 没有有效数据，跳过整个 group。**为什么**: 避免用 null 序列号覆盖有效数据。
2. **sequence 字段一次性更新** (第232-238行): 同一 group 的多个 sequence 字段必须原子更新。**为什么**: 保持序列号字段之间的一致性。
3. **aggReversed** (第243行): 当新记录序列号小于旧记录时，调用 `aggReversed(accumulator, field)` 而非 `agg`。**为什么**: 聚合操作可能不满足交换律，例如 `last_value` 语义下需要区分先后顺序。

### 1.4 Sequence Group 配置解析和字段映射

**源码位置**: Factory 构造函数，第392-491行

**配置格式**:
```
fields.{seq_field1},{seq_field2}.sequence-group = {value_field1},{value_field2},{value_field3}
```

**解析流程**:

1. **遍历所有 options** (第405行): 查找以 `fields.` 开头、以 `.sequence-group` 结尾的配置项
2. **解析 sequence 字段** (第409-418行): 从 key 中提取 `seq_field1,seq_field2`，映射为字段索引数组
3. **创建 FieldsComparator** (第420-421行): 为每个 sequence group 创建 `UserDefinedSeqComparator`
4. **映射 value 字段** (第422-434行): 将 value 的每个字段关联到对应的 FieldsComparator
5. **注册 sequence 字段自身** (第437-443行): sequence 字段也需要关联到自己的 comparator
6. **创建聚合器** (第445-451行): 调用 `createFieldAggregators` 为配置了聚合函数的字段创建聚合器

**冲突检测** (第454-491行):
- `ignoreDelete` 和 `removeRecordOnDelete` 不能同时启用
- `ignoreDelete` 和 `removeRecordOnSequenceGroup` 不能同时启用
- `removeRecordOnDelete` 和 `sequence-group` 不能同时启用
- `removeRecordOnSequenceGroup` 中引用的字段必须是 sequence group 的序列号字段

**为什么要做这些互斥检查?** 这些选项对删除记录的处理策略相互矛盾。例如 `ignoreDelete` 表示丢弃删除记录，而 `removeRecordOnDelete` 表示执行删除，两者语义冲突。

### 1.5 Partial Update + Aggregation 混合使用

**源码位置**: `createFieldAggregators` 方法，第626-656行；`getAggFuncName` 方法，第660-690行

**聚合函数分配逻辑** (`getAggFuncName`):

```
1. 如果字段是 sequence 字段 → 返回 null（不聚合）
2. 如果字段是 primary key 字段 → 返回 "primary-key"（主键不参与聚合）
3. 获取用户配置的 fields.{field_name}.aggregate-function
4. 如果没有配置，尝试获取 fields.default-aggregate-function
5. 如果有聚合函数名，校验：
   - last_non_null_value 不要求 sequence group
   - 其他聚合函数必须在 sequence group 保护下使用
```

**为什么聚合函数（除 last_non_null_value 外）必须配合 Sequence Group?**
在 Partial Update 场景下，不同时间点的记录以乱序方式到达。如果不使用 Sequence Group 提供的版本控制，聚合操作（如 sum）无法区分正常聚合和乱序重复聚合，可能导致结果错误。而 `last_non_null_value` 本质上就是 partial-update 的默认行为（取最后一个非空值），天然适配乱序场景。

**好处**: 在同一张表中，可以对不同字段使用不同的合并策略。例如：
- `name` 字段使用 partial-update 语义（取最新非空值）
- `total_amount` 字段使用 sum 聚合
- `last_login_time` 字段使用 last_value 聚合

### 1.6 删除处理的三种模式

**源码位置**: `add` 方法中的 retract 分支，第126-165行

当收到 retract 记录（`RowKind.DELETE` 或 `RowKind.UPDATE_BEFORE`）时：

#### 模式 1: ignoreDelete

```java
if (ignoreDelete) {
    return;  // 直接丢弃删除记录
}
```

**配置**: `'ignore-delete' = 'true'`

**为什么选择**: 当上游数据源发出的删除记录不需要在 Paimon 中生效时。例如 CDC 同步场景中上游执行了软删除（标记字段），不需要在 Paimon 中物理删除。

**好处**: 最简单的策略，无副作用。

#### 模式 2: removeRecordOnDelete

```java
if (removeRecordOnDelete) {
    if (kv.valueKind() == RowKind.DELETE) {
        currentDeleteRow = true;
        row = new GenericRow(getters.length);
        initRow(row, kv.value());
    }
    return;
}
```

**配置**: `'partial-update.remove-record-on-delete' = 'true'`

**为什么选择**: 当需要在收到 DELETE 记录时删除整行数据。只有 `RowKind.DELETE` 会触发删除，`UPDATE_BEFORE` 不会。

**好处**: 提供精确的删除语义。

**限制**: 不能与 sequence-group 同时使用（因为 sequence-group 需要更细粒度的删除控制）。

#### 模式 3: removeRecordOnSequenceGroup (基于序列组的部分删除)

```java
// 在 retractWithSequenceGroup 中处理
if (kv.valueKind() == RowKind.DELETE && sequenceGroupPartialDelete.contains(field)) {
    currentDeleteRow = true;
    row = new GenericRow(getters.length);
    initRow(row, kv.value());
    return;
}
```

**配置**: `'partial-update.remove-record-on-sequence-group' = 'seq_field1,seq_field2'`

**为什么选择**: 当只有特定序列组的 DELETE 记录应该触发整行删除时。例如订单表中，只有当"订单状态"序列组收到删除时才删除整行，而"物流信息"序列组的删除只清空对应字段。

**好处**: 最细粒度的删除控制，精确指定哪些序列组的删除行为会导致整行删除。

#### 默认行为：抛出异常

如果以上三种模式都未配置，收到删除记录会抛出 `IllegalArgumentException`，并提示用户选择一种删除处理策略。**为什么**: 防止用户在不理解删除语义的情况下无声丢失数据。

### 1.7 retractWithSequenceGroup 的实现

**源码位置**: 第271-342行

**触发条件**: `fieldSequenceEnabled == true` 且收到 retract 记录。

**处理逻辑**:

```
对于每个字段 i:
  1. 如果字段属于某个 sequence group:
     a. 跳过 sequence 字段全为 null 的 group
     b. 如果新记录的序列号 >= 旧记录的序列号:
        - 如果当前字段是 sequence 字段本身:
          · 检查是否需要触发整行删除 (sequenceGroupPartialDelete)
          · 如果是，设置 currentDeleteRow=true 并初始化新空行
          · 否则，更新 sequence 字段值
        - 如果当前字段是普通字段:
          · 无聚合器: 设置为 null（撤回语义）
          · 有聚合器: 调用 aggregator.retract(accumulator, inputField)
     c. 如果旧记录的序列号更大，但有聚合器:
        · 调用 aggregator.retract(accumulator, inputField)
```

**为什么 retract 对普通字段设为 null 而非保留旧值?**
retract 记录表示"撤回之前写入的值"，对于 partial-update 场景，将字段设为 null 是正确的撤回语义——表示该数据源不再提供该字段的值。

**为什么乱序的 retract 也需要处理聚合?** (第333-338行)
对于聚合字段，即使序列号较旧，retract 操作也需要执行，因为聚合是累积计算。例如 sum 聚合，撤回旧值需要减去对应金额，而不是简单忽略。

### 1.8 使用示例

#### 基础 Partial Update（无 Sequence Group）

```sql
CREATE TABLE user_profile (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    name STRING,
    email STRING,
    phone STRING,
    address STRING
) WITH (
    'merge-engine' = 'partial-update',
    'ignore-delete' = 'true'
);

-- 第一次写入：来自注册系统
INSERT INTO user_profile VALUES (1, 'Alice', 'alice@example.com', NULL, NULL);
-- 结果: (1, 'Alice', 'alice@example.com', NULL, NULL)

-- 第二次写入：来自手机绑定系统
INSERT INTO user_profile VALUES (1, NULL, NULL, '13800138000', NULL);
-- 结果: (1, 'Alice', 'alice@example.com', '13800138000', NULL)

-- 第三次写入：来自地址更新系统
INSERT INTO user_profile VALUES (1, NULL, NULL, NULL, 'Beijing');
-- 结果: (1, 'Alice', 'alice@example.com', '13800138000', 'Beijing')
```

#### Partial Update + Sequence Group

```sql
CREATE TABLE order_info (
    order_id BIGINT PRIMARY KEY NOT ENFORCED,
    order_status STRING,
    payment_amount DECIMAL(10,2),
    payment_ts BIGINT,
    shipping_address STRING,
    tracking_no STRING,
    shipping_ts BIGINT
) WITH (
    'merge-engine' = 'partial-update',
    'fields.payment_ts.sequence-group' = 'order_status,payment_amount',
    'fields.shipping_ts.sequence-group' = 'shipping_address,tracking_no'
);

-- 支付系统更新（payment_ts=100）
INSERT INTO order_info VALUES (1, 'PAID', 99.00, 100, NULL, NULL, NULL);
-- 结果: (1, 'PAID', 99.00, 100, NULL, NULL, NULL)

-- 物流系统更新（shipping_ts=200）
INSERT INTO order_info VALUES (1, NULL, NULL, NULL, 'Beijing', 'SF001', 200);
-- 结果: (1, 'PAID', 99.00, 100, 'Beijing', 'SF001', 200)

-- 支付系统乱序更新（payment_ts=50，比现有的100小，被忽略）
INSERT INTO order_info VALUES (1, 'PENDING', 0.00, 50, NULL, NULL, NULL);
-- 结果: (1, 'PAID', 99.00, 100, 'Beijing', 'SF001', 200)  -- 支付信息未变
```

#### Partial Update + Aggregation

```sql
CREATE TABLE user_behavior (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    latest_action STRING,
    action_count BIGINT,
    total_amount DECIMAL(10,2),
    action_ts BIGINT
) WITH (
    'merge-engine' = 'partial-update',
    'fields.action_ts.sequence-group' = 'latest_action,action_count,total_amount',
    'fields.action_count.aggregate-function' = 'sum',
    'fields.total_amount.aggregate-function' = 'sum'
);
```

---

## 2. 聚合表 (Aggregation) 深度分析

### 2.1 AggregateMergeFunction 核心机制

Aggregation 合并引擎通过配置 `'merge-engine' = 'aggregation'` 启用。与 Partial Update 不同，Aggregation 引擎的所有非主键字段都必须指定聚合函数。

**核心抽象类**: `FieldAggregator`
**源码路径**: `paimon-core/src/main/java/org/apache/paimon/mergetree/compact/aggregate/FieldAggregator.java`

```java
public abstract class FieldAggregator implements Serializable {
    protected final DataType fieldType;
    protected final String name;

    public abstract Object agg(Object accumulator, Object inputField);

    // 反向聚合（用于乱序数据处理）
    public Object aggReversed(Object accumulator, Object inputField) {
        return agg(inputField, accumulator);
    }

    // 撤回操作（用于处理 DELETE/UPDATE_BEFORE 记录）
    public Object retract(Object accumulator, Object retractField) {
        throw new UnsupportedOperationException(...);
    }
}
```

**三个核心方法的设计意图**:
- `agg(accumulator, inputField)`: 将新值合并到累加器中。**为什么**: 这是最基本的聚合操作。
- `aggReversed(accumulator, inputField)`: 当新记录的序列号小于旧记录时调用，默认实现为 `agg(inputField, accumulator)`。**为什么**: 某些聚合操作（如 last_value）不满足交换律，需要区分参数顺序。
- `retract(accumulator, retractField)`: 撤回之前聚合的值。**为什么**: 支持 CDC 场景中的删除和更新前记录处理。

### 2.2 getAggFuncName 的字段分配逻辑

**源码位置**: `PartialUpdateMergeFunction.getAggFuncName` 方法，第660-690行
（注意：此方法同时被 Partial Update 和 Aggregation 引擎使用）

**分配优先级**:

```
1. sequence 字段 → 返回 null（不参与聚合）
   为什么: sequence 字段是版本控制字段，不应被聚合修改

2. primary key 字段 → 返回 "primary-key"
   为什么: 主键是记录的唯一标识，必须保持不变

3. 用户显式配置的 fields.{field_name}.aggregate-function → 返回用户配置值
   为什么: 用户的显式配置拥有最高优先级

4. fields.default-aggregate-function → 返回默认聚合函数名
   为什么: 提供全表级别的默认聚合策略，减少逐字段配置

5. 以上都未匹配 → 返回 null
   为什么: 在 Partial Update 中表示使用默认的非空更新语义
```

**配置方式**:
```properties
# 单字段配置
fields.amount.aggregate-function = sum

# 默认聚合函数
fields.default-aggregate-function = last_non_null_value
```

### 2.3 内置聚合函数完整列表

Paimon 的聚合函数通过 SPI (`FieldAggregatorFactory`) 机制加载。以下是基于源码中 factory 包下所有实现的完整列表（共21种）：

| 聚合函数 | 标识符 | 支持的数据类型 | 行为描述 | 支持 retract |
|---------|--------|--------------|---------|-------------|
| `sum` | `sum` | NumericType | 求和 | 是（减法） |
| `product` | `product` | NumericType | 累乘 | 是（除法） |
| `count` | `count` | NumericType | 计数（累加输入值） | 是（减法） |
| `max` | `max` | ComparableType | 取最大值 | 否 |
| `min` | `min` | ComparableType | 取最小值 | 否 |
| `last_value` | `last_value` | 任意类型 | 取最后一个值（包括 null） | 是（恢复旧值） |
| `last_non_null_value` | `last_non_null_value` | 任意类型 | 取最后一个非空值 | 否 |
| `first_value` | `first_value` | 任意类型 | 取第一个值 | 否 |
| `first_non_null_value` | `first_non_null_value` | 任意类型 | 取第一个非空值 | 否 |
| `listagg` | `listagg` | STRING | 字符串拼接（可配置分隔符） | 否 |
| `bool_and` | `bool_and` | BOOLEAN | 逻辑与 | 否 |
| `bool_or` | `bool_or` | BOOLEAN | 逻辑或 | 否 |
| `collect` | `collect` | 任意类型 | 收集到 ARRAY | 是 |
| `merge_map` | `merge_map` | MAP | 合并 MAP | 是（删除 key） |
| `merge_map_with_key_time` | `merge_map_with_key_time` | MAP | 带时间戳的 MAP 合并 | 是 |
| `nested_update` | `nested_update` | ARRAY(ROW) | 嵌套行更新 | 是 |
| `nested_partial_update` | `nested_partial_update` | ARRAY(ROW) | 嵌套行部分更新 | 是 |
| `primary-key` | `primary-key` | 任意类型 | 主键字段（保持不变） | - |
| `theta_sketch` | `theta_sketch` | VARBINARY | Theta Sketch 近似去重 | 否 |
| `roaring_bitmap` | `roaring_bitmap` | VARBINARY | RoaringBitmap 合并 | 否 |
| `rbm64` | `rbm64` | VARBINARY | 64位 RoaringBitmap 合并 | 否 |
| `hll_sketch` | `hll_sketch` | VARBINARY | HLL Sketch 基数估计 | 否 |

**补充选项**:
- `fields.{field}.ignore-retract = true`: 让聚合函数忽略 retract 消息（由 `FieldIgnoreRetractAgg` 包装器实现）
- `fields.{field}.distinct = true`: 去重模式（用于 collect 等函数）
- `fields.{field}.list-agg-delimiter`: listagg 的分隔符配置

### 2.4 Retract 支持的聚合函数

支持 retract 的聚合函数及其撤回逻辑：

| 聚合函数 | retract 实现 | 说明 |
|---------|-------------|------|
| `sum` | `accumulator - retractField` | 从累加值中减去撤回值 |
| `product` | `accumulator / retractField` | 从累积值中除以撤回值 |
| `count` | `accumulator - retractField` | 从计数中减去撤回值 |
| `collect` | 从 ARRAY 中移除对应元素 | 精确移除 |
| `merge_map` | 从 MAP 中删除对应 key | 精确移除 |
| `last_value` | 恢复为 accumulator（旧值） | 如果撤回值等于当前值则重置 |
| `nested_update` | 根据嵌套 key 删除对应行 | 基于嵌套 key 匹配 |

**不支持 retract 的函数**: `max`, `min`, `first_value`, `first_non_null_value`, `last_non_null_value`, `listagg`, `bool_and`, `bool_or`, `theta_sketch`, `roaring_bitmap`, `rbm64`, `hll_sketch`

**为什么有些函数不支持 retract?**
- `max`/`min`: 撤回当前极值后无法恢复次极值（需要保存所有历史值）
- `first_value`: 撤回第一个值后无法恢复第二个值
- `listagg`: 拼接后无法安全拆分（分隔符可能出现在值中）

**如果需要使用不支持 retract 的函数又有 retract 记录**: 配置 `fields.{field}.ignore-retract = true`

### 2.5 自定义聚合函数的 SPI 扩展方法

**SPI 接口**: `FieldAggregatorFactory`
**源码路径**: `paimon-core/src/main/java/org/apache/paimon/mergetree/compact/aggregate/factory/FieldAggregatorFactory.java`

```java
public interface FieldAggregatorFactory extends Factory {
    FieldAggregator create(DataType fieldType, CoreOptions options, String field);
    String identifier();  // 聚合函数的唯一标识符
}
```

**扩展步骤**:

1. 实现 `FieldAggregator` 抽象类，覆写 `agg()` 方法，可选覆写 `retract()` 和 `aggReversed()`
2. 实现 `FieldAggregatorFactory` 接口，提供 `identifier()` 和 `create()` 方法
3. 在 `META-INF/services/org.apache.paimon.mergetree.compact.aggregate.factory.FieldAggregatorFactory` 文件中注册工厂类全限定名

**加载机制** (FieldAggregatorFactory.create 静态方法，第39-65行):
```java
FieldAggregatorFactory factory = FactoryUtil.discoverFactory(
    FieldAggregator.class.getClassLoader(),
    FieldAggregatorFactory.class,
    aggFuncName);  // 通过 identifier() 匹配
```

**为什么使用 SPI?**
- 插件化设计，用户可以在不修改 Paimon 源码的情况下添加自定义聚合函数
- 与 Java ServiceLoader 标准兼容

---

## 3. CDC 数据集成能力

### 3.1 CDC 模块架构

CDC 集成代码主要位于 `paimon-flink/paimon-flink-cdc` 模块。

```
SynchronizationActionBase (抽象基类)
    ├── SyncTableActionBase (表级同步基类)
    │   ├── MySqlSyncTableAction
    │   ├── PostgresSyncTableAction
    │   ├── KafkaSyncTableAction
    │   ├── PulsarSyncTableAction
    │   └── MongoDBSyncTableAction
    └── SyncDatabaseActionBase (库级同步基类)
        ├── MySqlSyncDatabaseAction
        ├── PostgresSyncDatabaseAction
        ├── KafkaSyncDatabaseAction
        ├── PulsarSyncDatabaseAction
        └── MongoDBSyncDatabaseAction
```

**核心数据流**:
```
CDC Source → CdcSourceRecord → RecordParser → RichCdcMultiplexRecord → EventParser → CdcRecord → Paimon Writer
```

### 3.2 支持的 CDC 源

**源码位置**: `SyncJobHandler.SourceType` 枚举，第254-268行

| CDC 源 | SourceType | 必需参数 | 解析器 |
|--------|-----------|---------|--------|
| MySQL | `MYSQL` | hostname, username, password, database-name | `MySqlRecordParser` |
| PostgreSQL | `POSTGRES` | hostname, username, password, database-name, schema-name, slot-name | `PostgresRecordParser` |
| Kafka | `KAFKA` | value.format, properties.bootstrap.servers, topic/topic-pattern | `DataFormat.createParser()` |
| Pulsar | `PULSAR` | value.format, pulsar.service-url, pulsar.subscription.name, topic/topic-pattern | `DataFormat.createParser()` |
| MongoDB | `MONGODB` | hosts, database | `MongoDBRecordParser` |

**为什么支持 Kafka/Pulsar?**
Kafka 和 Pulsar 不直接产生 CDC 数据，但可以作为 CDC 数据的中转。例如 Debezium 将 MySQL CDC 数据写入 Kafka，Paimon 从 Kafka 消费。通过 `value.format` 配置（如 `debezium-json`, `canal-json`, `maxwell-json`）指定数据格式。

**好处**: 解耦 CDC 采集和数据湖写入，提高架构灵活性。

### 3.3 SyncTableActionBase / SyncDatabaseActionBase 的构建流程

#### 表级同步 (SyncTableActionBase)

**源码路径**: `paimon-flink/paimon-flink-cdc/src/main/java/org/apache/paimon/flink/action/cdc/SyncTableActionBase.java`

**构建流程** (`build()` → `beforeBuildingSourceSink()` → `buildSink()`):

```
1. SynchronizationActionBase.build():
   a. syncJobHandler.checkRequiredOption()     -- 检查必需参数
   b. catalog.createDatabase(database, true)   -- 创建目标数据库
   c. beforeBuildingSourceSink()               -- 准备表和 schema
   d. buildDataStreamSource(buildSource())     -- 构建 CDC 数据源
   e. flatMap(recordParse())                   -- 解析为 RichCdcMultiplexRecord
   f. buildSink(input, parserFactory)          -- 构建 Paimon Sink

2. SyncTableActionBase.beforeBuildingSourceSink() (第115-149行):
   a. 尝试获取已存在的表
      - 成功: 检查 schema 兼容性 (assertSchemaCompatible)
      - 失败 (SchemaRetrievalException): 使用已有表 schema 构建计算列
   b. 表不存在:
      - 从 CDC 源获取 schema (retrieveSchema())
      - 构建计算列
      - 创建 Paimon 表
   c. 获取 fileStoreTable 引用

3. SyncTableActionBase.buildSink() (第163-179行):
   - 创建 CdcSinkBuilder
   - 配置 EventParser、表、TypeMapping
   - 构建并注册 Flink Sink
```

**为什么在 beforeBuildingSourceSink 中先检查表是否存在?**
支持两种使用场景：
- **首次同步**: CDC 源 schema 自动创建 Paimon 表
- **增量同步**: 验证已有表与 CDC 源 schema 兼容，避免类型冲突

#### 库级同步 (SyncDatabaseActionBase)

**源码路径**: `paimon-flink/paimon-flink-cdc/src/main/java/org/apache/paimon/flink/action/cdc/SyncDatabaseActionBase.java`

**额外能力**:
- `mergeShards`: 分库分表合并（默认 true）
- `tablePrefix` / `tableSuffix`: 目标表名前缀/后缀
- `tableMapping`: 自定义表名映射 (`HashMap<String, String>`)
- `dbPrefix` / `dbSuffix`: 数据库名前缀/后缀映射
- `includingTables` / `excludingTables`: 正则表达式过滤同步的表
- `includingDbs` / `excludingDbs`: 正则表达式过滤同步的数据库
- `partitionKeyMultiple`: 不同表可以有不同的分区键
- `mode`: `COMBINED` (合并模式) 或 `DIVIDED` (分离模式)

**buildSink 的两种模式** (第242-271行):
- **COMBINED 模式**: 所有表共用一个多路复用的 Sink (`FlinkCdcSyncDatabaseSinkBuilder`)
- **DIVIDED 模式**: 每个表一个独立的 Sink

### 3.4 CDC 数据解析：RichCdcMultiplexRecord 到 CdcRecord

**CdcRecord** (`paimon-flink/paimon-flink-cdc/src/main/java/org/apache/paimon/flink/sink/cdc/CdcRecord.java`):

```java
public class CdcRecord implements Serializable {
    private RowKind kind;              // INSERT, UPDATE_BEFORE, UPDATE_AFTER, DELETE
    private final Map<String, String> data;  // 字段名 → 字符串值
}
```

**为什么使用 `Map<String, String>` 而非强类型?**
CDC 数据来自多种格式（Debezium JSON, Canal JSON, Maxwell JSON 等），使用字符串 map 作为中间表示可以统一处理不同格式的数据，类型转换推迟到写入 Paimon 时进行。

**RichCdcMultiplexRecord** (`paimon-flink/paimon-flink-cdc/src/main/java/org/apache/paimon/flink/sink/cdc/RichCdcMultiplexRecord.java`):

```java
public class RichCdcMultiplexRecord implements Serializable {
    private final String databaseName;   // 来源数据库名
    private final String tableName;      // 来源表名
    private final CdcSchema cdcSchema;   // schema 信息（字段列表 + 主键）
    private final CdcRecord cdcRecord;   // 数据记录
}
```

**为什么需要 RichCdcMultiplexRecord?**
在库级同步场景中，一个 Flink DataStream 可能包含来自多个表的数据。RichCdcMultiplexRecord 携带了数据库名、表名和 schema 信息，使得下游算子能够将数据路由到正确的 Paimon 表。

**数据流转换链**:
```
CDC Source → CdcSourceRecord
    → RecordParser.flatMap() → RichCdcMultiplexRecord (携带 schema + 数据)
        → EventParser → CdcRecord (纯数据)
            → Paimon Writer (类型转换 + 写入)
```

### 3.5 Schema 自动演进

**核心类**: `UpdatedDataFieldsProcessFunction`
**源码路径**: `paimon-flink/paimon-flink-cdc/src/main/java/org/apache/paimon/flink/sink/cdc/UpdatedDataFieldsProcessFunction.java`

**处理流程**:

```
1. 接收 CdcSchema（来自 CDC 源的最新 schema）
2. actualUpdatedDataFields(): 计算实际变更的字段（排除已存在且相同的字段）
3. extractSchemaChanges(): 生成 SchemaChange 列表
4. applySchemaChange(): 逐个应用 schema 变更
5. updateLatestFields(): 更新本地缓存的最新字段集合
```

**类型拓宽规则** (`UpdatedDataFieldsProcessFunctionBase.canConvert`，第173-200行):

```
支持的类型转换方向（基于 TypeMapping 配置）:

STRING 类型族内: CHAR → VARCHAR（长度只增不减）
BINARY 类型族内: BINARY → VARBINARY（长度只增不减）
INTEGER 类型族内: TINYINT → SMALLINT → INTEGER → BIGINT
FLOATING_POINT 类型族内: FLOAT → DOUBLE
DECIMAL 类型族内: 精度只增不减（scale 必须相同）
TIMESTAMP 类型族内: 精度只增不减

跨类型族: INTEGER → FLOATING_POINT → DECIMAL → STRING（依赖 TypeMapping 配置）
```

**为什么类型只拓宽不缩窄?**
缩窄可能导致数据丢失。例如从 BIGINT 缩窄到 INT 可能溢出。这是数据安全性的保障。

**applySchemaChange 的容错处理** (第106-161行):
- **AddColumn**: 捕获 `ColumnAlreadyExistException`，因为分库分表合并时多个源表可能同时添加相同列
- **UpdateColumnType**: 通过 `canConvert` 检查，不兼容的类型变更会抛出 `UnsupportedOperationException`

**SchemaMergingUtils** (`paimon-core/src/main/java/org/apache/paimon/schema/SchemaMergingUtils.java`):

该工具类用于将新 schema 与现有 schema 合并，核心方法 `merge()`:
- **RowType 合并**: 保留所有现有字段，同名字段递归合并类型，新字段追加
- **MapType/ArrayType/MultisetType**: 递归合并元素类型
- **DecimalType**: 取更大的 precision（scale 必须相同）
- **原子类型**: 检查安全转换（`DataTypeCasts.supportsCast`）

### 3.6 库级同步

#### 分库分表合并

**配置**: `--merge-shards true`（默认）

**为什么需要?**
业务系统常将数据按 hash 分散到多个数据库和表（如 `order_0`, `order_1`, ..., `order_99`），在数据湖中需要合并为一张表进行分析。

**实现机制**: `TableNameConverter` 类负责表名转换，支持：
- 分库分表名规范化（去除尾部数字后缀后合并同名表）
- 自定义表名映射 (`--table-mapping`)
- 前缀/后缀添加 (`--table-prefix`, `--table-suffix`)
- 数据库级别的前缀/后缀 (`--db-prefix`, `--db-suffix`)

#### 表过滤

```
--including-tables 'order|user|product'     # 正则匹配要同步的表
--excluding-tables 'tmp_.*|test_.*'         # 正则排除不同步的表
--including-dbs 'db_prod_.*'                # 正则匹配要同步的数据库
--excluding-dbs 'db_test_.*'               # 正则排除不同步的数据库
```

**为什么用正则表达式?**
正则表达式提供了最大的灵活性，可以匹配命名规范的表名模式。

#### 新表动态发现

在 COMBINED 模式下，通过 `CdcDynamicTableParsingProcessFunction` 处理新表:
1. 已知表的数据直接路由到对应的 Sink
2. 新表的数据通过 Side Output 路由到多路复用 Sink
3. Schema 变更通过另一个 Side Output 路由到 `MultiTableUpdatedDataFieldsProcessFunction`

---

## 4. Changelog 产生机制详解

### 4.1 ChangelogProducer 枚举与 3 种模式 + FirstRow 特殊机制

**源码位置**: `CoreOptions.ChangelogProducer` 枚举，`paimon-api/.../CoreOptions.java` 第3948-3976行

```java
public enum ChangelogProducer {
    NONE,            // 不产生 changelog
    INPUT,           // flush 时双写到 changelog 文件
    FULL_COMPACTION, // 全量 compaction 时通过比对产生 changelog
    LOOKUP           // 通过 lookup compaction 产生 changelog
}
```

**重要说明**: ChangelogProducer 枚举只有 4 个值（包括 NONE），但实际产生 changelog 的模式只有 3 种（INPUT/FULL_COMPACTION/LOOKUP）。FirstRow 不是 ChangelogProducer 的枚举值，而是一种 MergeEngine，它有自己独特的 changelog 产生机制（通过 `FirstRowMergeFunctionWrapper`），不依赖 ChangelogProducer 配置。

### 4.2 INPUT 模式

**源码位置**: `MergeTreeWriter.flushWriteBuffer()`，第209-249行

```java
private void flushWriteBuffer(boolean waitForLatestCompaction, boolean forcedFullCompaction)
        throws Exception {
    if (writeBuffer.size() > 0) {
        // 如果是 INPUT 模式，创建 changelog writer
        final RollingFileWriter<KeyValue, DataFileMeta> changelogWriter =
                changelogProducer == ChangelogProducer.INPUT
                        ? writerFactory.createRollingChangelogFileWriter(0)
                        : null;
        final RollingFileWriter<KeyValue, DataFileMeta> dataWriter =
                writerFactory.createRollingMergeTreeFileWriter(0, FileSource.APPEND);

        try {
            // 双写: 数据文件 + changelog 文件
            writeBuffer.forEach(
                    keyComparator,
                    mergeFunction,
                    changelogWriter == null ? null : changelogWriter::write,
                    dataWriter::write);
        } finally {
            writeBuffer.clear();
            if (changelogWriter != null) {
                changelogWriter.close();
            }
            dataWriter.close();
        }

        if (changelogWriter != null) {
            newFilesChangelog.addAll(changelogWriter.result());
        }
        // ...
    }
}
```

**为什么选择在 flush 时双写?**
INPUT 模式直接将输入的原始变更（INSERT/UPDATE/DELETE）记录到 changelog 文件中，无需任何额外计算。这是最低延迟的 changelog 产生方式。

**好处**:
- 零额外计算开销
- changelog 包含完整的输入变更明细
- 最低延迟

**限制**:
- changelog 内容取决于输入数据质量，如果输入没有完整的 -U/+U 对，changelog 也不完整
- 存储成本翻倍（数据文件 + changelog 文件）
- 不适用于 partial-update 和 aggregation 引擎（因为输入的变更不等于最终的变更）

### 4.3 FULL_COMPACTION 模式

**核心类**: `FullChangelogMergeFunctionWrapper`
**源码路径**: `paimon-core/src/main/java/org/apache/paimon/mergetree/compact/FullChangelogMergeFunctionWrapper.java`

**为什么需要 FULL_COMPACTION 模式?**
对于 partial-update 和 aggregation 引擎，输入数据不直接反映最终的行变更。只有在全量 compaction 时，通过比对最高层（最终合并结果）和本次 compaction 的结果，才能得出真正的变更。

**比对逻辑** (`getResult()` 方法，第97-126行):

```
情况 1: 有多条记录参与合并 (isInitialized == true)
    - topLevelKv == null (该 key 在最高层不存在)
        · 合并结果是 ADD → 产生 INSERT changelog
    - topLevelKv != null (该 key 在最高层存在)
        · 合并结果是 DELETE → 产生 DELETE changelog
        · 合并结果是 ADD 且值不同 → 产生 UPDATE_BEFORE + UPDATE_AFTER changelog
        · 合并结果是 ADD 且值相同 → 无 changelog（通过 valueEqualiser 比对）

情况 2: 只有一条记录 (isInitialized == false)
    - topLevelKv == null 且是 ADD → 产生 INSERT changelog
    - 其他情况 → 无 changelog
```

**topLevelKv 的含义**:
`maxLevel` 是 LSM-tree 的最高层，只有全量 compaction 才会向最高层写入文件。topLevelKv 表示该 key 在上一次全量 compaction 的最终结果，即"历史状态"。

**好处**:
- 能为 partial-update 和 aggregation 引擎产生正确的 changelog
- 支持行去重（`changelog-producer.row-deduplicate`），避免无效的更新 changelog

**限制**:
- 只有在全量 compaction 完成后才能产生 changelog，延迟较高
- 依赖 compaction 调度频率

### 4.4 LOOKUP 模式

**核心类**: `LookupChangelogMergeFunctionWrapper`
**源码路径**: `paimon-core/src/main/java/org/apache/paimon/mergetree/compact/LookupChangelogMergeFunctionWrapper.java`

**为什么需要 LOOKUP 模式?**
FULL_COMPACTION 模式延迟高（必须等全量 compaction），而 INPUT 模式不适用于复杂合并引擎。LOOKUP 模式在每次涉及 Level-0 文件的 compaction 中，通过查找历史值来产生 changelog，实现较低延迟的精确 changelog。

**核心流程** (`getResult()` 方法，第104-146行):

```
1. 从本次 compaction 的记录中，找到最高层级(非 Level-0)的记录 (pickHighLevel)
2. 检查是否包含 Level-0 记录 (containLevel0)

3. 如果没有高层级记录 → 通过 lookup 函数查找历史值
   - 如果启用了 Deletion Vector: 查找结果包含文件位置信息，通知 DV 维护器
   - 如果查找到历史值: 插入到合并函数中参与合并

4. 执行合并得到最终结果

5. 只有当存在 Level-0 记录且策略要求产生 changelog 时，设置 changelog:
   - before == null (新 key)  && after 是 ADD → INSERT
   - before 是 ADD && after 是 DELETE → DELETE
   - before 是 ADD && after 是 ADD && 值不同 → UPDATE_BEFORE + UPDATE_AFTER
   - before 是 ADD && after 是 ADD && 值相同 → 无 changelog (row deduplicate)
```

**好处**:
- 延迟低于 FULL_COMPACTION（每次涉及 Level-0 的 compaction 都产生 changelog）
- 支持 partial-update 和 aggregation 引擎
- 产生精确的完整 changelog

**限制**:
- 需要额外的 lookup 操作（读取历史数据），有 I/O 开销
- 需要更多内存（lookup cache）

### 4.5 FirstRow 模式

**核心类**: `FirstRowMergeFunctionWrapper`
**源码路径**: `paimon-core/src/main/java/org/apache/paimon/mergetree/compact/FirstRowMergeFunctionWrapper.java`

**为什么 FirstRow 有独特的 changelog 机制?**
FirstRow 引擎只保留每个 key 的第一条记录。它的 changelog 语义非常简单：只有新 key 的第一次出现才产生 INSERT changelog。

**核心逻辑** (`getResult()` 方法，第56-71行):

```java
public ChangelogResult getResult() {
    reusedResult.reset();
    KeyValue result = mergeFunction.getResult();

    // 如果包含高层级记录，说明这个 key 之前已经存在，无需产生 changelog
    if (mergeFunction.containsHighLevel) {
        reusedResult.setResult(result);
        return reusedResult;
    }

    // 检查 key 是否在历史数据中已存在（通过 Bloom Filter 或 lookup）
    if (contains.test(result.key())) {
        return reusedResult;  // 已存在，不输出
    }

    // 新 key，产生 changelog 并输出
    return reusedResult.setResult(result).addChangelog(result);
}
```

**好处**:
- 极低的 changelog 产生开销（只需检查 key 是否存在）
- 非常适合去重场景（如日志去重、消息去重）

**限制**:
- 只能产生 INSERT changelog，没有 UPDATE 和 DELETE
- 配合 `ignore-delete = true` 使用

### 4.6 各模式的权衡对比

| 维度 | NONE | INPUT | FULL_COMPACTION | LOOKUP | FirstRow |
|------|------|-------|-----------------|--------|----------|
| **延迟** | - | 最低（实时） | 最高（等 compaction） | 中等 | 低 |
| **资源开销** | 无 | 存储翻倍 | compaction 时计算 | lookup I/O + 内存 | Bloom Filter 内存 |
| **changelog 完整性** | 无 | 取决于输入 | 完整 | 完整 | 仅 INSERT |
| **适用合并引擎** | 所有 | deduplicate | 所有 | 所有 | first-row |
| **适用场景** | 不需要下游消费 | 简单写入 | 复杂引擎 + 低频消费 | 复杂引擎 + 准实时消费 | 去重 |

---

## 5. 跨分区更新

### 5.1 GlobalIndexAssigner 的全局索引维护

**核心类**: `GlobalIndexAssigner`
**源码路径**: `paimon-core/src/main/java/org/apache/paimon/crosspartition/GlobalIndexAssigner.java`

**为什么需要跨分区更新?**
在分区表中，如果一条记录的分区键发生变化（例如订单从"待付款"分区移到"已付款"分区），需要在旧分区删除旧记录并在新分区插入新记录。GlobalIndexAssigner 维护一个全局索引，记录每个主键当前所在的分区和 bucket，从而支持跨分区更新。

**好处**:
- 自动处理跨分区记录迁移，上游只需发送普通更新
- 基于 RocksDB 的高效索引查找
- 支持动态 bucket 分配

**核心状态**:

```java
RocksDBValueState<InternalRow, PositiveIntInt> keyIndex;
// key: 主键 (InternalRow)
// value: (partitionId, bucketId) (PositiveIntInt)
```

### 5.2 RocksDB 存储的 key 到 partition-bucket 映射

**初始化** (`open` 方法，第113-180行):

```java
this.stateFactory = new RocksDBStateFactory(path, rocksdbOptions, crossPartitionUpsertIndexTtl);
this.keyIndex = stateFactory.valueState(
    INDEX_NAME,
    new RowCompactedSerializer(keyType),     // key 序列化器
    new PositiveIntIntSerializer(),           // value 序列化器 (partId, bucket)
    lookupCacheRows);
```

**为什么使用 RocksDB?**
- 支持超大数据量（磁盘存储，不受 JVM 堆内存限制）
- 高效的 key-value 查找（基于 LSM-tree）
- 支持 Bulk Load（bootstrap 阶段批量加载）
- 支持 TTL 过期清理

**记录处理流程** (`processInput` 方法，第243-272行):

```
对于每条输入记录:
1. 提取 partition 和 primary key
2. 将 partition 映射为 partId (IDMapping)
3. 在 RocksDB 中查找 key:
   a. 找到 → 获取 (previousPartId, previousBucket)
      - previousPartId == 当前 partId → 直接写入旧 bucket (同分区更新)
      - previousPartId != 当前 partId → 跨分区更新
        · 调用 existingProcessor.processExists() 处理旧分区
        · 如果返回 true，处理新分区的记录 (processNewRecord)
   b. 未找到 → 新记录，分配新 bucket (processNewRecord)
```

### 5.3 ExistingProcessor 的不同策略

**源码路径**: `paimon-core/src/main/java/org/apache/paimon/crosspartition/ExistingProcessor.java`

**工厂方法** (`ExistingProcessor.create`，第59-75行):
根据 `MergeEngine` 创建不同的处理器：

| MergeEngine | ExistingProcessor | 行为 | 为什么 |
|-------------|------------------|------|--------|
| `DEDUPLICATE` | `DeleteExistingProcessor` | 先删旧分区记录，再写新分区 | 去重语义要求旧记录被删除 |
| `PARTIAL_UPDATE` | `UseOldExistingProcessor` | 将新记录写入旧分区（不迁移） | partial-update 需要在旧位置累积更新 |
| `AGGREGATE` | `UseOldExistingProcessor` | 将新记录写入旧分区（不迁移） | 聚合需要在旧位置累积 |
| `FIRST_ROW` | `SkipNewExistingProcessor` | 丢弃新记录 | first-row 只保留首条，后续忽略 |

#### DeleteExistingProcessor (Deduplicate 引擎)

```java
public boolean processExists(InternalRow newRow, BinaryRow previousPart, int previousBucket) {
    // 1. 构造一条指向旧分区的删除记录
    InternalRow retract = setPartition.apply(newRow, previousPart);
    retract.setRowKind(RowKind.DELETE);
    collector.accept(retract, previousBucket);

    // 2. 减少旧分区的 bucket 计数
    bucketAssigner.decrement(previousPart, previousBucket);

    // 3. 返回 true，表示需要继续处理新记录（写入新分区）
    return true;
}
```

#### UseOldExistingProcessor (Partial Update / Aggregate 引擎)

```java
public boolean processExists(InternalRow newRow, BinaryRow previousPart, int previousBucket) {
    // 将新记录的分区改为旧分区，写入旧位置
    InternalRow newValue = setPartition.apply(newRow, previousPart);
    collector.accept(newValue, previousBucket);

    // 返回 false，不处理新分区的记录
    return false;
}
```

**为什么 Partial Update 和 Aggregate 使用旧分区?**
这两种引擎需要在同一位置累积更新。如果每次分区键变化都迁移记录，会丢失之前累积的聚合/部分更新状态。

#### SkipNewExistingProcessor (FirstRow 引擎)

```java
public boolean processExists(InternalRow newRow, BinaryRow previousPart, int previousBucket) {
    return false;  // 直接丢弃，不做任何处理
}
```

### 5.4 Bootstrap 流程

**为什么需要 Bootstrap?**
GlobalIndexAssigner 使用 RocksDB 存储全局索引，但 RocksDB 是本地状态。当 Flink 作业重启或首次启动时，需要从 Paimon 表的历史数据中重建索引。

**Bootstrap 流程**:

```
1. open() 阶段: 设置 bootstrap = true，创建 bootstrapKeys (排序缓冲区) 和 bootstrapRecords (记录缓冲区)

2. bootstrapKey(value) 阶段 (第182-192行):
   - 从 IndexBootstrap 读取所有历史主键 → (partition, bucket)
   - 序列化为 KV 对存入排序缓冲区

3. processInput(value) 阶段 (bootstrap=true 时):
   - 将新输入的记录缓存到 bootstrapRecords

4. endBootstrap(isEndInput) 阶段 (第198-241行):
   a. 将 bootstrapKeys 排序后通过 RocksDB Bulk Load 批量加载
      - 为什么用 Bulk Load: 比逐条 put 快一个数量级
      - 异常处理: 如果 Bulk Load 失败，提示可能有重复数据
   b. 如果 isEndInput=true 且 RocksDB 为空 → 使用 bulkLoadBootstrapRecords 优化
      - 为什么: 首次批量写入时，可以跳过 RocksDB 查找，直接排序分配 bucket
   c. 否则 → 逐条回放缓存的记录 (processInput)
```

**IndexBootstrap** (`paimon-core/src/main/java/org/apache/paimon/crosspartition/IndexBootstrap.java`):

```java
// 从 Paimon 表扫描所有历史主键
ReadBuilder readBuilder = table.copy(...)
    .newReadBuilder()
    .withProjection(keyProjection);  // 只读取主键列，减少 I/O

DataTableScan tableScan = (DataTableScan) readBuilder.newScan();
List<Split> splits = tableScan
    .withBucketFilter(bucket -> bucket % numAssigners == assignId)  // 按 assignId 分片
    .withLevelFilter(level -> true)  // 读取所有层级
    .plan().splits();
```

**为什么使用 `withBucketFilter`?**
在多并行度场景下，不同的 assigner 实例各自负责一部分 bucket，通过 bucket 过滤减少每个实例需要加载的数据量。

---

## 6. Schema Evolution 详解

### 6.1 SchemaManager.commitChanges 的完整流程

**源码路径**: `paimon-core/src/main/java/org/apache/paimon/schema/SchemaManager.java`

**commitChanges 方法** (第252-280行):

```java
public TableSchema commitChanges(List<SchemaChange> changes) {
    SnapshotManager snapshotManager = new SnapshotManager(fileIO, tableRoot, branch, null, null);
    LazyField<Boolean> hasSnapshots = new LazyField<>(() -> snapshotManager.latestSnapshot() != null);

    while (true) {
        // 1. 获取最新 schema
        TableSchema oldTableSchema = latest().orElseThrow(...);

        // 2. 生成新 schema
        TableSchema newTableSchema = generateTableSchema(
            oldTableSchema, changes, hasSnapshots, lazyIdentifier);

        try {
            // 3. 尝试提交（基于文件系统的乐观锁）
            boolean success = commit(newTableSchema);
            if (success) {
                return newTableSchema;
            }
            // 失败则重试（其他并发写入者已更新 schema）
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

**为什么使用 while(true) + 乐观锁?**
多个写入者可能同时修改 schema（例如多个 CDC 同步任务同时添加新列）。使用文件系统级别的原子操作（rename/create）作为锁，失败时重试保证最终一致性。

**generateTableSchema 方法** (第282-586行):

逐个处理 SchemaChange，返回新的 TableSchema（schema id = old id + 1）。

### 6.2 所有 SchemaChange 类型

**源码路径**: `paimon-api/src/main/java/org/apache/paimon/schema/SchemaChange.java`

| SchemaChange 类型 | 描述 | 对有快照的表的限制 |
|-------------------|------|-------------------|
| `SetOption` | 设置表选项 | 检查 immutable 选项不可变更 |
| `RemoveOption` | 移除表选项 | 检查不可移除的选项 |
| `UpdateComment` | 更新表注释 | 无限制 |
| `AddColumn` | 添加列 | 新列必须 nullable |
| `RenameColumn` | 重命名列 | 不能重命名分区键 |
| `DropColumn` | 删除列 | 不能删除分区键、主键、bucket-key |
| `UpdateColumnType` | 更新列类型 | 不能更新分区键/主键，必须类型兼容 |
| `UpdateColumnNullability` | 更新列的可空性 | nullable 列不能改为 non-nullable(可配置) |
| `UpdateColumnComment` | 更新列注释 | 无限制 |
| `UpdateColumnPosition` | 更新列位置 (FIRST/AFTER/BEFORE/LAST) | 无限制 |
| `UpdateColumnDefaultValue` | 更新列默认值 | 需要通过默认值校验 |
| `DropPrimaryKey` | 删除主键 | 只有空表可以执行 |

**为什么新列必须是 nullable?**
已有数据文件中不包含新列，读取时该列自动填充 null。如果新列不允许为 null，将导致旧数据文件读取失败。

**重命名列时的选项同步** (`applyRenameColumnsToOptions`，第745-800行):
当列重命名时，需要同步更新引用该列的选项：
- `bucket-key`
- `sequence.field`
- `fields.{field_name}.aggregate-function`
- `fields.{field_name}.ignore-retract`
- `fields.{field_name}.distinct`
- `fields.{field_name}.list-agg-delimiter`
- `fields.{seq_field}.sequence-group`（key 和 value 中的字段名都需要更新）

### 6.3 CDC 场景下的自动 Schema 演进

**核心类**: `UpdatedDataFieldsProcessFunction` (表级) / `MultiTableUpdatedDataFieldsProcessFunction` (库级)

**自动 Schema 演进流程**:

```
1. CDC 源产生 schema 变更事件
2. RecordParser 将变更转为 CdcSchema / RichCdcMultiplexRecord
3. EventParser 提取 schema 差异
4. UpdatedDataFieldsProcessFunction 处理差异:
   a. 计算实际变更的字段 (actualUpdatedDataFields)
   b. 生成 SchemaChange 列表 (extractSchemaChanges):
      - 新字段 → SchemaChange.AddColumn
      - 字段类型变更 → SchemaChange.UpdateColumnType
      - 字段注释变更 → SchemaChange.UpdateColumnComment
      - 表注释变更 → SchemaChange.UpdateComment
   c. 逐个应用 SchemaChange (applySchemaChange):
      - AddColumn: 调用 catalog.alterTable()
      - UpdateColumnType: 检查类型兼容性后调用 catalog.alterTable()
5. SchemaManager.commitChanges() 原子提交
```

**为什么 parallelism 必须为 1?**
Schema 变更必须串行执行，避免并发修改导致的冲突。虽然 SchemaManager 内部有乐观锁重试，但并行度为 1 可以减少不必要的冲突和重试。

---

## 7. 实战示例

### 7.1 Partial Update 建表和使用示例

```sql
-- 场景：多源汇聚的用户画像表
CREATE TABLE user_profile (
    user_id BIGINT PRIMARY KEY NOT ENFORCED,
    -- 基本信息（来自注册系统，payment_ts 控制版本）
    name STRING,
    gender STRING,
    register_ts BIGINT,
    -- 消费信息（来自支付系统，payment_ts 控制版本）
    total_spent DECIMAL(10,2),
    order_count INT,
    payment_ts BIGINT,
    -- 行为信息（来自日志系统，behavior_ts 控制版本）
    last_login_time TIMESTAMP,
    login_count INT,
    behavior_ts BIGINT
) WITH (
    'merge-engine' = 'partial-update',
    'fields.register_ts.sequence-group' = 'name,gender',
    'fields.payment_ts.sequence-group' = 'total_spent,order_count',
    'fields.behavior_ts.sequence-group' = 'last_login_time,login_count',
    'fields.total_spent.aggregate-function' = 'sum',
    'fields.order_count.aggregate-function' = 'sum',
    'fields.login_count.aggregate-function' = 'sum',
    'partial-update.remove-record-on-sequence-group' = 'register_ts'
);
```

**配置解读**:
- 三个独立的 sequence group，各自控制版本
- `total_spent`, `order_count`, `login_count` 使用 sum 聚合
- 当注册系统发送 DELETE 记录时，删除整行

### 7.2 Aggregation 建表和使用示例

```sql
-- 场景：实时指标聚合表
CREATE TABLE page_view_stats (
    page_id STRING,
    dt STRING,
    PRIMARY KEY (page_id, dt) NOT ENFORCED,
    pv BIGINT,
    uv BIGINT,
    total_duration BIGINT,
    max_duration BIGINT,
    unique_users VARBINARY
) PARTITIONED BY (dt) WITH (
    'merge-engine' = 'aggregation',
    'fields.pv.aggregate-function' = 'sum',
    'fields.uv.aggregate-function' = 'sum',
    'fields.total_duration.aggregate-function' = 'sum',
    'fields.max_duration.aggregate-function' = 'max',
    'fields.unique_users.aggregate-function' = 'roaring_bitmap'
);
```

### 7.3 MySQL CDC 同步到 Paimon 的完整配置

#### 表级同步

```bash
<FLINK_HOME>/bin/flink run \
    -c org.apache.paimon.flink.action.FlinkActions \
    paimon-flink-action.jar \
    mysql_sync_table \
    --warehouse hdfs:///paimon/warehouse \
    --database paimon_db \
    --table orders \
    --primary-keys order_id \
    --partition-keys dt \
    --mysql-conf hostname=mysql-host \
    --mysql-conf port=3306 \
    --mysql-conf username=root \
    --mysql-conf password=secret \
    --mysql-conf database-name=source_db \
    --mysql-conf table-name='orders' \
    --table-conf merge-engine=partial-update \
    --table-conf changelog-producer=lookup \
    --table-conf bucket=4 \
    --table-conf snapshot.time-retained=24h
```

#### 库级同步（分库分表合并）

```bash
<FLINK_HOME>/bin/flink run \
    -c org.apache.paimon.flink.action.FlinkActions \
    paimon-flink-action.jar \
    mysql_sync_database \
    --warehouse hdfs:///paimon/warehouse \
    --database paimon_db \
    --mysql-conf hostname=mysql-host \
    --mysql-conf port=3306 \
    --mysql-conf username=root \
    --mysql-conf password=secret \
    --mysql-conf database-name='source_db_\d+' \
    --merge-shards true \
    --including-tables 'order.*|user.*' \
    --excluding-tables 'tmp_.*' \
    --table-prefix '' \
    --table-suffix '' \
    --table-conf changelog-producer=input \
    --table-conf bucket=-1 \
    --type-mapping to-nullable,to-string
```

**关键参数说明**:
- `--merge-shards true`: 合并分库分表（如 `order_0`, `order_1` 合并为 `order`）
- `--including-tables`: 正则过滤要同步的表
- `--type-mapping to-nullable`: MySQL NOT NULL 列在 Paimon 中转为 nullable（避免 schema 演进时报错）
- `--type-mapping to-string`: 不支持的类型转为 STRING

### 7.4 跨分区更新的配置和注意事项

```sql
-- 启用跨分区更新的分区表
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY NOT ENFORCED,
    order_status STRING,
    amount DECIMAL(10,2),
    dt STRING
) PARTITIONED BY (dt) WITH (
    'merge-engine' = 'deduplicate',
    'bucket' = '-1',                          -- 必须使用动态 bucket
    'changelog-producer' = 'lookup',
    'cross-partition-upsert.index-ttl' = '7d' -- 索引 TTL
);
```

**注意事项**:

1. **必须使用动态 bucket (`bucket = -1`)**: 跨分区更新依赖 GlobalIndexAssigner，只有动态 bucket 模式才会启用此组件。

2. **只支持单写入**: GlobalIndexAssigner 使用本地 RocksDB 状态，不能有多个 Flink 作业同时写入同一张表。源码中对此有明确的错误提示（第219-224行）。

3. **合并引擎决定迁移策略**:
   - `deduplicate`: 删旧写新（跨分区迁移）
   - `partial-update` / `aggregation`: 写入旧分区（不迁移，保持聚合状态）
   - `first-row`: 丢弃新记录（首行已存在则忽略）

4. **索引 TTL**: `cross-partition-upsert.index-ttl` 控制 RocksDB 中索引条目的过期时间。过期后的 key 再次出现会被视为新记录。根据业务需求设置合理的 TTL，避免索引无限增长。

5. **Bootstrap 性能**: 首次启动或重启时，需要从 Paimon 表扫描所有历史主键重建索引。数据量大时 bootstrap 可能耗时较长。可以通过 `cross-partition-upsert.bootstrap-parallelism` 控制 bootstrap 的并行度。

6. **全局索引存储路径**: 可以通过 `global-index.external-path` 配置外部存储路径，默认存储在 `<table-root-directory>/index/` 下。

---

> **文档版本**: 基于 Paimon 1.5-SNAPSHOT 源码
> **最后更新**: 2026-04-21
