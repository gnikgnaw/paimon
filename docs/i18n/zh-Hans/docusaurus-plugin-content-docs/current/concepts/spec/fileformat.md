---
title: "FileFormat"
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

# 文件格式 {#file-format}

目前支持 Parquet、Avro、ORC、CSV、JSON 以及 Lance 文件格式。
- 推荐的列式格式是 Parquet，它具有较高的压缩率以及较快的列投影查询能力。
- 推荐的行式格式是 Avro，它在读写完整行（所有列）时具有良好的性能。
- 推荐用于测试的格式是 CSV，它具有更好的可读性，但读写性能最差。
- 推荐用于机器学习（ML）工作负载的格式是 Lance，它针对向量检索和机器学习场景进行了优化。

## PARQUET {#parquet}

Parquet 是 Paimon 的默认文件格式。

下表列出了从 Paimon 类型到 Parquet 类型的类型映射。

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">Paimon 类型</th>
        <th class="text-center">Parquet 类型</th>
        <th class="text-center">Parquet 逻辑类型</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td>CHAR / VARCHAR / STRING</td>
      <td>BINARY</td>
      <td>UTF8</td>
    </tr>
    <tr>
      <td>BOOLEAN</td>
      <td>BOOLEAN</td>
      <td></td>
    </tr>
    <tr>
      <td>BINARY / VARBINARY</td>
      <td>BINARY</td>
      <td></td>
    </tr>
    <tr>
      <td>DECIMAL(P, S)</td>
      <td>P <= 9: INT32, P <= 18: INT64, P > 18: FIXED_LEN_BYTE_ARRAY</td>
      <td>DECIMAL(P, S)</td>
    </tr>
    <tr>
      <td>TINYINT</td>
      <td>INT32</td>
      <td>INT_8</td>
    </tr>
    <tr>
      <td>SMALLINT</td>
      <td>INT32</td>
      <td>INT_16</td>
    </tr>
    <tr>
      <td>INT</td>
      <td>INT32</td>
      <td></td>
    </tr>
    <tr>
      <td>BIGINT</td>
      <td>INT64</td>
      <td></td>
    </tr>
    <tr>
      <td>FLOAT</td>
      <td>FLOAT</td>
      <td></td>
    </tr>
    <tr>
      <td>DOUBLE</td>
      <td>DOUBLE</td>
      <td></td>
    </tr>
    <tr>
      <td>DATE</td>
      <td>INT32</td>
      <td>DATE</td>
    </tr>
    <tr>
      <td>TIME</td>
      <td>INT32</td>
      <td>TIME_MILLIS</td>
    </tr>
    <tr>
      <td>TIMESTAMP(P)</td>
      <td>P <= 3: INT64, P <= 6: INT64, P > 6: INT96</td>
      <td>P <= 3: MILLIS, P <= 6: MICROS, P > 6: NONE</td>
    </tr>
    <tr>
      <td>TIMESTAMP_LOCAL_ZONE(P)</td>
      <td>P <= 3: INT64, P <= 6: INT64, P > 6: INT96</td>
      <td>P <= 3: MILLIS, P <= 6: MICROS, P > 6: NONE</td>
    </tr>
    <tr>
      <td>ARRAY</td>
      <td>3-LEVEL LIST</td>
      <td>LIST</td>
    </tr>
    <tr>
      <td>MAP</td>
      <td>3-LEVEL MAP</td>
      <td>MAP</td>
    </tr>
    <tr>
      <td>MULTISET</td>
      <td>3-LEVEL MAP</td>
      <td>MAP</td>
    </tr>
    <tr>
      <td>ROW</td>
      <td>GROUP</td>
      <td></td>
    </tr>
    </tbody>
</table>

限制：
1. [Parquet 不支持可为空的 map 键](https://github.com/apache/parquet-format/blob/master/LogicalTypes#maps)。
2. 精度为 9 的 Parquet TIMESTAMP 类型将使用 INT96，但该 int96 是经过时区转换的值，需要额外的调整。

## AVRO {#avro}

下表列出了从 Paimon 类型到 Avro 类型的类型映射。

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">Paimon 类型</th>
        <th class="text-left">Avro 类型</th>
        <th class="text-left">Avro 逻辑类型</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td>CHAR / VARCHAR / STRING</td>
      <td>string</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BOOLEAN</code></td>
      <td><code>boolean</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>BINARY / VARBINARY</code></td>
      <td><code>bytes</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>DECIMAL</code></td>
      <td><code>bytes</code></td>
      <td><code>decimal</code></td>
    </tr>
    <tr>
      <td><code>TINYINT</code></td>
      <td><code>int</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>SMALLINT</code></td>
      <td><code>int</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>INT</code></td>
      <td><code>int</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>BIGINT</code></td>
      <td><code>long</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>FLOAT</code></td>
      <td><code>float</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>DOUBLE</code></td>
      <td><code>double</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>DATE</code></td>
      <td><code>int</code></td>
      <td><code>date</code></td>
    </tr>
    <tr>
      <td><code>TIME</code></td>
      <td><code>int</code></td>
      <td><code>time-millis</code></td>
    </tr>
    <tr>
      <td><code>TIMESTAMP</code></td>
      <td>P <= 3: long, P <= 6: long, P > 6: unsupported</td>
      <td>P <= 3: timestampMillis, P <= 6: timestampMicros, P > 6: unsupported</td>
    </tr>
    <tr>
      <td><code>TIMESTAMP_LOCAL_ZONE</code></td>
      <td>P <= 3: long, P <= 6: long, P > 6: unsupported</td>
      <td>P <= 3: localTimestampMillis, P <= 6: localTimestampMicros, P > 6: unsupported</td>
    </tr>
    <tr>
      <td><code>ARRAY</code></td>
      <td><code>array</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>MAP</code><br>
      （键必须为 string/char/varchar 类型）</td>
      <td><code>map</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>MULTISET</code><br>
      （元素必须为 string/char/varchar 类型）</td>
      <td><code>map</code></td>
      <td></td>
    </tr>
    <tr>
      <td><code>ROW</code></td>
      <td><code>record</code></td>
      <td></td>
    </tr>
    </tbody>
</table>

注意：

除上述列出的类型外，对于可为空的类型，Paimon 会将可为空类型映射为 Avro 的 `union(something, null)`，
其中 `something` 是从 Paimon 类型转换而来的 Avro 类型。

你可以参考 [Avro 规范](https://avro.apache.org/docs/1.12.0/specification/)，以了解更多关于 Avro 类型的信息。

## ORC {#orc}

下表列出了从 Paimon 类型到 Orc 类型的类型映射。

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">Paimon 类型</th>
        <th class="text-center">Orc 物理类型</th>
        <th class="text-center">Orc 逻辑类型</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td>CHAR</td>
      <td>bytes</td>
      <td>CHAR</td>
    </tr>
    <tr>
      <td>VARCHAR</td>
      <td>bytes</td>
      <td>VARCHAR</td>
    </tr>
    <tr>
      <td>STRING</td>
      <td>bytes</td>
      <td>STRING</td>
    </tr>
    <tr>
      <td>BOOLEAN</td>
      <td>long</td>
      <td>BOOLEAN</td>
    </tr>
    <tr>
      <td>BYTES</td>
      <td>bytes</td>
      <td>BINARY</td>
    </tr>
    <tr>
      <td>DECIMAL</td>
      <td>decimal</td>
      <td>DECIMAL</td>
    </tr>
    <tr>
      <td>TINYINT</td>
      <td>long</td>
      <td>BYTE</td>
    </tr>
    <tr>
      <td>SMALLINT</td>
      <td>long</td>
      <td>SHORT</td>
    </tr>
    <tr>
      <td>INT</td>
      <td>long</td>
      <td>INT</td>
    </tr>
    <tr>
      <td>BIGINT</td>
      <td>long</td>
      <td>LONG</td>
    </tr>
    <tr>
      <td>FLOAT</td>
      <td>double</td>
      <td>FLOAT</td>
    </tr>
    <tr>
      <td>DOUBLE</td>
      <td>double</td>
      <td>DOUBLE</td>
    </tr>
    <tr>
      <td>DATE</td>
      <td>long</td>
      <td>DATE</td>
    </tr>
    <tr>
      <td>TIMESTAMP</td>
      <td>timestamp</td>
      <td>TIMESTAMP</td>
    </tr>
    <tr>
      <td>TIMESTAMP_LOCAL_ZONE</td>
      <td>timestamp</td>
      <td>TIMESTAMP_INSTANT</td>
    </tr>
    <tr>
      <td>ARRAY</td>
      <td>-</td>
      <td>LIST</td>
    </tr>
    <tr>
      <td>MAP</td>
      <td>-</td>
      <td>MAP</td>
    </tr>
    <tr>
      <td>ROW</td>
      <td>-</td>
      <td>STRUCT</td>
    </tr>
    </tbody>
</table>

限制：
1. ORC 在映射 `TIMESTAMP_LOCAL_ZONE` 类型时存在时区偏差，会保存与 UTC
   字面时间对应的毫秒值。由于兼容性问题，这一行为无法修改。

## CSV {#csv}

实验性特性，不推荐用于生产环境。

格式配置项：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 25%">配置项</th>
        <th class="text-center" style="width: 7%">默认值</th>
        <th class="text-center" style="width: 10%">类型</th>
        <th class="text-center" style="width: 42%">描述</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>csv.field-delimiter</h5></td>
      <td style="word-wrap: break-word;"><code>,</code></td>
      <td>String</td>
      <td>字段分隔符（默认为 <code>','</code>），必须为单个字符。你可以使用反斜杠来指定特殊字符，例如 <code>'\t'</code> 表示制表符。
      </td>
    </tr>
    <tr>
      <td><h5>csv.line-delimiter</h5></td>
      <td style="word-wrap: break-word;"><code>\n</code></td>
      <td>String</td>
      <td>CSV 格式的行分隔符</td>
    </tr>
    <tr>
      <td><h5>csv.quote-character</h5></td>
      <td style="word-wrap: break-word;"><code>"</code></td>
      <td>String</td>
      <td>用于包裹字段值的引用字符（默认为 <code>"</code>）。</td>
    </tr>
    <tr>
      <td><h5>csv.escape-character</h5></td>
      <td style="word-wrap: break-word;">\</td>
      <td>String</td>
      <td>CSV 格式的转义字符。</td>
    </tr>
   <tr>
      <td><h5>csv.include-header</h5></td>
      <td style="word-wrap: break-word;">false</td>
      <td>Boolean</td>
      <td>是否在 CSV 文件中包含表头。</td>
    </tr>
    <tr>
      <td><h5>csv.null-literal</h5></td>
      <td style="word-wrap: break-word;"><code>""</code></td>
      <td>String</td>
      <td>被解释为 null 值的 null 字面量字符串（默认禁用）。</td>
    </tr>
    <tr>
      <td><h5>csv.mode</h5></td>
      <td style="word-wrap: break-word;"><code>PERMISSIVE</code></td>
      <td>String</td>
      <td>允许在读取时处理损坏记录的模式。当前支持的取值有 <code>'PERMISSIVE'</code>、<code>'DROPMALFORMED'</code> 和 <code>'FAILFAST'</code>：
      <ul>
      <li><code>'PERMISSIVE'</code> 选项会将格式错误的字段设置为 null。</li>
      <li><code>'DROPMALFORMED'</code> 选项会忽略整条损坏的记录。</li>
      <li><code>'FAILFAST'</code> 选项会在遇到损坏记录时抛出异常。</li>
      </ul>
      </td>
    </tr>
    </tbody>
</table>

Paimon 的 CSV 格式使用 [jackson databind API](https://github.com/FasterXML/jackson-databind) 来解析和生成 CSV 字符串。

下表列出了从 Paimon 类型到 CSV 类型的类型映射。

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">Paimon 类型</th>
        <th class="text-left">CSV 类型</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td><code>CHAR / VARCHAR / STRING</code></td>
      <td><code>string</code></td>
    </tr>
    <tr>
      <td><code>BOOLEAN</code></td>
      <td><code>boolean</code></td>
    </tr>
    <tr>
      <td><code>BINARY / VARBINARY</code></td>
      <td><code>string with encoding: base64</code></td>
    </tr>
    <tr>
      <td><code>DECIMAL</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>TINYINT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>SMALLINT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>INT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>BIGINT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>FLOAT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>DOUBLE</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>DATE</code></td>
      <td><code>string with format: date</code></td>
    </tr>
    <tr>
      <td><code>TIME</code></td>
      <td><code>string with format: time</code></td>
    </tr>
    <tr>
      <td><code>TIMESTAMP</code></td>
      <td><code>string with format: date-time</code></td>
    </tr>
    <tr>
      <td><code>TIMESTAMP_LOCAL_ZONE</code></td>
      <td><code>string with format: date-time</code></td>
    </tr>
    </tbody>
</table>

## TEXT {#text}

实验性特性，不推荐用于生产环境。

格式配置项：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 25%">配置项</th>
        <th class="text-center" style="width: 7%">默认值</th>
        <th class="text-center" style="width: 10%">类型</th>
        <th class="text-center" style="width: 42%">描述</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>text.line-delimiter</h5></td>
      <td style="word-wrap: break-word;"><code>\n</code></td>
      <td>String</td>
      <td>TEXT 格式的行分隔符</td>
    </tr>
    </tbody>
</table>

Paimon 的 text 表只包含一个字段，且该字段为 string 类型。

## JSON {#json}

实验性特性，不推荐用于生产环境。

格式配置项：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 25%">配置项</th>
        <th class="text-center" style="width: 7%">默认值</th>
        <th class="text-center" style="width: 10%">类型</th>
        <th class="text-center" style="width: 42%">描述</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>json.ignore-parse-errors</h5></td>
      <td style="word-wrap: break-word;">false</td>
      <td>Boolean</td>
      <td>是否忽略 JSON 格式的解析错误。跳过存在解析错误的字段和行，而不是直接失败。出错时字段会被设置为 null。</td>
    </tr>
    <tr>
      <td><h5>json.map-null-key-mode</h5></td>
      <td style="word-wrap: break-word;"><code>FAIL</code></td>
      <td>String</td>
      <td>如何处理为 null 的 map 键。当前支持的取值有 <code>'FAIL'</code>、<code>'DROP'</code> 和 <code>'LITERAL'</code>：
      <ul>
      <li><code>'FAIL'</code> 选项会在遇到带有 null 键的 map 时抛出异常。</li>
      <li><code>'DROP'</code> 选项会丢弃 map 中 null 键的条目。</li>
      <li><code>'LITERAL'</code> 选项会用字符串字面量替换 null 键。该字符串字面量由 <code>json.map-null-key-literal</code> 配置项定义。</li>
      </ul>
      </td>
    </tr>
    <tr>
      <td><h5>json.map-null-key-literal</h5></td>
      <td style="word-wrap: break-word;"><code>null</code></td>
      <td>String</td>
      <td>当 <code>json.map-null-key-mode</code> 为 LITERAL 时，用于 null map 键的字面量。</td>
    </tr>
    <tr>
      <td><h5>json.line-delimiter</h5></td>
      <td style="word-wrap: break-word;"><code>\n</code></td>
      <td>String</td>
      <td>JSON 格式的行分隔符。</td>
    </tr>
    </tbody>
</table>

Paimon 的 JSON 格式使用 [jackson databind API](https://github.com/FasterXML/jackson-databind) 来解析和生成 JSON 字符串。

下表列出了从 Paimon 类型到 JSON 类型的类型映射。

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">Paimon 类型</th>
        <th class="text-left">JSON 类型</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td><code>CHAR / VARCHAR / STRING</code></td>
      <td><code>string</code></td>
    </tr>
    <tr>
      <td><code>BOOLEAN</code></td>
      <td><code>boolean</code></td>
    </tr>
    <tr>
      <td><code>BINARY / VARBINARY</code></td>
      <td><code>string with encoding: base64</code></td>
    </tr>
    <tr>
      <td><code>DECIMAL</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>TINYINT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>SMALLINT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>INT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>BIGINT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>FLOAT</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>DOUBLE</code></td>
      <td><code>number</code></td>
    </tr>
    <tr>
      <td><code>DATE</code></td>
      <td><code>string with format: date</code></td>
    </tr>
    <tr>
      <td><code>TIME</code></td>
      <td><code>string with format: time</code></td>
    </tr>
    <tr>
      <td><code>TIMESTAMP</code></td>
      <td><code>string with format: date-time</code></td>
    </tr>
    <tr>
      <td><code>TIMESTAMP_LOCAL_ZONE</code></td>
      <td><code>string with format: date-time (with UTC time zone)</code></td>
    </tr>
    <tr>
      <td><code>ARRAY</code></td>
      <td><code>array</code></td>
    </tr>
    <tr>
      <td><code>MAP</code></td>
      <td><code>object</code></td>
    </tr>
    <tr>
      <td><code>MULTISET</code></td>
      <td><code>object</code></td>
    </tr>
    <tr>
      <td><code>ROW</code></td>
      <td><code>object</code></td>
    </tr>
    </tbody>
</table>

## LANCE {#lance}

Lance 是一种针对机器学习和向量检索工作负载优化的现代列式数据格式。它提供高性能的读写操作，并原生支持 Apache Arrow。

下表列出了从 Paimon 类型到 Lance（Arrow）类型的类型映射。

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">Paimon 类型</th>
        <th class="text-center">Lance（Arrow）类型</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td>CHAR / VARCHAR / STRING</td>
      <td>UTF8</td>
    </tr>
    <tr>
      <td>BOOLEAN</td>
      <td>BOOL</td>
    </tr>
    <tr>
      <td>BINARY / VARBINARY</td>
      <td>BINARY</td>
    </tr>
    <tr>
      <td>DECIMAL(P, S)</td>
      <td>DECIMAL128(P, S)</td>
    </tr>
    <tr>
      <td>TINYINT</td>
      <td>INT8</td>
    </tr>
    <tr>
      <td>SMALLINT</td>
      <td>INT16</td>
    </tr>
    <tr>
      <td>INT</td>
      <td>INT32</td>
    </tr>
    <tr>
      <td>BIGINT</td>
      <td>INT64</td>
    </tr>
    <tr>
      <td>FLOAT</td>
      <td>FLOAT</td>
    </tr>
    <tr>
      <td>DOUBLE</td>
      <td>DOUBLE</td>
    </tr>
    <tr>
      <td>DATE</td>
      <td>DATE32</td>
    </tr>
    <tr>
      <td>TIME</td>
      <td>TIME32 / TIME64</td>
    </tr>
    <tr>
      <td>TIMESTAMP(P)</td>
      <td>TIMESTAMP（单位基于精度）</td>
    </tr>
    <tr>
      <td>ARRAY</td>
      <td>LIST</td>
    </tr>
    <tr>
      <td>MULTISET</td>
      <td>LIST</td>
    </tr>
    <tr>
      <td>ROW</td>
      <td>STRUCT</td>
    </tr>
    </tbody>
</table>

限制：
1. Lance 文件格式不支持 `MAP` 类型。
2. Lance 文件格式不支持 `TIMESTAMP_LOCAL_ZONE` 类型。

## BLOB {#blob}

BLOB 格式是一种专用格式，用于存储大型二进制对象，例如图像、视频以及其他多模态数据。与其他将数据内联存储的格式不同，BLOB 格式将大型二进制数据存储在单独的文件中，并采用针对随机访问优化的布局。

BLOB 文件使用 `.blob` 扩展名，并具有以下结构：

```
+------------------+
| Blob Entry 1     |
|   Magic Number   |  4 bytes (1481511375, Little Endian)
|   Blob Data      |  Variable length
|   Length         |  8 bytes (Little Endian)
|   CRC32          |  4 bytes (Little Endian)
+------------------+
| Blob Entry 2     |
|   ...            |
+------------------+
| Index            |  Variable (Delta-Varint compressed)
+------------------+
| Index Length     |  4 bytes (Little Endian)
| Version          |  1 byte
+------------------+
```

主要特性：
- **CRC32 校验和**：每个 blob 条目都带有一个 CRC32 校验和，用于数据完整性校验
- **索引化访问**：文件末尾的索引可对文件中的任意 blob 进行高效的随机访问
- **Delta-Varint 压缩**：索引采用 delta-varint 压缩以提升空间利用率

限制：
1. BLOB 格式每个文件仅支持单个 BLOB 类型的列。
2. BLOB 格式不支持谓词下推。
3. BLOB 列不支持统计信息收集。

有关使用细节、配置项和示例，请参见 [Blob 类型](../../append-table/blob)。
