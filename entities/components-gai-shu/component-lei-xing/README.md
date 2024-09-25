# Component 类型

为了满足各种使用场景，ECS 组件有多种类型。本节文档描述了 ECS 组件类型、它们的使用案例和性能考虑，以及如何创建它们。

| Topic                 | 描述                  |
| --------------------- | ------------------- |
| Unmanaged components  | 理解非托管组件及其使用方法。      |
| Managed components    | 理解托管组件及其使用方法。       |
| Shared components     | 理解共享组件及其使用方法。       |
| Cleanup components    | 理解清理组件及其使用方法。       |
| Tag components        | 理解标签组件及其使用方法。       |
| Buffer components     | 理解缓冲区组件及其使用方法。      |
| Chunk components      | 理解区块组件及其使用方法。       |
| Enableable components | 理解可启用组件及其使用方法。      |
| Singleton components  | 理解单例组件（即只有一个实例的组件）。 |

### 编辑器中的组件类型

在编辑器中，以下图标表示不同类型的组件。这些图标出现在相关的 Entities 窗口和检查器中。

| 图标 | 组件类型  |
| -- | ----- |
|    | 托管组件  |
|    | 共享组件  |
|    | 标签组件  |
|    | 缓冲区组件 |
|    | 区块组件  |
