# 调度数据更改的方法 (Ways to Schedule Data Changes)

实体组件系统（ECS）中有以下几种方法可以管理项目中的结构性变化：

### 实体命令缓冲区 (Entity Command Buffers, ECB)

使用 `EntityCommandBuffer` 来记录和执行一系列实体操作。

### EntityManager 的方法

直接使用 `EntityManager` 方法来进行结构性变化。

#### 两者的区别如下：

1. **如果你想从作业队列中排队结构性更改，你必须使用 ECB。**
2. **如果你想在主线程上执行结构性更改，并希望它们立即生效，请使用 `EntityManager` 中的方法。**
3. **如果你想在主线程上执行结构性更改，并希望它们在稍后时间（例如作业完成后）生效，请使用 ECB。**

#### 使用注意事项

* 只有在主线程上调用 `Playback` 时，ECB 中记录的更改才会被应用。
* 如果在 `Playback` 之后尝试对 ECB 记录进一步的更改，Unity 会抛出异常。

### 示例

以下是如何使用这两种方法的示例：

#### 使用实体命令缓冲区 (ECB)

```csharp
using Unity.Entities;
using Unity.Collections;

public partial class ExampleECBSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 创建一个临时的命令缓冲区
        var ecb = new EntityCommandBuffer(Allocator.TempJob);

        Entities.ForEach((Entity entity, in MyComponent component) =>
        {
            // 在命令缓冲区中记录要对实体执行的操作
            ecb.AddComponent(entity, new AnotherComponent { Value = 10 });

        }).WithoutBurst().Run(); // Managed API calls should run without Burst compilation

        // 在更新结束时播放缓冲区中的所有操作
        ecb.Playback(EntityManager);
        ecb.Dispose();
    }
}
```
