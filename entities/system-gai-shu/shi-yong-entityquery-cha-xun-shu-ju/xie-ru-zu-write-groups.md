# 写入组（Write Groups）

写入组提供了一种机制，即使你无法更改其他系统，也可以让一个系统覆盖另一个系统。

在 ECS 中，一个常见的模式是系统读取一组输入组件，并将结果写入另一组件。然而，你可能希望覆盖某系统的输出，使用基于不同输入集的另一个系统来更新输出组件。

目标组件类型的写入组由所有其他组件类型组成，这些组件类型通过 `WriteGroup` 特性应用于 ECS，以该目标组件类型作为参数。作为系统创建者，你可以使用写入组，使你的系统用户能够排除原本会被你的系统选择和处理的实体。这种过滤机制允许系统用户根据自己的逻辑更新排除实体的组件，同时让你的系统正常运行其余部分。

### 使用写入组

要使用写入组，必须在系统中的查询上使用写入组过滤选项。这将排除所有具有查询中任何可写组件的写入组中的组件的实体。

要覆盖使用写入组的系统，请将自己的组件类型标记为该系统输出组件的写入组的一部分。原始系统会忽略任何具有你组件的实体，你可以使用自己的系统更新这些实体的数据。

### 示例代码

#### 定义组件与写入组

假设我们有一个系统，它读取 `InputComponent` 并写入 `OutputComponent`。现在，我们想要创建一个新的系统，读取 `AlternativeInputComponent` 并写入到相同的 `OutputComponent`。

```csharp
public struct InputComponent : IComponentData
{
    public float Value;
}

public struct AlternativeInputComponent : IComponentData
{
    public float Value;
}

public struct OutputComponent : IComponentData
{
    public float Result;
}

// 标记 AlternativeInputComponent 为 OutputComponent 的写入组成员
[WriteGroup(typeof(OutputComponent))]
public struct MyWriteGroupComponent : IComponentData { }

//原始系统
[RequireMatchingQueriesForUpdate]
partial class OriginalSystem : SystemBase
{
    EntityQuery query;

    protected override void OnCreate()
    {
        // 创建一个查询，包含 InputComponent 和 OutputComponent
        query = new EntityQueryBuilder(Allocator.Temp)
            .WithAllRW<OutputComponent>()
            .WithAll<InputComponent>()
            .WithOptions(EntityQueryOptions.FilterWriteGroup)  // 使用写入组过滤选项
            .Build(this);
    }

    protected override void OnUpdate()
    {
        Entities.With(query).ForEach((ref OutputComponent output, in InputComponent input) =>
        {
            // 更新输出组件
            output.Result = input.Value * 2.0f;
        }).Schedule();
    }
}

//覆盖系统
[RequireMatchingQueriesForUpdate]
partial class OverrideSystem : SystemBase
{
    EntityQuery query;

    protected override void OnCreate()
    {
        // 创建一个查询，包含 AlternativeInputComponent 和 OutputComponent
        query = new EntityQueryBuilder(Allocator.Temp)
            .WithAllRW<OutputComponent>()
            .WithAll<AlternativeInputComponent, MyWriteGroupComponent>()
            .WithOptions(EntityQueryOptions.FilterWriteGroup)  // 使用写入组过滤选项
            .Build(this);
    }

    protected override void OnUpdate()
    {
        Entities.With(query).ForEach((ref OutputComponent output, in AlternativeInputComponent altInput) =>
        {
            // 使用替代输入组件更新输出组件
            output.Result = altInput.Value * 3.0f;
        }).Schedule();
    }
}

```
