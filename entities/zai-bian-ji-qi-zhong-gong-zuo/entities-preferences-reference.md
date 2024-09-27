# Entities Preferences Reference

#### Entities Preferences Reference

在编辑器的 Preferences 窗口中包含了一些特定于实体系统的设置，如下所示：

**Baking**

| 属性                      | 功能                                                     |
| ----------------------- | ------------------------------------------------------ |
| **Scene View Mode**     | 选择场景视图的数据模式。可以选择 Authoring Data 或 Runtime Data。        |
| **Live Baking Logging** | 启用此属性以输出实时烘焙触发器的日志。这有助于诊断导致烘焙发生的问题。                    |
| **Clear Entity cache**  | 强制 Unity 在下次加载 Sub Scenes 时或构建独立播放器时重新烘焙所有 Sub Scenes。 |

**Advanced**

| 属性                       | 功能                                                                           |
| ------------------------ | ---------------------------------------------------------------------------- |
| **Show Advanced Worlds** | 启用此属性以在不同的世界下拉菜单中显示高级世界。高级世界是支持主要世界的专业化世界，如 Staging world 或 Streaming world。 |

**Hierarchy Window**

| 属性                                                      | 功能                                                                     |
| ------------------------------------------------------- | ---------------------------------------------------------------------- |
| **Update Mode**                                         | 设置如何更新 Entities 层次结构窗口。                                                |
|                                                         | - **Synchronous**: 以阻塞方式更新层次结构。数据始终是最新的，但可能影响性能。                       |
|                                                         | - **Asynchronous**: 以非阻塞方式更新层次结构，如果需要，可以跨多个帧进行更新。数据可能会滞后几帧，但对性能的影响最小化。 |
| **Minimum Milliseconds Between Hierarchy Update Cycle** | 设置层次更新周期之间等待的最少时间（单位：毫秒）。增加此值以减少更新频率，从而降低对性能的影响。                       |
| **Exclude Unnamed Nodes For Search**                    | 启用此属性以在按字符串搜索的结果中排除未命名的实体。如果存在大量未命名的实体，这可以加快搜索速度。                      |

**Journaling**

| 属性                  | 功能                                                                                                  |
| ------------------- | --------------------------------------------------------------------------------------------------- |
| **Enabled**         | 启用此属性以启用 Journaling 数据记录。                                                                           |
| **Total Memory MB** | 设置分配用于存储 Journaling 记录数据的内存大小（单位：MB）。一旦满了，新记录将覆盖旧记录。                                                |
| **Post Process**    | 启用此属性以在 Journaling 窗口中对 Journaling 数据进行后处理。这包括在可能的情况下将 GetComponentDataRW 转换为 SetComponentData 等操作。 |

**Systems Window**

| 属性                                          | 功能                                                                    |
| ------------------------------------------- | --------------------------------------------------------------------- |
| **Show 0s in Entity Count And Time Column** | 启用此属性以在系统不匹配任何实体时，在实体计数列中显示 0。如果禁用此属性，当系统不匹配任何实体时，Unity 将在实体计数列中显示为空。 |
| **Show More Precision For Running Time**    | 启用此属性以将系统运行时间的精度从 2 位小数增加到 4 位小数。                                     |

通过这些设置，开发者可以自定义和优化 Unity 实体组件系统（ECS）的开发体验，从而提升开发效率和产品质量。
