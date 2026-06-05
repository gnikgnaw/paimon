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

# 概述 {#overview}

PyPaimon 是用于连接 Paimon Catalog、读写表的 Python 实现。全新 PyPaimon 的完整 Python 实现无需安装 JDK。

## 环境配置 {#environment-settings}

SDK 已发布于 [pypaimon](https://pypi.org/project/pypaimon/)。你可以通过以下命令安装：

```shell
pip install pypaimon
```

## 从源码构建 {#build-from-source}

你可以通过执行以下命令构建源码包：

```commandline
python3 setup.py sdist
```

该包位于 `dist/` 目录下。然后你可以通过执行以下命令安装该包：

```commandline
pip3 install dist/*.tar.gz
```

该命令会将该包及核心依赖安装到你的本地 Python 环境中。
