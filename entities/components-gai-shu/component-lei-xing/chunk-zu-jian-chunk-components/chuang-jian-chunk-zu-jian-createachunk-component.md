# 创建 Chunk 组件 (Create a chunk component)

Chunk 组件的定义与非托管组件相同。这意味着要创建一个 Chunk 组件，您需要创建一个继承自 `IComponentData` 的常规结构体。Chunk 组件和非托管组件之间的区别在于如何将它们添加到实体上。

#### 非托管组件示例

以下代码示例展示了一个非托管组件：

```csharp
public struct ExampleChunkComponent : IComponentData
{
    public int Value;
}
```

要将非托管组件用作 Chunk 组件，请使用 EntityManager.AddChunkComponentData(Entity) 将其添加到实体上。

```csharp
// 使用 EntityManager 添加 Chunk 组件
EntityManager.AddChunkComponentData<ExampleChunkComponent>(entity, new ExampleChunkComponent { Value = 10 });
```

通过这种方式，您可以创建并添加一个 Chunk 组件，以便在每个 chunk 上存储数据，而不是在每个实体上存储数据。
