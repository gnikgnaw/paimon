---
title: "SQL DDL"
sidebar_position: 2
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

# SQL DDL {#sql-ddl}

## 创建 Catalog {#create-catalog}

Paimon Catalog 目前支持三种类型的 metastore：

* `filesystem` metastore（默认），它将元数据和表文件都存储在文件系统中。
* `hive` metastore，它额外将元数据存储在 Hive metastore 中。用户可以直接从 Hive 访问这些表。
* `jdbc` metastore，它额外将元数据存储在 MySQL、Postgres 等关系型数据库中。

创建 Catalog 时的详细配置项请参见 [CatalogOptions](../maintenance/configurations#catalogoptions)。

### 创建文件系统 Catalog {#create-filesystem-catalog}

下面的 Flink SQL 注册并使用了一个名为 `my_catalog` 的 Paimon Catalog。元数据和表文件存储在 `hdfs:///path/to/warehouse` 下。

```sql
CREATE CATALOG my_catalog WITH (
    'type' = 'paimon',
    'warehouse' = 'hdfs:///path/to/warehouse'
);

USE CATALOG my_catalog;
```

你可以使用前缀 `table-default.` 为该 Catalog 中创建的表定义任意默认表配置项。

### 创建 Hive Catalog {#creating-hive-catalog}

使用 Paimon Hive Catalog 时，对 Catalog 的更改会直接影响对应的 Hive metastore。在这种 Catalog 中创建的表也可以直接从 Hive 访问。

要使用 Hive Catalog，数据库名、表名和字段名都应为**小写**。

Flink 中的 Paimon Hive Catalog 依赖 Flink Hive connector bundled jar。你应当首先下载 Hive connector bundled jar 并将其加入 classpath。

| Metastore 版本    |  Bundle 名称   | SQL Client JAR                                                                                                                                                                                                                                                                                                   |
|:------------------|:--------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 2.3.0 - 3.1.3     | Flink Bundle  | [下载](https://nightlies.apache.org/flink/flink-docs-stable/docs/connectors/table/hive/overview/#using-bundled-hive-jar) |
| 1.2.0 - x.x.x     | Presto Bundle | [下载](https://repo.maven.apache.org/maven2/com/facebook/presto/hive/hive-apache/1.2.2-2/hive-apache-1.2.2-2.jar) |

下面的 Flink SQL 注册并使用了一个名为 `my_hive` 的 Paimon Hive Catalog。元数据和表文件存储在 `hdfs:///path/to/warehouse` 下。此外，元数据也会存储在 Hive metastore 中。

如果你的 Hive 需要 Kerberos、LDAP、Ranger 等安全认证，或者你希望让 Paimon 表由 Apache Atlas 管理（在 hive-site.xml 中设置 'hive.metastore.event.listeners'），你可以将 hive-conf-dir 和
hadoop-conf-dir 参数指定为 hive-site.xml 文件所在路径。

```sql
CREATE CATALOG my_hive WITH (
    'type' = 'paimon',
    'metastore' = 'hive',
    -- 'uri' = 'thrift://<hive-metastore-host-name>:<port>', default use 'hive.metastore.uris' in HiveConf
    -- 'hive-conf-dir' = '...', this is recommended in the kerberos environment
    -- 'hadoop-conf-dir' = '...', this is recommended in the kerberos environment
    -- 'warehouse' = 'hdfs:///path/to/warehouse', default use 'hive.metastore.warehouse.dir' in HiveConf
);

USE CATALOG my_hive;
```

你可以使用前缀 `table-default.` 为该 Catalog 中创建的表定义任意默认表配置项。

此外，你还可以创建 [FlinkGenericCatalog](./quick-start)。

> 使用 Hive Catalog 通过 alter table 修改不兼容的列类型时，你需要配置 `hive.metastore.disallow.incompatible.col.type.changes=false`。参见 [HIVE-17832](https://issues.apache.org/jira/browse/HIVE-17832)。

> 如果你使用的是 Hive3，请禁用 Hive ACID：
>
> ```shell
> hive.strict.managed.tables=false
> hive.create.as.insert.only=false
> metastore.create.as.acid=false
> ```

#### 将分区同步到 Hive Metastore {#synchronizing-partitions-into-hive-metastore}

默认情况下，Paimon 不会将新创建的分区同步到 Hive metastore。用户在 Hive 中将看到一个未分区的表。分区下推将改为通过过滤器下推来实现。

如果你希望在 Hive 中看到一个分区表，并且将新创建的分区也同步到 Hive metastore，请将表属性 `metastore.partitioned-table` 设置为 true。另请参见 [CoreOptions](../maintenance/configurations#coreoptions)。

#### 为 Hive 表添加参数 {#adding-parameters-to-a-hive-table}

使用表配置项可以方便地定义 Hive 表参数。
以 `hive.` 为前缀的参数将自动定义到 Hive 表的 `TBLPROPERTIES` 中。
例如，使用配置项 `hive.table.owner=Jon` 会在创建过程中自动将参数 `table.owner=Jon` 添加到表属性中。

#### 在属性中设置 Location {#setting-location-in-properties}

如果你使用的是对象存储，并且不希望 Paimon 表/数据库的 location 被 Hive 的文件系统访问，
否则可能导致诸如 "No FileSystem for scheme: s3a" 之类的错误，
那么你可以通过配置 `location-in-properties` 在表/数据库的属性中设置 location。参见
[在属性中设置表/数据库的 location](../maintenance/configurations#hivecatalogoptions)

### 创建 JDBC Catalog {#creating-jdbc-catalog}

使用 Paimon JDBC Catalog 时，对 Catalog 的更改会直接存储到 SQLite、MySQL、postgres 等关系型数据库中。

目前，锁配置仅支持 MySQL 和 SQLite。如果你使用其他类型的数据库作为 Catalog 存储，请不要配置 `lock.enabled`。

Flink 中的 Paimon JDBC Catalog 需要正确添加用于连接数据库的相应 jar 包。你应当首先下载 JDBC connector bundled jar 并将其加入 classpath，例如 MySQL、postgres。

| 数据库类型     | Bundle 名称          | SQL Client JAR                                                             |
|:--------------|:---------------------|:---------------------------------------------------------------------------|
| mysql         | mysql-connector-java | [下载](https://mvnrepository.com/artifact/mysql/mysql-connector-java)  |
| postgres      | postgresql           | [下载](https://mvnrepository.com/artifact/org.postgresql/postgresql)   |

```sql
CREATE CATALOG my_jdbc WITH (
    'type' = 'paimon',
    'metastore' = 'jdbc',
    'uri' = 'jdbc:mysql://<host>:<port>/<databaseName>',
    'jdbc.user' = '...', 
    'jdbc.password' = '...', 
    'catalog-key'='jdbc',
    'warehouse' = 'hdfs:///path/to/warehouse'
);

USE CATALOG my_jdbc;
```
你可以通过 "jdbc." 配置任意由 JDBC 声明的连接参数，不同数据库之间的连接参数可能有所不同，请根据实际情况进行配置。

你还可以通过指定 "catalog-key" 对多个 Catalog 下的数据库执行逻辑隔离。

此外，在创建 JdbcCatalog 时，你可以通过配置 "lock-key-max-length" 来指定锁键的最大长度，其默认值为 255。由于该值是 {catalog-key}.{database-name}.{table-name} 的组合，请相应地进行调整。

你可以使用前缀 `table-default.` 为该 Catalog 中创建的表定义任意默认表配置项。

## 创建表 {#create-table}

使用 Paimon Catalog 之后，你就可以创建和删除表了。在 Paimon Catalog 中创建的表由该 Catalog 管理。
当从 Catalog 中删除表时，其表文件也会被一并删除。

下面的 SQL 假设你已经注册并正在使用一个 Paimon Catalog。它在该 Catalog 的 `default` 数据库中创建了一个名为
`my_table` 的托管表，包含五列，其中 `dt`、`hh` 和 `user_id` 是主键。

```sql
CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING,
    PRIMARY KEY (dt, hh, user_id) NOT ENFORCED
);
```

你可以创建分区表：

```sql
CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING,
    PRIMARY KEY (dt, hh, user_id) NOT ENFORCED
) PARTITIONED BY (dt, hh);
```

:::info

如果你需要跨分区 Upsert（主键不包含全部分区字段），请参见[跨分区 Upsert](../primary-key-table/data-distribution#cross-partitions-upsert)模式。

:::

:::info

通过配置 [partition.expiration-time](../maintenance/manage-partitions)，可以自动删除过期的分区。

:::

### 指定统计模式 {#specify-statistics-mode}

Paimon 会自动收集数据文件的统计信息以加速查询过程。支持四种模式：

- `full`：收集完整的指标：`null_count, min, max`。
- `truncate(length)`：length 可以是任意正数，默认模式为 `truncate(16)`，表示收集 null 计数以及截断长度为 16 的最小值/最大值。
  这主要是为了避免列过大而导致 manifest（清单）文件膨胀。
- `counts`：仅收集 null 计数。
- `none`：禁用元数据统计信息的收集。

统计信息收集器模式可以通过 `'metadata.stats-mode'` 配置，默认值为 `'truncate(16)'`。
你可以通过设置 `'fields.{field_name}.stats-mode'` 来配置字段级别的模式。

对于 `none` 统计模式，默认情况下 `metadata.stats-dense-store` 为 `true`，这将显著减少 manifest 的
存储大小。但读取引擎中的 Paimon sdk 至少需要 0.9.1 或 1.0.0 或更高版本。

### 字段默认值 {#field-default-value}

Paimon 表目前支持通过 `'fields.item_id.default-value'` 在表属性中为字段设置默认值，
注意，分区字段和主键字段不能指定默认值。

## Create Table As Select {#create-table-as-select}

表可以由查询的结果创建并填充，例如，我们有这样一条 sql：`CREATE TABLE table_b AS SELECT id, name FROM table_a`，
所得到的表 `table_b` 等价于通过以下语句创建表并插入数据：
`CREATE TABLE table_b (id INT, name STRING); INSERT INTO table_b SELECT id, name FROM table_a;`

我们可以在使用 `CREATE TABLE AS SELECT` 时指定主键或分区，语法请参考下面的 sql。

```sql
/* For streaming mode, you need to enable the checkpoint. */

CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT
);
CREATE TABLE my_table_as AS SELECT * FROM my_table;

/* partitioned table */
CREATE TABLE my_table_partition (
     user_id BIGINT,
     item_id BIGINT,
     behavior STRING,
     dt STRING,
     hh STRING
) PARTITIONED BY (dt, hh);
CREATE TABLE my_table_partition_as WITH ('partition' = 'dt') AS SELECT * FROM my_table_partition;
    
/* change options */
CREATE TABLE my_table_options (
       user_id BIGINT,
       item_id BIGINT
) WITH ('file.format' = 'orc');
CREATE TABLE my_table_options_as WITH ('file.format' = 'parquet') AS SELECT * FROM my_table_options;

/* primary key */
CREATE TABLE my_table_pk (
      user_id BIGINT,
      item_id BIGINT,
      behavior STRING,
      dt STRING,
      hh STRING,
      PRIMARY KEY (dt, hh, user_id) NOT ENFORCED
);
CREATE TABLE my_table_pk_as WITH ('primary-key' = 'dt,hh') AS SELECT * FROM my_table_pk;


/* primary key + partition */
CREATE TABLE my_table_all (
      user_id BIGINT,
      item_id BIGINT,
      behavior STRING,
      dt STRING,
      hh STRING,
      PRIMARY KEY (dt, hh, user_id) NOT ENFORCED 
) PARTITIONED BY (dt, hh);
CREATE TABLE my_table_all_as WITH ('primary-key' = 'dt,hh', 'partition' = 'dt') AS SELECT * FROM my_table_all;
```

## Create Table Like {#create-table-like}

要创建一个与另一张表具有相同 schema（表结构）、分区和表属性的表，请使用 CREATE TABLE LIKE。

```sql
CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING,
    PRIMARY KEY (dt, hh, user_id) NOT ENFORCED
);

CREATE TABLE my_table_like LIKE my_table (EXCLUDING OPTIONS);
```

## 使用 Flink 临时表 {#work-with-flink-temporary-tables}

Flink 临时表只是被记录下来，但不由当前 Flink SQL 会话管理。如果删除临时表，
其资源不会被删除。当 Flink SQL 会话关闭时，临时表也会被删除。

如果你想将 Paimon Catalog 与其他表一起使用，但又不想将它们存储到其他 Catalog 中，你可以
创建一个临时表。下面的 Flink SQL 创建了一个 Paimon Catalog 和一个临时表，并演示了
如何将这两种表一起使用。

```sql
CREATE CATALOG my_catalog WITH (
    'type' = 'paimon',
    'warehouse' = 'hdfs:///path/to/warehouse'
);

USE CATALOG my_catalog;

-- Assume that there is already a table named my_table in my_catalog

CREATE TEMPORARY TABLE temp_table (
    k INT,
    v STRING
) WITH (
    'connector' = 'filesystem',
    'path' = 'hdfs:///path/to/temp_table.csv',
    'format' = 'csv'
);

SELECT my_table.k, my_table.v, temp_table.v FROM my_table JOIN temp_table ON my_table.k = temp_table.k;
```
