# 变换助手概述

`TransformHelpers` 类包含对数学库的扩展，使你更容易处理变换矩阵。

特别是，这些扩展方法帮助你处理 `LocalToWorld` 组件中包含的 `float4x4` 矩阵。

### 扩展方法

你可以使用 `TransformHelpers` 中的扩展方法来最小化代码中矩阵数学的使用。例如，要将一个点从局部空间变换到世界空间，可以使用 `TransformPoint`：

```csharp
float3 myWorldPoint = myLocalToWorld.Value.TransformPoint(myLocalPoint);
```

或者，要将一个点从世界空间变换到局部空间，可以使用 InverseTransformPoint：

```csharp
float3 myLocalPoint = myLocalToWorld.Value.InverseTransformPoint(myWorldPoint);
```

## 其他方法

以下是一些不是扩展的其他方法。

### LookAtRotation

`LookAtRotation` 方法计算一个旋转，以便“前方”指向目标：

#### 示例代码

```csharp
float3 eyeWorldPosition = new float3(1, 2, 3);
float3 targetWorldPosition = new float3(4, 5, 6);
quaternion lookRotation = TransformHelpers.LookAtRotation(eyeWorldPosition, targetWorldPosition, math.up());

using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public partial class LookAtRotationSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 定义观察者和目标的位置
        float3 eyeWorldPosition = new float3(1, 2, 3);
        float3 targetWorldPosition = new float3(4, 5, 6);

        Entities
            .ForEach((ref LocalTransform transform) =>
            {
                // 计算指向目标的旋转
                quaternion lookRotation = TransformHelpers.LookAtRotation(eyeWorldPosition, targetWorldPosition, math.up());
                
                // 应用旋转到变换组件
                transform.Rotation = lookRotation;
            })
            .ScheduleParallel();
    }
}

```

## 计算世界变换矩阵

你可以使用 `ComputeWorldTransformMatrix` 方法立即使用实体的精确世界空间变换矩阵。例如：

* 当从一个可能是实体层次结构一部分的实体（如汽车对象的车轮）执行射线投射时，射线的原点必须在世界空间中，但实体的 `LocalTransform` 组件可能是相对于其父级的。
* 当一个实体的变换需要在世界空间中跟踪另一个实体的变换，并且目标实体或被跟踪的实体在变换层次结构中。
* 当一个实体的变换在 `LateSimulationSystemGroup` 中被修改（在 `TransformSystemGroup` 更新之后，但在 `PresentationSystemGroup` 运行之前），你可以使用 `ComputeWorldTransformMatrix` 为受影响的实体计算新的 `LocalToWorld` 值。

### 示例代码：使用 ComputeWorldTransformMatrix 方法

以下示例展示了如何使用 `ComputeWorldTransformMatrix` 方法来计算实体的世界变换矩阵：

```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public partial class WorldTransformExampleSystem : SystemBase
{
    protected override void OnCreate()
    {
        var entityManager = World.DefaultGameObjectInjectionWorld.EntityManager;

        // 创建一个简单的层次结构
        Entity parentEntity = entityManager.CreateEntity(typeof(LocalToWorld), typeof(LocalTransform));
        entityManager.SetComponentData(parentEntity, new LocalToWorld() { Value = float4x4.identity });
        entityManager.SetComponentData(parentEntity, LocalTransform.Identity);

        Entity childEntity = entityManager.CreateEntity(typeof(LocalToWorld), typeof(LocalTransform), typeof(Parent));
        entityManager.SetComponentData(childEntity, new Parent() { Value = parentEntity });
        entityManager.SetComponentData(childEntity, new LocalToWorld() { Value = float4x4.identity });
        entityManager.SetComponentData(childEntity, LocalTransform.Identity);

        // 移动父实体
        entityManager.SetComponentData(parentEntity, LocalTransform.FromPosition(1, 2, 3));

        // 此时，由于 ParentSystem 和 LocalToWorldSystem 尚未运行，子实体和父实体的 LocalToWorld 将仍为 identity。
        float4x4 childLocalToWorldMatrix = SystemAPI.GetComponent<LocalToWorld>(childEntity).Value; // 将为 identity

        // 以下内容应该放在 OnCreate() 或类似方法中。在这种情况下，你需要在 OnUpdate() 中调用 Lookup.Update。
        ComponentLookup<LocalTransform> localTransformLookup = SystemAPI.GetComponentLookup<LocalTransform>(true);
        ComponentLookup<Parent> parentLookup = SystemAPI.GetComponentLookup<Parent>(true);
        ComponentLookup<PostTransformMatrix> postTransformLookup = SystemAPI.GetComponentLookup<PostTransformMatrix>(true);

        // 如果你绝对需要在 LocalToWorldSystem 运行之前获取子实体最新的 LocalToWorld，这就是方法。它是一个昂贵的操作，所以请谨慎使用。
        TransformHelpers.ComputeWorldTransformMatrix(childEntity, out childLocalToWorldMatrix, ref localTransformLookup, ref parentLookup,
            ref postTransformLookup);
    }

    protected override void OnUpdate()
    {
        // 在 OnUpdate 中更新查找表
        ComponentLookup<LocalTransform> localTransformLookup = SystemAPI.GetComponentLookup<LocalTransform>(true);
        ComponentLookup<Parent> parentLookup = SystemAPI.GetComponentLookup<Parent>(true);
        ComponentLookup<PostTransformMatrix> postTransformLookup = SystemAPI.GetComponentLookup<PostTransformMatrix>(true);

        Entities
            .ForEach((Entity childEntity, ref LocalToWorld localToWorld) =>
            {
                // 计算并更新子实体的 LocalToWorld 矩阵
                TransformHelpers.ComputeWorldTransformMatrix(childEntity, out float4x4 newLocalToWorldMatrix, ref localTransformLookup, ref parentLookup, ref postTransformLookup);
                localToWorld.Value = newLocalToWorldMatrix;
            })
            .ScheduleParallel();
    }
}
```
