# 系统比较



要创建系统，你可以使用 `ISystem` 或 `SystemBase`。`ISystem` 提供对非托管内存的访问，而 `SystemBase` 则适用于存储托管数据。这两种系统类型都可以与所有的 Entities 包和 Job 系统一起使用。以下概述了这两种系统类型的差异。

#### 系统之间的差异

`ISystem` 兼容 Burst，比 `SystemBase` 更快，并且具有基于值的表示。一般来说，为了获得更好的性能优势，你应该优先使用 `ISystem` 而不是 `SystemBase`。然而，`SystemBase` 有一些方便的功能，但需要使用垃圾回收分配或增加 SourceGen 编译时间作为代价。

下表概述了它们的兼容性：

| Feature (特性)                                                                                                              | ISystem compatibility (兼容性) | SystemBase compatibility (兼容性) |
| ------------------------------------------------------------------------------------------------------------------------- | --------------------------- | ------------------------------ |
| <p><strong>Burst compile OnCreate, OnUpdate, and OnDestroy</strong><br>（在 OnCreate、OnUpdate 和 OnDestroy 上进行 Burst 编译）</p> | Yes                         | No                             |
| <p><strong>Unmanaged memory allocated</strong><br>（分配非托管内存）</p>                                                           | Yes                         | No                             |
| <p><strong>GC allocated</strong><br>（分配 GC 内存）</p>                                                                        | No                          | Yes                            |
| <p><strong>Can store managed data directly in system type</strong><br>（可以直接在系统类型中存储托管数据）</p>                              | No                          | Yes                            |
| <p><strong>Idiomatic foreach</strong><br>（惯用的 foreach 迭代）</p>                                                             | Yes                         | Yes                            |
| <p><strong>Entities.ForEach</strong><br>（实体验证 ForEach）</p>                                                                | No                          | Yes                            |
| <p><strong>Job.WithCode</strong><br>（作业 WithCode）</p>                                                                     | No                          | Yes                            |
| <p><strong>IJobEntity</strong><br>（实体作业接口）</p>                                                                            | Yes                         | Yes                            |
| <p><strong>IJobChunk</strong><br>（块作业接口）</p>                                                                              | Yes                         | Yes                            |
| <p><strong>Supports inheritance</strong><br>（支持继承）</p>                                                                    | No                          | Yes                            |

#### 多个系统实例

你可以在运行时手动创建同一系统类型的多个实例，并跟踪每个实例的 `SystemHandle`。然而，一般的 API（如 `GetExistingSystem` 和 `GetOrCreateSystem`）不支持多个系统实例。

你可以使用 `CreateSystem` API 创建运行时系统。
