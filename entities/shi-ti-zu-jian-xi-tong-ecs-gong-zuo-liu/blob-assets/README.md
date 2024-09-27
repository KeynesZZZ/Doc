# Blob Assets

##

本节包含有关 `Blob Assets` 的信息，包括什么是 `Blob Assets`、如何创建它们以及如何在烘焙过程中使用它们。

### 目录

| 主题                   | 描述                  |
| -------------------- | ------------------- |
| Blob assets concepts | 了解什么是 `Blob Assets` |
| Create a blob asset  | 创建一个 `Blob Asset`   |

### Blob Assets Concepts

`Blob Assets` 是一种高效存储和访问只读数据的机制。在 Unity 的 `Entity Component System (ECS)` 中，它们用于存储大量静态数据，例如配置表或其他不会在运行时发生变化的数据。`Blob Assets` 使用连续的内存块，提供了非常快速的读取性能，并且减少了内存碎片。

#### 优点

* **只读**: 确保数据在运行时不会被修改。
* **性能优化**: 连续的内存分配使得缓存命中率更高，读取速度更快。
* **内存效率**: 降低内存碎片，提高内存利用率。

### Create a Blob Asset

创建 `Blob Asset` 需要几个步骤：

1. **定义 Blob 数据结构**：
   * 首先，你需要定义将要存储在 `Blob Asset` 中的数据结构。
2. **创建 Builder**：
   * 使用 `BlobBuilder` 类来构建 `Blob Asset`。
3. **分配内存并完成构建**：
   * 将构建的数据复制到连续的内存块中，形成最终的 `Blob Asset`。

#### 示例代码：创建 Blob Asset

以下是如何创建一个简单的 `Blob Asset` 的示例：

**步骤 1: 定义 Blob 数据结构**

```csharp
using Unity.Entities;
using Unity.Collections;
using Unity.Mathematics;

public struct MyBlobData
{
    public float3 Position;
    public float3 Scale;
}
```
