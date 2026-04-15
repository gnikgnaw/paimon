# Apache Paimon Catalog 与元数据管理深度分析

> 基于 Paimon 1.5-SNAPSHOT 源码分析，commit: 7c93bd720
> 分析日期: 2026-04-15

---

## 目录

- [一、概述](#一概述)
- [二、Catalog 接口体系](#二catalog-接口体系)
  - [2.1 类层次结构](#21-类层次结构)
  - [2.2 Catalog 接口定义](#22-catalog-接口定义)
  - [2.3 AbstractCatalog 抽象基类](#23-abstractcatalog-抽象基类)
  - [2.4 FileSystemCatalog 实现](#24-filesystemcatalog-实现)
  - [2.5 DelegateCatalog 委托模式](#25-delegatecatalog-委托模式)
  - [2.6 CachingCatalog 缓存装饰器](#26-cachingcatalog-缓存装饰器)
  - [2.7 CatalogFactory SPI 加载机制](#27-catalogfactory-spi-加载机制)
  - [2.8 Identifier 表标识体系](#28-identifier-表标识体系)
- [三、Schema 演进机制](#三schema-演进机制)
  - [3.1 TableSchema 完整字段](#31-tableschema-完整字段)
  - [3.2 SchemaManager 管理器](#32-schemamanager-管理器)
  - [3.3 SchemaChange 所有变更类型](#33-schemachange-所有变更类型)
  - [3.4 乐观锁机制与原子写入](#34-乐观锁机制与原子写入)
  - [3.5 Schema 存储路径规则](#35-schema-存储路径规则)
- [四、Snapshot 管理](#四snapshot-管理)
  - [4.1 Snapshot 完整字段表格](#41-snapshot-完整字段表格)
  - [4.2 base/delta/changelog 三分法](#42-basedeltachangelog-三分法)
  - [4.3 CommitKind 提交类型](#43-commitkind-提交类型)
  - [4.4 SnapshotManager 管理器](#44-snapshotmanager-管理器)
- [五、Tag 和 Branch 机制](#五tag-和-branch-机制)
  - [5.1 TagManager 标签管理](#51-tagmanager-标签管理)
  - [5.2 Tag TTL 机制](#52-tag-ttl-机制)
  - [5.3 BranchManager 分支管理](#53-branchmanager-分支管理)
  - [5.4 分支路径规则与文件布局](#54-分支路径规则与文件布局)
- [六、Manifest 三级结构](#六manifest-三级结构)
  - [6.1 整体架构](#61-整体架构)
  - [6.2 ManifestList 清单列表](#62-manifestlist-清单列表)
  - [6.3 ManifestFileMeta 字段详解](#63-manifestfilemeta-字段详解)
  - [6.4 ManifestFile 与 ManifestEntry](#64-manifestfile-与-manifestentry)
  - [6.5 Manifest 写入与读取流程](#65-manifest-写入与读取流程)
- [七、Commit 流程](#七commit-流程)
  - [7.1 FileStoreCommitImpl.commit() 完整流程](#71-filestorecommitimplcommit-完整流程)
  - [7.2 tryCommit 乐观锁重试机制](#72-trycommit-乐观锁重试机制)
  - [7.3 tryCommitOnce 单次提交详解](#73-trycommitonce-单次提交详解)
  - [7.4 冲突检测 ConflictDetection](#74-冲突检测-conflictdetection)
  - [7.5 APPEND/COMPACT 分离提交](#75-appendcompact-分离提交)
  - [7.6 Overwrite 分区覆写](#76-overwrite-分区覆写)
- [八、CoreOptions 配置体系](#八coreoptions-配置体系)
- [九、与 Iceberg 的设计对比](#九与-iceberg-的设计对比)

---

## 一、概述

Apache Paimon 的 Catalog 体系是整个数据湖管理的核心枢纽。它负责管理数据库、表、视图、分区、快照、标签、分支等所有元数据对象。其设计遵循以下核心原则:

1. **文件系统即元数据存储**: 默认实现 `FileSystemCatalog` 将所有元数据（Schema、Snapshot、Manifest）直接以文件形式存储在文件系统/对象存储上，不依赖外部数据库，实现零外部依赖的部署模式。
2. **乐观并发控制**: Schema 演进和 Snapshot 提交均基于文件系统的原子重命名/创建操作实现乐观锁，无需引入分布式锁服务。
3. **装饰器模式扩展**: 通过 `DelegateCatalog` → `CachingCatalog` 等装饰器链，无侵入地叠加缓存、权限等能力。
4. **SPI 可插拔**: 通过 `CatalogFactory` SPI 机制支持 FileSystem、REST、Hive 等多种 Catalog 后端。

```
+------------------------------------------------------+
|                   计算引擎层                           |
|  (Flink / Spark / Hive / REST Client)                |
+------------------------------------------------------+
              |                          |
              v                          v
+------------------------+  +--------------------------+
| PrivilegedCatalog      |  |                          |
|   (权限装饰器, 可选)    |  |   RESTCatalog            |
+------------------------+  |   (REST API 后端)        |
              |              +--------------------------+
              v
+------------------------+
| CachingCatalog         |
|   (缓存装饰器)         |
+------------------------+
              |
              v
+------------------------+     +-----------------------+
| FileSystemCatalog      |     | HiveCatalog           |
|   (文件系统后端)       |     |   (Hive Metastore)    |
+------------------------+     +-----------------------+
         |           |
         v           v
+-------------+  +-----------+
| SchemaManager|  |SnapshotMgr|
+-------------+  +-----------+
         |           |
         v           v
+------------------------------------------------------+
|                    FileIO (文件系统抽象)               |
|   (HDFS / S3 / OSS / Azure / GCS / Local)            |
+------------------------------------------------------+
```

---

## 二、Catalog 接口体系

### 2.1 类层次结构

```
Catalog (接口, AutoCloseable)
|-- paimon-core: catalog/Catalog.java
|
+-- AbstractCatalog (抽象类)
|   |-- paimon-core: catalog/AbstractCatalog.java
|   |
|   +-- FileSystemCatalog
|       |-- paimon-core: catalog/FileSystemCatalog.java
|       |-- 使用 FileIO 直接读写文件系统元数据
|
+-- DelegateCatalog (抽象委托类)
|   |-- paimon-core: catalog/DelegateCatalog.java
|   |
|   +-- CachingCatalog
|   |   |-- paimon-core: catalog/CachingCatalog.java
|   |   |-- 基于 Caffeine Cache 缓存 Database/Table/Partition/Manifest
|   |
|   +-- PrivilegedCatalog
|       |-- paimon-core: privilege/PrivilegedCatalog.java
|       |-- 在操作前校验权限
|
+-- RESTCatalog
    |-- paimon-api: rest/RESTCatalog (通过 REST API 与远端 Catalog Server 交互)
```

**为什么这么设计?**

- `AbstractCatalog` 承载了核心的模板方法（Template Method）模式，将 database/table 的增删改查流程固定，子类只需实现 `xxxImpl()` 方法即可。这避免了每个 Catalog 实现重复处理参数校验、系统表判断等通用逻辑。
- `DelegateCatalog` 使用了委托模式（Delegate Pattern），将所有方法透传给被包装的 Catalog，子类可以选择性地覆盖感兴趣的方法。这比继承更灵活，支持运行时动态组合。
- 装饰器链 `PrivilegedCatalog → CachingCatalog → FileSystemCatalog` 的组合在 `CatalogFactory.createCatalog()` 中完成，各层职责单一，易于测试和维护。

### 2.2 Catalog 接口定义

> 源码: `paimon-core/src/main/java/org/apache/paimon/catalog/Catalog.java`

Catalog 接口定义了完整的元数据管理合约，可分为以下几组方法:

**Database 管理:**
| 方法 | 说明 |
|---|---|
| `listDatabases()` | 列出所有数据库名称 |
| `listDatabasesPaged(maxResults, pageToken, pattern)` | 分页列出数据库 |
| `createDatabase(name, ignoreIfExists, properties)` | 创建数据库 |
| `getDatabase(name)` | 获取数据库详情 |
| `dropDatabase(name, ignoreIfNotExists, cascade)` | 删除数据库 |
| `alterDatabase(name, changes, ignoreIfNotExists)` | 修改数据库属性 |

**Table 管理:**
| 方法 | 说明 |
|---|---|
| `getTable(identifier)` | 获取表对象（支持系统表 `$` 分隔） |
| `getTableById(tableId)` | 通过 tableId 获取表 |
| `listTables(databaseName)` | 列出库下所有表名 |
| `createTable(identifier, schema, ignoreIfExists)` | 创建表 |
| `dropTable(identifier, ignoreIfNotExists)` | 删除表 |
| `renameTable(fromTable, toTable, ignoreIfNotExists)` | 重命名表 |
| `alterTable(identifier, changes, ignoreIfNotExists)` | 变更表结构 |
| `invalidateTable(identifier)` | 使缓存失效 |

**Partition 管理:**
| 方法 | 说明 |
|---|---|
| `listPartitions(identifier)` | 列出所有分区 |
| `listPartitionsPaged(...)` | 分页列出分区 |
| `markDonePartitions(identifier, partitions)` | 标记分区完成 |

**版本管理（Tag/Branch/Snapshot）:**
| 方法 | 说明 |
|---|---|
| `commitSnapshot(identifier, tableUuid, snapshot, statistics)` | 提交快照 |
| `loadSnapshot(identifier)` | 加载最新快照 |
| `createTag(identifier, tagName, snapshotId, timeRetained, ignoreIfExists)` | 创建标签 |
| `deleteTag(identifier, tagName)` | 删除标签 |
| `createBranch(identifier, branch, fromTag)` | 创建分支 |
| `dropBranch(identifier, branch)` | 删除分支 |
| `fastForward(identifier, branch)` | 快进分支 |
| `rollbackTo(identifier, instant, fromSnapshot)` | 回滚到某个时间点 |

**能力探测方法（Capabilities）:**
| 方法 | 说明 |
|---|---|
| `supportsListObjectsPaged()` | 是否支持分页列表 |
| `supportsListByPattern()` | 是否支持模式匹配 |
| `supportsVersionManagement()` | 是否支持版本管理 |
| `caseSensitive()` | 是否大小写敏感 |

**为什么用能力探测模式?** 不同的 Catalog 后端能力差异巨大（FileSystem 不支持分页, REST 原生支持），通过能力探测方法让上层框架可以优雅降级，避免了大量的条件判断散布在各处。

### 2.3 AbstractCatalog 抽象基类

> 源码: `paimon-core/src/main/java/org/apache/paimon/catalog/AbstractCatalog.java`

AbstractCatalog 是模板方法模式的核心载体。它实现了 Catalog 接口的所有公共方法，并将具体的存储操作委派给 `xxxImpl()` 抽象方法:

```
公开方法（含校验逻辑）          抽象方法（子类实现）
---------------------------    -------------------------
createDatabase()          -->  createDatabaseImpl()
getDatabase()             -->  getDatabaseImpl()
dropDatabase()            -->  dropDatabaseImpl()
alterDatabase()           -->  alterDatabaseImpl()
listTables()              -->  listTablesImpl()
createTable()             -->  createTableImpl()
dropTable()               -->  dropTableImpl()
renameTable()             -->  renameTableImpl()
alterTable()              -->  alterTableImpl()
```

**核心字段:**

| 字段 | 类型 | 说明 |
|---|---|---|
| `fileIO` | `FileIO` | 文件系统抽象，用于读写元数据文件 |
| `tableDefaultOptions` | `Map<String, String>` | 全局表默认配置 |
| `context` | `CatalogContext` | Catalog 上下文配置 |

**关键通用逻辑:**

1. **系统数据库/表检查**: `createDatabase()` 中调用 `checkNotSystemDatabase(name)` 防止创建名为 `sys` 的数据库；`dropTable()` 中调用 `checkNotSystemTable(identifier)` 防止删除系统表。

2. **分支检查**: `dropTable()`, `createTable()`, `renameTable()` 中调用 `checkNotBranch(identifier)` 防止对分支引用直接进行 DDL 操作。

3. **表创建分类**: `createTable()` 根据 `CoreOptions.TYPE` 分派到不同的创建路径:
   ```java
   switch (Options.fromMap(schema.options()).get(TYPE)) {
       case TABLE:
       case MATERIALIZED_TABLE:
           createTableImpl(identifier, schema);   // 正常表
           break;
       case FORMAT_TABLE:
           createFormatTable(identifier, schema);  // 纯格式表
           break;
       case OBJECT_TABLE:
           throw new UnsupportedOperationException(...); // 不支持
   }
   ```

4. **getTable() 的统一加载**: 通过 `CatalogUtils.loadTable()` 统一处理普通表和系统表的加载，系统表通过 `$` 分隔符自动识别和创建。

5. **锁工厂**: 通过 `lockFactory()` 获取分布式锁（如 Hive MetaStore 锁），在对象存储场景下自动启用锁机制 (`lockEnabled()` 判断 `fileIO.isObjectStore()`)。

**为什么在 AbstractCatalog 层处理系统表判断?** 因为所有 Catalog 实现都需要统一处理系统表的路由逻辑（`$snapshots`, `$schemas` 等），将这一逻辑提升到抽象层避免了各实现的重复代码和潜在不一致。

### 2.4 FileSystemCatalog 实现

> 源码: `paimon-core/src/main/java/org/apache/paimon/catalog/FileSystemCatalog.java`

FileSystemCatalog 是最核心的 Catalog 实现，将元数据完全存储在文件系统上。

**文件系统目录布局:**

```
{warehouse}/
  +-- {database}.db/
       +-- {table}/
            +-- schema/
            |    +-- schema-0    (JSON 格式的 TableSchema)
            |    +-- schema-1
            |    +-- ...
            +-- snapshot/
            |    +-- snapshot-1  (JSON 格式的 Snapshot)
            |    +-- snapshot-2
            |    +-- ...
            +-- tag/
            |    +-- tag-{name}  (JSON 格式, 内容同 Snapshot + TTL 信息)
            +-- branch/
            |    +-- branch-{name}/
            |         +-- schema/
            |         +-- snapshot/
            |         +-- tag/
            +-- manifest/
            |    +-- manifest-list-{uuid}  (ManifestFileMeta 列表)
            |    +-- manifest-{uuid}       (ManifestEntry 列表)
            +-- data files...
```

**关键实现方法:**

| 方法 | 实现逻辑 |
|---|---|
| `listDatabases()` | 列出 `{warehouse}/` 下的 `.db` 结尾的目录 |
| `getDatabaseImpl()` | 检查 `{warehouse}/{name}.db/` 路径是否存在 |
| `createDatabaseImpl()` | 创建 `{warehouse}/{name}.db/` 目录 |
| `dropDatabaseImpl()` | 递归删除 `{warehouse}/{name}.db/` 目录 |
| `listTablesImpl()` | 列出 `{database}.db/` 下含 `schema/` 子目录的目录 |
| `loadTableSchema()` | 从 `{table}/schema/` 读取最新 schema 文件 |
| `createTableImpl()` | 通过 `SchemaManager.createTable()` 写入初始 schema |
| `dropTableImpl()` | 递归删除表目录及外部路径 |
| `renameTableImpl()` | 重命名表目录 (`fileIO.rename()`) |
| `alterTableImpl()` | 通过 `SchemaManager.commitChanges()` 提交 schema 变更 |

**锁机制:**

FileSystemCatalog 的 `createTableImpl()` 和 `alterTableImpl()` 都通过 `runWithLock()` 方法包装，确保在并发场景下的安全性:

```java
public <T> T runWithLock(Identifier identifier, Callable<T> callable) throws Exception {
    Optional<CatalogLockFactory> lockFactory = lockFactory();
    try (Lock lock = lockFactory
            .map(factory -> factory.createLock(lockContext().orElse(null)))
            .map(l -> Lock.fromCatalog(l, identifier))
            .orElseGet(Lock::empty)) {
        return lock.runWithLock(callable);
    }
}
```

**为什么 FileSystem Catalog 不支持 `alterDatabase()`?** 因为文件系统本身不提供目录级别的属性存储能力。数据库在文件系统上仅表现为一个目录，没有独立的元数据文件来持久化属性。需要属性支持的场景应使用 Hive Catalog 或 REST Catalog。

**为什么 `renameTable` 注释中警告对象存储?** 因为 S3、OSS 等对象存储的 rename 操作并非原子的——它们实际上是 copy + delete，在大表场景下可能只完成了部分文件的迁移就失败了，留下不一致的状态。

### 2.5 DelegateCatalog 委托模式

> 源码: `paimon-core/src/main/java/org/apache/paimon/catalog/DelegateCatalog.java`

DelegateCatalog 是一个抽象类，实现了 Catalog 接口的所有方法，每个方法都简单透传给 `wrapped` 成员:

```java
public abstract class DelegateCatalog implements Catalog {
    protected final Catalog wrapped;

    @Override
    public Table getTable(Identifier identifier) throws TableNotExistException {
        return wrapped.getTable(identifier);
    }
    // ... 其他所有方法同理
}
```

它还提供了一个静态工具方法 `rootCatalog()` 来剥去所有装饰器层，获取最底层的 Catalog:

```java
public static Catalog rootCatalog(Catalog catalog) {
    while (catalog instanceof DelegateCatalog) {
        catalog = ((DelegateCatalog) catalog).wrapped();
    }
    return catalog;
}
```

**为什么不直接用接口默认方法?** 因为 Java 8 的接口默认方法不能访问实例字段 (`wrapped`)。使用抽象类可以持有被委托对象的引用，且子类（如 CachingCatalog）可以选择性覆盖部分方法来插入缓存逻辑，而其余方法自动透传，大幅减少样板代码。

### 2.6 CachingCatalog 缓存装饰器

> 源码: `paimon-core/src/main/java/org/apache/paimon/catalog/CachingCatalog.java`

CachingCatalog 继承 DelegateCatalog，在元数据操作上添加基于 Caffeine Cache 的多级缓存:

**缓存类型:**

| 缓存 | 类型 | 说明 |
|---|---|---|
| `databaseCache` | `Cache<String, Database>` | 数据库元数据缓存 |
| `tableCache` | `Cache<Identifier, Table>` | 表对象缓存 |
| `partitionCache` | `Cache<Identifier, List<Partition>>` | 分区列表缓存（可选） |
| `manifestCache` | `SegmentsCache<Path>` | Manifest 文件内容缓存 |
| `dvMetaCache` | `DVMetaCache` | Deletion Vector 元数据缓存（可选） |

**缓存配置项:**

| 配置键 | 默认值 | 说明 |
|---|---|---|
| `cache.enabled` | `true` | 是否启用缓存 |
| `cache.expire-after-access` | - | 访问后过期时间 |
| `cache.expire-after-write` | - | 写入后过期时间 |
| `cache.snapshot-max-num-per-table` | - | 每表最大缓存快照数 |
| `cache.partition-max-num` | `0` | 最大缓存分区数（0=不缓存） |
| `cache.manifest-small-file-memory` | - | 小 manifest 文件缓存内存 |
| `cache.manifest-small-file-threshold` | - | 小文件阈值 |
| `cache.manifest-max-memory` | - | manifest 最大缓存内存 |
| `cache.dv-max-num` | - | DV 元数据最大缓存数 |

**缓存失效策略:**

CachingCatalog 在所有写操作后主动失效相关缓存:

```java
// dropTable 后失效该表及其所有分支的缓存
public void dropTable(Identifier identifier, boolean ignoreIfNotExists) {
    super.dropTable(identifier, ignoreIfNotExists);
    invalidateTable(identifier);
    // 同时清除所有分支表的缓存
    for (Identifier i : tableCache.asMap().keySet()) {
        if (identifier.getTableName().equals(i.getTableName())
                && identifier.getDatabaseName().equals(i.getDatabaseName())) {
            tableCache.invalidate(i);
        }
    }
}
```

**系统表的特殊缓存逻辑:** 系统表（如 `my_table$snapshots`）不会被直接缓存。CachingCatalog 会缓存原始表 `my_table`，然后在 `getTable()` 时动态包装为系统表。这样原始表的缓存可以被多个系统表共用:

```java
if (identifier.isSystemTable()) {
    Identifier originIdentifier = new Identifier(
            identifier.getDatabaseName(), identifier.getTableName(),
            identifier.getBranchName(), null);
    Table originTable = getTable(originIdentifier); // 使用缓存
    table = SystemTableLoader.load(identifier.getSystemTableName(), (FileStoreTable) originTable);
}
```

**创建时机:** `CachingCatalog` 在 `CatalogFactory.createCatalog()` 中通过 `CachingCatalog.tryToCreate()` 按需创建:

```java
static Catalog createCatalog(CatalogContext context, ClassLoader classLoader) {
    Catalog catalog = createUnwrappedCatalog(context, classLoader);
    catalog = CachingCatalog.tryToCreate(catalog, options);
    return PrivilegedCatalog.tryToCreate(catalog, options);
}
```

**为什么 Manifest 缓存单独用 SegmentsCache?** Manifest 文件是二进制格式的大文件，不适合放入通用的 key-value 缓存。`SegmentsCache` 是一个基于内存大小（而非条目数）进行限制的缓存，支持按文件大小阈值决定是否缓存（小文件缓存、大文件不缓存），更适合处理大小差异悬殊的文件。

### 2.7 CatalogFactory SPI 加载机制

> 源码: `paimon-core/src/main/java/org/apache/paimon/catalog/CatalogFactory.java`

CatalogFactory 是 Paimon 的 SPI 扩展点，继承自 `Factory` 接口，通过 Java SPI（ServiceLoader）机制加载:

```java
public interface CatalogFactory extends Factory {
    // 方式一: 需要 FileIO 和 warehouse 的创建方法
    default Catalog create(FileIO fileIO, Path warehouse, CatalogContext context);

    // 方式二: 只需 context 的创建方法（如 RESTCatalog）
    default Catalog create(CatalogContext context);
}
```

**加载流程:**

```
CatalogFactory.createCatalog(context, classLoader)
    |
    +-- createUnwrappedCatalog(context, classLoader)
    |     |
    |     +-- 从 context 中读取 "metastore" 配置
    |     +-- FactoryUtil.discoverFactory(classLoader, CatalogFactory.class, metastore)
    |     |     // 使用 Java SPI 发现匹配 identifier 的 CatalogFactory
    |     +-- 先尝试 catalogFactory.create(context)
    |     +-- 若 UnsupportedOperationException:
    |           +-- 读取 warehouse 路径
    |           +-- 创建 FileIO
    |           +-- catalogFactory.create(fileIO, warehousePath, context)
    |
    +-- CachingCatalog.tryToCreate(catalog, options)  // 包装缓存层
    +-- PrivilegedCatalog.tryToCreate(catalog, options) // 包装权限层
```

**已知的 CatalogFactory 实现:**

| Factory 类 | identifier | 说明 |
|---|---|---|
| `FileSystemCatalogFactory` | `"filesystem"` | 默认文件系统 Catalog |
| `HiveCatalogFactory` | `"hive"` | Hive MetaStore Catalog |
| `RESTCatalogFactory` | `"rest"` | REST API Catalog |

**为什么有两个 create 方法?** 这是一个向后兼容的设计。早期的 Catalog 实现（如 FileSystemCatalog）需要外部传入 `FileIO` 和 `warehouse`，而新的实现（如 RESTCatalog）完全自包含，只需 `CatalogContext` 即可。两个方法互为 fallback，通过 UnsupportedOperationException 进行分派。

### 2.8 Identifier 表标识体系

> 源码: `paimon-api/src/main/java/org/apache/paimon/catalog/Identifier.java`

Identifier 是 Paimon 中标识表/系统表/分支的统一标识符:

```
格式: {database}.{table}[$branch_{branchName}][$systemTable]

示例:
  my_db.my_table                         -- 普通表
  my_db.my_table$snapshots               -- 系统表
  my_db.my_table$branch_dev              -- 分支表
  my_db.my_table$branch_dev$snapshots    -- 分支的系统表
```

**字段:**

| 字段 | 类型 | 说明 |
|---|---|---|
| `database` | `String` | 数据库名称 |
| `object` | `String` | 完整的对象名（包含表名、分支、系统表信息） |
| `table` (transient) | `String` | 解析后的表名 |
| `branch` (transient) | `String` | 解析后的分支名（null 表示 main） |
| `systemTable` (transient) | `String` | 解析后的系统表名（null 表示非系统表） |

`object` 字段的解析通过 `splitObjectName()` 懒加载完成，使用 `$` 作为分隔符:

- 1 段: `table` (普通表)
- 2 段: `table$branch_xxx` 或 `table$systemTable`
- 3 段: `table$branch_xxx$systemTable`

**为什么用 `$` 作分隔符而不是 `.`?** 因为 `.` 已经被用于 `database.table` 的分隔。`$` 在 SQL 标识符中是合法但不常用的字符，既避免了与表名冲突，又能在 SQL 中直接使用（如 `SELECT * FROM my_table$snapshots`）。

---

## 三、Schema 演进机制

### 3.1 TableSchema 完整字段

> 源码: `paimon-api/src/main/java/org/apache/paimon/schema/TableSchema.java`

TableSchema 是表的完整元数据定义，比用户提供的 Schema 多了 ID、版本等内部信息:

| 字段 | 类型 | 说明 |
|---|---|---|
| `version` | `int` | Schema 格式版本（当前 v3，Paimon 0.7=v1, 0.8=v2） |
| `id` | `long` | Schema ID，单调递增 |
| `fields` | `List<DataField>` | 字段列表（不可变） |
| `highestFieldId` | `int` | 已分配的最大字段 ID（因为删除列后 ID 不会回收） |
| `partitionKeys` | `List<String>` | 分区键字段名 |
| `primaryKeys` | `List<String>` | 主键字段名 |
| `bucketKeys` | `List<String>` | 桶键字段名（派生字段） |
| `numBucket` | `int` | 桶数量（派生字段，来自 options） |
| `options` | `Map<String, String>` | 表级配置（CoreOptions） |
| `comment` | `String` (nullable) | 表注释 |
| `timeMillis` | `long` | Schema 创建时间戳 |

**关于 `highestFieldId` 的设计:** 当删除列后再添加新列时，新列的字段 ID 从 `highestFieldId + 1` 开始分配，确保 ID 永远不会重复。这对 Schema Evolution（如读取旧数据文件时的列映射）至关重要。

**关于 `bucketKeys` 的派生:** 如果用户未显式指定 `bucket-key` 配置，则默认使用 `primaryKeys`（去掉分区键部分）作为桶键:

```java
List<String> tmpBucketKeys = originalBucketKeys(); // 从 options 中读取
if (tmpBucketKeys.isEmpty()) {
    tmpBucketKeys = trimmedPrimaryKeys(); // fallback to primary keys
}
bucketKeys = tmpBucketKeys;
```

**序列化格式:** TableSchema 序列化为 JSON 字符串后写入文件系统。

### 3.2 SchemaManager 管理器

> 源码: `paimon-core/src/main/java/org/apache/paimon/schema/SchemaManager.java`

SchemaManager 是 Schema 的 CRUD 管理器，标注为 `@ThreadSafe`，其核心方法:

| 方法 | 说明 |
|---|---|
| `latest()` | 获取最新版本的 TableSchema（扫描 schema 目录，取最大 ID） |
| `schema(id)` | 读取指定 ID 的 TableSchema |
| `listAll()` | 列出所有版本的 TableSchema |
| `listAllIds()` | 列出所有 Schema ID |
| `createTable(schema)` | 创建新表（写入 schema-0） |
| `commitChanges(changes)` | 提交 SchemaChange 列表 |
| `mergeSchema(rowType, allowExplicitCast, ...)` | 合并新 RowType 到当前 Schema |
| `commit(newSchema)` | 原子写入新 Schema 文件 |

**Schema 文件命名:** `schema-{id}`，例如 `schema-0`, `schema-1`, `schema-2`。

**createTable 的乐观锁流程:**

```java
public TableSchema createTable(Schema schema, boolean externalTable) throws Exception {
    while (true) {
        Optional<TableSchema> latest = latest();
        if (latest.isPresent()) {
            // 表已存在
            if (externalTable) {
                checkSchemaForExternalTable(latestSchema.toSchema(), schema);
                return latestSchema;
            } else {
                throw new IllegalStateException("Schema in filesystem exists");
            }
        }
        TableSchema newSchema = TableSchema.create(0, schema); // id = 0
        // 验证: 用新 schema 实际创建 FileStoreTable 来确保配置合法
        FileStoreTableFactory.create(fileIO, tableRoot, newSchema).store();
        boolean success = commit(newSchema); // 原子写入
        if (success) return newSchema;
        // 如果失败（并发创建），循环重试
    }
}
```

### 3.3 SchemaChange 所有变更类型

> 源码: `paimon-api/src/main/java/org/apache/paimon/schema/SchemaChange.java`

SchemaChange 是一个接口，使用 Jackson 的 `@JsonSubTypes` 支持 JSON 多态序列化（用于 REST Catalog 的 API 传输）:

| 变更类型 | 类名 | 说明 |
|---|---|---|
| 设置表选项 | `SetOption` | 设置 key-value 配置项 |
| 删除表选项 | `RemoveOption` | 删除配置项 |
| 更新表注释 | `UpdateComment` | 更新或清除表注释 |
| 添加列 | `AddColumn` | 支持嵌套路径、位置指定(FIRST/AFTER/BEFORE) |
| 重命名列 | `RenameColumn` | 支持嵌套路径 |
| 删除列 | `DropColumn` | 支持嵌套路径 |
| 修改列类型 | `UpdateColumnType` | 支持保留空值属性的类型变更 |
| 修改列可空性 | `UpdateColumnNullability` | 更改列的 NULL/NOT NULL |
| 修改列注释 | `UpdateColumnComment` | 更新列注释 |
| 修改列默认值 | `UpdateColumnDefaultValue` | 更新列默认值 |
| 修改列位置 | `UpdateColumnPosition` | 通过 Move 改变列顺序 |
| 删除主键 | `DropPrimaryKey` | 删除主键定义 |

**Move 类型:**

```java
enum MoveType {
    FIRST,   // 移到最前
    AFTER,   // 移到指定列之后
    BEFORE   // 移到指定列之前（SchemaManager 中使用）
}
```

**AddColumn 的约束:** 新增列必须是可空的 (`isNullable() == true`)，这是为了保证向后兼容性——旧的数据文件不包含新列的数据，在读取时自动填充 null。

**commitChanges 的处理流程:**

```
commitChanges(List<SchemaChange> changes)
    |
    +-- while(true) // 乐观锁循环
    |     |
    |     +-- latest() 获取当前最新 Schema
    |     +-- generateTableSchema(oldSchema, changes, ...) // 纯函数，生成新 Schema
    |     |     |
    |     |     +-- 遍历每个 SchemaChange:
    |     |     |     SetOption        -> 更新 options map
    |     |     |     RemoveOption     -> 从 options 中移除
    |     |     |     UpdateComment    -> 更新 comment
    |     |     |     AddColumn        -> 分配新 fieldId, 添加到字段列表
    |     |     |     RenameColumn     -> 更新字段名及相关 options 中的引用
    |     |     |     DropColumn       -> 从字段列表中移除（校验非主键/分区键）
    |     |     |     UpdateColumnType -> 检查类型兼容性, 更新类型
    |     |     |     ...
    |     |     +-- 构造 newTableSchema (id = oldId + 1)
    |     |
    |     +-- commit(newSchema) // 原子写入 schema-{newId}
    |     +-- if (success) return newSchema
    |     +-- // 否则继续循环（别人先写了同一个 ID）
```

### 3.4 乐观锁机制与原子写入

> 源码: `SchemaManager.java` L1083-1088

Schema 的并发控制核心在 `commit()` 方法:

```java
public boolean commit(TableSchema newSchema) throws Exception {
    SchemaValidation.validateTableSchema(newSchema);
    SchemaValidation.validateFallbackBranch(this, newSchema);
    Path schemaPath = toSchemaPath(newSchema.id());
    return fileIO.tryToWriteAtomic(schemaPath, newSchema.toString());
}
```

**`tryToWriteAtomic()`** 是 FileIO 接口提供的原子写入方法:
- **本地文件系统/HDFS**: 使用原子重命名（先写临时文件，再 rename 到目标路径）。如果目标路径已存在则返回 false。
- **对象存储（S3/OSS）**: 使用 `putIfAbsent` 语义（如果支持），否则回退到先检查存在性再写入（需要配合分布式锁）。

**乐观锁逻辑:**
1. 读取当前最新 Schema ID（如 `5`）
2. 生成新 Schema 的 ID = `6`
3. 尝试原子写入 `schema-6`
4. 如果写入成功，操作完成
5. 如果写入失败（有人已经写了 `schema-6`），回到步骤 1 重试

**为什么不使用分布式锁?** 对于 HDFS 等提供原子重命名的文件系统，乐观锁已经足够安全且性能更好（无需额外的锁服务依赖）。只有在对象存储场景下，才需要通过 `CatalogLock` 引入外部锁（如 MySQL/ZooKeeper 锁）。

### 3.5 Schema 存储路径规则

```
主分支: {table_path}/schema/schema-{id}
其他分支: {table_path}/branch/branch-{branchName}/schema/schema-{id}
```

路径由 SchemaManager 中的 `branchPath()` + `schemaDirectory()` + `toSchemaPath()` 组合构建:

```java
private String branchPath() {
    return BranchManager.branchPath(tableRoot, branch);
    // main 分支: {tableRoot}
    // 其他分支: {tableRoot}/branch/branch-{branch}
}

public Path schemaDirectory() {
    return new Path(branchPath() + "/schema");
}

public Path toSchemaPath(long schemaId) {
    return new Path(branchPath() + "/schema/" + SCHEMA_PREFIX + schemaId);
}
```

---

## 四、Snapshot 管理

### 4.1 Snapshot 完整字段表格

> 源码: `paimon-api/src/main/java/org/apache/paimon/Snapshot.java`

Snapshot 是 Paimon 表在某个时间点的完整数据视图入口。当前版本为 v3。

| 字段 | 类型 | 是否可空 | 说明 |
|---|---|---|---|
| `version` | `int` | 否 | Snapshot 格式版本（当前 v3） |
| `id` | `long` | 否 | 快照 ID，从 1 开始单调递增 |
| `schemaId` | `long` | 否 | 对应的 Schema 版本 ID |
| `baseManifestList` | `String` | 否 | 基线 Manifest 列表文件名（包含所有历史文件的合并结果） |
| `baseManifestListSize` | `Long` | 是 | 基线列表文件大小（Paimon <=1.0 为 null） |
| `deltaManifestList` | `String` | 否 | 增量 Manifest 列表文件名（本次提交的变更） |
| `deltaManifestListSize` | `Long` | 是 | 增量列表文件大小 |
| `changelogManifestList` | `String` | 是 | Changelog Manifest 列表文件名 |
| `changelogManifestListSize` | `Long` | 是 | Changelog 列表文件大小 |
| `indexManifest` | `String` | 是 | 索引 Manifest 文件名（Deletion Vector 等） |
| `commitUser` | `String` | 否 | 提交者标识（用于去重和冲突检测） |
| `commitIdentifier` | `long` | 否 | 提交标识符（用于快照去重） |
| `commitKind` | `CommitKind` | 否 | 提交类型 (APPEND/COMPACT/OVERWRITE/ANALYZE) |
| `timeMillis` | `long` | 否 | 提交时间戳（毫秒） |
| `totalRecordCount` | `long` | 否 | 表的总记录数（累计） |
| `deltaRecordCount` | `long` | 否 | 本次变更的记录数（新增-删除） |
| `changelogRecordCount` | `Long` | 是 | Changelog 记录数 |
| `watermark` | `Long` | 是 | 水位线（用于流式读取） |
| `statistics` | `String` | 是 | 统计信息文件名 |
| `properties` | `Map<String,String>` | 是 | 附加属性 |
| `nextRowId` | `Long` | 是 | 下一个可用的行 ID（Row Tracking 特性） |

**commitIdentifier 的语义:** 如果多个 snapshot 具有相同的 commitIdentifier，则从任一 snapshot 读取的结果必须相同。commitIdentifier 较小的 snapshot 包含较老的数据。这用于 Flink 的精确一次语义下的去重。

### 4.2 base/delta/changelog 三分法

Snapshot 中的数据文件信息通过三个 ManifestList 来组织:

```
Snapshot
  |
  +-- baseManifestList  -----> [ManifestFileMeta_1, ManifestFileMeta_2, ...]
  |                               |                    |
  |                               v                    v
  |                          ManifestFile_1       ManifestFile_2
  |                          [Entry_A(ADD),       [Entry_D(ADD),
  |                           Entry_B(ADD),        Entry_E(ADD)]
  |                           Entry_C(ADD)]
  |
  +-- deltaManifestList -----> [ManifestFileMeta_3]
  |                               |
  |                               v
  |                          ManifestFile_3
  |                          [Entry_F(ADD),       // 本次新增的文件
  |                           Entry_G(DELETE)]    // 本次删除的文件
  |
  +-- changelogManifestList -> [ManifestFileMeta_4]  (可选)
                                  |
                                  v
                             ManifestFile_4
                             [changelog entries]  // 用于 CDC 场景
```

**三分法的设计原因:**

1. **baseManifestList（基线）**: 包含了截至本次提交前所有有效数据文件的合并 manifest。它是经过 manifest 合并优化的，可以高效地读取全量文件列表。

2. **deltaManifestList（增量）**: 仅包含本次提交新产生的变更（ADD 和 DELETE entries）。它的好处是:
   - **快速过期**: 过期旧快照时，只需扫描 delta 部分即可知道需要清理哪些文件
   - **流式读取**: 增量消费者只需读取 delta 部分即可获取变更

3. **changelogManifestList（变更日志）**: 专门记录 CDC 变更日志文件。与 delta 分离是因为 changelog 文件有独立的生命周期和存储策略。

**为什么不把所有文件放在一个列表中?** 将增量和全量分开存储是一个关键的性能优化。全量扫描只需读取 base，增量消费只需读取 delta，各取所需。如果混在一起，增量消费者需要对比两个版本的全量列表来计算差异，代价极高。

### 4.3 CommitKind 提交类型

> 源码: `paimon-api/src/main/java/org/apache/paimon/Snapshot.java` L454-470

```java
public enum CommitKind {
    APPEND,     // 追加新数据文件，不删除已有文件
    COMPACT,    // 合并压缩（不改变数据内容，只改变物理存储形式）
    OVERWRITE,  // 覆写分区或删除已有文件后添加新文件
    ANALYZE     // 收集统计信息
}
```

**为什么把 COMPACT 单独作为一种 CommitKind?** 因为 compaction 不改变逻辑数据内容，只改变物理文件组织。这意味着在冲突检测中，COMPACT 提交可以更宽松地处理——它不会与 APPEND 提交产生数据层面的冲突（尽管可能需要处理文件引用冲突）。

### 4.4 SnapshotManager 管理器

> 源码: `paimon-core/src/main/java/org/apache/paimon/utils/SnapshotManager.java`

SnapshotManager 管理快照文件的读写和生命周期:

**核心字段:**

| 字段 | 类型 | 说明 |
|---|---|---|
| `fileIO` | `FileIO` | 文件系统抽象 |
| `tablePath` | `Path` | 表根路径 |
| `branch` | `String` | 分支名称 |
| `snapshotLoader` | `SnapshotLoader` | 快照加载器（REST 模式用） |
| `cache` | `Cache<Path, Snapshot>` | 快照缓存 |

**路径规则:**

```
主分支: {tablePath}/snapshot/snapshot-{id}
其他分支: {tablePath}/branch/branch-{name}/snapshot/snapshot-{id}
```

**关键方法:**

| 方法 | 说明 |
|---|---|
| `latestSnapshot()` | 获取最新快照（先尝试 loader，再从文件系统扫描） |
| `snapshot(id)` | 读取指定 ID 的快照（带缓存） |
| `snapshotExists(id)` | 检查快照是否存在 |
| `earliestSnapshotId()` | 获取最早的快照 ID |
| `latestSnapshotOfUser(commitUser)` | 获取指定用户的最新快照（用于去重） |
| `deleteSnapshot(id)` | 删除快照文件 |

**latestSnapshot 的双重加载策略:**

```java
public Snapshot latestSnapshot() {
    if (snapshotLoader != null) {
        try {
            snapshot = snapshotLoader.load().orElse(null); // 尝试 REST 加载
        } catch (UnsupportedOperationException ignored) {
            snapshot = latestSnapshotFromFileSystem(); // fallback 到文件系统
        }
    } else {
        snapshot = latestSnapshotFromFileSystem();
    }
    // 放入缓存
    if (snapshot != null && cache != null) {
        cache.put(snapshotPath(snapshot.id()), snapshot);
    }
    return snapshot;
}
```

**为什么 `latestSnapshotFromFileSystem()` 扫描文件系统而不是维护一个指针文件?** 因为文件系统上的 snapshot 文件名包含了 ID（`snapshot-1`, `snapshot-2`），可以通过目录列表快速找到最大 ID。维护指针文件反而增加了额外的一致性风险（指针文件和实际快照文件之间可能不一致）。

---

## 五、Tag 和 Branch 机制

### 5.1 TagManager 标签管理

> 源码: `paimon-core/src/main/java/org/apache/paimon/utils/TagManager.java`

Tag 是对某个 Snapshot 的命名引用，类似 Git Tag。Tag 的物理文件内容就是被标记 Snapshot 的完整 JSON（可能附加 TTL 信息）。

**路径规则:**

```
{tablePath}/[branch/branch-{name}/]tag/tag-{tagName}
```

**核心方法:**

| 方法 | 说明 |
|---|---|
| `createTag(snapshot, tagName, timeRetained, callbacks, ignoreIfExists)` | 创建 Tag |
| `replaceTag(snapshot, tagName, timeRetained, callbacks)` | 替换已有 Tag |
| `deleteTag(tagName, tagDeletion, snapshotManager, callbacks)` | 删除 Tag 及其独有的数据文件 |
| `deleteAllTagsOfOneSnapshot(tagNames, tagDeletion, snapshotManager)` | 删除指向同一快照的所有 Tag |
| `renameTag(tagName, targetTagName)` | 重命名 Tag |
| `tagExists(tagName)` | 检查 Tag 是否存在 |
| `taggedSnapshots()` | 获取所有 Tag 对应的快照列表 |

**Tag 创建的文件写入:**

```java
private void createOrReplaceTag(Snapshot snapshot, String tagName,
        Duration timeRetained, List<TagCallback> callbacks) {
    validateNoAutoTag(tagName, snapshot);
    String content = timeRetained != null
            ? Tag.fromSnapshotAndTagTtl(snapshot, timeRetained, LocalDateTime.now()).toJson()
            : snapshot.toJson();
    Path tagPath = tagPath(tagName);
    fileIO.overwriteFileUtf8(tagPath, content);  // 直接覆写（幂等操作）
    // 触发回调（如 Iceberg 兼容层）
    if (callbacks != null) {
        callbacks.forEach(callback -> callback.notifyCreation(tagName, snapshot.id()));
    }
}
```

**Tag 删除的安全逻辑:**

当删除 Tag 时，需要判断被标记的 Snapshot 是否仍然被其他 Snapshot 或 Tag 引用:
- 如果对应的 Snapshot 仍然存在于快照链中 → 只删除 Tag 文件
- 如果 Snapshot 已被过期删除，Tag 是最后一个引用 → 还需要清理被 Tag 独占的数据文件

### 5.2 Tag TTL 机制

> 源码: `paimon-core/src/main/java/org/apache/paimon/tag/Tag.java`

Tag 类继承自 Snapshot，额外添加了 TTL 相关字段:

| 字段 | 类型 | 说明 |
|---|---|---|
| `tagCreateTime` | `LocalDateTime` (nullable) | Tag 创建时间 |
| `tagTimeRetained` | `Duration` (nullable) | Tag 保留时间 |

**向后兼容设计:** 当 `timeRetained` 未指定时，不写入 `tagCreateTime` 字段。这确保了老版本 Paimon（<= 0.7）的读取器仍然能正确解析 Tag 文件（因为 Tag 文件的 JSON 格式使用了 `@JsonIgnoreProperties(ignoreUnknown = true)`，未知字段会被忽略）。

**TTL 过期处理:** TagManager 持有一个 `TagPeriodHandler`，在 Flink/Spark 作业的生命周期中周期性检查 Tag 是否过期:

```java
// 过期条件: tagCreateTime + tagTimeRetained < 当前时间
if (tag.tagCreateTime() != null && tag.tagTimeRetained() != null) {
    LocalDateTime expireTime = tag.tagCreateTime().plus(tag.tagTimeRetained());
    if (expireTime.isBefore(LocalDateTime.now())) {
        // 执行删除
    }
}
```

### 5.3 BranchManager 分支管理

> 接口: `paimon-core/src/main/java/org/apache/paimon/utils/BranchManager.java`
> 实现: `paimon-core/src/main/java/org/apache/paimon/utils/FileSystemBranchManager.java`

BranchManager 管理表的分支机制。分支允许在同一张表上创建独立的演进路径，类似 Git 的分支。

**接口方法:**

| 方法 | 说明 |
|---|---|
| `createBranch(branchName)` | 从最新 Schema 创建空分支 |
| `createBranch(branchName, tagName)` | 从指定 Tag 创建分支 |
| `dropBranch(branchName)` | 删除分支及其所有数据 |
| `fastForward(branchName)` | 将分支的最新状态快进到主分支 |
| `renameBranch(fromBranch, toBranch)` | 重命名分支 |
| `branches()` | 列出所有分支 |

**静态工具方法:**

```java
// 构建分支路径
static String branchPath(Path tablePath, String branch) {
    return isMainBranch(branch)
            ? tablePath.toString()
            : tablePath.toString() + "/branch/" + BRANCH_PREFIX + branch;
}

// 规范化分支名（null/空 → "main"）
static String normalizeBranch(String branch) {
    return StringUtils.isNullOrWhitespaceOnly(branch) ? DEFAULT_MAIN_BRANCH : branch;
}
```

**分支名验证规则:**
1. 不能是 `"main"`（保留名称）
2. 不能为空或全空白
3. 不能是纯数字字符串（避免与 snapshot ID 混淆）

### 5.4 分支路径规则与文件布局

**FileSystemBranchManager 的创建逻辑:**

**从最新 Schema 创建空分支:**
```java
public void createBranch(String branchName, boolean ignoreIfExists) {
    validateBranch(branchName);
    TableSchema latestSchema = schemaManager.latest().get();
    copySchemasToBranch(branchName, latestSchema.id());
    // 结果: {table}/branch/branch-{name}/schema/schema-0..N 被复制
}
```

**从 Tag 创建分支:**
```java
public void createBranch(String branchName, String tagName, boolean ignoreIfExists) {
    validateBranch(branchName);
    Snapshot snapshot = tagManager.getOrThrow(tagName).trimToSnapshot();
    // 复制 Tag 文件
    fileIO.copyFile(tagManager.tagPath(tagName),
                    tagManager.copyWithBranch(branchName).tagPath(tagName), true);
    // 复制 Snapshot 文件
    fileIO.copyFile(snapshotManager.snapshotPath(snapshot.id()),
                    snapshotManager.copyWithBranch(branchName).snapshotPath(snapshot.id()), true);
    // 复制所有 Schema 文件（到 tag 对应的 schema id）
    copySchemasToBranch(branchName, snapshot.schemaId());
}
```

**文件布局示例:**

```
{warehouse}/my_db.db/my_table/
  +-- schema/                          # 主分支 schema
  |    +-- schema-0
  |    +-- schema-1
  +-- snapshot/                        # 主分支 snapshot
  |    +-- snapshot-1
  |    +-- snapshot-2
  +-- tag/                             # 主分支 tag
  |    +-- tag-release_v1
  +-- branch/
  |    +-- branch-dev/                 # dev 分支
  |    |    +-- schema/
  |    |    |    +-- schema-0
  |    |    |    +-- schema-1
  |    |    +-- snapshot/
  |    |    |    +-- snapshot-1        # 从 tag 复制而来
  |    |    +-- tag/
  |    |         +-- tag-release_v1   # 从主分支复制而来
  |    +-- branch-feature_x/           # feature_x 分支
  |         +-- schema/
  |              +-- schema-0
  |              +-- schema-1
  +-- manifest/                        # 共享的 manifest 文件
  +-- data/                            # 共享的数据文件
```

**为什么分支共享 manifest 和 data 目录?** 分支创建时通过复制 snapshot/schema/tag 文件实现逻辑隔离，但底层的数据文件和 manifest 文件是共享的。这是一个写时复制（Copy-on-Write）策略——分支创建是轻量的 O(1) 操作，新的写入会生成新的数据文件，不会修改已有文件。

---

## 六、Manifest 三级结构

### 6.1 整体架构

Paimon 使用三级 Manifest 结构来管理数据文件的元数据:

```
Snapshot (JSON 文件)
  |
  |  baseManifestList / deltaManifestList / changelogManifestList
  |           (文件名引用)
  |
  v
ManifestList (二进制文件, 包含 ManifestFileMeta 列表)
  |
  |  fileName 引用
  |
  v
ManifestFile (二进制文件, 包含 ManifestEntry 列表)
  |
  |  DataFileMeta 引用
  |
  v
Data Files (实际数据文件: ORC / Parquet / Avro)
```

**为什么用三级结构而不是直接在 Snapshot 中记录所有文件?**

1. **可伸缩性**: 一个大表可能包含数百万个数据文件。如果全部记录在 Snapshot JSON 中，每次提交都需要重写整个文件，代价极高。
2. **增量写入**: 通过三级结构，每次提交只需:
   - 写入新的 ManifestFile（包含新增/删除的 entries）
   - 写入新的 ManifestList（引用更新后的 ManifestFile 集合）
   - 写入新的 Snapshot（引用新的 ManifestList）
3. **Manifest 合并优化**: 随着提交次数增加，ManifestFile 数量也会增加。可以周期性地将多个小 ManifestFile 合并为大文件，减少读取时的 IO 次数。

### 6.2 ManifestList 清单列表

> 源码: `paimon-core/src/main/java/org/apache/paimon/manifest/ManifestList.java`

ManifestList 继承自 `ObjectsFile<ManifestFileMeta>`，是一个包含多个 `ManifestFileMeta` 的二进制文件:

**读取方法:**

| 方法 | 说明 |
|---|---|
| `readAllManifests(snapshot)` | 读取所有 manifest（data + changelog） |
| `readDataManifests(snapshot)` | 读取数据 manifest（base + delta） |
| `readDeltaManifests(snapshot)` | 仅读取 delta manifest |
| `readChangelogManifests(snapshot)` | 仅读取 changelog manifest |

**读取实现:**

```java
public List<ManifestFileMeta> readDataManifests(Snapshot snapshot) {
    List<ManifestFileMeta> result = new ArrayList<>();
    result.addAll(read(snapshot.baseManifestList(), snapshot.baseManifestListSize()));
    result.addAll(readDeltaManifests(snapshot));
    return result;
}
```

**写入方法:**

```java
public Pair<String, Long> write(List<ManifestFileMeta> metas) {
    return super.writeWithoutRolling(metas.iterator());
    // 返回 (文件名, 文件大小)
}
```

ManifestList 写入时不使用滚动写入（rolling write），因为一个 ManifestList 文件通常不会太大（只包含 ManifestFileMeta 的元数据，不包含实际数据条目）。

### 6.3 ManifestFileMeta 字段详解

> 源码: `paimon-core/src/main/java/org/apache/paimon/manifest/ManifestFileMeta.java`

ManifestFileMeta 是 ManifestFile 的"摘要信息"，用于快速过滤不需要读取的 ManifestFile:

| 字段 | 类型 | 说明 |
|---|---|---|
| `fileName` | `String` | ManifestFile 的文件名 |
| `fileSize` | `long` | 文件大小（字节） |
| `numAddedFiles` | `long` | 包含的 ADD 类型 entry 数量 |
| `numDeletedFiles` | `long` | 包含的 DELETE 类型 entry 数量 |
| `partitionStats` | `SimpleStats` | 分区字段的统计信息（min/max/null count） |
| `schemaId` | `long` | 写入时的 Schema ID |
| `minBucket` | `Integer` (nullable) | 最小桶编号 |
| `maxBucket` | `Integer` (nullable) | 最大桶编号 |
| `minLevel` | `Integer` (nullable) | 最小 LSM level |
| `maxLevel` | `Integer` (nullable) | 最大 LSM level |
| `minRowId` | `Long` (nullable) | 最小行 ID（Row Tracking） |
| `maxRowId` | `Long` (nullable) | 最大行 ID |

**过滤优化的核心:** `partitionStats` 字段记录了该 ManifestFile 中所有 entry 的分区字段 min/max。在查询时，可以根据分区过滤条件快速跳过不相关的 ManifestFile，避免读取其内容。类似地，`minBucket`/`maxBucket` 和 `minLevel`/`maxLevel` 也支持桶级和 level 级的过滤。

**为什么 minBucket/maxBucket 等字段可空?** 这些是后来添加的优化字段。为了向后兼容旧版本生成的 ManifestFileMeta，这些字段设计为可空。当为 null 时，无法进行对应维度的过滤，但不影响正确性。

### 6.4 ManifestFile 与 ManifestEntry

> 源码:
> - `paimon-core/src/main/java/org/apache/paimon/manifest/ManifestFile.java`
> - `paimon-core/src/main/java/org/apache/paimon/manifest/ManifestEntry.java`
> - `paimon-core/src/main/java/org/apache/paimon/manifest/FileKind.java`

**ManifestFile** 继承自 `ObjectsFile<ManifestEntry>`，包含多个 `ManifestEntry`。它支持滚动写入（rolling write），当文件大小达到阈值时自动滚动到新文件。

**ManifestEntry** 表示一个数据文件的添加或删除操作:

| 字段 | 类型 | 说明 |
|---|---|---|
| `_KIND` | `TinyInt` (FileKind) | ADD=0 表示添加文件, DELETE=1 表示删除文件 |
| `_PARTITION` | `bytes` (BinaryRow) | 分区值 |
| `_BUCKET` | `int` | 桶编号 |
| `_TOTAL_BUCKETS` | `int` | 总桶数 |
| `_FILE` | `DataFileMeta` | 数据文件的完整元数据 |

**FileKind 枚举:**

```java
public enum FileKind {
    ADD((byte) 0),     // 文件添加
    DELETE((byte) 1);  // 文件删除
}
```

**ADD/DELETE 的语义:** ManifestEntry 并不直接修改文件系统上的数据文件。它们是逻辑操作记录:
- `ADD`: 表示一个新的数据文件被添加到表中
- `DELETE`: 表示一个已有的数据文件从逻辑上被移除（物理文件在 snapshot 过期时才会被清理）

通过对所有 ManifestEntry 进行合并（merge），可以得到表在某个 snapshot 时的完整有效文件列表: 同一个文件的最后一个 entry 决定其状态。

### 6.5 Manifest 写入与读取流程

**写入流程（在 FileStoreCommitImpl.tryCommitOnce 中）:**

```
1. 收集变更 (collectChanges)
   CommitMessage[] --> ManifestEntryChanges
   分离为: appendTableFiles, compactTableFiles, appendChangelog, compactChangelog

2. 合并旧的 base manifest
   mergeBeforeManifests = manifestList.readDataManifests(latestSnapshot)
   mergeAfterManifests = ManifestFileMerger.merge(
       mergeBeforeManifests,
       manifestFile,
       targetSize,          // 目标文件大小
       mergeMinCount,       // 最小合并文件数
       fullCompactionSize,  // 全量压缩阈值
       partitionType,
       parallelism)
   baseManifestList = manifestList.write(mergeAfterManifests)

3. 写入增量 delta
   deltaManifestList = manifestList.write(manifestFile.write(deltaFiles))

4. 写入 changelog（如有）
   changelogManifestList = manifestList.write(manifestFile.write(changelogFiles))

5. 写入 index manifest（如有）
   indexManifest = indexManifestFile.writeIndexFiles(oldIndexManifest, indexFiles)
```

**读取流程（在 FileStoreScan 中）:**

```
1. 读取 Snapshot（从 SnapshotManager）

2. 通过 ManifestList 读取 ManifestFileMeta 列表
   List<ManifestFileMeta> metas = manifestList.readDataManifests(snapshot)

3. 利用 ManifestFileMeta 的 partitionStats 过滤不相关的 manifest
   metas = metas.stream()
       .filter(meta -> partitionFilter.test(meta.partitionStats()))
       .collect(...)

4. 读取过滤后的 ManifestFile 获取 ManifestEntry 列表
   for (ManifestFileMeta meta : metas) {
       entries.addAll(manifestFile.read(meta.fileName(), meta.fileSize()))
   }

5. 合并 entries (FileEntry.mergeEntries)
   得到有效的数据文件列表
```

**Manifest 合并（ManifestFileMerger）的时机和策略:** 每次提交时都会尝试合并旧的 base manifest 文件。合并规则:
- 当 ManifestFile 中的 DELETE entries 过多（已删除文件的比例高）时，合并可以消除这些无效记录
- 当小的 ManifestFile 数量超过阈值时，合并可以减少后续读取的 IO 次数
- 合并结果的目标文件大小由 `manifest.target-file-size` 控制

---

## 七、Commit 流程

### 7.1 FileStoreCommitImpl.commit() 完整流程

> 源码: `paimon-core/src/main/java/org/apache/paimon/operation/FileStoreCommitImpl.java`

`commit(ManifestCommittable committable, boolean checkAppendFiles)` 是 Paimon 数据写入的最终汇聚点。其完整流程如下:

```
commit(committable, checkAppendFiles)
  |
  +-- [1] collectChanges(committable.fileCommittables())
  |     |-- 遍历所有 CommitMessage
  |     |-- 分类到 ManifestEntryChanges:
  |     |     appendTableFiles    (新增数据文件)
  |     |     compactTableFiles   (压缩产生的文件变更)
  |     |     appendChangelog     (追加 changelog)
  |     |     compactChangelog    (压缩产生的 changelog)
  |     |     appendIndexFiles   (新增索引文件)
  |     |     compactIndexFiles  (压缩产生的索引变更)
  |
  +-- [2] 判断是否需要 APPEND 提交
  |     |-- 条件: appendTableFiles/appendChangelog/appendIndexFiles 非空
  |     |-- 确定 commitKind = APPEND
  |     |-- 特殊情况: 如果 appendFiles 中包含 DELETE 或 DV
  |     |     +-- 升级为 commitKind = OVERWRITE, 允许回滚
  |     |
  |     +-- tryCommit(appendFiles, APPEND, ...)  --------> [详见 7.2]
  |     |     generatedSnapshot += 1
  |
  +-- [3] 判断是否需要 COMPACT 提交
  |     |-- 条件: compactTableFiles/compactChangelog/compactIndexFiles 非空
  |     +-- tryCommit(compactFiles, COMPACT, ...)
  |           generatedSnapshot += 1
  |
  +-- [4] 报告指标 (commitMetrics)
  |
  +-- return generatedSnapshot  // 可能是 0, 1, 或 2
```

**关键设计: APPEND 和 COMPACT 分离为两次独立提交.** 一次 `commit()` 调用可能产生 0 到 2 个 Snapshot:
- 如果只有追加数据: 1 个 APPEND snapshot
- 如果只有压缩: 1 个 COMPACT snapshot
- 如果两者都有: 2 个 snapshot（先 APPEND 再 COMPACT）
- 如果全部为空且 ignoreEmptyCommit=true: 0 个 snapshot

### 7.2 tryCommit 乐观锁重试机制

> 源码: `FileStoreCommitImpl.java` L683-732

```java
private int tryCommit(
        CommitChangesProvider changesProvider,
        long identifier, Long watermark, Map<String,String> properties,
        CommitKind commitKind, boolean allowRollback,
        boolean detectConflicts, String statsFileName) {
    int retryCount = 0;
    RetryCommitResult retryResult = null;
    long startMillis = System.currentTimeMillis();

    while (true) {
        // [1] 获取最新快照
        Snapshot latestSnapshot = snapshotManager.latestSnapshot();

        // [2] 获取变更内容（可能随 latestSnapshot 变化而不同，如 OVERWRITE）
        CommitChanges changes = changesProvider.provide(latestSnapshot);

        // [3] 尝试单次提交
        CommitResult result = tryCommitOnce(
                retryResult, changes.tableFiles, changes.changelogFiles,
                changes.indexFiles, identifier, watermark, properties,
                commitKind, allowRollback, latestSnapshot,
                detectConflicts, statsFileName);

        // [4] 成功则退出
        if (result.isSuccess()) break;

        // [5] 失败检查超时和最大重试次数
        retryResult = (RetryCommitResult) result;
        if (System.currentTimeMillis() - startMillis > options.commitTimeout()
                || retryCount >= options.commitMaxRetries()) {
            throw new RuntimeException("Commit failed after " + ... + " retries");
        }

        // [6] 等待后重试（随机退避）
        retryWaiter.retryWait(retryCount);
        retryCount++;
    }
    return retryCount + 1; // 返回总尝试次数
}
```

**RetryWaiter 退避策略:** 使用配置的 `commit.min-retry-wait` 和 `commit.max-retry-wait` 之间的随机时间进行退避，避免多个提交者同时重试造成惊群效应。

**为什么用 `while(true)` 而不是固定次数?** 因为超时和重试次数是两个独立的限制条件。在流式处理场景下，可能需要在较长时间内持续重试（如等待其他 compaction 完成），但同时也不能无限重试。

### 7.3 tryCommitOnce 单次提交详解

> 源码: `FileStoreCommitImpl.java` L765-1046

`tryCommitOnce` 是一次完整的提交尝试:

```
tryCommitOnce(retryResult, deltaFiles, changelogFiles, indexFiles,
              identifier, watermark, properties, commitKind,
              allowRollback, latestSnapshot, detectConflicts, statsFileName)
  |
  +-- [1] 检查提交是否已完成（去重）
  |     |-- 如果是 CommitFailRetryResult 且 latestSnapshot 不为 null
  |     |-- 扫描 [上次snapshot.id+1 ... latestSnapshot.id] 范围的所有 snapshot
  |     +-- 如果找到 commitUser + identifier + commitKind 匹配的 snapshot
  |           return SuccessCommitResult  // 提交已经成功了，无需再提交
  |
  +-- [2] 确定 newSnapshotId = latestSnapshot.id + 1 (或 1 如果无历史)
  |
  +-- [3] 严格模式检查 (strictModeChecker)
  |
  +-- [4] 冲突检测（如果 detectConflicts = true）
  |     |-- 读取已变更分区的所有文件 (baseDataFiles)
  |     |-- 利用增量读取优化（如果是重试，从上次的 baseDataFiles 开始增量更新）
  |     |-- 丢弃重复文件（discardDuplicate 选项）
  |     |-- conflictDetection.checkConflicts(latestSnapshot, baseDataFiles, deltaFiles, ...)
  |     |     |
  |     |     +-- [详见 7.4]
  |     |-- 如果冲突 + 允许回滚:
  |     |     rollback.tryToRollback(latestSnapshot) → RetryCommitResult
  |     |-- 如果冲突 + 不允许回滚: throw RuntimeException
  |
  +-- [5] 构建新 Snapshot
  |     |-- 读取并合并旧 base manifest → 写入新 baseManifestList
  |     |-- 写入 deltaManifestList
  |     |-- 写入 changelogManifestList（如有）
  |     |-- 写入 indexManifest（如有）
  |     |-- 处理 watermark 合并（取当前和历史的最大值）
  |     |-- Row Tracking 行 ID 分配
  |     |-- 计算 totalRecordCount, deltaRecordCount
  |     |-- 构造 Snapshot 对象
  |
  +-- [6] 触发 commitPreCallbacks
  |
  +-- [7] 原子提交 Snapshot
  |     |-- success = commitSnapshotImpl(newSnapshot, deltaStatistics)
  |     |-- 内部: snapshotCommit.commit(snapshot, ...) 或 fileIO.tryToWriteAtomic(...)
  |
  +-- [8] 处理结果
        |-- 成功: 触发 commitCallbacks, return SuccessCommitResult
        |-- 失败 (AtomicRename 返回 false): return RetryCommitResult(CommitFail)
        |-- 异常: return RetryCommitResult(CommitFail + exception)
```

**去重检测的核心:** 步骤 [1] 中的去重逻辑确保了 exactly-once 语义。如果由于网络超时等原因，客户端不确定上次提交是否成功，在重试时会先检查上次的提交是否已经被持久化。判断条件是三元组 `(commitUser, commitIdentifier, commitKind)` 的匹配。

**增量冲突检测优化:** 步骤 [4] 中，如果是重试（retryResult != null），不会重新读取所有 base 数据文件，而是在上次读取的基础上增量更新:

```java
if (commitFailRetry != null && commitFailRetry.baseDataFiles != null) {
    baseDataFiles = new ArrayList<>(commitFailRetry.baseDataFiles);
    List<SimpleFileEntry> incremental = scanner.readIncrementalChanges(
            commitFailRetry.latestSnapshot, latestSnapshot, changedPartitions);
    baseDataFiles.addAll(incremental);
    baseDataFiles = new ArrayList<>(FileEntry.mergeEntries(baseDataFiles));
}
```

### 7.4 冲突检测 ConflictDetection

> 源码: `paimon-core/src/main/java/org/apache/paimon/operation/commit/ConflictDetection.java`

ConflictDetection 在提交时检测与已有数据的冲突:

```java
public Optional<RuntimeException> checkConflicts(
        Snapshot latestSnapshot,
        List<SimpleFileEntry> baseEntries,
        List<SimpleFileEntry> deltaEntries,
        List<IndexManifestEntry> deltaIndexEntries,
        CommitKind commitKind) {
    // [1] DV 场景下的 entry 增强（enrichment）
    //     为 base 和 delta 的 entry 附加 DV 信息
    if (deletionVectorsEnabled && bucketMode == BUCKET_UNAWARE) {
        baseEntries = buildBaseEntriesWithDV(...);
        deltaEntries = buildDeltaEntriesWithDV(...);
    }

    // [2] 桶数一致性检查
    //     同一分区内的所有文件必须具有相同的 totalBuckets
    checkBucketKeepSame(baseEntries, deltaEntries, commitKind, ...);

    // [3] Delta 内部一致性检查
    //     不允许对同一个文件同时 ADD 和 DELETE
    FileEntry.mergeEntries(deltaEntries);

    // [4] Base + Delta 合并检查
    //     验证要删除的文件确实存在于 base 中
    mergedEntries = FileEntry.mergeEntries(allEntries);

    // [5] 合并后检查删除标记
    //     确保合并后没有残留的 DELETE entry（即被删除的文件确实存在）
    checkDeleteInEntries(mergedEntries, conflictException);

    // [6] Key Range 重叠检查
    //     对于主键表，检查 delta 文件的 key range 是否与 base 文件冲突
    checkKeyRange(baseEntries, deltaEntries, mergedEntries, baseCommitUser);

    // [7] Row ID 范围冲突检查
    checkRowIdRangeConflicts(commitKind, mergedEntries);

    // [8] Row ID 从特定 Snapshot 开始的检查
    checkForRowIdFromSnapshot(latestSnapshot, deltaEntries, deltaIndexEntries);
}
```

**Key Range 检查的含义:** 对于主键表，每个数据文件记录了其包含的 key 范围（minKey/maxKey）。如果两个并发写入产生了 key 范围重叠的文件，可能导致同一个 key 出现在多个文件中，违反主键唯一性约束。

**为什么 OVERWRITE 提交跳过桶数一致性检查?** 因为 OVERWRITE 会替换整个分区的数据，新数据的桶数可能与旧数据不同（例如表配置变更后的重写），这是合法的。

### 7.5 APPEND/COMPACT 分离提交

在 `commit()` 方法中，APPEND 和 COMPACT 被分为两次独立的 `tryCommit()` 调用:

```java
// 第一次提交: APPEND（新数据）
if (!appendTableFiles.isEmpty() || !appendChangelog.isEmpty() || !appendIndexFiles.isEmpty()) {
    tryCommit(
        CommitChangesProvider.provider(appendTableFiles, appendChangelog, appendIndexFiles),
        committable.identifier(), committable.watermark(), committable.properties(),
        CommitKind.APPEND, allowRollback, checkAppendFiles, null);
    generatedSnapshot += 1;
}

// 第二次提交: COMPACT（压缩结果）
if (!compactTableFiles.isEmpty() || !compactChangelog.isEmpty() || !compactIndexFiles.isEmpty()) {
    tryCommit(
        CommitChangesProvider.provider(compactTableFiles, compactChangelog, compactIndexFiles),
        committable.identifier(), committable.watermark(), committable.properties(),
        CommitKind.COMPACT, false, true, null);
    generatedSnapshot += 1;
}
```

**为什么分离?**

1. **语义清晰**: APPEND 表示新数据的到来，COMPACT 表示物理存储的优化。将它们分为不同的 snapshot 让流式消费者可以精确控制只消费 APPEND 类型的变更。

2. **冲突隔离**: APPEND 提交可能需要回滚（当检测到冲突时），但 COMPACT 提交不需要。分离后可以独立处理各自的错误恢复逻辑。

3. **特殊 APPEND → OVERWRITE 升级**: 当 APPEND 的变更中包含 DELETE 操作或 Deletion Vector 时，会自动升级为 OVERWRITE 提交，启用更严格的冲突检测和回滚机制。这个升级逻辑不应影响 COMPACT 提交。

### 7.6 Overwrite 分区覆写

> 源码: `FileStoreCommitImpl.java` L411-538

`overwritePartition()` 方法处理分区覆写逻辑:

```
overwritePartition(partition, committable, properties)
  |
  +-- collectChanges(committable.fileCommittables())
  |
  +-- 确定 partitionFilter:
  |     |-- 动态分区覆写 (dynamicPartitionOverwrite=true):
  |     |     从 appendTableFiles 中提取实际写入的分区
  |     |-- 静态分区覆写:
  |     |     根据指定的 partition map 构建 PartitionPredicate
  |     |     + 校验所有变更文件都属于指定分区
  |
  +-- tryUpgrade (overwriteUpgrade=true 时)
  |     如果 overwrite 的文件没有 key range 重叠，升级到最高 level
  |
  +-- tryOverwritePartition(partitionFilter, ...)
  |     |-- 内部调用 scanner.readOverwriteChanges()
  |     |     读取匹配分区的所有已有文件，生成 DELETE entries
  |     |     合并 DELETE entries + 新的 ADD entries
  |     +-- tryCommit(..., CommitKind.OVERWRITE, ...)
  |
  +-- COMPACT 提交（如果有压缩变更）
```

**动态 vs 静态分区覆写:**
- **静态覆写**: 用户明确指定要覆写的分区（如 `INSERT OVERWRITE PARTITION(dt='2024-01-01')`）。所有变更文件必须属于指定分区，否则抛异常。
- **动态覆写**: 系统根据实际写入的数据自动确定要覆写的分区。如果没有写入任何数据，则跳过覆写（不删除任何文件）。

---

## 八、CoreOptions 配置体系

> 源码: `paimon-api/src/main/java/org/apache/paimon/CoreOptions.java`

CoreOptions 是 Paimon 的配置中心，定义了所有表级配置项。与 Catalog 和元数据管理相关的配置:

**Commit 相关:**

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `commit.timeout` | `5min` | 提交超时时间 |
| `commit.max-retries` | `∞` | 最大重试次数 |
| `commit.min-retry-wait` | `1s` | 最小重试等待 |
| `commit.max-retry-wait` | `5s` | 最大重试等待 |
| `commit.discard-duplicate-files` | `false` | 是否丢弃重复文件 |

**Manifest 相关:**

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `manifest.target-file-size` | `8MB` | Manifest 文件目标大小 |
| `manifest.merge-min-count` | `30` | 触发 manifest 合并的最小文件数 |
| `manifest.full-compaction-threshold-size` | - | 全量压缩阈值 |

**Snapshot/Tag 相关:**

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `snapshot.time-retained` | - | Snapshot 保留时间 |
| `snapshot.num-retained.min` | `10` | 最少保留的 Snapshot 数 |
| `snapshot.num-retained.max` | `∞` | 最多保留的 Snapshot 数 |
| `tag.automatic-creation` | `none` | 自动创建 Tag 的模式 |
| `tag.creation-period` | - | 自动 Tag 创建周期 |
| `tag.default-time-retained` | - | Tag 默认保留时间 |

**Schema 相关:**

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `bucket` | `-1` | 桶数（-1=动态） |
| `bucket-key` | - | 桶键（为空则使用主键） |
| `sequence.field` | - | 序列字段 |
| `partition.default-name` | `__DEFAULT_PARTITION__` | 默认分区名 |

**Cache 相关（CatalogOptions）:**

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `cache.enabled` | `true` | 启用 Catalog 缓存 |
| `cache.expire-after-access` | - | 访问后过期时间 |
| `cache.expire-after-write` | - | 写入后过期时间 |
| `cache.manifest-small-file-memory` | - | 小 manifest 缓存内存 |
| `cache.manifest-max-memory` | - | manifest 最大缓存内存 |
| `cache.partition-max-num` | `0` | 分区缓存最大数量 |

配置项通过 `CoreOptions.fromMap(options)` 工厂方法创建，内部使用 Paimon 自定义的 `Options` 类进行类型安全的配置读取:

```java
public class CoreOptions implements Serializable {
    private final Options options;

    public CoreOptions(Map<String, String> options) {
        this(Options.fromMap(options));
    }

    public int bucket() {
        return options.get(BUCKET);
    }

    // ... 其他配置的 getter
}
```

---

## 九、与 Iceberg 的设计对比

### 9.1 Snapshot 设计对比

| 维度 | Paimon | Iceberg |
|---|---|---|
| **快照文件格式** | JSON 文件（`snapshot-{id}`） | 独立的 snap-{id}.avro 文件 |
| **Manifest 层级** | 三级: Snapshot → ManifestList → ManifestFile → DataFile | 三级: Snapshot → ManifestList → ManifestFile → DataFile |
| **增量追踪** | base/delta/changelog 三分法 | 每个 manifest 文件有 `added_snapshot_id` 标记所属的快照 |
| **Changelog 支持** | 原生支持 changelogManifestList | 无内建 changelog，需要对比两个 snapshot 计算 diff |
| **版本号** | 全局单调递增 ID (`snapshot-1`, `snapshot-2`) | 全局单调递增序列号 + UUID |
| **提交类型** | CommitKind: APPEND/COMPACT/OVERWRITE/ANALYZE | operation: append/overwrite/replace/delete |
| **统计信息** | 独立的 statistics 文件引用 | 内嵌在 manifest file 中 |
| **Watermark** | 原生支持 watermark 字段 | 无内建 watermark |

**Paimon 的 delta 优势:** Paimon 的 base/delta 分离设计使得增量消费非常高效——流式读取器只需读取 delta manifest 即可获取变更，无需比对全量。Iceberg 需要通过 `added_snapshot_id` 过滤或对比两个版本的 manifest list 来实现类似功能。

**Paimon 的 changelog 优势:** Paimon 原生支持 changelog 文件，可以在写入时直接产生 CDC 格式的变更记录。Iceberg 需要通过 "equality delete files" 或 "position delete files" 的增量读取来间接实现 CDC，语义表达不如 Paimon 直接。

### 9.2 Catalog 设计对比

| 维度 | Paimon | Iceberg |
|---|---|---|
| **默认 Catalog** | FileSystemCatalog（文件系统即元数据） | HadoopCatalog / RESTCatalog |
| **SPI 机制** | CatalogFactory（Java SPI） | CatalogUtil + catalog-impl 配置 |
| **缓存策略** | CachingCatalog（装饰器模式，Caffeine） | CachingCatalog（类似装饰器，Caffeine） |
| **锁机制** | CatalogLockFactory SPI（可选） | LockManager（写入内置锁） |
| **Schema 存储** | 独立 schema-{id} 文件，每次变更新增文件 | 嵌入在 table metadata v{version}.metadata.json 中 |
| **分支支持** | 原生 BranchManager，文件系统级实现 | 通过 Ref（Iceberg 1.5+）, 需要 Catalog 级支持 |
| **Tag 支持** | 原生 TagManager，Tag 即 Snapshot 副本 | 通过 Ref 引用 Snapshot |

**Schema 演进对比:**
- Paimon: 每次 schema 变更写入新的 `schema-{id}` 文件。旧数据文件通过 `schemaId` 关联读取时需要的 schema，在 `SchemaEvolutionUtil` 中进行列映射和类型转换。
- Iceberg: schema 变更嵌入在 `metadata.json` 文件中，每次变更重写整个 metadata 文件。通过 `schema-id` 字段追踪版本。

**Paimon 独立 schema 文件的好处:**
1. 不需要重写整个 metadata 文件，变更操作更轻量
2. 乐观锁基于文件原子创建实现，不需要读取-修改-写入（RMW）操作
3. schema 历史天然可追溯，每个版本独立存在

**Iceberg 合并 metadata 的好处:**
1. 所有元数据在一个文件中，读取时不需要多次 IO
2. metadata 文件中包含 schema、partition spec、sort order 等完整信息

### 9.3 并发控制对比

| 维度 | Paimon | Iceberg |
|---|---|---|
| **写入并发控制** | 乐观锁（原子文件创建） + 可选 CatalogLock | 乐观锁（CAS on metadata.json）+ 可选 LockManager |
| **冲突检测粒度** | 分区级 + 桶级 + Key Range 级 | 分区级 + 文件级 |
| **重试策略** | 随机退避 + 超时限制 | 固定次数重试 |
| **Exactly-Once** | commitUser + commitIdentifier 去重 | snapshot sequence number 去重 |

**Paimon 的 Key Range 冲突检测更精细:** Paimon 会检查新写入文件的 key range 是否与现有文件重叠。这对于 LSM-tree 的正确性至关重要——同一个 bucket 内不应该在同一个 level 出现 key range 重叠的文件。Iceberg 主要关注分区级别的冲突。

### 9.4 总结

Paimon 和 Iceberg 在 Catalog 与元数据管理层面采用了相似的分层架构（Snapshot → ManifestList → ManifestFile），但在细节设计上各有侧重:

- **Paimon 更侧重流式场景**: 原生的 changelog 支持、watermark 追踪、delta 增量读取，使其在实时数据湖场景下具有优势。
- **Paimon 更轻量**: 独立的 schema 文件、文件系统即元数据的设计，使得部署和运维成本更低，不强制依赖外部元数据服务。
- **Iceberg 更成熟**: 在事务语义、多引擎兼容性、SQL 标准支持等方面积累更深。
- **两者都在演进**: Paimon 新增了 REST Catalog 支持，Iceberg 也在增强流式能力，设计差异在逐步缩小。
