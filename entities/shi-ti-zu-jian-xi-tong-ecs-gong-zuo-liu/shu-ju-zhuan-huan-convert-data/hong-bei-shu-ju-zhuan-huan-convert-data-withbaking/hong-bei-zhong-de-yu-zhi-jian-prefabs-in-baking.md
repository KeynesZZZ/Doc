# 烘焙中的预制件(Prefabs in baking )

在烘焙过程中，预制件会被烘焙成实体预制件。一个实体预制件是一个具有以下组件的实体：

* **预制标签**：标识该实体为预制件，并默认将其排除在查询之外。
* **`LinkedEntityGroup` 缓冲区**：以平面列表形式存储预制件中的所有子项。例如，可以快速创建预制件中的整套实体，而无需遍历层次结构。

可以像使用 GameObject 预制件一样使用实体预制件，因为它们可以在运行时实例化。然而，要在运行时使用它们，你必须烘焙 GameObject 预制件并使其在实体场景中可用。

### 注意点

当预制件实例存在于子场景层次结构中时，烘焙会将它们视为普通的 GameObjects，因为它们没有 `Prefab` 或 `LinkedEntityGroup` 组件。

### 创建和注册实体预制件

为了确保预制件被烘焙并在实体场景中可用，你必须将它们注册到一个 baker。这确保了对预制件对象的依赖，并且预制件被烘焙并接收到适当的组件。当你在组件中引用实体预制件时，Unity 会将内容序列化到使用它的子场景中。

#### 示例代码

以下代码展示了如何创建和注册一个实体预制件：

**`EntityPrefabComponent` 结构体**

```csharp
public struct EntityPrefabComponent : IComponentData
{
    public Entity Value;
}

public class GetPrefabAuthoring : MonoBehaviour
{
    public GameObject Prefab;
}
public class GetPrefabBaker : Baker<GetPrefabAuthoring>
{
    public override void Bake(GetPrefabAuthoring authoring)
    {
        // 在 Baker 中注册预制件
        var entityPrefab = GetEntity(authoring.Prefab, TransformUsageFlags.Dynamic);
        // 添加实体引用到一个组件中，以便稍后实例化
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new EntityPrefabComponent() {Value = entityPrefab});
    }
}

```

## 替代方案：在烘焙期间引用实体预制件

在烘焙过程中，可以使用 `EntityPrefabReference` 结构体来引用实体预制件。这会将预制件的 ECS 内容序列化到一个单独的实体场景文件中，该文件可以在运行时加载，然后再使用预制件。这可以防止 Unity 在每个使用它的子场景中重复预制件。

### 示例代码

以下代码展示了如何使用 `EntityPrefabReference` 来引用实体预制件：

#### `EntityPrefabReferenceComponent` 结构体

```csharp
public struct EntityPrefabReferenceComponent : IComponentData
{
    public EntityPrefabReference Value;
}

public class GetPrefabReferenceAuthoring : MonoBehaviour
{
    public GameObject Prefab;
}

public class GetPrefabReferenceBaker : Baker<GetPrefabReferenceAuthoring>
{
    public override void Bake(GetPrefabReferenceAuthoring authoring)
    {
        // 从 GameObject 创建一个 EntityPrefabReference。这将允许
        // 序列化过程将预制件序列化到其自己的实体场景文件中，而不是
        // 在每个使用它的地方都重复预制件 ECS 内容。
        var entityPrefab = new EntityPrefabReference(authoring.Prefab);
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new EntityPrefabReferenceComponent() {Value = entityPrefab});
    }
}

```

## 实例化预制件

要实例化在组件中引用的预制件，可以使用 `EntityManager` 或实体命令缓冲区 (`EntityCommandBuffer`)。

### 示例代码

以下代码展示了如何在系统中实例化预制件：

#### `InstantiatePrefabSystem` 结构体

```csharp
public partial struct InstantiatePrefabSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // 获取所有具有实体引用组件的实体
        foreach (var prefab in 
                 SystemAPI.Query<RefRO<EntityPrefabComponent>>())
        {
            // 实例化预制件实体
            var instance = ecb.Instantiate(prefab.ValueRO.Value);
            
            // 注意：返回的实例仅在 ECB 中相关
            // 因为在调用 ECB.Playback 之前，实体不会被创建到 EntityManager 中
            ecb.AddComponent<ComponentA>(instance);
        }

        // 回放命令缓冲区，将创建的实体写入 EntityManager
        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```

注意：场景部分组件：实例化的预制件将包含一个 SceneSection 组件。这可能会影响实体的生命周期。 这种机制确保在运行时正确实例化和初始化预制件，同时利用 EntityCommandBuffer 来保证变更的一致性和效率。

## 实例化引用了 `EntityPrefabReference` 的预制件

要实例化通过 `EntityPrefabReference` 引用的预制件，必须将 `RequestEntityPrefabLoaded` 结构体添加到实体中。这是因为 Unity 需要在使用预制件之前加载它。`RequestEntityPrefabLoaded` 确保预制件被加载，并且结果添加到 `PrefabLoadResult` 组件中。Unity 将 `PrefabLoadResult` 组件添加到包含 `RequestEntityPrefabLoaded` 的同一实体。

### 示例代码

以下代码展示了如何在系统中处理和实例化引用了 `EntityPrefabReference` 的预制件：

#### `InstantiatePrefabReferenceSystem` 结构体

```csharp
public partial struct InstantiatePrefabReferenceSystem : ISystem
{
    public void OnStartRunning(ref SystemState state)
    {
        // 向具有 EntityPrefabReference 但尚未具有 PrefabLoadResult 的实体添加
        // RequestEntityPrefabLoaded 组件（PrefabLoadResult 在预制件加载时添加）
        // 注意：加载预制件可能需要几个帧
        var query = SystemAPI.QueryBuilder()
            .WithAll<EntityPrefabReferenceComponent>()
            .WithNone<PrefabLoadResult>().Build();
        state.EntityManager.AddComponent<RequestEntityPrefabLoaded>(query);
    }

    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // 对于具有 PrefabLoadResult 组件的实体（Unity 已经加载了预制件），
        // 从 PrefabLoadResult 获取已加载的预制件并实例化它
        foreach (var (prefab, entity) in 
                 SystemAPI.Query<RefRO<PrefabLoadResult>>().WithEntityAccess())
        {
            var instance = ecb.Instantiate(prefab.ValueRO.PrefabRoot);

            // 删除 RequestEntityPrefabLoaded 和 PrefabLoadResult，以防止
            // 预制件多次加载和实例化
            ecb.RemoveComponent<RequestEntityPrefabLoaded>(entity);
            ecb.RemoveComponent<PrefabLoadResult>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```

## 查询中的预制件

默认情况下，Unity 将预制件排除在查询之外。要在查询中包含实体预制件，可以使用查询中的 `IncludePrefab` 字段。

### 示例代码

以下代码展示了如何在查询中包含预制件：

#### 包含预制件的查询

```csharp
// 此查询将返回所有烘焙的实体，包括预制件实体
var prefabQuery = SystemAPI.QueryBuilder()
    .WithAll<BakedEntity>()
    .WithOptions(EntityQueryOptions.IncludePrefab)
    .Build();
```

## 销毁预制件实例

要销毁预制件实例，可以使用 `EntityManager` 或实体命令缓冲区 (`EntityCommandBuffer`)，与销毁普通实体的方式相同。需要注意的是，销毁预制件会带来结构更改的开销。

* 使用 EntityCommandBuffer：通过 EntityCommandBuffer 可以批量记录和执行销毁操作，以提高性能。&#x20;
* 结构更改成本：销毁预制件会带来结构更改的开销，因此应谨慎处理频繁的销毁操作。

### 示例代码

以下代码展示了如何在系统中销毁预制件实例：

#### 销毁预制件实例的系统

```csharp
public partial struct DestroyPrefabInstanceSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        foreach (var (component, entity) in 
                 SystemAPI.Query<RefRO<RotationSpeed>>().WithEntityAccess())
        {
            // 如果 RotationSpeed 组件的 RadiansPerSecond 小于或等于 0，则销毁实体
            if (component.ValueRO.RadiansPerSecond <= 0)
            {
                ecb.DestroyEntity(entity);
            }
        }

        // 回放命令缓冲区，将销毁命令应用到 EntityManager
        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```
