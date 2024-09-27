# 实体命令缓冲区概述 (Entity Command Buffer Overview)

实体命令缓冲区（ECB）存储一队列线程安全的命令，你可以向其中添加命令，并在稍后回放这些命令。你可以使用 ECB 从作业中调度结构性更改，并在作业完成后在主线程上执行这些更改。你也可以在主线程上使用 ECB 来延迟更改，或者多次回放一组更改。

### ECB 方法

`EntityCommandBuffer` 中的方法记录命令，这些命令与 `EntityManager` 中的方法类似。例如：

* **CreateEntity(EntityArchetype)**: 注册一个创建具有指定原型的新实体的命令。
* **DestroyEntity(Entity)**: 注册一个销毁实体的命令。
* **SetComponent(Entity, T)**: 注册一个设置实体上类型为 T 的组件的值的命令。
* **AddComponent(Entity)**: 注册一个向实体添加类型为 T 的组件的命令。
* **RemoveComponent(EntityQuery)**: 注册一个从所有匹配查询的实体中移除类型为 T 的组件的命令。

### 命令缓冲区创建的临时实体

由 `EntityCommandBuffer` 的 `CreateEntity()` 和 `Instantiate()` 方法返回的实体是特殊的；它们在缓冲区回放之前并不完全存在，但仍然可以在同一缓冲区内的后续命令中使用。这些临时实体有两个有效的用途：

1. **命令可以针对同一缓冲区中早期命令创建的临时实体**。例如，在同一缓冲区中先前创建的临时实体 e 上调用 `EntityCommandBuffer.AddComponent<T>(e)` 是有效的。
2. **传递给命令的非托管组件值可以包含对临时实体的引用**。例如，如果 e 或 e2（或两者）是来自同一命令缓冲区的临时实体，则 `EntityCommandBuffer.SetComponent(e2, new Parent{ Value = e})` 是有效的。这包括包含实体字段的 `IBufferElementData` 组件（动态缓冲区）。

在命令缓冲区回放期间，缓冲区中先前创建的临时实体的有效引用将自动替换为对应的“真实”实体的引用。无法在回放命令缓冲区后确定给定临时实体对应的“真实”实体。

将临时实体传递给 `EntityManager` 方法，或从一个命令缓冲区引用另一个命令缓冲区中的临时实体是无效的。这两种情况都会引发异常。

### 实体命令缓冲区的安全性 (Entity Command Buffer Safety)

`EntityCommandBuffer` 有一个作业安全句柄，类似于本地容器。这种安全性仅在 Unity 编辑器中可用，在播放器构建中不可用。如果你尝试在使用 ECB 的未完成计划作业上执行以下操作，安全检查会抛出异常：

* 通过其 `AddComponent`、`Playback`、`Dispose` 或其他方法访问 `EntityCommandBuffer`。
* 调度另一个访问相同 `EntityCommandBuffer` 的作业，除非新作业依赖于已调度的作业。

#### 注意事项

最佳实践是为每个不同的作业使用单独的 ECB。这是因为如果你在连续作业中重用 ECB，这些作业可能会使用重叠的排序键集（例如，两者都使用 `ChunkIndexInQuery`），并且作业记录的命令可能会交错。
