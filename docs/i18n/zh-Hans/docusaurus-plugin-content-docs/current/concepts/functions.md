---
title: "函数"
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

# 函数 {#functions}

Paimon 引入了一个函数（Function）抽象，旨在以一种面向计算引擎的标准格式来支持函数，以解决以下问题：

- **统一的列级过滤与处理：** 便于在列级别执行操作，包括对数据进行加密和解密等任务。

- **参数化视图能力：** 支持在视图中执行参数化操作，提升数据检索过程的动态性与易用性。

## 支持的函数类型 {#types-of-functions-supported}

目前，Paimon 支持三种类型的函数：

1. **文件函数（File Function）：** 用户可以在文件中定义函数，为函数定义提供灵活性和模块化支持。

2. **Lambda 函数（Lambda Function）：** 让用户能够使用 Java lambda 表达式来定义函数，从而实现内联、简洁且函数式风格的操作。

3. **SQL 函数（SQL Function）：** 用户可以直接在 SQL 中定义函数，与基于 SQL 的数据处理无缝集成。

## 在 Flink 中使用文件函数 {#file-function-usage-in-flink}

Paimon 函数可以在 Apache Flink 中使用，以执行复杂的数据操作。下面是在 Flink 环境中创建、修改和删除函数的 SQL 命令。

### 创建函数 {#create-function}

在 Flink SQL 中创建一个新函数：

```sql
-- Flink SQL
CREATE FUNCTION mydb.parse_str
    AS 'com.streaming.flink.udf.StrUdf' 
    LANGUAGE JAVA
    USING JAR 'oss://my_bucket/my_location/udf.jar' [, JAR 'oss://my_bucket/my_location/a.jar'];
```

该语句在 `mydb` 数据库中创建了一个名为 `parse_str` 的基于 Java 的用户自定义函数，并使用对象存储位置中指定的 JAR 文件。

### 修改函数 {#alter-function}

在 Flink SQL 中修改一个已有的函数：

```sql
-- Flink SQL
ALTER FUNCTION mydb.parse_str
    AS 'com.streaming.flink.udf.StrUdf2' 
    LANGUAGE JAVA;
```

该命令将 `parse_str` 函数的实现更改为使用新的 Java 类定义。

### 删除函数 {#drop-function}

在 Flink SQL 中移除一个函数：

```sql
-- Flink SQL
DROP FUNCTION mydb.parse_str;
```

该语句从 `mydb` 数据库中删除已有的 `parse_str` 函数，放弃其功能。

## Spark 中的函数 {#functions-in-spark}

参见 [SQL 函数](../spark/sql-functions#user-defined-function)
