---
title: "FUSE 支持"
sidebar_position: 7
---


# FUSE 支持 {#fuse-support}

当使用 PyPaimon REST Catalog 访问远程对象存储（例如 OSS、S3 或 HDFS）时，数据访问通常会经过远程存储 SDK。然而，在远程存储路径通过 FUSE（用户空间文件系统，Filesystem in Userspace）挂载到本地的场景下，用户可以直接通过本地文件系统路径访问数据，以获得更好的性能。

该特性使得 PyPaimon 在 FUSE 挂载可用时能够使用本地文件访问，从而绕过远程存储 SDK。

## 配置 {#configuration}

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|---------|-------------|
| `fuse.enabled` | Boolean | `false` | 是否启用 FUSE 本地路径映射 |
| `fuse.root` | String | （无） | FUSE 挂载的本地根路径，例如 `/mnt/fuse/warehouse` |
| `fuse.validation-mode` | String | `strict` | 校验模式：`strict`、`warn` 或 `none` |

## 用法 {#usage}

```python
from pypaimon import CatalogFactory

catalog_options = {
    'metastore': 'rest',
    'uri': 'http://rest-server:8080',
    'warehouse': 'oss://my-catalog/',
    'token.provider': 'xxx',

    # FUSE local path configuration
    'fuse.enabled': 'true',
    'fuse.root': '/mnt/fuse/warehouse',
    'fuse.validation-mode': 'strict'
}

catalog = CatalogFactory.create(catalog_options)
```

## 校验模式 {#validation-modes}

校验会在首次访问数据时执行，以验证 FUSE 挂载是否正确。当本地路径不存在时，`validation-mode` 控制其行为：

| 模式 | 行为 | 适用场景 |
|------|----------|----------|
| `strict` | 抛出异常，阻断操作 | 生产环境，安全优先 |
| `warn` | 记录警告日志，回退到默认 FileIO | 测试环境，兼容性优先 |
| `none` | 跳过校验，直接使用 | 受信任环境，性能优先 |

**注意**：配置错误（例如 `fuse.enabled=true` 但未配置 `fuse.root`）将直接抛出异常，无论处于何种校验模式。

## 工作原理 {#how-it-works}

1. 当 `fuse.enabled=true` 时，PyPaimon 会尝试使用本地文件访问
2. 在首次访问数据时，触发校验（除非模式为 `none`）
3. 校验会获取 `default` 数据库的位置，并将其转换为本地路径
4. 如果本地路径存在，后续数据访问将使用 `FuseLocalFileIO`
5. 路径转换使用数据库/表的逻辑名称：远程路径 `oss://<catalog-id>/<db-id>/<table-id>` → 本地路径 `<root>/<db-name>/<table-name>`
6. 如果校验失败，其行为取决于 `validation-mode`

## 示例场景 {#example-scenario}

假设你有如下环境：
- 远程存储路径使用 UUID：`oss://clg-paimon-xxx/db-xxx/tbl-xxx`
- FUSE 挂载：`/mnt/fuse/warehouse`（挂载到 `pvfs://demo_catalog`）
- FUSE 暴露逻辑名称：`/mnt/fuse/warehouse/my_db/my_table`

```python
from pypaimon import CatalogFactory

catalog = CatalogFactory.create({
    'metastore': 'rest',
    'uri': 'http://rest-server:8080',
    'warehouse': 'oss://my-catalog/',
    'fuse.enabled': 'true',
    'fuse.root': '/mnt/fuse/warehouse',
    'fuse.validation-mode': 'none'
})

# When reading table 'my_db.my_table', PyPaimon will:
# 1. Convert "oss://clg-paimon-xxx/db-xxx/tbl-xxx" to "/mnt/fuse/warehouse/my_db/my_table"
# 2. Use FuseLocalFileIO to read from local path
table = catalog.get_table('my_db.my_table')
reader = table.new_read_builder().new_read()
```

## 限制 {#limitations}

- 仅支持 Catalog 级别的 FUSE 挂载（单个 `fuse.root` 配置）
- 校验仅检查本地路径是否存在，不检查数据一致性
- 如果 FUSE 挂载在校验之后变为不可用，文件操作可能会失败
