# 向实体添加组件

要向实体添加组件，可以使用该实体所在世界的 `EntityManager`。您可以向单个实体或多个实体同时添加组件。

向实体添加组件是一个结构性更改，这意味着实体会移动到不同的块中。因此，不能直接从作业中向实体添加组件。相反，必须使用 `EntityCommandBuffer` 来记录添加组件的意图，以便稍后执行。

#### 向单个实体添加组件

以下代码示例展示了如何创建一个新实体，然后从主线程向该实体添加组件：

```csharp
public partial struct AddComponentToSingleEntitySystemExample : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        // 创建一个新实体
        var entity = state.EntityManager.CreateEntity();
        
        // 向该实体添加 Rotation 组件
        state.EntityManager.AddComponent<Rotation>(entity);
    }
}
```

### 向多个实体添加组件

以下代码示例展示了如何获取所有附加有 `ComponentA` 组件的实体，并从主线程向它们添加 `ComponentB` 组件：

```csharp
// 定义 ComponentA
struct ComponentA : IComponentData {}

// 定义 ComponentB
struct ComponentB : IComponentData {}

public partial struct AddComponentToMultipleEntitiesSystemExample : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        // 创建一个查询，获取所有具有 ComponentA 的实体
        var query = state.GetEntityQuery(typeof(ComponentA));
        
        // 向查询结果中的所有实体添加 ComponentB
        state.EntityManager.AddComponent<ComponentB>(query);
    }
}
```
