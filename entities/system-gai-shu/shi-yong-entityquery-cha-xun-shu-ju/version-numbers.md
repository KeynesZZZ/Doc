# Version numbers

版本号（也称为代数）在 ECS 架构中用于检测潜在的变化，并实施有效的优化策略，例如在数据自上次帧以来未发生变化时跳过处理。对实体进行快速版本检查有助于提高应用程序的性能。

本页概述了 ECS 使用的所有不同版本号，以及导致它们变化的条件。

## 版本号结构

所有版本号都是 32 位有符号整数。当它们溢出时，它们会环绕回到最小值：在 C# 中，整型溢出是已定义的行为。这意味着在比较版本号时，应使用等号（==）或不等号（!=）运算符，而不是关系运算符。

例如，检查 `VersionB` 是否比 `VersionA` 更新的正确方式如下：

```csharp
bool VersionBIsMoreRecent = (VersionB - VersionA) > 0;
```

## 版本号详解

### 实体版本号

一个 `EntityId` 包含索引和版本号。由于 ECS 会回收索引，每次销毁实体时，`EntityManager` 都会增加版本号。如果在 `EntityManager` 中查找 `EntityId` 时版本号不匹配，则表示所引用的实体已不存在。

例如，在通过 `EntityId` 获取单位跟踪的敌方位置之前，可以调用 `ComponentDataFromEntity.Exists` 来检查实体是否仍然存在，这会使用版本号来进行验证。

#### 示例代码：检查实体是否存在

```csharp
public struct Enemy : IComponentData { public Entity Target; }

public partial class CheckEnemySystem : SystemBase
{
    protected override void OnUpdate()
    {
        var positionFromEntity = GetComponentDataFromEntity<Translation>(true);

        Entities.ForEach((ref Enemy enemy, in Translation translation) =>
        {
            if (positionFromEntity.Exists(enemy.Target))
            {
                // 敌人目标实体存在，可以获取其位置
                var targetPosition = positionFromEntity[enemy.Target];
                // 处理逻辑...
            }
            else
            {
                // 敌人目标实体已不存在
            }
        }).Schedule();
    }
}
```

#### 世界版本号

&#x20;每次创建或销毁管理器（如系统）时，ECS 都会增加世界的版本号

#### 系统版本号

EntityDataManager.GlobalVersion 在每次系统更新前都会增加。

你应该将此版本号与 System.LastSystemVersion 一起使用。后者在每次系统更新后获取 EntityDataManager.GlobalVersion 的值。

你还应将此版本号与 Chunk.ChangeVersion\[] 一起使用。

#### 块更改版本号

对于原型中的每个组件类型，此数组包含该组件在块中最后一次以可写方式访问时的 EntityDataManager.GlobalVersion 值。这并不保证有任何变化，只是可能发生了变化。

尽管共享组件也存储了版本号，但无法以可写方式访问它们，因此没有意义。

当在 Entities.ForEach 构造中使用 WithChangeFilter() 方法时，ECS 会将特定组件的 Chunk.ChangeVersion 与 System.LastSystemVersion 进行比较，仅处理那些在系统最后一次运行后以可写方式访问过的块。

示例代码：使用 WithChangeFilter 进行优化

```csharp
public struct Health : IComponentData { public int Value; }

public partial class UpdateDamageModelSystem : SystemBase
{
    private EntityQuery _query;

    protected override void OnCreate()
    {
        // 创建查询，包含 Health 组件
        _query = GetEntityQuery(ComponentType.ReadOnly<Health>());
        _query.SetChangedVersionFilter(typeof(Health));
    }

    protected override void OnUpdate()
    {
        Entities.With(_query).ForEach((in Health health) =>
        {
            // 更新伤害模型的逻辑
        }).Schedule();
    }
}

```

#### 非共享组件版本号&#x20;

对于每种非共享组件类型，每当涉及该类型的迭代器变得无效时，ECS 会增加 EntityManager.m\_ComponentTypeOrderVersion\[] 版本号。换句话说，任何可能修改该类型数组的操作都会增加版本号。

示例代码：更新静态对象的边界框

```csharp
// Some code
public struct StaticObject : IComponentData { }
public struct BoundingBox : IComponentData { public AABB Bounds; }

public partial class UpdateBoundingBoxSystem : SystemBase
{
    private uint _lastStaticObjectVersion;

    protected override void OnCreate()
    {
        _lastStaticObjectVersion = EntityManager.GetComponentTypeVersion<StaticObject>();
    }

    protected override void OnUpdate()
    {
        uint currentVersion = EntityManager.GetComponentTypeVersion<StaticObject>();

        if (_lastStaticObjectVersion != currentVersion)
        {
            // 更新边界框逻辑
            _lastStaticObjectVersion = currentVersion;
        }
    }
}

```

#### 共享组件版本号&#x20;

当引用共享组件的块中的实体发生任何结构变化时，SharedComponentDataManager.m\_SharedComponentVersion\[] 版本号会增加。

示例代码：基于共享组件版本号进行优化

```csharp
public struct SharedTag : ISharedComponentData { public int Tag; }

public partial class CountEntitiesByTagSystem : SystemBase
{
    private Dictionary<int, int> _tagCounts = new Dictionary<int, int>();
    private NativeArray<int> _sharedComponentVersions;

    protected override void OnUpdate()
    {
        var sharedComponentData = GetSharedComponentData<SharedTag>(Allocator.TempJob);
        var sharedComponentVersion = EntityManager.GetSharedComponentOrderVersion<SharedTag>();

        // 检查版本号是否变化
        foreach (var kvp in sharedComponentData)
        {
            if (_sharedComponentVersions[kvp.Key] != sharedComponentVersion[kvp.Key])
            {
                // 版本号改变，重新计算实体数
                _tagCounts[kvp.Value.Tag] = CalculateEntityCount(kvp.Value.Tag);
                _sharedComponentVersions[kvp.Key] = sharedComponentVersion[kvp.Key];
            }
        }
    }

    private int CalculateEntityCount(int tag)
    {
        // 实现你的计数逻辑
    }
}

```
