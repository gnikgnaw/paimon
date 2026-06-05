---
title: "数据类型"
sidebar_position: 8
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

# 数据类型 {#data-types}

数据类型描述了表生态系统中某个值的逻辑类型。它可用于声明操作的输入和/或输出类型。

Paimon 支持的所有数据类型如下：

<table class="table table-bordered">
    <thead>
    <tr>
      <th class="text-left" style="width: 10%">数据类型</th>
      <th class="text-left" style="width: 30%">说明</th>
    </tr>
    </thead>
    <tbody>
    <tr>
      <td><code>BOOLEAN</code></td>
      <td><code>布尔类型，采用（可能的）三值逻辑：TRUE、FALSE 和 UNKNOWN。</code></td>
    </tr>
    <tr>
      <td><code>CHAR</code><br>
          <code>CHAR(n)</code>
      </td>
      <td><code>定长字符串的数据类型。</code><br><br>
          <code>该类型可使用 CHAR(n) 声明，其中 n 为码点（code point）数量。n 的取值范围必须在 1 到 2,147,483,647 之间（含两端）。若未指定长度，则 n 等于 1。</code>
      </td>
    </tr>
    <tr>
      <td><code>VARCHAR</code><br>
          <code>VARCHAR(n)</code><br><br>
          <code>STRING</code>
      </td>
      <td><code>变长字符串的数据类型。</code><br><br>
          <code>该类型可使用 VARCHAR(n) 声明，其中 n 为码点（code point）的最大数量。n 的取值范围必须在 1 到 2,147,483,647 之间（含两端）。若未指定长度，则 n 等于 1。</code><br><br>
          <code>STRING 是 VARCHAR(2147483647) 的同义词。</code>
      </td>
    </tr>
    <tr>
      <td><code>BINARY</code><br>
          <code>BINARY(n)</code><br><br>
      </td>
      <td><code>定长二进制字符串（即字节序列）的数据类型。</code><br><br>
          <code>该类型可使用 BINARY(n) 声明，其中 n 为字节数。n 的取值范围必须在 1 到 2,147,483,647 之间（含两端）。若未指定长度，则 n 等于 1。</code>
      </td>
    </tr>
    <tr>
      <td><code>VARBINARY</code><br>
          <code>VARBINARY(n)</code><br><br>
          <code>BYTES</code>
      </td>
      <td><code>变长二进制字符串（即字节序列）的数据类型。</code><br><br>
          <code>该类型可使用 VARBINARY(n) 声明，其中 n 为字节的最大数量。n 的取值范围必须在 1 到 2,147,483,647 之间（含两端）。若未指定长度，则 n 等于 1。</code><br><br>
          <code>BYTES 是 VARBINARY(2147483647) 的同义词。</code>
      </td>
    </tr>
    <tr>
      <td><code>DECIMAL</code><br>
          <code>DECIMAL(p)</code><br>
          <code>DECIMAL(p, s)</code>
      </td>
      <td><code>具有固定精度（precision）和小数位数（scale）的十进制数的数据类型。</code><br><br>
          <code>该类型可使用 DECIMAL(p, s) 声明，其中 p 为数字的总位数（精度），s 为数字小数点右侧的位数（小数位数）。p 的取值范围必须在 1 到 38 之间（含两端）。s 的取值范围必须在 0 到 p 之间（含两端）。p 的默认值为 10。s 的默认值为 0。</code>
      </td>
    </tr>
    <tr>
      <td><code>TINYINT</code></td>
      <td><code>1 字节有符号整数的数据类型，取值范围为 -128 到 127。</code></td>
    </tr>
    <tr>
      <td><code>SMALLINT</code></td>
      <td><code>2 字节有符号整数的数据类型，取值范围为 -32,768 到 32,767。</code></td>
    </tr>
    <tr>
      <td><code>INT</code></td>
      <td><code>4 字节有符号整数的数据类型，取值范围为 -2,147,483,648 到 2,147,483,647。</code></td>
    </tr>
    <tr>
      <td><code>BIGINT</code></td>
      <td><code>8 字节有符号整数的数据类型，取值范围为 -9,223,372,036,854,775,808 到 9,223,372,036,854,775,807。</code></td>
    </tr>
    <tr>
      <td><code>FLOAT</code></td>
      <td><code>4 字节单精度浮点数的数据类型。</code><br><br>
          <code>与 SQL 标准相比，该类型不接受参数。</code>
      </td>
    </tr>
    <tr>
      <td><code>DOUBLE</code></td>
      <td><code>8 字节双精度浮点数的数据类型。</code></td>
    </tr>
    <tr>
      <td><code>DATE</code></td>
      <td><code>由 年-月-日 组成的日期的数据类型，取值范围为 0000-01-01 到 9999-12-31。</code><br><br>
          <code>与 SQL 标准相比，取值范围从 0000 年开始。</code>
      </td>
    </tr>
    <tr>
      <td><code>TIME</code><br>
          <code>TIME(p)</code>
      </td>
      <td><code>不带时区的时间的数据类型，由 时:分:秒[.小数] 组成，最高可达纳秒精度，取值范围为 00:00:00.000000000 到 23:59:59.999999999。</code><br><br>
          <code>该类型可使用 TIME(p) 声明，其中 p 为小数秒的位数（精度）。p 的取值范围必须在 0 到 9 之间（含两端）。若未指定精度，则 p 等于 0。</code>
      </td>
    </tr>
    <tr>
      <td><code>TIMESTAMP</code><br>
          <code>TIMESTAMP(p)</code>
      </td>
      <td><code>不带时区的时间戳的数据类型，由 年-月-日 时:分:秒[.小数] 组成，最高可达纳秒精度，取值范围为 0000-01-01 00:00:00.000000000 到 9999-12-31 23:59:59.999999999。</code><br><br>
          <code>该类型可使用 TIMESTAMP(p) 声明，其中 p 为小数秒的位数（精度）。p 的取值范围必须在 0 到 9 之间（含两端）。若未指定精度，则 p 等于 6。</code>
      </td>
    </tr>
    <tr>
      <td><code>TIMESTAMP WITH LOCAL TIME ZONE</code><br>
          <code>TIMESTAMP(p) WITH LOCAL TIME ZONE</code>
      </td>
      <td><code>带本地时区的时间戳的数据类型，由 年-月-日 时:分:秒[.小数] 时区 组成，最高可达纳秒精度，取值范围为 0000-01-01 00:00:00.000000000 +14:59 到 9999-12-31 23:59:59.999999999 -14:59。</code><br><br>
          <code>该类型通过允许根据所配置的会话时区来解释 UTC 时间戳，填补了无时区与强制时区时间戳类型之间的空白。与 int 之间的转换表示自纪元（epoch）以来的秒数。与 long 之间的转换表示自纪元以来的毫秒数。</code>
      </td>
    </tr>
    <tr>
      <td><code>ARRAY&lt;t&gt;</code></td>
      <td><code>由相同子类型的元素组成的数组的数据类型。</code><br><br>
          <code>与 SQL 标准相比，数组的最大基数无法指定，而是固定为 2,147,483,647。此外，任何有效类型均可作为子类型。</code><br><br>
          <code>该类型可使用 ARRAY&lt;t&gt; 声明，其中 t 为所包含元素的数据类型。</code>
      </td>
    </tr>
    <tr>
      <td><code>MAP&lt;kt, vt&gt;</code></td>
      <td><code>关联数组的数据类型，将键（包括 NULL）映射到值（包括 NULL）。Map 不能包含重复的键；每个键最多映射到一个值。</code><br><br>
          <code>对元素类型没有限制；确保唯一性由用户负责。</code><br><br>
          <code>该类型可使用 MAP&lt;kt, vt&gt; 声明，其中 kt 为键元素的数据类型，vt 为值元素的数据类型。</code>
      </td>
    </tr>
    <tr>
      <td><code>MULTISET&lt;t&gt;</code></td>
      <td><code>多重集合（即 bag）的数据类型。与集合不同，它允许具有共同子类型的每个元素出现多个实例。每个唯一值（包括 NULL）都被映射到某个重数（multiplicity）。</code><br><br>
          <code>对元素类型没有限制；确保唯一性由用户负责。</code><br><br>
          <code>该类型可使用 MULTISET&lt;t&gt; 声明，其中 t 为所包含元素的数据类型。</code>
      </td>
    </tr>
    <tr>
      <td><code>ROW&lt;n0 t0, n1 t1, ...&gt;</code><br>
          <code>ROW&lt;n0 t0 'd0', n1 t1 'd1', ...&gt;</code>
      </td>
      <td><code>由一系列字段组成的数据类型。</code><br><br>
          <code>一个字段由字段名、字段类型和可选的描述组成。表的一行最具体的类型即是行类型（row type）。在这种情况下，行中的每一列对应于行类型中与该列具有相同序号位置的字段。</code><br><br>
          <code>与 SQL 标准相比，可选的字段描述简化了对复杂结构的处理。</code><br><br>
          <code>行类型类似于其他不符合标准的框架中的 STRUCT 类型。</code><br><br>
          <code>该类型可使用 ROW&lt;n0 t0 'd0', n1 t1 'd1', ...&gt; 声明，其中 n 为字段的唯一名称，t 为字段的逻辑类型，d 为字段的描述。</code>
      </td>
    </tr>
    <tr>
      <td><code>VARIANT</code></td>
      <td><code>半结构化数据的数据类型。</code><br><br>
          <code>专为存储任意半结构化数据而设计，包括 ARRAY、MAP 和标量类型。VARIANT 只能存储键类型为 STRING 的 MAP 类型。</code><br><br>
          <code>注意：需要 Flink 2.0+ 和 Spark 4.0+。</code>
      </td>
    </tr>
    <tr>
      <td><code>BLOB</code></td>
      <td><code>二进制大对象的数据类型。</code><br><br>
          <code>专为存储大型二进制数据而设计，例如图像、视频、音频文件以及其他多模态数据。与内联存储数据的 BYTES 类型不同，BLOB 将大型二进制数据存储在独立的文件中并维护对它们的引用，从而为大对象提供更好的性能。</code><br><br>
          <code>注意：需要将 'row-tracking.enabled' 和 'data-evolution.enabled' 设置为 true。详见 <a href="../append-table/blob">Blob 类型</a>。</code>
      </td>
    </tr>
    </tbody>
</table>
