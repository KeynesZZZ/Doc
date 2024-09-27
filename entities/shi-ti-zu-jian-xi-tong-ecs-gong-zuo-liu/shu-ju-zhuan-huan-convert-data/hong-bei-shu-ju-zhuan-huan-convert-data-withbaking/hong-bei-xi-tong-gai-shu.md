# 烘焙系统概述

烘焙系统通过查询操作 ECS 数据，并批量处理组件和实体。由于烘焙系统是一个系统，它可以使用作业和 Burst 编译，这对于重型处理非常理想。

烘焙系统与 baker 的数据处理方式不同。baker 从托管的创作数据中读取，并逐个处理组件，而烘焙系统则以批量的方式处理数据。

在运行烘焙系统之前，烘焙过程会创建所有实体（PreBakingSystemGroup中的系统除外）。这意味着烘焙系统可以处理最初创建的所有实体以及 bakers 创建的实体。

烘焙系统不会自动跟踪依赖关系和结构变化。因此，你必须显式声明依赖关系。此外，在添加组件时，为了保持增量烘焙，你必须手动跟踪并撤销更改。

烘焙系统可以以任何方式改变世界，包括创建新实体。然而，任何你在烘焙系统中创建的实体都不会最终出现在烘焙的实体场景中。你可以在烘焙系统中创建实体以在烘焙系统之间传递数据，但如果你希望实体最终出现在烘焙的实体场景中，你必须在 baker 中创建它。当你在 baker 中创建实体时，`CreateAdditionalEntity` 会配置该实体以正确地工作于烘焙和实时烘焙中。

### 创建一个烘焙系统

要创建一个烘焙系统，请使用 `[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]` 属性标记它。这使得烘焙过程能够发现烘焙系统并将它们添加到烘焙世界中。Unity 会在每次烘焙过程中更新烘焙系统。

以下是一个烘焙系统的示例，它向每个具有另一个组件的实体添加标签组件。如果组件不存在，它还会移除标签。否则，移除组件会导致标签仍然留在实体上，从而造成不一致的结果：

```csharp
public struct AnotherTag : IComponentData { }

[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]
partial struct AddTagToRotationBakingSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var queryMissingTag = SystemAPI.QueryBuilder()
            .WithAll<RotationSpeed>()
            .WithNone<AnotherTag>()
            .Build();

        state.EntityManager.AddComponent<AnotherTag>(queryMissingTag);

        // 省略此函数的第二部分将在实时烘焙过程中导致不一致的结果。
        // 添加的标签即使在移除 RotationSpeed 组件后也会留在实体上。

        var queryCleanupTag = SystemAPI.QueryBuilder()
            .WithAll<AnotherTag>()
            .WithNone<RotationSpeed>()
            .Build();

        state.EntityManager.RemoveComponent<AnotherTag>(queryCleanupTag);
    }
}
```
