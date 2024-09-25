# 使用 Chunk 组件 (Use chunk components)

与其他组件类型相比，Chunk 组件使用一组不同的 API 来添加、移除、获取和设置它们。例如，要将 Chunk 组件添加到实体上，您需要使用 `EntityManager.AddChunkComponentData` 而不是常规的 `EntityManager.AddComponent`。

#### 示例代码

以下代码示例展示了如何添加、设置和获取一个 Chunk 组件。假设存在一个名为 `ExampleChunkComp` 的 Chunk 组件和一个名为 `ExampleComponent` 的非 Chunk 组件：

```csharp
private void ChunkComponentExample(Entity e)
{
    // 将 ExampleChunkComp 添加到传入实体的 chunk 中。
    EntityManager.AddChunkComponentData<ExampleChunkComp>(e);

    // 查找所有具有 ExampleComponent 和 ExampleChunkComponent 的 chunk。
    // 为了区分 chunk 组件和常规的 IComponentData，必须使用 ComponentType.ChunkComponent 指定 chunk 组件。
    EntityQuery query = GetEntityQuery(typeof(ExampleComponent), ComponentType.ChunkComponent<ExampleChunkComp>());
    NativeArray<ArchetypeChunk> chunks = query.ToArchetypeChunkArray(Allocator.Temp);

    // 设置第一个 chunk 的 ExampleChunkComp 值。
    EntityManager.SetChunkComponentData<ExampleChunkComp>(chunks[0], new ExampleChunkComp { Value = 6 });

    // 获取第一个 chunk 的 ExampleChunkComp 值。
    ExampleChunkComp exampleChunkComp = EntityManager.GetChunkComponentData<ExampleChunkComp>(chunks[0]);
    Debug.Log(exampleChunkComp.Value);    // 6

    // 释放分配的内存
    chunks.Dispose();
}
```

注意

如果您只想从一个 Chunk 组件读取而不写入，请在定义查询时使用 `ComponentType.ChunkComponentReadOnly`。将查询中包含的组件标记为只读有助于避免不必要的作业调度约束。

尽管 Chunk 组件属于 chunk 本身，但在实体上添加或移除 Chunk 组件会改变其原型（archetype）并导致结构性更改。

注意

Unity 会将新创建的 Chunk 组件值初始化为这些类型的默认值。

您还可以通过任何一个 chunk 中的实体来获取和设置该 chunk 的 Chunk 组件：

#### 示例代码

```csharp
private void ChunkComponentExample(Entity e)
{
    // 设置实体的 chunk 的 ExampleChunkComp 值。
    EntityManager.SetChunkComponentData<MyChunkComp>(e, new MyChunkComp { Value = 6 });

    // 获取实体的 chunk 的 ExampleChunkComp 值。
    MyChunkComp myChunkComp = EntityManager.GetChunkComponentData<MyChunkComp>(e);
    Debug.Log(myChunkComp.Value);    // 6
}
```

### 在作业中使用 Chunk 组件 (Use chunk components in jobs)

作业不能使用 `EntityManager`，因此要访问 Chunk 组件，您需要使用它的 `ComponentTypeHandle`。

#### 示例代码

以下代码示例展示了如何在作业 (`IJobChunk`) 中获取和设置 Chunk 组件：

```csharp
struct MyJob : IJobChunk
{
    public ComponentTypeHandle<ExampleChunkComponent> ExampleChunkCompHandle;

    public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
    {
        // 获取 chunk 的 ExampleChunkComp。
        ExampleChunkComponent myChunkComp = chunk.GetChunkComponentData(ExampleChunkCompHandle);

        // 设置 chunk 的 ExampleChunkComp。
        chunk.SetChunkComponentData(ExampleChunkCompHandle, new ExampleChunkComponent { Value = 7 });
    }
}
```
