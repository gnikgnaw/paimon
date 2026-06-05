---
title: "Trino"
sidebar_position: 5
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

# Trino

本文档是在 Trino 中使用 Paimon 的指南。

## 版本 {#version}

Paimon 目前支持 Trino 440。

## 文件系统 {#filesystem}

从 0.8 版本开始，Paimon 的所有操作都共享 Trino 文件系统，这意味着在使用 trino-paimon 之前，你需要先配置 Trino 文件系统。你可以在 Trino 官方网站上找到如何为 Trino 配置文件系统的相关信息。

## 准备 Paimon Jar 文件 {#preparing-paimon-jar-file}

[下载](../project/download)

你也可以从源代码手动构建一个 bundled jar。不过，在编译之前需要完成几个准备步骤：

- 要从源代码构建，请[克隆该 git 仓库](@@TRINO_GITHUB_REPO@@)。
- 在本地安装 JDK21，并将 JDK21 配置为全局环境变量；

然后，你可以使用以下命令构建 bundled jar：

```bash
mvn clean install -DskipTests
```

你可以在 `./paimon-trino-<trino-version>/target/paimon-trino-<trino-version>-@@VERSION@@-plugin.tar.gz` 中找到 Trino 连接器 jar。

我们使用 [hadoop-apache](https://mvnrepository.com/artifact/io.trino.hadoop/hadoop-apache) 作为 Hadoop 的依赖，
默认的 Hadoop 依赖通常同时支持 Hadoop 2 和 Hadoop 3。
如果你遇到不受支持的场景，可以指定相应的 Apache Hadoop 版本。

例如，如果你想使用 Hadoop 3.3.5-1，可以使用以下命令构建该 jar：
```bash
mvn clean install -DskipTests -Dhadoop.apache.version=3.3.5-1
```

## 配置 Paimon Catalog {#configure-paimon-catalog}

### 安装 Paimon 连接器 {#install-paimon-connector}
```bash
tar -zxf paimon-trino-<trino-version>-@@VERSION@@-plugin.tar.gz -C ${TRINO_HOME}/plugin
```

> 注意：对于 JDK 21，在部署 Trino 时，应添加 jvm 选项：`--add-opens=java.base/sun.nio.ch=ALL-UNNAMED --add-opens=java.base/java.nio=ALL-UNNAMED`

### 配置 {#configure}

Catalog 通过在 etc/catalog 目录下创建一个 catalog properties 文件来注册。例如，创建 etc/catalog/paimon.properties 文件并写入以下内容，即可将 paimon 连接器挂载为 paimon catalog：

```properties
connector.name=paimon
warehouse=file:/tmp/warehouse
```

如果你使用的是 HDFS，请从以下方式中选择一种来配置你的 HDFS：

- 设置环境变量 HADOOP_HOME。
- 设置环境变量 HADOOP_CONF_DIR。
- 在 properties 中配置 `hadoop-conf-dir`。

如果你使用的是 Hadoop 文件系统，仍然可以使用 trino-hdfs 和 trino-hive 来配置它。
例如，如果你使用 oss 作为存储，可以根据 [Trino 参考文档](https://trino.io/docs/current/connector/hive.html#hdfs-configuration) 在 `paimon.properties` 中写入：

```properties
hive.config.resources=/path/to/core-site.xml
```

然后，根据 [Jindo 参考文档](https://github.com/aliyun/alibabacloud-jindodata/blob/master/docs/user/4.x/4.6.x/4.6.12/oss/presto/jindosdk_on_presto) 配置 core-site.xml

## Kerberos {#kerberos}

在 properties 中使用 KERBEROS 认证时，你可以配置 kerberos keytab 文件。

```properties
security.kerberos.login.principal=hadoop-user
security.kerberos.login.keytab=/etc/trino/hdfs.keytab
```

Keytab 文件必须分发到集群中运行 Trino 的每个节点上。

## 创建 Schema {#create-schema}

```sql
CREATE SCHEMA paimon.test_db;
```

## 创建表 {#create-table}

```sql
CREATE TABLE paimon.test_db.orders (
    order_key bigint,
    orders_tatus varchar,
    total_price decimal(18,4),
    order_date date
)
WITH (
    file_format = 'ORC',
    primary_key = ARRAY['order_key','order_date'],
    partitioned_by = ARRAY['order_date'],
    bucket = '2',
    bucket_key = 'order_key',
    changelog_producer = 'input'
);
```

## 添加列 {#add-column}

```sql
CREATE TABLE paimon.test_db.orders (
    order_key bigint,
    orders_tatus varchar,
    total_price decimal(18,4),
    order_date date
)
WITH (
    file_format = 'ORC',
    primary_key = ARRAY['order_key','order_date'],
    partitioned_by = ARRAY['order_date'],
    bucket = '2',
    bucket_key = 'order_key',
    changelog_producer = 'input'
);

ALTER TABLE paimon.test_db.orders ADD COLUMN shipping_address varchar;
```

## 查询 {#query}

```sql
SELECT * FROM paimon.test_db.orders;
```

## 使用时间旅行进行查询 {#query-with-time-traveling}

```sql
-- read the snapshot from specified timestamp
SELECT * FROM t FOR TIMESTAMP AS OF TIMESTAMP '2023-01-01 00:00:00 Asia/Shanghai';

-- read the snapshot with id 1L (use snapshot id as version)
SELECT * FROM t FOR VERSION AS OF 1;

-- read tag 'my-tag'
SELECT * FROM t FOR VERSION AS OF 'my-tag';

```

:::warning

如果标签（Tag）的名称是一个数字并且等于某个快照 id，VERSION AS OF 语法会优先考虑标签。例如，如果
你有一个基于快照 2 创建、名为 '1' 的标签，那么语句 `SELECT * FROM paimon.test_db.orders FOR VERSION AS OF '1'` 实际查询的是快照 2
而不是快照 1。

:::

## 插入 {#insert}

```sql
INSERT INTO paimon.test_db.orders VALUES (.....);
```

支持：
- 固定分桶的主键表。
- bucket 为 -1 的非主键表。

## Trino 到 Paimon 的类型映射 {#trino-to-paimon-type-mapping}

本节列出 Trino 与 Paimon 之间所有受支持的类型转换。
Trino 的所有数据类型都位于 `io.trino.spi.type` 包中。

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 10%">Trino 数据类型</th>
      <th class="text-left" style="width: 10%">Paimon 数据类型</th>
      <th class="text-left" style="width: 5%">原子类型</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><code>RowType</code></td>
      <td><code>RowType</code></td>
      <td>false</td>
    </tr>
    <tr>
      <td><code>MapType</code></td>
      <td><code>MapType</code></td>
      <td>false</td>
    </tr>
    <tr>
      <td><code>ArrayType</code></td>
      <td><code>ArrayType</code></td>
      <td>false</td>
    </tr>
    <tr>
      <td><code>BooleanType</code></td>
      <td><code>BooleanType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>TinyintType</code></td>
      <td><code>TinyIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>SmallintType</code></td>
      <td><code>SmallIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>IntegerType</code></td>
      <td><code>IntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>BigintType</code></td>
      <td><code>BigIntType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>RealType</code></td>
      <td><code>FloatType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>DoubleType</code></td>
      <td><code>DoubleType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>CharType(length)</code></td>
      <td><code>CharType(length)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>VarCharType(VarCharType.MAX_LENGTH)</code></td>
      <td><code>VarCharType(VarCharType.MAX_LENGTH)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>VarCharType(length)</code></td>
      <td><code>VarCharType(length), length is less than VarCharType.MAX_LENGTH</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>DateType</code></td>
      <td><code>DateType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>TimestampType</code></td>
      <td><code>TimestampType</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>DecimalType(precision, scale)</code></td>
      <td><code>DecimalType(precision, scale)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>VarBinaryType(length)</code></td>
      <td><code>VarBinaryType(length)</code></td>
      <td>true</td>
    </tr>
    <tr>
      <td><code>TimestampWithTimeZoneType</code></td>
      <td><code>LocalZonedTimestampType</code></td>
      <td>true</td>
    </tr>
    </tbody>
</table>

## 临时目录 {#tmp-dir}

Paimon 会将一些 jar 解压到临时目录中用于代码生成（codegen）。默认情况下，Trino 会使用 `'/tmp'` 作为临时
目录，但 `'/tmp'` 可能会被定期删除。

你可以在 Trino 启动时配置以下环境变量：
```shell
-Djava.io.tmpdir=/path/to/other/tmpdir
```

让 Paimon 使用一个安全的临时目录。
