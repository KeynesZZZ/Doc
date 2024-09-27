# 在运行时加载弱引用对象

#### 在运行时加载弱引用对象

Unity 不会自动加载弱引用对象，因此你需要在使用它们之前手动加载。要在运行时加载对象，你必须在 ECS 组件中存储该对象的弱引用。弱引用可以是 `WeakObjectReference` 或 `UntypedWeakReferenceId`。有关如何将对象的弱引用存储在组件中的信息，请参阅弱引用对象。

**使用 WeakObjectReference 在运行时加载对象**

以下代码示例展示了如何使用 `WeakObjectReferences` 从 `ISystem` 加载并渲染带有材质的网格。有关如何将 `WeakObjectReference` 传递给组件的信息，请参阅从检查器弱引用对象。

**示例代码：运行时加载和渲染对象**

```csharp
using Unity.Entities;
using Unity.Entities.Content;
using Unity.Transforms;
using UnityEngine;

public struct WeakObjectReferenceData : IComponentData
{
    public bool startedLoad;
    public WeakObjectReference<Mesh> mesh;
    public WeakObjectReference<Material> material;
}

[WorldSystemFilter(WorldSystemFilterFlags.Default | WorldSystemFilterFlags.Editor)]
[UpdateInGroup(typeof(PresentationSystemGroup))]
public partial struct RenderFromWeakObjectReferenceSystem : ISystem
{
    public void OnCreate(ref SystemState state) { }
    public void OnDestroy(ref SystemState state) { }
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, dec) in SystemAPI.Query<RefRW<LocalToWorld>, RefRW<WeakObjectReferenceData>>())
        {
            if (!dec.ValueRW.startedLoad)
            {
                dec.ValueRW.mesh.LoadAsync();
                dec.ValueRW.material.LoadAsync();
                dec.ValueRW.startedLoad = true;
            }
            if (dec.ValueRW.mesh.LoadingStatus == ObjectLoadingStatus.Completed &&
                dec.ValueRW.material.LoadingStatus == ObjectLoadingStatus.Completed)
            {
                Graphics.DrawMesh(dec.ValueRO.mesh.Result,
                    transform.ValueRO.Value, dec.ValueRO.material.Result, 0);
            }
        }
    }
}
```

#### 使用 UntypedWeakReferenceId 在运行时加载对象

以下代码示例展示了如何使用 `RuntimeContentManager` API 和 `UntypedWeakReferenceIds` 从 `ISystem` 加载并渲染带有材质的网格。有关如何将 `UntypedWeakReferenceId` 传递给组件的信息，请参阅从 C# 脚本弱引用对象。

**示例代码：运行时加载和渲染对象**

```csharp
using Unity.Entities;
using Unity.Entities.Content;
using Unity.Transforms;
using UnityEngine;
using Unity.Entities.Serialization;

public struct ObjectUntypedWeakReferenceIdData : IComponentData
{
    public bool startedLoad;
    public UntypedWeakReferenceId mesh;
    public UntypedWeakReferenceId material;
}

[WorldSystemFilter(WorldSystemFilterFlags.Default | WorldSystemFilterFlags.Editor)]
[UpdateInGroup(typeof(PresentationSystemGroup))]
public partial struct RenderFromUntypedWeakReferenceIdSystem : ISystem
{
    public void OnCreate(ref SystemState state) { }
    public void OnDestroy(ref SystemState state) { }
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, dec) in SystemAPI.Query<RefRW<LocalToWorld>, RefRW<ObjectUntypedWeakReferenceIdData>>())
        {
            if (!dec.ValueRO.startedLoad)
            {
                RuntimeContentManager.LoadObjectAsync(dec.ValueRO.mesh);
                RuntimeContentManager.LoadObjectAsync(dec.ValueRO.material);
                dec.ValueRW.startedLoad = true;
            }
            if (RuntimeContentManager.GetObjectLoadingStatus(dec.ValueRO.mesh) == ObjectLoadingStatus.Completed &&
                RuntimeContentManager.GetObjectLoadingStatus(dec.ValueRO.material) == ObjectLoadingStatus.Completed)
            {
                Mesh mesh = RuntimeContentManager.GetObjectValue<Mesh>(dec.ValueRO.mesh);
                Material material = RuntimeContentManager.GetObjectValue<Material>(dec.ValueRO.material);
                Graphics.DrawMesh(mesh, transform.ValueRO.Value, material, 0);
            }
        }
    }
}
```
