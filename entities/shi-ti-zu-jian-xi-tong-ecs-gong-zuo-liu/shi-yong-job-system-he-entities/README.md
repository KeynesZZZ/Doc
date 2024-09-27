# 使用 Job System 和 Entities

##

Entities 包广泛使用 C# 作业系统。在你的系统代码中尽可能使用作业。

### 主线程访问

对于主线程访问，你可以在 `SystemAPI` 中使用 C# 的 `foreach` 遍历查询对象。

### 方便的作业调度

对于方便的作业调度，可以使用 `IJobEntity`。

### 手动控制的作业调度

在特定情况下，或者当你需要手动控制时，可以使用 `IJobChunk` 的 `Schedule()` 和 `ScheduleParallel()` 方法，在主线程外进行数据转换。

### 作业依赖关系管理

当你调度作业时，ECS 会跟踪哪些系统读取和写入了哪些组件。在组件集重叠的情况下，后续系统的 `Dependency` 属性将包括前面系统调度作业的作业句柄。

要了解更多关于系统的信息，请参阅 \[系统概念]\(System concepts)。
