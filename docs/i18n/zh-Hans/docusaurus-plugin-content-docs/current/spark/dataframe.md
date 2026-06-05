---
title: "DataFrame"
sidebar_position: 9
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

# DataFrame {#dataframe}

Paimon 支持通过 Spark DataFrame API 创建表、插入数据以及查询。

## 创建表 {#create-table}
如有需要，你可以通过 `option` 指定表属性，或通过 `partitionBy` 设置分区列。

```scala
val data: DataFrame = Seq((1, "x1", "p1"), (2, "x2", "p2")).toDF("a", "b", "pt")

data.write.format("paimon")
  .option("primary-key", "a,pt")
  .option("k1", "v1")
  .partitionBy("pt")
  .saveAsTable("test_tbl") // or .save("/path/to/default.db/test_tbl")
```

## 插入 {#insert}

### Insert Into {#insert-into}
你可以通过将模式设置为 `append` 来实现 INSERT INTO 语义。

```scala
val data: DataFrame = ...

data.write.format("paimon")
  .mode("append")
  .insertInto("test_tbl") // or .saveAsTable("test_tbl") or .save("/path/to/default.db/test_tbl")
```

注意：`insertInto` 会忽略列名，仅按位置进行写入；如果你需要按列名写入，请改用 `saveAsTable` 或 `save`。

### Insert Overwrite {#insert-overwrite}
你可以通过将模式设置为 `overwrite` 并配合 `insertInto` 来实现 INSERT OVERWRITE 语义。

它支持对分区表进行动态分区覆盖。
要启用动态覆盖，你需要将 Spark 会话配置项 `spark.sql.sources.partitionOverwriteMode` 设置为 `dynamic`。

```scala
val data: DataFrame = ...

data.write.format("paimon")
  .mode("overwrite")
  .insertInto("test_tbl")
```

## 替换表 {#replace-table}
你可以通过将模式设置为 `overwrite` 并配合 `saveAsTable` 或 `save` 来实现 REPLACE TABLE 语义。

它会先删除已存在的表，然后创建一个新表，
因此如有需要，你需要指定该表的属性或分区列。

```scala
val data: DataFrame = ...

data.write.format("paimon")
  .option("primary-key", "a,pt")
  .option("k1", "v1")
  .partitionBy("pt")
  .mode("overwrite")
  .saveAsTable("test_tbl") // or .save("/path/to/default.db/test_tbl")
```

## 查询 {#query}

```scala
spark.read.format("paimon")
  .table("t") // or .load("/path/to/default.db/test_tbl")
  .show()
```

要指定 Catalog 或数据库，你可以使用

```scala
// recommend
spark.read.format("paimon")
  .table("<catalogName>.<databaseName>.<tableName>")

// or
spark.read.format("paimon")
  .option("catalog", "<catalogName>")
  .option("database", "<databaseName>")
  .option("table", "<tableName>")
  .load("/path/to/default.db/test_tbl")
```

你可以通过 option 指定其他读取配置：

```scala
// time travel
spark.read.format("paimon")
  .option("scan.snapshot-id", 1)
  .table("t")
```
