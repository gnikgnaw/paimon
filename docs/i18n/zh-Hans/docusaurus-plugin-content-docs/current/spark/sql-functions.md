---
title: "SQL 函数"
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

# SQL 函数 {#sql-functions}

本节介绍所有可用的 Paimon Spark 函数。

## 内置函数 {#built-in-function}

### max_pt {#max_pt}

`sys.max_pt($table_name)`

它接受一个字符串类型的字面量来指定表名，并返回最大的有效顶层分区值。
- **valid（有效）**：包含数据文件的分区
- **toplevel（顶层）**：如果表有多个分区列，则只返回第一个分区值

在以下情况下会抛出异常：
- 该表不是分区表
- 该分区表没有分区
- 所有分区都不包含数据文件

**示例**

```sql
SELECT sys.max_pt('t');
-- 20250101
 
SELECT * FROM t where pt = sys.max_pt('t');
-- a, 20250101
```

### path_to_descriptor {#path_to_descriptor}

`sys.path_to_descriptor($file_path)`

将文件路径（STRING）转换为 Blob 描述符（BINARY）。该函数在处理存储于外部文件中的 Blob 数据时非常有用。它会创建一个引用指定路径下文件的 Blob 描述符。

**参数：**
- `file_path`（STRING）：包含 Blob 数据的外部文件的路径。

**返回值：**
- 一个表示序列化后的 Blob 描述符的 BINARY 值。

**示例**

```sql
-- Insert blob data using path_to_descriptor function
INSERT INTO t VALUES ('1', 'paimon', sys.path_to_descriptor('file:///path/to/blob_file'));

-- Insert with partition
INSERT OVERWRITE TABLE t PARTITION(ds='1017', batch='test')
VALUES ('1', 'paimon', '1024', '12345678', '20241017', sys.path_to_descriptor('file:///path/to/blob_file'));
```

### descriptor_to_string {#descriptor_to_string}

`sys.descriptor_to_string($descriptor)`

将 Blob 描述符（BINARY）转换为其字符串表示形式（STRING）。该函数在调试或以人类可读的格式显示 Blob 描述符的内容时非常有用。

**参数：**
- `descriptor`（BINARY）：要转换的 Blob 描述符字节。

**返回值：**
- Blob 描述符的 STRING 表示形式。

**示例**

```sql
-- Convert a blob descriptor to string for inspection
SELECT sys.descriptor_to_string(content) FROM t WHERE id = '1';
-- [BlobDescriptor{version=1', uri='/path/to/data-2c103f6f-3857-4062-abc3-2e260374a68e-1.blob', offset=4, length=1048576}]
```

## 用户自定义函数 {#user-defined-function}

Paimon Spark 支持两种类型的用户自定义函数：lambda 函数和基于文件的函数。

该特性目前仅支持 REST Catalog。

### Lambda 函数 {#lambda-function}

允许用户使用 Java lambda 表达式来定义函数，从而实现内联、简洁且函数式风格的操作。

**示例**

```sql
-- Create Function
CALL sys.create_function(`function` => 'my_db.area_func',
  `inputParams` => '[{"id": 0, "name":"length", "type":"INT"}, {"id": 1, "name":"width", "type":"INT"}]',
  `returnParams` => '[{"id": 0, "name":"area", "type":"BIGINT"}]',
  `deterministic` => true,
  `comment` => 'comment',
  `options` => 'k1=v1,k2=v2'
);

-- Alter Function
CALL sys.alter_function(`function` => 'my_db.area_func',
  `change` => '{"action" : "addDefinition", "name" : "spark", "definition" : {"type" : "lambda", "definition" : "(Integer length, Integer width) -> { return (long) length * width; }", "language": "JAVA" } }'
);

-- Drop Function
CALL sys.drop_function(`function` => 'my_db.area_func');
```

### File 函数 {#file-function}

用户可以在文件中定义函数，为函数定义提供灵活性和模块化支持，目前仅支持 jar 文件。

目前支持 UDF 和 UDAF 的 Spark 或 Hive 实现，参见 [Spark UDFs](https://spark.apache.org/docs/latest/sql-ref-functions.html#udfs-user-defined-functions)

该特性要求 Spark 3.4 或更高版本。

**示例**

```sql
-- Create Function or Temporary Function (Temporary function should not specify database name)
CREATE [TEMPORARY] FUNCTION <mydb>.simple_udf
AS 'com.example.SimpleUdf' 
USING JAR '/tmp/SimpleUdf.jar' [, JAR '/tmp/SimpleUdfR.jar'];

-- Create or Replace Temporary Function (Temporary function should not specify database name)
CREATE OR REPLACE [TEMPORARY] FUNCTION <mydb>.simple_udf 
AS 'com.example.SimpleUdf'
USING JAR '/tmp/SimpleUdf.jar';
       
-- Describe Function
DESCRIBE FUNCTION [EXTENDED] <mydb>.simple_udf;

-- Drop Function
DROP [TEMPORARY] FUNCTION <mydb>.simple_udf;
```
