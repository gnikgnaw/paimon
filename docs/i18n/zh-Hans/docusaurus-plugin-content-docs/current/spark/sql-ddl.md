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

## Catalog {#catalog}

### 创建 Catalog {#create-catalog}

Paimon Catalog 目前支持三种类型的 metastore：

* `filesystem` metastore（默认），它将元数据和表文件都存储在文件系统中。
* `hive` metastore，它额外将元数据存储在 Hive metastore 中。用户可以直接从 Hive 访问这些表。
* `jdbc` metastore，它额外将元数据存储在 MySQL、Postgres 等关系型数据库中。

创建 Catalog 时的详细配置项请参见 [CatalogOptions](../maintenance/configurations#catalogoptions)。

#### 创建文件系统 Catalog {#create-filesystem-catalog}

下面的 Spark SQL 注册并使用了一个名为 `my_catalog` 的 Paimon Catalog。元数据和表文件存储在 `hdfs:///path/to/warehouse` 下。

下面的 shell 命令注册了一个名为 `paimon` 的 Paimon Catalog。元数据和表文件存储在 `hdfs:///path/to/warehouse` 下。

```bash
spark-sql ... \
    --conf spark.sql.catalog.paimon=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon.warehouse=hdfs:///path/to/warehouse
```

你可以使用前缀 `spark.sql.catalog.paimon.table-default.` 为在该 Catalog 中创建的表定义任意默认表配置项。

`spark-sql` 启动后，你可以使用下面的 SQL 切换到 `paimon` Catalog 的 `default` 数据库。

```sql
USE paimon.default;
```

#### 创建 Hive Catalog {#creating-hive-catalog}

通过使用 Paimon Hive Catalog，对 Catalog 的更改将直接影响对应的 Hive metastore。在这种 Catalog 中创建的表也可以直接从 Hive 访问。

要使用 Hive Catalog，数据库名、表名和字段名都应为**小写**。

你的 Spark 安装应该能够检测到，或者已经包含了 Hive 依赖。更多信息请参见[此处](https://spark.apache.org/docs/latest/sql-data-sources-hive-tables.html)。

下面的 shell 命令注册了一个名为 `paimon` 的 Paimon Hive Catalog。元数据和表文件存储在 `hdfs:///path/to/warehouse` 下。此外，元数据还会存储在 Hive metastore 中。

```bash
spark-sql ... \
    --conf spark.sql.catalog.paimon=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon.warehouse=hdfs:///path/to/warehouse \
    --conf spark.sql.catalog.paimon.metastore=hive \
    --conf spark.sql.catalog.paimon.uri=thrift://<hive-metastore-host-name>:<port>
```

你可以使用前缀 `spark.sql.catalog.paimon.table-default.` 为在该 Catalog 中创建的表定义任意默认表配置项。

`spark-sql` 启动后，你可以使用下面的 SQL 切换到 `paimon` Catalog 的 `default` 数据库。

```sql
USE paimon.default;
```

此外，你也可以创建 [SparkGenericCatalog](./quick-start)。

**将分区同步到 Hive Metastore**

默认情况下，Paimon 不会将新创建的分区同步到 Hive metastore。用户在 Hive 中会看到一张未分区的表。分区下推将改为通过过滤器下推来执行。

如果你希望在 Hive 中看到一张分区表，并且也将新创建的分区同步到 Hive metastore，请将表属性 `metastore.partitioned-table` 设置为 true。另请参见 [CoreOptions](../maintenance/configurations#coreoptions)。

#### 创建 JDBC Catalog {#creating-jdbc-catalog}

通过使用 Paimon JDBC Catalog，对 Catalog 的更改将直接存储在 SQLite、MySQL、postgres 等关系型数据库中。

目前，仅 MySQL 和 SQLite 支持锁配置。如果你使用其他类型的数据库进行 Catalog 存储，请不要配置 `lock.enabled`。

Spark 中的 Paimon JDBC Catalog 需要正确添加用于连接数据库的对应 jar 包。你应先下载 JDBC 连接器的 bundled jar 并将其添加到 classpath，例如 MySQL、postgres。

| 数据库类型 | Bundle 名称           | SQL Client JAR                                                             |
|:--------------|:---------------------|:---------------------------------------------------------------------------|
| mysql         | mysql-connector-java | [下载](https://mvnrepository.com/artifact/mysql/mysql-connector-java)  |
| postgres      | postgresql           | [下载](https://mvnrepository.com/artifact/org.postgresql/postgresql)   |

```bash
spark-sql ... \
    --conf spark.sql.catalog.paimon=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon.warehouse=hdfs:///path/to/warehouse \
    --conf spark.sql.catalog.paimon.metastore=jdbc \
    --conf spark.sql.catalog.paimon.uri=jdbc:mysql://<host>:<port>/<databaseName> \
    --conf spark.sql.catalog.paimon.jdbc.user=... \
    --conf spark.sql.catalog.paimon.jdbc.password=...
    
```

```sql
USE paimon.default;
```
#### 创建 REST Catalog {#creating-rest-catalog}

通过使用 Paimon REST Catalog，对 Catalog 的更改将直接存储在远程服务器中。

##### bear token {#bear-token}
```bash
spark-sql ... \
    --conf spark.sql.catalog.paimon=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon.metastore=rest \
    --conf spark.sql.catalog.paimon.uri=<catalog server url> \
    --conf spark.sql.catalog.paimon.token.provider=bear \
    --conf spark.sql.catalog.paimon.token=<token>
    
```

##### dlf ak {#dlf-ak}
```bash
spark-sql ... \
    --conf spark.sql.catalog.paimon=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon.metastore=rest \
    --conf spark.sql.catalog.paimon.uri=<catalog server url> \
    --conf spark.sql.catalog.paimon.token.provider=dlf \
    --conf spark.sql.catalog.paimon.dlf.access-key-id=<access-key-id> \
    --conf spark.sql.catalog.paimon.dlf.access-key-secret=<security-token>
    
```

##### dlf sts token {#dlf-sts-token}
```bash
spark-sql ... \
    --conf spark.sql.catalog.paimon=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon.metastore=rest \
    --conf spark.sql.catalog.paimon.uri=<catalog server url> \
    --conf spark.sql.catalog.paimon.token.provider=dlf \
    --conf spark.sql.catalog.paimon.dlf.access-key-id=<access-key-id> \
    --conf spark.sql.catalog.paimon.dlf.access-key-secret=<access-key-secret> \
    --conf spark.sql.catalog.paimon.dlf.security-token=<security-token>
    
    
```

```sql
USE paimon.default;
```

## Table {#table}

### 创建表 {#create-table}

使用 Paimon Catalog 后，你可以创建和删除表。在 Paimon Catalog 中创建的表由该 Catalog 管理。
当从 Catalog 中删除表时，其表文件也会被一并删除。

下面的 SQL 假设你已经注册并正在使用一个 Paimon Catalog。它在该 Catalog 的 `default` 数据库中创建了一张名为
`my_table` 的受管表（managed table），包含五列，其中 `dt`、`hh` 和 `user_id` 为主键。

```sql
CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING
) TBLPROPERTIES (
    'primary-key' = 'dt,hh,user_id'
);
```

你也可以创建分区表：

```sql
CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING
) PARTITIONED BY (dt, hh) TBLPROPERTIES (
    'primary-key' = 'dt,hh,user_id'
);
```

### 创建外部表 {#create-external-table}

当 Catalog 的 `metastore` 类型为 `hive` 时，如果在创建表时指定了 `location`，则该表将被视为外部表（external table）；否则它将是一张受管表。

当你删除一张外部表时，仅会移除 Hive 中的元数据，而实际的数据文件不会被删除；删除受管表则会同时删除数据。

```sql
CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING
) PARTITIONED BY (dt, hh) TBLPROPERTIES (
    'primary-key' = 'dt,hh,user_id'
) LOCATION '/path/to/table';
```

此外，如果指定的 location 中已经存储有数据，你可以在不显式指定字段、分区、属性或其他信息的情况下创建表。
在这种情况下，新表将从已有表的元数据中继承所有这些信息。

不过，如果你手动指定它们，需要确保它们与已有表的信息保持一致（属性可以是子集）。因此，强烈建议不要指定它们。

```sql
CREATE TABLE my_table LOCATION '/path/to/table';
```

### 通过查询结果创建表（Create Table As Select） {#create-table-as-select}

表可以通过查询结果创建并填充数据，例如，我们有这样一个 SQL：`CREATE TABLE table_b AS SELECT id, name FORM table_a`，
得到的表 `table_b` 等同于使用以下语句先创建表再插入数据：
`CREATE TABLE table_b (id INT, name STRING); INSERT INTO table_b SELECT id, name FROM table_a;`

在使用 `CREATE TABLE AS SELECT` 时，我们可以指定主键或分区，语法请参考以下 SQL。

```sql
CREATE TABLE my_table (
     user_id BIGINT,
     item_id BIGINT
);
CREATE TABLE my_table_as AS SELECT * FROM my_table;

/* partitioned table*/
CREATE TABLE my_table_partition (
      user_id BIGINT,
      item_id BIGINT,
      behavior STRING,
      dt STRING,
      hh STRING
) PARTITIONED BY (dt, hh);
CREATE TABLE my_table_partition_as PARTITIONED BY (dt) AS SELECT * FROM my_table_partition;

/* change TBLPROPERTIES */
CREATE TABLE my_table_options (
       user_id BIGINT,
       item_id BIGINT
) TBLPROPERTIES ('file.format' = 'orc');
CREATE TABLE my_table_options_as TBLPROPERTIES ('file.format' = 'parquet') AS SELECT * FROM my_table_options;


/* primary key */
CREATE TABLE my_table_pk (
     user_id BIGINT,
     item_id BIGINT,
     behavior STRING,
     dt STRING,
     hh STRING
) TBLPROPERTIES (
    'primary-key' = 'dt,hh,user_id'
);
CREATE TABLE my_table_pk_as TBLPROPERTIES ('primary-key' = 'dt') AS SELECT * FROM my_table_pk;

/* primary key + partition */
CREATE TABLE my_table_all (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING,
    dt STRING,
    hh STRING
) PARTITIONED BY (dt, hh) TBLPROPERTIES (
    'primary-key' = 'dt,hh,user_id'
);
CREATE TABLE my_table_all_as PARTITIONED BY (dt) TBLPROPERTIES ('primary-key' = 'dt,hh') AS SELECT * FROM my_table_all;
```

### 替换表（Replace Table） {#replace-table}

自 **Spark 3.4** 起，Paimon 支持在 Spark `REPLACE TABLE` 时保留快照历史。

```sql
CREATE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    behavior STRING
) TBLPROPERTIES (
    'primary-key' = 'user_id',
    'bucket' = '2'
);

INSERT INTO my_table VALUES (1, 10, 'pv');

REPLACE TABLE my_table (
    user_id BIGINT,
    item_id BIGINT,
    category STRING
) TBLPROPERTIES (
    'primary-key' = 'user_id',
    'bucket' = '4'
);
```

在 Paimon 中，这并非原子性替换。Paimon 将 Spark 的「drop+create」替换路径改为
截断（truncate）当前表并提交一个新的 schema，同时保留表的 location 和快照
历史。当前表会变为空表并使用新的 schema，但旧快照仍可通过时间旅行进行
查询。

```sql
SELECT * FROM my_table;

SELECT * FROM my_table VERSION AS OF 1;
```

`REPLACE TABLE` 要求表已存在。如果表不存在，请改用
`CREATE OR REPLACE TABLE`。

`REPLACE TABLE` 不接受 `AS SELECT`。要替换一张表并用查询结果填充它，
请使用 `CREATE OR REPLACE TABLE ... AS SELECT`。

```sql
CREATE OR REPLACE TABLE my_table
TBLPROPERTIES (
    'primary-key' = 'user_id',
    'bucket' = '4'
)
AS SELECT user_id, item_id, behavior FROM source_table;
```

当已有表与目标表使用不同的表类型时，
会改用其回退的「drop+create」行为，而非保留快照的替换
行为。

### 基于已有表创建表（Create Table Like） {#create-table-like}

可以基于一张已有的源表创建一张新表。自 **Spark 3.4** 起可用。

```sql
CREATE TABLE target_table LIKE source_table;
```

`CREATE TABLE LIKE` 会复制源表的 schema 和分区。

在 `SparkCatalog` 中，如果未指定 `USING xxx`，目标表会继承源表的 provider。

在 `SparkGenericCatalog` 中，使用 `USING paimon` 来启用 Paimon 的 `CREATE TABLE LIKE` 语义。

当 Paimon 处理该命令时，只有在源表和目标表的 provider 相同时才会复制注释和表属性。如果 provider 不同，则仅复制注释。

`path`、`provider`、`location`、`owner`、`external` 和 `is-managed-location` 永远不会被复制。用户仍可通过 `TBLPROPERTIES` 覆盖目标表。

`SparkCatalog` 不支持 `STORED AS`。在 `SparkGenericCatalog` 中，未带 `USING paimon` 的命令会使用 Spark 原生行为。

```sql
CREATE TABLE source_tbl (
    id INT,
    name STRING,
    pt STRING
) COMMENT 'source comment'
PARTITIONED BY (pt)
TBLPROPERTIES ('primary-key' = 'id,pt', 'bucket' = '5');

-- target inherits the source provider
CREATE TABLE target_tbl LIKE source_tbl;
```

## 视图 {#view}

视图基于一条 SQL 查询的结果集，当使用 `org.apache.paimon.spark.SparkCatalog` 时，视图由 Paimon 自身管理。
而在这种情况下，当 `metastore` 类型为 `hive` 或 `rest` 时才支持视图。

### 创建或替换视图（Create Or Replace View） {#create-or-replace-view}

CREATE VIEW 会构造一张没有物理数据的虚拟表。

```sql
-- create a view or a temporary view. (temporary view should not specify database name)
CREATE [TEMPORARY] VIEW <mydb>.v1 AS SELECT * FROM t1;

-- create a view or a temporary view, if a view of same name already exists, it will be replaced. (temporary view should not specify database name)
CREATE OR REPLACE [TEMPORARY] VIEW <mydb>.v1 AS SELECT * FROM t1;
```

### 删除视图（Drop View） {#drop-view}

DROP VIEW 会从 Catalog 中移除与指定视图关联的元数据。

```sql
-- drop a view or a temporary view.
DROP VIEW <mydb>.v1;
```

## 标签（Tag） {#tag}
### 创建或替换标签（Create Or Replace Tag） {#create-or-replace-tag}
创建或替换标签的语法包含以下选项。
- 创建标签时可指定或不指定快照 id 和时间保留期。
- 使用 `IF NOT EXISTS` 语法时，创建一个已存在的标签不会失败。
- 使用 `REPLACE TAG` 或 `CREATE OR REPLACE TAG` 语法更新标签。

```sql
-- create a tag based on the latest snapshot and no retention.
ALTER TABLE T CREATE TAG `TAG-1`;

-- create a tag based on the latest snapshot and no retention if it doesn't exist.
ALTER TABLE T CREATE TAG IF NOT EXISTS `TAG-1`;

-- create a tag based on the latest snapshot and retain it for 7 day.
ALTER TABLE T CREATE TAG `TAG-2` RETAIN 7 DAYS;

-- create a tag based on snapshot-1 and no retention.
ALTER TABLE T CREATE TAG `TAG-3` AS OF VERSION 1;

-- create a tag based on snapshot-2 and retain it for 12 hour.
ALTER TABLE T CREATE TAG `TAG-4` AS OF VERSION 2 RETAIN 12 HOURS;

-- replace a existed tag with new snapshot id and new retention
ALTER TABLE T REPLACE TAG `TAG-4` AS OF VERSION 2 RETAIN 24 HOURS;

-- create or replace a tag, create tag if it not exist, replace tag if it exists.
ALTER TABLE T CREATE OR REPLACE TAG `TAG-5` AS OF VERSION 2 RETAIN 24 HOURS;
```
注意：如果设置了 tag.automatic-creation，那么对于一个快照只能创建一个自动标签。

### 删除标签（Delete Tag） {#delete-tag}
删除一张表的一个或多个标签。
```sql
-- delete a tag.
ALTER TABLE T DELETE TAG `TAG-1`;

-- delete a tag if it exists.
ALTER TABLE T DELETE TAG IF EXISTS `TAG-1`

-- delete multiple tags, delimiter is ','.
ALTER TABLE T DELETE TAG `TAG-1,TAG-2`;
```

### 重命名标签（Rename Tag） {#rename-tag}
用新的标签名重命名一个已有的标签。
```sql
ALTER TABLE T RENAME TAG `TAG-1` TO `TAG-2`;
```

### 查看标签（Show Tags） {#show-tags}
列出一张表的所有标签。
```sql
SHOW TAGS T;
```
