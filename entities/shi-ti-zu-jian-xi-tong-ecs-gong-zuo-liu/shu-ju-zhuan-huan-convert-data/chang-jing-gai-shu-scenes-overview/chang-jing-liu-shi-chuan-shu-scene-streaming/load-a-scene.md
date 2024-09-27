# Load a Scene

## 加载场景

你可以使用子场景或 SceneSystem API 来加载场景。

如果使用子场景，当 `AutoLoadScene` 字段设置为 `true` 时，Unity 会在启用 `SubScene` 组件时流式传输引用的场景。

要在没有 `SubScene` 组件的情况下直接控制流式传输，请使用 SceneSystem 高级 API。用于异步加载场景的静态方法是 `SceneSystem.LoadSceneAsync`。默认情况下，对 `SceneSystem.LoadSceneAsync` 的调用会加载元实体和部分内容的所有内容。对该方法的调用应该发生在系统的 `OnUpdate` 方法中。

所有版本的该方法都需要接收一个参数来唯一标识要加载的场景。你可以使用以下方式之一来标识场景：

* 一个 EntitySceneReference。
* 一个 Hash128 GUID。
* 一个场景元实体。

注意： 构建过程仅检测由 EntitySceneReference 和 SubScene MonoBehaviour 引用的作者场景。构建过程不检测由 GUID 引用的场景，因此它们的实体场景文件将在构建中丢失。因此，应避免使用 GUID 来标识场景。

在播放模式下，无论使用哪种方法引用场景，所有场景始终可用。如果相应的实体场景文件丢失或过时，加载过程中会触发烘焙。

## 使用 EntitySceneReference 加载场景

`SceneSystem.LoadSceneAsync` 在使用 GUID 或 EntitySceneReference 作为参数时返回加载场景的元实体（scene meta Entity）。你可以将此元实体用于后续调用，以引用加载场景，例如卸载和重新加载场景内容。

### 示例：使用 EntitySceneReference 加载场景

使用 `EntitySceneReference` 是在烘焙期间保持对场景引用并在运行时加载它们的推荐方法。以下示例展示了如何在烘焙期间存储对场景的引用：

#### 运行时组件

`SceneSystem` 使用 `EntitySceneReference` 来标识场景。

```csharp
public struct SceneLoader : IComponentData
{
    public EntitySceneReference SceneReference;
}
#if UNITY_EDITOR
public class SceneLoaderAuthoring : MonoBehaviour
{
    public UnityEditor.SceneAsset Scene;

    class Baker : Baker<SceneLoaderAuthoring>
    {
        public override void Bake(SceneLoaderAuthoring authoring)
        {
            var reference = new EntitySceneReference(authoring.Scene);
            var entity = GetEntity(TransformUsageFlags.None);
            AddComponent(entity, new SceneLoader
            {
                SceneReference = reference
            });
        }
    }
}
#endif

```

通过这种方式，你可以在编辑期间指定场景，并在运行时以高效的方式加载这些场景。

以下示例展示了如何在系统中使用存储的引用来加载场景：

```csharp
[RequireMatchingQueriesForUpdate]
public partial class SceneLoaderSystem : SystemBase
{
    private EntityQuery newRequests;

    protected override void OnCreate()
    {
        // 创建一个查询，用于获取所有包含 SceneLoader 组件的数据实体
        newRequests = GetEntityQuery(typeof(SceneLoader));
    }

    protected override void OnUpdate()
    {
        // 获取查询结果中的 SceneLoader 组件数据数组
        var requests = newRequests.ToComponentDataArray<SceneLoader>(Allocator.Temp);

        // 不能使用 foreach 来遍历查询结果，因为 SceneSystem.LoadSceneAsync 会进行结构性更改
        for (int i = 0; i < requests.Length; i += 1)
        {
            // 异步加载场景
            SceneSystem.LoadSceneAsync(World.Unmanaged, requests[i].SceneReference);
        }

        // 释放临时分配的内存
        requests.Dispose();
        
        // 销毁查询结果中的实体，以避免重复加载相同的场景请求
        EntityManager.DestroyEntity(newRequests);
    }
}
```

## 使用 `SceneSystem.LoadSceneAsync` 加载场景

在调用 `SceneSystem.LoadSceneAsync` 时，仅创建场景实体。Unity 使用这个实体来内部控制加载过程的其余部分。

### 场景加载过程

* **初始调用**: 在调用 `SceneSystem.LoadSceneAsync` 期间，只会创建场景实体。
* **后续过程**: 场景头、部分实体及其内容不会在此调用期间加载，而是在几帧之后准备好。

由于 `SceneSystem.LoadSceneAsync` 可以进行结构性更改，这些结构性更改使得我们不能在查询的 `foreach` 循环中调用该函数。

### 加载参数

`LoadParameters` 结构是可选的，默认情况下 `SceneSystem.LoadSceneAsync` 会加载元实体和所有部分内容。

#### `SceneLoadFlags` 枚举

该枚举控制加载，并具有以下字段：

* **DisableAutoLoad**:
  * Unity 创建场景和部分元实体，但不加载部分内容。当场景加载完成后，你可以访问 `ResolvedSectionEntity` 缓冲区以加载单个部分的内容。
  * `LoadParameters.AutoLoad` 属性是用于设置 `DisableAutoLoad` 的辅助属性。
* **BlockOnStreamIn**:
  * Unity 同步执行场景加载。`SceneSystem.LoadSceneAsync` 的调用仅在场景完全加载时返回。
* **NewInstance**:
  * 在世界中创建场景的新副本。此标志用于场景实例化。

### 示例代码

#### 定义 `LoadParameters` 和使用 `SceneSystem.LoadSceneAsync`

```csharp
public partial class SceneLoaderSystem : SystemBase
{
    private EntityQuery newRequests;

    protected override void OnCreate()
    {
        // 创建一个查询，用于获取所有包含 SceneLoader 组件的数据实体
        newRequests = GetEntityQuery(typeof(SceneLoader));
    }

    protected override void OnUpdate()
    {
        // 获取查询结果中的 SceneLoader 组件数据数组
        var requests = newRequests.ToComponentDataArray<SceneLoader>(Allocator.Temp);

        // 不能使用 foreach 来遍历查询结果，因为 SceneSystem.LoadSceneAsync 会进行结构性更改
        for (int i = 0; i < requests.Length; i += 1)
        {
            // 设置加载参数
            LoadParameters loadParams = new LoadParameters
            {
                Flags = SceneLoadFlags.DisableAutoLoad | SceneLoadFlags.BlockOnStreamIn
            };

            // 异步加载场景
            var sceneMetaEntity = SceneSystem.LoadSceneAsync(World.Unmanaged, requests[i].SceneReference, loadParams);

            // 处理 sceneMetaEntity（例如：可以使用场景元实体进行进一步操作）
        }

        // 释放临时分配的内存
        requests.Dispose();
        
        // 销毁查询结果中的实体，以避免重复加载相同的场景请求
        EntityManager.DestroyEntity(newRequests);
    }
}
```

## 卸载场景

要卸载场景，请使用 `SceneSystem.UnloadScene`：

```csharp
var unloadParameters = SceneSystem.UnloadParameters.DestroyMetaEntities;
SceneSystem.UnloadScene(World.Unmanaged, sceneEntity, unloadParameters);
```

你可以使用场景的 GUID 或 EntitySceneReference，而不是场景元实体来调用 `SceneSystem.UnloadScene`，但这有以下缺点：

* 该方法必须搜索表示匹配 GUID 的场景的元实体，这可能会影响性能。
* 如果同一场景的多个实例已加载，则按 GUID 卸载只会卸载一个实例。

### 默认行为

默认情况下，`SceneSystem.UnloadScene` 仅卸载部分内容，但保留场景和部分的元实体。这在稍后将再次加载场景时非常有用，因为预先准备好这些元实体可以加快场景的加载速度。

### 完全卸载

要卸载内容并删除元实体，请使用 `UnloadParameters.DestroyMetaEntities` 调用 `SceneSystem.UnloadScene`。
