# Native container component支持

在编写游戏和引擎代码时，通常需要维护和更新单个长期存在的数据结构，这些数据结构可能会被多个系统使用。我们发现将这些容器放在单例组件（Singleton components）上是很方便的。

#### 安全性限制

当将原生容器放在组件上时，为确保安全性，我们对该组件施加了一些限制。特别地，不允许使用 `IJobChunk` 或 `IJobEntity` 调度针对这些组件的作业。这是因为这些作业已经通过容器访问了这些组件，而作业安全系统并不会扫描嵌套容器，因为那样会使作业调度时间过长。

#### 使用原生容器调度作业

然而，允许调度针对容器本身的作业，因此可以在主线程上获取组件，提取容器，并调度针对容器本身的作业。Singleton 函数专为此种情况设计，避免不必要的依赖完成操作，从而可以跨多个系统链式调度针对同一容器的作业，而不会创建同步点。

#### 示例代码

以下示例展示了如何在单例组件上使用原生容器，并调度作业以操作该容器：

```csharp
using Unity.Collections;
using Unity.Entities;
using Unity.Jobs;
using Unity.Mathematics;

// 定义一个包含原生容器的单例组件
public struct SingletonContainerComponent : IComponentData
{
    public NativeArray<float3> Positions;
}

public partial struct InitializeSingletonContainerSystem : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        var entity = state.EntityManager.CreateEntity(typeof(SingletonContainerComponent));
        var positions = new NativeArray<float3>(100, Allocator.Persistent);
        state.EntityManager.SetComponentData(entity, new SingletonContainerComponent { Positions = positions });
    }

    public void OnDestroy(ref SystemState state)
    {
        // 确保在销毁系统时释放原生容器
        var query = state.GetEntityQuery(ComponentType.ReadOnly<SingletonContainerComponent>());
        if (query.CalculateEntityCount() > 0)
        {
            var container = SystemAPI.GetSingleton<SingletonContainerComponent>();
            container.Positions.Dispose();
        }
    }
}

// 一个简单的作业，用于操作原生容器中的数据
public struct UpdatePositionsJob : IJobParallelFor
{
    public NativeArray<float3> Positions;

    public void Execute(int index)
    {
        // 修改每个位置数据
        Positions[index] = new float3(index, index, index);
    }
}

public partial struct UpdateSingletonContainerSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // 获取单例组件
        var container = SystemAPI.GetSingleton<SingletonContainerComponent>();

        // 创建并调度作业
        var job = new UpdatePositionsJob
        {
            Positions = container.Positions
        };
        state.Dependency = job.Schedule(container.Positions.Length, 64, state.Dependency);
    }
}
```
