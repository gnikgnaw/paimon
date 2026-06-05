---
title: "PyJindoSDK Support"
sidebar_position: 8
---


# PyJindoSDK 支持 {#pyjindosdk-support}

## 简介 {#introduction}

[JindoSDK](https://github.com/aliyun/alibabacloud-jindodata) 是阿里云开发的高性能存储 SDK，用于访问 OSS（对象存储服务）及其他云存储系统。它提供了经过优化的 I/O 性能，并与阿里云生态系统深度集成。

PyPaimon 现已支持使用 [PyJindoSDK](https://github.com/aliyun/alibabacloud-jindodata)（JindoSDK 的 Python 绑定）来访问 OSS。与基于 PyArrow 的 S3FileSystem 的旧版实现相比，PyJindoSDK 在使用 OSS 时提供了更好的性能和兼容性。

## 用法 {#usage}

### 安装 {#installation}

通过 pip 安装 `pyjindosdk`：

```shell
pip install pyjindosdk
```

安装完成后，PyPaimon 会自动使用 PyJindoSDK 作为访问 OSS 的默认文件 I/O 实现，无需额外配置。

### 回退到旧版实现 {#fallback-to-legacy-implementation}

由于 JindoSDK 是原生实现，并非所有操作系统或平台版本都提供预构建的 Python 包。如果出于任何原因需要回退到基于 PyArrow 的旧版实现，有两种方式可供选择：

**方式 1：将 Catalog 配置项 `fs.oss.impl` 设置为 `legacy`**

```python
from pypaimon import CatalogFactory

catalog_options = {
    'metastore': 'rest',
    'uri': 'http://rest-server:8080',
    'warehouse': 'oss://my-bucket/warehouse',

    # Fallback to the legacy PyArrow S3FileSystem implementation
    'fs.oss.impl': 'legacy',
}

catalog = CatalogFactory.create(catalog_options)
```

**方式 2：卸载 pyjindosdk**

只需卸载 `pyjindosdk` 包，PyPaimon 就会自动回退到旧版实现：

```shell
pip uninstall pyjindosdk
```
