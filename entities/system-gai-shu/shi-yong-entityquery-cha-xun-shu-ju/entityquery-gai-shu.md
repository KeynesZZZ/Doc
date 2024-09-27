# EntityQuery 概述

`EntityQuery` 用于查找具有指定组件类型集合的原型。然后，它将原型的块（chunks）收集到一个数组中，供系统处理。

例如，如果一个查询匹配组件类型 A 和 B，那么该查询将收集所有包含这两个组件类型的原型块，无论这些原型可能还有其他什么组件类型。因此，包含组件类型 A、B 和 C 的原型也会匹配这个查询。

你可以使用 `EntityQuery` 执行以下操作：

* 运行作业以处理选定的实体和组件。
* 获取包含所有选定实体的 `NativeArray`。
* 通过组件类型获取选定实体的 `NativeArray`。

`EntityQuery` 返回的实体和组件数组是并行的。这意味着在任何数组中，相同的索引值总是应用于相同的实体。

### 在编辑器中的查询

在编辑器中，以下图标代表一个查询： ![Query Icon](https://example.com/query-icon.png)。当你使用特定的 Entities 窗口和检查器时，你会看到这个图标。你还可以使用查询窗口查看与所选查询匹配的组件和实体。

### 示例代码

下面是如何创建和使用 `EntityQuery` 的示例：



```csharp
// 创建一个查询，查找具有 LocalToWorld 和 BoidTarget 组件的实体
EntityQuery query = GetEntityQuery(ComponentType.ReadOnly<LocalToWorld>(), ComponentType.ReadOnly<BoidTarget>());

//运行作业处理选定实体和组件
public partial class ProcessEntitiesSystem : SystemBase
{
    protected override void OnUpdate()
    {
        EntityQuery query = GetEntityQuery(ComponentType.ReadOnly<LocalToWorld>(), ComponentType.ReadOnly<BoidTarget>());

        // 使用查询结果运行一个作业来处理实体
        Entities.WithStoreEntityQueryInField(ref query).ForEach((in LocalToWorld localToWorld) =>
        {
            // 处理代码...
        }).ScheduleParallel();
    }
}

//获取包含所有选定实体的 NativeArray
NativeArray<Entity> entities = query.ToEntityArray(Allocator.TempJob);

// 使用实体数组进行操作...

entities.Dispose();

//通过组件类型获取选定实体的 NativeArray
NativeArray<LocalToWorld> localToWorlds = query.ToComponentDataArray<LocalToWorld>(Allocator.TempJob);

// 使用组件数据数组进行操作...

localToWorlds.Dispose();

```
