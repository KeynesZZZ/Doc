# 单例组件（Singleton components ）

单例组件是指在给定世界中只有一个实例的组件。例如，如果一个世界中只有一个实体具有类型为 `T` 的组件，那么 `T` 就是一个单例组件。

如果将单例组件添加到另一个实体，则它不再是单例组件。此外，单例组件可以存在于另一个世界中，而不影响其单例状态。

#### 单例组件 API

Entities 包含多个用于处理单例组件的 API：

| 命名空间              | 方法                    |
| ----------------- | --------------------- |
| **EntityManager** | CreateSingleton       |
|                   | GetSingletonEntity    |
|                   | GetSingleton          |
|                   | GetSingletonRW        |
|                   | TryGetSingleton       |
|                   | HasSingleton          |
|                   | TryGetSingletonBuffer |
|                   | TryGetSingletonEntity |
|                   | GetSingletonBuffer    |
|                   | SetSingleton          |
| **SystemAPI**     | GetSingletonEntity    |
|                   | GetSingleton          |
|                   | GetSingletonRW        |
|                   | TryGetSingleton       |
|                   | HasSingleton          |
|                   | TryGetSingletonBuffer |
|                   | TryGetSingletonEntity |
|                   | GetSingletonBuffer    |
|                   | SetSingleton          |

当您知道某个组件只有一个实例时，使用单例组件 API 是非常有用的。例如，如果您有一个单人游戏应用程序且只需要一个 `PlayerController` 组件实例，您可以使用单例 API 来简化代码。此外，在基于服务器的架构中，客户端实现通常只跟踪其实例的时间戳，因此单例 API 方便且简化了大量手动编写的代码。

#### 依赖完成

在系统代码中，单例组件在依赖完成方面具有特殊情况。对于正常的组件访问，诸如 `EntityManager.GetComponentData` 或 `SystemAPI.GetComponent` 等 API 会确保在返回请求的数据之前，任何可能在工作线程上写入相同组件数据的正在运行的作业完成。

然而，单例 API 调用并不能确保首先完成正在运行的作业。Jobs Debugger 在无效访问时会记录错误，您需要手动使用 `EntityManager.CompleteDependencyBeforeRO` 或 `EntityManager.CompleteDependencyBeforeRW` 完成依赖，或者需要重构数据依赖。

在使用 `GetSingletonRW` 获取读/写访问权限时也要小心。因为返回了组件数据的引用，所以可能在作业读取或写入数据时修改数据。`GetSingletonRW` 的最佳实践是：

* 仅用于访问组件中的 `NativeContainer`。这是因为本地容器具有与 Jobs Debugger 兼容的安全机制，与 ECS 组件安全机制分开。
* 检查 Jobs Debugger 是否有错误。任何错误都表示需要重构或手动完成的依赖问题。
