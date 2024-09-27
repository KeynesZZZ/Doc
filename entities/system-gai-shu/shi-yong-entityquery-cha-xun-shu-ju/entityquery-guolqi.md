# EntityQuery 过滤器

为了进一步分类实体，你可以使用过滤器根据以下条件排除实体：

* **共享组件过滤器**：基于共享组件的特定值过滤实体集。
* **变更过滤器**：基于特定组件类型的值是否已更改来过滤实体集。
* **可启用组件**

设置的过滤器会一直生效，直到你调用查询对象的 `ResetFilter` 方法。

要忽略查询的活动块过滤器，可以使用名称以 "IgnoreFilter" 结尾的 `EntityQuery` 方法。这些方法通常比等效的过滤方法更高效。例如，参见 `IsEmpty` 和 `IsEmptyIgnoreFilter` 的区别。

### 使用共享组件过滤器

要使用共享组件过滤器，将共享组件与任何其他需要的组件一起包含在 `EntityQuery` 中，并调用 `SetSharedComponentFilter` 方法。然后传入一个包含要选择的值的相同 `ISharedComponent` 类型的结构体。所有值都必须匹配。你可以向过滤器添加最多两个不同的共享组件。

你可以随时更改过滤器，但如果你更改了过滤器，则不会更改从组的 `ToComponentDataArray<T>` 或 `ToEntityArray` 方法接收到的任何现有实体或组件数组。你必须重新创建这些数组。

### 示例代码

以下示例定义了一个名为 `SharedGrouping` 的共享组件和一个仅处理 `group` 字段设置为 1 的实体的系统。

```csharp
public struct SharedGrouping : ISharedComponentData
{
    public int Group;
}

[RequireMatchingQueriesForUpdate]
partial class ImpulseSystem : SystemBase
{
    EntityQuery query;

    protected override void OnCreate()
    {
        // 创建一个查询，包含 ObjectPosition、Displacement 和 SharedGrouping 组件
        query = new EntityQueryBuilder(Allocator.Temp)
            .WithAllRW<ObjectPosition>()
            .WithAll<Displacement, SharedGrouping>()
            .Build(this);
    }

    protected override void OnUpdate()
    {
        // 默认情况下（没有过滤器），计算具有所需组件的所有实体。
        query.ResetFilter();
        int unfilteredCount = query.CalculateEntityCount();

        // 使用过滤器，只计算 SharedGrouping.Group 等于 1 的实体。
        query.SetSharedComponentFilter(new SharedGrouping { Group = 1 });
        int filteredCount = query.CalculateEntityCount();

        // 许多查询方法包括一个忽略任何活动过滤器的变体。这些变体通常更高效，
        // 并且在保守的上限结果是可以接受时应使用。
        int ignoreFilterCount = query.CalculateEntityCountWithoutFiltering();
    }
}


```

## 使用变更过滤器

如果你只需要在组件值发生变化时更新实体，可以使用 `SetChangedVersionFilter` 方法将该组件添加到 `EntityQuery` 过滤器中。例如，以下 `EntityQuery` 仅包括已经有其他系统写入 `Translation` 组件的块中的实体。

### 示例代码

#### 创建并使用变更过滤器的系统

```csharp
EntityQuery query;

protected override void OnCreate()
{
    // 创建一个查询，包含 LocalToWorld 和 ObjectPosition 组件
    query = new EntityQueryBuilder(Allocator.Temp)
        .WithAllRW<LocalToWorld>()
        .WithAll<ObjectPosition>()
        .Build(this);
    
    // 设置变更版本过滤器，只包括 ObjectPosition 组件已被更改的实体
    query.SetChangedVersionFilter(typeof(ObjectPosition));
}
```

注意

* 变更版本过滤器：通过调用 SetChangedVersionFilter 方法，查询将只包括某个组件自上次检查以来已被修改的实体。&#x20;
* 提高效率：使用变更过滤器可以显著提高系统性能，因为它避免了不必要的更新。

## 使用变更过滤器的注意事项

为了提高效率，变更过滤器适用于整个原型块，而不是单个实体。变更过滤器还仅检查声明对组件具有写访问权的系统是否已运行，而不检查其是否更改了任何数据。例如，如果另一个可以写入该组件类型的作业访问了该块，变更过滤器会包括该块中的所有实体。这就是为什么你应该始终声明对不需要修改的组件具有只读权限。

### 示例代码

#### 创建并使用变更过滤器的系统

```csharp
EntityQuery query;

protected override void OnCreate()
{
    // 创建一个查询，包含 LocalToWorld 和 ObjectPosition 组件
    query = new EntityQueryBuilder(Allocator.Temp)
        .WithAllRW<LocalToWorld>()   // 声明写访问
        .WithAll<ObjectPosition>()
        .Build(this);
    
    // 设置变更版本过滤器，只包括 ObjectPosition 组件已被更改的实体
    query.SetChangedVersionFilter(typeof(ObjectPosition));
}

protected override void OnUpdate()
{
    // 获取符合条件的实体数
    int entityCount = query.CalculateEntityCount();
    
    if (entityCount > 0)
    {
        // 对这些实体进行处理...
    }
}
```

## 通过可启用组件进行过滤

可启用组件允许在运行时启用和禁用单个实体上的组件。禁用实体上的组件不会将该实体移到新的原型中，但对于 `EntityQuery` 匹配的目的，实体被视为没有该组件。具体来说：

* 如果一个实体的组件 T 被禁用，它将不匹配需要组件 T 的查询（使用 `WithAll<T>()`）。
* 如果一个实体的组件 T 被禁用，它将匹配排除组件 T 的查询（使用 `WithNone<T>()`）。

大多数 `EntityQuery` 操作，如 `ToEntityArray` 和 `CalculateEntityCount`，会自动筛选出那些由于其可启用组件状态而不匹配查询的实体。要禁用此过滤，请使用这些操作的 `IgnoreFilter` 变体，或者在创建查询时传递 `EntityQueryOptions.IgnoreComponentEnabledState`。

详见 [可启用组件文档](https://docs.unity3d.com/Packages/com.unity.entities@latest/manual/enableable-components.html) 获取更多详情。

### 示例代码

#### 定义组件

```csharp
public struct MyEnableableComponent : IComponentData, IEnableableComponent { }

[RequireMatchingQueriesForUpdate]
partial class EnableableComponentSystem : SystemBase
{
    EntityQuery query;

    protected override void OnCreate()
    {
        // 创建一个查询，包含 LocalToWorld 和可启用的 MyEnableableComponent 组件
        query = new EntityQueryBuilder(Allocator.Temp)
            .WithAllRW<LocalToWorld>()
            .WithAll<MyEnableableComponent>()
            .WithOptions(EntityQueryOptions.IgnoreComponentEnabledState)  // 忽略组件启用状态
            .Build(this);
    }

    protected override void OnUpdate()
    {
        // 获取符合条件的实体数，自动忽略已禁用的 MyEnableableComponent
        int entityCount = query.CalculateEntityCount();
        
        // 获取符合条件的实体数组
        NativeArray<Entity> entities = query.ToEntityArray(Allocator.TempJob);

        if (entityCount > 0)
        {
            // 对这些实体进行处理...
        }

        // 释放 NativeArray
        entities.Dispose();
    }
}

```
