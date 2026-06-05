---
title: "Amoro"
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

# Apache Amoro 与 Paimon {#apache-amoro-with-paimon}

**[Apache Amoro（孵化中）](https://amoro.apache.org)** 是一个构建在开放数据湖格式之上的湖仓管理系统。Amoro 与 Flink、Spark、Trino 等计算引擎协同工作，为湖仓提供可插拔的
**[表维护（Table Maintenance）](https://amoro.apache.org/docs/latest/self-optimizing/)** 能力，从而带来开箱即用的数据仓库体验，并帮助数据平台或产品轻松构建基础设施解耦、流批融合且湖原生的架构。
**[AMS](https://amoro.apache.org/docs/latest/#architecture)（Amoro Management Service，Amoro 管理服务）** 提供湖仓管理能力，例如自优化（self-optimizing）、数据过期等。它还为所有计算引擎提供统一的 Catalog 服务，并且可以与现有的 metastore 服务（如 HMS，Hive Metastore）结合使用。


# 表格式 {#table-format}

Apache Amoro 支持 Paimon 所支持的所有 Catalog 类型，包括常见的 Catalog：Hadoop、Hive、Glue、JDBC、Nessie 以及其他第三方 Catalog。
Amoro 支持 Paimon 所支持的所有存储类型，包括常见的存储：Hadoop、S3、GCS、ECS、OSS 等等。

![](/img/amoro-paimon.png)

未来还将支持 Paimon 的自动优化策略，用户可以通过与 Amoro 自动优化相配合，获得最佳的平衡体验
