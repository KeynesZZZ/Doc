# 访问数据的方法 (Ways to Access Data)

在系统中访问数据的方式取决于你使用的是托管的 `SystemBase` 系统还是非托管的 `ISystem` 系统。以下是几种在系统中访问数据的方法：

### SystemState

`SystemState` 提供了一些属性和方法，用于在 `ISystem` 系统中访问原始实体状态数据。`SystemBase` 和 `SystemAPI` 本质上都使用 `SystemState` 中的数据。

#### 获取世界信息

可以通过以下属性获取世界相关的信息：

* `state.World`
* `state.WorldUnmanaged`
* `state.WorldUpdateAllocator`
* `state.GlobalSystemVersion`
* `state.EntityManager`

#### 查询系统信息

以下 API 可用于查询系统状态：

| API                        | Description        |
| -------------------------- | ------------------ |
| `state.Dependency`         | 获取或完成系统的依赖项。       |
| `state.CompleteDependency` | 完成系统的依赖项。          |
| `RequireForUpdate`         | 标记系统需要更新。          |
| `RequireAnyForUpdate`      | 标记系统在任何依赖项变化时需要更新。 |
| `ShouldRunSystem`          | 判断系统是否需要运行。        |
| `Enabled`                  | 确定系统何时需要运行。        |
| `state.SystemHandle`       | 获取系统的句柄。           |
| `state.LastSystemVersion`  | 获取系统的版本号。          |
| `state.DebugName`          | 获取系统的调试名称。         |

#### 依赖数据

以下 API 用于获取可以作为系统依赖项的数据：

| API                                   | Description                                              |
| ------------------------------------- | -------------------------------------------------------- |
| `GetEntityQuery`                      | 获取查询。注意：`EntityQueryBuilder.Build` 是获取查询的首选方法，应尽可能使用此方法。 |
| `GetBufferLookup`                     | 获取缓冲区查找。                                                 |
| `GetComponentLookup`                  | 获取组件查找。                                                  |
| `GetEntityStorageInfoLookup`          | 获取实体存储信息查找。                                              |
| `GetComponentTypeHandle`              | 获取组件类型句柄。                                                |
| `GetBufferTypeHandle`                 | 获取缓冲区类型句柄。                                               |
| `GetEntityTypeHandle`                 | 获取实体类型句柄。                                                |
| `GetSharedComponentTypeHandle`        | 获取共享组件类型句柄。                                              |
| `GetDynamicComponentTypeHandle`       | 获取动态组件类型句柄。                                              |
| `GetDynamicSharedComponentTypeHandle` | 获取动态共享组件类型句柄。                                            |

这些方法都会将给定类型添加为依赖项。例如，调用 `state.GetComponentTypeHandle<MyComp>(isReadOnly: true)` 会将 `MyComp` 添加为可读的依赖项。这意味着 `state.Dependency` 包含了所有写入 `MyComp` 的先前系统的 `state.Dependency`。`GetEntityQuery` 对查询中的每个组件具有相同的功能，而查找方法会添加该类型的依赖项。

### SystemBase

`SystemBase` 中包含与 `SystemState` 相同的方法，但它们以 `this.` 而不是 `state.` 为前缀。

### SystemAPI

`SystemAPI` 是一个提供缓存和实用方法的类，用于访问实体世界中的数据。它适用于 `SystemBase` 中的非静态方法以及以 `ref SystemState` 作为参数的 `ISystem` 非静态方法。由于这些方法可以直接在 `Update` 中使用，因此使用 `SystemAPI` 没有运行时成本，所以尽可能使用 `SystemAPI` 来访问数据。

#### 示例

```csharp
using Unity.Entities;

public partial class MySystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 使用 SystemAPI 获取和操作组件数据
        foreach(var (myComponent, entity) in SystemAPI.Query<RefRW<MyComponent>>())
        {
            // 修改组件数据
            myComponent.ValueRW.SomeField += 1;
        }
    }
}
```
