# 场景实例化

##

要在一个世界中创建同一场景的多个实例，可以使用 `SceneSystem.LoadSceneAsync` 并带有标志 `SceneLoadFlags.NewInstance`。这对于例如有不同瓷砖（每个瓷砖由一个场景表示）并且你想用这些瓷砖填充世界的情况非常有用。

当你以这种方式创建一个场景时，从加载调用返回的场景元实体将引用新创建的实例。

### 场景实例化步骤概述

以下是实例化场景并对其应用唯一变换的步骤：

1. 使用 `SceneSystem.LoadSceneAsync` 和标志 `SceneLoadFlags.NewInstance` 加载场景。
2. 存储返回的场景元实体以在后续步骤中使用。
3. 向场景元实体添加 `PostLoadCommandBuffer`：
   * 创建一个 `EntityCommandBuffer`。
   * 使用该 `EntityCommandBuffer` 创建一个新实体。向其中添加具有唯一实例信息的组件。
   * 创建一个 `PostLoadCommandBuffer` 并存储 `EntityCommandBuffer`。
   * 将 `PostLoadCommandBuffer` 组件添加到场景元实体。不要释放 `EntityCommandBuffer`。
4. 编写一个系统来对实例化的场景应用唯一变换。
   * 创建系统并将其分配到 `ProcessAfterLoadGroup`。
   * 从 `EntityCommandBuffer` 中创建的实体查询实例信息。
   * 使用该信息将变换应用于实例中的实体。

#### 示例代码：实例化场景并应用偏移

**定义组件**

```csharp
public struct PostLoadOffset : IComponentData
{
    public float3 Offset;
}

var loadParameters = new SceneSystem.LoadParameters()
{
    Flags = SceneLoadFlags.NewInstance
};
var sceneEntity = SceneSystem.LoadSceneAsync(state.WorldUnmanaged, sceneReference, loadParameters);

var ecb = new EntityCommandBuffer(Allocator.Persistent, PlaybackPolicy.MultiPlayback);
var postLoadEntity = ecb.CreateEntity();
var postLoadOffset = new PostLoadOffset
{
    Offset = sceneOffset
};
ecb.AddComponent(postLoadEntity, postLoadOffset);

var postLoadCommandBuffer = new PostLoadCommandBuffer()
{
    CommandBuffer = ecb
};
state.EntityManager.AddComponentData(sceneEntity, postLoadCommandBuffer);

```

上述代码使用了一个名为 PostLoadOffset 的组件来存储要应用于实例的偏移量。

```csharp
[UpdateInGroup(typeof(ProcessAfterLoadGroup))]
public partial struct PostprocessSystem : ISystem
{
    private EntityQuery offsetQuery;

    public void OnCreate(ref SystemState state)
    {
        offsetQuery = new EntityQueryBuilder(Allocator.Temp)
            .WithAll<PostLoadOffset>()
            .Build(ref state);
        state.RequireForUpdate(offsetQuery);
    }

    public void OnUpdate(ref SystemState state)
    {
        // 从 EntityCommandBuffer 创建的实体中查询实例信息。
        var offsets = offsetQuery.ToComponentDataArray<PostLoadOffset>(Allocator.Temp);
        foreach (var offset in offsets)
        {
            // 使用该信息将变换应用于实例中的实体。
            foreach (var transform in SystemAPI.Query<RefRW<LocalTransform>>())
            {
                transform.ValueRW.Position += offset.Offset;
            }
        }
        state.EntityManager.DestroyEntity(offsetQuery);
    }
}

```
