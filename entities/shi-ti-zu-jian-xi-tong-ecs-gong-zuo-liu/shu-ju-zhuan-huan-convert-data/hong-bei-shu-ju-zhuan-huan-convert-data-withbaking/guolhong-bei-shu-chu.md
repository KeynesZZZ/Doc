# 过滤烘焙输出

默认情况下，转换世界中创建的每个实体和组件都是烘焙输出的一部分。

然而，并非每个创作场景中的 GameObject 都需要作为一个实体保留下来。例如，样条曲线上的控制点可能只在创作期间使用，在烘焙结束时可以丢弃。

### 排除实体

要从烘焙输出中排除实体，可以向实体添加 `BakingOnlyEntity` 标签组件。当你添加这个标签组件时，Unity 不会将该实体存储在实体场景中，也不会将其合并到主世界。一个 baker 可以直接将 `BakingOnlyEntity` 添加到一个实体，或者你可以将 `BakingOnlyEntityAuthoring` 添加到一个 GameObject 以达到相同的效果。

### 排除组件

你也可以使用以下属性排除组件：

* **`[BakingType]`**：过滤标记有此属性的任何组件，使其不包括在烘焙输出中。
* **`[TemporaryBakingType]`**：销毁标记有此属性的任何组件，使其不包括在烘焙输出中。这意味着标记有此属性的组件不会从一个烘焙通道保留到下一个，只在特定 baker 运行时存在。

可以排除组件以便从 baker 向烘焙系统传递信息。例如，一个 baker 可以将实体的边界框记录为烘焙类型，然后在烘焙过程中，一个烘焙系统可以收集所有边界框并计算凸包。如果只有凸包是有用的，边界框可以被丢弃。

在这个例子中，你必须使用 `[BakingType]` 属性，而不是 `[TemporaryBakingType]`。这是因为凸包系统需要访问所有边界框，而不仅仅是那些改变过的。然而，如果是一个需要处理与地面平面相交的所有实体的系统，那么最好只处理那些边界框确实发生变化的实体。因此，你可以使用 `[TemporaryBakingType]`。

### 示例代码

以下是使用 `[TemporaryBakingType]` 的示例：

#### 临时烘焙数据组件

```csharp
[TemporaryBakingType]
public struct TemporaryBakingData : IComponentData
{
    public float Mass;
}

public struct SomeComputedData : IComponentData
{
    // 计算此数据在 Baker 中太昂贵，我们希望利用 Burst（或者可能是 Jobs）
    public float ComputedValue;
}

public class RigidBodyBaker : Baker<Rigidbody>
{
    public override void Bake(Rigidbody authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new TemporaryBakingData{Mass = authoring.mass});

        // 即使我们不计算数据，我们也从 Baker 添加类型
        AddComponent(entity, new SomeComputedData());
    }
}

[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]
[BurstCompile]
partial struct SomeComputingBakingSystem : ISystem
{
    // 我们在这里进行性能关键的工作，因此使用 Burst
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // 因为 TemporaryBakingData 是 [TemporaryBakingType]，所以它只存在于 RigidBodyBaker 运行的同一烘焙通道
        // 这意味着只有当 RigidBodyBaker 的输入发生变化导致重新烘焙时，此烘焙系统才会运行
        // 此外，因为我们在此系统中不使用托管类型，所以可以使用 Burst
        foreach (var (computedComponent, bakingData) in
                 SystemAPI.Query<RefRW<SomeComputedData>, RefRO<TemporaryBakingData>>())
        {
            var mass = bakingData.ValueRO.Mass;
            float result = 0;
            // 在这里进行重计算，利用 Burst
            // result = ...
            computedComponent.ValueRW.ComputedValue = result;
        }
    }
}

```
