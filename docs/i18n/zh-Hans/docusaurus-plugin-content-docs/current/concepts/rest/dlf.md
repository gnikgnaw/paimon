---
title: "DLF Token"
sidebar_position: 3
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

# DLF Token {#dlf-token}

DLF（Data Lake Formation，数据湖构建）是一个用于统一元数据与数据存储管理的全托管平台，旨在为客户提供元数据管理、存储管理、权限管理、存储分析以及存储优化等功能。

DLF 针对不同环境提供了多种鉴权方式。

:::info

`'warehouse'` 是你在服务端的 Catalog 实例名称，而不是路径。

:::

## 使用访问密钥（Access Key） {#use-the-access-key}

```sql
CREATE CATALOG `paimon-rest-catalog`
WITH (
    'type' = 'paimon',
    'uri' = '<catalog server url>',
    'metastore' = 'rest',
    'warehouse' = 'my_instance_name',
    'token.provider' = 'dlf',
    'dlf.access-key-id'='<access-key-id>',
    'dlf.access-key-secret'='<access-key-secret>',
);
```

- `uri`：访问 DLF Rest Catalog Server 的 URI。
- `warehouse`：DLF Catalog 名称
- `token.provider`：token provider（token 提供方）
- `dlf.access-key-id`：访问 DLF 服务所需的 Access Key ID，通常指你的 RAM 用户的 AccessKey
- `dlf.access-key-secret`：访问 DLF 服务所需的 Access Key Secret

你可以为某个 RAM 用户授予特定权限，并使用该 RAM 用户的访问密钥长期访问你的 DLF 资源。与使用阿里云账号访问密钥相比，使用 RAM 用户访问密钥访问 DLF 资源更为安全。

## 使用 STS 临时访问令牌 {#use-the-sts-temporary-access-token}

通过 STS 服务，你可以为用户生成临时访问令牌，使其能够在有效期内访问受策略限制的 DLF 资源。

```sql
CREATE CATALOG `paimon-rest-catalog`
WITH (
    'type' = 'paimon',
    'uri' = '<catalog server url>',
    'metastore' = 'rest',
    'warehouse' = 'my_instance_name',
    'token.provider' = 'dlf',
    'dlf.access-key-id'='<access-key-id>',
    'dlf.access-key-secret'='<access-key-secret>',
    'dlf.security-token'='<security-token>'
);
```

在某些环境中，临时访问令牌可以通过本地文件定期刷新：

```sql
CREATE CATALOG `paimon-rest-catalog`
WITH (
    'type' = 'paimon',
    'uri' = '<catalog server url>',
    'metastore' = 'rest',
    'warehouse' = 'my_instance_name',
    'token.provider' = 'dlf',
    'dlf.token-path' = 'my_token_path_in_disk'
);
```

## 使用来自阿里云 ECS 角色的 STS Token {#use-the-sts-token-from-aliyun-ecs-role}

实例 RAM 角色是指授予某个 ECS 实例的 RAM 角色。该 RAM 角色是一种标准的服务角色，其受信实体为云服务器。通过使用实例 RAM 角色，无需配置 AccessKey 即可在 ECS 实例内部获取临时访问令牌（STS Token）。

```sql
CREATE CATALOG `paimon-rest-catalog`
WITH (
    'type' = 'paimon',
    'uri' = '<catalog server url>',
    'metastore' = 'rest',
    'warehouse' = 'my_instance_name',
    'token.provider' = 'dlf',
    'dlf.token-loader' = 'ecs'
    -- optional, loader can obtain it through ecs metadata service
    -- 'dlf.token-ecs-role-name' = 'my_ecs_role_name'
);
```

## DLF Endpoint 配置 {#dlf-endpoint-configuration}

Paimon 支持两种类型的 DLF endpoint，并会自动选择合适的签名算法：

- **DLF VPC endpoints**（例如 `cn-hangzhou-vpc.dlf.aliyuncs.com`）：推荐在 VPC 环境中使用，具有更优的性能和更低的延迟。
- **DLF OpenAPI endpoints**（例如 `dlfnext.cn-hangzhou.aliyuncs.com`）：通过阿里云 API 基础设施支持公网访问。
  **注意：** 目前 OpenAPI Endpoints 仅支持由字母数字字符（A-Z、a-z、0-9）和特定符号组成的数据库名与表名。

只需配置 endpoint URI，Paimon 会自动处理鉴权：

```sql
CREATE CATALOG `paimon-rest-catalog`
WITH (
    'type' = 'paimon',
    'uri' = 'https://${region}-vpc.dlf.aliyuncs.com',  -- or OpenAPI endpoint: https://dlfnext.cn-hangzhou.aliyuncs.com
    'metastore' = 'rest',
    'warehouse' = 'my_instance_name',
    'token.provider' = 'dlf',
    'dlf.access-key-id'='<access-key-id>',
    'dlf.access-key-secret'='<access-key-secret>'
);
```
