# Allocators 概述

#### Allocators 概述

Entities 和 Collections 包有不同的 allocators，用于管理内存分配。不同的 allocators 以不同方式组织和跟踪其内存。以下是可用的 allocators：

* **Allocator.Temp**：用于短期分配的快速 allocator，每个线程上创建。
* **Allocator.TempJob**：必须在创建后的4帧内释放的短期 allocator。
* **Allocator.Persistent**：用于无限生命周期分配的最慢 allocator。
* **Rewindable allocator**：一种快速且线程安全的自定义 allocator，可以一次性回退并释放所有分配。
* **World update allocator**：世界拥有的双重可回退 allocator，快速且线程安全。
* **Entity command buffer allocator**：实体命令缓冲系统拥有并用于创建命令缓冲的可回退 allocator。
* **System group allocator**：设置其速率管理器时由组件系统组创建的可选双重可回退 allocator。用于在与世界更新不同速率的系统组中进行的分配。

#### Allocator 功能比较

不同的 allocators 具有以下不同功能：

| Allocator 类型                    | 自定义 Allocator | 使用前需要创建 | 生命周期        | 自动释放分配 | 可以将分配传递到作业 |
| ------------------------------- | ------------- | ------- | ----------- | ------ | ---------- |
| Allocator.Temp                  | 否             | 否       | 一帧或一个作业     | 是      | 否          |
| Allocator.TempJob               | 否             | 否       | 创建后4帧内      | 否      | 是          |
| Allocator.Persistent            | 否             | 否       | 无限          | 否      | 是          |
| Rewindable allocator            | 是             | 是       | 无限          | 否      | 是          |
| World update allocator          | 是 - 双重可回退     | 否       | 每2帧         | 是      | 是          |
| Entity command buffer allocator | 是 - 可回退       | 否       | 与实体命令缓冲相同   | 是      | 是          |
| System group allocator          | 是 - 双重可回退     | 是       | 2次固定速率系统组更新 | 是      | 是          |
