# Transform 概念

你可以使用 `Unity.Transforms` 命名空间来控制项目中任何实体的位置、旋转和缩放。

### 变换系统中的组件

以下组件用于变换系统：

* `LocalToWorld`
* `LocalTransform`
* `PostTransformMatrix`
* `Parent`
* `Child`

### 更新变换系统的两个系统

* `ParentSystem`
* `LocalToWorldSystem`

### 变换层次结构

`Unity.Transforms` 是分层的，这意味着你可以基于实体之间的关系来变换实体。

例如，汽车车身可以是其车轮的父级。车轮是车身的子级。当车身移动时，车轮也随之移动。你还可以相对于车身移动和旋转车轮。

一个实体可以有多个子级，但只能有一个父级。子级可以有自己的子实体。这些多级的父子关系形成了一个变换层次结构。层次结构顶端没有父级的实体称为根。

要声明变换层次结构，你必须从下向上进行声明。这意味着你使用 `Parent` 来声明一个实体的父级，而不是声明它的子级。如果你想声明一个实体的子级，找到你想要作为子级的实体并将它们的父级设置为目标实体。更多信息请参阅 [使用层次结构文档](https://docs.unity3d.com/Manual/ecs-system-update-order.html)。

### LocalToWorld 组件

`LocalToWorld` 矩阵表示从局部空间到世界空间的变换。渲染系统使用这个矩阵来渲染实体的几何形状。默认情况下，`LocalToWorldSystem` 从 `LocalToWorld` 组件更新此组件。这个默认更新意味着你不需要更新 `LocalToWorld`，系统会为你进行更新。要禁用此自动更新，可以在组件上使用 `[WriteGroup(typeof(LocalToWorld))]` 属性。有了这个写入组，你可以完全控制 `LocalToWorld`。更多信息请参阅 [写入组文档](https://docs.unity3d.com/Manual/ecs-component-groups.html)。

#### 注意

`LocalToWorld` 组件值在 `SimulationSystemGroup` 运行时可能会过时或无效。这是因为变换系统只在 `TransformSystemGroup` 运行时更新组件值。它可能还包含为了图形平滑目的应用的额外偏移。因此，虽然 `LocalToWorld` 组件在延迟可接受时可以作为实体世界空间变换的快速近似值，但如果你需要准确的、最新的世界变换进行模拟，则不应依赖它。在这种情况下，请使用 `ComputeWorldTransformMatrix` 方法。

## LocalTransform 组件

`LocalTransform` 组件有三个属性。它们控制实体的位置、旋转和缩放。当此组件存在时，它控制 `LocalToWorld`：

### 组件定义

```csharp
public struct LocalTransform : IComponentData
{
    public float3 Position;
    public float Scale;
    public quaternion Rotation;
}
```

如果实体具有 Parent 组件，Position、Rotation 和 Scale 是相对于其父级来说的。 如果实体没有 Parent 组件，则变换是相对于世界原点的。

### PostTransformMatrix Component

`LocalTransform` 只支持均匀缩放。要渲染具有非均匀缩放的几何体，你必须使用 `PostTransformMatrix` 矩阵。它在 `LocalTransform` 之后应用。你还可以使用此组件引入剪切变换，或相对于其枢轴平移实体。

#### 组件定义

```csharp
public struct PostTransformMatrix : IComponentData
{
    public float4x4 Value;
}
```

### Parent Component

&#x20;Parent 组件定义层次结构。你需要将其添加到每个你希望成为层次结构一部分的子组件上。

### Child Component

&#x20;Child 组件缓冲区保存父级的所有子组件。ParentSystem 管理此缓冲区及其内容。你只需管理 Parent 组件。系统会负责维护相应的 Child 组件。

### ParentSystem

ParentSystem 根据子组件的 Parent 组件维护 Child 组件缓冲区。当你在子组件上设置 Parent 组件时，Unity 仅在 ParentSystem 运行后更新父组件的 Child 组件。

### LocalToWorldSystem

LocalToWorldSystem 基于 LocalTransform 组件和层次结构计算并更新 LocalToWorld 组件。当你在实体上设置 LocalTransform 组件时，Unity 仅在 LocalToWorldSystem 运行后更新其 LocalToWorld 组件。



以下示例展示了如何设置父子关系，以及如何使用 PostTransformMatrix 来实现非均匀缩放：

```csharp
using Unity.Entities;
using Unity.Transforms;
using Unity.Mathematics;

public partial class TransformExampleSystem : SystemBase
{
    protected override void OnUpdate()
    {
        var entityManager = World.DefaultGameObjectInjectionWorld.EntityManager;

        // 创建父实体
        Entity parentEntity = entityManager.CreateEntity(typeof(LocalTransform), typeof(Child));
        entityManager.SetComponentData(parentEntity, new LocalTransform
        {
            Position = new float3(0, 0, 0),
            Rotation = quaternion.identity,
            Scale = 1f
        });

        // 创建子实体
        Entity childEntity = entityManager.CreateEntity(typeof(LocalTransform), typeof(Parent), typeof(PostTransformMatrix));
        entityManager.SetComponentData(childEntity, new LocalTransform
        {
            Position = new float3(1, 0, 0),
            Rotation = quaternion.identity,
            Scale = 1f
        });
        entityManager.SetComponentData(childEntity, new Parent { Value = parentEntity });
        entityManager.SetComponentData(childEntity, new PostTransformMatrix 
        { 
            Value = float4x4.Scale(new float3(2f, 1f, 1f)) // 非均匀缩放
        });

        Entities
            .ForEach((ref LocalTransform transform, in Velocity velocity) =>
            {
                transform.Position += velocity.Value * SystemAPI.Time.DeltaTime;
            })
            .ScheduleParallel();
    }
}

```
