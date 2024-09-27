# EntityManager 概述

`EntityManager` 是一个 API，提供了创建、读取、更新和销毁项目中的实体的实用方法。

### 实体管理器 (EntityManager)

每个世界（World）都有一个 `EntityManager`，你可以使用它来管理该世界中的所有实体。在可能的情况下，最佳实践是使用 `SystemAPI` 中的方法来访问世界中的实体数据，而不是直接使用 `EntityManager`。然而，在主线程上进行结构性更改时，`EntityManager` 非常有用。

#### 结构性更改

`EntityManager` API 中的一些操作会导致结构性更改。要执行结构性更改，`EntityManager` 会等待所有正在运行的作业完成，这会创建一个同步点。这个同步点会阻塞主线程，并阻止你的应用程序充分利用所有 CPU 内核，这可能会导致性能问题。

作为替代方案，你可以使用实体命令缓冲区 (ECB) 来排队结构性更改，以便在某个时刻一起执行。然而，ECBs 也有其自身的性能考虑。

#### `EntityManager` 与 ECB 的主要区别

* 如果要在主线程上立即执行结构性更改，请使用 `EntityManager`。这比使用 ECB 更高效。
* 不能在作业中使用 `EntityManager`，因此它与 `IJobChunk` 和 `IJobEntity` 等作业类型不兼容。可以在作业中使用 ECB 排队结构性更改，但必须在作业完成后在主线程上执行这些更改。更多信息请参阅 [调度数据更改的方式](https://docs.unity3d.com/Manual/system-update-order.html)。
* 在 `SystemAPI.Query` 中只能使用 `CreateEntity`、`CreateArchetype` 和 `Instantiate`。如果想在 `SystemAPI.Query` 中添加组件，需要使用 `EntityCommandBuffer.AddComponent`。

### 关键的 `EntityManager` 方法

世界中的实体通过世界的 `EntityManager` 创建、销毁和修改。关键的 `EntityManager` 方法包括：

| 方法                   | 描述                            |
| -------------------- | ----------------------------- |
| `CreateEntity`       | 创建一个新实体。                      |
| `Instantiate`        | 使用现有实体的所有组件副本创建一个新实体。         |
| `DestroyEntity`      | 销毁一个现有实体。                     |
| `AddComponent<T>`    | 向现有实体添加一个类型为 `T` 的组件。         |
| `RemoveComponent<T>` | 从现有实体移除一个类型为 `T` 的组件。         |
| `HasComponent<T>`    | 如果实体具有类型为 `T` 的组件，则返回 `true`。 |

以上所有方法都是结构性更改操作。

### 示例代码

#### 使用 `EntityManager` 创建实体并添加组件

```csharp
public void CreateAndModifyEntities(EntityManager entityManager)
{
    // 创建一个新实体
    Entity newEntity = entityManager.CreateEntity();

    // 向新实体添加一个组件
    entityManager.AddComponent<FooComp>(newEntity);

    // 检查实体是否具有某个组件
    bool hasFooComp = entityManager.HasComponent<FooComp>(newEntity);
    
    // 移除组件
    if (hasFooComp)
    {
        entityManager.RemoveComponent<FooComp>(newEntity);
    }

    // 销毁实体
    entityManager.DestroyEntity(newEntity);
}
```

使用 ECB 在作业中排队结构性更改

```csharp
public struct MyJob : IJobEntity
{
    public EntityCommandBuffer.ParallelWriter ecb;

    public void Execute(Entity entity, [EntityInQueryIndex] int index)
    {
        // 在作业中排队添加组件
        ecb.AddComponent<BarComp>(index, entity);
    }
}

public void ScheduleJobWithECB(EntityCommandBuffer ecb)
{
    var job = new MyJob
    {
        ecb = ecb.AsParallelWriter()
    };

    // 调度作业
    job.Schedule();
}

```

