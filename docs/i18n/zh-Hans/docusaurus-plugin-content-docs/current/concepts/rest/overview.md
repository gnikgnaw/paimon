---
title: "概述"
sidebar_position: 1
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

# RESTCatalog {#restcatalog}

## 概述 {#overview}

Paimon REST Catalog 提供了一个轻量级实现，用于访问 Catalog 服务。Paimon 可以通过一个实现了 REST API 的 Catalog 服务器来访问该
Catalog 服务。你可以在 [REST API](./rest-api) 中查看所有的 API。

![](/img/rest-catalog.svg)

## 核心特性 {#key-features}

1. 用户自定义的技术专属逻辑实现
    - 所有技术专属的逻辑都位于 Catalog 服务器内部。
    - 这确保了用户可以定义归用户所有的逻辑。
2. 解耦的架构
    - REST Catalog 通过定义良好的 REST API 与 Catalog 服务器进行交互。
    - 这种解耦使得 Catalog 服务器与客户端可以独立演进和扩展。
3. 语言无关
    - 开发者可以使用任意编程语言来实现 Catalog 服务器，只要它遵循指定的 REST API 即可。
    - 这种灵活性使得团队能够沿用其现有的技术栈和专长。
4. 支持任意 Catalog 后端
    - REST Catalog 被设计为可与任意 Catalog 后端协同工作。
    - 只要它们实现了相关的 API，就能与 REST Catalog 无缝集成。

## 结论 {#conclusion}

REST Catalog 为访问 Catalog 服务提供了一套灵活适配的方案。根据 [REST API](./rest-api)，它与 Catalog 服务实现了
解耦。

技术专属的逻辑被封装在 Catalog 服务器上。同时，Catalog 服务器支持任意
后端和语言。

## Token Provider {#token-provider}

RESTCatalog 支持多种访问认证方式，包括以下几种：

1. [Bear Token](./bear)。
2. [DLF Token](./dlf)。

## REST Open API {#rest-open-api}

参见 [REST API](./rest-api)。
