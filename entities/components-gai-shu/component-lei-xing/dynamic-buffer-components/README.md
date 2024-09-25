# Dynamic buffer components

动态缓冲组件是作为可调整大小数组的组件。

| Topic                                              | 描述                                       |
| -------------------------------------------------- | ---------------------------------------- |
| Introducing dynamic buffer components              | 介绍动态缓冲组件：理解动态缓冲组件及其使用场景。                 |
| Create a dynamic buffer component                  | 创建一个新的动态缓冲组件以在你的应用程序中使用。                 |
| Access all dynamic buffers in a chunk              | 使用 `BufferAccessor<T>` 获取块中特定类型的所有动态缓冲区。 |
| Reuse a dynamic buffer for multiple entities       | 在主线程上访问动态缓冲区并将其数据用于多个实体。                 |
| Access dynamic buffers from jobs                   | 创建 `BufferLookup` 查找，在不位于主线程时访问动态缓冲区。    |
| Modify dynamic buffers with an EntityCommandBuffer | 使用 `EntityCommandBuffer` 推迟对动态缓冲区的修改。    |
| Reinterpret a dynamic buffer                       | 将动态缓冲区的内容重新解释为另一种类型。                     |

### 介绍动态缓冲组件 (Introducing dynamic buffer components)

动态缓冲组件用于存储可调整大小的数据数组，非常适合需要频繁更新和扩展的数据集合。它们在实体组件系统 (ECS) 中提供了灵活的数据结构，能够高效地处理大量数据。

