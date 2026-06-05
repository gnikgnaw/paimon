# Catalog 与元数据管理：源码专家手册

> 版本：Apache Paimon 1.5-SNAPSHOT
> 源码模块：`paimon-api`（Catalog 接口、Identifier、Snapshot、TableSchema、SchemaChange、REST auth）、`paimon-core`（AbstractCatalog / FileSystemCatalog / CachingCatalog / RESTCatalog、SchemaManager、SnapshotManager、TagManager、BranchManager、manifest、operation/commit）、`paimon-common`（RESTTokenFileIO、FileIO 原子写、HintFileUtils）
> 更新日期：2026-06-03

**一句话定位。** Catalog 子系统是 Paimon 元数据的"控制面"——它把"库 / 表 / 分区 / 快照 / 标签 / 分支"这些逻辑对象，落到文件系统目录、Hive Metastore、JDBC 库或远端 REST Server 上，并通过**乐观并发控制**（文件原子创建 + 重试退避，或服务端仲裁）保证多写并发安全。它的两个设计支柱是：（1）**文件即元数据**——Schema / Snapshot / Manifest 全是 JSON 或自描述二进制文件，无需"读-改-写"整块元数据；（2）**装饰器 + SPI**——`PrivilegedCatalog → CachingCatalog → {FileSystem|Hive|Jdbc|REST}Catalog` 链式叠加，由 `CatalogFactory` 通过 Java SPI 装配。读完这篇你应当能够：改 commit 的冲突检测逻辑、定位"提交反复重试 / 失败"的线上问题、读懂 REST Catalog 的 token 刷新与 PVFS data-token 走向、判断一次 `ALTER TABLE` 为何卡住、知道一次写入到底产生几个 snapshot。

---

## 目录

- [一、全局架构与数据流](#一全局架构与数据流)
- [二、Catalog 接口体系](#二catalog-接口体系)
  - [2.1 类层次与装配链](#21-类层次与装配链)
  - [2.2 Catalog 接口契约与能力探测](#22-catalog-接口契约与能力探测)
  - [2.3 AbstractCatalog 模板方法](#23-abstractcatalog-模板方法)
  - [2.4 FileSystemCatalog 与原子锁](#24-filesystemcatalog-与原子锁)
  - [2.5 DelegateCatalog / CachingCatalog](#25-delegatecatalog--cachingcatalog)
  - [2.6 CatalogFactory SPI 与四种 metastore](#26-catalogfactory-spi-与四种-metastore)
  - [2.7 Identifier 标识解析](#27-identifier-标识解析)
- [三、REST Catalog（云原生重点）](#三rest-catalog云原生重点)
  - [3.1 RESTCatalog 与 RESTApi 分层](#31-restcatalog-与-restapi-分层)
  - [3.2 认证体系：AuthProvider / Bear / DLF](#32-认证体系authprovider--bear--dlf)
  - [3.3 DLF 签名与 token 刷新](#33-dlf-签名与-token-刷新)
  - [3.4 Data Token、RESTTokenFileIO 与 PVFS](#34-data-tokenresttokenfileio-与-pvfs)
  - [3.5 REST 并发控制与排障](#35-rest-并发控制与排障)
- [四、Schema 演进机制](#四schema-演进机制)
  - [4.1 TableSchema 字段与派生量](#41-tableschema-字段与派生量)
  - [4.2 SchemaManager 与乐观锁 commit](#42-schemamanager-与乐观锁-commit)
  - [4.3 SchemaChange 类型与生成纯函数](#43-schemachange-类型与生成纯函数)
  - [4.4 类型变更校验与列映射读取](#44-类型变更校验与列映射读取)
  - [4.5 Schema 存储路径与回退分支](#45-schema-存储路径与回退分支)
- [五、Snapshot 管理](#五snapshot-管理)
  - [5.1 Snapshot 字段表（v3）](#51-snapshot-字段表v3)
  - [5.2 base/delta/changelog 三分法](#52-basedeltachangelog-三分法)
  - [5.3 SnapshotManager 与 LATEST hint](#53-snapshotmanager-与-latest-hint)
  - [5.4 快照过期与去重查询](#54-快照过期与去重查询)
- [六、Tag 与 Branch](#六tag-与-branch)
  - [6.1 TagManager 与 Tag TTL](#61-tagmanager-与-tag-ttl)
  - [6.2 BranchManager 写时复制](#62-branchmanager-写时复制)
- [七、Manifest 三级结构](#七manifest-三级结构)
  - [7.1 三级结构与 ObjectsFile](#71-三级结构与-objectsfile)
  - [7.2 ManifestFileMeta 过滤剪枝](#72-manifestfilemeta-过滤剪枝)
  - [7.3 ManifestEntry 与合并语义](#73-manifestentry-与合并语义)
  - [7.4 Manifest 合并时机与策略](#74-manifest-合并时机与策略)
- [八、Commit 乐观锁全流程](#八commit-乐观锁全流程)
  - [8.1 commit() 与 APPEND/COMPACT 分离](#81-commit-与-appendcompact-分离)
  - [8.2 tryCommit 重试退避](#82-trycommit-重试退避)
  - [8.3 tryCommitOnce 去重与增量冲突检测](#83-trycommitonce-去重与增量冲突检测)
  - [8.4 ConflictDetection 多层检查](#84-conflictdetection-多层检查)
  - [8.5 Overwrite 分区覆写](#85-overwrite-分区覆写)
  - [8.6 提交排障速查](#86-提交排障速查)
- [九、系统表与 CoreOptions](#九系统表与-coreoptions)
- [十、与 Iceberg 的设计对比](#十与-iceberg-的设计对比)
- [十一、设计决策总结](#十一设计决策总结)

---

## 一、全局架构与数据流

Paimon 把"元数据"拆成四类持久化对象，每类都是独立文件、独立生命周期：

| 对象 | 物理形态 | 写入者 | 谁引用它 |
|---|---|---|---|
| TableSchema | `schema/schema-{id}`（JSON） | SchemaManager | Snapshot.schemaId、每个 DataFileMeta.schemaId、ManifestFileMeta.schemaId |
| Snapshot | `snapshot/snapshot-{id}`（JSON） | FileStoreCommitImpl | LATEST hint、Tag、Branch、消费者位点 |
| ManifestList / ManifestFile | `manifest/*`（自描述二进制 ObjectsFile） | manifestList / manifestFile.write | Snapshot 的三个 manifestList 字段、indexManifest |
| Tag / Branch | `tag/tag-{name}`、`branch/branch-{name}/...` | TagManager / BranchManager | 用户引用、过期保护 |

控制面（Catalog）只负责"库 / 表的目录结构 + schema 入口"；一旦拿到 `Table` 对象，数据面的读写（Snapshot / Manifest / Commit）就由 `FileStore` 体系接管。装配链如下：

```
计算引擎 (Flink / Spark / Hive / pypaimon)
        │ CatalogFactory.createCatalog(context)
        ▼
  PrivilegedCatalog        (权限装饰器, 可选 — privilege/*)
        │ wrapped
        ▼
  CachingCatalog           (Caffeine 缓存装饰器, 默认开启)
        │ wrapped
        ▼
  ┌─────────────┬─────────────┬──────────────┬──────────────┐
  │FileSystemCat│ HiveCatalog │ JdbcCatalog  │ RESTCatalog  │
  │(目录即元数据)│(Hive MS)    │(关系库)       │(远端 Server) │
  └──────┬──────┴──────┬──────┴──────┬───────┴──────┬───────┘
         │             │             │              │ RESTApi(HTTP)
         ▼             ▼             ▼              ▼
   SchemaManager / SnapshotManager / Manifest* / FileStoreCommitImpl
         │
         ▼
   FileIO 抽象 (HDFS / S3 / OSS / 本地 / RESTTokenFileIO 包裹的临时凭证 FileIO)
```

注意四处易被误解的边界：

1. **`RESTCatalog` 不继承 `AbstractCatalog`** —— 它直接 `implements Catalog`，因为它几乎所有方法都转发给 `RESTApi` 的 HTTP 调用，模板方法那套校验逻辑由它自己内联（仍调用 `CatalogUtils.checkNotSystemTable` 等静态方法）。`FileSystemCatalog` / `HiveCatalog` / `JdbcCatalog` 才继承 `AbstractCatalog`。
2. **缓存层在装饰链中间**，所以 REST 也能被 `CachingCatalog` 包裹；但 REST 还有自己内部的 token / FileIO 缓存（见 3.4），两套缓存目标不同——一个缓存逻辑表对象，一个缓存按凭证构造的物理 FileIO。
3. **数据文件的 FileIO 与元数据 FileIO 可能不同**：REST 模式下元数据走 catalog token，数据文件走每表独立的 data token（`RESTTokenFileIO`）。
4. **一次写入提交可能产生 0、1 或 2 个 snapshot**：APPEND 与 COMPACT 各自一次提交（见第八章），这是理解 Paimon 提交语义的第一道坎。

**为什么把元数据拆成独立文件而非单块 metadata？** Iceberg / Delta 把 schema、partition spec、快照引用都塞进一个 `metadata.json`（或 `_delta_log`），每次变更要"读整块 + 改 + 写整块"。Paimon 选择拆分：schema 改了只写一个新 `schema-{id}`，快照提交只写一个新 `snapshot-{id}` + 增量 manifest，互不干扰。代价是读取路径要多次 IO（先读 snapshot、再读 manifest list、再读 manifest、再按 schemaId 取 schema），但换来了"每类元数据各自乐观锁、各自演进"的解耦能力，这正是 Paimon 支持高频流式提交的基础。

**读路径数据流（拿到 Table 之后）：**

```
Table.newReadBuilder()
  → TableScan.plan()
      → SnapshotManager.latestSnapshot()        # LATEST hint + 验证(见 5.3)
      → ManifestList.readDataManifests(snapshot) # base + delta 的 ManifestFileMeta
      → 用 ManifestFileMeta.partitionStats 按分区谓词剪枝(见 7.2)
      → ManifestFile.read(meta) 读 ManifestEntry
      → FileEntry.mergeEntries 合并 ADD/DELETE 得有效文件列表
      → 按 DataFileMeta.schemaId 取对应 schema 做列映射(见 4.4)
  → 产出 Split[]
TableRead.createReader(split) → 读实际数据文件(ORC/Parquet/Avro)
```

**写路径数据流：**

```
Table.newBatchWriteBuilder()/newStreamWriteBuilder()
  → TableWrite.write(row)                        # 缓冲 + 排序 + 落数据文件, 产 CommitMessage
  → TableCommit.commit(ManifestCommittable)
      → FileStoreCommitImpl.commit(...)          # 第八章主线
          → collectChanges 分桶 → tryCommit(APPEND) [+ tryCommit(COMPACT)]
              → tryCommitOnce: 去重 → 冲突检测 → 构建 Snapshot → 原子写
```

把这两条链记在脑里，后续每章都是在补这条主干上某个节点的细节。

**贯穿全篇的两个不变式**（看任何细节时回到这两条）：

1. **写永不改旧文件，只追加新文件 + 原子写一个新 snapshot**。schema、manifest、data 都只新增不改；唯一的"切换"动作是原子写 `snapshot-{id+1}`（或 REST 服务端提交）。这让任何写入要么完整生效（新 snapshot 可见）要么完全不可见（新 snapshot 没写成），天然原子。
2. **读永远从一个确定的 snapshot 出发，向下展开**。snapshot → manifest list → manifest → data，每层靠文件名 / id 引用下一层，按 schemaId 关联 schema。读到哪个 snapshot 就看到哪个一致快照，互不干扰。

乐观锁、精确一次、时间旅行、分支、Tag——全是这两个不变式的推论。后续每个机制若看不懂，回到"它如何维持这两条"通常就能想通。

**index manifest 的位置**：除三个 data manifest list，Snapshot 还有一个 `indexManifest` 字段，指向索引文件清单（Deletion Vector 文件、file index 等的元数据）。它与 data manifest 平行——data manifest 管"哪些数据文件存在"，index manifest 管"这些数据文件附带哪些索引 / 删除向量"。提交时 `indexManifestFile.writeIndexFiles(oldIndexManifest, indexFiles)` 增量更新（复用旧的 + 加新的），与 data manifest 的合并逻辑独立。读取 merge-on-read 表时先读 index manifest 拿到每个数据文件的 DV，再据此跳过被删的行——这是 DV 表读路径多出来的一步，但避免了"删一行重写整文件"的代价。

至此元数据全景：Snapshot 是中枢，向下挂 4 类清单（base / delta / changelog data manifest + index manifest），向上被 LATEST hint / Tag / Branch / 消费者位点引用，横向用 schemaId 关联 schema、用 commitUser/Identifier 支撑精确一次。后续各章逐一拆解这张图的每个节点。

---

## 二、Catalog 接口体系

### ① 要解决什么问题

数据湖要被 Flink / Spark / Hive / Python 多引擎访问，每个引擎都需要"建库建表、改 schema、列分区、管快照 / 标签 / 分支"。若无统一抽象，每个引擎各写一套，必然出现行为不一致（例如 A 引擎允许删主键列、B 引擎不允许；A 把系统表当普通表删掉）。同时部署形态差异巨大：开发机要零依赖（filesystem）、传统数仓要接 Hive、云上要 REST、轻量场景要 JDBC。Catalog 抽象把"做什么"（接口）与"怎么存"（实现）解耦，并用装饰器叠加缓存 / 权限等横切关注点。

### ② 设计原理与取舍（最重要）

- **模板方法 + 委托双模式并存**：`AbstractCatalog` 用模板方法固化校验与路由（系统库 / 系统表 / 分支拦截、建表类型分派），子类只填 `xxxImpl()`；`DelegateCatalog` 用委托模式做装饰器，二者职责正交。模板方法解决"通用逻辑下沉"，委托模式解决"横切能力叠加"——若只用继承做装饰器会导致组合爆炸（缓存×权限×后端三维组合）。
- **能力探测而非异常驱动**：不同后端能力天差地别（filesystem 不支持分页 list、不支持 alterDatabase；REST 原生支持分页与版本管理）。用 `supportsListObjectsPaged()` / `supportsVersionManagement()` / `caseSensitive()` 让上层优雅降级，避免到处 try-catch `UnsupportedOperationException`——异常驱动的代价是"用失败来探测能力"，既慢又脏。
- **接口分组而非扁平大接口**：Database / Table / Partition / 版本管理 / 能力探测五组方法逻辑清晰，但仍在一个接口里（而非拆成多个小接口），是为让装饰器只需实现一个 `wrapped`，避免多接口转发的样板。
- **取舍**：filesystem 后端简单零依赖，代价是缺数据库属性、缺分页、对象存储下 rename 非原子需外部锁；REST 后端能力最全但引入网络往返与可用性依赖。Paimon 把"默认最简单"作为优先（filesystem 是 default），让上手成本最低。

### ③ 关键源码

能力探测与系统库常量（`paimon-core/.../catalog/Catalog.java:650-1231`）：

```java
boolean supportsListObjectsPaged();
default boolean supportsListByPattern() { ... }
default boolean supportsListTableByType() { ... }
boolean supportsVersionManagement();        // Tag/Branch/Snapshot 版本能力
boolean supportsPartitionModification();
boolean caseSensitive();
String SYSTEM_DATABASE_NAME = "sys";        // 保留库名，禁止用户创建
```

### ④ 风险/陷阱/边界

- 系统表（`my_table$snapshots`）、分支引用（`my_table$branch_dev`）**不能直接 DDL**：`AbstractCatalog` 在 `dropTable` / `createTable` / `renameTable` 入口用 `checkNotSystemTable` / `checkNotBranch` 拦截，抛异常而非静默忽略。删分支必须走 `BranchManager.dropBranch()`。
- `FileSystemCatalog.alterDatabase()` 抛 `UnsupportedOperationException`——目录无属性存储位。需要库属性请用 hive / jdbc / rest。
- 对象存储 warehouse 下 `renameTable` 危险：S3 / OSS 的 rename = copy+delete 非原子，大表迁移中途失败会留半态。
- `caseSensitive()` 在 hive 后端常返回 false（Hive Metastore 库表名大小写不敏感），上层若用大小写区分的名字会撞表——跨后端迁移要先确认这一项。

### ⑤ 收益与代价

收益：单一接口适配四种后端、横切能力可插拔、上层引擎代码与存储解耦、能力降级优雅。代价：抽象层带来一层间接调用与"能力矩阵"心智负担；filesystem 的功能缺口需用文档与异常显式暴露。

**Catalog 层排障速查**：

| 现象 | 根因 | 处置 |
|---|---|---|
| `UnsupportedOperationException: alterDatabase` | filesystem 后端不支持库属性 | 换 hive / jdbc / rest |
| `DROP TABLE t$snapshots` 报错 | 系统表不可 DDL | 操作原表，系统表只读 |
| `DROP TABLE t$branch_dev` 报错 | 分支不可当表删 | 用 `dropBranch` |
| 读到陈旧元数据 | 多进程共享 warehouse + 缓存未过期 | 调短 `cache.expire-after-write` 或 `invalidateTable` |
| 对象存储建表 / 改 schema 偶发丢失 | 无原子 rename 未配锁 | hive / jdbc + `lock.enabled=true` |
| getTable 慢（循环里） | 缓存被频繁 invalidate 或未启用 | 启用缓存、避免循环内 invalidate |
| 跨后端迁移后表名撞车 | hive 大小写不敏感（`caseSensitive=false`） | 迁移前统一命名规范 |
| REST getTable 报 403 | token 失效 / 时钟漂移 / 权限不足 | 查刷新日志、校 NTP、查服务端鉴权 |
| REST 频繁初始化慢 | 每次 new Catalog 都拉 config | 复用 Catalog 实例 |

### 2.1 类层次与装配链

```
Catalog (interface, AutoCloseable)              paimon-core: catalog/Catalog.java
├─ AbstractCatalog (abstract, 模板方法)          catalog/AbstractCatalog.java
│   ├─ FileSystemCatalog                         catalog/FileSystemCatalog.java
│   ├─ HiveCatalog                               paimon-hive
│   └─ JdbcCatalog                               catalog/JdbcCatalog
├─ DelegateCatalog (abstract, 委托/装饰器)        catalog/DelegateCatalog.java
│   ├─ CachingCatalog                            catalog/CachingCatalog.java
│   └─ PrivilegedCatalog                         privilege/PrivilegedCatalog.java
└─ RESTCatalog (直接 implements)                 rest/RESTCatalog.java
```

`DelegateCatalog.rootCatalog()` 静态方法可剥去所有装饰层拿到最底层实现：

```java
public static Catalog rootCatalog(Catalog catalog) {
    while (catalog instanceof DelegateCatalog) {
        catalog = ((DelegateCatalog) catalog).wrapped();
    }
    return catalog;
}
```

常用于需要绕过缓存 / 权限直接操作底层的场景（如内部维护工具直接拿 `FileSystemCatalog` 的 `fileIO`）。

**为何 `AbstractCatalog` 与 `DelegateCatalog` 并列两条线，而非让装饰器也继承 `AbstractCatalog`？** 因为模板方法的 `xxxImpl()` 假设"我就是真正干活的人"，而装饰器的本质是"我把活转交给 wrapped"。两者的契约根本不同——硬塞进一条继承链会让装饰器被迫实现一堆它本不该实现的 `xxxImpl()`。

### 2.2 Catalog 接口契约与能力探测

> 源码：`paimon-core/.../catalog/Catalog.java`（`interface Catalog extends AutoCloseable`，约 1230 行）

按职责分组（仅列关键方法，签名以源码为准）：

**Database**：`listDatabases()` / `listDatabasesPaged(maxResults, pageToken, namePattern)` / `createDatabase` / `getDatabase` / `dropDatabase(name, ignoreIfNotExists, cascade)` / `alterDatabase`。

**Table**：`getTable(identifier)` / `getTableById` / `listTables` / `listTablesPaged` / `createTable` / `dropTable` / `renameTable` / `alterTable(identifier, List<SchemaChange>, ignoreIfNotExists)` / `invalidateTable`。

**Partition**：`listPartitions` / `listPartitionsPaged` / `markDonePartitions` / `alterPartitions`（受 `supportsPartitionModification()` 约束）。

**版本管理（受 `supportsVersionManagement()` 约束）**：`commitSnapshot(identifier, tableUuid, snapshot, statistics)` / `loadSnapshot` / `createTag` / `deleteTag` / `createBranch` / `dropBranch` / `fastForward` / `rollbackTo`。

**能力探测**：`supportsListObjectsPaged` / `supportsListByPattern` / `supportsListTableByType` / `supportsVersionManagement` / `supportsPartitionModification` / `caseSensitive`。

把 `commitSnapshot` 设计成接口方法是关键——它让"快照提交"可以由服务端做并发仲裁（REST 后端，见 3.5），而 filesystem 后端则靠本地原子文件创建。同理 `createTag` / `createBranch` / `rollbackTo` 进接口，意味着 REST 后端可以把这些版本操作的事务性收敛到服务端，而不必由客户端做多步文件操作（多步操作在网络不可靠时难保原子）。

### 2.3 AbstractCatalog 模板方法

> 源码：`paimon-core/.../catalog/AbstractCatalog.java`

公开方法（含校验）→ 抽象方法（子类实现）的映射：

```
createDatabase()  → createDatabaseImpl()      dropTable()    → dropTableImpl()
getDatabase()     → getDatabaseImpl()         renameTable()  → renameTableImpl()
dropDatabase()    → dropDatabaseImpl()        alterTable()   → alterTableImpl()
listTables()      → listTablesImpl()          createTable()  → createTableImpl()
```

核心字段：`fileIO`（读写元数据文件）、`tableDefaultOptions`（全局表默认配置，会在建表时 merge 进表 options）、`context`（CatalogContext）。

核心通用逻辑：

1. **系统库 / 表拦截**：`createDatabase` 调 `checkNotSystemDatabase`（禁止建 `sys`）；`dropTable` 调 `checkNotSystemTable`。
2. **分支拦截**：DDL 入口调 `checkNotBranch`，防止把 `$branch_x` 当普通表操作。
3. **建表分派**：按 `CoreOptions.TYPE` 路由——`TABLE` / `MATERIALIZED_TABLE` 走 `createTableImpl`，`FORMAT_TABLE` 走 `createFormatTable`，`OBJECT_TABLE` 在 filesystem 下抛 `UnsupportedOperationException`：

```java
switch (Options.fromMap(schema.options()).get(TYPE)) {
    case TABLE:
    case MATERIALIZED_TABLE:
        createTableImpl(identifier, schema);    // 正常表/物化表
        break;
    case FORMAT_TABLE:
        createFormatTable(identifier, schema);  // 纯格式表(CSV/Parquet 目录)
        break;
    case OBJECT_TABLE:
        throw new UnsupportedOperationException(...);
}
```

4. **getTable 统一加载**：`CatalogUtils.loadTable()` 同时处理普通表与系统表，`$` 分隔的系统表名自动经 `SystemTableLoader` 包装。
5. **锁开关**：`lockEnabled()` 依据 `fileIO.isObjectStore()` 判断——对象存储默认需要外部锁。

`AbstractCatalog` 还内置了 `CachingFileIO` / `LocalCacheManager` 的支持（其 import 可见），用于对元数据文件 IO 做本地缓存——这与 `CachingCatalog` 缓存逻辑表对象正交，是更底层的"文件字节缓存"。`FactoryUtil` 用于 SPI 发现各类 Factory（CatalogLockFactory、FileIO 等），是模板方法里"按配置装配可插拔组件"的统一入口。

**为何系统表路由要在抽象层？** 所有后端都需统一处理 `$snapshots` / `$schemas` 等路由，提升到抽象层避免各实现各判一遍导致行为漂移（比如某实现漏判 `$audit_log` 就会把它当普通表去文件系统找，得到莫名其妙的"表不存在"）。系统表本身无独立存储——它是对原表 Snapshot / Schema / Manifest 文件的运行时投影（`SystemTableLoader.load` 包装原表的 FileStore），所以"建 / 删系统表"无意义、被拦截。

**建表端到端 trace（filesystem 后端）**，理解每一步在哪个类：

```
Catalog.createTable(id, schema, ignoreIfExists)
  → AbstractCatalog.createTable:
       checkNotSystemTable / checkNotBranch(id)        # 拦截非法目标
       tableDefaultOptions 合并进 schema.options()     # 注入全局默认
       按 TYPE 分派 → createTableImpl(id, schema)
  → FileSystemCatalog.createTableImpl:
       runWithLock(id, () -> {
         SchemaManager(fileIO, tablePath).createTable(schema, false)  # 写 schema-0
       })
  → SchemaManager.createTable:
       latest() 若存在 → 抛"已存在"(除非 external)
       TableSchema.create(0, schema)
       FileStoreTableFactory.create(...).store()        # 用新 schema 试建表, 校验配置
       commit(newSchema) → fileIO.tryToWriteAtomic(schema-0)   # 原子落盘
  → CachingCatalog 包裹层: 建成后下次 getTable 缓存该表对象
```

注意 `FileStoreTableFactory.create(...).store()` 这步——它在写 schema-0 之前用候选 schema 实际构造一次 `FileStoreTable`，目的是把"bucket / 主键 / 序列字段 / merge-engine 配置互相矛盾"这类错误在建表时就暴露，而非等到第一次写入才崩。

### 2.4 FileSystemCatalog 与原子锁

> 源码：`paimon-core/.../catalog/FileSystemCatalog.java`

目录布局（main 分支）：

```
{warehouse}/{database}.db/{table}/
  schema/schema-{id}            # TableSchema JSON
  snapshot/snapshot-{id}        # Snapshot JSON  + LATEST/EARLIEST hint
  tag/tag-{name}                # Snapshot JSON (+TTL)
  branch/branch-{name}/{schema,snapshot,tag}/...
  manifest/{manifest-list-*, manifest-*, index-manifest-*}
  bucket-{n}/{data-files}       # 实际数据文件
```

关键方法实现：`listDatabases` 列 warehouse 下 `.db` 目录；`listTablesImpl` 列 `.db` 下含 `schema/` 子目录的目录；`createTableImpl` 经 `SchemaManager.createTable` 写 schema-0；`alterTableImpl` 经 `SchemaManager.commitChanges` 提交变更；`renameTableImpl` 用 `fileIO.rename`。

**为何"含 schema/ 子目录才算表"**：filesystem 后端没有"表注册表"，只能靠目录约定识别——`{db}.db/` 下的目录若有 `schema/` 子目录就是 Paimon 表，否则是普通目录（如临时目录、其它格式数据）。这也意味着手动 `mkdir foo/schema && cp schema-0 foo/schema/` 就能"恢复"一张被误删元数据但数据文件还在的表——filesystem 后端的元数据可手工修复，这是它运维上的一个隐性优势（REST 则不行，元数据在服务端）。

**manifest 目录文件命名**：`manifest-list-{uuid}-{n}`（ManifestFileMeta 列表）、`manifest-{uuid}-{n}`（ManifestEntry）、`index-manifest-{uuid}-{n}`（DV 等索引）。uuid 保证并发写不撞名，无需协调；这也是为什么 manifest 不复用、靠合并 + 过期清理回收（追加式 uuid 命名天然支持乐观并发）。

`createTableImpl` / `alterTableImpl` 用 `runWithLock()` 包裹——拿到 `CatalogLockFactory`（filesystem 默认无；hive / jdbc 提供）就 `Lock.fromCatalog(...)`，否则 `Lock.empty()`（HDFS 靠原子 rename 即可）：

```java
public <T> T runWithLock(Identifier identifier, Callable<T> callable) throws Exception {
    return lockFactory()
        .map(factory -> factory.createLock(lockContext().orElse(null)))
        .map(l -> Lock.fromCatalog(l, identifier))
        .orElseGet(Lock::empty)
        .runWithLock(callable);
}
```

`Lock`（`paimon-core/.../operation/Lock.java`）的 `EmptyLock` 直接 `callable.call()`，`CatalogLockImpl` 则把 `(database, objectName, callable)` 转给 `CatalogLock.runWithLock`——后者实现可能是 Hive MetaStore 锁或 JDBC 行锁。

`Lock` 接口标 `@Public @since 0.4.0`（稳定 API），`fromCatalog(catalogLock, identifier)` 在 catalogLock 为 null 时退回 `EmptyLock`——优雅处理"未配锁"的情形。这层抽象让"加不加锁、用什么锁"对 SchemaManager / FileStoreCommit 透明：它们只管 `runWithLock(callable)`，锁的有无与实现由 Catalog 装配决定。对象存储场景下，正是这把锁把"先 exists 再写"的 TOCTOU 窗口串行化（同一表的并发 DDL / commit 被锁排队），补上原子 rename 缺失的安全性。

**为何 alterDatabase 不支持**：库在 FS 上只是目录，无属性文件可持久化。**为何 rename 警示对象存储**：S3 / OSS 的 rename 非原子（copy+delete），大表迁移中途失败留半态。**为何 filesystem 默认无锁仍安全**：HDFS / 本地 FS 的 rename 是原子的，乐观锁基于"写 `schema-{id}` 已存在则失败"即可，无需外部锁；只有对象存储缺原子语义时才需补锁。

**`tryToWriteAtomic` 的两种实现路径**（这是整个乐观锁的物理基础，值得追到底）：

- **HDFS / 本地**：先写 `{path}.tmp.{uuid}` 临时文件，再 `rename(tmp, path)`；FS 保证 rename 原子且目标已存在则失败，返回 false。
- **对象存储**：若底层支持条件写（如 S3 的 If-None-Match / OSS 的 x-oss-forbid-overwrite）则用 `putIfAbsent` 语义；否则只能"先 exists 再写"，存在 TOCTOU 窗口，必须配 `CatalogLock` 串行化。这就是 concurrency-control 文档反复强调"对象存储要配 hive/jdbc + lock.enabled"的根因。

判断当前 FileIO 走哪条路径：`fileIO.isObjectStore()`——它同时决定 `AbstractCatalog.lockEnabled()` 是否默认开锁。

### 2.5 DelegateCatalog / CachingCatalog

> 源码：`catalog/DelegateCatalog.java`、`catalog/CachingCatalog.java`

`DelegateCatalog` 持有 `protected final Catalog wrapped`，所有方法默认透传，子类按需覆写：

```java
public abstract class DelegateCatalog implements Catalog {
    protected final Catalog wrapped;
    @Override public Table getTable(Identifier id) throws TableNotExistException {
        return wrapped.getTable(id);   // 其余方法同理透传
    }
}
```

不用接口默认方法的原因：Java 8 default 方法访问不了实例字段 `wrapped`。

`CachingCatalog` 的多级缓存（基于 shaded Caffeine）：

| 缓存 | 键/值 | 说明 |
|---|---|---|
| `databaseCache` | `String → Database` | 库元数据 |
| `tableCache` | `Identifier → Table` | 表对象（系统表不直接缓存，缓存原表后动态包装） |
| `partitionCache` | `Identifier → List<Partition>` | 默认关（`cache.partition-max-num=0`） |
| `manifestCache` | `SegmentsCache<Path>` | 按**内存字节**而非条目数限制，小文件入缓存、大文件直读 |
| `dvMetaCache` | DV 元数据 | Deletion Vector 元数据（可选） |

系统表缓存的关键技巧——只缓存原表，系统表运行时包装，多个系统表共享一份原表缓存：

```java
if (identifier.isSystemTable()) {
    Identifier origin = new Identifier(
        identifier.getDatabaseName(), identifier.getTableName(),
        identifier.getBranchName(), null);
    Table originTable = getTable(origin);                 // 命中原表缓存
    table = SystemTableLoader.load(
        identifier.getSystemTableName(), (FileStoreTable) originTable);
}
```

写操作后主动失效——`dropTable` 会清掉该表及其所有分支表的缓存条目：

```java
public void dropTable(Identifier identifier, boolean ignoreIfNotExists) {
    super.dropTable(identifier, ignoreIfNotExists);
    invalidateTable(identifier);
    for (Identifier i : tableCache.asMap().keySet()) {
        if (identifier.getTableName().equals(i.getTableName())
                && identifier.getDatabaseName().equals(i.getDatabaseName())) {
            tableCache.invalidate(i);   // 连带分支表缓存一并失效
        }
    }
}
```

**为何 Manifest 单独用 `SegmentsCache`？** Manifest 是大小悬殊的二进制大文件（几 KB 到几十 MB），普通 KV 缓存按"条目数"限制无法防止内存爆掉；`SegmentsCache` 按字节数限制，并用 `cache.manifest-small-file-threshold`（默认 1MB）决定小文件入缓存、大文件直读。**多进程共享 warehouse 的一致性陷阱**：`CachingCatalog` 是进程内缓存，进程 A 改了 schema，进程 B 的缓存不会自动失效，靠 `cache.expire-after-write`（默认 30min）兜底——共享场景务必把过期时间调短或显式 `invalidateTable`。

`SegmentsCache` 缓存的是 manifest 文件**已读入内存的 MemorySegment 序列**（而非反序列化后的对象），命中时直接从内存段反序列化 entry，省去文件 IO。它是 catalog 级而非表级——多张表的 manifest 共用一个按 `cache.manifest-max-memory` 限额的池子，避免某张大表把内存吃光。`dvMetaCache`（Deletion Vector 元数据，限额 `cache.deletion-vectors.max-num` 默认 10 万）同理用于 merge-on-read 表加速 DV 查找。这三个数据级缓存（manifest / DV）与表对象缓存分离，是因为它们的访问模式（大对象、按字节、catalog 级共享）与"逻辑表对象"（小对象、按条目、表级）截然不同。

`CachingCatalog` 在 `CatalogFactory.createCatalog` 中按需创建（`cache.enabled` 默认 true）：

```java
static Catalog createCatalog(CatalogContext context, ClassLoader classLoader) {
    Catalog catalog = createUnwrappedCatalog(context, classLoader);
    catalog = CachingCatalog.tryToCreate(catalog, options);
    return PrivilegedCatalog.tryToCreate(catalog, options);
}
```

**缓存的过期与失效策略对比**（理解何时该用哪个）：

| 机制 | 触发 | 适用 |
|---|---|---|
| `expire-after-access`（10min） | 末次访问后计时 | 热表常驻、冷表自然淘汰 |
| `expire-after-write`（30min） | 写入后计时（硬上限） | 防止多进程下缓存无限陈旧 |
| 主动 `invalidateTable` | DDL 写操作后 | 本进程改了元数据，立即失效 |
| `partitionCache` 默认关 | `partition-max-num=0` | 分区数巨大时避免内存爆 |

**多进程一致性的真实风险与对策**：`expire-after-write` 是"多进程共享 warehouse"下唯一的兜底——进程 A 改 schema、进程 B 缓存的旧表对象最多陈旧 30min。若业务无法容忍这个窗口（如 A 加列后 B 立即要读到），要么调短 write 过期，要么 B 在关键读前显式 `invalidateTable`，要么直接用 REST Catalog（服务端是唯一真相源，缓存可由服务端 ETag / 版本号驱动失效）。

**陷阱：频繁 `invalidateTable` 反成性能杀手**——每次失效都让下次访问回落到文件系统全量加载（读 schema 目录 + 最新 snapshot）。在循环里对每张表 invalidate 再 getTable，相当于关掉了缓存。

**`tableCache` 缓存的是 `Table` 对象（含其内部的 schema / path / FileIO 等），不是数据**——所以缓存命中省的是"加载表定义"的开销（读 schema、建 FileStoreTable），不是省数据读。一张表的 `Table` 对象可被多次 newRead / newWrite 复用；invalidate 后这些引用仍可用（已持有的对象不受影响），只是下次 getTable 会重建。这意味着缓存失效不会打断正在进行的读写——它只影响"下一次从 catalog 取表"。理解这点能避免"以为 invalidate 会中断在跑的查询"的误解。

### 2.6 CatalogFactory SPI 与四种 metastore

> 源码：`catalog/CatalogFactory.java`

```java
public interface CatalogFactory extends Factory {
    default Catalog create(FileIO fileIO, Path warehouse, CatalogContext ctx);  // 需要 warehouse
    default Catalog create(CatalogContext ctx);                                  // 自包含(REST)
}
```

装配流程（`createUnwrappedCatalog`）：

```
读 "metastore" 配置（默认 "filesystem"）
  → FactoryUtil.discoverFactory(classLoader, CatalogFactory.class, metastore)   // Java SPI
  → 先试 catalogFactory.create(context)
  → 抛 UnsupportedOperationException 则：
       读 warehouse 路径（必须配置，否则报错）
       创建 FileIO + 检查/创建 warehouse 目录
       catalogFactory.create(fileIO, warehousePath, context)
```

**官方支持四种 metastore**（旧文仅提三种，已补 `jdbc`）：

| identifier | Factory | 元数据存储 | warehouse 含义 |
|---|---|---|---|
| `filesystem`（默认） | FileSystemCatalogFactory | 文件系统目录 | 仓库路径 |
| `hive` | HiveCatalogFactory | Hive Metastore（+ schema 同步） | 默认用 `hive.metastore.warehouse.dir` |
| `jdbc` | JdbcCatalogFactory | 关系库（MySQL / Postgres / SQLite） | 仓库路径（数据仍在 FS） |
| `rest` | RESTCatalogFactory | 远端 REST Server | DLF 下为**实例名**，非路径 |

JDBC catalog 把库表元数据存关系库（`uri`=`jdbc:mysql://...`、`jdbc.user/password`、`catalog-key`），但数据文件仍在 warehouse 指向的 FS 上——它解决的是"要一个轻量的、带行锁的元数据库，但不想上 Hive Metastore"的场景。两个 `create` 方法互为 fallback，是为兼容"需外部 FileIO 的老实现"与"完全自包含的 REST 新实现"——通过 `UnsupportedOperationException` 做分派，避免引入显式的"能力标志"参数。

**程序化创建与使用 Catalog**（来自 `program-api/catalog-api`，三种后端）：

```java
// filesystem
Map<String,String> opts = new HashMap<>();
opts.put("warehouse", "hdfs:///path/to/warehouse");
Catalog fs = CatalogFactory.createCatalog(CatalogContext.create(Options.fromMap(opts)));

// hive
opts.put("metastore", "hive"); opts.put("uri", "thrift://hms:9083");
Catalog hive = CatalogFactory.createCatalog(CatalogContext.create(Options.fromMap(opts)));

// rest (DLF)
opts.put("metastore", "rest"); opts.put("uri", "https://...dlf.aliyuncs.com");
opts.put("warehouse", "my_instance_name");      // 注意: 实例名, 非路径
opts.put("token.provider", "dlf");
opts.put("dlf.access-key-id", "..."); opts.put("dlf.access-key-secret", "...");
Catalog rest = CatalogFactory.createCatalog(CatalogContext.create(Options.fromMap(opts)));

// 用法: 增删改查均抛 checked 异常, 必须处理
catalog.createDatabase("my_db", false);                       // ignoreIfExists=false
catalog.dropDatabase("my_db", false, true);                   // ignoreIfNotExists, cascade
boolean exists = catalog.databaseExists("my_db");
Table table = catalog.getTable(Identifier.create("my_db", "t")); // 普通表
Table sys   = catalog.getTable(Identifier.create("my_db", "t$snapshots")); // 系统表
catalog.renameTable(Identifier.create("my_db","t"), Identifier.create("my_db","t2"), false);
```

`CatalogContext` 承载 options + Hadoop conf + FileIO 偏好；`CatalogFactory.createCatalog` 内部就是 2.6 的装配流程。建议把 Catalog 实例缓存复用（尤其 REST，避免每次 config 往返）。

### 2.7 Identifier 标识解析

> 源码：`paimon-api/.../catalog/Identifier.java`

格式 `{database}.{table}[$branch_{branch}][$systemTable]`。示例：

```
my_db.my_table                        普通表
my_db.my_table$snapshots              系统表
my_db.my_table$branch_dev             分支表
my_db.my_table$branch_dev$snapshots   分支的系统表
```

字段：`database`（库名）、`object`（完整对象名）、`table` / `branch` / `systemTable`（均 `transient` 派生）。`object` 懒解析（`splitObjectName()`，分隔符 `$`）：1 段=普通表；2 段=`table$branch_x` 或 `table$系统表`；3 段=`table$branch_x$系统表`。

**为何用 `$` 而非 `.`**：`.` 已用于 `db.table`；`$` 在 SQL 标识符里合法且少用，`SELECT * FROM t$snapshots` 可直接写。**为何 `table` / `branch` / `systemTable` 设 `transient`**：序列化只传 `database`+`object`（最小信息），跨网络 / 跨进程时由接收端 `splitObjectName` 重新解析——避免序列化冗余且保证解析逻辑单一来源。

**`fullName()` 与系统表判定**：`Identifier.getFullName()` 返回 `database.object`（含 `$` 后缀），用于日志 / 缓存键；`isSystemTable()` 判断 `systemTable != null`，`getSystemTableName()` 取系统表名。分支判定 `getBranchName()` 解析 `$branch_` 前缀。**边界**：分支名嵌在 object 里用 `branch_` 前缀（而非裸分支名），所以 `t$branch_dev` 的分支名是 `dev`，`BRANCH_PREFIX="branch_"` 这个前缀是解析约定——若分支名本身含 `$` 会破坏解析，故 6.2 的 `validateBranch` 禁用特殊字符。

**为何用 transient + 懒解析 + 缓存于派生字段**：`Identifier` 在 Flink / Spark 里频繁创建、传递、当 Map key。若构造时就解析 object（split 字符串），每个 Identifier 都要做一次字符串操作；而很多 Identifier 创建后只比较 equals（按 database+object）不取 table/branch——懒解析让"只比较不取分量"的场景零解析开销。一旦首次取 `getTableName()` 才解析并缓存到 transient 字段，后续复用。这是"按需计算 + 缓存"在高频小对象上的典型优化。

---

## 三、REST Catalog（云原生重点）

### ① 要解决什么问题

filesystem / hive / jdbc 都把"并发仲裁"压在客户端（原子 rename + 外部锁），在大规模多租户云环境下，这意味着每个客户端都要直连存储并持有长期强凭证——既不安全（凭证泄露面大）也难治理（权限分散在各引擎配置里）。REST Catalog 把元数据控制面收敛到服务端：客户端只发 HTTP，服务端统一做鉴权、并发提交仲裁、对象路径生成（UUID）、按表下发临时数据凭证。好处是客户端轻量、跨语言（pypaimon 也能接）、权限集中、凭证最小化。

### ② 设计原理与取舍（最重要）

- **两段式初始化**：`RESTApi` 构造时先用本地 options 发一次 `GET config?warehouse=...`，把服务端下发的配置 merge 进来（含 token 相关参数与缓存参数），再构建真正的 `restAuthFunction`。这让"服务端控制客户端行为"成为可能——比如服务端可以强制开启 `data-token.enabled`。
- **认证与签名分离**：`AuthProvider` 只负责"往请求头塞鉴权信息"，具体是静态 Bearer 还是 DLF v4 签名由实现决定；token 的生命周期管理（刷新）也封在 provider 内。新增认证方式只需加一个 `AuthProvider` + SPI Factory，不动 `RESTApi`。
- **元数据凭证 vs 数据凭证分离**：访问 catalog 用 catalog token（长期一点）；读写表数据文件用服务端按表 / 按操作下发的 **data token**（短期、最小权限），通过 `RESTTokenFileIO` 包裹真实 FileIO。这是"权限最小化"的核心——即使 data token 泄露，也只影响一张表一小段时间。
- **路径虚拟化**：服务端生成 UUID 物理路径，对用户暴露 `pvfs://catalog/db/table/...` 虚拟路径（PVFS），所有访问过 REST 权限系统，无需另维护一套 FS 权限。
- **取舍**：引入网络依赖与服务端可用性约束、初始化往返开销、调试链路变长；换来集中鉴权审计、跨语言、UUID 路径隔离、最小权限数据访问。在大型组织里这笔账通常划算。

**REST Catalog 的四个官方设计目标**（overview 文档）落到源码：(1) 技术特定逻辑全在服务端——客户端只发标准 REST，业务定制（如自定义权限、自定义存储后端）封在 server；(2) 解耦架构——客户端与服务端通过良定义的 REST API 交互，可独立演进 / 扩容；(3) 语言无关——任何语言实现的 server 只要遵守 API 即可（pypaimon 客户端就是例证）；(4) 支持任意 catalog 后端——server 背后可以是 DLF、自建元数据库、甚至代理到 Hive，客户端无感。这就是为什么 `RESTApi` 只认 REST 协议、不关心服务端实现——它把"控制面实现自由"作为头等目标，代价是把可用性 / 一致性责任也压到了服务端。

### ③ 关键源码

`RESTCatalog implements Catalog`（非 AbstractCatalog），核心字段与构造（`rest/RESTCatalog.java:99-120`）：

```java
public class RESTCatalog implements Catalog {
    private final RESTApi api;                 // 所有 HTTP 调用的入口
    private final CatalogContext context;
    private final boolean dataTokenEnabled;    // 是否启用按表 data token
    protected final Map<String, String> tableDefaultOptions;
    @Nullable private final LocalCacheManager cacheManager;  // 本地数据缓存(可选)

    public RESTCatalog(CatalogContext context) { this(context, true); }
    public RESTCatalog(CatalogContext context, boolean configRequired) {
        this.api = new RESTApi(context.options(), configRequired);
        this.context = CatalogContext.create(api.options(),       // 用 merge 后的 options
                context.hadoopConf(), context.preferIO(), context.fallbackIO());
        this.dataTokenEnabled = api.options().get(RESTTokenFileIO.DATA_TOKEN_ENABLED);
        this.tableDefaultOptions = CatalogUtils.tableDefaultOptions(this.context.options().toMap());
    }
}
```

### ④ 风险/陷阱/边界

- `configRequired=true` 时每次 `new RESTCatalog` 都有一次同步 HTTP 往返（拉 config）。频繁创建 Catalog 实例会放大延迟，应复用（尤其 Flink 每个 subtask 别各建一个）。
- DLF 模式下 `warehouse` 是**实例名**不是路径（doc 明确强调），写成路径会鉴权失败。
- token 刷新基于本地时钟，客户端时钟漂移过大会导致提前 / 滞后刷新甚至签名失败——容器环境注意 NTP。
- `RESTTokenFileIO` 序列化后 `apiInstance` 变 null（`transient`），反序列化端首次用时会按 `catalogContext` 重建 `RESTApi`——若 context 里缺凭证会在运行时才报错。

### ⑤ 收益与代价

收益：集中鉴权与审计、跨语言、UUID 路径天然隔离、data token 最小权限、客户端轻量。代价：强依赖服务端可用性与网络、初始化往返开销、调试需看服务端日志、时钟敏感。

### 3.1 RESTCatalog 与 RESTApi 分层

> 源码：`paimon-core/.../rest/RESTCatalog.java`（1274 行）、`paimon-api/.../rest/RESTApi.java`（1639 行）

`RESTCatalog` 是"协议适配层"——把 Catalog 接口语义翻译成 REST 资源操作（database / table / partition / tag / branch / snapshot / view / function 各有资源路径）；`RESTApi` 是"HTTP 客户端层"——封装 `HttpClient`、`RESTAuthFunction`、`ResourcePaths`、Jackson 序列化（`OBJECT_MAPPER`）。两段式初始化（`RESTApi.java:190-213`）：

```java
public RESTApi(Options options, boolean configRequired) {
    this.client = new HttpClient(options.get(RESTCatalogOptions.URI));
    AuthProvider authProvider = createAuthProvider(options);
    Map<String,String> baseHeaders = extractPrefixMap(options, HEADER_PREFIX);
    if (configRequired) {                                  // 第一次握手：拉服务端配置
        String warehouse = options.get(WAREHOUSE);
        Map<String,String> q = StringUtils.isNotEmpty(warehouse)
                ? ImmutableMap.of(WAREHOUSE.key(), RESTUtil.encodeString(warehouse))
                : ImmutableMap.of();
        options = new Options(client.get(ResourcePaths.config(), q, ConfigResponse.class,
                new RESTAuthFunction(baseHeaders, authProvider)).merge(options.toMap()));
        baseHeaders.putAll(extractPrefixMap(options, HEADER_PREFIX));
    }
    this.restAuthFunction = new RESTAuthFunction(baseHeaders, authProvider);  // 后续请求复用
    this.options = options;
    this.resourcePaths = ResourcePaths.forCatalogProperties(options);
}
```

`RESTAuthFunction` 是一个 `Function<RESTAuthParameter, Map<String,String>>`——每次请求把基础头 + AuthProvider 生成的鉴权头合并后下发给 `HttpClient`。把"配置 merge"前置，意味着服务端可以下发 `data-token.enabled`、缓存参数等控制客户端行为；`options()` 返回的就是 merge 后的最终配置，`RESTCatalog` 据此构建自己的 `context`。

**资源路径与 HTTP 方法映射**（`ResourcePaths` + `RESTApi` 的各方法）：Catalog 接口操作翻译成 REST 资源——`GET /v1/{prefix}/databases`（列库）、`POST .../databases`（建库）、`GET .../databases/{db}/tables/{t}`（取表）、`POST .../tables`（建表）、`POST .../tables/{t}/commit`（提交快照）、`GET .../tables/{t}/snapshot`（loadSnapshot）、tag / branch / partition / view / function 各有路径。`ResourcePaths.forCatalogProperties(options)` 用服务端下发的 prefix 拼前缀（多租户隔离）。**分页**靠 `pageToken` 透传：`listTablesPaged(maxResults, pageToken, ...)` → REST query 带 token → 返回 `PagedList`（数据 + nextPageToken），客户端循环直到 token 为空——这正是 `supportsListObjectsPaged()` 在 REST 返回 true、filesystem 返回 false 的能力差异落点。

**HttpClient 与重试**：`HttpClient` 封装底层 HTTP（连接池、超时），对 5xx / 网络抖动可能做有限重试；对 4xx（鉴权 / 参数错）直接抛对应 `RESTException` 子类不重试。这与 `FileStoreCommitImpl` 的乐观锁重试是两层——HTTP 层重传瞬时网络错，commit 层重试快照冲突，职责不同别混淆。

**三层重试别混淆**（排障时定位是哪层在重试）：(1) HTTP 层（HttpClient）重传瞬时网络 / 5xx；(2) commit 层（tryCommit）重试快照冲突（snapshot conflict）；(3) 作业层（Flink failover）应对 files conflict 重启。三层超时 / 次数独立配置，现象不同：HTTP 重试表现为单次请求变慢、commit 重试表现为 `attempts` 指标升高、作业重试表现为整个 task restart。看到"慢"先分清是哪层——HTTP 慢查网络 / 服务端，commit 慢查并发冲突，作业反复 restart 查 files conflict。混淆这三层会把"网络抖动"误判成"并发冲突"开错药方。

### 3.2 认证体系：AuthProvider / Bear / DLF

> 源码：`paimon-api/.../rest/auth/`

`AuthProvider` 接口极简（`auth/AuthProvider.java`）：

```java
public interface AuthProvider {
    Map<String,String> mergeAuthHeader(Map<String,String> baseHeader, RESTAuthParameter param);
}
```

`token.provider` 决定实现（经 `AuthProviderFactory` SPI、`AuthProviderEnum` 枚举）：

| provider | 实现类 | 鉴权方式 |
|---|---|---|
| `bear` | `BearTokenAuthProvider` | 静态 / 文件 token → `Authorization: Bearer <token>` |
| `dlf` | `DLFAuthProvider` | 阿里云 DLF v4 签名（access-key / STS / ECS role） |

Bear 最简单：把 token 塞进 `Authorization` 头，服务端校验合法性即认证通过；token 可来自配置 `token` 或周期刷新的文件。DLF 复杂在于它要**对每个请求做签名**（含 body 的 SHA256、时间戳、host），且 token 本身可能是动态的（STS / ECS role）——这就需要"签名器 + token 加载器 + 刷新逻辑"三件套。

`RESTAuthParameter` 携带本次请求的方法、路径、query、body data 等签名所需信息——这是为什么签名不能在 `RESTApi` 构造时一次性算好，而必须每请求一算。

`RESTAuthFunction`（持 baseHeaders + AuthProvider）是把"每请求签名"封装成函数对象：`HttpClient` 发请求时调它拿到完整鉴权头。这样 `HttpClient` 不关心鉴权细节、`AuthProvider` 不关心 HTTP 细节，二者经 `RESTAuthFunction` 解耦。换 token / 换签名算法只动 AuthProvider，HttpClient 与 RESTApi 的请求逻辑零改动——这是认证可插拔的工程落点。

**Bear vs DLF 的安全模型差异**：Bear token 是"持有即认证"（bearer 语义）——一旦泄露，在过期前任何人都能冒用，且请求不绑定具体内容（无 body 签名）。DLF v4 签名则把请求方法 / 路径 / body 摘要 / 时间戳一起签名，重放窗口受 `x-dlf-date` 时间戳约束、内容被篡改即签名失效——安全性高得多。所以生产云环境推荐 DLF（尤其配合 STS / ECS role 的短期凭证），Bear 更适合内网可信环境或自建 server 的简单鉴权。`AuthProviderEnum` 枚举可扩展更多 provider，新增只需实现 `AuthProvider` + `AuthProviderFactory` 注册 SPI，不动 `RESTApi`。

### 3.3 DLF 签名与 token 刷新

> 源码：`auth/DLFAuthProvider.java`、`auth/DLFDefaultSigner.java`、`auth/DLFOpenApiSigner.java`、`auth/DLFTokenLoader.java` 及其实现

`mergeAuthHeader` 每次请求都重新签名（`DLFAuthProvider.java:92-111`）：

```java
public Map<String,String> mergeAuthHeader(Map<String,String> baseHeader, RESTAuthParameter p) {
    DLFToken token = getFreshToken();                       // 可能触发刷新
    Instant now = Instant.now();
    String host = extractHost(uri);
    Map<String,String> signHeaders = signer.signHeaders(p.data(), now, token.getSecurityToken(), host);
    String authorization = signer.authorization(p, token, host, signHeaders);
    Map<String,String> out = new HashMap<>(baseHeader);
    out.putAll(signHeaders);
    out.put(DLF_AUTHORIZATION_HEADER_KEY, authorization);   // 含 x-dlf-date / x-dlf-security-token / x-dlf-content-sha256 ...
    return out;
}
```

签名头集合（`DLFAuthProvider` 常量）：`Authorization`、`Content-MD5`、`Content-Type`、`x-dlf-date`（格式 `yyyyMMdd'T'HHmmss'Z'`）、`x-dlf-security-token`、`x-dlf-version`、`x-dlf-content-sha256`（值 `UNSIGNED-PAYLOAD`）。

**双签名算法自动选择**（`createSigner`）：按 endpoint 类型选 `DLFDefaultSigner`（VPC endpoint，如 `cn-hangzhou-vpc.dlf.aliyuncs.com`）或 `DLFOpenApiSigner`（公网 OpenAPI endpoint，如 `dlfnext.cn-hangzhou.aliyuncs.com`）。OpenAPI 端目前对库 / 表名字符集有限制（仅字母数字与特定符号），VPC 端无此限制且延迟更低。

**token 刷新：双重检查锁 + 1 小时安全提前量**（`DLFAuthProvider.java:155-190`）：

```java
DLFToken getFreshToken() {
    if (shouldRefresh()) {
        synchronized (this) {
            if (shouldRefresh()) { refreshToken(); }   // this.token = tokenLoader.loadToken();
        }
    }
    return token;
}
private boolean shouldRefresh() {
    if (token == null) return true;
    Long expireTime = token.getExpirationAtMills();
    if (expireTime == null) return false;              // 永不过期(静态 AK)
    return expireTime - System.currentTimeMillis() < TOKEN_EXPIRATION_SAFE_TIME_MILLIS;
}
```

`TOKEN_EXPIRATION_SAFE_TIME_MILLIS = 3_600_000L`（1 小时，`RESTApi.java:160`）——在过期前 1 小时就刷新，给签名、网络往返、时钟漂移留足缓冲。`token` 字段 `volatile`，配合双重检查锁保证可见性与"只有一个线程真正刷新"。

token 来源（`DLFTokenLoader` 三种实现，经 `DLFTokenLoaderFactory` SPI 装配）：

| Loader | 触发配置 | 行为 |
|---|---|---|
| 直接 access-key | `dlf.access-key-id/secret`(+`dlf.security-token`) | 静态 token，`getExpirationAtMills()` 为 null，永不刷新 |
| `DLFLocalFileTokenLoader` | `dlf.token-path` | 周期读本地文件（外部进程刷写），适合 STS 由 sidecar 维护 |
| `DLFECSTokenLoader` | `dlf.token-loader=ecs`(+`dlf.token-ecs-role-name`) | 从 ECS 元数据服务取实例 RAM role 的 STS token，无需配 AK |

三种方式的安全梯度：**直接 AK**（RAM 用户长期 access key）最方便但凭证长期有效、泄露风险最高，仅建议受控环境；**STS 临时凭证**（带 `dlf.security-token` 或 `token-path` 文件）有有效期，泄露影响有限，是推荐做法；**ECS role**（`token-loader=ecs`）最安全——AK 完全不落地，由 ECS 实例的 RAM role 在元数据服务里动态发 STS token，机器即身份。生产云上首选 ECS role，其次 STS 文件，尽量避免长期 AK。三者对 Paimon 是透明的——`DLFAuthProvider` 只调 `tokenLoader.loadToken()`，不关心 token 怎么来的，这正是"加载器抽象"的价值。

**边界 / 排障**：`refreshToken` 持表级 `synchronized`，高并发请求只一个线程真刷新、其余阻塞等待；若 `tokenLoader.loadToken()` 慢（ECS metadata 网络抖动），会短暂阻塞所有发请求线程——表现为"周期性请求毛刺"。日志 `begin/end refresh meta token for loader [...] expiresAtMillis [...]` 是定位刷新行为的钩子。

**region 与 endpoint 推断**：`DLFAuthProvider` 持有 `region` 与 `signingAlgorithm`，`extractHost(uri)` 从 uri 剥协议与路径取 host（`https://host:port/prefix` → `host:port`）参与签名。VPC endpoint 与 OpenAPI endpoint 的 region 推断、签名算法选择都在构造期确定——配错 endpoint（如 VPC 环境填了公网 OpenAPI）会导致延迟高或签名不匹配。**时钟敏感的具体表现**：DLF v4 签名含 `x-dlf-date` 时间戳，服务端校验请求时间在允许窗口内（防重放）；若客户端时钟比服务端快/慢超过窗口（通常几分钟），所有请求被拒——容器无 NTP 是常见坑，表现为"突然全部 403 / 签名错误"。

### 3.4 Data Token、RESTTokenFileIO 与 PVFS

> 源码：`paimon-common/.../rest/RESTTokenFileIO.java`

REST Catalog 生成 UUID 路径并下发**按表的短期数据凭证**，让客户端直连对象存储读写数据文件，而无需持有长期强凭证。`RESTTokenFileIO` 是 `FileIO` 装饰器，每个方法先 `tryToRefreshToken()` 再委托给真实 FileIO：

```java
public static final ConfigOption<Boolean> DATA_TOKEN_ENABLED =
    ConfigOptions.key("data-token.enabled").booleanType().defaultValue(false);

public FileIO fileIO() throws IOException {
    tryToRefreshToken();
    FileIO fileIO = FILE_IO_CACHE.getIfPresent(token);     // 以 RESTToken 为 key 缓存 FileIO
    if (fileIO != null) return fileIO;
    synchronized (FILE_IO_CACHE) {
        fileIO = FILE_IO_CACHE.getIfPresent(token);
        if (fileIO != null) return fileIO;
        Options options = new Options(RESTUtil.merge(
            catalogContext.options().toMap(), token.token()));   // token 携带 AK/SK/STS 注入 options
        options.set(FILE_IO_ALLOW_CACHE, false);                 // 禁用内层 FileIO 自身缓存
        fileIO = FileIO.get(path, CatalogContext.create(options, ...));
        FILE_IO_CACHE.put(token, fileIO);
        return fileIO;
    }
}
```

`FILE_IO_CACHE` 是全局静态 Caffeine（`maximumSize(1000)`、`expireAfterAccess(10, HOURS)`、移除时 `IOUtils.closeQuietly`、专用守护线程调度器）。关键设计点：

- **以 `RESTToken` 为缓存 key**：token 刷新后是新对象，自然命中不到旧 FileIO，从而用新凭证重建——天然实现"凭证轮换"，旧 FileIO 在空闲过期时被关闭。这比"显式重配 FileIO"优雅得多。
- **刷新阈值同样是 1h 安全提前量**（`tryToRefreshToken` 复用 `TOKEN_EXPIRATION_SAFE_TIME_MILLIS`），双重检查锁。
- **`token` 字段 `volatile` 且可序列化**：Flink 算子序列化时把当前 token 带过去，反序列化端先用它、过期再刷新，避免一启动就齐刷刷打 REST Server（惊群）。`apiInstance` 是 `transient`，反序列化后按 `catalogContext` 懒重建。
- **`configure()` 直接抛 `UnsupportedOperationException`**：它是被动包装器，配置只能来自构造时传入的 context，不允许二次 configure。

**PVFS（Paimon Virtual Storage）**：因 UUID 路径难以人工访问，PVFS 暴露 `pvfs://catalog/db/table/...` 虚拟路径。`listStatus(pvfs://my_catalog/)` 返回所有库的虚拟路径，`listStatus(pvfs://my_catalog/my_db)` 返回所有表；`newInputStream(pvfs://.../my_table/a.csv)` 时先向 REST Server 解析真实路径，再用真实 FileSystem 读写。所有访问都过 REST Catalog 的权限系统，无需另维护一套 FS 权限。实现为 Hadoop `FileSystem`（`org.apache.paimon.vfs.hadoop.PaimonVirtualFileSystem` / `Pvfs`，配 `fs.pvfs.uri` / `fs.pvfs.token.provider` / `fs.pvfs.token`）+ Python fsspec 风格 SDK（`pypaimon.PaimonVirtualFileSystem`），因此 Spark / Hadoop shell / PyArrow / Ray 都能透明接入。

PVFS 覆盖三类内置存储：Paimon Table、Format Table、Object Table（亦称 Fileset / Volume）——后两者尤其需要直接 FS 访问。**本质上 PVFS 与 `RESTTokenFileIO` 是同一思路在"用户可见路径"层的延伸**：路径虚拟化 + 凭证由服务端按权限下发，把"存储寻址"与"存储鉴权"都收敛到 REST Server。

**本地数据缓存 `LocalCacheManager` / `CachingFileIO`**：REST 模式下数据文件常在远端对象存储，重复读（如多次扫描同一 manifest / 小文件）代价高。`RESTCatalog` 可选持有 `LocalCacheManager`，配合 `CachingFileIO`（`paimon-common/fs/cache`）把热数据缓存到本地磁盘 / 内存，`IO_CACHE_ENABLED` 配置控制开关。这与 `RESTTokenFileIO` 正交：前者管"读得快"，后者管"读得到（凭证）"。三者叠加时实际 FileIO 形如 `CachingFileIO( RESTTokenFileIO( ResolvingFileIO( 真实对象存储 FileIO ) ) )`——`ResolvingFileIO` 负责按 scheme 路由到具体实现。

### 3.5 REST 表加载与并发控制

filesystem 模式靠"本地原子写 `snapshot-{id+1}`"做乐观锁；REST 模式把这步交给服务端：`Catalog.commitSnapshot(identifier, tableUuid, snapshot, statistics)` 是接口方法，REST 实现把它翻译成一次 HTTP 提交，由服务端判定快照 ID 是否被抢占（snapshot conflict）、要删的文件是否已被删（files conflict）。`tableUuid` 用于服务端识别表身份——避免 drop+create 同名表后把老提交误认到新表。

`FileStoreCommitImpl` 通过 `SnapshotCommit` 抽象屏蔽两种提交方式：filesystem 实现走 `fileIO.tryToWriteAtomic`（本地原子），REST 实现走 `catalog.commitSnapshot`（HTTP 让服务端仲裁）。`tryCommitOnce` 步骤 7 调的 `commitSnapshotImpl` 不关心底层是哪种——它只看返回 true（成功）/ false（冲突重试）。这层抽象让乐观锁的"读 latest → 算 → 提交 → 成败"主循环对 filesystem / REST 完全统一，差异只在最后一步"提交动作"的实现。这正是把 `commitSnapshot` 提到 `Catalog` 接口的回报：上层 commit 逻辑写一套，跑在四种后端上。

**`SnapshotCommit` 与去重的协作**：filesystem 的 `tryToWriteAtomic(snapshot-{id+1})` 若返回 false（被抢），上层重试时先做去重检查（8.3 步骤 1）——若发现被抢那个快照其实是自己上次成功写的（三元组匹配），就认成功不再写。REST 的 `commitSnapshot` 则可能由服务端直接做去重（服务端知道 tableUuid + commitUser + identifier），返回"已提交"。两种路径殊途同归：精确一次的去重既可在客户端（扫快照链）也可在服务端（REST），由 `SnapshotCommit` 实现选择，对上层透明。

**REST 表加载（getTable）流程**：`RESTCatalog.getTable` → `RESTApi` 发 `GET /databases/{db}/tables/{table}` → 返回 `GetTableResponse`（含 schema、path、options、uuid）→ 构造 `TableSchema` → 经 `CatalogUtils` / `FileStoreTableFactory` 组装 `FileStoreTable`。关键区别于 filesystem：表的物理 `path` 由服务端返回（UUID 路径），客户端不自己拼；表的 `FileIO` 若 `dataTokenEnabled` 则包成 `RESTTokenFileIO`（每次 IO 带最新 data token）。`loadSnapshot` 同理走 HTTP，由服务端返回 `TableSnapshot`，这也是 `SnapshotManager.snapshotLoader` 非空时优先走 loader 的原因（见 5.3）。

**为何 REST 表的 path 由服务端给**：filesystem 表 path 可由 warehouse + db + table 推出（`{warehouse}/{db}.db/{table}`），但 REST 用 UUID 路径（隔离 + 防猜测 + 支持表改名不动数据），客户端无法推算，只能由服务端在 `GetTableResponse` 里返回。这也意味着 REST 表的"重命名"是元数据操作（改服务端的 name→uuid 映射），数据文件原地不动——比 filesystem 的目录 rename 更轻、更安全（无对象存储 rename 非原子问题）。这是 REST 架构相对 filesystem 的一个隐性优势。

官方 concurrency-control 文档定义的两类失败在 REST 下语义不变，只是仲裁点从文件系统挪到服务端：

1. **Snapshot conflict**：快照 ID 被别的作业抢了 → 客户端重新取 latest 再提（`tryCommit` 循环消化）。
2. **Files conflict**：要逻辑删除的文件已被删 → 该提交节点无法继续，流式作业会故意 failover 重启，从最新状态重试。

**排障要点**：REST 提交失败的根因常在服务端，客户端只看到 `RESTException` 子类（`BadRequestException` / `ForbiddenException` / `ServiceFailureException` / `AlreadyExistsException` / `NoSuchResourceException` / `NotImplementedException`，见 `rest/exceptions/`）。`ForbiddenException` 多为 token / 权限问题（查 3.3 刷新日志与服务端鉴权日志）；反复 `AlreadyExistsException`(snapshot) 说明高并发写撞车，参考"关写侧 compaction + 独立 compaction 作业"。`NotImplementedException` 说明该服务端未实现某可选 API（如分页 / 版本管理），此时 `supportsXxx()` 应返回 false 让上层降级——若仍调用即是客户端 bug。

---

## 四、Schema 演进机制

### ① 要解决什么问题

业务演进要加列 / 改类型 / 删列，但已写入的 TB 级数据文件不可能重写。必须做到：新旧 schema 共存、读旧文件时按列 ID 映射对齐、并发改 schema 不互相覆盖、每个数据文件能找回它写入时的 schema。

### ② 设计原理与取舍（最重要）

- **独立 schema 文件而非内嵌**：每次变更只写一个新 `schema-{id}`，O(1) 轻量，多个 snapshot 共享同一 schema；代价是读取时多一次 IO 取 schema。对比 Iceberg 把 schema 嵌进 metadata.json 每次重写整块——Paimon 的方式让"高频流式写 + 偶发 schema 改"互不拖累。
- **字段 ID 永不回收**：`highestFieldId` 单调增，删列后新列从 `highestFieldId+1` 起，杜绝"删 A 再加同名 A"导致的旧数据错读。列映射靠 field id 而非列名 / 位置——这是演进正确性的锚点。
- **乐观锁**：读最新 → 纯函数算新 schema → 原子写 `schema-{oldId+1}`，撞车则重试。无分布式锁依赖。把"算新 schema"做成无副作用纯函数，重试只需重算，简单且无脏状态。
- **新增列强制 nullable**：旧文件没这列，读时只能补 null，故 `addColumn(nullable=false)` 直接拒绝——这是向后兼容的硬约束。
- **类型变更只允许安全扩宽**：INT→BIGINT 可，BIGINT→INT 默认不可（除非 `allowExplicitCast`），保证不丢精度；不安全转换必须 overwrite 重写数据。

### ③ 关键源码

乐观锁原子写（`SchemaManager.java`，`commit`）：

```java
public boolean commit(TableSchema newSchema) throws Exception {
    SchemaValidation.validateTableSchema(newSchema);
    SchemaValidation.validateFallbackBranch(this, newSchema);
    Path schemaPath = toSchemaPath(newSchema.id());
    return fileIO.tryToWriteAtomic(schemaPath, newSchema.toString());  // 目标已存在则返回 false
}
```

`tryToWriteAtomic` 在 HDFS / 本地 FS 上用"写临时文件 + 原子 rename"；目标已存在 rename 失败返回 false，触发上层重试。对象存储无原子 rename 时退化为 `putIfAbsent` 语义或配合外部锁。

### ④ 风险/陷阱/边界

- 不能删主键 / 分区键列（会破坏分区与桶结构），`generateTableSchema` 内校验后抛异常。
- schema 文件永不清理——旧数据文件可能仍引用旧 schema id。
- 高频单条变更会刷出大量 schema 文件；应**批量** `commitChanges(List<SchemaChange>)` 一次只产一个新文件。
- Schema 变更不需要停写：乐观锁允许写入过程中改 schema，但写入可能因此重试。
- 重命名列会同步改 options 里对该列的引用（如 `bucket-key`、`sequence.field`）——若手动改 options 与列名不一致会校验失败。

### ⑤ 收益与代价

收益：演进零重写、并发安全、历史可追溯、读旧文件可精确列映射。代价：读路径多一次 schema IO、schema 文件单调累积、类型变更受限（不安全转换需 overwrite 重写）。

**Schema 与 Snapshot 的解耦在版本管理上的回报**：因为 schema 独立于 snapshot，时间旅行到 snapshot-N 时自动用 N.schemaId 对应的 schema 读取——历史数据按历史 schema 解析，不会被当前 schema "污染"。分支 / Tag 也各自记 schemaId，互不干扰。若 schema 内嵌 snapshot，每次改 schema 都要重写所有引用它的快照，时间旅行与分支的实现会复杂得多。这就是 4.0 节"独立 schema 文件"取舍在版本管理维度的具体收益——一个设计决策的价值往往在多个机制里复利显现。

**程序化 API 的完整约束清单**（来自 `program-api/catalog-api`，对应 `SchemaValidation` / `generateTableSchema` 校验）：

- 新增列不能 `NOT NULL`（向后兼容）；
- 不能改分区列类型（破坏分区布局）；
- 不能改主键列的 nullability（主键恒非空）；
- 嵌套 ROW 类型的列**不支持改类型**，也不支持把普通列改成嵌套 ROW；
- 删列 / 改列经 `fieldNames` 数组支持嵌套路径（如 `["col5","f1"]` 改 ROW 内子字段注释 / nullability）。

异常类型对应：`alterTable` 抛 `ColumnAlreadyExistException` / `ColumnNotExistException` / `TableNotExistException`；建表抛 `TableAlreadyExistException` / `DatabaseNotExistException`；`alterDatabase` 仅 hive / jdbc 支持（filesystem 抛 `UnsupportedOperationException`），改库属性用 `DatabaseChange.setProperty / removeProperty`。这些 checked 异常都定义在 `Catalog` 接口内部——程序化调用必须处理。

### 4.1 TableSchema 字段与派生量

> 源码：`paimon-api/.../schema/TableSchema.java`

| 字段 | 类型 | 说明 |
|---|---|---|
| `version` | int | schema 格式版本（当前 v3） |
| `id` | long | schema ID，单调递增（schema-0 起） |
| `fields` | List\<DataField\> | 字段（不可变，每个含全局 field id） |
| `highestFieldId` | int | 已分配最大 field id（删列不回收） |
| `partitionKeys` / `primaryKeys` | List\<String\> | 分区键 / 主键 |
| `bucketKeys` / `numBucket` | 派生 | 桶键 / 桶数（来自 options） |
| `options` | Map | 表级 CoreOptions |
| `comment` | String? | 表注释 |
| `timeMillis` | long | 创建时间戳 |

`bucketKeys` 派生规则——未显式配 `bucket-key` 时回退到主键（去掉分区键部分）：

```java
List<String> tmp = originalBucketKeys();        // 从 options 读 bucket-key
if (tmp.isEmpty()) tmp = trimmedPrimaryKeys();  // fallback: 主键去分区键
bucketKeys = tmp;
```

`highestFieldId` 的存在使"删除列 A → 新增列 A"得到不同 field id，读旧文件时旧 A 仍按旧 id 解析、新 A 按新 id，不会张冠李戴。序列化为 JSON 写盘。

具体例子说明 field id 为何是稳定锚点：

```
schema-0: [id=0 user_id BIGINT, id=1 name STRING, id=2 age INT]   highestFieldId=2
ALTER RENAME name → username:
schema-1: [id=0 user_id, id=1 username STRING, id=2 age]          # id 不变, 只改 name
ALTER DROP age; ALTER ADD age STRING:
schema-2: [id=0 user_id, id=1 username, id=3 age STRING]          # 新 age 是 id=3, 非 id=2!
                                                                   highestFieldId=3
```

用 schema-0 写的数据文件里 age 是 id=2 的 INT；用 schema-2 读时，当前 schema 的 age 是 id=3 STRING，旧文件没有 id=3 → age 读出来是 null（而非把旧 INT 值错当新 STRING 列）。这正是 field id 永不回收要防的"列身份混淆"。若按列名映射，旧 age 和新 age 同名会被误认作同一列，类型还不同，必然读错。

### 4.2 SchemaManager 与乐观锁 commit

> 源码：`paimon-core/.../schema/SchemaManager.java`（`@ThreadSafe`）

核心方法：`latest()`（扫目录取最大 id）/ `schema(id)` / `listAll()` / `listAllIds()` / `createTable(schema)` / `commitChanges(changes)` / `mergeSchema(rowType, allowExplicitCast)` / `commit(newSchema)`。

`createTable` 的乐观锁循环——并在写盘前用新 schema 实际构造一次 `FileStoreTable` 来验证配置合法（如 bucket / 主键 / 序列字段配置矛盾会在此暴露）：

```java
public TableSchema createTable(Schema schema, boolean externalTable) throws Exception {
    while (true) {
        Optional<TableSchema> latest = latest();
        if (latest.isPresent()) {                    // 表已存在
            if (externalTable) { checkSchemaForExternalTable(latest.get().toSchema(), schema); return latest.get(); }
            else throw new IllegalStateException("Schema in filesystem exists");
        }
        TableSchema newSchema = TableSchema.create(0, schema);          // id=0
        FileStoreTableFactory.create(fileIO, tableRoot, newSchema).store(); // 配置校验
        if (commit(newSchema)) return newSchema;     // 原子写 schema-0
        // 并发建表撞车 → 重试
    }
}
```

`commitChanges` 同样在乐观锁循环内：`latest()` → 纯函数 `generateTableSchema(oldSchema, changes)` 生成新 schema（id=oldId+1）→ `commit` 原子写 `schema-{newId}`，失败则重新取 latest 再算。`generateTableSchema` 遍历每个 `SchemaChange`：SetOption 改 options map、AddColumn 分配新 fieldId 加字段、RenameColumn 改名并更新 options 引用、DropColumn 移除并校验非主键 / 分区键、UpdateColumnType 检查兼容性后改类型……最后构造 id=oldId+1 的 newTableSchema。

**批量提交为何只产一个 schema 文件**：`commitChanges` 接收 `List<SchemaChange>`，在**一次** `generateTableSchema` 里把整批变更全部应用到 oldSchema 上，只产出**一个**新 schema（id=oldId+1）。所以"加 5 列 + 改 2 个 option"批量提交只写 `schema-{N+1}` 一个文件；若拆成 7 次单独 commitChanges 则写 7 个文件（N+1 到 N+7）。这是 4.4 性能陷阱"高频单条变更刷文件"的对策依据——能合就合成一个 list 一次提交。

### 4.3 SchemaChange 类型与生成纯函数

> 源码：`paimon-api/.../schema/SchemaChange.java`（Jackson `@JsonSubTypes` 多态，供 REST 传输）

| 变更 | 类 | 要点 |
|---|---|---|
| SetOption / RemoveOption | 配置增删 | 改 options map |
| UpdateComment | 表注释 | 更新或清除 |
| AddColumn | 加列 | 支持嵌套路径、FIRST / AFTER / BEFORE 定位；**必须 nullable** |
| RenameColumn | 改名 | 同步更新 options 里对该列的引用 |
| DropColumn | 删列 | 校验非主键 / 分区键 |
| UpdateColumnType | 改类型 | 兼容性校验（安全扩宽） |
| UpdateColumnNullability | 改可空 | NULL ↔ NOT NULL |
| UpdateColumnComment / DefaultValue | 改注释 / 默认值 | |
| UpdateColumnPosition | 改顺序 | 经 Move(FIRST / AFTER / BEFORE) |
| DropPrimaryKey | 删主键 | |

Move 类型枚举：`FIRST`（最前）、`AFTER`（指定列后）、`BEFORE`（指定列前）。

**嵌套列与多态序列化**：`AddColumn` / `RenameColumn` / `DropColumn` 都支持 `fieldNames`（字符串数组）表达嵌套路径——如对 ROW 类型列里的子字段操作 `["address", "city"]`。这让结构化类型（ROW / ARRAY\<ROW\> / MAP）也能做 schema 演进。`SchemaChange` 用 Jackson `@JsonTypeInfo` + `@JsonSubTypes` 声明多态：每个子类有 type 标签（如 `"add-column"`），序列化成 JSON 后能被 REST Server 还原成正确的子类——这是 REST 后端把 `alterTable(changes)` 整个变更列表传给服务端执行的前提（filesystem 后端在本地执行，不需要序列化）。

**AddColumn 的位置语义**：`addColumn(name, type)` 默认加到末尾；`addColumn(name, type, comment, Move.first(name))` 加到最前；`Move.after(name, refColumn)` 加到某列后。位置只影响列的展示 / 写入顺序，不影响 field id（id 永远是 highestFieldId+1）。读旧文件时列映射按 id 不按位置，所以调整位置对旧数据读取无影响——位置纯粹是"人类可读顺序"的元数据。这与某些系统"列位置即列身份"（按 ordinal 映射）形成对比：Paimon 位置可随意调而不破坏数据，因为身份锚在 id 上。

`AddColumn` 的 nullable 约束在生成阶段校验：

```java
// generateTableSchema 内 AddColumn 分支（简化）
if (!addColumn.isNullable()) {
    throw new IllegalArgumentException("ADD COLUMN ... cannot specify NOT NULL.");
}
int newId = highestFieldId + 1;   // 永不回收
```

把"算新 schema"做成纯函数 `generateTableSchema` 的好处：重试时只需重新取 latest 再算一遍，无副作用、无脏状态；也便于 REST 后端把 changes 序列化传给服务端由服务端算。

### 4.4 类型变更校验与列映射读取

读取旧数据文件时，文件元数据记录其写入时的 `schemaId`；读 schema（当前）与文件 schema 不一致时，`SchemaEvolutionUtil` 按 **field id**（非列名、非位置）做映射：当前 schema 有而旧文件没有的列补 null，类型不同则做安全转换（如旧文件 INT、当前 BIGINT，读时提升）。这就是"字段 ID 永不回收"的根本动因——列名可变、位置可变，唯有 field id 是稳定锚点。

类型变更校验在 `generateTableSchema` 的 `UpdateColumnType` 分支：默认只放行无损扩宽（数值位宽变大、精度变大），`allowExplicitCast=true`（显式 cast 或 `mergeSchema` 写入合并场景）才放宽到可能有损的转换。`mergeSchema(rowType, allowExplicitCast, ...)` 用于"写入时自动合并 schema"——把写入数据的 RowType 合并进当前 schema，新增列自动 addColumn，是 CDC 入湖自动加列的底层机制。

**实战：一次 `ALTER TABLE ADD COLUMN` 在源码里走过的路**（理解后能定位任何 schema 变更卡点）：

```
Flink/Spark SQL: ALTER TABLE t ADD COLUMN age INT
  → Catalog.alterTable(id, [SchemaChange.addColumn("age", INT, null, true)], false)
  → AbstractCatalog.alterTable: checkNotSystemTable / checkNotBranch
  → FileSystemCatalog.alterTableImpl → runWithLock(id, () -> schemaManager.commitChanges(changes))
  → SchemaManager.commitChanges: while(true) {
        old = latest()                            # 读 schema-5
        new = generateTableSchema(old, changes)   # 纯函数: age 分配 fieldId=highestFieldId+1, id=6
        if (commit(new)) return                   # tryToWriteAtomic(schema-6)
        # 写失败(别人抢先写了 schema-6) → 重试
     }
```

常见卡点对应源码位置：`age INT NOT NULL` → `generateTableSchema` 的 AddColumn nullable 校验抛错；删分区键 → DropColumn 校验抛错；并发 DDL 反复重试不收敛 → `commit` 一直返回 false（另有作业高频改 schema），需排查谁在改。

### 4.5 Schema 存储路径与回退分支

```
主分支:   {tableRoot}/schema/schema-{id}
其他分支: {tableRoot}/branch/branch-{branchName}/schema/schema-{id}
```

路径由 `branchPath() + schemaDirectory() + toSchemaPath()` 拼接：

```java
private String branchPath() { return BranchManager.branchPath(tableRoot, branch); }
public Path schemaDirectory() { return new Path(branchPath() + "/schema"); }
public Path toSchemaPath(long id) { return new Path(branchPath() + "/schema/" + SCHEMA_PREFIX + id); }
```

`commit` 里的 `SchemaValidation.validateFallbackBranch` 校验"回退分支"配置（`scan.fallback-branch`）——主分支查不到数据时回退到指定分支读，要求两分支 schema 兼容，这里在提交新 schema 时就提前校验避免运行期才报错。

**Schema 层排障速查**：

| 现象 | 根因 | 处置 |
|---|---|---|
| `ADD COLUMN ... cannot specify NOT NULL` | 新增列强制 nullable | 去掉 NOT NULL，需非空则 overwrite 重写 |
| `Cannot update partition column type` | 分区列类型不可改 | 重建表或新建分区方案 |
| 改类型报不兼容 | 非安全转换（如 BIGINT→INT） | overwrite 重写数据，或用 explicit cast 场景 |
| schema 文件目录爆量 | 高频单条 setOption / addColumn | 批量 `commitChanges(List)` |
| 并发 DDL 长时间不收敛 | 多作业高频改同表 schema | 排查谁在改，串行化 DDL |
| 读旧数据列错位 | 误以为按列名映射 | 实际按 field id 映射，确认未手工改 schema 文件 |
| `nested row type ... update not supported` | 嵌套 ROW 列不支持改类型 | 拆分或重建该列 |

**为何 schema 文件永不删**：任意存活的数据文件都带 `schemaId`，读它时要按对应 schema 做列映射（4.4）。即使该 schema 对应的列后来被删，只要还有用它写的数据文件存活，schema 文件就不能删。所以 schema 目录随时间单调增长是正常的——它远小于数据，通常无需治理；真要清理只能在确认无数据文件引用某 schema 后手工删（极少做）。

---

## 五、Snapshot 管理

### ① 要解决什么问题

时间旅行（查历史时点）、增量消费（只读新增变更）、回滚（恢复到正确状态）、精确一次（分布式重试不产生重复提交）。

### ② 设计原理与取舍（最重要）

- **Snapshot 只是元数据入口**：不含数据，只引用 ManifestList。建快照是 O(1)（写一个小 JSON），多快照共享底层数据文件，靠"无引用即可清理"管理生命周期。代价是需要引用计数式的过期清理（不能简单删快照就删数据）。
- **base/delta/changelog 三分**：全量读只看 base，增量消费只看 delta，过期清理只扫 delta，CDC 走 changelog——各取所需，避免增量消费者去 diff 两份全量。这是 Paimon 流式增量读高效的根因。
- **`(commitUser, commitIdentifier, commitKind)` 三元组去重**：实现精确一次；同一 user 内 identifier 单调递增（如 Flink job id + checkpoint id）。
- **LATEST hint 乐观加速**：避免每次 list 目录（对象存储 list 慢），但 hint 可能过时，需"看 id+1 是否存在"来验证——一个典型的"乐观假设 + 廉价验证 + 失败降级"模式。
- **watermark 只增不减**：保证流式单调，回滚也不回退 watermark——这是流式语义的必然要求，代价是回滚后可能跳过部分数据。

- **schemaId 关联而非内嵌 schema**：Snapshot 只存 `schemaId`（一个 long），不内嵌完整 schema。这让"改 schema 不重写 snapshot、多 snapshot 共享 schema"成立；读取时按 snapshot.schemaId 取对应 schema 文件。代价是读 snapshot 后要多一次 IO 取 schema，但 schema 文件小且可缓存，收益（解耦 + 共享）远大于成本。

### ③ 关键源码

见 5.3 的 `latestSnapshot()` 与 `HintFileUtils.findLatest()`。

### ④ 风险/陷阱/边界

- `commitIdentifier` 仅在同 `commitUser` 内唯一，跨 user 可重复——去重必须带上 user 与 kind。
- `snapshot.time-retained` 过短可能让运行中的长查询读到被删快照而失败；务必配 `num-retained.min` 兜底。
- Snapshot 删除只删 snapshot 文件，数据文件待无引用后由过期任务清理（异步）。
- LATEST hint 在高并发写下可能短暂落后，`findLatest` 会自动验证并降级到全量扫描，但全量扫描在对象存储上慢——监控这一退化路径的频率。

### ⑤ 收益与代价

收益：O(1) 快照、统一支撑时间旅行 / 增量 / 回滚 / 精确一次。代价：需要引用计数式过期清理、hint 需验证、快照文件数需控量。

**快照保留配置的协同语义**（`time-retained` + `num-retained.min/max` 三者如何共同决定删谁）：过期任务从最早快照向后扫，一个快照被删的条件是——同时满足"超出 `time-retained` 时长"且"删后存活数 ≥ `num-retained.min`"；同时存活数不得超 `num-retained.max`（超了即使没到时间也删）。即 min 是"无论多旧都至少保留这么多"的下限保护，max 是"无论多新都最多保留这么多"的上限，time 是中间的时间策略。

**典型误配与后果**：

- 只配 `time-retained=10min` 不配 `num-retained.min`：10 分钟内若提交 100 个快照全保留（没到时间），存储暴涨；且 10 分钟后可能一次性删到只剩很少，长查询撞空。
- `time-retained` 短于最长查询耗时：运行 15 分钟的批查询读 snapshot-N，期间 N 被过期删除 → 查询读文件失败。**对策**：`num-retained.min` 给足（如 10~20）+ `time-retained` ≥ 最长查询时长。被 Tag 引用的快照不受过期影响（6.1），关键版本打 tag 是更稳的保护。

### 5.1 Snapshot 字段表（v3）

> 源码：`paimon-api/.../Snapshot.java`

| 字段 | 类型 | 可空 | 说明 |
|---|---|---|---|
| version / id / schemaId | int / long / long | 否 | 格式版本 v3 / 快照 id（从 1） / 对应 schema |
| baseManifestList(+Size) | String(+Long) | Size 旧版可空 | 基线 manifest list（历史合并结果） |
| deltaManifestList(+Size) | String(+Long) | Size 旧版可空 | 本次增量（ADD / DELETE） |
| changelogManifestList(+Size) | String(+Long) | 是 | CDC 变更日志 |
| indexManifest | String | 是 | 索引（Deletion Vector 等） |
| commitUser / commitIdentifier / commitKind | String / long / enum | 否 | 去重三元组 |
| timeMillis | long | 否 | 提交时间 |
| totalRecordCount / deltaRecordCount | long | 否 | 累计 / 本次净变更行数 |
| changelogRecordCount / watermark / statistics / properties | 各 | 是 | changelog 行数 / 水位 / 统计文件名 / 附加属性 |
| nextRowId | Long | 是 | Row Tracking 下一可用行 id |

`commitIdentifier` 语义：相同 identifier 的快照读取结果必须一致，较小者含较老数据——这是 Flink 精确一次去重的基础。这也解释了为什么 commitIdentifier 必须在同一 commitUser 内单调递增：去重靠"扫快照链找匹配三元组"，若 identifier 非单调，无法判断"这个 identifier 是否已提交过"。Flink 用 checkpoint id（天然单调）作 identifier 正合此意。

**Snapshot 格式版本演进（version 字段）**：v1（早期）只有单一 manifestList；v2 引入 base/delta 分离 + commitKind；v3（当前）加 changelogManifestList、watermark、statistics、properties、nextRowId 等。读取器按 version 决定如何解析——`Snapshot` 类用 `@JsonIgnoreProperties(ignoreUnknown=true)` 容忍未知字段，使新版写的快照能被旧版读（未知字段忽略），旧版写的快照被新版读时缺失字段为 null（如 `baseManifestListSize` 在 ≤1.0 为 null，读取时按 null 处理回退到全量 read）。这种"向前 + 向后"双向兼容是滚动升级的基础——集群里新老版本 Paimon 可短期共存。

**`baseManifestListSize` 为何要存**：早期版本（≤1.0）读 ManifestList 要先 `getFileStatus` 拿文件大小再读，多一次 IO；新版把 size 直接存进 Snapshot，读时省掉 stat 调用（尤其对象存储 stat 慢）。为 null（旧快照）时回退到先 stat 再读。这是个典型的"用元数据冗余换 IO"小优化——存一个 long，省一次远程 stat。同理 `deltaManifestListSize` / `changelogManifestListSize`。读取代码必须判 null 兼容旧快照，这也是 5.1 字段表里这些 Size 标"旧版可空"的原因。

**`properties` 字段**：v3 加的 `Map<String,String>` 附加属性，存提交时的额外元信息（如来源标记、自定义审计字段）。它让 Snapshot 可携带"约定外"的信息而不改格式——新需求往 properties 塞，不用每次都升 version。这是"留扩展位"的设计：核心字段固定、properties 兜底未来需求。配合 `@JsonIgnoreProperties(ignoreUnknown=true)`，老读者遇到 properties 里的未知约定也不会崩。这种"固定字段 + 开放 properties + 忽略未知"的三件套，是 Paimon 元数据格式能长期平滑演进的工程基础。

CommitKind 枚举（`Snapshot.java`）：

```java
public enum CommitKind {
    APPEND,     // 追加新数据，不删已有文件
    COMPACT,    // 合并压缩，不改逻辑数据，只改物理组织
    OVERWRITE,  // 覆写分区或删文件后加新文件
    ANALYZE     // 收集统计信息
}
```

把 COMPACT 单列：它不改逻辑数据，冲突检测可更宽松（不会与 APPEND 产生数据层冲突，只可能文件引用冲突），且流式消费者可跳过 COMPACT 快照只读 APPEND。

**四种 CommitKind 的产生场景与下游语义**：

| CommitKind | 由什么产生 | 流式消费者如何对待 |
|---|---|---|
| APPEND | 普通写入新数据 | 消费其 delta 的新增行 |
| COMPACT | LSM compaction / 小文件合并 | 跳过（数据未变，只是物理重组） |
| OVERWRITE | INSERT OVERWRITE / APPEND 含 DELETE/DV 升级 | 视为分区 / 文件替换，需重读受影响范围 |
| ANALYZE | 统计信息收集（ANALYZE TABLE） | 跳过（只更新 statistics，无数据变更） |

流式读默认只关心 APPEND（与 OVERWRITE 的数据变更），COMPACT / ANALYZE 不产出新行——这就是为什么"提交次数 ≠ 流式消费到的变更次数"。下游若把每个快照都当数据变更处理，会在 COMPACT 快照上空转甚至重复消费。`Snapshot.commitKind()` 是流式 source 决定"这个快照要不要发数据"的判据。

**`nextRowId` 与 Row Tracking**：v3 新增的 Row Tracking 特性给每行分配全局唯一、单调的 row id（用于 CDC 去重 / 行级血缘 / 增量物化视图）。`nextRowId` 记录"下一个可分配的 row id"，提交时 `tryCommitOnce` 给本次新增的数据文件分配 `[nextRowId, nextRowId+rowCount)` 区间并更新 snapshot.nextRowId。ManifestFileMeta 的 `minRowId / maxRowId`（7.2）记录文件的 row id 区间用于剪枝，ConflictDetection 的 `checkRowIdRangeConflicts`（8.4）保证并发分配的区间不重叠。这是 8.1 里"APPEND 含 Row ID 检查时也 allowRollback=true"的原因——row id 分配冲突需回滚重分配。

`commitIdentifier` / `commitUser` 的另一用途是**审计追溯**：`$snapshots` 系统表暴露这两列，排查"某个错误数据是哪个作业、哪个 checkpoint 写进来的"时按它定位。

**totalRecordCount vs deltaRecordCount**：前者是累计（截至本快照表的总行数），后者是本次净变更（新增 - 删除）。两者都是"行数"非"文件数"——来自 manifest entry 里 DataFileMeta 的 rowCount 累加。它们让 `$snapshots` 能直接看出每次提交的数据量级、表的增长曲线，无需扫数据文件。负的 deltaRecordCount 说明该快照净删除（DELETE / OVERWRITE 删多于增）。这两个计数在 `tryCommitOnce` 构建 Snapshot 时算出：base 的 totalRecordCount + delta 的净变更 = 新 totalRecordCount。它们是元数据级的"廉价统计"，比 `SELECT count(*)` 快得多（后者要扫数据）。

### 5.2 base/delta/changelog 三分法

```
Snapshot
 ├ baseManifestList   → [ManifestFileMeta...] → ManifestFile [Entry(ADD)...]   全量(合并后)
 ├ deltaManifestList  → [ManifestFileMeta]    → ManifestFile [ADD/DELETE]      本次增量
 └ changelogManifestList(可选) → ...           changelog entries                CDC
```

`baseManifestList` 经 `ManifestFileMerger` 合并优化，可高效读全量；`deltaManifestList` 只含本次 ADD / DELETE，支撑两件事：

1. **快速过期**：清理旧快照时只需扫 delta 即知哪些文件本次新增 / 删除。
2. **流式增量读**：增量消费者只读 delta 即得变更，无需 diff 两份全量。

changelog 单列是因其有独立生命周期与存储策略（changelog 可单独过期、单独配 producer）。**为何不把所有文件放一个列表**：增量与全量分离是关键性能优化；混在一起的话增量消费者要对比两版全量列表算差异，代价极高。一句话：base 服务"读全量"，delta 服务"读增量 + 快速过期"，changelog 服务"CDC 行级消费"——三者的读者不同、生命周期不同，故分三个 list 各自演进。

**delta 与 changelog 的区别**（易混）：delta 记的是"数据文件的增删"（哪些 DataFile 进 / 出表），changelog 记的是"行级变更日志"（INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE 的具体行）。主键表更新时，delta 反映新数据文件覆盖旧的（文件级），changelog 则产出"老值 -U / 新值 +U"供下游 CDC 消费（行级）。`changelog-producer` 配置（none / input / lookup / full-compaction）决定是否及如何生成 changelog——none 则 changelogManifestList 为空，下游只能拿到 delta 的文件级变更、自己算行差异。这是 Paimon 流式 CDC 能力的核心开关。

### 5.3 SnapshotManager 与 LATEST hint

> 源码：`paimon-core/.../utils/SnapshotManager.java`、`utils/HintFileUtils.java`

`latestSnapshot()` 双重加载：有 `snapshotLoader`（REST 模式）先走它，抛 `UnsupportedOperationException` 再回退 `latestSnapshotFromFileSystem()`；结果入 `Cache<Path,Snapshot>`：

```java
public Snapshot latestSnapshot() {
    Snapshot snapshot;
    if (snapshotLoader != null) {
        try { snapshot = snapshotLoader.load().orElse(null); }
        catch (UnsupportedOperationException ignored) { snapshot = latestSnapshotFromFileSystem(); }
    } else { snapshot = latestSnapshotFromFileSystem(); }
    if (snapshot != null && cache != null) cache.put(snapshotPath(snapshot.id()), snapshot);
    return snapshot;
}
```

文件系统侧最终调 `HintFileUtils.findLatest`，逻辑精确如下：

```java
@Nullable
public static Long findLatest(FileIO fileIO, Path dir, String prefix, Function<Long,Path> file)
        throws IOException {
    Long snapshotId = readHint(fileIO, LATEST, dir);          // 读 {dir}/LATEST
    if (snapshotId != null && snapshotId > 0) {
        long nextSnapshot = snapshotId + 1;
        if (!fileIO.exists(file.apply(nextSnapshot))) {        // id+1 不存在 → hint 确为最新
            return snapshotId;
        }
    }
    return findByListFiles(fileIO, Math::max, dir, prefix);    // 否则全量 list 取最大
}
```

`readHint` 自带 3 次重试（间隔 1ms）抵御瞬时读失败；`commitLatestHint` 写 hint 时也 3 次重试 + 随机退避（500~1500ms），失败抛出。**为何要验证 id+1**：hint 可能因并发写、文件系统缓存、网络延迟而落后；用"下一个不存在"廉价确认最新——命中时仅 1~2 次文件操作，失效时自动降级全量扫描，兼顾性能与正确性。`EARLIEST` 同理，但验证条件是"该 id 文件存在"（`findEarliest`）。`isEmpty()` 也复用：`readHint(EARLIEST)==null` 即视为空表。

**hint 不是真相、只是加速器**：hint 文件写失败（`commitLatestHint` 抛异常）不影响正确性——下次 `findLatest` 读到旧 hint，验证 id+1 存在后自动降级全量扫描仍得正确结果。所以 hint 写入用"尽力而为 + 重试"而非"必须成功"。这是把"性能优化"与"正确性保证"彻底解耦的设计：真相永远是"目录里 id 最大的 snapshot 文件"，hint 只是大多数情况下省去 list 的捷径。理解这点就明白为何高并发下偶尔 list 全目录不是 bug 而是预期的降级路径——监控它的频率而非试图消除它。

### 5.4 快照过期与去重查询

SnapshotManager 其他关键方法：`snapshot(id)`（带缓存读）、`snapshotExists(id)`、`earliestSnapshotId()`（`findEarliest` + `EARLIEST_SNAPSHOT_DEFAULT_RETRY_NUM=3` 重试容忍并发删除）、`latestSnapshotOfUser(commitUser)`（去重用，扫快照链找该 user 最新提交）、`deleteSnapshot(id)`。

`latestSnapshotOfUser` 是精确一次的关键查询：Flink 重启后要知道"我（commitUser）上次提交到哪个 checkpoint（commitIdentifier）了"，据此跳过已提交的 checkpoint。`earliestSnapshotId` 带 3 次重试是因为过期任务可能正在删最早快照，单次读可能撞上"文件刚被删"的窗口。

**时间旅行 / 增量读如何落到 Snapshot**（理解 scan.mode 的源码语义）：

| scan 配置 | 解析到的起始/范围 | 走哪个 SnapshotManager 方法 |
|---|---|---|
| `scan.snapshot-id=N` | 从 snapshot-N 读全量 | `snapshot(N)` |
| `scan.timestamp-millis=T` | 找 timeMillis ≤ T 的最大快照 | 遍历快照链按 timeMillis 二分 |
| `scan.mode=from-snapshot` + id | 从该 id 起读 delta（增量） | 逐个 `snapshot(i)` 读 deltaManifestList |
| `scan.mode=latest-full`（默认流式起点） | 当前 latest 全量 + 后续增量 | `latestSnapshot()` 再追 delta |
| `scan.tag-name=X` / `scan.version` | 读 tag X 指向的快照 | TagManager 读 tag 文件 |

时间旅行查历史时点的本质是"定位到某个 snapshot id 再按全量读路径走"；增量读的本质是"连续读多个 snapshot 的 deltaManifestList"。这就是为什么 base/delta 分离（5.2）能让增量读高效——增量读永远只碰 delta。

```sql
-- 时间旅行: 三种等价定位方式
SELECT * FROM t /*+ OPTIONS('scan.snapshot-id'='100') */;                  -- 按 id
SELECT * FROM t /*+ OPTIONS('scan.timestamp-millis'='1704067200000') */;  -- 按时间
SELECT * FROM t /*+ OPTIONS('scan.tag-name'='release_v1') */;             -- 按 tag
-- 增量读: 读 [start, end] 之间的 delta
SELECT * FROM t /*+ OPTIONS('incremental-between'='95,100') */;
-- 流式: 从某快照起持续读后续 delta
SELECT * FROM t /*+ OPTIONS('scan.mode'='from-snapshot','scan.snapshot-id'='100') */;
```

这些 hint 最终都落到 `SnapshotManager` 的某个方法 + 三分法的某个 manifest list 读取上——理解了 5.1~5.3 就能反推任意 scan hint 的源码行为。

**过期清理（SnapshotDeletion / ExpireSnapshots）与本章的关系**：过期任务按 `snapshot.time-retained` + `num-retained.min/max` 决定删哪些旧快照；删快照时要算"哪些数据文件不再被任何存活快照引用"——它只需扫被删快照的 deltaManifestList（本次新增 / 删除的文件），而非全量 base，这正是 delta 设计的第三个收益。被 Tag 引用的快照不在删除范围（6.1）。

具体清理算法（为何只扫 delta 就够）：要删快照 S（id=N），其 deltaManifestList 记了 S 相对 N-1 新增 / 删除的文件。一个文件能被物理删除的条件是"它在某个被删快照里被 ADD，且在所有存活快照里都不再被引用"。由于 base 是历史合并、delta 是增量，逐个删最早快照时只需看该快照 delta 里 ADD 的文件是否在更晚的存活快照里仍有效——有效则保留、无效（已被后续 DELETE）则可清。这把"全量引用计数"降成"扫 delta 增量"，是过期清理在大表上可行的关键。**注意**：清理是后台异步的，删 snapshot 文件与删数据文件分两步，中间崩溃也安全（孤儿数据文件下次清理再回收，不会误删存活文件）。

---

## 六、Tag 与 Branch

### ① 要解决什么问题

Tag：给重要快照命名并阻止过期（月末快照、发布版本）。Branch：在同表上开独立演进路径做并行开发 / AB 实验，互不污染。

### ② 设计原理与取舍（最重要）

- **Tag = Snapshot 副本（+TTL）**：tag 文件内容就是被标记 snapshot 的 JSON（可选追加 TTL 字段），读 tag 即读 snapshot，无需额外映射表。实现极简，但代价是 tag 会阻止其引用的数据被过期清理。
- **Branch = 写时复制**：建分支只复制 schema / snapshot / tag 这些小元数据文件，data / manifest 目录共享；O(1) 创建，新写入产生新文件。删分支时需判定哪些文件是分支独占的（逻辑复杂）。
- **fast-forward 只快进不合并**：要求主分支无新提交（分支是主分支的直接后继），避免三方合并的冲突解决复杂度——若主分支已前进，fast-forward 失败需人工处理。

为何 Tag 文件直接是 Snapshot JSON（+TTL）而非引用：若 Tag 只存"指向 snapshot-N"的引用，那 snapshot-N 被过期删除后 Tag 就悬空了。Paimon 让 Tag 文件**内联完整 snapshot 内容**——即使原 snapshot 文件被删，Tag 仍自包含可读（它引用的数据文件因 Tag 存在而不被清理）。这就是 6.1 "Tag 阻止数据清理" 的实现基础：Tag 文件本身就是一个独立的、永不随快照链过期的 snapshot 副本。`trimToSnapshot()` 则反向——从 Tag 剥掉 TTL 字段还原成纯 Snapshot（建分支时用）。

### ③ 关键源码

Tag 创建（`TagManager`，幂等覆写）：

```java
private void createOrReplaceTag(Snapshot snapshot, String tagName, Duration timeRetained,
        List<TagCallback> callbacks) {
    validateNoAutoTag(tagName, snapshot);
    String content = timeRetained != null
        ? Tag.fromSnapshotAndTagTtl(snapshot, timeRetained, LocalDateTime.now()).toJson()
        : snapshot.toJson();
    fileIO.overwriteFileUtf8(tagPath(tagName), content);   // 直接覆写(幂等)
    if (callbacks != null) callbacks.forEach(cb -> cb.notifyCreation(tagName, snapshot.id()));
}
```

### ④ 风险/陷阱/边界

- Tag 阻止其引用的 snapshot 与数据文件被过期清理，可能让存储持续增长——用 TTL 自动回收。
- 分支不能直接 `DROP TABLE $branch_x`，须 `dropBranch`。`dropBranch` 会删分支独有的数据文件，删前确认无需保留。
- 分支名禁用：`main`（保留）、空白、纯数字（避免与 snapshot id 混淆）。
- fast-forward 不是 merge：主分支若在分支期间有新提交，fast-forward 失败。

### ⑤ 收益与代价

收益：命名快照保护 + 轻量并行开发，全部共享数据零拷贝。代价：删分支需判文件独占性、Tag 长期保留拖住存储。

### 6.1 TagManager 与 Tag TTL

> 源码：`paimon-core/.../utils/TagManager.java`、`paimon-core/.../tag/Tag.java`

路径 `{tablePath}/[branch/branch-{name}/]tag/tag-{tagName}`。核心方法：`createTag` / `replaceTag` / `deleteTag(tagName, tagDeletion, snapshotManager, callbacks)` / `deleteAllTagsOfOneSnapshot` / `renameTag` / `tagExists` / `taggedSnapshots`。

`Tag` 继承 `Snapshot`，加两字段 `tagCreateTime`(LocalDateTime?) / `tagTimeRetained`(Duration?)：

```java
// 过期判定
if (tag.tagCreateTime() != null && tag.tagTimeRetained() != null) {
    LocalDateTime expire = tag.tagCreateTime().plus(tag.tagTimeRetained());
    if (expire.isBefore(LocalDateTime.now())) { /* 删除 */ }
}
```

**向后兼容**：未指定 TTL 时不写这两字段，老版本读取器靠 `@JsonIgnoreProperties(ignoreUnknown=true)` 忽略未知字段，仍能把 tag 文件当普通 Snapshot 解析。

`renameTag` / `replaceTag` 的语义：`renameTag(from, to)` 改名（拷贝到新名删旧名）；`replaceTag` 用新 snapshot 内容覆写已有 tag（`overwriteFileUtf8` 幂等覆写）——常用于"把某个稳定 tag 指向更新的快照"。`deleteAllTagsOfOneSnapshot` 处理"多个 tag 指向同一快照"时的批量删除（一个快照可被多个 tag 命名）。这些操作都只动 tag 文件（小 JSON），不碰数据——Tag 管理的轻量正源于"tag = 一个自包含 snapshot JSON 副本"这个朴素设计。

`taggedSnapshots()` 返回所有 tag 对应的快照列表，过期清理时用它判断"哪些快照被 tag 引用因而不能删"。这把 Tag 的"保护语义"落到清理逻辑：过期任务在删快照前先查 `taggedSnapshots`，被引用的跳过。所以即便快照超出 `time-retained`，只要有 tag 指着就不删——Tag 是凌驾于快照过期策略之上的"钉子"。生产中给关键版本打 tag 比调大 `num-retained` 更精准（后者保留一段连续快照，前者只钉住需要的那几个）。

**删除 Tag 的安全逻辑**：若被标记 snapshot 仍在快照链中 → 只删 tag 文件；若 snapshot 已过期且 tag 是最后引用 → 还需清理 tag 独占的数据文件（`tagDeletion` 负责判定哪些文件不再被任何快照 / tag 引用）。

**自动创建 Tag（`tag.automatic-creation`）**：除手动 `createTag`，Paimon 支持按时间周期自动打 tag（如每天 / 每小时一个），由 `TagPeriodHandler` 驱动。配置 `tag.automatic-creation=process-time/watermark`、`tag.creation-period=daily/hourly`、`tag.default-time-retained`（自动 tag 的 TTL）。`validateNoAutoTag(tagName, snapshot)`（见上面 `createOrReplaceTag`）防止手动 tag 名与自动 tag 命名规则撞车。自动 tag 常用于"每天保留一个稳定快照供下游批读"——下游按 tag 名（如 `2024-01-01`）读当天数据，不受快照过期影响。这是流批一体里"流写 + 批读固定版本"的关键机制。

**process-time vs watermark 触发**：`automatic-creation=process-time` 按墙钟时间到点打 tag（简单但若作业有延迟，tag 对应的数据可能不完整）；`=watermark` 按事件时间 watermark 越过周期边界才打（保证 tag 数据"事件时间完整"，但 watermark 推不动就不打）。下游若严格要求"某天的 tag 含当天全部数据"应用 watermark 触发——它把 5.3 的 watermark 单调性与 tag 创建挂钩，确保 tag 是"事件时间意义上完整"的版本。这是 Paimon 把流式 watermark 语义延伸到版本管理的一个精巧设计。

### 6.2 BranchManager 写时复制

> 源码：`utils/BranchManager.java`（接口）、`utils/FileSystemBranchManager.java`（实现）

接口方法：`createBranch(branchName)`（从最新 schema 建空分支）/ `createBranch(branchName, tagName)`（从 tag 建）/ `dropBranch` / `fastForward` / `renameBranch` / `branches()`。

静态工具：

```java
static String branchPath(Path tablePath, String branch) {
    return isMainBranch(branch) ? tablePath.toString()
        : tablePath.toString() + "/branch/" + BRANCH_PREFIX + branch;
}
static String normalizeBranch(String branch) {
    return StringUtils.isNullOrWhitespaceOnly(branch) ? DEFAULT_MAIN_BRANCH : branch;  // "main"
}
```

分支名验证规则（`validateBranch`）：(1) 不能是 `main`（保留名）；(2) 不能空 / 全空白；(3) 不能纯数字（避免与 snapshot id 混淆——`branchPath` 解析 `branch/branch-{name}` 时纯数字名会引起歧义）。`normalizeBranch` 把 null / 空白归一为 `main`，使"不指定分支"与"指定 main"等价——上层无需到处判 null。

从 Tag 建分支（复制 tag + snapshot + 到该 schemaId 为止的所有 schema，data / manifest 不复制）：

```java
public void createBranch(String branchName, String tagName, boolean ignoreIfExists) {
    validateBranch(branchName);
    Snapshot snapshot = tagManager.getOrThrow(tagName).trimToSnapshot();
    fileIO.copyFile(tagManager.tagPath(tagName),
                    tagManager.copyWithBranch(branchName).tagPath(tagName), true);
    fileIO.copyFile(snapshotManager.snapshotPath(snapshot.id()),
                    snapshotManager.copyWithBranch(branchName).snapshotPath(snapshot.id()), true);
    copySchemasToBranch(branchName, snapshot.schemaId());
}
```

文件布局示意：

```
{table}/
  schema/  snapshot/  tag/                 # main 分支元数据
  branch/branch-dev/{schema,snapshot,tag}/ # dev 分支元数据(复制而来)
  manifest/  bucket-*/                      # 全分支共享(写时复制)
```

**为何共享 manifest / data**：分支隔离靠复制 snapshot / schema / tag 实现，底层文件共享——建分支是轻量 O(1)，新写入产生新文件不改已有文件。这与 Git 的对象共享思路一致。

**fastForward 的语义与限制**：`fastForward(branchName)` 把主分支指针"快进"到分支的最新快照——前提是分支是主分支的直接后继（主分支自分支创建后无新提交）。实现上需把分支的 snapshot / schema / tag 复制 / 提升回主分支，并更新主分支 LATEST hint。若主分支已前进（`main: 100→101`，`dev: 100→102`），fastForward 检测到主分支非分支祖先而失败——Paimon 不做三方合并，需人工取舍。这与 Git fast-forward only 的 merge 策略同理：简单、无冲突解决逻辑，但要求线性历史。

**删分支的文件独占判定**：`dropBranch` 删 `branch/branch-{name}/` 元数据目录后，还需识别"仅该分支写入、不被主分支或其它分支引用"的数据文件并清理——这比建分支复杂得多（建分支只复制元数据，删分支要做引用分析）。这就是"分支隔离 vs 数据共享"取舍的代价落点。

**Tag / Branch 典型工作流**（流批一体场景）：

```sql
-- 1. 流作业持续写主分支(write-only), 独立 compaction 作业做合并
-- 2. 每天自动打 tag(或手动)固定一个稳定版本
CALL sys.create_tag('my_db.orders', 'snapshot_2024_01_01', 100);
-- 3. 批作业按 tag 读固定版本, 不受快照过期影响
SELECT * FROM `my_db.orders` /*+ OPTIONS('scan.tag-name'='snapshot_2024_01_01') */;
-- 4. 从 tag 开实验分支跑新逻辑, 不污染生产
CALL sys.create_branch('my_db.orders', 'exp_dedup', 'snapshot_2024_01_01');
INSERT INTO `my_db.orders$branch_exp_dedup` SELECT ... ;  -- 在分支上写
SELECT * FROM `my_db.orders$branch_exp_dedup`;            -- 验证
-- 5. 实验通过且主分支无新提交 → fast-forward 合回; 否则丢弃
CALL sys.drop_branch('my_db.orders', 'exp_dedup');
```

要点：tag 解决"批读需要稳定版本"，branch 解决"实验不污染生产"，二者都靠"共享数据文件 + 独立元数据"实现零拷贝。**生产建议**：实验分支用完即删（避免独占文件长期占存储），重要版本用 tag 长期固定（配 TTL 自动回收临时 tag）。

---

## 七、Manifest 三级结构

### ① 要解决什么问题

大表可有数百万数据文件。若全记在 Snapshot 里，每次提交都要重写整块；查询时也无法按分区 / 桶 / level 快速剪枝；提交多了 manifest 碎片化拖慢读取。需要可伸缩、增量、可过滤、可合并的元数据组织。

### ② 设计原理与取舍（最重要）

- **三级**（Snapshot→ManifestList→ManifestFile→DataFile）：Snapshot 恒定小（只存 list 文件名），每次提交只写新增的 ManifestFile + 新 ManifestList。增量写入是核心收益。
- **ManifestFileMeta 摘要剪枝**：在 list 层就带 partitionStats、min/maxBucket、min/maxLevel，查询时跳过整个无关 ManifestFile，不读其内容。用少量统计换大量 IO 节省。
- **ADD/DELETE 逻辑化**：entry 不动文件系统，提交是纯追加写；合并所有 entry 得有效文件集。这使提交无需修改已有文件，天然适配对象存储。
- **取舍**：多一层间接与合并成本，换来大表可伸缩与查询剪枝；统计信息占额外存储，但性价比高。

### ③ 关键源码

ManifestList 读数据 manifest（base+delta，`ManifestList.java`）：

```java
public List<ManifestFileMeta> readDataManifests(Snapshot snapshot) {
    List<ManifestFileMeta> result = new ArrayList<>();
    result.addAll(read(snapshot.baseManifestList(), snapshot.baseManifestListSize()));
    result.addAll(readDeltaManifests(snapshot));
    return result;
}
```

### ④ 风险/陷阱/边界

- `minBucket / maxBucket / minLevel / maxLevel / minRowId / maxRowId` 是后加优化字段，旧文件里为 null，用前必判空否则 NPE。
- Manifest 合并非实时：受 `manifest.merge-min-count`（默认 30）阈值约束，未达阈值不合并。
- `manifest.target-file-size` 过小→ManifestFile 过多、读小文件多；过大→合并成本高。对象存储建议增大。
- ManifestList / ManifestFile 在 snapshot 过期前不会清理。

### ⑤ 收益与代价

收益：支撑百万级文件、增量提交、多维剪枝。代价：层级与合并复杂度、统计信息存储开销、对象存储下小 manifest 读较慢。

### 7.1 三级结构与 ObjectsFile

> 源码：`manifest/ManifestList.java`、`manifest/ManifestFile.java`（均继承 `ObjectsFile<T>`）

```
Snapshot (JSON)
  │ baseManifestList / deltaManifestList / changelogManifestList (文件名)
  ▼
ManifestList (二进制 ObjectsFile<ManifestFileMeta>)
  │ fileName 引用
  ▼
ManifestFile (二进制 ObjectsFile<ManifestEntry>)
  │ DataFileMeta 引用
  ▼
Data Files (ORC / Parquet / Avro)
```

`ManifestList` 是 `ObjectsFile<ManifestFileMeta>`，`write` 用 `writeWithoutRolling`（list 文件不大、不滚动）；`ManifestFile` 是 `ObjectsFile<ManifestEntry>`，支持滚动写（达 `target-file-size` 自动开新文件）。读方法：`readAllManifests`(data+changelog) / `readDataManifests`(base+delta) / `readDeltaManifests` / `readChangelogManifests`。

**`ObjectsFile<T>` 是什么**：一个把对象列表序列化进单个自描述文件的通用容器——内部用 Paimon 的行格式（含 schema 头）把每个 T 写成一行，读时按行反序列化。manifest 用它而非 JSON 是因为：(1) 二进制紧凑（百万 entry 时 JSON 体积与解析都不可接受）；(2) 自描述（文件头带 schema，向后兼容字段增减）；(3) 支持按 offset 部分读。ManifestList 不滚动是因其条目少（几十到几百个 ManifestFileMeta）；ManifestFile 滚动是因 entry 可能极多（一个大提交几十万文件），需控制单文件大小。

`readDataManifests` 返回 base + delta 的全部 ManifestFileMeta（全量视图）；`readDeltaManifests` 只返回 delta（增量视图）；`readChangelogManifests` 取 changelog。这三个方法的分工直接对应 5.2 三分法——全量读调第一个，增量 / CDC 读调后两个，各取所需不互相拖累。

### 7.2 ManifestFileMeta 过滤剪枝

> 源码：`manifest/ManifestFileMeta.java`

| 字段 | 类型 | 用途 |
|---|---|---|
| fileName / fileSize | String / long | 文件定位 |
| numAddedFiles / numDeletedFiles | long | ADD / DELETE 计数（合并决策） |
| partitionStats | SimpleStats | 分区 min / max / nullCount → 分区剪枝 |
| schemaId | long | 写入时 schema |
| minBucket / maxBucket | Integer? | 桶剪枝 |
| minLevel / maxLevel | Integer? | LSM level 剪枝 |
| minRowId / maxRowId | Long? | Row Tracking 剪枝 |

`FileStoreScan` 用 `partitionStats` + 分区谓词在 list 层就过滤掉整个无关 ManifestFile，避免读其全部 entry——这是大表查询性能的关键剪枝点：

```java
// 读取流程(FileStoreScan, 简化)
for (ManifestFileMeta meta : manifestList.readDataManifests(snapshot)) {
    if (!partitionFilter.test(meta.partitionStats())) continue;   // 剪枝：整文件跳过
    entries.addAll(manifestFile.read(meta.fileName(), meta.fileSize()));
}
entries = FileEntry.mergeEntries(entries);   // 合并得有效文件列表
```

**为何 min/maxBucket 等可空**：后加的优化字段，为兼容旧版生成的 meta 设为可空；为 null 时该维度无法剪枝但不影响正确性（退化为读取后再过滤）。

**`partitionStats`（SimpleStats）的结构与剪枝原理**：它对该 ManifestFile 内所有 entry 的分区字段值聚合出 `minValues` / `maxValues`（BinaryRow）+ `nullCounts`。查询带分区谓词（如 `dt = '2024-01-01'` 或 `dt > '2024-01'`）时，`PartitionPredicate` 用 min/max 做区间判断：若谓词与 [min, max] 无交集，整个 ManifestFile 跳过。这是"分区裁剪"的元数据级实现——不读 ManifestFile 内容就能排除整批文件。代价是每个 ManifestFile 多存一份 SimpleStats，但相对其包含的成百上千 entry，开销可忽略，剪枝收益极大。桶剪枝（`minBucket/maxBucket`）、level 剪枝（`minLevel/maxLevel`）同理——分别支持"只查特定 bucket"和"LSM 分层读取"。

### 7.3 ManifestEntry 与合并语义

> 源码：`manifest/ManifestEntry.java`、`manifest/FileKind.java`

ManifestEntry 字段：`_KIND`(FileKind: ADD=0 / DELETE=1)、`_PARTITION`(BinaryRow)、`_BUCKET`(int)、`_TOTAL_BUCKETS`(int)、`_FILE`(DataFileMeta 完整元数据)。

```java
public enum FileKind { ADD((byte) 0), DELETE((byte) 1); }
```

ADD / DELETE 是逻辑标记（DELETE 不立即删物理文件，过期才清）。`FileEntry.mergeEntries` 把所有 entry 归并：同一文件最后一个 entry 决定其存在与否（ADD 后又 DELETE 即抵消），得到有效文件列表。这是"提交即追加、读取即合并"的核心——写路径永不修改旧文件，读路径通过合并还原逻辑状态。

合并语义的具体例子：

```
ManifestFile-1: [ADD file-a, ADD file-b]
ManifestFile-2: [DELETE file-a, ADD file-c]   # compaction 把 a 合进 c
ManifestFile-3: [ADD file-d]
                       ↓ FileEntry.mergeEntries
有效文件: {file-b, file-c, file-d}            # file-a 的 ADD 被后续 DELETE 抵消
```

注意合并是按"文件标识 +（分区, 桶）"分组判定的——同一物理文件在同一 (partition, bucket) 下的 ADD 与 DELETE 才能抵消。这保证了 LSM compaction（把若干小文件合成大文件 = DELETE 旧 + ADD 新）在元数据层正确反映：旧文件逻辑消失、新文件逻辑出现，读取者只看到合并后的有效集。`SimpleFileEntry` 是 `ManifestEntry` 的轻量投影（只留冲突检测必需字段），冲突检测用它而非完整 entry 以省内存。

**为何冲突检测用 SimpleFileEntry 而非完整 ManifestEntry**：冲突检测只需要 (kind, partition, bucket, 文件标识, keyRange, level, rowIdRange)，不需要完整 DataFileMeta（含列统计、schema id、各种 min/max 等几十个字段）。高并发提交时，变更分区可能含成千上万文件，若都用完整 entry 做检测，内存与 GC 压力大；`SimpleFileEntry` 把每条压到必需字段，让冲突检测在大变更集下仍可控。这也是 8.3 增量冲突检测缓存 `baseDataFiles` 用 `List<SimpleFileEntry>` 而非完整 entry 的原因——缓存的是轻量投影，重试时增量 merge 也省内存。这种"为特定用途做窄投影"的模式在 Paimon 里反复出现（如 `SimpleStats` 之于完整统计），核心是"按需取最小字段集"以控成本。

### 7.4 Manifest 合并时机与策略

每次提交（`tryCommitOnce` 构建新 Snapshot 时）都会尝试合并旧 base manifest：

```
mergeBefore = manifestList.readDataManifests(latestSnapshot)
mergeAfter  = ManifestFileMerger.merge(mergeBefore, manifestFile,
                targetSize,          // manifest.target-file-size (8MB)
                mergeMinCount,       // manifest.merge-min-count (30)
                fullCompactionSize,  // manifest.full-compaction-threshold-size (16MB)
                partitionType, parallelism)
baseManifestList = manifestList.write(mergeAfter)
```

合并触发条件：小 ManifestFile 数超过 `merge-min-count`（减少后续读 IO），或 DELETE entries 占比高（消除无效记录、缩小元数据）。合并时若一对 ADD / DELETE 都在合并范围内可直接消除。`full-compaction-threshold-size` 控制何时做全量压缩（把所有 manifest 重整为目标大小的大文件）。合并是写入路径的一部分——所以频繁小提交会反复触发合并、拖慢写入，应增大批量。

**完整写入流程（在 tryCommitOnce 构建 Snapshot 阶段）：**

```
1. collectChanges: CommitMessage[] → 分桶 appendTableFiles / compactTableFiles /
   appendChangelog / compactChangelog / appendIndexFiles / compactIndexFiles
2. 合并旧 base manifest:
   mergeBefore = manifestList.readDataManifests(latestSnapshot)
   mergeAfter  = ManifestFileMerger.merge(mergeBefore, manifestFile, targetSize,
                   mergeMinCount, fullCompactionSize, partitionType, parallelism)
   baseManifestList = manifestList.write(mergeAfter)
3. 写增量 delta:  deltaManifestList = manifestList.write(manifestFile.write(deltaFiles))
4. 写 changelog(如有): changelogManifestList = manifestList.write(manifestFile.write(changelogFiles))
5. 写 index manifest(如有): indexManifest = indexManifestFile.writeIndexFiles(oldIndexManifest, indexFiles)
```

注意第 2 步：**每次提交都重写 base manifest list**（合并后），但 ManifestFileMerger 在文件数 / DELETE 占比未达阈值时会尽量复用旧 ManifestFile（只新建必要的），所以"重写 list"不等于"重写所有 manifest 内容"——list 本身很小（只含 ManifestFileMeta）。这是 Paimon 在"提交轻量"与"读取不碎片化"之间的平衡点。

**索引写入的增量复用（第 5 步）**：`indexManifestFile.writeIndexFiles(oldIndexManifest, indexFiles)` 不是从头重写，而是读旧 index manifest + 叠加本次新增 / 删除的索引文件条目，写出新 index manifest。这与 data manifest 合并思路一致——增量更新、复用旧条目，避免大表每次提交重写全部索引元数据。DV 表频繁删除（每次删都产新 DV 文件）时这个增量复用尤其重要，否则 index manifest 会和数据一样膨胀。至此五步（分桶 → 合并 base → 写 delta → 写 changelog → 写 index）全部串起：每步都是"增量产出 + 尽量复用旧文件"，最终汇成一个 Snapshot 引用的几个 list 文件名——这就是把"一次提交"压到几 KB 元数据写入的工程实现。

**三级结构的伸缩性算账**（为何非要三级）：设一张表 100 万数据文件，每个 DataFileMeta 约 200 字节。

- **若两级（Snapshot 直接列文件）**：每个 snapshot 要存 100 万 × 200B ≈ 200MB，每次提交（哪怕只加 10 个文件）都要重写这 200MB——完全不可行。
- **三级**：Snapshot 只存 3 个 list 文件名（几十字节）；ManifestList 存 N 个 ManifestFileMeta（每个约 100 字节，N 通常几十到几百）；ManifestFile 才存 DataFileMeta。新增 10 个文件只需写一个含 10 条 entry 的新 ManifestFile（2KB）+ 一个新 ManifestList（几 KB）+ 一个新 Snapshot（几百字节），其余 ManifestFile 原样复用。提交成本从 200MB 降到几 KB——这就是三级结构的全部意义。

读取时再靠 ManifestFileMeta 的统计（7.2）剪枝：分区谓词命中少数 ManifestFile 就只读那几个，避免把 100 万文件的元数据全读一遍。**伸缩性（增量写）+ 剪枝（按需读）**是三级结构的两大收益，缺一不可。

**与 LSM 的协同**：ManifestFileMeta 的 `minLevel/maxLevel` 让读取能按 LSM level 组织——主键表 merge-on-read 时要把同 key 的多个版本（分散在不同 level）合并，先按 level 剪枝拿到相关 ManifestFile、再读出文件做归并。`minBucket/maxBucket` 则支持"只读某 bucket"（点查或按 bucket 并行）。这两个维度的剪枝是 Paimon LSM 主键表点查（配合 LookupLevels）和并行扫描的元数据基础——没有它们，每次点查都要扫全表 manifest。这也回扣 8.4 的 key-range 冲突检测：写时保证同 level 同 bucket 无 key 重叠，读时才能按 level/bucket 高效定位。元数据剪枝（读）与冲突检测（写）共同维护 LSM 的有序结构。

**Manifest 性能调优速查**：

| 症状 | 配置 | 调整方向 | 原因 |
|---|---|---|---|
| 查询读大量小 manifest 慢（尤其对象存储） | `manifest.target-file-size` | 调大（如 16~32MB） | 减少 ManifestFile 个数，合并成大文件少 IO |
| ManifestFile 长期碎片化不合并 | `manifest.merge-min-count` | 调小（如 10） | 更早触发合并 |
| 写入因频繁全量合并变慢 | `manifest.full-compaction-threshold-size` | 调大 | 推迟全量压缩频率 |
| 元数据膨胀（DELETE entry 多） | 提交时自动合并 | 增大批量提交 | 合并消除 ADD/DELETE 对 |
| Catalog 侧重复读 manifest | `cache.manifest-small-file-memory` | 调大 | SegmentsCache 容纳更多小文件 |

调优总原则：**对象存储偏向"大文件少 IO"（调大 target-file-size + manifest 缓存），HDFS 容忍更多小文件**。批量提交（减少提交频次）几乎总是正收益——它同时减轻 manifest 合并、冲突检测、快照膨胀三方面压力。

---

## 八、Commit 乐观锁全流程

### ① 要解决什么问题

多写并发安全、精确一次去重、提交原子性、主键表 key-range 冲突检测、APPEND 与 COMPACT 语义分离。

### ② 设计原理与取舍（最重要）

- **乐观锁 + 重试退避**：读 latest → 算新 snapshot → 原子写 `snapshot-{id+1}`，撞车则随机退避重试，受 `commit.max-retries`（10）与 `commit.timeout` 双限。无锁服务依赖，低冲突场景一次成功。
- **APPEND / COMPACT 分两次提交**：语义清晰（新数据 vs 物理优化）、流式消费可只吃 APPEND、错误恢复策略不同（APPEND 可回滚、COMPACT 不回滚）。代价是一次 commit 可能产两个 snapshot。
- **增量冲突检测**：重试时不重读全部 base，只增量补 `[上次latest, 现latest]` 的变更再 merge——大幅降低高冲突下的重试 IO。
- **多层冲突检测**：桶数一致性、delta 内部一致性、删除存在性、key-range 重叠、Row ID 范围——比 Iceberg / Delta 的分区 / 文件级更细。
- **取舍**：低冲突无锁高吞吐；高冲突反复重试（极端下需关写侧 compaction + 独立 compaction 作业，见官方 concurrency-control 文档）。

### ③ 关键源码

`FileStoreCommitImpl` 类注释明确警告：**任何 commit 期间的异常绝不能吞，必须抛出以重启作业**；建议把 `FileStoreCommitTest` 跑上千次验证修改正确性（`FileStoreCommitImpl.java:124-126`）。核心协作者：`snapshotCommit`（原子提交，filesystem 走原子写、REST 走服务端）、`conflictDetection`、`scanner`（CommitScanner，增量读）、`rollback`（CommitRollback，可空）、`retryWaiter`、`strictModeChecker`（可空）。

### ④ 风险/陷阱/边界

- 一次 `commit()` 产生 0~2 个 snapshot（无变更且 `ignoreEmptyCommit` 则 0）。
- 冲突检测仅针对变更分区，不同分区并发不冲突。
- APPEND 若含 DELETE / DV，经 `shouldBeOverwriteCommit` 升级为 OVERWRITE 并允许回滚（不是简单"if hasDelete||hasDV"，由该方法综合判定）。
- 对象存储须配外部锁（hive / jdbc + `lock.enabled`），否则原子写不可靠可能丢快照。

### ⑤ 收益与代价

收益：无锁高吞吐、精确一次、细粒度冲突检测、可回滚。代价：高冲突下重试放大延迟、冲突检测需读变更分区元数据、双快照增加元数据量。

### 8.1 commit() 与 APPEND/COMPACT 分离

> 源码：`paimon-core/.../operation/FileStoreCommitImpl.java:290-380`

```java
public int commit(ManifestCommittable committable, boolean checkAppendFiles) {
    int generatedSnapshot = 0, attempts = 0;
    ManifestEntryChanges changes = collectChanges(committable.fileCommittables());
    List<SimpleFileEntry> appendSimpleEntries = SimpleFileEntry.from(changes.appendTableFiles);
    if (!ignoreEmptyCommit || !changes.appendTableFiles.isEmpty()
            || !changes.appendChangelog.isEmpty() || !changes.appendIndexFiles.isEmpty()) {
        CommitKind commitKind = CommitKind.APPEND;
        if (appendCommitCheckConflict) checkAppendFiles = true;
        boolean allowRollback = false;
        if (conflictDetection.shouldBeOverwriteCommit(appendSimpleEntries, changes.appendIndexFiles)) {
            commitKind = CommitKind.OVERWRITE; checkAppendFiles = true; allowRollback = true;  // 升级
        }
        if (conflictDetection.hasRowIdCheckFromSnapshot()) { checkAppendFiles = true; allowRollback = true; }
        attempts += tryCommit(CommitChangesProvider.provider(changes.appendTableFiles,
                changes.appendChangelog, changes.appendIndexFiles),
                committable.identifier(), committable.watermark(), committable.properties(),
                commitKind, allowRollback, checkAppendFiles, null);
        generatedSnapshot += 1;
    }
    if (!changes.compactTableFiles.isEmpty() || !changes.compactChangelog.isEmpty()
            || !changes.compactIndexFiles.isEmpty()) {
        attempts += tryCommit(..., CommitKind.COMPACT, false /*不回滚*/, true /*强制检冲突*/, null);
        generatedSnapshot += 1;
    }
    return generatedSnapshot;
}
```

`collectChanges` 把所有 `CommitMessage` 按 append / compact × table / changelog / index 六类分桶。APPEND 在前、COMPACT 在后。**为何分离**：(1) 语义清晰，流式消费者可只消费 APPEND；(2) 冲突隔离，APPEND 可回滚而 COMPACT 不可，分离后各自处理错误恢复；(3) APPEND→OVERWRITE 升级逻辑不应波及 COMPACT。

**一次 commit 的 snapshot 产出矩阵**（理解 `commit()` 返回值）：

| 变更内容 | 产出 snapshot | 返回值 |
|---|---|---|
| 只有新数据 | 1 个 APPEND | 1 |
| 新数据含 DELETE / DV | 1 个 OVERWRITE（APPEND 升级） | 1 |
| 只有压缩 | 1 个 COMPACT | 1 |
| 新数据 + 压缩 | APPEND(或 OVERWRITE) + COMPACT，先后两个 | 2 |
| 无变更且 `ignoreEmptyCommit=true` | 无 | 0 |
| 无变更且 `ignoreEmptyCommit=false` | 1 个空 APPEND | 1 |

`ignoreEmptyCommit` 默认 true（流式场景空 checkpoint 不该产快照），但某些场景（如需要 watermark 推进的空提交）会设 false。**误以为"每次 commit 必产 1 个 snapshot"是常见 bug 源**——下游若按"提交次数 = 快照数"假设做位点管理会错位。`appendCommitCheckConflict` 字段控制 APPEND 是否强制检冲突（默认 false，APPEND 通常无冲突），仅在特殊表配置下置 true。

### 8.2 tryCommit 重试退避

> 源码：`FileStoreCommitImpl.java:692-741`

```java
private int tryCommit(CommitChangesProvider changesProvider, long identifier, @Nullable Long watermark,
        Map<String,String> properties, CommitKind commitKind, boolean allowRollback,
        boolean detectConflicts, @Nullable String statsFileName) {
    int retryCount = 0; RetryCommitResult retryResult = null;
    long startMillis = System.currentTimeMillis();
    while (true) {
        Snapshot latestSnapshot = snapshotManager.latestSnapshot();
        CommitChanges changes = changesProvider.provide(latestSnapshot);   // OVERWRITE 随 latest 变化
        CommitResult result = tryCommitOnce(retryResult, changes.tableFiles, changes.changelogFiles,
                changes.indexFiles, identifier, watermark, properties, commitKind,
                allowRollback, latestSnapshot, detectConflicts, statsFileName);
        if (result.isSuccess()) break;
        retryResult = (RetryCommitResult) result;
        if (System.currentTimeMillis() - startMillis > options.commitTimeout()
                || retryCount >= options.commitMaxRetries()) {
            throw new RuntimeException(String.format(
                "Commit failed after %s millis with %s retries, there maybe exist commit conflicts "
                + "between multiple jobs.", options.commitTimeout(), retryCount), retryResult.exception);
        }
        retryWaiter.retryWait(retryCount);   // 随机退避 [commit.min-retry-wait, commit.max-retry-wait]
        retryCount++;
    }
    return retryCount + 1;
}
```

`RetryWaiter` 由 `(commit.min-retry-wait=10ms, commit.max-retry-wait=10s)` 构造，用随机退避避免多写者惊群。**为何 `while(true)` 而非定次循环**：超时与重试次数是两个独立闸门——流式场景可能需较久持续重试（等其它 compaction 落地），但仍受 `commit.timeout` 上限保护；`changesProvider.provide(latestSnapshot)` 每轮重算 changes 是因为 OVERWRITE 的 DELETE 集合依赖当前 latest（要删的"已有文件"随 latest 变化）。

**返回值 `retryCount + 1` 的含义**：返回总尝试次数（首次 + 重试次数），上报给 `CommitMetrics.attempts`。监控这个指标的 P99 能直接反映冲突激烈程度——长期 > 1 说明并发写撞车频繁，该考虑 write-only + 独立 compaction。`commit.timeout` 与 `commit.max-retries` 谁先触发就抛异常：超时优先用于"重试不贵但就是一直撞"的场景兜底，max-retries 用于"每次重试都很贵（读大量 base）"的场景限次。两者配合避免无限重试拖垮作业。

### 8.3 tryCommitOnce 去重与增量冲突检测

> 源码：`FileStoreCommitImpl.java`（`tryCommitOnce`，约 760~1050 行）

流程：

```
[1] 去重：重试且 latestSnapshot 非空时，扫 [上次id+1, latest.id] 找
        (commitUser, commitIdentifier, commitKind) 匹配的快照 → 命中即 SuccessCommitResult
[2] newSnapshotId = latest.id + 1 (无历史则 1)
[3] strictModeChecker 检查(可选)
[4] 冲突检测(detectConflicts):
      读变更分区 base 文件(重试走增量优化) → discardDuplicate(可选)
      → conflictDetection.checkConflicts(...)
      冲突 + 允许回滚 → rollback.tryToRollback(latest) → RetryCommitResult
      冲突 + 不许回滚 → throw
[5] 构建新 Snapshot:
      合并旧 base → 写 baseManifestList; 写 deltaManifestList;
      写 changelog/index manifest; watermark 取 max; Row Tracking 行 id 分配;
      算 total/deltaRecordCount; 构造 Snapshot
[6] commitPreCallbacks
[7] 原子提交: commitSnapshotImpl → snapshotCommit.commit(...) (filesystem 原子写 / REST 服务端)
[8] 成功 → commitCallbacks + SuccessCommitResult; 原子写返回 false/异常 → RetryCommitResult
```

去重（步骤 1）是精确一次兜底：网络超时后客户端不确定上次是否成功，重试时先查是否已持久化，判据是三元组匹配。增量冲突检测（步骤 4，重试时不重读全部 base）：

```java
if (commitFailRetry != null && commitFailRetry.baseDataFiles != null) {
    baseDataFiles = new ArrayList<>(commitFailRetry.baseDataFiles);
    List<SimpleFileEntry> incremental = scanner.readIncrementalChanges(
        commitFailRetry.latestSnapshot, latestSnapshot, changedPartitions);   // 只读增量
    baseDataFiles.addAll(incremental);
    baseDataFiles = new ArrayList<>(FileEntry.mergeEntries(baseDataFiles));
}
```

**Flink 精确一次的端到端配合**：Flink Sink 用 `commitUser = job uid`（跨重启稳定）、`commitIdentifier = checkpoint id`（单调递增）。checkpoint barrier 对齐后 Sink 把本 checkpoint 的 CommitMessage 攒成 `ManifestCommittable` 提交。若 checkpoint 100 提交后 JM 失败、作业从 checkpoint 99 恢复并重发 checkpoint 100 的提交，去重逻辑（步骤 1）扫到已有 `(job_uid, 100, APPEND)` 快照即返回成功、不重复写——这就是 Paimon 在 Flink 下无重复数据的根本保证。`latestSnapshotOfUser`（5.4）则用于恢复时确定"我提交到哪了"。

**两阶段提交的 Paimon 映射**：Flink 的两阶段提交 Sink 中，"prepare"阶段把数据写成文件（产 CommitMessage，但还没进 snapshot——对读者不可见）、"commit"阶段（checkpoint 完成时）才 `FileStoreCommitImpl.commit` 把它们提进 snapshot。即使 commit 阶段重复触发（恢复重放），去重保证幂等。这把"分布式两阶段提交"压缩成"写文件（可重做）+ 原子写 snapshot（幂等去重）"两步——没有真正的分布式事务协调者，靠"snapshot 原子写 + 三元组去重"达成等效的精确一次。这是理解 Paimon 为何不需要外部事务协调即可精确一次的关键。

**回滚（CommitRollback）与严格模式（StrictModeChecker）**：

- `rollback`（可空）：仅当 `allowRollback=true`（OVERWRITE 或 Row ID 检查场景）时启用。检测到冲突时 `rollback.tryToRollback(latestSnapshot)` 尝试删除本次已写但未提交成功的文件，返回 `RetryCommitResult` 触发重试——避免重试残留垃圾文件。APPEND / COMPACT 普通提交 `allowRollback=false`，冲突直接抛异常由作业 failover 处理（更简单、由框架兜底清理）。
- `strictModeChecker`（可空，由 `commit.strict-mode.last-safe-snapshot` 等配置启用）：在提交前做更严格的安全检查，用于对一致性要求极高的场景，发现风险即拒绝提交。

`FileStoreCommitImpl` 的 `commitCleaner` 在异常路径负责清理半成品文件，与类注释"任何异常必须抛出"配合——异常抛出后由上层 failover，cleaner 尽力清理已写的孤儿文件，但**真正的安全保证来自"未写成功的 snapshot 文件不会被任何人读到"**（snapshot 是最后一步原子写）。

**watermark 合并的具体规则**：构建新 Snapshot 时（步骤 5），新 watermark = max(本次提交的 watermark, latestSnapshot 的 watermark)。这保证 watermark 单调不减——即使本次提交携带的 watermark 比上一个快照小（如某 subtask 进度落后），也取历史最大值。**为何必须单调**：下游流式读用 watermark 做事件时间推进、触发窗口计算；若 watermark 回退，已触发的窗口会被认为"还能来数据"，破坏 exactly-once 与窗口正确性。回滚到旧快照也不会让后续提交的 watermark 回退，正是这个 max 规则的体现——这是"watermark 只增不减"在源码里的落点。

**`commit.strict-mode.last-safe-snapshot`**：严格模式下 `StrictModeChecker` 会校验提交相对某个"安全快照"的关系，用于对一致性极敏感的场景额外加固（牺牲一点吞吐换更强保证）。默认关闭，普通场景靠乐观锁 + 冲突检测已足够。

### 8.4 ConflictDetection 多层检查

> 源码：`paimon-core/.../operation/commit/ConflictDetection.java`（独立类，778 行）

`checkConflicts(latestSnapshot, baseEntries, deltaEntries, deltaIndexEntries, commitKind)` 逐层：

1. **DV enrichment**：DV 开启且 `BUCKET_UNAWARE` 时，给 base / delta entry 附 DV 信息（`SimpleFileEntryWithDV`），让后续检查能感知删除向量。
2. **桶数一致性** `checkBucketKeepSame`：同分区所有文件 `totalBuckets` 必须一致。**OVERWRITE 跳过此检查**——覆写整分区，桶数可随表配置变更而变（如改 bucket 数后重写），合法。
3. **delta 内部一致性**：`mergeEntries(deltaEntries)` 不允许对同一文件同时 ADD+DELETE。
4. **base+delta 合并 + 删除存在性** `checkDeleteInEntries`：要删的文件必须在 base 中存在，否则说明被别的作业删过（files conflict）→ 合并后残留 DELETE 即报冲突。
5. **Key Range 重叠** `checkKeyRange`：主键表 delta 文件的 [minKey, maxKey] 不能与 base 同 bucket 同 level 文件重叠——否则同 key 落多文件破坏 LSM 正确性。用 `RangeHelper` / `Range` 做区间重叠判定。
6. **Row ID 范围冲突** `checkRowIdRangeConflicts` 与 **从特定 snapshot 起的 Row ID 检查** `checkForRowIdFromSnapshot`（Row Tracking）。

`checkConflicts` 返回 `Optional<RuntimeException>` 而非直接抛——让调用方（`tryCommitOnce`）决定"冲突 + 允许回滚则触发 rollback 重试，否则抛出"。这个返回值设计把"检测"与"处置"解耦：同一套检测逻辑，APPEND/COMPACT 直接抛、OVERWRITE/Row-ID 场景走回滚重试。

Key Range 检查是 Paimon 比 Iceberg / Delta 更细的冲突粒度（后者多止于分区 / 文件级）——对 LSM 主键表的正确性至关重要。`shouldBeOverwriteCommit` 与 `hasRowIdCheckFromSnapshot` 也定义在此类，供 `commit()` 决定是否升级 OVERWRITE / 强制检冲突。

**逐层检查的"快速失败"顺序设计**：检查从廉价到昂贵排列——桶数一致性（只比整数）→ delta 内部合并（小集合）→ base+delta 合并 + 删除存在性（中等）→ key range 重叠（需区间排序比较，最贵）。任一层失败立即返回 `Optional<RuntimeException>`，避免做完后面昂贵检查才发现前面就该失败。这种"廉价检查前置"是冲突检测在高并发下仍可接受的关键。

**为何 OVERWRITE 跳过桶数一致性而仍做其它检查**：OVERWRITE 替换整分区，新数据桶数可与旧不同（合法）；但它仍不能违反"要删的文件必须存在"（否则与别的覆写冲突）和 key range 正确性。所以是"跳过特定一层"而非"跳过全部"——精确放宽而非粗放。

**DV（Deletion Vector）场景的特殊处理**：DV 开启 + `BUCKET_UNAWARE`（append-only 表的删除）时，删除不是删整个数据文件，而是给文件附一个删除位图。冲突检测必须感知"两个作业是否对同一文件的重叠行打了删除标记"，故先做 enrichment 把 DV 信息附到 entry 上（`SimpleFileEntryWithDV`），再让后续检查能正确判定行级冲突——这是 merge-on-read 表并发删除安全的基础。

**冲突检测的具体冲突例子**（主键表 key range 重叠）：

```
base:  bucket-0, level-0, file-X  keyRange=[1, 100]
作业A: bucket-0, level-0, 新增 file-Y keyRange=[50, 150]   # 与 X 重叠!
       → checkKeyRange 发现 [50,150] ∩ [1,100] ≠ ∅ 且同 bucket 同 level
       → 报冲突: 同一 key(如 60) 会同时存在于 X 和 Y, 违反主键唯一性/LSM 有序性
```

为何这对 LSM 致命：LSM 读取假设同一 level 的 SortedRun 之间 key 不重叠（可二分定位），重叠会让点查 / 合并读到重复或错乱的 key。Iceberg / Delta 无 LSM 结构、主键去重靠 merge-on-read delete file，不需要这层检查——这是 Paimon 为主键表 LSM 正确性付出的额外冲突检测成本，也是它能做高效点查（LookupLevels）的前提。`checkBucketKeepSame` 的例子：分区 dt=x 下 base 文件 totalBuckets=4，若 delta 文件 totalBuckets=8（表 rescale 过但未 OVERWRITE）→ 报冲突，因为同分区桶数必须一致才能正确路由 key 到桶。

### 8.5 Overwrite 分区覆写

> 源码：`FileStoreCommitImpl.java`（`overwrite`，约 435-543 行）

```
overwrite(partition, committable, properties)
  → collectChanges
  → partitionFilter:
        动态覆写(dynamicPartitionOverwrite): 从写入文件提取实际分区
        静态覆写: 按指定 partition map 构 PartitionPredicate + 校验变更文件都属该分区
  → tryUpgrade(overwriteUpgrade): 覆写文件无 key 重叠则升到最高 level
  → tryOverwritePartition: scanner.readOverwriteChanges 读匹配分区已有文件生成 DELETE,
        合并 DELETE + 新 ADD → tryCommit(..., CommitKind.OVERWRITE, ...)
  → 有 compact 变更再提一次 COMPACT
```

静态覆写：变更文件必须全属指定分区，否则抛异常（防止 `INSERT OVERWRITE PARTITION(dt='x')` 误写到别的分区）。动态覆写：无写入则跳过（不删任何文件）——这点常被误解，动态覆写空数据不会清空分区。

**静态 vs 动态覆写的实战差异**：

```sql
-- 静态: 明确指定分区, 该分区被整体替换(即使新数据为空也清空该分区)
INSERT OVERWRITE t PARTITION (dt='2024-01-01') SELECT ... ;

-- 动态(dynamic-partition-overwrite=true, 默认): 只覆写实际写入数据涉及的分区
INSERT OVERWRITE t SELECT ... ;  -- 写入数据落在 dt in (01-01, 01-02) 则只覆这两个分区,
                                 -- 其它分区不动; 若 SELECT 结果为空则什么都不删
```

源码上 `tryOverwritePartition` 先用 `scanner.readOverwriteChanges(partitionFilter)` 读匹配分区的所有现存文件生成 DELETE entries，再合并本次的 ADD entries，整体作为一次 OVERWRITE 提交。这就是为什么覆写是原子的——DELETE 旧 + ADD 新在同一个 snapshot 里生效，不存在"删完旧的、还没写新的"中间态。`tryUpgrade`（`overwriteUpgrade=true`）会在覆写文件无 key 重叠时把它们提升到最高 LSM level，省去后续 compaction——这是覆写场景的一个小优化。**误删风险**：静态覆写空 SELECT 会清空指定分区（符合 SQL OVERWRITE 语义但易误操作），动态覆写空 SELECT 则安全（什么都不删），生产中要分清用哪个。

### 8.6 提交排障速查

| 现象 | 可能根因 | 排查 / 处置 |
|---|---|---|
| `Commit failed after ... retries` | 多作业高并发写同分区，乐观锁反复撞车 | 关写侧 compaction（`write-only=true`）+ 独立 compaction 作业；调大 `commit.max-retries` / `commit.timeout` |
| 对象存储偶发丢快照 | 无原子 rename 且未配锁 | hive / jdbc metastore + `lock.enabled=true` |
| 长查询读取报快照不存在 | `snapshot.time-retained` 过短 | 调大保留时间，配 `num-retained.min` |
| files conflict 反复重启 | 两流式作业同写、compaction 删文件冲突 | 同第一行：分离 compaction |
| 一次写入产生 2 个 snapshot | APPEND + COMPACT 分离提交 | 正常现象，非 bug |
| 提交极慢 | 变更分区数据文件多，冲突检测读全量 base | 增大批量、减少分区扇出；关注增量检测是否生效 |

**官方推荐的根治高冲突方案（concurrency-control 文档核心结论）**：冲突的本质是"删文件"，而删文件源于 compaction。两个流式作业同时写同一张表、各自做 compaction，会互相删对方的文件 → files conflict → 不断 failover 重启。根治办法是**关掉写作业的 compaction**（表配置 `write-only=true`），另起一个独立的专用 compaction 作业（dedicated compaction job）统一做合并。这样写作业只 APPEND 不删文件、不产生 files conflict，compaction 由单一作业串行执行无并发删除——这是 Paimon 多写场景的标准部署模式。

**snapshot conflict 与 files conflict 的本质区别**（排障时先分清是哪种）：

- **snapshot conflict**：快照 ID 被抢（别人先写了 `snapshot-{id+1}`）——`tryCommit` 重新取 latest 重算即可恢复，对用户透明，是"正常的并发竞争"。
- **files conflict**：要删的文件已被别人删了（`checkDeleteInEntries` 报合并后残留 DELETE）——这是"语义冲突"，无法靠重试解决（重试还是删不到），流式作业故意 failover 从最新状态重来，仍冲突则反复重启。看到作业反复 restart 且日志含 conflict，基本是 files conflict，按上面的 write-only 方案处置。

**官方文档的核心论断再强调**：concurrency-control 文档明确"冲突的本质在于删文件，而删文件源于 compaction"。所以单写作业（无并发）永远不会有 files conflict；只有多个作业同时写同一表且各自做 compaction 才会。这把排查方向收窄到"是不是多作业并发写 + 都开了 compaction"——是则 write-only + 独立 compaction，否则查其它（如对象存储锁配置、时钟、超时配置）。Paimon 保证此处不丢不重，但反复重启不是好状态，需主动消除并发删除源。

---

## 九、系统表与 CoreOptions

**系统表**（`table/system/`，经 `SystemTableLoader` 按 `$名` 加载，只读、不可 DDL）：`$snapshots`（快照链：id / schemaId / commitKind / commitUser / 记录数 / watermark）、`$schemas`（schema 历史）、`$manifests`（当前快照的 ManifestFileMeta）、`$files`（当前数据文件 + 统计）、`$options`、`$partitions`、`$buckets`、`$tags`、`$branches`、`$consumers`（流式消费位点）、`$audit_log`（含 rowkind 的变更审计）等。它们是把上述元数据文件投影成行的"虚拟视图"，是排障第一入口——例如查 `$snapshots` 看 commitKind / commitUser 定位是谁在反复提交、查 `$files` 看文件分布是否倾斜。

**系统表排障实战 SQL**（与本文各章对应）：

```sql
-- 谁在反复提交、提交类型分布(对应第八章)
SELECT commit_user, commit_kind, count(*) FROM my_db.t$snapshots GROUP BY 1,2;
-- schema 演进历史(对应第四章)
SELECT schema_id, fields, partition_keys, primary_keys FROM my_db.t$schemas;
-- manifest 是否碎片化(对应第七章): 看行数与文件大小
SELECT count(*), avg(file_size), sum(num_added_files), sum(num_deleted_files)
  FROM my_db.t$manifests;
-- 数据是否倾斜: 各分区/桶文件数与大小
SELECT partition, bucket, count(*), sum(file_size_in_bytes) FROM my_db.t$files GROUP BY 1,2;
-- 流式消费位点(排查消费滞后)
SELECT * FROM my_db.t$consumers;
-- tag/branch 现状
SELECT * FROM my_db.t$tags;  SELECT * FROM my_db.t$branches;
```

系统表本身不缓存（CachingCatalog 缓存原表后动态包装，见 2.5），所以查 `$snapshots` 总是反映最新文件系统状态——适合做实时排障。

**CoreOptions / CatalogOptions 关键项**（`paimon-api/.../CoreOptions.java`，通过 `CoreOptions.fromMap` 经类型安全的 `Options` 读取）：

| 类别 | 配置 | 默认 | 说明 |
|---|---|---|---|
| Commit | commit.max-retries | 10 | 乐观锁最大重试 |
| | commit.timeout | - | 提交超时（与重试次数共同限流） |
| | commit.min / max-retry-wait | 10ms / 10s | 随机退避区间 |
| | commit.discard-duplicate-files | false | 丢弃重复文件 |
| Manifest | manifest.target-file-size | 8MB | ManifestFile 目标大小 |
| | manifest.merge-min-count | 30 | 触发合并的最小文件数 |
| | manifest.full-compaction-threshold-size | 16MB | 全量压缩阈值 |
| Snapshot/Tag | snapshot.time-retained | 1h | 快照保留时长 |
| | snapshot.num-retained.min / max | 10 / ∞ | 保留数量上下限 |
| | tag.automatic-creation / creation-period / default-time-retained | none / - / - | 自动 Tag |
| Schema | bucket | -1 | 桶数（-1=动态） |
| | bucket-key / sequence.field | - | 桶键（空则用主键） / 序列字段 |
| | partition.default-name | `__DEFAULT_PARTITION__` | 默认分区名 |
| Cache | cache.expire-after-access / write | 10min / 30min | Catalog 缓存过期 |
| | cache.manifest-small-file-memory / threshold | 128MB / 1MB | 小 manifest 缓存 |
| | cache.snapshot.max-num-per-table | 20 | 每表快照缓存数 |
| | cache.partition-max-num | 0 | 分区缓存（0=关） |
| | cache.deletion-vectors.max-num | 100000 | DV 元数据缓存数 |
| REST | data-token.enabled | false | 按表 data token（PVFS / 直连数据） |
| | token.provider / token | - | bear / dlf；bear 的静态 token |
| | dlf.access-key-id/secret、dlf.token-path、dlf.token-loader | - | DLF 凭证来源 |

配置项通过 `CoreOptions.fromMap(options)` 工厂创建，内部用类型安全的 `Options` + `ConfigOption` 读取（每个配置有 key / 类型 / 默认值 / 描述），避免散落的字符串硬编码。

**缓存配置调优要点**：

- `cache.expire-after-access` / `expire-after-write`：单进程独占 warehouse 可放宽（减少重读）；多进程共享必须收紧 write 过期（一致性窗口）。
- `cache.snapshot.max-num-per-table`（20）：流式频繁取快照的表可调大，减少重复读 snapshot 文件。
- `cache.manifest-small-file-memory`（128MB）/ `manifest-max-memory`：manifest 多的大表调大可显著减少 manifest 重读 IO；内存紧张则调小。
- `cache.partition-max-num`（0=关）：分区数可控（几千以内）且频繁列分区时可开启；分区爆炸（百万级）保持关闭避免 OOM。
- `cache.deletion-vectors.max-num`（10万）：merge-on-read + DV 表调大可加速 DV 查找。

**一条总规律**：缓存换内存换一致性。读多写少、单进程、表元数据稳定 → 放宽缓存；多进程共享、元数据频繁变 → 收紧或显式失效。REST 模式下还有服务端可驱动失效，一致性更可控。

**配置读取的类型安全保障**：`ConfigOption` 把每个配置的 key / 类型 / 默认值 / 校验绑在一起（如 `commit.max-retries` 是 `intType().defaultValue(10)`），`Options.get(CONFIG_OPTION)` 直接返回正确类型并应用默认值——避免了"到处 `Integer.parseInt(map.get("..."))` 且各处默认值不一致"的散乱。改一个默认值只动 `CoreOptions` 里的 `ConfigOption` 定义，全局生效。这是 Paimon 配置体系的工程基础，也是为什么本文各处给的默认值能信赖——它们就是源码里 ConfigOption 的 defaultValue。

---

## 十、与 Iceberg 的设计对比

### 10.1 Snapshot 与元数据

| 维度 | Paimon | Iceberg |
|---|---|---|
| 快照格式 | JSON `snapshot-{id}` | Avro `snap-{id}.avro` |
| Manifest 层级 | 三级 | 三级 |
| 增量追踪 | base/delta/changelog 三分 | manifest 带 `added_snapshot_id` |
| Changelog | 原生 changelogManifestList | 无内建，靠 equality/position delete 间接 |
| Watermark | 原生字段 | 无 |
| 统计 | 独立 statistics 文件引用 | 内嵌 manifest |
| 精确一次 | commitUser + commitIdentifier | sequence number |

**Paimon 的 delta 优势**：流式增量只读 delta manifest，无需 diff 全量；Iceberg 需按 `added_snapshot_id` 过滤或比对两版 manifest list。**Paimon 的 changelog 优势**：写入时直接产 CDC 变更记录，语义比 Iceberg 的 delta file 间接读更直接。

### 10.2 Catalog 与 Schema

| 维度 | Paimon | Iceberg |
|---|---|---|
| 默认 Catalog | FileSystemCatalog | HadoopCatalog / RESTCatalog |
| SPI | CatalogFactory（Java SPI） | CatalogUtil + catalog-impl |
| 缓存 | CachingCatalog（Caffeine 装饰器） | CachingCatalog（类似） |
| 锁 | CatalogLockFactory SPI（可选） | LockManager（内置） |
| Schema 存储 | 独立 schema-{id}，每变更新增文件 | 嵌入 metadata.json，每变更重写整块 |
| 分支 / Tag | 原生 BranchManager / TagManager（FS 级） | Ref（1.5+），Catalog 级 |

**Paimon 独立 schema 文件的好处**：变更轻量、乐观锁基于文件原子创建（无需 RMW）、历史天然可追溯。**Iceberg 合并 metadata 的好处**：单文件读取一次到位、信息完整（schema + partition spec + sort order）。

### 10.3 并发控制

| 维度 | Paimon | Iceberg |
|---|---|---|
| 写并发 | 乐观锁（原子文件 / 服务端仲裁）+ 可选 CatalogLock | 乐观锁（CAS on metadata.json）+ 可选 LockManager |
| 冲突粒度 | 分区 + 桶 + Key Range + Row ID | 分区 + 文件 |
| 重试 | 随机退避 + 超时 | 固定次数 |
| Exactly-Once | commitUser + commitIdentifier 去重 | sequence number |

**Paimon 的 Key Range 冲突检测更精细**：检查新文件 key range 是否与现有文件重叠，对 LSM 主键表正确性至关重要（同 bucket 同 level 不应有重叠文件）；Iceberg 主要关注分区 / 文件级。

**REST Catalog 的对位**：Iceberg 的 REST Catalog 规范较成熟、生态广（多引擎已支持）；Paimon REST Catalog 是后起但定位更"重"——不仅做元数据控制面，还管 data token 下发与 PVFS 路径虚拟化，把存储鉴权也收编。两者都把"控制面集中化、客户端轻量化"作为方向，差异在 Paimon 把流式语义（changelog / watermark / 精确一次）与存储凭证治理一起做进了协议。

**小结**：Paimon 更侧重流式（原生 changelog / watermark / delta 增量读）、更轻量（独立 schema 文件、文件即元数据、可选 REST）；Iceberg 在事务语义、多引擎兼容、SQL 标准积累更深。两者都在演进——Paimon 加 REST Catalog，Iceberg 增强流式，差异在收敛。

**选型建议落点**：纯批 / 多引擎互操作（Trino + Spark + Flink 同读一表）成熟度优先 → Iceberg 当前更稳；实时入湖 + CDC + 流批一体 + 主键更新 → Paimon 的 LSM + changelog + key-range 冲突检测是结构性优势；云上多租户集中治理 → 两者 REST Catalog 都可选，看现有生态。元数据运维成本上 Paimon filesystem 后端最低（零外部依赖、可手工修复），但生产高并发仍建议 hive/jdbc/rest 配锁或服务端仲裁。

---

## 十一、设计决策总结

| 决策点 | 选择 | 取舍 / 代价 | 替代方案 |
|---|---|---|---|
| 元数据存储 | 文件即元数据（独立 JSON / 二进制文件） | 读路径多次 IO；文件单调累积 | 内嵌单一 metadata（Iceberg / Delta），需 RMW 重写整块 |
| 并发控制 | 乐观锁（原子文件创建 / 服务端仲裁）+ 重试退避 | 高冲突反复重试；对象存储需外部锁 | 悲观分布式锁，性能稳但重依赖 |
| Catalog 扩展 | 模板方法(AbstractCatalog) + 装饰器(Delegate) + SPI | 一层间接 + 能力矩阵心智负担 | 各引擎各实现，易不一致 |
| 后端选型 | filesystem / hive / jdbc / rest 四选一 | 各有能力缺口 | 绑死单一后端 |
| Schema 演进 | 独立 schema-{id} + field id 永不回收 | schema 文件累积 | 内嵌 schema 重写 metadata |
| 类型变更 | 仅安全扩宽（除非 allowExplicitCast） | 不安全转换需 overwrite 重写 | 宽松转换，风险丢精度 |
| Snapshot | 仅元数据入口 + base/delta/changelog 三分 | 需引用计数式过期清理 | 单 manifest，增量需 diff 全量 |
| LATEST hint | 乐观 hint + id+1 验证 + list 兜底 | hint 失效退化为全量扫描 | 每次 list 目录，对象存储慢 |
| Tag / Branch | Tag=Snapshot 副本；Branch=写时复制 | 删分支需判文件独占；Tag 拖住存储 | 复制数据，慢且费空间 |
| Manifest | 三级 + Meta 多维统计剪枝 | 层级与合并复杂度 | 两级，无法支撑大表 |
| Commit 语义 | APPEND / COMPACT 分离 + 细粒度 key-range 冲突 | 双快照增量；冲突检测读元数据 | 单一提交 + 分区级冲突，粒度粗 |
| 增量冲突检测 | 重试只增量补 [上次, 现] 变更再 merge | 需缓存上次 baseDataFiles | 每次重读全量 base，高冲突慢 |
| REST 认证 | AuthProvider 抽象 + DLF v4 签名 + 1h 提前刷新 | 网络依赖 + 时钟敏感 | 仅静态长期凭证，安全性差 |
| REST 数据访问 | data token + RESTTokenFileIO（按 token 缓存 FileIO）+ PVFS | 强依赖服务端下发凭证 | 客户端持长期强凭证直连，权限难收敛 |
| 系统表 | 元数据文件的运行时投影（只读、不可 DDL） | 不可写、随原表变 | 物化成真实表，需同步维护 |
| 配置体系 | ConfigOption 类型安全 + 默认值集中 | — | 散落字符串 parse，默认值易不一致 |
