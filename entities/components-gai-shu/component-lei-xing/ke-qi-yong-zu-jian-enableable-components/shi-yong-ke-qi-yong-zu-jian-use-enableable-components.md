# 使用可启用组件 (Use enableable components)

您只能将 `IComponentData` 和 `IBufferElementData` 组件设为可启用。要实现这一点，请实现 `IEnableableComponent` 接口。

当您使用可启用组件时，目标实体不会改变其原型（archetype），ECS 不会移动任何数据，并且组件的现有值保持不变。这意味着您可以在工作线程上运行的作业中启用和禁用组件，而无需使用实体命令缓冲区或创建同步点。

然而，为防止竞争条件，对可启用组件具有写访问权限的作业可能会导致主线程操作阻塞，直到作业完成，即使作业未在任何实体上启用或禁用该组件。

通过 `CreateEntity()` 创建的新实体上的所有可启用组件默认都是启用的。从预制件实例化的实体继承预制件的启用或禁用状态。

### 可启用组件方法 (Enableable component methods)

要使用可启用组件，您可以在 `EntityManager`、`ComponentLookup<T>`、`EntityCommandBuffer` 和 `ArchetypeChunk` 上使用以下方法：

#### 方法简介

* **`IsComponentEnabled<T>(Entity e)`**： 如果实体 `e` 拥有组件 `T` 且该组件已启用，则返回 `true`。如果实体 `e` 拥有组件 `T` 但该组件已禁用，则返回 `false`。如果实体 `e` 不拥有组件 `T` 或者 `T` 没有实现 `IEnableableComponent`，则会断言。
* **`SetComponentEnabled<T>(Entity e, bool enable)`**： 如果实体 `e` 拥有组件 `T`，则根据 `enable` 的值启用或禁用该组件。如果实体 `e` 不拥有组件 `T` 或者 `T` 没有实现 `IEnableableComponent`，则会断言。

#### 示例代码

以下是一个示例，展示了如何使用这些方法来工作：

```csharp
public partial struct EnableableComponentSystem : ISystem
{
    public void OnUpdate(ref SystemState system)
    {
        Entity e = system.EntityManager.CreateEntity(typeof(Health));

        ComponentLookup<Health> healthLookup = system.GetComponentLookup<Health>();

        // 判断 Health 组件是否启用（默认情况下是启用的）
        bool isEnabled = healthLookup.IsComponentEnabled(e); // 返回 true

        // 禁用实体的 Health 组件
        healthLookup.SetComponentEnabled(e, false);

        // 尽管被禁用，仍然可以读取和修改该组件
        Health h = healthLookup[e];
    }
}
```

您可以使用 `ComponentLookup<T>.SetComponentEnabled(Entity, bool)` 在工作线程中安全地启用或禁用实体，因为不需要结构性更改。作业必须对组件 `T` 具有写访问权限。避免在另一个线程可能处理的实体上启用或禁用组件，因为这通常会导致竞争条件。

### 查询可启用组件 (Querying enableable components)

如果实体的组件 `T` 被禁用，则在查询中它表现得好像根本没有该组件。例如，如果实体 `E` 拥有组件 `T1`（已启用）、`T2`（已禁用）和 `T3`（已禁用）：

* **不匹配**：需要同时拥有 `T1` 和 `T2` 的查询。
* **匹配**：需要 `T1` 并排除 `T2` 的查询。
* **不匹配**：将 `T2` 和 `T3` 作为可选组件的查询，因为它没有至少一个启用的这些组件。

所有 `EntityQuery` 方法都会自动处理可启用组件。例如，`query.CalculateEntityCount()` 计算符合查询条件的实体数量时，会考虑它们的组件是启用还是禁用状态。以下是两个例外情况：

1. **方法名以 IgnoreFilter 结尾**： 这些方法将所有组件视为已启用。因为只有结构性更改会影响其结果， 所以这些方法不需要同步点。与尊重过滤的变体相比，它们通常效率更高。
2. **使用 `EntityQueryOptions.IgnoreComponentEnabledState` 创建的查询**： 在确定实体是否匹配查询时，这些查询忽略所有匹配原型中实体的当前启用/禁用状态。

以下是一个使用 `EntityManager.IsComponentEnabled` 查询已禁用组件的示例：

```csharp
public partial struct EnableableHealthSystem : ISystem
{
    public void OnUpdate(ref SystemState system)
    {
        Entity e1 = system.EntityManager.CreateEntity(typeof(Health), typeof(Translation));
        Entity e2 = system.EntityManager.CreateEntity(typeof(Health), typeof(Translation));

        // true（组件默认状态为启用）
        bool isEnabled = system.EntityManager.IsComponentEnabled<Health>(e1);

        // 禁用第一个实体上的 Health 组件
        system.EntityManager.SetComponentEnabled<Health>(e1, false);

        // 创建一个 EntityQuery，查询具有 Health 和 Translation 组件的实体
        EntityQuery query = new EntityQueryBuilder(Allocator.Temp).WithAll<Health, Translation>().Build(ref system);

        // 返回的数组不包括第一个实体
        var entities = query.ToEntityArray(Allocator.Temp);

        // 返回的数组不包括第一个实体的 Health 组件
        var healths = query.ToComponentDataArray<Health>(Allocator.Temp);

        // 返回的数组不包括第一个实体的 Translation 组件
        var translations = query.ToComponentDataArray<Translation>(Allocator.Temp);

        // 此查询匹配无论是否启用的组件
        var queryIgnoreEnableable = new EntityQueryBuilder(Allocator.Temp)
                                    .WithAll<Health, Translation>()
                                    .WithOptions(EntityQueryOptions.IgnoreComponentEnabledState)
                                    .Build(ref system);

        // 返回的数组包括两个实体的 Translation 组件
        var translationsAll = queryIgnoreEnableable.ToComponentDataArray<Translation>(Allocator.Temp);
    }
}
```

### 异步操作 (Asynchronous operations)

为了安全且确定性地处理可启用组件，所有同步的 `EntityQuery` 操作（忽略过滤的除外）会自动等待任何具有写入访问权限的正在运行的作业完成，以确保查询中的可启用组件安全。所有异步的 `EntityQuery` 操作（以 `Async` 结尾的那些）也会自动在这些正在运行的作业上插入一个输入依赖项。

#### 异步 EntityQuery 收集和分散操作

例如，`EntityQuery.ToEntityArrayAsync()` 等异步收集和分散操作会调度一个作业来执行所请求的操作。这些方法必须返回一个 `NativeList` 而不是 `NativeArray`，因为在作业运行之前，无法知道查询匹配的实体数量，但容器必须立即返回给调用者。

这个列表的初始容量是根据查询可能匹配的最大实体数量保守估算的，但其最终长度可能会更低。在异步收集或分散作业完成之前，对列表的任何读取或写入（包括其当前长度、容量或基本指针）都会导致 `JobsDebugger` 安全错误。不过，您可以安全地将该列表传递给依赖的后续作业。

#### 示例代码

以下是如何使用异步操作的示例：

```csharp
public partial struct AsyncEnableableHealthSystem : ISystem
{
    public void OnUpdate(ref SystemState system)
    {
        EntityQuery query = new EntityQueryBuilder(Allocator.Temp).WithAll<Health, Translation>().Build(ref system);

        // 创建一个空的 NativeList 来存储查询结果
        var entityList = new NativeList<Entity>(Allocator.TempJob);

        // 启动一个异步任务来填充这个列表
        JobHandle handle = query.ToEntityArrayAsync(entityList, Allocator.TempJob, out JobHandle arrayHandle);
        
        // 保证主线程上的作业依赖于以上的异步作业
        system.Dependency = JobHandle.CombineDependencies(system.Dependency, arrayHandle);

        // 可以将这个列表传递给后续依赖的作业
        var dependentJob = new ProcessEntitiesJob
        {
            Entities = entityList
        }.Schedule(arrayHandle);

        // 保证系统依赖于这个后续作业
        system.Dependency = JobHandle.CombineDependencies(system.Dependency, dependentJob);
    }
}

public struct ProcessEntitiesJob : IJob
{
    public NativeList<Entity> Entities;

    public void Execute()
    {
        for (int i = 0; i < Entities.Length; i++)
        {
            // 处理每个实体
            Entity entity = Entities[i];
            // 这里进行实际的处理逻辑
        }
    }
}
```
