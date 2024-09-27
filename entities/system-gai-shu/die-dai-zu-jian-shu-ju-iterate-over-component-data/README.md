# 迭代组件数据（Iterate over component data）

在创建系统时，迭代数据是最常见的任务之一。一个系统通常处理一组实体，从一个或多个组件中读取数据，进行计算，然后将结果写入另一个组件。

最有效的迭代实体和组件的方法是在作业中按顺序处理组件。这可以利用所有可用核心的处理能力，并通过数据局部性来避免 CPU 缓存未命中。

本节解释了以下几种迭代实体数据的方法：

| 主题                                      | 描述                                          |
| --------------------------------------- | ------------------------------------------- |
| 使用 `SystemAPI.Query` 迭代数据               | 在主线程上迭代数据集合。                                |
| 使用 `IJobEntity` 迭代数据                    | 编写一次代码并使用 `IJobEntity` 创建多个调度。              |
| 迭代数据块                                   | 使用 `IJobChunk` 迭代包含匹配实体的原型块。                |
| 手动迭代数据                                  | 手动迭代实体或原型块。                                 |
| 在 `SystemBase` 系统中使用 `Entities.ForEach` | 在 `SystemBase` 中使用 `Entities.ForEach` 迭代实体。 |

### 使用 `SystemAPI.Query` 迭代数据

`SystemAPI.Query` 提供了一种在主线程上迭代数据的方法。

```csharp
public partial struct MySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (foo, entity) in SystemAPI.Query<FooComp>().WithEntityAccess())
        {
            // 处理 foo 数据
        }
    }
}
```
