# Apache Paimon Catalog 与元数据管理深度分析

> 基于 Paimon 1.5-SNAPSHOT 源码分析，commit: 55f4fd175
> 分析日期: 2026-04-21

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

### 解决什么问题

**核心业务问题：**
1. **统一的元数据管理入口**：数据湖需要管理数据库、表、分区、快照、标签等多种元数据对象，如果没有统一的抽象层，每个计算引擎（Flink/Spark/Hive）都需要实现自己的元数据管理逻辑，导致重复开发和不一致。
2. **多种存储后端的适配**：不同的部署场景需要不同的元数据存储方案——小规模场景可以直接用文件系统，企业场景需要集成 Hive Metastore，云原生场景需要 REST API。
3. **零外部依赖的部署**：传统数据湖（如 Hive）强依赖 MySQL/PostgreSQL 存储元数据，增加了部署复杂度和故障点。

**没有这个设计的后果：**
- 每个计算引擎需要自己实现表的创建、删除、Schema 演进逻辑，代码重复且容易出现不一致
- 无法在不同的元数据存储后端之间切换，被绑定到特定的基础设施
- 无法通过装饰器模式灵活地添加缓存、权限控制等横切关注点

**实际场景：**
```java
// 场景1: 开发环境使用本地文件系统
Map<String, String> options = new HashMap<>();
options.put("warehouse", "/tmp/paimon");
options.put("metastore", "filesystem");
Catalog catalog = CatalogFactory.createCatalog(CatalogContext.create(options));

// 场景2: 生产环境集成 Hive Metastore
options.put("metastore", "hive");
options.put("uri", "thrift://hive-metastore:9083");
Catalog catalog = CatalogFactory.createCatalog(CatalogContext.create(options));

// 场景3: 云原生环境使用 REST Catalog
options.put("metastore", "rest");
options.put("uri", "https://catalog-server.example.com");
Catalog catalog = CatalogFactory.createCatalog(CatalogContext.create(options));
```

### 有什么坑

**误区陷阱：**
1. **系统表不能直接 DDL**：尝试 `DROP TABLE my_table$snapshots` 会抛异常。系统表是虚拟的，只能通过原始表操作。
2. **分支表不能直接 DDL**：尝试 `DROP TABLE my_table$branch_dev` 会失败。分支需要通过 `BranchManager.dropBranch()` 删除。
3. **FileSystemCatalog 不支持 alterDatabase**：因为文件系统目录没有属性存储能力，尝试修改数据库属性会抛 `UnsupportedOperationException`。

**错误配置：**
```java
// 错误：忘记配置 warehouse
Map<String, String> options = new HashMap<>();
options.put("metastore", "filesystem");
// 缺少 warehouse 配置
Catalog catalog = CatalogFactory.createCatalog(...); // 抛异常

// 错误：在对象存储上使用 renameTable
options.put("warehouse", "s3://bucket/warehouse");
catalog.renameTable(from, to); // S3 的 rename 不是原子的，可能导致数据不一致
```

**生产环境注意事项：**
1. **对象存储必须启用锁机制**：S3/OSS 等对象存储的原子操作能力有限，必须配置 `CatalogLockFactory`（如基于 MySQL 的锁）来保证并发安全。
2. **缓存配置要合理**：`CachingCatalog` 默认启用，但如果多个进程共享同一个 warehouse，缓存过期时间不能太长，否则会读到过期的元数据。
3. **避免频繁的 invalidateTable**：每次调用都会清空缓存，导致下次访问需要重新读取文件系统，影响性能。

**性能陷阱：**
```java
// 陷阱：在循环中反复 getTable
for (String tableName : catalog.listTables(db)) {
    Table table = catalog.getTable(Identifier.create(db, tableName));
    // 如果缓存未启用，每次都会读取文件系统
}

// 优化：启用缓存并批量操作
options.put("cache.enabled", "true");
options.put("cache.expire-after-access", "10min");
```

### 核心概念解释

**Catalog（目录）：**
元数据管理的统一接口，类似于关系数据库中的 `INFORMATION_SCHEMA`。它管理数据库、表、分区、快照等所有元数据对象。

**FileSystemCatalog vs HiveCatalog：**
- **FileSystemCatalog**：将元数据直接存储为文件系统上的文件（JSON 格式），零外部依赖，适合小规模部署和开发环境。
- **HiveCatalog**：将元数据存储在 Hive Metastore 中，适合已有 Hive 基础设施的企业环境，可以与 Hive 表共存。

**装饰器模式（Decorator Pattern）：**
通过 `DelegateCatalog` 实现的设计模式，允许在不修改原始 Catalog 的情况下动态添加功能：
```
PrivilegedCatalog (权限控制)
  → CachingCatalog (缓存)
    → FileSystemCatalog (实际存储)
```

**SPI（Service Provider Interface）：**
Java 的插件机制，通过 `META-INF/services/org.apache.paimon.catalog.CatalogFactory` 文件声明实现类，运行时动态加载。这使得用户可以自定义 Catalog 实现而无需修改 Paimon 核心代码。

**Identifier（标识符）：**
Paimon 中表的完整标识，格式为 `{database}.{table}[$branch_{name}][$systemTable]`。例如：
- `my_db.my_table` - 普通表
- `my_db.my_table$snapshots` - 系统表
- `my_db.my_table$branch_dev` - 分支表
- `my_db.my_table$branch_dev$files` - 分支的系统表

### 设计理念

**为什么这样设计：**

1. **接口与实现分离**：`Catalog` 接口定义了"做什么"，`FileSystemCatalog`/`HiveCatalog` 定义了"怎么做"。这使得上层代码（Flink/Spark）不需要关心底层存储细节。

2. **模板方法模式**：`AbstractCatalog` 实现了通用逻辑（参数校验、系统表判断），子类只需实现 `xxxImpl()` 方法。这避免了每个实现都重复处理边界条件。

3. **装饰器链的灵活组合**：通过装饰器模式，可以在运行时动态组合功能：
   ```java
   // 开发环境：只需基础功能
   Catalog catalog = new FileSystemCatalog(...);
   
   // 生产环境：添加缓存和权限
   catalog = new CachingCatalog(catalog);
   catalog = new PrivilegedCatalog(catalog);
   ```

4. **能力探测而非异常处理**：通过 `supportsListObjectsPaged()`、`supportsVersionManagement()` 等方法，上层代码可以优雅降级，而不是捕获异常：
   ```java
   if (catalog.supportsListObjectsPaged()) {
       // 使用分页 API
   } else {
       // 回退到全量列表
   }
   ```

**权衡取舍：**

1. **FileSystemCatalog 的简单性 vs 功能限制**：
   - 优势：零外部依赖，部署简单，适合小规模场景
   - 劣势：不支持数据库属性、分页列表等高级功能

2. **乐观锁 vs 悲观锁**：
   - Paimon 选择乐观锁（基于文件原子创建），避免了分布式锁的复杂性
   - 代价是在高并发场景下可能需要多次重试

3. **缓存一致性 vs 性能**：
   - `CachingCatalog` 提升了读取性能，但在多进程场景下可能读到过期数据
   - 通过 `cache.expire-after-access` 配置平衡一致性和性能

**架构演进：**

Paimon 的 Catalog 设计经历了以下演进：
1. **v0.1-0.3**：只有 FileSystemCatalog，功能简单
2. **v0.4-0.6**：引入 HiveCatalog，支持企业级部署
3. **v0.7-0.9**：引入 CachingCatalog 装饰器，优化性能
4. **v1.0+**：引入 RESTCatalog，支持云原生架构

**业界对比：**

| 特性 | Paimon | Iceberg | Delta Lake |
|------|--------|---------|------------|
| 默认 Catalog | FileSystemCatalog | HadoopCatalog | 无独立 Catalog（依赖 Spark） |
| 零外部依赖 | ✅ | ✅ | ❌（需要 Spark Metastore） |
| Hive 集成 | ✅ | ✅ | ✅ |
| REST Catalog | ✅ | ✅ | ❌ |
| 装饰器模式 | ✅ | ✅ | ❌ |

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
    |-- paimon-core: rest/RESTCatalog (通过 REST API 与远端 Catalog Server 交互)
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
| `partitionCache` | `Cache<Identifier, List<Partition>>` | 分区列表缓存（可选，默认不启用） |
| `manifestCache` | `SegmentsCache<Path>` | Manifest 文件内容缓存 |
| `dvMetaCache` | `DVMetaCache` | Deletion Vector 元数据缓存（可选） |

**缓存配置项:**

| 配置键 | 默认值 | 说明 |
|---|---|---|
| `cache.enabled` | `true` | 是否启用缓存 |
| `cache.expire-after-access` | - | 访问后过期时间（必须为正数） |
| `cache.expire-after-write` | - | 写入后过期时间（必须为正数） |
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
    |     +-- 从 context 中读取 "metastore" 配置（默认 "filesystem"）
    |     +-- FactoryUtil.discoverFactory(classLoader, CatalogFactory.class, metastore)
    |     |     // 使用 Java SPI 发现匹配 identifier 的 CatalogFactory
    |     +-- 先尝试 catalogFactory.create(context)
    |     +-- 若 UnsupportedOperationException:
    |           +-- 读取 warehouse 路径（必须配置）
    |           +-- 创建 FileIO 并检查/创建 warehouse 目录
    |           +-- catalogFactory.create(fileIO, warehousePath, context)
    |
    +-- CachingCatalog.tryToCreate(catalog, options)  // 包装缓存层（如果启用）
    +-- PrivilegedCatalog.tryToCreate(catalog, options) // 包装权限层（如果启用）
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

### 解决什么问题

**核心业务问题：**
1. **表结构变更的向后兼容性**：业务需求变化时需要添加列、修改列类型、删除列，但已有的数据文件不能重写（成本太高），必须支持新旧 Schema 共存。
2. **并发 Schema 变更的安全性**：多个用户可能同时修改表结构（如添加不同的列），需要保证变更不会相互覆盖或产生冲突。
3. **Schema 历史的可追溯性**：需要知道每个数据文件是用哪个版本的 Schema 写入的，以便在读取时正确解析。

**没有这个设计的后果：**
- 每次 Schema 变更都需要重写所有历史数据，成本极高且不可行
- 并发变更会导致后提交的覆盖先提交的，丢失变更
- 无法读取旧版本的数据文件，或者读取时数据错乱

**实际场景：**
```java
// 场景1: 业务需要添加新字段
// 表中已有 1TB 数据，不可能重写
ALTER TABLE user_events ADD COLUMN user_age INT;

// 场景2: 两个团队同时修改 Schema
// 团队A: ALTER TABLE orders ADD COLUMN discount DECIMAL(10,2);
// 团队B: ALTER TABLE orders ADD COLUMN tax DECIMAL(10,2);
// 如果没有乐观锁，后提交的会覆盖先提交的

// 场景3: 读取历史数据
// 数据文件 file-1.parquet 用 schema-0 写入（3列）
// 数据文件 file-2.parquet 用 schema-1 写入（4列）
// 查询时需要自动对齐列，缺失的列填充 null
```

### 有什么坑

**误区陷阱：**
1. **新增列必须可空**：尝试 `ADD COLUMN age INT NOT NULL` 会失败。因为旧数据文件没有这一列，读取时只能填充 null。
   ```java
   // 错误
   SchemaChange.addColumn("age", DataTypes.INT(), null, false); // nullable=false 会抛异常
   
   // 正确
   SchemaChange.addColumn("age", DataTypes.INT(), null, true); // nullable=true
   ```

2. **不能删除主键或分区键**：尝试 `DROP COLUMN partition_date` 会失败，因为这会破坏表的分区结构。
   ```java
   // 错误：删除分区键
   SchemaChange.dropColumn("partition_date"); // 抛异常
   ```

3. **类型变更有严格限制**：只能进行"安全"的类型转换（如 INT → BIGINT），不能进行可能丢失精度的转换（如 BIGINT → INT）。
   ```java
   // 安全的类型变更
   SchemaChange.updateColumnType("age", DataTypes.BIGINT()); // INT → BIGINT ✅
   
   // 不安全的类型变更
   SchemaChange.updateColumnType("amount", DataTypes.INT()); // BIGINT → INT ❌
   ```

**错误配置：**
```java
// 错误：在高并发场景下未配置足够的重试次数
options.put("commit.max-retries", "1"); // 太少，容易失败
// 正确
options.put("commit.max-retries", "10");
```

**生产环境注意事项：**
1. **Schema 变更需要停写吗？** 不需要。Paimon 的乐观锁机制允许在写入过程中变更 Schema，但可能导致写入重试。
2. **Schema ID 会回收吗？** 不会。即使删除列，`highestFieldId` 也会持续增长，确保字段 ID 永不重复。
3. **Schema 文件会被清理吗？** 不会。所有历史 Schema 文件都会保留，因为旧的数据文件可能仍然引用它们。

**性能陷阱：**
```java
// 陷阱：频繁的 Schema 变更导致 schema 文件过多
for (int i = 0; i < 1000; i++) {
    schemaManager.commitChanges(SchemaChange.setOption("key" + i, "value"));
    // 每次都会生成新的 schema-{id} 文件
}

// 优化：批量提交变更
List<SchemaChange> changes = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
    changes.add(SchemaChange.setOption("key" + i, "value"));
}
schemaManager.commitChanges(changes); // 只生成一个新 schema 文件
```

### 核心概念解释

**Schema vs TableSchema：**
- **Schema**：用户提供的表结构定义（字段、主键、分区键、配置），不包含内部信息。
- **TableSchema**：持久化的完整表结构，包含 Schema ID、字段 ID、创建时间等内部信息。

**字段 ID（Field ID）：**
每个字段都有一个全局唯一的 ID，用于在 Schema 演进时追踪字段的身份。即使字段被重命名，ID 也不变。
```java
// schema-0: id=0, name="user_id", type=BIGINT
// schema-1: 重命名字段
// id=0, name="uid", type=BIGINT  // ID 不变，name 变了
```

**highestFieldId：**
已分配的最大字段 ID。删除字段后，ID 不会回收，新字段从 `highestFieldId + 1` 开始分配。这确保了字段 ID 的唯一性。

**SchemaChange：**
表示一次 Schema 变更操作的抽象，支持多态序列化（用于 REST API）。常见类型：
- `AddColumn` - 添加列
- `DropColumn` - 删除列
- `RenameColumn` - 重命名列
- `UpdateColumnType` - 修改列类型
- `SetOption` - 设置表配置
- `RemoveOption` - 删除表配置

**乐观锁（Optimistic Locking）：**
不使用分布式锁，而是通过"读取-计算-写入-检查"的方式实现并发控制：
1. 读取当前最新 Schema（如 schema-5）
2. 基于 schema-5 计算新 Schema（schema-6）
3. 尝试原子写入 schema-6
4. 如果写入失败（别人已经写了 schema-6），回到步骤 1 重试

### 设计理念

**为什么这样设计：**

1. **独立的 Schema 文件而非嵌入 Snapshot**：
   - 优势：Schema 变更不需要重写 Snapshot，操作更轻量
   - 优势：多个 Snapshot 可以共享同一个 Schema，节省存储
   - 劣势：读取时需要额外的 IO 获取 Schema 文件

2. **字段 ID 永不回收**：
   - 确保了字段身份的唯一性，即使字段被删除后重新添加同名字段，也会分配新的 ID
   - 避免了"删除列A → 添加列A"导致的数据混淆

3. **乐观锁而非悲观锁**：
   - 避免了分布式锁的复杂性和性能开销
   - 在低冲突场景下性能更好
   - 代价是高冲突场景下需要多次重试

4. **新增列必须可空**：
   - 这是向后兼容性的必然要求
   - 旧数据文件没有新列的数据，读取时只能填充 null
   - 如果允许 NOT NULL，会导致旧数据无法读取

**权衡取舍：**

1. **Schema 文件数量 vs 查询性能**：
   - 每次变更都生成新文件，导致 schema 目录下文件数量增长
   - 但查询时只需读取最新的 schema 文件，不影响性能
   - 历史 schema 文件用于读取旧数据文件

2. **类型变更的安全性 vs 灵活性**：
   - Paimon 只允许"安全"的类型转换（如 INT → BIGINT）
   - 这保证了数据不会丢失精度，但限制了灵活性
   - 如果需要不安全的转换，必须通过 `INSERT OVERWRITE` 重写数据

3. **并发性能 vs 一致性**：
   - 乐观锁在低冲突场景下性能好，但高冲突场景下会频繁重试
   - 可以通过配置 `commit.max-retries` 和 `commit.max-retry-wait` 调整

**架构演进：**

Paimon 的 Schema 演进机制经历了以下版本：
- **v1 (Paimon 0.4-0.6)**：基础的 Schema 演进，支持添加/删除列
- **v2 (Paimon 0.7-0.8)**：增加了字段 ID 追踪，支持列重命名
- **v3 (Paimon 0.9+)**：增加了嵌套字段的支持，支持复杂类型的 Schema 演进

**业界对比：**

| 特性 | Paimon | Iceberg | Delta Lake |
|------|--------|---------|------------|
| Schema 存储 | 独立文件 (schema-{id}) | 嵌入 metadata.json | 嵌入 _delta_log |
| 字段 ID | ✅ 永不回收 | ✅ 永不回收 | ❌ 无字段 ID |
| 并发控制 | 乐观锁（文件原子创建） | 乐观锁（CAS on metadata） | 乐观锁（日志追加） |
| 类型变更 | 严格限制（安全转换） | 严格限制 | 较宽松 |
| 嵌套字段 | ✅ 支持 | ✅ 支持 | ✅ 支持 |

### 3.1 TableSchema 完整字段

> 源码: `paimon-api/src/main/java/org/apache/paimon/schema/TableSchema.java`

TableSchema 是表的完整元数据定义，比用户提供的 Schema 多了 ID、版本等内部信息:

| 字段 | 类型 | 说明 |
|---|---|---|
| `version` | `int` | Schema 格式版本（当前 v3，Paimon 0.4-0.6=v1, 0.7-0.8=v2, 0.9+=v3） |
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

### 解决什么问题

**核心业务问题：**
1. **时间旅行（Time Travel）**：用户需要查询历史某个时间点的数据，如"查询昨天下午3点的订单表"。
2. **增量消费**：流式处理需要知道"从上次消费到现在新增了哪些数据"，而不是每次都扫描全表。
3. **数据回滚**：发现数据错误后需要回滚到之前的正确状态，如"回滚到1小时前的快照"。
4. **精确一次语义（Exactly-Once）**：分布式写入场景下，需要防止同一批数据被重复提交。

**没有这个设计的后果：**
- 无法查询历史数据，只能看到最新状态
- 增量消费需要对比两次全量扫描的结果，效率极低
- 数据错误后无法恢复，只能从备份重新导入
- 分布式写入可能产生重复数据，破坏数据一致性

**实际场景：**
```sql
-- 场景1: 时间旅行查询
SELECT * FROM orders /*+ OPTIONS('scan.timestamp-millis'='1704067200000') */;

-- 场景2: 增量消费
SELECT * FROM orders /*+ OPTIONS('scan.mode'='from-snapshot', 'scan.snapshot-id'='100') */;

-- 场景3: 数据回滚
CALL sys.rollback_to('my_db.orders', 95); -- 回滚到 snapshot-95

-- 场景4: 精确一次写入
// Flink checkpoint 100 提交了 snapshot-50
// 如果 checkpoint 100 失败重启，重新提交时会检测到 snapshot-50 已存在，跳过提交
```

### 有什么坑

**误区陷阱：**
1. **Snapshot 不是数据备份**：Snapshot 只是元数据的快照，底层数据文件是共享的。删除 Snapshot 不会立即删除数据文件，但过期后会清理。
   ```java
   // 误区：以为删除 snapshot 就删除了数据
   snapshotManager.deleteSnapshot(100); // 只删除 snapshot-100 文件
   // 数据文件仍然存在，直到没有任何 snapshot 引用它们
   ```

2. **commitIdentifier 不是全局唯一的**：它只在同一个 `commitUser` 内唯一。不同的 commitUser 可以有相同的 commitIdentifier。
   ```java
   // 正确的去重判断
   boolean isDuplicate = (snapshot.commitUser().equals(myUser) 
                          && snapshot.commitIdentifier() == myIdentifier
                          && snapshot.commitKind() == myKind);
   ```

3. **Snapshot 过期配置要谨慎**：如果 `snapshot.time-retained` 设置太短，可能导致正在运行的长查询失败（查询的 Snapshot 被删除了）。
   ```java
   // 危险配置
   options.put("snapshot.time-retained", "5min"); // 太短
   // 如果有查询运行超过5分钟，可能读取失败
   
   // 安全配置
   options.put("snapshot.time-retained", "1h");
   options.put("snapshot.num-retained.min", "10"); // 至少保留10个
   ```

**错误配置：**
```java
// 错误：只配置了 time-retained，没有配置 num-retained.min
options.put("snapshot.time-retained", "10min");
// 如果10分钟内提交了100个 snapshot，全部会被保留，占用大量空间

// 正确：同时配置时间和数量限制
options.put("snapshot.time-retained", "1h");
options.put("snapshot.num-retained.min", "10");
options.put("snapshot.num-retained.max", "100");
```

**生产环境注意事项：**
1. **LATEST hint 文件可能不准确**：在高并发写入场景下，hint 文件可能指向一个不是最新的 Snapshot。Paimon 会自动验证并回退到目录扫描。
2. **Snapshot 过期是异步的**：配置 `snapshot.time-retained` 后，过期的 Snapshot 不会立即删除，需要等待后台任务执行。
3. **Watermark 只增不减**：即使回滚到旧 Snapshot，watermark 也不会回退，这是为了保证流式处理的正确性。

**性能陷阱：**
```java
// 陷阱：频繁调用 latestSnapshot() 导致大量文件系统扫描
for (int i = 0; i < 1000; i++) {
    Snapshot snapshot = snapshotManager.latestSnapshot();
    // 如果 LATEST hint 失效，每次都会扫描目录
}

// 优化：使用缓存
Snapshot snapshot = snapshotManager.latestSnapshot(); // 只查询一次
for (int i = 0; i < 1000; i++) {
    // 使用缓存的 snapshot
}
```

### 核心概念解释

**Snapshot（快照）：**
表在某个时间点的完整数据视图入口。它不包含实际数据，只包含指向数据文件的元数据（通过 ManifestList）。

**Snapshot ID：**
全局单调递增的快照编号，从 1 开始。每次提交都会生成新的 Snapshot ID。

**commitUser + commitIdentifier：**
用于精确一次语义的去重机制：
- `commitUser`：提交者的唯一标识（如 Flink job ID）
- `commitIdentifier`：提交者内部的单调递增序列号（如 Flink checkpoint ID）
- 组合起来可以唯一标识一次提交

**CommitKind（提交类型）：**
- `APPEND`：追加新数据，不删除已有文件
- `COMPACT`：压缩合并，不改变逻辑数据
- `OVERWRITE`：覆写分区或删除文件
- `ANALYZE`：收集统计信息

**base/delta/changelog 三分法：**
- `baseManifestList`：截至本次提交的所有有效数据文件（合并后的全量）
- `deltaManifestList`：本次提交的增量变更（新增/删除的文件）
- `changelogManifestList`：CDC 变更日志文件（可选）

**Watermark（水位线）：**
流式处理中的事件时间进度标记。Snapshot 记录了提交时的 watermark，用于流式读取的断点续传。

**LATEST hint 文件：**
位于 `{snapshot_dir}/LATEST` 的提示文件，记录最新的 Snapshot ID，用于快速定位，避免每次都扫描目录。

### 设计理念

**为什么这样设计：**

1. **base/delta 分离的核心价值**：
   - 全量查询只需读取 base，性能高
   - 增量消费只需读取 delta，避免对比全量
   - Snapshot 过期时只需扫描 delta 即可知道哪些文件可以删除
   ```java
   // 全量读取
   List<ManifestFileMeta> files = manifestList.readDataManifests(snapshot);
   // 只读取 base + delta
   
   // 增量读取
   List<ManifestFileMeta> delta = manifestList.readDeltaManifests(snapshot);
   // 只读取 delta，效率高
   ```

2. **Snapshot 只是元数据，不是数据副本**：
   - 多个 Snapshot 共享底层数据文件，节省存储空间
   - 创建 Snapshot 是 O(1) 操作，只需写入一个小的 JSON 文件
   - 代价是需要引用计数机制来管理数据文件的生命周期

3. **commitIdentifier 的精确一次语义**：
   - 分布式写入场景下，同一批数据可能因为重试被提交多次
   - 通过 `(commitUser, commitIdentifier, commitKind)` 三元组去重
   - 如果发现已存在相同的 Snapshot，直接返回成功，不重复提交

4. **LATEST hint 文件的优化**：
   - 避免每次都扫描目录（对象存储上的 list 操作很慢）
   - 但 hint 可能不准确（并发写入、缓存延迟），需要验证
   - 验证方法：检查 hint 指向的 Snapshot ID + 1 是否存在

**权衡取舍：**

1. **Snapshot 数量 vs 存储空间**：
   - 每个 Snapshot 都是一个文件，过多会占用空间和 inode
   - 通过 `snapshot.num-retained.max` 限制数量
   - 通过 `snapshot.time-retained` 自动过期

2. **LATEST hint 的准确性 vs 性能**：
   - hint 文件可能不准确，但大多数情况下是准确的
   - 准确时只需 1-2 次文件操作，不准确时回退到目录扫描
   - 这是一个"乐观假设"的优化

3. **Watermark 只增不减 vs 回滚灵活性**：
   - 保证流式处理的单调性，避免数据重复消费
   - 代价是回滚后 watermark 不会回退，可能导致部分数据被跳过
   - 这是流式语义的必然要求

**架构演进：**

Snapshot 格式经历了三个版本：
- **v1 (Paimon 0.4-0.6)**：基础的 Snapshot，只有 manifestList 字段
- **v2 (Paimon 0.7-0.8)**：增加了 base/delta 分离，增加了 commitKind
- **v3 (Paimon 0.9+)**：增加了 changelogManifestList、watermark、statistics、nextRowId 等字段

**业界对比：**

| 特性 | Paimon | Iceberg | Delta Lake |
|------|--------|---------|------------|
| Snapshot 格式 | JSON 文件 | Avro 文件 | JSON 日志 |
| 增量追踪 | base/delta 分离 | added_snapshot_id 标记 | 日志追加 |
| Changelog 支持 | ✅ 原生支持 | ❌ 需要计算 diff | ❌ 需要计算 diff |
| Watermark | ✅ 原生支持 | ❌ 无 | ❌ 无 |
| 精确一次 | commitUser + commitIdentifier | sequence number | txn version |
| LATEST hint | ✅ 有 | ❌ 无 | ✅ 有（_last_checkpoint） |

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
| `deltaManifestList` | `String` | 否 | 增量 Manifest 列表文件名（本次提交新增/删除的文件） |
| `deltaManifestListSize` | `Long` | 是 | 增量列表文件大小（Paimon <=1.0 为 null） |
| `changelogManifestList` | `String` | 是 | Changelog Manifest 列表文件名（CDC 变更日志） |
| `changelogManifestListSize` | `Long` | 是 | Changelog 列表文件大小（Paimon <=1.0 为 null） |
| `indexManifest` | `String` | 是 | 索引 Manifest 文件名（Deletion Vector 等） |
| `commitUser` | `String` | 否 | 提交者标识（用于去重和冲突检测） |
| `commitIdentifier` | `long` | 否 | 提交标识符（用于快照去重，精确一次语义） |
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

**`latestSnapshotFromFileSystem()` 的查找策略：hint 文件优先 + 验证 + 目录列表兜底。** 

Paimon 维护了一个 LATEST hint 文件（位于 `{snapshot_dir}/LATEST`），用于快速定位最新快照。`HintFileUtils.findLatest()` 的查找逻辑：

1. 首先尝试读取 LATEST hint 文件获取快照 ID
2. 如果 hint 存在且有效（> 0），验证该 ID 的下一个快照（ID+1）是否存在
3. 只有当下一个快照不存在时，才确认 hint 指向的是真正的最新快照
4. 如果 hint 文件不存在、无效或已过时（下一个快照存在），则回退到列出目录下所有快照文件取最大 ID 的方式（`findByListFiles()`）
5. 提交成功后通过 `commitLatestHint()` 更新 hint 文件

**为什么需要验证步骤？** hint 文件可能因为并发写入、网络延迟、文件系统缓存等原因指向一个不是最新的快照。通过验证 ID+1 是否存在，可以确保 hint 的准确性，避免返回过期的快照 ID。这种设计兼顾了性能（hint 命中时只需 1-2 次文件操作）和正确性（hint 失效时自动降级到全量扫描）。

---

## 五、Tag 和 Branch 机制

### 解决什么问题

**核心业务问题：**
1. **数据版本管理**：需要为重要的数据版本打标签，如"2024年度财报数据"、"v1.0发布版本"，防止被自动过期删除。
2. **并行开发隔离**：多个团队需要在同一张表上进行独立的开发和测试，互不干扰，类似 Git 的分支功能。
3. **A/B 测试**：需要在生产数据的基础上创建实验分支，测试新的数据处理逻辑，不影响生产环境。
4. **数据快照保留**：某些快照需要长期保留（如月末快照），但又不想保留所有中间快照。

**没有这个设计的后果：**
- 重要的历史数据会被自动过期删除，无法追溯
- 多团队开发时相互干扰，测试数据污染生产数据
- 无法进行安全的实验，只能在生产环境直接修改
- 需要手动管理快照保留策略，容易出错

**实际场景：**
```sql
-- 场景1: 为月末快照打标签
CALL sys.create_tag('my_db.orders', 'month_end_2024_01', 100);

-- 场景2: 创建开发分支进行测试
CALL sys.create_branch('my_db.orders', 'dev_team_a', 'month_end_2024_01');
-- 在分支上进行开发
INSERT INTO `my_db.orders$branch_dev_team_a` VALUES (...);

-- 场景3: 测试通过后合并到主分支
CALL sys.fast_forward('my_db.orders', 'dev_team_a');

-- 场景4: 设置 Tag 自动过期
CALL sys.create_tag('my_db.orders', 'daily_backup', 100, '7 d'); -- 7天后自动删除
```

### 有什么坑

**误区陷阱：**
1. **Tag 不是数据副本**：Tag 只是对 Snapshot 的引用，底层数据文件是共享的。删除 Tag 不会立即删除数据，但如果没有其他引用，数据会被清理。
   ```java
   // 误区：以为 Tag 是独立的数据副本
   tagManager.createTag(snapshot, "backup", null, null, false);
   // Tag 只是引用，不会复制数据文件
   ```

2. **分支不能直接 DDL**：尝试 `DROP TABLE my_table$branch_dev` 会失败，必须使用 `BranchManager.dropBranch()`。
   ```sql
   -- 错误
   DROP TABLE my_table$branch_dev; -- 抛异常
   
   -- 正确
   CALL sys.drop_branch('my_db.my_table', 'dev');
   ```

3. **fast_forward 不是 merge**：`fast_forward` 只是将主分支的指针移动到分支的最新 Snapshot，不会合并冲突。如果主分支在此期间有新提交，fast_forward 会失败。
   ```java
   // 场景：主分支和分支都有新提交
   // main: snapshot-100 → snapshot-101
   // dev:  snapshot-100 → snapshot-102
   branchManager.fastForward("dev"); // 失败，因为 main 已经前进到 101
   ```

**错误配置：**
```java
// 错误：Tag TTL 设置太短
tagManager.createTag(snapshot, "important", Duration.ofHours(1), ...);
// 1小时后 Tag 就会被删除，可能还没来得及使用

// 错误：创建分支时忘记指定 Tag
branchManager.createBranch("dev"); // 从最新 Schema 创建空分支
// 如果想从某个历史版本创建分支，必须先打 Tag
```

**生产环境注意事项：**
1. **Tag 会阻止数据清理**：被 Tag 引用的 Snapshot 和数据文件不会被过期删除，即使超过了 `snapshot.time-retained`。
2. **分支共享数据文件**：分支创建时不会复制数据文件，只复制元数据（Schema/Snapshot/Tag）。新写入的数据会生成新文件。
3. **删除分支要谨慎**：`dropBranch` 会删除分支独有的数据文件。如果分支有重要数据，删除前要先 fast_forward 或导出。

**性能陷阱：**
```java
// 陷阱：频繁创建和删除 Tag
for (int i = 0; i < 1000; i++) {
    tagManager.createTag(snapshot, "temp_" + i, null, null, false);
    tagManager.deleteTag("temp_" + i, ...);
}
// 每次都会进行文件 IO，性能差

// 优化：使用 Snapshot 的 num-retained 配置，而不是手动管理 Tag
options.put("snapshot.num-retained.min", "1000");
```

### 核心概念解释

**Tag（标签）：**
对某个 Snapshot 的命名引用，类似 Git Tag。Tag 的物理文件内容就是被标记 Snapshot 的完整 JSON（可能附加 TTL 信息）。

**Tag TTL（Time To Live）：**
Tag 的自动过期时间。创建 Tag 时可以指定 `timeRetained`，到期后 Tag 会被自动删除。

**Branch（分支）：**
表的独立演进路径，拥有自己的 Schema、Snapshot、Tag，但共享底层数据文件。类似 Git 的分支。

**Main Branch（主分支）：**
默认的分支，名称为 `"main"`。所有表创建时都在主分支上。

**Fast Forward（快进）：**
将分支的最新状态合并到主分支。要求主分支没有新的提交（即分支是主分支的"直接后继"）。

**写时复制（Copy-on-Write）：**
分支创建时不复制数据文件，只复制元数据。新写入的数据会生成新文件，不会修改已有文件。

### 设计理念

**为什么这样设计：**

1. **Tag 即 Snapshot 副本**：
   - Tag 文件的内容就是 Snapshot 的 JSON，加上可选的 TTL 信息
   - 这使得 Tag 的实现非常简单，不需要额外的数据结构
   - 读取 Tag 就是读取 Snapshot，无需额外的映射表
   ```java
   // Tag 文件内容
   {
     "version": 3,
     "id": 100,
     "schemaId": 5,
     "baseManifestList": "manifest-list-xxx",
     ...
     "tagCreateTime": "2024-01-01T00:00:00",  // Tag 特有字段
     "tagTimeRetained": "PT168H"              // Tag 特有字段
   }
   ```

2. **分支共享数据文件的写时复制**：
   - 分支创建是 O(1) 操作，只需复制少量元数据文件
   - 节省存储空间，避免大量数据复制
   - 新写入的数据会生成新文件，不会影响其他分支
   ```
   main:   snapshot-100 → file-1, file-2
   branch: snapshot-100 → file-1, file-2  (共享)
           snapshot-101 → file-1, file-2, file-3  (file-3 是新的)
   ```

3. **分支路径的文件系统布局**：
   - 主分支：`{table}/schema/`, `{table}/snapshot/`, `{table}/tag/`
   - 其他分支：`{table}/branch/branch-{name}/schema/`, `{table}/branch/branch-{name}/snapshot/`
   - 数据文件和 manifest 文件在 `{table}/manifest/` 和 `{table}/data/` 下共享

4. **fast_forward 的限制**：
   - 只允许"快进"式合并，不支持"三方合并"
   - 这简化了实现，避免了复杂的冲突解决逻辑
   - 如果需要合并有冲突的分支，需要手动处理

**权衡取舍：**

1. **Tag 阻止数据清理 vs 存储成本**：
   - Tag 会阻止被引用的数据文件被删除，可能导致存储空间持续增长
   - 通过 Tag TTL 机制自动清理过期 Tag
   - 需要在数据保留和存储成本之间平衡

2. **分支隔离 vs 数据共享**：
   - 分支之间逻辑隔离，但物理共享数据文件
   - 优势：创建分支快速，节省存储
   - 劣势：删除分支时需要判断哪些文件是独有的，逻辑复杂

3. **fast_forward 的简单性 vs 灵活性**：
   - 只支持快进式合并，不支持复杂的合并策略
   - 优势：实现简单，不需要冲突解决
   - 劣势：如果主分支有新提交，无法自动合并

**架构演进：**

Tag 和 Branch 机制的演进：
- **v0.4-0.6**：只有 Snapshot，没有 Tag 和 Branch
- **v0.7-0.8**：引入 Tag 机制，支持命名快照
- **v0.9+**：引入 Branch 机制，支持并行开发
- **v1.0+**：增加 Tag TTL 自动过期功能

**业界对比：**

| 特性 | Paimon | Iceberg | Delta Lake |
|------|--------|---------|------------|
| Tag 支持 | ✅ 原生支持 | ✅ 通过 Ref（1.5+） | ❌ 无 |
| Branch 支持 | ✅ 原生支持 | ✅ 通过 Ref（1.5+） | ❌ 无 |
| Tag TTL | ✅ 支持 | ❌ 无 | - |
| Fast Forward | ✅ 支持 | ✅ 支持 | - |
| 写时复制 | ✅ 是 | ✅ 是 | - |
| 文件系统布局 | 独立目录 | 通过 Ref 文件 | - |

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

### 解决什么问题

**核心业务问题：**
1. **大表的元数据可伸缩性**：一个大表可能包含数百万个数据文件，如果全部记录在 Snapshot 中，每次提交都需要重写整个文件，代价极高。
2. **增量提交的效率**：每次提交只涉及少量文件的变更，不应该重写所有文件的元数据。
3. **查询时的文件过滤**：查询时需要根据分区、桶、LSM level 等条件快速过滤出相关的数据文件，避免读取所有元数据。
4. **Manifest 文件的碎片化**：随着提交次数增加，Manifest 文件数量也会增加，需要周期性合并以减少读取时的 IO 次数。

**没有这个设计的后果：**
- Snapshot 文件会随着表的增长而无限膨胀，最终无法处理
- 每次提交都需要重写所有文件的元数据，性能极差
- 查询时需要读取所有数据文件的元数据，无法快速过滤
- 元数据文件碎片化严重，影响查询性能

**实际场景：**
```java
// 场景1: 大表的增量提交
// 表有 1,000,000 个数据文件
// 本次提交只新增了 10 个文件
// 如果没有三级结构，需要重写包含 1,000,010 个文件的元数据
// 有了三级结构，只需写入包含 10 个文件的新 ManifestFile

// 场景2: 分区过滤
// 查询 WHERE partition_date = '2024-01-01'
// ManifestFileMeta 的 partitionStats 记录了每个 ManifestFile 的分区范围
// 可以快速跳过不相关的 ManifestFile，避免读取其内容

// 场景3: Manifest 合并
// 经过 100 次提交，产生了 100 个小的 ManifestFile
// 合并为 10 个大的 ManifestFile，减少读取时的 IO 次数
```

### 有什么坑

**误区陷阱：**
1. **ManifestFileMeta 的统计信息可能为空**：`minBucket`、`maxBucket`、`minLevel`、`maxLevel` 等字段是后来添加的优化字段，旧版本生成的 ManifestFileMeta 中这些字段为 null。
   ```java
   // 错误：直接使用可能为 null 的字段
   if (meta.minBucket() == 0) { ... } // NPE 风险
   
   // 正确：先判空
   if (meta.minBucket() != null && meta.minBucket() == 0) { ... }
   ```

2. **ManifestEntry 的 ADD/DELETE 不是物理操作**：它们是逻辑操作记录，不会立即修改文件系统。
   ```java
   // 误区：以为 DELETE entry 会立即删除文件
   ManifestEntry entry = new ManifestEntry(FileKind.DELETE, partition, bucket, file);
   // 文件仍然存在于文件系统，只是逻辑上被标记为删除
   ```

3. **Manifest 合并不是实时的**：合并在提交时触发，但有阈值限制（`manifest.merge-min-count`）。如果 ManifestFile 数量未达到阈值，不会合并。
   ```java
   // 配置了合并阈值
   options.put("manifest.merge-min-count", "30");
   // 只有当 ManifestFile 数量 >= 30 时才会触发合并
   ```

**错误配置：**
```java
// 错误：manifest.target-file-size 设置太小
options.put("manifest.target-file-size", "1MB"); // 太小
// 导致 ManifestFile 数量过多，查询时需要读取大量小文件

// 错误：manifest.merge-min-count 设置太大
options.put("manifest.merge-min-count", "1000"); // 太大
// 导致 ManifestFile 长期不合并，碎片化严重

// 推荐配置
options.put("manifest.target-file-size", "8MB");
options.put("manifest.merge-min-count", "30");
options.put("manifest.full-compaction-threshold-size", "16MB");
```

**生产环境注意事项：**
1. **ManifestList 文件不会被清理**：即使 Snapshot 被删除，其引用的 ManifestList 文件也不会立即删除。需要等待 Snapshot 过期后，后台任务才会清理。
2. **Manifest 缓存要合理配置**：`CachingCatalog` 会缓存 Manifest 文件内容，但如果缓存配置不当，可能占用大量内存。
3. **对象存储上的 Manifest 读取性能**：对象存储的小文件读取性能较差，建议增大 `manifest.target-file-size`。

**性能陷阱：**
```java
// 陷阱：频繁读取 ManifestFile 内容
for (ManifestFileMeta meta : manifestList.read(...)) {
    List<ManifestEntry> entries = manifestFile.read(meta.fileName(), meta.fileSize());
    // 每次都会进行文件 IO
}

// 优化：利用 ManifestFileMeta 的统计信息过滤
for (ManifestFileMeta meta : manifestList.read(...)) {
    if (!partitionFilter.test(meta.partitionStats())) {
        continue; // 跳过不相关的 ManifestFile
    }
    List<ManifestEntry> entries = manifestFile.read(meta.fileName(), meta.fileSize());
}
```

### 核心概念解释

**三级结构：**
```
Snapshot (JSON 文件)
  ↓ 引用
ManifestList (二进制文件, 包含 ManifestFileMeta 列表)
  ↓ 引用
ManifestFile (二进制文件, 包含 ManifestEntry 列表)
  ↓ 引用
Data Files (实际数据文件)
```

**ManifestList（清单列表）：**
包含多个 `ManifestFileMeta` 的二进制文件。Snapshot 通过 `baseManifestList`、`deltaManifestList`、`changelogManifestList` 三个字段引用不同类型的 ManifestList。

**ManifestFileMeta（清单文件元数据）：**
ManifestFile 的"摘要信息"，包含文件名、大小、ADD/DELETE 数量、分区统计信息等。用于快速过滤不需要读取的 ManifestFile。

**ManifestFile（清单文件）：**
包含多个 `ManifestEntry` 的二进制文件。每个 ManifestEntry 表示一个数据文件的添加或删除操作。

**ManifestEntry（清单条目）：**
表示一个数据文件的逻辑操作（ADD 或 DELETE），包含分区、桶、文件元数据等信息。

**FileKind（文件操作类型）：**
- `ADD (0)`：文件添加
- `DELETE (1)`：文件删除

**Manifest 合并（Manifest Merge）：**
将多个小的 ManifestFile 合并为少量大的 ManifestFile，减少读取时的 IO 次数。合并时会消除 DELETE entries（如果对应的 ADD entry 也在合并范围内）。

### 设计理念

**为什么这样设计：**

1. **三级结构的核心价值**：
   - **可伸缩性**：支持数百万个数据文件，Snapshot 文件大小保持恒定（只包含 ManifestList 的文件名）
   - **增量写入**：每次提交只需写入新的 ManifestFile 和 ManifestList，不需要重写所有元数据
   - **快速过滤**：通过 ManifestFileMeta 的统计信息快速跳过不相关的 ManifestFile

2. **ManifestFileMeta 的统计信息**：
   - `partitionStats`：记录分区字段的 min/max/null count，支持分区过滤
   - `minBucket`/`maxBucket`：支持桶级过滤
   - `minLevel`/`maxLevel`：支持 LSM level 过滤
   - 这些统计信息使得查询时可以跳过大量不相关的 ManifestFile，大幅提升性能

3. **ADD/DELETE 的逻辑操作**：
   - ManifestEntry 不直接修改文件系统，只记录逻辑操作
   - 通过合并所有 ManifestEntry 可以得到有效文件列表
   - 这使得提交操作是纯粹的追加写入，不需要修改已有文件

4. **Manifest 合并的必要性**：
   - 随着提交次数增加，ManifestFile 数量也会增加
   - 过多的小文件会导致查询时的 IO 次数过多
   - 合并可以消除 DELETE entries，减少元数据大小
   ```java
   // 合并前
   ManifestFile-1: [ADD file-1, ADD file-2]
   ManifestFile-2: [DELETE file-1, ADD file-3]
   ManifestFile-3: [ADD file-4]
   
   // 合并后
   ManifestFile-merged: [ADD file-2, ADD file-3, ADD file-4]
   // file-1 的 ADD 和 DELETE 被消除
   ```

**权衡取舍：**

1. **三级结构的复杂性 vs 可伸缩性**：
   - 三级结构增加了实现复杂度，需要管理多层文件
   - 但这是支持大表的唯一方式，否则 Snapshot 文件会无限膨胀

2. **ManifestFileMeta 统计信息的准确性 vs 存储开销**：
   - 统计信息需要额外的存储空间
   - 但可以大幅减少查询时的 IO，性价比很高

3. **Manifest 合并的频率 vs 写入性能**：
   - 频繁合并可以减少 ManifestFile 数量，提升查询性能
   - 但合并本身需要读取和重写 ManifestFile，影响写入性能
   - 通过 `manifest.merge-min-count` 和 `manifest.full-compaction-threshold-size` 平衡

**架构演进：**

Manifest 结构的演进：
- **v0.1-0.3**：只有两级结构（Snapshot → DataFile），无法支持大表
- **v0.4-0.6**：引入三级结构（Snapshot → ManifestList → ManifestFile → DataFile）
- **v0.7-0.8**：增加 ManifestFileMeta 的统计信息（partitionStats）
- **v0.9+**：增加 minBucket/maxBucket/minLevel/maxLevel 等优化字段

**业界对比：**

| 特性 | Paimon | Iceberg | Delta Lake |
|------|--------|---------|------------|
| 层级结构 | 三级 | 三级 | 两级（Snapshot → DataFile） |
| 统计信息 | ✅ 丰富（分区/桶/level） | ✅ 丰富（分区/列） | ✅ 基础（分区） |
| Manifest 合并 | ✅ 自动合并 | ✅ 自动合并 | ❌ 无 |
| 增量追踪 | base/delta 分离 | added_snapshot_id | 日志追加 |
| 文件格式 | 二进制（Avro） | Avro | Parquet |

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

### 解决什么问题

**核心业务问题：**
1. **并发写入的安全性**：多个写入者可能同时向同一张表提交数据，需要保证数据不会相互覆盖或产生冲突。
2. **精确一次语义（Exactly-Once）**：分布式写入场景下，同一批数据可能因为重试被提交多次，需要自动去重。
3. **数据一致性**：提交必须是原子的——要么全部成功，要么全部失败，不能出现部分成功的中间状态。
4. **冲突检测**：对于主键表，需要检测并发写入是否产生了 key range 重叠，避免破坏主键唯一性。
5. **APPEND 和 COMPACT 的分离**：追加新数据和压缩旧数据是两种不同的操作，需要分别提交和管理。

**没有这个设计的后果：**
- 并发写入会相互覆盖，导致数据丢失
- 重试会产生重复数据，破坏数据一致性
- 提交失败后可能留下不一致的中间状态
- 主键表可能出现重复的主键值
- 无法区分数据变更和物理优化，影响流式消费

**实际场景：**
```java
// 场景1: 并发写入
// 两个 Flink job 同时向同一张表写入数据
// Job A: 提交 snapshot-100（新增 file-1, file-2）
// Job B: 提交 snapshot-100（新增 file-3, file-4）
// 乐观锁机制确保只有一个成功，另一个重试为 snapshot-101

// 场景2: 精确一次语义
// Flink checkpoint 100 提交了 snapshot-50
// 如果 checkpoint 100 失败重启，重新提交时会检测到 snapshot-50 已存在
// 通过 (commitUser, commitIdentifier, commitKind) 去重，跳过重复提交

// 场景3: 冲突检测
// 两个写入者同时向同一个 bucket 写入数据
// Writer A: 写入 key range [1, 100]
// Writer B: 写入 key range [50, 150]
// 冲突检测发现 key range 重叠，拒绝提交

// 场景4: APPEND/COMPACT 分离
// 一次提交包含新数据和压缩结果
// 生成两个 snapshot: snapshot-100 (APPEND), snapshot-101 (COMPACT)
// 流式消费者只消费 APPEND 类型的 snapshot
```

### 有什么坑

**误区陷阱：**
1. **commit() 可能生成 0-2 个 Snapshot**：不是每次 commit 都会生成 Snapshot。如果没有变更且 `ignoreEmptyCommit=true`，不会生成 Snapshot。
   ```java
   // 误区：以为每次 commit 都会生成 Snapshot
   int count = commit.commit(committable, true);
   // count 可能是 0（无变更）、1（只有 APPEND 或 COMPACT）、2（APPEND + COMPACT）
   ```

2. **冲突检测不是全局的**：只检测变更分区内的冲突，不同分区的并发写入不会冲突。
   ```java
   // 不会冲突：写入不同分区
   Writer A: INSERT INTO t PARTITION(dt='2024-01-01') ...
   Writer B: INSERT INTO t PARTITION(dt='2024-01-02') ...
   
   // 可能冲突：写入相同分区
   Writer A: INSERT INTO t PARTITION(dt='2024-01-01') ...
   Writer B: INSERT INTO t PARTITION(dt='2024-01-01') ...
   ```

3. **OVERWRITE 会自动升级**：如果 APPEND 提交包含 DELETE 操作或 Deletion Vector，会自动升级为 OVERWRITE，启用更严格的冲突检测。
   ```java
   // 原本是 APPEND
   CommitKind kind = CommitKind.APPEND;
   // 但如果包含 DELETE 或 DV
   if (hasDelete || hasDV) {
       kind = CommitKind.OVERWRITE; // 自动升级
   }
   ```

**错误配置：**
```java
// 错误：commit.max-retries 设置太小
options.put("commit.max-retries", "1"); // 太少
// 在高并发场景下容易失败

// 错误：commit.timeout 设置太短
options.put("commit.timeout", "10s"); // 太短
// 如果冲突检测需要读取大量文件，可能超时

// 推荐配置
options.put("commit.max-retries", "10");
options.put("commit.timeout", "5min");
options.put("commit.min-retry-wait", "10ms");
options.put("commit.max-retry-wait", "10s");
```

**生产环境注意事项：**
1. **对象存储必须启用锁**：S3/OSS 等对象存储的原子操作能力有限，必须配置 `CatalogLock` 来保证并发安全。
2. **重试会增加延迟**：在高并发场景下，重试次数可能很多，导致提交延迟增加。需要监控 `retryCount` 指标。
3. **冲突检测会读取大量文件**：如果变更的分区包含大量数据文件，冲突检测需要读取所有文件的元数据，影响性能。

**性能陷阱：**
```java
// 陷阱：频繁的小批量提交
for (int i = 0; i < 1000; i++) {
    commit.commit(singleRecord, true);
    // 每次都会进行冲突检测和 Manifest 合并，性能差
}

// 优化：批量提交
List<CommitMessage> batch = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
    batch.add(singleRecord);
}
commit.commit(ManifestCommittable.fromMessages(batch), true);
```

### 核心概念解释

**CommitMessage（提交消息）：**
表示一次写入操作产生的文件变更，包含新增的数据文件、删除的数据文件、changelog 文件、索引文件等。

**ManifestCommittable（清单可提交对象）：**
包含多个 `CommitMessage` 的聚合对象，表示一次完整的提交。

**CommitKind（提交类型）：**
- `APPEND`：追加新数据，不删除已有文件
- `COMPACT`：压缩合并，不改变逻辑数据
- `OVERWRITE`：覆写分区或删除文件
- `ANALYZE`：收集统计信息

**乐观锁（Optimistic Locking）：**
不使用分布式锁，而是通过"读取-计算-写入-检查"的方式实现并发控制。如果写入失败（别人已经写了相同的 Snapshot ID），则重试。

**冲突检测（Conflict Detection）：**
检查并发写入是否产生了数据冲突，如 key range 重叠、桶数不一致等。

**去重检测（Deduplication）：**
通过 `(commitUser, commitIdentifier, commitKind)` 三元组判断提交是否已经完成，避免重复提交。

**回滚（Rollback）：**
当检测到冲突且允许回滚时，尝试删除冲突的文件，然后重试提交。

**增量冲突检测优化：**
重试时不重新读取所有 base 数据文件，而是在上次读取的基础上增量更新。

### 设计理念

**为什么这样设计：**

1. **APPEND 和 COMPACT 分离提交**：
   - 语义清晰：APPEND 表示新数据，COMPACT 表示物理优化
   - 流式消费者可以只消费 APPEND 类型的 Snapshot
   - 冲突处理不同：APPEND 可能需要回滚，COMPACT 不需要
   ```java
   // 一次 commit 可能生成两个 Snapshot
   commit(committable) {
       if (hasAppend) {
           tryCommit(..., CommitKind.APPEND, ...); // snapshot-100
       }
       if (hasCompact) {
           tryCommit(..., CommitKind.COMPACT, ...); // snapshot-101
       }
   }
   ```

2. **乐观锁的核心价值**：
   - 避免了分布式锁的复杂性和性能开销
   - 在低冲突场景下性能更好（大多数情况下一次成功）
   - 代价是高冲突场景下需要多次重试
   ```java
   while (true) {
       Snapshot latest = snapshotManager.latestSnapshot();
       Snapshot newSnapshot = buildSnapshot(latest.id() + 1, ...);
       boolean success = fileIO.tryToWriteAtomic(snapshotPath, newSnapshot);
       if (success) break; // 成功
       // 失败则重试
   }
   ```

3. **精确一次语义的去重机制**：
   - 分布式写入场景下，同一批数据可能因为重试被提交多次
   - 通过 `(commitUser, commitIdentifier, commitKind)` 三元组去重
   - 如果发现已存在相同的 Snapshot，直接返回成功
   ```java
   // 检查是否已经提交
   for (long id = lastSnapshot.id() + 1; id <= latestSnapshot.id(); id++) {
       Snapshot s = snapshotManager.snapshot(id);
       if (s.commitUser().equals(myUser) 
           && s.commitIdentifier() == myIdentifier
           && s.commitKind() == myKind) {
           return SuccessCommitResult; // 已经提交过了
       }
   }
   ```

4. **冲突检测的多层次检查**：
   - 桶数一致性：同一分区内的所有文件必须具有相同的 totalBuckets
   - Key Range 重叠：主键表的文件 key range 不能重叠
   - 删除文件存在性：要删除的文件必须存在于 base 中
   - Row ID 范围冲突：Row Tracking 场景下的 ID 范围不能重叠

5. **增量冲突检测优化**：
   - 重试时不重新读取所有 base 数据文件
   - 在上次读取的基础上增量更新（读取 [lastSnapshot, latestSnapshot] 之间的变更）
   - 大幅减少重试时的 IO 开销
   ```java
   if (isRetry && hasBaseDataFiles) {
       baseDataFiles = previousBaseDataFiles;
       List<SimpleFileEntry> incremental = scanner.readIncrementalChanges(
           previousSnapshot, latestSnapshot, changedPartitions);
       baseDataFiles.addAll(incremental);
       baseDataFiles = FileEntry.mergeEntries(baseDataFiles);
   }
   ```

**权衡取舍：**

1. **乐观锁 vs 悲观锁**：
   - 乐观锁：低冲突场景性能好，高冲突场景需要多次重试
   - 悲观锁：性能稳定，但需要分布式锁服务，增加复杂度
   - Paimon 选择乐观锁，通过配置 `commit.max-retries` 和随机退避平衡

2. **冲突检测的粒度 vs 性能**：
   - 细粒度检测（key range 级别）可以减少误判，提高并发度
   - 但需要读取所有文件的元数据，影响性能
   - 通过增量检测优化减少重试时的开销

3. **APPEND/COMPACT 分离 vs 提交次数**：
   - 分离提交使得语义清晰，流式消费更高效
   - 但一次 commit 可能生成两个 Snapshot，增加元数据数量
   - 通过 Snapshot 过期机制自动清理

**架构演进：**

Commit 流程的演进：
- **v0.1-0.3**：简单的乐观锁，无冲突检测
- **v0.4-0.6**：增加冲突检测，支持 key range 检查
- **v0.7-0.8**：增加 APPEND/COMPACT 分离，增加去重检测
- **v0.9+**：增加增量冲突检测优化，增加 Deletion Vector 支持

**业界对比：**

| 特性 | Paimon | Iceberg | Delta Lake |
|------|--------|---------|------------|
| 并发控制 | 乐观锁 | 乐观锁 | 乐观锁 |
| 冲突检测 | 分区/桶/key range | 分区/文件 | 分区/文件 |
| 精确一次 | commitUser + commitIdentifier | sequence number | txn version |
| 重试策略 | 随机退避 + 超时 | 固定次数 | 固定次数 |
| APPEND/COMPACT 分离 | ✅ 是 | ❌ 否 | ❌ 否 |
| 增量冲突检测 | ✅ 支持 | ❌ 无 | ❌ 无 |
| 回滚机制 | ✅ 支持 | ❌ 无 | ❌ 无 |

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
  |     |-- 条件: appendTableFiles/appendChangelog/appendIndexFiles 非空 或 ignoreEmptyCommit=false
  |     |-- 确定 commitKind = APPEND
  |     |-- 特殊情况: 如果 appendFiles 中包含 DELETE 或 DV
  |     |     +-- 升级为 commitKind = OVERWRITE, 允许回滚
  |     |
  |     +-- tryCommit(appendFiles, APPEND/OVERWRITE, ...)  --------> [详见 7.2]
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
| `commit.timeout` | - | 提交超时时间（无默认值） |
| `commit.max-retries` | `10` | 最大重试次数 |
| `commit.min-retry-wait` | `10ms` | 最小重试等待 |
| `commit.max-retry-wait` | `10s` | 最大重试等待 |
| `commit.discard-duplicate-files` | `false` | 是否丢弃重复文件 |

**Manifest 相关:**

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `manifest.target-file-size` | `8MB` | Manifest 文件目标大小 |
| `manifest.merge-min-count` | `30` | 触发 manifest 合并的最小文件数 |
| `manifest.full-compaction-threshold-size` | `16MB` | 全量压缩阈值 |

**Snapshot/Tag 相关:**

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `snapshot.time-retained` | `1h` | Snapshot 保留时间 |
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
| `cache-enabled` | `true` | 启用 Catalog 缓存 |
| `cache.expire-after-access` | `10min` | 访问后过期时间 |
| `cache.expire-after-write` | `30min` | 写入后过期时间 |
| `cache.manifest-small-file-memory` | `128MB` | 小 manifest 缓存内存 |
| `cache.manifest-small-file-threshold` | `1MB` | 小文件阈值 |
| `cache.manifest-max-memory` | - | manifest 最大缓存内存 |
| `cache.partition-max-num` | `0` | 分区缓存最大数量 |
| `cache.snapshot.max-num-per-table` | `20` | 每表最大缓存快照数 |
| `cache.deletion-vectors.max-num` | `100000` | DV 元数据最大缓存数 |

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
