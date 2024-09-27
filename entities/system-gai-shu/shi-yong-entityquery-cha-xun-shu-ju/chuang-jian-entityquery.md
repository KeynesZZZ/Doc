# 创建 EntityQuery

要创建一个实体查询，你可以将组件类型传递给 `EntityQueryBuilder` 辅助类型。以下示例定义了一个查找所有具有 `ObjectRotation` 和 `ObjectRotationSpeed` 组件的实体的 `EntityQuery`：

```csharp
EntityQuery query = new EntityQueryBuilder(Allocator.Temp)
    .WithAllRW<ObjectRotation>()  // 表明系统会写入 ObjectRotation
    .WithAll<ObjectRotationSpeed>()  // 指定读取 ObjectRotationSpeed
    .Build(this);
```

该查询使用 EntityQueryBuilder.WithAllRW 来表明系统会对 ObjectRotation 进行写操作。你应该尽可能指定只读访问，因为对数据的只读访问约束较少，这有助于作业调度器更高效地执行作业。

## 指定系统选择的原型（Archetypes）

查询仅匹配包含你指定组件的原型。你可以使用以下 `EntityQueryBuilder` 方法来指定组件：

* **`WithAll<T>()`**：要匹配查询，实体的原型必须包含所有查询所需的组件，并且这些组件必须在该实体上启用。
* **`WithAny<T>()`**：要匹配查询，实体的原型必须至少包含查询的一个可选组件，并且这些组件必须在该实体上启用。
* **`WithNone<T>()`**：要匹配查询，实体的原型不能包含任何查询排除的组件，或者这些组件存在但在该实体上被禁用。
* **`WithDisabled<T>()`**：要匹配查询，实体的原型必须包含此组件，并且该组件必须在该实体上被禁用。
* **`WithAbsent<T>()`**：要匹配查询，实体的原型不能包含指定的组件。
* **`WithPresent<T>()`**：要匹配查询，实体的原型必须包含指定的组件（无论它们是否启用）。

例如，以下查询包括包含 `ObjectRotation` 和 `ObjectRotationSpeed` 组件的原型，但排除任何包含 `Static` 组件的原型：

```csharp
EntityQuery query = new EntityQueryBuilder(Allocator.Temp)
    .WithAllRW<ObjectRotation>()  // 表明系统会写入 ObjectRotation
    .WithAll<ObjectRotationSpeed>()  // 指定读取 ObjectRotationSpeed
    .WithNone<Static>()  // 排除包含 Static 组件的原型
    .Build(this);
```

## 重要提示

为了处理可选组件，使用 `ArchetypeChunk.Has<T>` 方法来确定一个块是否包含可选组件。这是因为同一个块中的所有实体具有相同的组件，所以你只需要检查一次每个块中是否存在可选组件，而不是逐个实体进行检查。

markdown 复制代码

## 使用 `EntityQueryBuilder.WithOptions()` 查找特定原型

你可以使用 `EntityQueryBuilder.WithOptions()` 来查找特定的原型。以下是一些例子：

* **`IncludePrefab`**：包括包含 `Prefab` 标签组件的原型。
* **`IncludeDisabledEntities`**：包括包含 `Disabled` 标签组件的原型。
* **`FilterWriteGroup`**：仅包含查询中明确包含写入组（WriteGroup）中的组件的实体，排除具有相同写入组中的任何其他组件的实体。

请参阅 `EntityQueryOptions` 以获取完整的选项列表。

## 根据写入组进行过滤

在以下示例中，`LuigiComponent` 和 `MarioComponent` 是基于 `CharacterComponent` 组件的同一个写入组中的组件。这个查询使用了 `FilterWriteGroup` 选项来要求 `CharacterComponent` 和 `MarioComponent`：

### 示例代码

#### 定义组件和系统

```csharp
// 定义基础组件
public struct CharacterComponent : IComponentData { }

// 将 LuigiComponent 写入到 CharacterComponent 中
[WriteGroup(typeof(CharacterComponent))]
public struct LuigiComponent : IComponentData { }

// 将 MarioComponent 写入到 CharacterComponent 中
[WriteGroup(typeof(CharacterComponent))]
public struct MarioComponent : IComponentData { }

[RequireMatchingQueriesForUpdate]
public partial class ECSSystem : SystemBase
{
    protected override void OnCreate()
    {
        // 创建一个查询，包含 CharacterComponent 和 MarioComponent，并使用 FilterWriteGroup 选项
        var query = new EntityQueryBuilder(Allocator.Temp)
            .WithAllRW<CharacterComponent>()
            .WithAll<MarioComponent>()
            .WithOptions(EntityQueryOptions.FilterWriteGroup)  // 过滤写入组
            .Build(this);
    }

    protected override void OnUpdate()
    {
        throw new NotImplementedException();
    }
}
```

这个查询排除了同时包含 LuigiComponent 和 MarioComponent 的实体，因为 LuigiComponent 并未明确包含在查询中。

这种方法比使用 None 字段更高效，因为你不需要更改其他系统使用的查询，只要它们也使用写入组即可。

你可以使用写入组来扩展现有系统。例如，如果你在其他系统中已经定义了 CharacterComponent 和 LuigiComponent 作为你无法控制的库的一部分，你可以将 MarioComponent 放入与 LuigiComponent 相同的写入组，以改变 CharacterComponent 的更新方式。对于添加了 MarioComponent 的任何实体，该系统将更新 CharacterComponent，但原始系统不会更新它。对于没有 MarioComponent 的实体，原始系统仍会像以前一样更新 CharacterComponent。有关更多信息，请参阅关于写入组的文档。



通过使用写入组（WriteGroups），你可以实现更加高效且灵活的系统设计。

这个查询排除了同时包含 `LuigiComponent` 和 `MarioComponent` 的实体，因为 `LuigiComponent` 并未明确包含在查询中。

这种方法比使用 `None` 字段更高效，因为你不需要更改其他系统使用的查询，只要它们也使用写入组即可。

### 示例代码

#### 定义组件和系统

假设我们有如下定义的组件：

```csharp
// 基础组件
public struct CharacterComponent : IComponentData { }

// LuigiComponent 属于 CharacterComponent 的写入组
[WriteGroup(typeof(CharacterComponent))]
public struct LuigiComponent : IComponentData { }

// MarioComponent 也属于 CharacterComponent 的写入组
[WriteGroup(typeof(CharacterComponent))]
public struct MarioComponent : IComponentData { }
```

## 执行查询

通常，当你调度一个使用实体查询的作业时，会执行该查询。你也可以调用以下 `EntityQuery` 方法之一，以返回实体、组件或原型块的数组：

* **`ToEntityArray`**：返回选定实体的数组。
* **`ToComponentDataArray`**：返回选定实体的类型为 T 的组件数组。
* **`CreateArchetypeChunkArray`**：返回包含选定实体的所有块。因为查询操作基于原型、共享组件值和变更过滤器，而这些在一个块中的所有实体都是相同的，所以返回的块集合中存储的实体集与 `ToEntityArray` 返回的实体集相同。

上述方法的异步版本也可用，它们会调度一个作业以收集请求的数据。这些变体中的一些必须返回 `NativeList` 而不是 `NativeArray` 以支持可启用的组件。请参阅 `ToEntityListAsync`、`ToComponentDataListAsync` 和 `CreateArchetypeChunkArrayAsync`。

### 示例代码

#### 使用同步方法执行查询

```csharp
public partial class ExecuteQuerySystem : SystemBase
{
    private EntityQuery m_Query;

    protected override void OnCreate()
    {
        // 创建一个查询，包含 CharacterComponent 和 MarioComponent，并使用 FilterWriteGroup 选项
        m_Query = new EntityQueryBuilder(Allocator.Temp)
            .WithAllRW<CharacterComponent>()
            .WithAll<MarioComponent>()
            .WithOptions(EntityQueryOptions.FilterWriteGroup)  // 过滤写入组
            .Build(this);
    }

    protected override void OnUpdate()
    {
        // 获取实体数组
        NativeArray<Entity> entities = m_Query.ToEntityArray(Allocator.TempJob);

        // 获取组件数据数组
        NativeArray<MarioComponent> marioComponents = m_Query.ToComponentDataArray<MarioComponent>(Allocator.TempJob);

        // 获取原型块数组
        NativeArray<ArchetypeChunk> chunks = m_Query.CreateArchetypeChunkArray(Allocator.TempJob);

        // 进行处理...
        
        // 别忘了释放 NativeArray
        entities.Dispose();
        marioComponents.Dispose();
        chunks.Dispose();
    }
}
```
