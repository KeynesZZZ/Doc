# Chunk 组件 (Chunk components)

Chunk 组件是一种在每个 chunk 中存储值而不是每个实体中存储值的组件类型。它们提供与共享组件类似的功能，但在一些基本方面有所不同。

#### 主题描述

| 主题          | 描述                          |
| ----------- | --------------------------- |
| 引入 Chunk 组件 | 了解 Chunk 组件及其使用场景。          |
| 创建 Chunk 组件 | 创建一个新的 Chunk 组件以在您的应用程序中使用。 |
| 使用 Chunk 组件 | 了解如何使用 Chunk 组件特定的 API。     |

#### 引入 Chunk 组件

Chunk 组件允许您在每个 chunk 而不是每个实体上存储数据，这对于存储在同一 chunk 中的所有实体具有相同值的数据非常有效。

#### 创建 Chunk 组件

要创建一个 Chunk 组件，首先定义一个实现 `IComponentData` 接口的结构：

```csharp
public struct MyChunkComponent : IComponentData
{
    public int Value;
}
```
