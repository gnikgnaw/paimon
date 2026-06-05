---
title: "PVFS"
sidebar_position: 6
---

<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Paimon 虚拟存储（Paimon Virtual Storage） {#paimon-virtual-storage}

REST Catalog 提供了内置存储，包括 Paimon Table、Format Table 和 Object Table（也称为 Fileset 或 Volume），
它们都需要直接访问文件系统。而我们的 REST Catalog 会生成 UUID 路径，这使得直接访问文件系统变得困难。

因此有了 PVFS，它允许用户通过类似 `pvfs://catalog_name/database_name/table_name/` 的方式进行访问，
使用该路径访问 REST Catalog 中的所有内部表，包括 Paimon Table、Format Table 和 Object Table。
另一个优势是，用户对该文件系统的所有访问都通过 Paimon REST Catalog 的权限系统进行，
无需再维护另一套文件系统权限系统。

## API 行为（API Behavior） {#api-behavior}

例如，如果你有一个名为 'my_catalog' 的 Catalog，list 行为应当如下：

- `listStatus(Path('pvfs://my_catalog/'))`：返回所有数据库，FileStatus 中仅包含虚拟路径。
- `listStatus(Path('pvfs://my_catalog/my_database'))`：返回所有表，FileStatus 中仅包含虚拟路径。

所有路径都返回虚拟路径，读写文件时实际会按照表的真实路径来读写数据。

- `newInputStream(Path('pvfs://my_catalog/my_database/my_table'))`：从 rest server 获取真实路径，并使用真实的文件系统读取数据。

## Java SDK {#java-sdk}

提供一个实现 Hadoop FileSystem 的 Java SDK。通过这种方式，计算引擎可以非常轻松地集成 'PVFS'。

例如，Java 代码可以这样写：

```java
Configuration conf = new Configuration();
conf.set("fs.AbstractFileSystem.pvfs.impl", "org.apache.paimon.vfs.hadoop.Pvfs");
conf.set("fs.pvfs.impl", "org.apache.paimon.vfs.hadoop.PaimonVirtualFileSystem");
conf.set("fs.pvfs.uri", "http://localhost:10000");
conf.set("fs.pvfs.token.provider", "bear");
conf.set("fs.pvfs.token", "token");
Path path = new Path("pvfs://catalog_name/database_name/table_name/a.csv");
FileSystem fs = path.getFileSystem(conf);
FileStatus fileStatus = fs.getFileStatus(path);
```

例如，Spark SQL 可以这样写：

```scala
val spark = SparkSession.builder()
.appName("PVFS CSV Analysis")
.config("spark.hadoop.fs.pvfs.impl", "org.apache.paimon.vfs.hadoop.PaimonVirtualFileSystem")
.config("spark.hadoop.fs.pvfs.uri", "http://localhost:10000")
.config("spark.hadoop.fs.pvfs.token.provider", "bear")
.config("spark.hadoop.fs.pvfs.token", "token")
.getOrCreate()
spark.sql(
s"""
|CREATE TEMPORARY VIEW csv_table
|USING csv
|OPTIONS (
|  path 'pvfs://catalog_name/database_name/my_format_table_name/a.csv',
|  header 'true',
|  inferSchema 'true'
|)
""".stripMargin
)

spark.sql("SELECT * FROM csv_table LIMIT 5").show()
```

例如，使用 Hadoop shell 命令：

```xml
<!-- Configure following configuration in hadoop `core-site.xml` -->
<property>
  <name>fs.AbstractFileSystem.pvfs.impl</name>
  <value>org.apache.paimon.vfs.hadoop.Pvfs</value>
</property>

<property>
  <name>fs.pvfs.impl</name>
  <value>org.apache.paimon.vfs.hadoop.PaimonVirtualFileSystem</value>
</property>

<property>
  <name>fs.pvfs.uri</name>
  <value>http://localhost:10000</value>
</property>

<property>
  <name>fs.pvfs.token.provider</name>
  <value>bear</value>
</property>

<property>
  <name>fs.pvfs.token</name>
  <value>token</value>
</property>
```

示例：执行 hadoop shell 列出虚拟路径

```shell
./${HADOOP_HOME}/bin/hadoop dfs -ls pvfs://catalog_name/database_name/table_name
```

## Python SDK {#python-sdk}

Python SDK 提供 fsspec 风格的 API，可以轻松集成到 Python 生态中。

例如，Python 代码可以这样写：

```python
import pypaimon

options = {
    'uri': 'key',
    'token.provider': 'bear',
    'token': '<token>'
}
fs = pypaimon.PaimonVirtualFileSystem(options)
fs.ls("pvfs://catalog_name/database_name/table_name")
```

例如，Pyarrow 可以这样写：

```python
import pypaimon
import pyarrow.parquet as pq

options = {
    'uri': 'key',
    'token.provider': 'bear',
    'token': '<token>'
}
fs = pypaimon.PaimonVirtualFileSystem(options)
path = 'pvfs://catalog_name/database_name/table_name/a.parquet'
dataset = pq.ParquetDataset(path, filesystem=fs)
table = dataset.read()
df = table.to_pandas()
```

例如，Ray 可以这样写：

```python
import pypaimon
import ray

options = {
    'uri': 'key',
    'token.provider': 'bear',
    'token': '<token>'
}
fs = pypaimon.PaimonVirtualFileSystem(options)

ds = ray.data.read_parquet(filesystem=fs,paths="pvfs://....parquet")
```
