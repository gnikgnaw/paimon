---
title: "向量存储"
sidebar_position: 7
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

# 向量存储 {#vector-storage}

## 概述 {#overview}

随着 AI 场景的爆发式增长，向量存储变得越来越重要。

Paimon 提供了专门针对向量数据设计的优化存储方案，以满足各类场景的需求。

## 向量数据类型 {#vector-data-type}

向量数据有许多种类型，其中稠密向量（dense vector）是最常用的。它们通常表示为定长、紧密排列的数组，一般不包含 `null` 元素。

Paimon 支持定义 `VECTOR<t, n>` 类型的列，它表示一个定长、稠密的向量列，其中：
 - **`t`**：向量的元素类型。当前支持七种基本类型：`BOOLEAN`、`TINYINT`、`SMALLINT`、`INT`、`BIGINT`、`FLOAT`、`DOUBLE`；
 - **`n`**：向量维度，必须是不超过 `2,147,483,647` 的正整数；
 - **`null constraint`**：`VECTOR` 类型支持定义为 `NOT NULL` 或默认可空。不过，如果某个具体的 `VECTOR` 值本身不为 `null`，则其元素不允许为 `null`。

与变长数组相比，这些特性使得稠密向量在存储与内存表示上更加简洁，其优势包括：
 - 语义约束更自然，在数据存储层即可避免长度不匹配、`null` 元素等异常；
 - 点查（point-lookup）性能更好，无需存储和访问偏移量数组；
 - 与专用向量引擎中的类型表示更贴合，在查询时通常可以避免内存拷贝和类型转换。

示例：使用 Java API 定义一个带有 `VECTOR` 列的表并写入一行数据。
```java
public class CreateTableWithVector {

    public static void main(String[] args) throws Exception {
        // Schema
        Schema.Builder schemaBuilder = Schema.newBuilder();
        schemaBuilder.column("id", DataTypes.BIGINT());
        schemaBuilder.column("embed", DataTypes.VECTOR(3, DataTypes.FLOAT()));
        schemaBuilder.option(CoreOptions.FILE_FORMAT.key(), "lance");
        schemaBuilder.option(CoreOptions.FILE_COMPRESSION.key(), "none");
        Schema schema = schemaBuilder.build();

        // Create catalog
        String database = "default";
        String tempPath = System.getProperty("java.io.tmpdir") + UUID.randomUUID();
        Path warehouse = new Path(TraceableFileIO.SCHEME + "://" + tempPath);
        Identifier identifier = Identifier.create("default", "my_table");
        try (Catalog catalog = CatalogFactory.createCatalog(CatalogContext.create(warehouse))) {

            // Create table
            catalog.createDatabase(database, true);
            catalog.createTable(identifier, schema, true);
            FileStoreTable table = (FileStoreTable) catalog.getTable(identifier);

            // Write data
            BatchWriteBuilder builder = table.newBatchWriteBuilder();
            InternalVector vector = BinaryVector.fromPrimitiveArray(new float[] {1.0f, 2.0f, 3.0f});
            try (BatchTableWrite batchTableWrite = builder.newWrite()) {
                try (BatchTableCommit commit = builder.newCommit()) {
                    batchTableWrite.write(GenericRow.of(1L, vector));
                    commit.commit(batchTableWrite.prepareCommit());
                }
            }

            // Read data
            ReadBuilder readBuilder = table.newReadBuilder();
            TableScan.Plan plan = readBuilder.newScan().plan();
            try (RecordReader<InternalRow> reader = readBuilder.newRead().createReader(plan)) {
                reader.forEachRemaining(row -> {
                    float[] readVector = row.getVector(1).toFloatArray();
                    System.out.println(Arrays.toString(readVector));
                });
            }
        }
    }
}
```

**注意**：
 - `VECTOR` 类型的列不能用作主键列、分区列，也不能用于排序。

## 引擎层表示 {#engine-level-representation}

由于引擎层通常没有专门的向量类型，为了在引擎 SQL 中支持 `VECTOR` 类型，Paimon 提供了一个单独的配置，用于将引擎的 `ARRAY` 类型转换为 Paimon 的 `VECTOR` 类型。

用法：
 - **`'vector-field'`**：将列声明为 `VECTOR` 类型，多个列之间用逗号（`,`）分隔；
 - **`'field.{field-name}.vector-dim'`**：声明该向量列的维度。

示例：使用 Flink SQL 定义一个带有 `VECTOR` 列的表。
```sql
CREATE TABLE IF NOT EXISTS ts_table (
    id BIGINT,
    embed1 ARRAY<FLOAT>,
    embed2 ARRAY<FLOAT>
) WITH (
    'file.format' = 'lance',
    'vector-field' = 'embed1,embed2',
    'field.embed1.vector-dim' = '128',
    'field.embed2.vector-dim' = '768'
);
```

**注意**：
 - 在定义 `vector-field` 列时，必须提供向量维度，否则 CREATE TABLE 语句会失败；
 - 当前仅 Flink SQL 支持此配置，其他引擎尚未实现。

## 向量专用文件格式 {#dedicated-file-format-for-vector}

在将 `VECTOR` 类型映射到文件格式层时，理想的存储格式是 `FixedSizeList`。目前，这仅通过 `paimon-arrow` 集成在某些文件格式（例如 `lance`）上得到支持。这意味着，要使用 `VECTOR` 类型，必须通过 `file.format` 指定某种特定格式，而这会带来全局性的影响。特别地，这对标量数据和多模态（Blob）数据可能并不友好。

因此，Paimon 提供了一种方案，可以在 Data Evolution 表中将向量列单独存储。

布局：
```
table/
├── bucket-0/
│   ├── data-uuid-0.parquet      # Contains id, name columns
│   ├── data-uuid-1.blob         # Contains blob data
│   ├── data-uuid-2.vector.lance # Contains vector data using lance format
│   └── ...
├── manifest/
├── schema/
└── snapshot/
```

用法：
 - **`vector.file.format`**：将 `VECTOR` 类型的列以指定的文件格式单独存储；
 - **`vector.target-file-size`**：如果单独存储，指定向量数据的目标文件大小，默认为 `10 * 'target-file-size'`。

示例：使用 Flink SQL 将 `VECTOR` 列单独存储。
```sql
CREATE TABLE IF NOT EXISTS ts_table (
    id BIGINT,
    embed ARRAY<FLOAT>
) WITH (
    'file.format' = 'parquet',
    'vector.file.format' = 'lance',
    'vector-field' = 'embed',
    'field.embed.vector-dim' = '128',
    'row-tracking.enabled' = 'true',
    'data-evolution.enabled' = 'true'
);
```
