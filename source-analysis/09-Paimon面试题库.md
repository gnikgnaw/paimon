# Apache Paimon 源码级面试题库

> 本文档从面试官视角整理了 Apache Paimon (1.5-SNAPSHOT) 核心知识点，涵盖 LSM Merge-Tree、Snapshot/Manifest 管理、Merge 引擎、Flink 集成、Deletion Vectors、查询优化和运维调优等关键领域。所有答案均基于源码级分析，附有精确的源码路径和代码片段。
>
> **难度标记**: 🟢 基础 | 🟡 中级 | 🟠 高级 | 🔴 专家

---

## 目录（按题目导航）

### 一、LSM Merge-Tree 核心原理
- [1.1 Paimon 为什么选择 LSM-Tree？与 Iceberg 的不可变文件模型有何区别？](#q-1-1) 🟢
- [1.2 Levels 的数据结构设计：Level0 用 TreeSet，高层用 SortedRun，为什么？](#q-1-2) 🟡
- [1.3 Universal Compaction 的三级选择策略是如何工作的？](#q-1-3) 🟠
- [1.4 反压机制 shouldWaitForLatestCompaction 是如何实现的？](#q-1-4) 🟡
- [1.5 IntervalPartition 的 Section 划分算法原理是什么？](#q-1-5) 🟠

### 二、Snapshot 与 Manifest 管理
- [2.1 Paimon Snapshot 的 base/delta/changelog manifest list 三分法设计有什么含义？](#q-2-1) 🟡
- [2.2 一次 commit 为什么可能产生两个 Snapshot（APPEND + COMPACT）？](#q-2-2) 🟠
- [2.3 Manifest 合并机制是如何工作的？](#q-2-3) 🟠
- [2.4 commitIdentifier 的去重作用是什么？](#q-2-4) 🟡

### 三、Merge 引擎
- [3.1 四种 MergeFunction 的实现原理和适用场景分别是什么？](#q-3-1) 🟡
- [3.2 PartialUpdate 的 Sequence Group 机制是如何工作的？](#q-3-2) 🟠
- [3.3 LookupMergeFunction 的包装设计有什么巧妙之处？](#q-3-3) 🔴
- [3.4 Changelog 产生的四种模式有什么区别？](#q-3-4) 🟠
- [3.5 Paimon 的 20+ 内置聚合函数体系是如何组织的？](#q-3-5) 🟡

### 四、Flink 集成
- [4.1 Flink Sink 的算子拓扑（Writer → Committer）是如何设计的？](#q-4-1) 🟡
- [4.2 Checkpoint 两阶段提交的 Exactly-Once 保证是如何实现的？](#q-4-2) 🟠
- [4.3 CDC 同步的完整链路和 Schema 自动演进是如何工作的？](#q-4-3) 🟠
- [4.4 Lookup Join 的缓存和增量刷新机制是什么？](#q-4-4) 🟡

### 五、Deletion Vectors 与文件索引
- [5.1 DV 的 RoaringBitmap 实现和 V1/V2 格式差异是什么？](#q-5-1) 🟡
- [5.2 DV 如何让文件变为 rawConvertible？](#q-5-2) 🟠
- [5.3 四种文件索引的原理和选择策略是什么？](#q-5-3) 🟠
- [5.4 Predicate 体系如何连接查询和索引？](#q-5-4) 🟡

### 六、查询优化
- [6.1 多层文件过滤（分区 → Manifest → 文件 → 行）是如何工作的？](#q-6-1) 🟡
- [6.2 Z-Order/Hilbert 排序如何改善查询性能？](#q-6-2) 🟠
- [6.3 LookupLevels 的点查优化机制是什么？](#q-6-3) 🔴

### 七、运维与性能调优
- [7.1 Compaction 参数调优策略有哪些？](#q-7-1) 🟡
- [7.2 Bucket 选择对性能有什么影响？](#q-7-2) 🟢
- [7.3 小文件问题的成因和治理方案是什么？](#q-7-3) 🟡
- [7.4 Snapshot 过期的多重保护机制是什么？](#q-7-4) 🟠

### 八、Paimon vs Iceberg 对比
- [8.1 存储模型的根本差异是什么？](#q-8-1) 🟢
- [8.2 流式更新能力有什么本质区别？](#q-8-2) 🟡
- [8.3 小文件治理理念有什么不同？](#q-8-3) 🟡

---

## 一、LSM Merge-Tree 核心原理

<a id="q-1-1"></a>
### 1.1 Paimon 为什么选择 LSM-Tree？与 Iceberg 的不可变文件模型有何区别？

**🟢 基础**

**核心答案：**

Paimon 选择 LSM-Tree 的根本原因是它需要支持**流式实时更新**场景。LSM-Tree 将随机写转化为顺序写，所有新数据先写入内存的 Write Buffer（`SortBufferWriteBuffer`），buffer 满后 flush 为 Level0 文件，然后通过后台 Compaction 逐步合并到高层。这种架构天然适合高频 upsert 操作。

相比之下，Iceberg 采用的是 Copy-on-Write（COW）或 Merge-on-Read（MOR）的不可变文件模型。Iceberg 每次更新要么重写整个文件（COW），要么产生 Delete File 在读时合并（MOR）。对于高频更新场景，Iceberg 的 COW 写放大严重，MOR 则带来读放大。Paimon 通过 LSM-Tree 的分层合并，在写放大和读放大之间取得了更好的平衡。

从源码看，Paimon 的主键表使用 `KeyValueFileStore`，其核心数据结构就是 `Levels`（LSM-Tree 的分层管理），而 Append-Only 表使用 `AppendOnlyFileStore`，不涉及 LSM 逻辑。

**源码证据：**

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/Levels.java:39-46
public class Levels {
    private final Comparator<InternalRow> keyComparator;
    private final TreeSet<DataFileMeta> level0;  // Level0: 无序，按 sequence number 排列
    private final List<SortedRun> levels;         // Level1+: 每层是一个有序的 SortedRun
    // ...
}
```

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/MergeTreeWriter.java:164-174
@Override
public void write(KeyValue kv) throws Exception {
    long sequenceNumber = newSequenceNumber();
    boolean success = writeBuffer.put(sequenceNumber, kv.valueKind(), kv.key(), kv.value());
    if (!success) {
        flushWriteBuffer(false, false);  // buffer 满了，flush 到 Level0
        success = writeBuffer.put(sequenceNumber, kv.valueKind(), kv.key(), kv.value());
        // ...
    }
}
```

**面试口述建议：**

> "Paimon 选择 LSM-Tree 是因为它定位于流式数据湖，核心需求是支持高频实时更新。LSM-Tree 把随机写转化为顺序写，数据先写入内存 Write Buffer，满了 flush 成 Level0 文件，再通过后台 Compaction 合并。而 Iceberg 是不可变文件模型，更新要么重写文件要么产生删除文件。Paimon 的 LSM 架构在写放大和读放大之间做了更好的 trade-off，特别适合 CDC 实时入湖这种场景。"

---

<a id="q-1-2"></a>
### 1.2 Levels 的数据结构设计：Level0 用 TreeSet，高层用 SortedRun，为什么？

**🟡 中级**

**核心答案：**

`Levels` 类对 Level0 和高层级采用了不同的数据结构，这反映了它们截然不同的语义：

**Level0 使用 `TreeSet<DataFileMeta>`**：Level0 是从 Write Buffer flush 出来的文件，每个文件内部有序但**文件之间键范围可能重叠**。TreeSet 的排序规则是按 `maxSequenceNumber` 降序排列，确保最新的文件排在最前面。当多个文件的 maxSequenceNumber 相同时（多 Job 并发写同一个 merge tree 的场景），会依次比较 minSequenceNumber、creationTime、fileName 来保证 TreeSet 的唯一性。

**高层级使用 `List<SortedRun>`**：Level1 及以上的每一层是一个 `SortedRun`，即**一组按 key 排序且键范围互不重叠的文件列表**。SortedRun 内部通过 `validate()` 方法严格校验不变量：`files[i].maxKey < files[i+1].minKey`。高层级的这种设计保证了高效的 key 范围查找——可以直接通过二分查找定位目标文件。

**源码证据：**

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/Levels.java:59-83
this.level0 =
    new TreeSet<>(
        (a, b) -> {
            if (a.maxSequenceNumber() != b.maxSequenceNumber()) {
                // 按 maxSequenceNumber 降序: 新数据在前
                return Long.compare(b.maxSequenceNumber(), a.maxSequenceNumber());
            } else {
                // 多 Job 并发写场景处理: 避免 TreeSet "去重"
                int minSeqCompare = Long.compare(a.minSequenceNumber(), b.minSequenceNumber());
                if (minSeqCompare != 0) return minSeqCompare;
                int timeCompare = a.creationTime().compareTo(b.creationTime());
                if (timeCompare != 0) return timeCompare;
                return a.fileName().compareTo(b.fileName());  // 最终兜底
            }
        });
```

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/SortedRun.java:88-94
public void validate(Comparator<InternalRow> comparator) {
    for (int i = 1; i < files.size(); i++) {
        Preconditions.checkState(
                comparator.compare(files.get(i).minKey(), files.get(i - 1).maxKey()) > 0,
                "SortedRun is not sorted and may contain overlapping key intervals. This is a bug.");
    }
}
```

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/Levels.java:128-135
public int numberOfSortedRuns() {
    int numberOfSortedRuns = level0.size();  // Level0 每个文件算一个 SortedRun
    for (SortedRun run : levels) {
        if (run.nonEmpty()) {
            numberOfSortedRuns++;  // 高层每层算一个 SortedRun
        }
    }
    return numberOfSortedRuns;
}
```

**面试口述建议：**

> "Levels 对 Level0 和高层级用了不同的数据结构。Level0 用 TreeSet，因为 flush 出来的文件之间键范围可能重叠，TreeSet 按 sequence number 降序排列保证最新数据优先。高层级用 SortedRun，也就是一组键范围不重叠的有序文件列表，这样可以做高效的二分查找。一个关键细节是 Level0 的 TreeSet 比较器需要处理多 Job 并发写的情况——当两个文件的 maxSequenceNumber 相同时，会继续比较 minSequenceNumber、创建时间和文件名来保证唯一性。numberOfSortedRuns 方法体现了核心思想：Level0 每个文件是一个独立的 SortedRun，高层每个层级是一个 SortedRun。"

---

<a id="q-1-3"></a>
### 1.3 Universal Compaction 的三级选择策略是如何工作的？

**🟠 高级**

**核心答案：**

Paimon 的 `UniversalCompaction` 参考了 RocksDB 的 Universal Compaction 策略，通过三级优先级来选择需要合并的文件：

**第 0 级 — Early Full Compaction（可选）**：如果配置了 `EarlyFullCompaction`，会先检查是否满足全量合并的触发条件，包括三个子条件：全量合并时间间隔（`compaction.optimization-interval`）、总数据大小阈值（`compaction.total-size-threshold`）、增量数据大小阈值（`compaction.incremental-size-threshold`）。

**第 1 级 — Size Amplification（空间放大检查）**：当 SortedRun 数量达到 `num-sorted-run.compaction-trigger` 时，计算除最底层外所有 run 的总大小与最底层 run 大小的比值。如果 `candidateSize * 100 > maxSizeAmp * earliestRunSize`，触发全量合并。这控制了空间放大倍数。

**第 2 级 — Size Ratio（大小比例检查）**：从最新的 SortedRun 开始，累积大小逐步向旧的 run 扩展。如果累积大小乘以 `(100 + sizeRatio) / 100` 小于下一个 run 的大小，停止扩展；否则继续包含更多 run。当选择了超过 1 个 run 时，触发合并。

**第 3 级 — File Num（文件数量兜底）**：如果 `runs.size() > numRunCompactionTrigger`，以 `runs.size() - numRunCompactionTrigger + 1` 作为候选数量，使用 Size Ratio 的逻辑选择合并范围。

特别值得注意的是，当需要 Lookup（如 changelog-producer=lookup）时，策略会被 `ForceUpLevel0Compaction` 包装，确保 Level0 文件被强制上推到高层，因为 Lookup 需要从高层 SST 文件中查找历史值。

**源码证据：**

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/compact/UniversalCompaction.java:67-107
@Override
public Optional<CompactUnit> pick(int numLevels, List<LevelSortedRun> runs) {
    int maxLevel = numLevels - 1;
    // 0 try full compaction by trigger
    if (earlyFullCompact != null) {
        Optional<CompactUnit> unit = earlyFullCompact.tryFullCompact(numLevels, runs);
        if (unit.isPresent()) return unit;
    }
    // 1 checking for reducing size amplification
    CompactUnit unit = pickForSizeAmp(maxLevel, runs);
    if (unit != null) return Optional.of(unit);
    // 2 checking for size ratio
    unit = pickForSizeRatio(maxLevel, runs);
    if (unit != null) return Optional.of(unit);
    // 3 checking for file num
    if (runs.size() > numRunCompactionTrigger) {
        int candidateCount = runs.size() - numRunCompactionTrigger + 1;
        return Optional.ofNullable(pickForSizeRatio(maxLevel, runs, candidateCount));
    }
    return Optional.empty();
}
```

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/compact/UniversalCompaction.java:125-147
CompactUnit pickForSizeAmp(int maxLevel, List<LevelSortedRun> runs) {
    if (runs.size() < numRunCompactionTrigger) return null;
    long candidateSize = runs.subList(0, runs.size() - 1).stream()
            .map(LevelSortedRun::run).mapToLong(SortedRun::totalSize).sum();
    long earliestRunSize = runs.get(runs.size() - 1).run().totalSize();
    // size amplification = percentage of additional size
    if (candidateSize * 100 > maxSizeAmp * earliestRunSize) {
        return CompactUnit.fromLevelRuns(maxLevel, runs);  // 全量合并
    }
    return null;
}
```

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/compact/UniversalCompaction.java:163-182
public CompactUnit pickForSizeRatio(
        int maxLevel, List<LevelSortedRun> runs, int candidateCount, boolean forcePick) {
    long candidateSize = candidateSize(runs, candidateCount);
    for (int i = candidateCount; i < runs.size(); i++) {
        LevelSortedRun next = runs.get(i);
        if (candidateSize * (100.0 + sizeRatio + ratioForOffPeak()) / 100.0
                < next.run().totalSize()) {
            break;  // 下一个 run 太大，停止扩展
        }
        candidateSize += next.run().totalSize();
        candidateCount++;
    }
    if (forcePick || candidateCount > 1) {
        return createUnit(runs, maxLevel, candidateCount);
    }
    return null;
}
```

**面试口述建议：**

> "Paimon 的 Universal Compaction 有三级优先策略。第一级检查空间放大——所有非底层 run 的总大小与底层 run 的比值，超过阈值就全量合并。第二级检查大小比例——从最新的 run 开始累积，如果累积大小乘以 (1 + sizeRatio) 还小于下一个 run，就停止。第三级是文件数量兜底——超过触发数量就强制选几个 run 合并。这套策略参考了 RocksDB 的设计，目标是降低写放大。另外还有一个第 0 级的 EarlyFullCompaction，可以按时间间隔或数据量触发全量合并。"

---

<a id="q-1-4"></a>
### 1.4 反压机制 shouldWaitForLatestCompaction 是如何实现的？

**🟡 中级**

**核心答案：**

Paimon 通过 `MergeTreeCompactManager` 中的两个方法实现写入反压，核心思想是当 SortedRun 数量超过阈值时，阻塞写入等待 Compaction 完成：

**`shouldWaitForLatestCompaction()`**：当 `levels.numberOfSortedRuns() > numSortedRunStopTrigger` 时返回 true。这个方法在每次 flush 后被检查——如果 SortedRun 太多，说明 Compaction 跟不上写入速度，此时写入线程会阻塞等待当前 Compaction 任务完成。

**`shouldWaitForPreparingCheckpoint()`**：当 `levels.numberOfSortedRuns() > numSortedRunStopTrigger + 1` 时返回 true。这是一个更宽松的阈值（+1），用于 checkpoint 准备阶段。如果此时 SortedRun 也超标，说明积压更严重。

`numSortedRunStopTrigger` 的默认值通常比 `numSortedRunCompactionTrigger` 大，形成一个"软限制触发 Compaction + 硬限制阻塞写入"的双层控制。这种设计避免了 SortedRun 无限增长导致读放大失控。

**源码证据：**

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/compact/MergeTreeCompactManager.java:109-117
@Override
public boolean shouldWaitForLatestCompaction() {
    return levels.numberOfSortedRuns() > numSortedRunStopTrigger;
}

@Override
public boolean shouldWaitForPreparingCheckpoint() {
    // cast to long to avoid Numeric overflow
    return levels.numberOfSortedRuns() > (long) numSortedRunStopTrigger + 1;
}
```

**面试口述建议：**

> "Paimon 用 SortedRun 数量作为反压指标。当 SortedRun 数量超过 num-sorted-run.stop-trigger 时，shouldWaitForLatestCompaction 返回 true，写入线程会阻塞等待 Compaction 完成。这是一个硬限制，防止读放大失控。还有一个 shouldWaitForPreparingCheckpoint 用于 checkpoint 阶段，阈值更宽松一点。整体形成了 '软触发 Compaction + 硬限制阻塞写入' 的双层控制。"

---

<a id="q-1-5"></a>
### 1.5 IntervalPartition 的 Section 划分算法原理是什么？

**🟠 高级**

**核心答案：**

`IntervalPartition` 算法的目标是将一组可能键范围重叠的文件划分为**最少数量**的 SortedRun。它分为两层逻辑：

**外层 — Section 划分**：将文件按 minKey 排序后，遍历并维护一个当前 Section 的右边界 `bound`。当一个新文件的 minKey 大于当前 bound 时，说明与当前 Section 没有重叠，结束当前 Section 并开始新的。这样外层产生多个键范围互不重叠的 Section，可以独立处理。

**内层 — 贪心分配 SortedRun**：在每个 Section 内部，使用**优先队列**实现贪心算法。优先队列按每个 SortedRun 的最后一个文件的 maxKey 排序（最小堆）。对于每个新文件，从优先队列中取出 maxKey 最小的 run，如果该文件的 minKey 大于 run 的 maxKey，说明不重叠，可以追加到该 run 中；否则创建新 run。这本质上是经典的**区间图着色问题**（Interval Graph Coloring），贪心策略保证了使用最少的 SortedRun 数量。

**源码证据：**

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/compact/IntervalPartition.java:67-91
public List<List<SortedRun>> partition() {
    List<List<SortedRun>> result = new ArrayList<>();
    List<DataFileMeta> section = new ArrayList<>();
    BinaryRow bound = null;
    for (DataFileMeta meta : files) {
        if (!section.isEmpty() && keyComparator.compare(meta.minKey(), bound) > 0) {
            // 新文件的 minKey > 当前 Section 的右边界，断开 Section
            result.add(partition(section));
            section.clear();
            bound = null;
        }
        section.add(meta);
        if (bound == null || keyComparator.compare(meta.maxKey(), bound) > 0) {
            bound = meta.maxKey();  // 更新右边界
        }
    }
    if (!section.isEmpty()) result.add(partition(section));
    return result;
}
```

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/mergetree/compact/IntervalPartition.java:93-125
private List<SortedRun> partition(List<DataFileMeta> metas) {
    PriorityQueue<List<DataFileMeta>> queue = new PriorityQueue<>(
            (o1, o2) -> keyComparator.compare(
                    o1.get(o1.size() - 1).maxKey(),
                    o2.get(o2.size() - 1).maxKey()));  // 按最后文件的 maxKey 排序
    List<DataFileMeta> firstRun = new ArrayList<>();
    firstRun.add(metas.get(0));
    queue.add(firstRun);
    for (int i = 1; i < metas.size(); i++) {
        DataFileMeta meta = metas.get(i);
        List<DataFileMeta> top = queue.poll();
        if (keyComparator.compare(meta.minKey(), top.get(top.size() - 1).maxKey()) > 0) {
            top.add(meta);  // 不重叠，追加到已有 run
        } else {
            List<DataFileMeta> newRun = new ArrayList<>();
            newRun.add(meta);
            queue.add(newRun);  // 重叠，创建新 run
        }
        queue.add(top);
    }
    return queue.stream().map(SortedRun::fromSorted).collect(Collectors.toList());
}
```

**面试口述建议：**

> "IntervalPartition 分两层。外层按键范围将文件切成互不重叠的 Section，这样每个 Section 可以独立合并。内层在每个 Section 内用贪心算法将文件分配到最少数量的 SortedRun 中——用优先队列维护每个 run 的 maxKey，每个新文件优先追加到 maxKey 最小的不重叠 run 里。这本质是区间图着色问题的最优贪心解。Section 的好处是减少了每次 Compaction 需要合并的数据量。"

---

## 二、Snapshot 与 Manifest 管理

<a id="q-2-1"></a>
### 2.1 Paimon Snapshot 的 base/delta/changelog manifest list 三分法设计有什么含义？

**🟡 中级**

**核心答案：**

Paimon 的每个 Snapshot 包含三种 ManifestList，服务于不同的使用场景：

**baseManifestList**：记录从前一个 Snapshot 继承并合并后的全量文件变更清单。它代表了到这个 Snapshot 为止，表中"应该存在的所有文件"的完整描述（以 ADD/DELETE 条目形式）。对于全表扫描，只需要读 base manifest list。

**deltaManifestList**：仅记录**本次 Snapshot 新增的文件变更**。它是 base 的增量。设计这个增量层的目的有二：一是加速快照过期——过期时只需读 delta 来判断哪些文件是本快照引入的；二是支持流式读取——消费者只需读连续 Snapshot 的 delta 就能获得增量变更，而不必做两个 base 的 diff。

**changelogManifestList**：仅记录本次 Snapshot 产生的 changelog 文件。它与 delta 的区别在于：delta 记录的是表文件的增减，changelog 记录的是变更日志（用于下游 CDC 消费）。只有在 `changelog-producer` 不为 NONE 时才会有值。

**源码证据：**

```java
// 源码: paimon-api/src/main/java/org/apache/paimon/Snapshot.java:84-115
// a manifest list recording all changes from the previous snapshots
@JsonProperty(FIELD_BASE_MANIFEST_LIST)
protected final String baseManifestList;

// a manifest list recording all new changes occurred in this snapshot
// for faster expire and streaming reads
@JsonProperty(FIELD_DELTA_MANIFEST_LIST)
protected final String deltaManifestList;

// a manifest list recording all changelog produced in this snapshot
// null if no changelog is produced
@JsonProperty(FIELD_CHANGELOG_MANIFEST_LIST)
@Nullable
protected final String changelogManifestList;
```

**面试口述建议：**

> "Paimon 的每个 Snapshot 有三个 Manifest List。baseManifestList 是全量的，记录到这个快照为止所有文件的完整描述。deltaManifestList 是增量的，只记录这次 commit 新增的变更，用于快照过期和流式读。changelogManifestList 记录变更日志文件，用于 CDC 下游消费。这种三分法的好处是每种场景都能精准读取最少的数据。比如流式消费只读 delta，全表扫描读 base，CDC 订阅读 changelog。"

---

<a id="q-2-2"></a>
### 2.2 一次 commit 为什么可能产生两个 Snapshot（APPEND + COMPACT）？

**🟠 高级**

**核心答案：**

在 `FileStoreCommitImpl.commit()` 方法中，一次 `ManifestCommittable` 的提交最多会生成两个 Snapshot。源码将文件变更分为两类：`appendTableFiles/appendChangelog/appendIndexFiles` 和 `compactTableFiles/compactChangelog/compactIndexFiles`。

**原因**：写入（APPEND）和压缩（COMPACT）是两个独立的语义操作，它们的冲突检测策略不同。APPEND 产生的新文件需要检查与其他并发写入的冲突（checkAppendFiles），而 COMPACT 产生的文件是对已有文件的合并替换，冲突检测逻辑不同。将它们拆分为独立的 Snapshot，可以更精确地处理并发冲突，同时保证即使 COMPACT 的 Snapshot 提交失败，APPEND 的数据也已经安全落地。

另一个重要原因是：COMPACT 操作的 Snapshot 可以与其他写入者的 COMPACT 操作合并或重试，因为它只是在重新组织已有数据。但 APPEND 操作引入了新数据，如果混在一起，重试逻辑会变得复杂。

**源码证据：**

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/operation/FileStoreCommitImpl.java:288-374
@Override
public int commit(ManifestCommittable committable, boolean checkAppendFiles) {
    // ...
    int generatedSnapshot = 0;
    ManifestEntryChanges changes = collectChanges(committable.fileCommittables());
    
    // 第一个 Snapshot: APPEND
    if (!ignoreEmptyCommit
            || !changes.appendTableFiles.isEmpty()
            || !changes.appendChangelog.isEmpty()
            || !changes.appendIndexFiles.isEmpty()) {
        CommitKind commitKind = CommitKind.APPEND;
        // ... (冲突检测和提交)
        generatedSnapshot += 1;
    }

    // 第二个 Snapshot: COMPACT
    if (!changes.compactTableFiles.isEmpty()
            || !changes.compactChangelog.isEmpty()
            || !changes.compactIndexFiles.isEmpty()) {
        attempts += tryCommit(
                CommitChangesProvider.provider(
                        changes.compactTableFiles,
                        changes.compactChangelog,
                        changes.compactIndexFiles),
                committable.identifier(),
                // ...
                CommitKind.COMPACT,
                false, true, null);
        generatedSnapshot += 1;
    }
    return generatedSnapshot;  // 最多返回 2
}
```

**面试口述建议：**

> "Paimon 一次 commit 最多产生两个 Snapshot，一个是 APPEND 类型记录新增数据，一个是 COMPACT 类型记录合并结果。拆分的原因有两个：第一，它们的冲突检测策略不同，APPEND 需要检查并发写入冲突，COMPACT 不需要；第二，拆分后即使 COMPACT 提交失败，新数据已经安全落地，重试也更简单。在 FileStoreCommitImpl.commit 方法中可以清楚看到这个两阶段提交的逻辑。"

---

<a id="q-2-3"></a>
### 2.3 Manifest 合并机制是如何工作的？

**🟠 高级**

**核心答案：**

Paimon 的 Manifest 合并由 `ManifestFileMerger` 实现，采用"Minor Compaction + Full Compaction"的双层策略：

**Minor Compaction**：遍历所有 ManifestFileMeta，按大小累积。当累积大小达到 `suggestedMetaSize` 时，将这批 Manifest 读出来，通过 `FileEntry.mergeEntries()` 进行条目合并（同一个文件的 ADD 和 DELETE 条目相互抵消），然后写出新的 Manifest 文件。如果最后剩余的小 Manifest 数量超过 `suggestedMinMetaCount`，也进行合并。

**Full Compaction**：当需要变更的 Manifest 文件总大小超过 `manifestFullCompactionSize` 触发。Full Compaction 会：(1) 读取所有 DELETE 条目的标识符集合；(2) 尝试按分区过滤跳过不需要处理的 base 文件；(3) 对所有 Manifest 文件进行读取、过滤已删除条目后重写。

合并操作是**原子性**的：如果过程中抛异常，所有新创建的 Manifest 文件会被清理。

**源码证据：**

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/operation/ManifestFileMerger.java:60-97
public static List<ManifestFileMeta> merge(
        List<ManifestFileMeta> input, ManifestFile manifestFile,
        long suggestedMetaSize, int suggestedMinMetaCount,
        long manifestFullCompactionSize, RowType partitionType,
        @Nullable Integer manifestReadParallelism) {
    List<ManifestFileMeta> newFilesForAbort = new ArrayList<>();
    try {
        Optional<List<ManifestFileMeta>> fullCompacted =
                tryFullCompaction(input, newFilesForAbort, manifestFile,
                        suggestedMetaSize, manifestFullCompactionSize,
                        partitionType, manifestReadParallelism);
        return fullCompacted.orElseGet(() ->
                tryMinorCompaction(input, newFilesForAbort, manifestFile,
                        suggestedMetaSize, suggestedMinMetaCount, manifestReadParallelism));
    } catch (Throwable e) {
        // 异常时清理所有新创建的文件
        for (ManifestFileMeta manifest : newFilesForAbort) {
            manifestFile.delete(manifest.fileName());
        }
        throw new RuntimeException(e);
    }
}
```

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/operation/ManifestFileMerger.java:136-154
private static void mergeCandidates(...) {
    if (candidates.size() == 1) { result.add(candidates.get(0)); return; }
    Map<FileEntry.Identifier, ManifestEntry> map = new LinkedHashMap<>();
    FileEntry.mergeEntries(manifestFile, candidates, map, manifestReadParallelism);
    // ADD + DELETE 相互抵消
    if (!map.isEmpty()) {
        List<ManifestFileMeta> merged = manifestFile.write(new ArrayList<>(map.values()));
        result.addAll(merged);
        newMetas.addAll(merged);
    }
}
```

**面试口述建议：**

> "Manifest 合并分两层。Minor Compaction 按文件大小累积，达到阈值后读出条目、合并抵消（同一文件的 ADD 和 DELETE 抵消）、重写。Full Compaction 在变更文件总大小超过阈值时触发，会读出所有 DELETE 标识符，然后过滤掉已删除条目后全量重写。整个过程是原子的，失败会回滚清理新文件。核心实现在 ManifestFileMerger.merge 方法中。"

---

<a id="q-2-4"></a>
### 2.4 commitIdentifier 的去重作用是什么？

**🟡 中级**

**核心答案：**

`commitIdentifier` 是 Snapshot 中的一个关键字段，在 Flink 场景下它对应 checkpoint ID，主要作用是**实现 Exactly-Once 语义中的提交去重**。

当 Flink 作业从 checkpoint 恢复时，可能会重复提交之前已经成功的 committable。`FileStoreCommitImpl.filterCommitted()` 方法通过比较 committable 的 identifier 与最近一个 Snapshot 的 commitIdentifier 来实现去重：如果 `committable.identifier() <= latestSnapshot.commitIdentifier()`，说明这个 committable 已经被提交过了，直接跳过。

此外，commitIdentifier 还保证了快照的因果序——identifier 小的快照一定比 identifier 大的快照更早提交。

**源码证据：**

```java
// 源码: paimon-api/src/main/java/org/apache/paimon/Snapshot.java:126-134
// Mainly for snapshot deduplication.
//
// If multiple snapshots have the same commitIdentifier, reading from any of these
// snapshots must produce the same table.
//
// If snapshot A has a smaller commitIdentifier than snapshot B, then snapshot A
// must be committed before snapshot B, and thus snapshot A must contain older
// records than snapshot B.
@JsonProperty(FIELD_COMMIT_IDENTIFIER)
protected final long commitIdentifier;
```

```java
// 源码: paimon-core/src/main/java/org/apache/paimon/operation/FileStoreCommitImpl.java:270-285
if (latestSnapshot.isPresent()) {
    List<ManifestCommittable> result = new ArrayList<>();
    for (ManifestCommittable committable : committables) {
        // if committable is newer than latest snapshot, then it hasn't been committed
        if (committable.identifier() > latestSnapshot.get().commitIdentifier()) {
            result.add(committable);
        } else {
            commitCallbacks.forEach(callback -> callback.retry(committable));
        }
    }
    return result;
}
```

**面试口述建议：**

> "commitIdentifier 在 Flink 场景下对应 checkpoint ID，核心作用是提交去重。当作业从 checkpoint 恢复重启时，可能会重复提交之前已成功的数据。FileStoreCommitImpl 的 filterCommitted 方法会比较 committable 的 identifier 和最新 Snapshot 的 commitIdentifier，如果小于等于就跳过。Snapshot 的注释明确写了：identifier 小的一定比 identifier 大的更早提交。"

---

## 三、Merge 引擎

<a id="q-3-1"></a>
### 3.1 四种 MergeFunction 的实现原理和适用场景分别是什么？

**🟡 中级**

**核心答案：**

Paimon 的 `MergeEngine` 枚举定义了四种合并引擎，每种引擎对应一个 `MergeFunction` 实现：

| 引擎 | 实现类 | 核心逻辑 | 适用场景 |
|---|---|---|---|
| **deduplicate** | `DeduplicateMergeFunction` | 只保留最新的一条记录（按 sequence number），`add()` 直接用新值覆盖 `latestKv` | 最通用的去重场景，CDC 入湖 |
| **partial-update** | `PartialUpdateMergeFunction` | 合并时只更新非 null 字段，用 `GenericRow` 逐字段保存，null 字段保持旧值 | 多源汇聚、宽表构建 |
| **aggregation** | `AggregateMergeFunction` | 每个字段配置独立的聚合器（sum/count/max 等），用 `FieldAggregator` 数组逐字段聚合 | 实时指标汇总、计数器 |
| **first-row** | `FirstRowMergeFunction` | 只保留第一条记录，后续同 key 记录被丢弃。默认不接受 DELETE 记录 | 去重保留首条，如日志去重 |

关键实现细节：
- `DeduplicateMergeFunction` 是最简单的——`requireCopy()` 返回 false，因为它只保存最后一条引用。
- `FirstRowMergeFunction` 的 `requireCopy()` 返回 true，因为它保存的是第一条的引用，后续可能被覆盖。
- `PartialUpdateMergeFunction` 支持 `ignoreDelete` 和 `sequence-group` 两种高级特性。
- `AggregateMergeFunction` 区分 `agg`（正向聚合）和 `retract`（撤回）两种操作。

**源码证据：**

```java
// 源码: paimon-api/src/main/java/org/apache/paimon/CoreOptions.java:3807-3814
public enum MergeEngine implements DescribedEnum {
    DEDUPLICATE("deduplicate", "De-duplicate and keep the last row."),
    PARTIAL_UPDATE("partial-update", "Partial update non-null fields."),
    AGGREGATE("aggregation", "Aggregate fields with same primary key."),
    FIRST_ROW("first-row", "De-duplicate and keep the first row.");
}
```

```java
// 源码: paimon-core/.../DeduplicateMergeFunction.java:48-55
@Override
public void add(KeyValue kv) {
    if (ignoreDelete && kv.valueKind().isRetract()) return;
    latestKv = kv;  // 简单粗暴：直接保留最新
}
```

```java
// 源码: paimon-core/.../FirstRowMergeFunction.java:49-68
@Override
public void add(KeyValue kv) {
    if (kv.valueKind().isRetract()) {
        if (ignoreDelete) {
            return;
        } else {
            throw new IllegalArgumentException(
                    "By default, First row merge engine can not accept DELETE/UPDATE_BEFORE records.\n"
                            + "You can config 'ignore-delete' to ignore the DELETE/UPDATE_BEFORE records.");
        }
    }
    if (first == null) {
        this.first = kv;  // 只保存第一条
    }
    if (kv.level() > 0) {
        containsHighLevel = true;
    }
}
```

**面试口述建议：**

> "Paimon 有四种 MergeFunction。Deduplicate 最简单，只保留最新记录，适合 CDC 去重。PartialUpdate 按字段合并，null 字段保持旧值，适合宽表构建。Aggregation 每个字段可以配不同的聚合函数，适合实时指标汇总。FirstRow 只保留首条记录，适合日志去重。一个有趣的实现细节是 FirstRow 的 requireCopy 返回 true 因为它缓存了第一条引用，而 Deduplicate 返回 false 因为它只要最后一条。"

---

<a id="q-3-2"></a>
### 3.2 PartialUpdate 的 Sequence Group 机制是如何工作的？

**🟠 高级**

**核心答案：**

Sequence Group 是 PartialUpdate 的一项高级特性，解决的问题是：当多个数据源分别更新不同字段时，如何保证**每组字段的更新按各自的序列号排序**，而不是全局统一排序。

在没有 Sequence Group 的情况下，PartialUpdate 按全局 sequence number 判断新旧。但如果源 A 更新字段 {name, age}，源 B 更新字段 {salary, dept}，全局 sequence number 无法正确处理乱序到达的情况。

Sequence Group 允许用户为每组字段指定独立的序列号字段。配置方式是 `fields.{sequence-field}.sequence-group = field1,field2,...`。在 `PartialUpdateMergeFunction.add()` 中，对于每个 Sequence Group，会使用各自的 `FieldsComparator` 比较序列号，只有当新记录的序列号更大时才更新该组的字段。

此外，Sequence Group 还支持 `fieldAggregators`，即每个字段可以配置独立的聚合函数（如 sum、collect），在 Partial Update 的基础上实现更复杂的合并逻辑。

**源码证据：**

```java
// 源码: paimon-core/.../PartialUpdateMergeFunction.java:67-68
public static final String SEQUENCE_GROUP = "sequence-group";
```

```java
// 源码: paimon-core/.../PartialUpdateMergeFunction.java:69-76
private final InternalRow.FieldGetter[] getters;
private final boolean ignoreDelete;
private final List<WrapperWithFieldIndex<FieldsComparator>> fieldSeqComparators;
private final boolean fieldSequenceEnabled;
private final List<WrapperWithFieldIndex<FieldAggregator>> fieldAggregators;
private final boolean removeRecordOnDelete;
private final Set<Integer> sequenceGroupPartialDelete;
```

```java
// 源码: paimon-core/.../PartialUpdateMergeFunction.java:122-144
@Override
public void add(KeyValue kv) {
    currentKey = kv.key();
    currentDeleteRow = false;
    if (kv.valueKind().isRetract()) {
        // ...
        if (fieldSequenceEnabled) {
            retractWithSequenceGroup(kv);  // 按 Sequence Group 处理 retract
            return;
        }
        // ...
    }
    // ...
}
```

**面试口述建议：**

> "Sequence Group 解决的是多源写入同一张宽表时的乱序问题。比如源 A 更新 name 和 age，源 B 更新 salary 和 dept，全局 sequence number 无法正确处理。Sequence Group 允许每组字段指定各自的序列号字段，合并时按组独立比较序列号。配置方式是 fields.seq_field.sequence-group = field1,field2。源码中 PartialUpdateMergeFunction 维护了 fieldSeqComparators 列表，每个 Sequence Group 有独立的比较器。"

---

<a id="q-3-3"></a>
### 3.3 LookupMergeFunction 的包装设计有什么巧妙之处？

**🔴 专家**

**核心答案：**

`LookupMergeFunction` 是一个**装饰器**（Wrapper），它包装了任意一个 `MergeFunction`（如 Deduplicate、PartialUpdate、Aggregation），在 Compaction 过程中通过 Lookup 机制获取历史值来正确合并。

其巧妙之处在于：

1. **只考虑最新的高层记录**：`pickHighLevel()` 方法从 candidates 中选取 level > 0 的最低层的记录作为 "历史值"。因为每次 Lookup Compaction 都会查询旧的合并结果，所以最低层（最近被合并的）的高层记录就是上一次的最终结果。

2. **合并时只取 Level0 + 一条高层记录**：`getResult()` 调用内部 MergeFunction 时，只传入 Level0 的记录和 pickHighLevel 选出的那条记录。这样即使有多层高层数据，也只需处理一条。

3. **FirstRow 的豁免**：如果被包装的是 `FirstRowMergeFunction`，`wrap()` 方法直接返回原始工厂，不进行包装。因为 FirstRow 只关心第一条记录，不需要查历史值。

4. **containLevel0() 判断**：用于决定是否需要从高层查询历史值。如果当前合并的文件中没有 Level0 数据（即纯高层合并），就不需要 Lookup。

**源码证据：**

```java
// 源码: paimon-core/.../LookupMergeFunction.java:76-96
@Nullable
public KeyValue pickHighLevel() {
    KeyValue highLevel = null;
    try (CloseableIterator<KeyValue> iterator = candidates.iterator()) {
        while (iterator.hasNext()) {
            KeyValue kv = iterator.next();
            if (kv.level() <= 0) continue;  // 跳过 Level0 及以下
            // 选择 level 最小的高层记录（最近被合并的）
            if (highLevel == null || kv.level() < highLevel.level()) {
                highLevel = kv;
            }
        }
    }
    return highLevel;
}
```

```java
// 源码: paimon-core/.../LookupMergeFunction.java:107-123
@Override
public KeyValue getResult() {
    mergeFunction.reset();
    KeyValue highLevel = pickHighLevel();
    try (CloseableIterator<KeyValue> iterator = candidates.iterator()) {
        while (iterator.hasNext()) {
            KeyValue kv = iterator.next();
            if (kv.level() <= 0 || kv == highLevel) {
                mergeFunction.add(kv);  // 只合并 Level0 + 一条高层记录
            }
        }
    }
    return mergeFunction.getResult();
}
```

```java
// 源码: paimon-core/.../LookupMergeFunction.java:130-141
public static MergeFunctionFactory<KeyValue> wrap(...) {
    if (wrapped.create() instanceof FirstRowMergeFunction) {
        return wrapped;  // FirstRow 不需要包装
    }
    return new Factory(wrapped, options, keyType, valueType);
}
```

**面试口述建议：**

> "LookupMergeFunction 是一个装饰器，包装了实际的 MergeFunction。核心思想是：Compaction 时不需要读取所有历史版本，只需要查上一次合并的结果。pickHighLevel 方法找到 candidates 中 level 最小的高层记录作为历史值，然后 getResult 只合并 Level0 的新数据和这一条历史值。一个有趣的细节是 FirstRow 被豁免了包装，因为它只关心第一条记录不需要 Lookup。另一个是 containLevel0 方法——如果没有 Level0 数据，说明是纯高层合并，不需要 Lookup 查历史。"

---

<a id="q-3-4"></a>
### 3.4 Changelog 产生的四种模式有什么区别？

**🟠 高级**

**核心答案：**

Paimon 的 `ChangelogProducer` 枚举定义了四种 changelog 产生模式：

| 模式 | 机制 | 产生时机 | 成本 | 场景 |
|---|---|---|---|---|
| **NONE** | 不产生 changelog 文件 | - | 无额外成本 | 不需要流式消费变更 |
| **INPUT** | 将输入数据双写到 changelog 文件 | 内存表 flush 时 | 写放大约 2x | 输入已是完整 CDC 流（有 +I/-D/-U+U） |
| **FULL_COMPACTION** | 在 Full Compaction 时对比前后差异产生 changelog | Full Compaction 完成时 | Compaction 时额外计算 | 不需要实时 changelog，可以容忍延迟 |
| **LOOKUP** | 通过 Lookup Compaction 查询历史值产生精确 changelog | Lookup Compaction 时 | 额外的 Lookup I/O | 需要实时精确 changelog |

核心区别：
- **INPUT** 最简单但要求输入本身就是 CDC 格式，changelog 的质量取决于输入。
- **LOOKUP** 和 **FULL_COMPACTION** 都能从非 CDC 输入（如纯 INSERT）推导出精确 changelog，但 LOOKUP 是实时的（每次 Compaction 都产生），FULL_COMPACTION 是延迟的（只在 Full Compaction 时产生）。
- **LOOKUP** 需要额外的磁盘空间存储 Lookup SST 文件，代价是 I/O 和磁盘。

**源码证据：**

```java
// 源码: paimon-api/src/main/java/org/apache/paimon/CoreOptions.java:3948-3957
public enum ChangelogProducer implements DescribedEnum {
    NONE("none", "No changelog file."),
    INPUT("input", "Double write to a changelog file when flushing memory table, the changelog is from input."),
    FULL_COMPACTION("full-compaction", "Generate changelog files with each full compaction."),
    LOOKUP("lookup", "Generate changelog files through 'lookup' compaction.");
}
```

**面试口述建议：**

> "Paimon 有四种 changelog 模式。NONE 不产生 changelog。INPUT 在 flush 时双写，前提是输入必须是 CDC 格式。FULL_COMPACTION 在全量合并时对比差异产生，有延迟但成本低。LOOKUP 通过查询历史值实时产生精确 changelog，成本最高但最实时。选择策略是：如果输入已是 CDC 流用 INPUT；需要实时 changelog 用 LOOKUP；能容忍延迟用 FULL_COMPACTION；不需要就用 NONE。"

---

<a id="q-3-5"></a>
### 3.5 Paimon 的 20+ 内置聚合函数体系是如何组织的？

**🟡 中级**

**核心答案：**

Paimon 的聚合函数体系采用 **SPI (Service Provider Interface) + Factory** 模式组织，核心接口是 `FieldAggregator`，工厂接口是 `FieldAggregatorFactory`。每种聚合函数通过工厂类注册，以下是主要的内置聚合函数：

| 聚合函数 | 工厂类 | 功能 |
|---|---|---|
| sum | `FieldSumAggFactory` | 求和 |
| min / max | `FieldMinAggFactory` / `FieldMaxAggFactory` | 最小值/最大值 |
| count | (通过 sum 实现) | 计数 |
| product | `FieldProductAggFactory` | 乘积 |
| first_value / first_non_null_value | `FieldFirstValueAggFactory` / `FieldFirstNonNullValueAggFactory` | 首个值 |
| last_value / last_non_null_value | `FieldLastValueAggFactory` / `FieldLastNonNullValueAggFactory` | 最新值 |
| listagg | `FieldListaggAggFactory` | 字符串聚合 |
| bool_and / bool_or | `FieldBoolAndAggFactory` / `FieldBoolOrAggFactory` | 布尔聚合 |
| collect | `FieldCollectAggFactory` | 收集到数组 |
| merge_map | `FieldMergeMapAggFactory` | Map 合并 |
| merge_map_with_key_time | `FieldMergeMapWithKeyTimeAggFactory` | 带时间的 Map 合并 |
| nested_update | `FieldNestedUpdateAggFactory` | 嵌套更新 |
| nested_partial_update | `FieldNestedPartialUpdateAggFactory` | 嵌套部分更新 |
| roaring_bitmap_32 / roaring_bitmap_64 | `FieldRoaringBitmap32AggFactory` / `FieldRoaringBitmap64AggFactory` | RoaringBitmap 聚合 |
| theta_sketch | `FieldThetaSketchAggFactory` | Theta Sketch 近似去重 |
| hll_sketch | `FieldHllSketchAggFactory` | HyperLogLog Sketch |
| primary_key | `FieldPrimaryKeyAggFactory` | 主键字段（占位符） |

在 `AggregateMergeFunction` 中，每个字段维护一个 `FieldAggregator` 实例。合并时对每条记录的每个字段调用 `agg()` 或 `retract()` 方法。

**源码证据：**

```java
// 源码: paimon-core/.../aggregate/AggregateMergeFunction.java:81-101
@Override
public void add(KeyValue kv) {
    latestKv = kv;
    boolean isRetract = kv.valueKind().isRetract();
    for (int i = 0; i < getters.length; i++) {
        FieldAggregator fieldAggregator = aggregators[i];
        Object accumulator = getters[i].getFieldOrNull(row);
        Object inputField = getters[i].getFieldOrNull(kv.value());
        Object mergedField = isRetract
                ? fieldAggregator.retract(accumulator, inputField)
                : fieldAggregator.agg(accumulator, inputField);
        row.setField(i, mergedField);
    }
}
```

**面试口述建议：**

> "Paimon 有超过 20 种内置聚合函数，通过 SPI + Factory 模式组织。每个字段可以配置独立的聚合函数，如 sum、max、merge_map、roaring_bitmap 等。在 AggregateMergeFunction 的 add 方法中，对每条记录逐字段调用各自的 agg 或 retract 方法。还支持一些特殊的聚合如 nested_update（嵌套更新）和 theta_sketch（近似去重），覆盖了大多数实时数仓的聚合需求。"

---

## 四、Flink 集成

<a id="q-4-1"></a>
### 4.1 Flink Sink 的算子拓扑（Writer → Committer）是如何设计的？

**🟡 中级**

**核心答案：**

Paimon 的 Flink Sink 拓扑由两个核心算子组成：

**Writer 算子**（多并行度）：负责接收输入数据，写入 Paimon 的 Write Buffer，触发 flush 和后台 Compaction。在 `FlinkSink` 中通过 `doWrite()` 方法创建，并行度与 Flink 作业的并行度一致。每个 Writer 实例负责一个或多个 bucket 的写入。

**Committer 算子**（并行度为 1）：负责将所有 Writer 产生的 `Committable`（包含 `CommitMessage`）聚合后提交为 Paimon 的 Snapshot。`CommitterOperator` 强制要求并行度为 1（`forceSingleParallelism=true`），因为 Snapshot 的创建必须是原子的、全局唯一的。

数据流向：`Input → Writer[N] → Committable → Committer[1] → Snapshot`

Writer 在 Flink checkpoint barrier 到来时，将当前的 `CommitMessage`（包含新文件和 compact 结果）封装为 `Committable` 发送给 Committer。Committer 在 `notifyCheckpointComplete()` 时批量提交。

**源码证据：**

```java
// 源码: paimon-flink/.../FlinkSink.java:97-100
public DataStreamSink<?> sinkFrom(DataStream<T> input, String initialCommitUser) {
    // do the actually writing action, no snapshot generated in this stage
    DataStream<Committable> written = doWrite(input, initialCommitUser, null);
    // commit the committable to generate a new snapshot
```

```java
// 源码: paimon-flink/.../CommitterOperator.java:118-122
Preconditions.checkArgument(
        !forceSingleParallelism
                || RuntimeContextUtils.getNumberOfParallelSubtasks(getRuntimeContext()) == 1,
        "Committer Operator parallelism in paimon MUST be one.");
```

```java
// 源码: paimon-flink/.../CommitterOperator.java:73
// Group the committable by the checkpoint id.
protected final NavigableMap<Long, GlobalCommitT> committablesPerCheckpoint;
```

**面试口述建议：**

> "Paimon 的 Flink Sink 是典型的两阶段拓扑。多并行度的 Writer 算子负责写数据，触发 flush 和 Compaction。单并行度的 Committer 算子负责收集所有 Writer 的 CommitMessage，在 checkpoint complete 时原子提交 Snapshot。Committer 强制并行度为 1 是因为 Snapshot 必须全局唯一。Writer 通过 Flink 的 checkpoint barrier 机制向 Committer 发送 Committable 数据。"

---

<a id="q-4-2"></a>
### 4.2 Checkpoint 两阶段提交的 Exactly-Once 保证是如何实现的？

**🟠 高级**

**核心答案：**

Paimon 的 Exactly-Once 保证通过三个关键机制实现：

**1. Pre-commit 阶段（snapshotState）**：在 Flink checkpoint barrier 到来时，`CommitterOperator.snapshotState()` 将当前收集的 inputs 通过 `pollInputs()` 转化为 committables，并通过 `committableStateManager.snapshotState()` 持久化到 Flink 的 operator state 中。这确保了即使 Committer 崩溃，重启后也能恢复待提交的数据。

**2. Commit 阶段（notifyCheckpointComplete）**：当 Flink Job Manager 通知 checkpoint 完成时，`notifyCheckpointComplete()` 被调用，执行 `commitUpToCheckpoint()`。它从 `committablesPerCheckpoint`（NavigableMap，按 checkpoint ID 排序）中取出所有小于等于当前 checkpoint ID 的 committables，批量提交。

**3. 去重恢复**：如果 checkpoint 完成但 Committer 在提交 Paimon Snapshot 后、确认前崩溃了，重启后 `filterCommitted()` 通过 `commitIdentifier` 去重，跳过已提交的 committable。

**源码证据：**

```java
// 源码: paimon-flink/.../CommitterOperator.java:163-168
@Override
public void snapshotState(StateSnapshotContext context) throws Exception {
    super.snapshotState(context);
    pollInputs();
    committableStateManager.snapshotState(context, committables(committablesPerCheckpoint));
}
```

```java
// 源码: paimon-flink/.../CommitterOperator.java:190-218
@Override
public void notifyCheckpointComplete(long checkpointId) throws Exception {
    super.notifyCheckpointComplete(checkpointId);
    commitUpToCheckpoint(endInput ? END_INPUT_CHECKPOINT_ID : checkpointId);
}

private void commitUpToCheckpoint(long checkpointId) throws Exception {
    NavigableMap<Long, GlobalCommitT> headMap =
            committablesPerCheckpoint.headMap(checkpointId, true);
    List<GlobalCommitT> committables = committables(headMap);
    // ...
    if (checkpointId == END_INPUT_CHECKPOINT_ID) {
        committer.filterAndCommit(committables, false, true);  // 带去重
    } else {
        committer.commit(committables);
    }
    headMap.clear();
}
```

**面试口述建议：**

> "Paimon 的 Exactly-Once 有三道保障。第一，snapshotState 时将待提交数据持久化到 Flink operator state，崩溃可恢复。第二，notifyCheckpointComplete 时才真正提交 Snapshot，用 NavigableMap 的 headMap 批量取出所有小于等于当前 checkpoint ID 的数据提交。第三，恢复后通过 commitIdentifier 比较去重，跳过已提交的数据。这三者配合保证了即使在各种故障场景下也不会重复或丢失数据。"

---

<a id="q-4-3"></a>
### 4.3 CDC 同步的完整链路和 Schema 自动演进是如何工作的？

**🟠 高级**

**核心答案：**

Paimon 的 CDC 同步通过 `paimon-flink-cdc` 模块实现，核心组件是 `CdcSinkBuilder` 和 `RichCdcSinkBuilder`，支持从 MySQL/Kafka 等数据源将 CDC 变更实时同步到 Paimon 表。

**完整链路**：
1. **Source**：使用 Flink CDC Connector（如 mysql-cdc）读取 binlog 变更
2. **解析**：将 CDC 事件解析为 Paimon 的内部格式（包含 schema 信息）
3. **Schema 检测**：检查上游 schema 变更（如新增列、类型变更），自动触发 Paimon 表的 schema evolution
4. **写入**：通过 Writer 算子写入对应的 bucket
5. **提交**：Committer 算子提交 Snapshot

**Schema 自动演进**的核心在于：
- 每条 CDC 记录携带其 schema 信息
- Writer 在写入时检测 schema 变更
- 通过 `SchemaManager` 发起 schema change 请求
- Paimon 的 schema evolution 是基于 schema ID 的，每次变更产生新的 schema ID

RichCdcSinkBuilder 相比 CdcSinkBuilder 增加了更丰富的功能，如分库分表合并等。

**源码证据：**

```java
// 源码: paimon-flink/paimon-flink-cdc/src/main/java/org/apache/paimon/flink/sink/cdc/CdcSinkBuilder.java
// CdcSinkBuilder 是 CDC 同步的核心构建器
```

```java
// 源码: paimon-flink/paimon-flink-cdc/src/main/java/org/apache/paimon/flink/sink/cdc/RichCdcSinkBuilder.java
// RichCdcSinkBuilder 扩展了 CdcSinkBuilder，支持分库分表合并等高级特性
```

**面试口述建议：**

> "Paimon 的 CDC 同步链路是：Flink CDC Source 读 binlog，解析为 Paimon 内部格式，检测 schema 变更并自动演进，写入 Paimon 表，Committer 提交。Schema 自动演进是核心亮点——每条 CDC 记录携带 schema 信息，Writer 检测到变更后通过 SchemaManager 自动发起 schema change。通过 CdcSinkBuilder 和 RichCdcSinkBuilder 支持单表同步和分库分表合并两种场景。"

---

<a id="q-4-4"></a>
### 4.4 Lookup Join 的缓存和增量刷新机制是什么？

**🟡 中级**

**核心答案：**

Paimon 在 Flink 中支持 Lookup Join，由 `FileStoreLookupFunction` 实现。核心机制包括：

**缓存模式**：支持两种模式（`LookupCacheMode`）：
- **AUTO**：根据表的特性自动选择
- **FULL**：全量加载维表数据到本地 RocksDB，适合小维表
- **PARTIAL**：部分缓存，按需加载

**增量刷新**：通过 `CONTINUOUS_DISCOVERY_INTERVAL` 配置刷新间隔。`FileStoreLookupFunction` 维护一个 `LookupTable` 实例，定期检查 Paimon 表的新 Snapshot，增量加载变更数据到本地缓存。

**分区加载**：通过 `PartitionLoader` 支持按分区加载维表数据，减少内存占用。

**源码证据：**

```java
// 源码: paimon-flink/.../lookup/FileStoreLookupFunction.java:80-98
public class FileStoreLookupFunction implements Serializable, Closeable {
    private final FileStoreTable table;
    @Nullable private final PartitionLoader partitionLoader;
    private final List<String> projectFields;
    private final List<String> joinKeys;
    @Nullable private final Predicate predicate;
    @Nullable private final RefreshBlacklist refreshBlacklist;
    @Nullable private final ShuffleStrategy strategy;
    // ...
    private transient LookupTable lookupTable;
}
```

**面试口述建议：**

> "Paimon 的 Lookup Join 通过 FileStoreLookupFunction 实现，支持全量和部分缓存两种模式。全量模式把维表数据加载到本地 RocksDB，部分模式按需加载。增量刷新通过定期发现新 Snapshot 并增量加载变更实现，配置参数是 continuous.discovery-interval。还支持按分区加载来减少内存占用，以及 ShuffleStrategy 来优化 Lookup Join 的数据分布。"

---

## 五、Deletion Vectors 与文件索引

<a id="q-5-1"></a>
### 5.1 DV 的 RoaringBitmap 实现和 V1/V2 格式差异是什么？

**🟡 中级**

**核心答案：**

Paimon 的 Deletion Vector 有两个实现版本：

**V1 — `BitmapDeletionVector`**：基于 `RoaringBitmap32`（32 位 RoaringBitmap），最大支持 2^31 行。Magic Number 为 `1581511376`。position 参数为 long 类型但内部转为 int 使用，通过 `checkPosition()` 检查不越界。

**V2 — `Bitmap64DeletionVector`**：基于 `OptimizedRoaringBitmap64`（64 位 RoaringBitmap），最大支持 2^63 行。Magic Number 为 `1681511377`。注释中标注"Mostly copied from iceberg"。V2 直接支持 long 类型 position，无需转换。

**读取时的格式判断**：`DeletionVector.read()` 方法先读取 magic number，根据值判断是 V1 还是 V2 格式，然后调用对应的反序列化方法。V2 的 magic number 在存储时使用了 little-endian 字节序，读取时需要 `toLittleEndianInt()` 转换。

两个版本的序列化格式均包含：bitmapLength + magicNumber + bitmapData + CRC32。

**源码证据：**

```java
// 源码: paimon-core/.../deletionvectors/BitmapDeletionVector.java:34-39
public class BitmapDeletionVector implements DeletionVector {
    public static final int MAGIC_NUMBER = 1581511376;
    private final RoaringBitmap32 roaringBitmap;
    @Override
    public void delete(long position) {
        checkPosition(position);
        roaringBitmap.add((int) position);  // 强转 int
    }
}
```

```java
// 源码: paimon-core/.../deletionvectors/Bitmap64DeletionVector.java:38-64
public class Bitmap64DeletionVector implements DeletionVector {
    public static final int MAGIC_NUMBER = 1681511377;
    // Mostly copied from iceberg.
    private final OptimizedRoaringBitmap64 roaringBitmap;
    @Override
    public void delete(long position) {
        roaringBitmap.add(position);  // 直接使用 long
    }
}
```

```java
// 源码: paimon-core/.../deletionvectors/DeletionVector.java:97-146
static DeletionVector read(DataInputStream dis, @Nullable Long length) throws IOException {
    int bitmapLength = dis.readInt();
    int magicNumber = dis.readInt();
    if (magicNumber == BitmapDeletionVector.MAGIC_NUMBER) {
        // V1 格式
        return BitmapDeletionVector.deserializeFromByteBuffer(ByteBuffer.wrap(bytes));
    } else if (toLittleEndianInt(magicNumber) == Bitmap64DeletionVector.MAGIC_NUMBER) {
        // V2 格式
        return Bitmap64DeletionVector.deserializeFromBitmapDataBytes(bytes);
    }
}
```

**面试口述建议：**

> "Paimon 的 Deletion Vector 有 V1 和 V2 两个版本。V1 用 32 位 RoaringBitmap，最大支持约 21 亿行。V2 用 64 位 RoaringBitmap，支持更大文件，代码注释说是从 Iceberg 移植的。读取时通过 magic number 区分格式，V2 用了 little-endian 字节序。两者都包含 CRC32 校验。V1 的 delete 方法要把 long 强转 int，V2 直接使用 long。"

---

<a id="q-5-2"></a>
### 5.2 DV 如何让文件变为 rawConvertible？

**🟠 高级**

**核心答案：**

在 Paimon 中，`rawConvertible` 是 `DataSplit` 的一个属性，表示该 Split 中的数据文件**可以直接以原始文件格式（Parquet/ORC）暴露给查询引擎**，无需经过 Paimon 的 Merge 逻辑。

**DV 让主键表也能 rawConvertible 的原理**：

对于主键表（PK table），通常需要 Merge-on-Read 来合并不同 Level 的重复 key。但如果开启了 DV（Deletion Vectors），Compaction 时不再物理删除旧记录，而是在 DV 文件中标记旧记录的行号为已删除。这样每个数据文件 + 对应的 DV 文件就构成了一个自包含的、不需要跨文件 Merge 的数据单元。查询引擎只需按行号跳过 DV 标记的行即可。

当 `DataSplit.rawConvertible()` 为 true 时，`convertToRawFiles()` 方法可以将数据文件直接转为 `RawFile` 列表，供 Spark/Flink 的原生 Parquet/ORC reader 读取。`rawMergedRowCountAvailable()` 方法还会检查 DV 文件是否携带 cardinality 信息（标记的行数），以便精确计算剩余行数。

**源码证据：**

```java
// 源码: paimon-core/.../table/source/DataSplit.java:78
private boolean rawConvertible;

// 源码: paimon-core/.../table/source/DataSplit.java:115-117
public boolean rawConvertible() {
    return rawConvertible;
}

// 源码: paimon-core/.../table/source/DataSplit.java:143-148
private boolean rawMergedRowCountAvailable() {
    return rawConvertible
            && (dataDeletionFiles == null
                    || dataDeletionFiles.stream()
                            .allMatch(f -> f == null || f.cardinality() != null));
}

// 源码: paimon-core/.../table/source/DataSplit.java:247-254
@Override
public Optional<List<RawFile>> convertToRawFiles() {
    if (rawConvertible) {
        return Optional.of(
                dataFiles.stream()
                        .map(f -> makeRawTableFile(bucketPath, f))
                        .collect(Collectors.toList()));
    } else { return Optional.empty(); }
}
```

**面试口述建议：**

> "rawConvertible 表示 Split 的数据文件可以直接暴露给引擎的原生 reader，不需要经过 Paimon 的 Merge 逻辑。DV 让主键表也能 rawConvertible 的关键是：开启 DV 后，旧记录不再物理删除，而是在 DV 文件中标记行号。这样每个数据文件 + DV 文件就是自包含的，引擎只需按行号跳过标记的行。DataSplit 的 convertToRawFiles 方法会把数据文件直接转为 RawFile 列表，让 Spark/Flink 用原生 Parquet reader 读取。"

---

<a id="q-5-3"></a>
### 5.3 四种文件索引的原理和选择策略是什么？

**🟠 高级**

**核心答案：**

Paimon 支持四种嵌入式文件索引（Data File Index），存储在独立的索引文件中（由 `FileIndexFormat` 定义格式），用于在读取时快速过滤不匹配的数据行或文件：

| 索引类型 | 实现位置 | 原理 | 适用查询 | 适用数据 |
|---|---|---|---|---|
| **Bloom Filter** | `bloomfilter/BloomFilterFileIndex` | 基于 FastHash 的概率型数据结构，判断值"可能存在"或"一定不存在" | 等值查询（=, IN） | 高基数列（如 ID） |
| **Bitmap** | `bitmap/BitmapFileIndex` | 为每个不同的值构建一个 RoaringBitmap，标记包含该值的行号 | 等值查询、IN、IS NULL | 低基数列（如枚举） |
| **BSI (Bit-Sliced Index)** | `bsi/BitSliceIndexBitmapFileIndex` | 将数值按二进制位切片，每个 bit 一个 RoaringBitmap | 范围查询（>、<、BETWEEN） | 数值型列 |
| **Range Bitmap** | `rangebitmap/RangeBitmapFileIndex` | 基于字典编码 + BitSliceIndex，支持有序数据的范围查询 | 范围查询 | 有序数值列 |

**选择策略**：
- 等值查询且高基数 → Bloom Filter（空间效率高，有误判）
- 等值查询且低基数 → Bitmap（精确，支持 IS NULL）
- 范围查询 → BSI 或 Range Bitmap
- Range Bitmap 适合数据有序且带字典编码的场景

**源码证据：**

```java
// 索引文件格式: paimon-common/.../fileindex/FileIndexFormat.java:48-60
// File index file format. Put all column and offset in the header.
// ______________________________________
// |     magic    ｜version｜head length  |
// |--------------------------------------|
// |            column number             |
// |--------------------------------------|
// |   column 1        ｜ index number    |
// |--------------------------------------|
// |  index name 1 ｜start pos ｜length   |
```

文件索引的四种实现分别位于：
- `paimon-common/src/main/java/org/apache/paimon/fileindex/bloomfilter/`
- `paimon-common/src/main/java/org/apache/paimon/fileindex/bitmap/`
- `paimon-common/src/main/java/org/apache/paimon/fileindex/bsi/`
- `paimon-common/src/main/java/org/apache/paimon/fileindex/rangebitmap/`

**面试口述建议：**

> "Paimon 有四种嵌入式文件索引。Bloom Filter 适合高基数列的等值查询，有误判但空间效率高。Bitmap 为每个值建一个 RoaringBitmap，适合低基数列，精确且支持 IS NULL。BSI 是 Bit-Sliced Index，把数值按二进制位切片，适合范围查询。Range Bitmap 在 BSI 基础上加了字典编码，适合有序数据。索引文件格式是自定义的，header 存列名和偏移量，body 存各列的各种索引数据。"

---

<a id="q-5-4"></a>
### 5.4 Predicate 体系如何连接查询和索引？

**🟡 中级**

**核心答案：**

Paimon 的 Predicate 体系是查询下推的核心桥梁，定义在 `paimon-common/predicate/` 包中。Predicate 由 `PredicateBuilder` 构建，支持 `Equal`、`NotEqual`、`GreaterThan`、`LessThan`、`In`、`IsNull`、`IsNotNull`、`Between` 等常见谓词。

**连接机制**：

1. **Manifest 层过滤**：Predicate 被转换为分区过滤条件（`PartitionPredicate`），用于在读 Manifest 时跳过不匹配的分区。ManifestFileMeta 中的 `partitionStats` 存储了每个 Manifest 文件的分区范围统计，可以做第一层剪枝。

2. **文件层过滤**：DataFileMeta 包含每列的 min/max 统计信息，Predicate 通过这些统计信息判断整个文件是否可能包含匹配的数据。

3. **文件索引层过滤**：`FileIndexPredicate` 将 Predicate 转换为对文件索引的查询。例如 `Equal` 谓词会转换为 Bloom Filter 的 `testHash()` 或 Bitmap 的精确查找。

4. **行级过滤**：对于 Bitmap 索引，可以返回精确的行号集合，配合 `ApplyBitmapIndexRecordReader` 在读取时直接跳到匹配的行。

**面试口述建议：**

> "Paimon 的 Predicate 体系是查询优化的核心。它连接了四个层次的过滤：Manifest 层通过分区统计跳过不相关的 Manifest 文件，文件层通过 min/max 统计跳过不相关的数据文件，文件索引层通过 Bloom Filter 或 Bitmap 进一步过滤，行级别 Bitmap 索引可以直接定位匹配的行号。FileIndexPredicate 负责将 Predicate 转换为对索引的查询操作。"

---

## 六、查询优化

<a id="q-6-1"></a>
### 6.1 多层文件过滤（分区 → Manifest → 文件 → 行）是如何工作的？

**🟡 中级**

**核心答案：**

Paimon 的查询优化采用多层渐进式过滤，从粗到细逐步缩小扫描范围：

**第 1 层 — 分区过滤**：根据查询条件中的分区列谓词，直接跳过不相关的分区目录。这是最粗粒度的过滤，效果最显著。

**第 2 层 — Manifest 文件过滤**：`ManifestFileMeta` 中存储了 `partitionStats`（分区范围统计）、`minBucket/maxBucket`（bucket 范围）、`minLevel/maxLevel`（Level 范围）。在读取 Manifest List 时，根据这些统计信息跳过不相关的 Manifest 文件。

**第 3 层 — 数据文件过滤**：`DataFileMeta` 包含每列的 min/max 统计信息（`SimpleStats`），通过 Predicate 与统计信息的比较跳过不可能包含匹配数据的文件。例如查询 `age > 30`，如果某文件的 `max(age) = 25`，则跳过。

**第 4 层 — 文件索引/行级过滤**：如果配置了文件索引（Bloom Filter、Bitmap 等），在打开文件前先查询索引。Bitmap 索引甚至可以返回精确的行号集合，直接跳到匹配行。

**源码证据：**

```java
// ManifestFileMeta 的统计字段用于第 2 层过滤
// 源码: paimon-core/.../manifest/ManifestFileMeta.java:44-60
public static final RowType SCHEMA = new RowType(false, Arrays.asList(
    new DataField(0, "_FILE_NAME", ...),
    new DataField(4, "_PARTITION_STATS", SimpleStats.SCHEMA),
    new DataField(6, "_MIN_BUCKET", new IntType(true)),
    new DataField(7, "_MAX_BUCKET", new IntType(true)),
    new DataField(8, "_MIN_LEVEL", new IntType(true)),
    new DataField(9, "_MAX_LEVEL", new IntType(true)),
    // ...
));
```

**面试口述建议：**

> "Paimon 的查询优化是四层渐进过滤。第一层分区剪枝直接跳过不相关分区。第二层通过 ManifestFileMeta 中的 partitionStats、bucket 范围、level 范围跳过不相关的 Manifest 文件。第三层通过 DataFileMeta 中的列级 min/max 统计跳过不相关的数据文件。第四层通过文件索引做更精确的过滤，Bitmap 索引甚至能精确到行号。每一层都大幅减少下一层的数据量。"

---

<a id="q-6-2"></a>
### 6.2 Z-Order/Hilbert 排序如何改善查询性能？

**🟠 高级**

**核心答案：**

Z-Order 和 Hilbert 排序是多维排序技术，用于改善多列查询的数据局部性。

**问题背景**：传统单列排序只对该列的范围查询有效。如果查询同时过滤列 A 和列 B，按列 A 排序的数据在列 B 上的值是随机分散的，无法有效利用 min/max 统计做文件级剪枝。

**Z-Order 排序**：将多个列的值交织（interleave）成一个一维空间填充曲线（Space-Filling Curve），相邻的多维数据点在一维空间中也尽量相邻。这样排序后，每个文件在多个列上都有较紧凑的值域范围，min/max 过滤效果更好。

**Hilbert 排序**：比 Z-Order 更优的空间填充曲线，具有更好的聚簇性（clustering）。相邻的点在 Hilbert 曲线上距离更近，文件级过滤效果更好。

在 Paimon 中，Z-Order/Hilbert 排序主要通过 Sort Compact 操作实现，对数据文件按指定列的 Z-Order 或 Hilbert 值重排后写出。

**面试口述建议：**

> "Z-Order 和 Hilbert 排序解决的是多列查询的数据局部性问题。单列排序只对该列有效，但查询经常同时过滤多列。Z-Order 将多列的值交织成一维空间填充曲线，确保多维空间中相邻的数据在文件中也相邻。这样每个文件在多列上的 min/max 范围更紧凑，过滤效果更好。Hilbert 曲线比 Z-Order 聚簇性更好。在 Paimon 中通过 Sort Compact 操作实现。"

---

<a id="q-6-3"></a>
### 6.3 LookupLevels 的点查优化机制是什么？

**🔴 专家**

**核心答案：**

`LookupLevels` 是 Paimon 的点查（point lookup）优化核心组件，它为 Merge Tree 的每个数据文件构建本地 Lookup SST 文件，支持按 key 快速查找。

**核心机制**：

1. **SST 文件构建**：首次对某个文件做 Lookup 时，`createLookupFile()` 读取整个数据文件，将 key-value 对写入本地磁盘的 SST 文件（通过 `LookupStoreWriter`），同时可选地构建 BloomFilter。

2. **Caffeine 缓存**：使用 Caffeine Cache（`lookupFileCache`）缓存 `LookupFile` 对象。缓存键是文件名，缓存支持按保留时间和最大磁盘大小淘汰。Compaction 产生新文件时，通过 `DropFileCallback` 机制使旧缓存失效。

3. **分层查找**：查找时按 Level 从低到高依次查找。先查 Level0（按 sequence number 从新到旧），再查高层。`LookupUtils` 提供了标准的查找流程。

4. **Schema 演进支持**：如果 Lookup 文件的 schema 与当前不同（老文件），通过 `PersistProcessor` 的 schema ID 机制处理值的反序列化。

5. **远程 Lookup 文件**：支持 `lookupRemoteFileEnabled`，可以从远程存储下载预构建的 Lookup 文件，避免本地重建的开销。

**源码证据：**

```java
// 源码: paimon-core/.../mergetree/LookupLevels.java:131-170
@Nullable
public T lookup(InternalRow key, int startLevel) throws IOException {
    return LookupUtils.lookup(levels, key, startLevel, this::lookup, this::lookupLevel0);
}

@Nullable
private T lookup(InternalRow key, DataFileMeta file) throws IOException {
    LookupFile lookupFile = lookupFileCache.getIfPresent(file.fileName());
    boolean newCreatedLookupFile = false;
    if (lookupFile == null) {
        lookupFile = createLookupFile(file);  // 首次访问，创建 SST 文件
        newCreatedLookupFile = true;
    }
    byte[] keyBytes = keySerializer.serializeToBytes(key);
    byte[] valueBytes = lookupFile.get(keyBytes);  // 在 SST 文件中查找
    if (valueBytes == null) return null;
    return getOrCreateProcessor(lookupFile.schemaId(), lookupFile.serVersion())
            .readFromDisk(key, lookupFile.level(), valueBytes, file.fileName());
}
```

```java
// 源码: paimon-core/.../mergetree/LookupLevels.java:125-128
@Override
public void notifyDropFile(String file) {
    lookupFileCache.invalidate(file);  // Compaction 删除文件时使缓存失效
}
```

**面试口述建议：**

> "LookupLevels 是 Paimon 点查优化的核心。它为每个数据文件在本地构建 Lookup SST 文件，支持按 key 快速查找。用 Caffeine Cache 缓存 SST 文件，Compaction 删除旧文件时通过 DropFileCallback 失效缓存。查找时按 Level 从低到高——先查 Level0 按 sequence number 从新到旧，再查高层。还支持远程 Lookup 文件下载和 Schema 演进。关键优化点是避免了每次 Compaction 都要重读整个数据文件。"

---

## 七、运维与性能调优

<a id="q-7-1"></a>
### 7.1 Compaction 参数调优策略有哪些？

**🟡 中级**

**核心答案：**

Paimon Compaction 的核心参数及调优策略：

| 参数 | 默认值 | 作用 | 调优建议 |
|---|---|---|---|
| `num-sorted-run.compaction-trigger` | 5 | SortedRun 数量达到此值触发 Compaction | 降低 → 更及时合并，读放大小；升高 → 写放大小 |
| `num-sorted-run.stop-trigger` | trigger + 3 | 达到此值阻塞写入 | 与 trigger 的差值决定了缓冲空间 |
| `compaction.max-size-amplification-percent` | 200 | 空间放大百分比阈值 | 降低 → 更频繁全量合并，空间利用率高 |
| `compaction.size-ratio` | 1 | Size Ratio 合并的比例系数 | 升高 → 更容易合并相邻 run |
| `target-file-size` | 128MB | 目标文件大小 | 小文件多可适当增大 |
| `compaction.optimization-interval` | null | 定时全量合并间隔 | 需要定期 Full Compaction 时配置 |
| `write-only` | false | 是否禁用 Compaction | 写入密集时设 true，单独起 Compaction 作业 |
| `num-levels` | 动态 | LSM-Tree 的层数 | 一般不需手动配置 |

**核心策略**：
- **写入密集型**：增大 `num-sorted-run.compaction-trigger`，或设置 `write-only=true` 加独立 Compaction 作业
- **读取敏感型**：降低 `num-sorted-run.compaction-trigger`，减少 SortedRun 数量
- **空间敏感型**：降低 `max-size-amplification-percent`
- **Lookup 场景**：使用 `ForceUpLevel0Compaction`，通过 `lookup-compact-max-interval` 控制合并频率

**源码证据：**

```java
// 源码: paimon-core/.../compact/MergeTreeCompactManagerFactory.java:191-223
private CompactStrategy createCompactStrategy(CoreOptions options) {
    if (options.needLookup()) {
        // Lookup 场景: 根据 GENTLE/RADICAL 模式决定 compactMaxInterval
        Integer compactMaxInterval = null;
        switch (options.lookupCompact()) {
            case GENTLE:
                compactMaxInterval = options.lookupCompactMaxInterval();
                break;
            case RADICAL:
                break;
        }
        return new ForceUpLevel0Compaction(
                new UniversalCompaction(
                        options.maxSizeAmplificationPercent(),
                        options.sortedRunSizeRatio(),
                        options.numSortedRunCompactionTrigger(),
                        EarlyFullCompaction.create(options),
                        OffPeakHours.create(options)),
                compactMaxInterval);
    }
    // 普通场景
    UniversalCompaction universal =
            new UniversalCompaction(
                    options.maxSizeAmplificationPercent(),
                    options.sortedRunSizeRatio(),
                    options.numSortedRunCompactionTrigger(),
                    EarlyFullCompaction.create(options),
                    OffPeakHours.create(options));
    if (options.compactionForceUpLevel0()) {
        return new ForceUpLevel0Compaction(universal, null);
    } else {
        return universal;
    }
}
```

**面试口述建议：**

> "Paimon Compaction 调优的核心参数有三组。触发参数：num-sorted-run.compaction-trigger 控制何时开始合并，stop-trigger 控制何时阻塞写入。策略参数：max-size-amplification-percent 控制空间放大，size-ratio 控制相邻 run 的合并阈值。高级参数：optimization-interval 定时全量合并，write-only 禁用写入时合并。策略选择上，写入密集型增大 trigger 或用 write-only，读取敏感型降低 trigger，Lookup 场景会自动使用 ForceUpLevel0Compaction。"

---

<a id="q-7-2"></a>
### 7.2 Bucket 选择对性能有什么影响？

**🟢 基础**

**核心答案：**

Bucket 是 Paimon 在每个分区内部的数据分片单位。每个 bucket 独立管理自己的 LSM-Tree（对于主键表）或文件列表（对于 append 表）。

**Bucket 数量的影响**：

- **过少**：每个 bucket 内数据量大，单个 Compaction 任务耗时长，可能成为瓶颈。Flink 的写入并行度受限于 bucket 数量。
- **过多**：每个 bucket 的数据量小，产生大量小文件。LSM-Tree 的 Level0 文件频繁 flush 但数据量少，增加 I/O 开销。
- **合理值**：建议每个 bucket 的数据量在几百 MB 到几 GB 之间。

**Bucket 模式**：
- **FIXED**：固定 bucket 数量，通过 `bucket = N` 配置。适合数据量可预估的场景。
- **DYNAMIC**：动态调整 bucket 数量。适合数据量波动较大的场景。
- **BUCKET_UNAWARE**：不使用 bucket 概念，数据直接写入。适合 append-only 表。

Bucket 数量直接影响了 Flink 的写入并行度——每个 Writer 实例负责一个或多个 bucket，bucket 数量过少会导致写入成为瓶颈。

**面试口述建议：**

> "Bucket 是分区内的数据分片单位，每个 bucket 有独立的 LSM-Tree。数量太少导致单个 bucket 数据过大，Compaction 慢且写入并行度受限。太多则产生大量小文件。建议每个 bucket 几百 MB 到几 GB。Paimon 有三种 bucket 模式：固定数量、动态调整、无 bucket（append-only）。选择时要综合考虑数据量、写入并行度和文件数量。"

---

<a id="q-7-3"></a>
### 7.3 小文件问题的成因和治理方案是什么？

**🟡 中级**

**核心答案：**

**小文件成因**：

1. **高频 Checkpoint**：每次 Flink checkpoint 都会 flush Write Buffer 产生 Level0 文件。checkpoint 间隔越短，文件越小。
2. **Bucket 数量过多**：数据分散到过多 bucket，每个 bucket 的文件更小。
3. **Compaction 不及时**：写入速度超过 Compaction 速度，Level0 文件堆积。
4. **数据倾斜**：某些 bucket 数据很少，但每次 checkpoint 都会产生文件。

**治理方案**：

1. **调大 `target-file-size`**：增加目标文件大小（默认 128MB），让 Compaction 产生更大的文件。
2. **调大 Checkpoint 间隔**：减少 flush 频率，让每次 flush 的数据量更大。
3. **合理设置 Bucket 数量**：根据数据量合理预估 bucket 数。
4. **独立 Compaction 作业**：设置 `write-only=true`，单独起一个 Compaction 作业（更多资源专门做合并）。
5. **配置 `compaction.optimization-interval`**：定期触发 Full Compaction，将所有小文件合并为大文件。
6. **Sort Compact**：通过 Sort Compaction 按特定列排序后重写，同时合并小文件。
7. **Append 表的异步 Compaction**：append-only 表可以配置后台 Compaction 合并小文件。

**面试口述建议：**

> "小文件主要有四个成因：高频 checkpoint、bucket 过多、Compaction 不及时、数据倾斜。治理方案包括：调大 target-file-size、调大 checkpoint 间隔、合理设置 bucket 数量、独立 Compaction 作业、定期 Full Compaction。Paimon 的 LSM-Tree 架构天然通过 Compaction 合并小文件，关键是确保 Compaction 能跟上写入速度。"

---

<a id="q-7-4"></a>
### 7.4 Snapshot 过期的多重保护机制是什么？

**🟠 高级**

**核心答案：**

Paimon 的 Snapshot 过期由 `SnapshotDeletion` 和相关的 expire 逻辑实现，有多重保护机制防止误删：

**1. 最小保留数量**（`snapshot.num-retained.min`）：至少保留这么多个 Snapshot，即使它们已经过期。

**2. 最大保留数量**（`snapshot.num-retained.max`）：最多保留这么多个 Snapshot，超出的按时间顺序删除最旧的。

**3. 时间保留**（`snapshot.time-retained`）：在指定时间内的 Snapshot 不会被删除。

**4. Tag 保护**：被 Tag 引用的 Snapshot 不会被删除，因为 Tag 代表用户标记的重要快照。

**5. Consumer 保护**：如果有流式消费者（Consumer）还在消费某个 Snapshot 之后的数据，该 Snapshot 及其之后的所有 Snapshot 不会被删除。

**6. Changelog 保护**：如果启用了 changelog，相关的 changelog 文件在被消费前不会被删除。

**7. 原子性保护**：过期操作中，先删除数据文件和 Manifest 文件，最后才删除 Snapshot 文件本身。如果过程中失败，下次过期操作会重试。

**源码证据：**

```java
// 源码: paimon-core/.../operation/SnapshotDeletion.java:41-43
public class SnapshotDeletion extends FileDeletionBase<Snapshot> {
    private final boolean produceChangelog;
    // ...
}
```

**面试口述建议：**

> "Paimon 的 Snapshot 过期有多重保护。数量保护有 min 和 max 两个阈值。时间保护通过 time-retained 配置。Tag 保护确保被标记的 Snapshot 不被删除。Consumer 保护确保流式消费者还在用的 Snapshot 不被删除。Changelog 保护确保未消费的变更日志不丢。最后是原子性——先删数据文件再删 Snapshot 文件，失败可重试。这些机制层层保护，确保不会误删有用的数据。"

---

## 八、Paimon vs Iceberg 对比

<a id="q-8-1"></a>
### 8.1 存储模型的根本差异是什么？

**🟢 基础**

**核心答案：**

| 对比维度 | Paimon | Iceberg |
|---|---|---|
| **核心模型** | LSM-Tree（分层合并树） | 不可变文件 + Snapshot |
| **更新方式** | 写入 → Write Buffer → Level0 → Compaction 合并 | COW（重写文件）或 MOR（Delete File + 读时合并） |
| **主键支持** | 原生支持，主键表用 KeyValueFileStore | V2 引入 Equality Delete，但本质是 MOR |
| **存储结构** | 每个 bucket 是一个 LSM-Tree（分层文件） | 扁平的数据文件 + 独立的元数据层 |
| **元数据管理** | Snapshot → ManifestList → Manifest → DataFileMeta | Snapshot → ManifestList → Manifest → DataFile |
| **写放大** | 较低（LSM 分层合并） | COW 极高（重写整个文件），MOR 较低 |
| **读放大** | 取决于 SortedRun 数量 | COW 无放大，MOR 需合并 Delete File |

**根本差异**：Paimon 的 LSM-Tree 模型是为**高频流式更新**而生的——数据先到内存再异步合并，天然适合 upsert。Iceberg 的不可变文件模型更偏向**批处理**——每次写入产生完整的文件，适合大批量追加。

**面试口述建议：**

> "根本差异在于存储模型。Paimon 用 LSM-Tree，数据先写内存缓冲区再异步合并到磁盘，天然适合高频更新。Iceberg 用不可变文件模型，更新要么重写整个文件(COW)要么产生 Delete File(MOR)。Paimon 的写放大较低但有读放大，Iceberg 的 COW 写放大极高但读性能好。两者的元数据管理形式上类似，都是 Snapshot → ManifestList → Manifest 的层级结构。"

---

<a id="q-8-2"></a>
### 8.2 流式更新能力有什么本质区别？

**🟡 中级**

**核心答案：**

**Paimon 的流式更新是原生能力**：
- 直接支持 upsert/delete 操作，写入路径天然处理 RowKind（INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE）
- 四种 MergeFunction 提供丰富的合并语义
- Changelog 产生机制（INPUT/LOOKUP/FULL_COMPACTION）为下游提供完整的 CDC 语义
- 反压机制（shouldWaitForLatestCompaction）保证合并跟得上写入

**Iceberg 的流式更新是后加的能力**：
- V1 不支持行级更新，V2 通过 Equality Delete 和 Position Delete 实现
- 更新本质是"写新数据 + 写删除标记"，读时合并
- 缺乏原生的 changelog 产生能力，需要引擎层（Flink）额外处理
- 没有内置的合并策略（如 PartialUpdate、Aggregation）

**关键对比**：
- Paimon 的 Sequence Number 是存储层原生管理的，而 Iceberg 需要依赖引擎层
- Paimon 的 Compaction 自动处理旧版本数据清理，Iceberg 需要显式的 Rewrite Data Files
- Paimon 的 Lookup 机制可以实时产生精确 changelog，Iceberg 没有等价能力

**面试口述建议：**

> "Paimon 的流式更新是原生设计——LSM-Tree 天然支持 upsert，四种 MergeFunction 提供丰富语义，Changelog 产生机制为下游提供完整 CDC。Iceberg 的更新是后加的——通过 Equality Delete 和 Position Delete 实现 MOR，本质是写删除标记、读时合并。Paimon 还有 Iceberg 没有的能力：PartialUpdate、Aggregation、实时 Changelog 产生。核心差异是 Paimon 把更新作为第一公民，Iceberg 把它作为扩展能力。"

---

<a id="q-8-3"></a>
### 8.3 小文件治理理念有什么不同？

**🟡 中级**

**核心答案：**

| 对比维度 | Paimon | Iceberg |
|---|---|---|
| **治理时机** | 写入时自动合并（后台 Compaction） | 写入后显式触发（Rewrite Data Files Action） |
| **触发方式** | 自动（SortedRun 数量驱动） | 手动/定时任务（用户主动调用） |
| **合并粒度** | 按 bucket 内的 Level 合并 | 按 Bin-Pack/Sort/Z-Order 策略重写 |
| **对写入的影响** | 后台线程异步执行，有反压机制 | 独立任务，不影响写入但占用资源 |
| **理念** | **写入即治理**——Compaction 是 LSM-Tree 的核心组成部分 | **事后治理**——小文件合并是独立的维护操作 |

**核心区别**：

Paimon 的理念是"小文件是 LSM-Tree 的正常状态，Compaction 自动处理"。每次 flush 产生的 Level0 小文件会被 Universal Compaction 自动合并到高层。如果 Compaction 跟不上，反压机制会减慢写入。

Iceberg 的理念是"写入尽量产生大文件，小文件是异常状态需要治理"。Iceberg 的 Rewrite Data Files 是一个独立的 Action，支持 BinPack（按大小合并）、Sort（排序后重写）、Z-Order（多维排序重写）三种策略。

**面试口述建议：**

> "两者的小文件治理理念截然不同。Paimon 认为小文件是 LSM-Tree 的正常状态，通过内置的 Universal Compaction 自动异步合并，是'写入即治理'。Iceberg 认为小文件是异常状态，通过独立的 Rewrite Data Files Action 事后治理。Paimon 的优势是自动化不需要人为干预，劣势是 Compaction 消耗后台资源。Iceberg 的优势是治理策略更灵活（BinPack/Sort/Z-Order），劣势是需要用户主动触发或配置定时任务。"

---

> **文档版本说明**：本文档基于 Apache Paimon 1.5-SNAPSHOT (master 分支，commit: 55f4fd175) 源码分析编写。
