# 从实体中移除组件

要从实体中移除组件，可以使用该实体所在世界的 `EntityManager`。

**重要提示**：从实体中移除组件是一个结构性更改，这意味着实体会移动到不同的原型块（archetype chunk）。

#### 从主线程移除组件

您可以直接从主线程中移除实体的组件。以下代码示例获取所有附加有 `Rotation` 组件的实体，然后移除它们的 `Rotation` 组件：

```csharp
// 定义 Rotation 组件
struct Rotation : IComponentData {}

public partial struct RemoveComponentSystemExample : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        // 创建一个查询，获取所有具有 Rotation 组件的实体
        var query = state.GetEntityQuery(typeof(Rotation));
        
        // 移除查询结果中的所有实体的 Rotation 组件
        state.EntityManager.RemoveComponent<Rotation>(query);
    }
}
```

### 从作业中移除组件

由于从实体中移除组件是一个结构性更改，因此不能直接在作业中进行。相反，必须使用 `EntityCommandBuffer` 来记录稍后移除组件的意图。

#### 使用 EntityCommandBuffer 从作业中移除组件

以下示例展示了如何在作业中记录移除组件的意图，并在作业完成后实际执行此操作：

```csharp
public partial struct RemoveComponentInJobSystemExample : ISystem
{
    private EntityCommandBufferSystem ecbSystem;

    public void OnCreate(ref SystemState state)
    {
        // 获取 EntityCommandBufferSystem
        ecbSystem = state.World.GetOrCreateSystemManaged<EndSimulationEntityCommandBufferSystem>();
    }

    public void OnUpdate(ref SystemState state)
    {
        var ecb = ecbSystem.CreateCommandBuffer().AsParallelWriter();

        Entities.ForEach((Entity entity, int entityInQueryIndex, in Rotation rotation) =>
        {
            // 在作业中记录移除组件的意图
            ecb.RemoveComponent<Rotation>(entityInQueryIndex, entity);
        }).ScheduleParallel();
        
        // 确保命令缓冲区在结束时播放
        ecbSystem.AddJobHandleForProducer(state.Dependency);
    }
}
```
