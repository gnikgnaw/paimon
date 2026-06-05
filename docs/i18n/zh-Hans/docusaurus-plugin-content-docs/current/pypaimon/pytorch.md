---
title: "PyTorch"
sidebar_position: 4
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

# PyTorch {#pytorch}

## 读 {#read}

这需要先安装 `torch`。

你可以把所有数据读入 `torch.utils.data.Dataset` 或 `torch.utils.data.IterableDataset`：

```python
from torch.utils.data import DataLoader

table_read = read_builder.new_read()
dataset = table_read.to_torch(splits, streaming=True, prefetch_concurrency=2)
dataloader = DataLoader(
    dataset,
    batch_size=2,
    num_workers=2,  # Concurrency to read data
    shuffle=False
)

# Collect all data from dataloader
for batch_idx, batch_data in enumerate(dataloader):
    print(batch_data)

# output:
#   {'user_id': tensor([1, 2]), 'behavior': ['a', 'b']}
#   {'user_id': tensor([3, 4]), 'behavior': ['c', 'd']}
#   {'user_id': tensor([5, 6]), 'behavior': ['e', 'f']}
#   {'user_id': tensor([7, 8]), 'behavior': ['g', 'h']}
```

当 `streaming` 参数为 true 时，将以迭代方式读取；
当为 false 时，则会把全量数据读入内存。

**`prefetch_concurrency`**（默认值：1）：当 streaming 为 true 时，用于在每个 DataLoader worker 内部进行并行预取的线程数。将其设为大于 1 的值，可在多个线程间划分 split 并提升读取吞吐。当 streaming 为 false 时该参数无效。
