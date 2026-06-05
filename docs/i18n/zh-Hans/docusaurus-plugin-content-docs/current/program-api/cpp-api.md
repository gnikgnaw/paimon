---
title: "Cpp API"
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

# Cpp API {#cpp-api}

Paimon C++ 是 Apache Paimon 的高性能 C++ 实现。Paimon C++ 旨在提供一个原生、高性能且可扩展的实现，使原生引擎能够以最高的效率访问 Paimon 数据湖格式。

## 环境配置 {#environment-settings}

[Paimon C++](https://github.com/alibaba/paimon-cpp.git) 目前由阿里巴巴开源社区管理。你可以查阅[文档](https://alibaba.github.io/paimon-cpp/getting_started.html)以获取有关环境配置的更多细节。

```sh
git clone https://github.com/alibaba/paimon-cpp.git
cd paimon-cpp
mkdir build-release
cd build-release
cmake ..
make -j8       # if you have 8 CPU cores, otherwise adjust
make install
```

## 创建 Catalog {#create-catalog}

在接触 Table 之前，你需要先创建一个 Catalog。

```c++
#include "paimon/catalog/catalog.h"

// Note that keys and values are all string
std::map<std::string, std::string> options;
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::Catalog> catalog,
                       paimon::Catalog::Create(root_path, options));
```

当前 C++ Paimon 仅支持文件系统 Catalog。未来我们将支持 REST Catalog。参见 [Catalog](../concepts/catalog)。

你可以使用该 Catalog 创建用于写入数据的表。

## 创建数据库 {#create-database}

表位于某个数据库中。如果你想在一个新数据库中创建表，应当先创建该数据库。

```c++
PAIMON_RETURN_NOT_OK(catalog->CreateDatabase('database_name', options, /*ignore_if_exists=*/false));
```

## 创建表 {#create-table}

表的 schema（表结构）包含字段定义、分区键、主键以及表的配置项。字段定义由 `Arrow::Schema` 描述。除字段定义外的所有参数均为可选项。

例如：

```c++
arrow::FieldVector fields = {
    arrow::field("f0", arrow::utf8()),
    arrow::field("f1", arrow::int32()),
    arrow::field("f2", arrow::int32()),
    arrow::field("f3", arrow::float64()),
};
std::shared_ptr<arrow::Schema> schema = arrow::schema(fields);
::ArrowSchema arrow_schema;
arrow::Status arrow_status = arrow::ExportSchema(*schema, &arrow_schema);
if (!arrow_status.ok()) {
    return paimon::Status::Invalid(arrow_status.message());
}
PAIMON_RETURN_NOT_OK(catalog->CreateTable(paimon::Identifier(db_name, table_name),
                                            &arrow_schema,
                                            /*partition_keys=*/{},
                                            /*primary_keys=*/{}, options,
                                            /*ignore_if_exists=*/false));
```

有关所有受支持的 `arrow-to-paimon` 数据类型映射，参见 [Data Types](https://alibaba.github.io/paimon-cpp/user_guide/data_types.html)。

## 批式写入 {#batch-write}

Paimon 表写入采用两阶段提交（Two-Phase Commit），你可以多次写入，但一旦提交，便不能再写入更多数据。C++ Paimon 使用 Apache Arrow 作为[内存格式]，更多细节请查阅[文档](https://alibaba.github.io/paimon-cpp/user_guide/arrow.html)。

例如：
```c++
arrow::Result<std::shared_ptr<arrow::StructArray>> PrepareData(const arrow::FieldVector& fields) {
    arrow::StringBuilder f0_builder;
    arrow::Int32Builder f1_builder;
    arrow::Int32Builder f2_builder;
    arrow::DoubleBuilder f3_builder;

    std::vector<std::tuple<std::string, int, int, double>> data = {
        {"Alice", 1, 0, 11.0}, {"Bob", 1, 1, 12.1}, {"Cathy", 1, 2, 13.2}};

    for (const auto& row : data) {
        ARROW_RETURN_NOT_OK(f0_builder.Append(std::get<0>(row)));
        ARROW_RETURN_NOT_OK(f1_builder.Append(std::get<1>(row)));
        ARROW_RETURN_NOT_OK(f2_builder.Append(std::get<2>(row)));
        ARROW_RETURN_NOT_OK(f3_builder.Append(std::get<3>(row)));
    }

    std::shared_ptr<arrow::Array> f0_array, f1_array, f2_array, f3_array;
    ARROW_RETURN_NOT_OK(f0_builder.Finish(&f0_array));
    ARROW_RETURN_NOT_OK(f1_builder.Finish(&f1_array));
    ARROW_RETURN_NOT_OK(f2_builder.Finish(&f2_array));
    ARROW_RETURN_NOT_OK(f3_builder.Finish(&f3_array));

    std::vector<std::shared_ptr<arrow::Array>> children = {f0_array, f1_array, f2_array, f3_array};
    auto struct_type = arrow::struct_(fields);
    return std::make_shared<arrow::StructArray>(struct_type, f0_array->length(), children);
}
```

```c++
std::string table_path = root_path + "/" + db_name + ".db/" + table_name;
std::string commit_user = "some_commit_user";
// write
paimon::WriteContextBuilder context_builder(table_path, commit_user);
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::WriteContext> write_context,
                        context_builder.SetOptions(options).Finish());
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::FileStoreWrite> writer,
                        paimon::FileStoreWrite::Create(std::move(write_context)));
// prepare data
auto struct_array = PrepareData(fields);
if (!struct_array.ok()) {
    return paimon::Status::Invalid(struct_array.status().ToString());
}
::ArrowArray arrow_array;
arrow_status = arrow::ExportArray(*struct_array.ValueUnsafe(), &arrow_array);
if (!arrow_status.ok()) {
    return paimon::Status::Invalid(arrow_status.message());
}
paimon::RecordBatchBuilder batch_builder(&arrow_array);
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::RecordBatch> record_batch,
                        batch_builder.Finish());
PAIMON_RETURN_NOT_OK(writer->Write(std::move(record_batch)));
PAIMON_ASSIGN_OR_RAISE(std::vector<std::shared_ptr<paimon::CommitMessage>> commit_message,
                        writer->PrepareCommit());

// commit
paimon::CommitContextBuilder commit_context_builder(table_path, commit_user);
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::CommitContext> commit_context,
                        commit_context_builder.SetOptions(options).Finish());
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::FileStoreCommit> committer,
                        paimon::FileStoreCommit::Create(std::move(commit_context)));
PAIMON_RETURN_NOT_OK(committer->Commit(commit_message));
```

## 批式读取 {#batch-read}

### 谓词下推 {#predicate-pushdown}

`ReadContextBuilder` 用于向 reader 传递上下文，下推与过滤由 reader 完成。

```c++
ReadContextBuilder read_context_builder(table_path);
```

你可以使用 `PredicateBuilder` 构建过滤条件，并通过 `ReadContextBuilder` 将其下推：

```c++
# Example filter: 'f3' > 12.0 OR 'f1' == 1
PAIMON_ASSIGN_OR_RAISE(
    auto predicate,
    PredicateBuilder::Or(
        {PredicateBuilder::GreaterThan(/*field_index=*/3, /*field_name=*/"f3",
                                        FieldType::DOUBLE, Literal(static_cast<double>(12.0))),
        PredicateBuilder::Equal(/*field_index=*/1, /*field_name=*/"f1", FieldType::INT,
                                    Literal(1))}));
ReadContextBuilder read_context_builder(table_path);
read_context_builder.SetPredicate(predicate).EnablePredicateFilter(true);
```

你也可以通过 `ReadContextBuilder` 下推投影：

```c++
# select f3 and f2 columns
read_context_builder.SetReadSchema({"f3", "f1", "f2"});
```

### 生成 Splits {#generate-splits}

随后你可以进入扫描计划（Scan Plan）阶段以获取 `splits`：

```c++
// scan
paimon::ScanContextBuilder scan_context_builder(table_path);
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::ScanContext> scan_context,
                        scan_context_builder.SetOptions(options).Finish());
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::TableScan> scanner,
                        paimon::TableScan::Create(std::move(scan_context)));
PAIMON_ASSIGN_OR_RAISE(std::shared_ptr<paimon::Plan> plan, scanner->CreatePlan());
auto splits = plan->Splits();
```

最后，你可以从这些 `splits` 中将数据读取为 arrow 格式。

### 读取 Apache Arrow {#read-apache-arrow}

这需要已安装 `C++ Arrow`。

```c++
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::ReadContext> read_context,
                        read_context_builder.SetOptions(options).Finish());
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::TableRead> table_read,
                        paimon::TableRead::Create(std::move(read_context)));
PAIMON_ASSIGN_OR_RAISE(std::unique_ptr<paimon::BatchReader> batch_reader,
                        table_read->CreateReader(splits));
arrow::ArrayVector result_array_vector;
while (true) {
    PAIMON_ASSIGN_OR_RAISE(paimon::BatchReader::ReadBatch batch, batch_reader->NextBatch());
    if (paimon::BatchReader::IsEofBatch(batch)) {
        break;
    }
    auto& [c_array, c_schema] = batch;
    auto arrow_result = arrow::ImportArray(c_array.get(), c_schema.get());
    if (!arrow_result.ok()) {
        return paimon::Status::Invalid(arrow_result.status().ToString());
    }
    auto result_array = arrow_result.ValueUnsafe();
    result_array_vector.push_back(result_array);
}
auto chunk_result = arrow::ChunkedArray::Make(result_array_vector);
if (!chunk_result.ok()) {
    return paimon::Status::Invalid(chunk_result.status().ToString());
}
```

## 文档 {#documentation}

更多信息，参见 [C++ Paimon 文档](https://alibaba.github.io/paimon-cpp/index.html)。
