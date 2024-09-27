# 在运行时加载弱引用场景

#### 在运行时加载弱引用场景

Unity 不会自动加载弱引用场景，因此你需要在使用它们之前手动加载。要在运行时加载场景，你必须在 ECS 组件中存储该场景的弱引用。弱引用可以是 `WeakObjectSceneReference` 或 `UntypedWeakReferenceId`。有关如何将场景的弱引用存储在组件中的信息，请参阅弱引用场景。

**使用 WeakObjectSceneReference 在运行时加载场景**

以下代码示例展示了如何使用 `WeakObjectSceneReferences` 从 `ISystem` 加载场景。有关如何将 `WeakObjectSceneReference` 传递给组件的信息，请参阅从检查器弱引用场景。

**示例代码：运行时加载场景**

```csharp
using Unity.Entities;
using Unity.Entities.Content;
using Unity.Loading;
using UnityEngine.SceneManagement;

public struct WeakObjectSceneReferenceData : IComponentData
{
    public bool startedLoad;
    public WeakObjectSceneReference scene;
}

[WorldSystemFilter(WorldSystemFilterFlags.Default | WorldSystemFilterFlags.Editor)]
[UpdateInGroup(typeof(PresentationSystemGroup))]
public partial struct LoadSceneFromWeakObjectReferenceSystem : ISystem
{
    public void OnCreate(ref SystemState state) { }
    public void OnDestroy(ref SystemState state) { }
    public void OnUpdate(ref SystemState state)
    {
        foreach (var sceneData in SystemAPI.Query<RefRW<WeakObjectSceneReferenceData>>())
        {
            if (!sceneData.ValueRO.startedLoad)
            {
                sceneData.ValueRW.scene.LoadAsync(new ContentSceneParameters()
                {
                    loadSceneMode = LoadSceneMode.Additive
                });
                sceneData.ValueRW.startedLoad = true;
            }
        }
    }
}
```

#### 使用 UntypedWeakReferenceId 在运行时加载场景

以下代码示例展示了如何使用 `RuntimeContentManager` API 和 `UntypedWeakReferenceId` 从 `ISystem` 加载场景。有关如何将 `UntypedWeakReferenceId` 传递给组件的信息，请参阅从 C# 脚本弱引用对象。

**示例代码：运行时加载场景**

```csharp
using Unity.Entities;
using Unity.Entities.Content;
using Unity.Entities.Serialization;
using UnityEngine.SceneManagement;

public struct SceneUntypedWeakReferenceIdData : IComponentData
{
    public bool startedLoad;
    public UntypedWeakReferenceId scene;
}

[WorldSystemFilter(WorldSystemFilterFlags.Default | WorldSystemFilterFlags.Editor)]
[UpdateInGroup(typeof(PresentationSystemGroup))]
public partial struct LoadSceneFromUntypedWeakReferenceIdSystem : ISystem
{
    public void OnCreate(ref SystemState state) { }
    public void OnDestroy(ref SystemState state) { }
    public void OnUpdate(ref SystemState state)
    {
        foreach (var sceneData in SystemAPI.Query<RefRW<SceneUntypedWeakReferenceIdData>>())
        {
            if (!sceneData.ValueRO.startedLoad)
            {
                RuntimeContentManager.LoadSceneAsync(sceneData.ValueRO.scene,
                    new Unity.Loading.ContentSceneParameters()
                    {
                        loadSceneMode = LoadSceneMode.Additive
                    });
                sceneData.ValueRW.startedLoad = true;
            }
        }
    }
}
```
