# ISystem 概述

要创建一个非托管系统，实现接口类型 `ISystem`。

#### 实现抽象方法

你必须实现以下抽象方法，这些方法可以进行 Burst 编译：

| 方法 (Method)   | 说明 (Description)      |
| ------------- | --------------------- |
| **OnCreate**  | 系统事件回调，在使用前初始化系统及其数据。 |
| **OnUpdate**  | 系统事件回调，添加系统每帧必须执行的工作。 |
| **OnDestroy** | 系统事件回调，在销毁前清理资源。      |

`ISystem` 系统不是通过基类继承的，如同 `SystemBase` 系统。相反，每个 `OnCreate`、`OnUpdate` 和 `OnDestroy` 方法都有一个 `ref SystemState` 参数，你可以使用它来访问 `World`、`WorldUnmanaged` 或上下文 `World` 数据以及 API，例如 `EntityManager`。

#### 可选实现 ISystemStartStop

你也可以选择性地实现接口 `ISystemStartStop`，它提供以下回调：

| 方法 (Method)        | 说明 (Description)                            |
| ------------------ | ------------------------------------------- |
| **OnStartRunning** | 在第一次调用 `OnUpdate` 之前，以及系统在停止或禁用后恢复时的系统事件回调。 |
| **OnStopRunning**  | 系统被禁用或不再匹配系统更新所需的任何组件时的系统事件回调。              |

#### 调度作业

所有系统事件都在主线程上运行。最佳实践是使用 `OnUpdate` 方法调度作业以执行大部分工作。要从系统中调度作业，请使用以下之一：

* **IJobEntity**：在多个实体中迭代组件数据，可以跨系统重用。
* **IJobChunk**：按原型块迭代数据。

#### 回调方法顺序

`ISystem` 中有几个回调方法，Unity 在系统创建过程中会在各种情况下调用这些方法，你可以使用它们来调度系统每帧必须执行的工作：

* **OnCreate**：在 ECS 创建系统时调用。
* **OnStartRunning**：在首次调用 `OnUpdate` 之前以及系统恢复运行时调用。
* **OnUpdate**：只要系统有工作要做，每帧都会调用。有关系统何时有工作要做的详细信息，请参见 `ShouldRunSystem`。
* **OnStopRunning**：在 `OnDestroy` 之前调用。当系统停止运行时也会调用，如果没有实体匹配系统的 `RequireForUpdate`，或者你将系统的 `Enabled` 属性设置为 `false`。如果没有指定 `RequireForUpdate`，系统会持续运行，除非被禁用或销毁。
* **OnDestroy**：在 ECS 销毁系统时调用。

下图说明了系统的事件顺序：

<figure><img src="../../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

父系统组的 `OnUpdate` 方法触发其组内所有系统的 `OnUpdate` 方法。有关系统如何更新的更多信息，请参见 Update order of systems（系统的更新顺序）。
