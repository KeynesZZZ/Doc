# SystemBase 概述

要创建一个托管系统，实现抽象类 `SystemBase`。

你必须使用 `OnUpdate` 系统事件回调来添加系统每帧必须执行的工作。`ComponentSystemBase` 命名空间中的所有其他回调方法都是可选的。

所有系统事件都在主线程上运行。最佳实践是使用 `OnUpdate` 方法调度作业以执行大部分工作。要从系统中调度作业，你可以使用以下机制之一：

* **Entities.ForEach**：迭代组件数据。
* **Job.WithCode**：将 lambda 表达式作为单个后台作业执行。
* **IJobEntity**：在多个系统中迭代组件数据。
* **IJobChunk**：按原型块迭代数据。

以下示例说明了如何使用 `Entities.ForEach` 实现一个根据另一个组件的值更新组件的系统：

```csharp
public struct Position : IComponentData
{
    public float3 Value;
}

public struct Velocity : IComponentData
{
    public float3 Value;
}

[RequireMatchingQueriesForUpdate]
public partial class ECSSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 在 ForEach 中捕获的局部变量
        float dT = SystemAPI.Time.DeltaTime;

        Entities
            .WithName("Update_Displacement")
            .ForEach(
                (ref Position position, in Velocity velocity) =>
                {
                    position = new Position()
                    {
                        Value = position.Value + velocity.Value * dT
                    };
                }
            )
            .ScheduleParallel();
    }
}
```

### 回调方法顺序

`SystemBase` 中有几个回调方法，Unity 在系统创建过程中会在各种情况下调用这些方法，你可以使用它们来调度系统每帧必须执行的工作：

* **OnCreate**：在系统创建时调用。
* **OnStartRunning**：在首次调用 `OnUpdate` 之前以及系统恢复运行时调用。
* **OnUpdate**：只要系统有工作要做，每帧都会调用。有关系统何时有工作要做的详细信息，请参见 `ShouldRunSystem`。
* **OnStopRunning**：在 `OnDestroy` 之前调用。当系统停止运行时也会调用，如果没有实体匹配系统的 `RequireForUpdate`，或者系统的 `Enabled` 属性设置为 `false`。请注意，如果没有指定 `RequireForUpdate`，系统会持续运行，除非被禁用或销毁。
* **OnDestroy**：在系统销毁时调用。

以下图示说明了系统的事件顺序：

<figure><img src="../../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

父系统组的 `OnUpdate` 方法触发其组内所有系统的 `OnUpdate` 方法。有关系统如何更新的更多信息，请参见 Update order of systems（系统的更新顺序）。
