# 使用 IJobEntity 迭代组件数据

当你有一个希望在多个系统中使用的数据转换，并且每个系统调用方式不同，可以使用 `IJobEntity` 迭代 `ComponentData`。它创建了一个 `IJobChunk` 作业，因此你只需考虑想要转换的数据。

请注意，无论是在 `SystemBase` 还是 `ISystem` 中，`IJobEntity` 的工作方式都是相同的。

## 创建一个 `IJobEntity` 作业

要创建一个 `IJobEntity` 作业，编写一个使用 `IJobEntity` 接口的结构体，并实现你自己的自定义 `Execute` 方法。

使用 `partial` 关键字，因为源生成会在项目的 `project/Temp/GeneratedCode/...` 文件夹中生成一个实现 `IJobChunk` 的结构体。

下面的示例每帧向每个 `SampleComponent` 添加一。

### 示例代码

```csharp
public struct SampleComponent : IComponentData 
{
    public float Value; 
}
public partial struct ASampleJob : IJobEntity
{
    // 向每个 SampleComponent 的值加一
    void Execute(ref SampleComponent sample)
    {
        sample.Value += 1f;
    }
}
public partial class ASample : SystemBase
{
    protected override void OnUpdate()
    {
        // 调度作业
        new ASampleJob().ScheduleParallel();
    }
}


```

## 指定查询

你可以通过以下方式为 `IJobEntity` 指定查询：

1. 手动创建查询，以指定不同的调用需求。
2. 使用 `IJobEntity` 属性，根据其给定的 `Execute` 参数和作业结构体中的说明来创建查询。

下面的示例展示了这两种选项：

### 示例代码

#### 定义组件

首先定义一个组件：

```csharp
public struct SampleComponent : IComponentData 
{
    public float Value; 
}
//实现一个部分结构体来定义你的作业逻辑：
public partial struct QueryJob : IJobEntity
{
    // 遍历所有 SampleComponents 并增加它们的值
    public void Execute(ref SampleComponent sample)
    {
        sample.Value += 1;
    }
}
//在 SystemBase 系统中创建和调度这个作业，并指定查询：
[RequireMatchingQueriesForUpdate]
public partial class QuerySystem : SystemBase
{
    // 查询匹配 QueryJob，指定 `BoidTarget`
    EntityQuery query_boidtarget;

    // 查询匹配 QueryJob，指定 `BoidObstacle`
    EntityQuery query_boidobstacle;

    protected override void OnCreate()
    {
        // 包含 `QueryJob` 中所有 Execute 参数，以及用户指定组件 `BoidTarget` 的查询
        query_boidtarget = GetEntityQuery(
            ComponentType.ReadWrite<SampleComponent>(),
            ComponentType.ReadOnly<BoidTarget>()
        );

        // 包含 `QueryJob` 中所有 Execute 参数，以及用户指定组件 `BoidObstacle` 的查询
        query_boidobstacle = GetEntityQuery(
            ComponentType.ReadWrite<SampleComponent>(),
            ComponentType.ReadOnly<BoidObstacle>()
        );
    }

    protected override void OnUpdate()
    {
        // 使用 BoidTarget 查询
        new QueryJob().ScheduleParallel(query_boidtarget);

        // 使用 BoidObstacle 查询
        new QueryJob().ScheduleParallel(query_boidobstacle);

        // 使用自动创建的匹配 `QueryJob` 参数的查询
        new QueryJob().ScheduleParallel();
    }
}

```

## 属性

`IJobEntity` 具有以下内置属性：

| 属性                                                        | 描述                                                                                               |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `Unity.Entities.WithAll(params Type[])`                   | 设置在作业结构体上。收窄查询，使实体必须匹配提供的所有类型。                                                                   |
| `Unity.Entities.WithAny(params Type[])`                   | 设置在作业结构体上。收窄查询，使实体必须匹配提供的任意类型中的一个。                                                               |
| `Unity.Entities.WithNone(params Type[])`                  | 设置在作业结构体上。收窄查询，使实体必须不包含提供的任何类型。                                                                  |
| `Unity.Entities.WithChangeFilter(params Type[])`          | 设置在作业结构体上或附加到 `Execute` 方法的参数上。收窄查询，使实体必须在给定组件的原型块中有变化。                                          |
| `Unity.Entities.WithOptions(params EntityQueryOptions[])` | 设置在作业结构体上。更改查询的范围，以使用所描述的 `EntityQueryOptions`。                                                  |
| `Unity.Entities.EntityIndexInQuery`                       | 设置在 `Execute` 中的 `int` 参数上，获取当前查询中的索引，用于当前实体迭代。这与 `Entities.ForEach` 中的 `entityInQueryIndex` 相同。 |

以下是一个使用 `EntityIndexInQuery` 属性的示例：

```csharp
[RequireMatchingQueriesForUpdate]
public partial class EntityInQuerySystem : SystemBase
{
    // 这个查询应匹配 `CopyPositionsJob` 的参数
    EntityQuery query;

    protected override void OnCreate()
    {
        // 获取与 `CopyPositionsJob` 参数匹配的查询
        query = GetEntityQuery(ComponentType.ReadOnly<LocalToWorld>());
    }

    protected override void OnUpdate()
    {
        // 获取一个本机数组，其大小等于查询找到的实体数量
        var positions = new NativeArray<float3>(query.CalculateEntityCount(), World.UpdateAllocator.ToAllocator);

        // 在并行线程上为该数组调度作业
        new CopyPositionsJob { copyPositions = positions }.ScheduleParallel();

        // 处理后释放作业找到的位置数组
        positions.Dispose(Dependency);
    }
}

// 定义组件
public struct LocalToWorld : IComponentData
{
    public float3 Position;
}

// 创建带有 EntityIndexInQuery 的 IJobEntity 作业
public partial struct CopyPositionsJob : IJobEntity
{
    public NativeArray<float3> copyPositions;

    void Execute([EntityIndexInQuery] int entityIndex, in LocalToWorld localToWorld)
    {
        // 将每个 LocalToWorld 的位置复制到相应索引的 positions 数组中
        copyPositions[entityIndex] = localToWorld.Position;
    }
}
```

因为 `IJobEntity` 是一个作业，你还可以使用所有适用于作业的属性：

* **`Unity.Burst.BurstCompile`**：将作业编译为高效的本机代码。
* **`Unity.Collections.DeallocateOnJobCompletion`**：在作业完成时自动释放分配的内存。
* **`Unity.Collections.NativeDisableParallelForRestriction`**：允许并行访问不安全的本机集合。
* **`Unity.Burst.BurstDiscard`**：防止 Burst 编译器编译标记的方法或代码段。
* **`Unity.Collections.LowLevel.Unsafe.NativeSetThreadIndex`**：设置线程索引，用于确定当前线程的本地数据。
* **`Unity.Burst.NoAlias`**：提示编译器指针不会重叠，从而允许更多优化。

## `Execute` 参数

以下是所有可以用于 `IJobEntity` 的 `Execute` 参数列表及其描述：

| 参数                      | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `IComponentData`        | 标记为 `ref` 进行读写访问，或者 `in` 进行只读访问组件数据。                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `ICleanupComponentData` | 标记为 `ref` 进行读写访问，或者 `in` 进行只读访问清理组件数据。                                                                                                                                                                                                                                                                                                                                                                                                                |
| `ISharedComponent`      | 标记为 `in` 进行只读访问共享组件数据。如果这是托管的，则无法使用 Burst 编译或调度它。请改用 `.Run`。                                                                                                                                                                                                                                                                                                                                                                                          |
| `Managed components`    | 使用值复制进行读写访问，或标记为 `in` 进行只读访问托管组件。例如，`UnityEngine.Transform`。将托管组件标记为 `ref` 是错误的，不能 Burst 编译或调度它。请改用 `.Run`。                                                                                                                                                                                                                                                                                                                                           |
| `Entity`                | 获取当前实体。这仅是值复制，因此不要标记为 `ref` 或 `in`。                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `DynamicBuffer<T>`      | 获取动态缓冲区。标记为 `ref` 进行读写访问，标记为 `in` 进行只读访问。                                                                                                                                                                                                                                                                                                                                                                                                             |
| `IAspect`               | 获取方面（Aspect）。方面作为引用，因此无法分配它们。但是，可以使用 `ref` 和值复制标记为读写，使用 `in` 标记为只读访问。                                                                                                                                                                                                                                                                                                                                                                                 |
| `int`                   | <p>有三种支持的 <code>int</code> 参数：</p><ul><li>标记为 <code>[Unity.Entities.ChunkIndexInQuery]</code> 以获取查询中当前原型块的索引。</li><li>标记为 <code>[Unity.Entities.EntityIndexInChunk]</code> 以获取当前原型块中当前实体的索引。可以添加 <code>EntityIndexInChunk</code> 和 <code>ChunkIndexInQuery</code> 以获得每个实体的唯一标识符。</li><li>标记为 <code>[Unity.Entities.EntityIndexInQuery]</code> 以获取查询的打包索引。此参数内部使用 <code>EntityQuery.CalculateBaseEntityIndexArray[Async]</code>，这会对性能产生负面影响。</li></ul> |

## 比较 `IJobEntity` 和 `Entities.ForEach`

`IJobEntity` 与 `Entities.ForEach` 类似，但 `IJobEntity` 可以在多个系统中重复使用，因此应尽可能优先使用 `IJobEntity`。下面是一个 `Entities.ForEach` 的示例：

#### `Entities.ForEach` 示例

```csharp
[RequireMatchingQueriesForUpdate]
public partial class BoidForEachSystem : SystemBase
{
    EntityQuery m_BoidQuery;
    EntityQuery m_ObstacleQuery;
    EntityQuery m_TargetQuery;

    protected override void OnUpdate()
    {
        // 计算查询中的实体数量
        var boidCount = m_BoidQuery.CalculateEntityCount();
        var obstacleCount = m_ObstacleQuery.CalculateEntityCount();
        var targetCount = m_TargetQuery.CalculateEntityCount();

        // 分配数组以存储与各自查询匹配的实体数据
        var cellSeparation = CollectionHelper.CreateNativeArray<float3, RewindableAllocator>(boidCount, ref World.UpdateAllocator);
        var copyTargetPositions = CollectionHelper.CreateNativeArray<float3, RewindableAllocator>(targetCount, ref World.UpdateAllocator);
        var copyObstaclePositions = CollectionHelper.CreateNativeArray<float3, RewindableAllocator>(obstacleCount, ref World.UpdateAllocator);

        // 为各自查询存储数组调度作业
        Entities
            .WithSharedComponentFilter(new BoidSetting { num = 1 })
            .ForEach((int entityInQueryIndex, in LocalToWorld localToWorld) =>
            {
                cellSeparation[entityInQueryIndex] = localToWorld.Position;
            })
            .ScheduleParallel();

        Entities
            .WithAll<BoidTarget>()
            .WithStoreEntityQueryInField(ref m_TargetQuery)
            .ForEach((int entityInQueryIndex, in LocalToWorld localToWorld) =>
            {
                copyTargetPositions[entityInQueryIndex] = localToWorld.Position;
            })
            .ScheduleParallel();

        Entities
            .WithAll<BoidObstacle>()
            .WithStoreEntityQueryInField(ref m_ObstacleQuery)
            .ForEach((int entityInQueryIndex, in LocalToWorld localToWorld) =>
            {
                copyObstaclePositions[entityInQueryIndex] = localToWorld.Position;
            })
            .ScheduleParallel();
    }
}
```

你可以使用 IJobEntity 重写上述代码：

```
[BurstCompile]
partial struct CopyPositionsJob : IJobEntity
{
    public NativeArray<float3> copyPositions;

    // 遍历所有 `LocalToWorld` 并将它们的位置存储在 `copyPositions` 中
    public void Execute([EntityIndexInQuery] int entityIndexInQuery, in LocalToWorld localToWorld)
    {
        copyPositions[entityIndexInQuery] = localToWorld.Position;
    }
}

[RequireMatchingQueriesForUpdate]
public partial class BoidJobEntitySystem : SystemBase
{
    EntityQuery m_BoidQuery;
    EntityQuery m_ObstacleQuery;
    EntityQuery m_TargetQuery;

    protected override void OnCreate()
    {
        // 获取包含 `CopyPositionsJob` 所需组件的相应查询
        m_BoidQuery = GetEntityQuery(typeof(LocalToWorld));
        m_BoidQuery.SetSharedComponentFilter(new BoidSetting { num = 1 });

        m_ObstacleQuery = GetEntityQuery(typeof(LocalToWorld), typeof(BoidObstacle));
        m_TargetQuery = GetEntityQuery(typeof(LocalToWorld), typeof(BoidTarget));
    }

    protected override void OnUpdate()
    {
        // 计算查询中的实体数量
        var boidCount = m_BoidQuery.CalculateEntityCount();
        var obstacleCount = m_ObstacleQuery.CalculateEntityCount();
        var targetCount = m_TargetQuery.CalculateEntityCount();

        // 分配数组以存储与各自查询匹配的实体数据
        var cellSeparation = CollectionHelper.CreateNativeArray<float3, RewindableAllocator>(boidCount, ref World.UpdateAllocator);
        var copyTargetPositions = CollectionHelper.CreateNativeArray<float3, RewindableAllocator>(targetCount, ref World.UpdateAllocator);
        var copyObstaclePositions = CollectionHelper.CreateNativeArray<float3, RewindableAllocator>(obstacleCount, ref World.UpdateAllocator);

        // 为各自查询存储数组调度作业
        new CopyPositionsJob { copyPositions = cellSeparation }.ScheduleParallel(m_BoidQuery);
        new CopyPositionsJob { copyPositions = copyTargetPositions }.ScheduleParallel(m_TargetQuery);
        new CopyPositionsJob { copyPositions = copyObstaclePositions }.ScheduleParallel(m_ObstacleQuery);
    }
}

```

虽然 Entities.ForEach 在某些情况下可能是有效的解决方案，但 IJobEntity 提供了更多灵活性和优化机会，建议在可能的情况下优先使用 IJobEntity。
