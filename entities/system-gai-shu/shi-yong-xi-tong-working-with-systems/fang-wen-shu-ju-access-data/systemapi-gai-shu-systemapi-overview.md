# SystemAPI 概述 (SystemAPI Overview)

`SystemAPI` 是一个提供缓存和实用方法的类，用于访问实体世界中的数据。它适用于 `SystemBase` 中的非静态方法以及以 `ref SystemState` 作为参数的 `ISystem` 非静态方法。

### 功能

你可以使用 `SystemAPI` 执行以下操作：

* **遍历数据**：根据查询条件检索每个匹配实体的数据。
* **构建查询**：获取缓存的 `EntityQuery`，可用于调度作业或检索有关该查询的信息。
* **访问数据**：获取组件数据、缓冲区和实体存储信息。
* **访问单例**：查找数据的单实例，也称为单例。

所有 `SystemAPI` 方法直接映射到放置它们的系统。这意味着诸如 `SystemAPI.GetSingleton<T>()` 之类的调用会检查系统包含的世界是否能够执行该操作。

### 实现机制

`SystemAPI` 使用存根方法，这意味着它们都直接调用 `ThrowCodeGenException`。这是因为 `SystemAPI` 使用 Roslyn 源生成器替换了这些方法的正确查找。因此，你不能在不支持的上下文中调用 `SystemAPI`。

要检查你的系统，请使用支持源生成的 IDE，例如 Visual Studio 2022+ 或 Rider 2021.3.3+。然后你可以选择“转到定义”来检查使用 `SystemAPI` 的系统上生成的代码。这说明为什么需要将系统标记为 `partial`。

#### 遍历数据 (Iterate Through Data)

要在主线程上遍历数据集合，可以在 `ISystem` 和 `SystemBase` 系统类型中使用 `Query` 方法。它使用 C# 传统的 `foreach` 语法。更多信息，请参阅 SystemAPI.Query 概述。

**示例**

```csharp
using Unity.Entities;

public partial class MySystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 使用 SystemAPI.Query 遍历数据
        foreach(var (myComponent, entity) in SystemAPI.Query<RefRW<MyComponent>>())
        {
            // 修改组件数据
            myComponent.ValueRW.SomeField += 1;
        }
    }
}
```

#### 构建查询 (Query Building)&#x20;

QueryBuilder 方法获取一个 EntityQuery，你可以使用它来调度作业或检索有关查询的信息。它遵循与 EntityQueryBuilder 相同的语法。

使用 SystemAPI.QueryBuilder 的好处是该方法会缓存数据。以下示例展示了如何完全编译 SystemAPI 调用：

```csharp
/// SystemAPI 调用
SystemAPI.QueryBuilder().WithAll<HealthData>().Build();

/// ECS 编译如下：
EntityQuery query;
public void OnCreate(ref SystemState state){
    query = new EntityQueryBuilder(state.WorldUpdateAllocator).WithAll<HealthData>().Build(ref state);
}

public void OnUpdate(ref SystemState state){
    // 在更新中使用查询
    query;
}

```

## 访问数据 (Access Data)

`SystemAPI` 包含以下实用方法，可用于访问系统世界中的数据：

### 数据类型与 API 对应关系

| Data Type | API                            |
| --------- | ------------------------------ |
| **组件数据**  | `GetComponentLookup`           |
|           | `GetComponent`                 |
|           | `SetComponent`                 |
|           | `HasComponent`                 |
|           | `IsComponentEnabled`           |
|           | `SetComponentEnabled`          |
| **缓冲区**   | `GetBufferLookup`              |
|           | `GetBuffer`                    |
|           | `HasBuffer`                    |
|           | `IsBufferEnabled`              |
|           | `SetBufferEnabled`             |
| **实体信息**  | `GetEntityStorageInfoLookup`   |
|           | `Exists`                       |
| **方面**    | `GetAspect`                    |
| **句柄**    | `GetEntityTypeHandle`          |
|           | `GetComponentTypeHandle`       |
|           | `GetBufferTypeHandle`          |
|           | `GetSharedComponentTypeHandle` |

### 缓存与同步机制

这些 `SystemAPI` 方法在系统的 `OnCreate` 中缓存，并在任何调用前调用 `.Update`。此外，当你调用这些方法时，ECS 确保调用在获取查找访问之前已同步。这意味着诸如 `SystemAPI.SetBuffer<MyElement>` 这样的调用会导致所有当前写入 `MyElement` 的作业完成，而像 `GetEntityTypeHandle` 和 `GetBufferLookup` 这样的调用不会引起同步。

这种机制在将数据传递给诸如 `IJobEntity` 和 `IJobChunk` 等作业时非常有用，因为它不会在主线程上引起同步。例如：

```csharp
new MyJob { healthLookup = SystemAPI.GetComponentLookup<HealthData>(isReadOnly: true) };
```

由于 ECS 缓存了这些数据，你可以直接在 OnUpdate 中调用它们。你不需要编写整个更新逻辑，因为其等价于：

```csharp
ComponentLookup<HealthData> lookup_HealthData_RO;

public void OnCreate(ref SystemState state)
{
    lookup_HealthData_RO = state.GetComponentLookup<HealthData>(isReadOnly: true);
}

public void OnUpdate(ref SystemState state)
{
    lookup_HealthData_RO.Update(ref state);
    new MyJob { healthLookup = lookup_HealthData_RO };
}

```

## Entities.ForEach 兼容性 (Entities.ForEach Compatibility)

在 `Entities.ForEach` 中，仅有一部分 `SystemAPI` 方法可用。以下是这些方法的列表：

### 数据类型与 API 对应关系

| Data Type | API                          |
| --------- | ---------------------------- |
| **组件数据**  | `GetComponentLookup`         |
|           | `GetComponent`               |
|           | `SetComponent`               |
|           | `HasComponent`               |
| **缓冲区**   | `GetBufferLookup`            |
|           | `GetBuffer`                  |
|           | `HasBuffer`                  |
| **实体信息**  | `GetEntityStorageInfoLookup` |
|           | `Exists`                     |
| **方面**    | `GetAspect`                  |

### 示例

下面是如何在 `Entities.ForEach` 中使用这些 `SystemAPI` 方法的示例：

#### 获取和设置组件数据

```csharp
using Unity.Entities;

public partial class ExampleSystem : SystemBase
{
    protected override void OnUpdate()
    {
        Entities.ForEach((Entity entity, in SomeComponent someComponent) =>
        {
            // 获取组件数据
            var myComponent = SystemAPI.GetComponent<MyComponent>(entity);

            // 设置组件数据
            SystemAPI.SetComponent(entity, new MyComponent { Value = myComponent.Value + 1 });

            // 检查组件是否存在
            bool hasMyComponent = SystemAPI.HasComponent<MyComponent>(entity);

        }).Schedule();
    }
}
```



## 访问单例 (Access Singletons)

`SystemAPI` 提供了一些单例方法，这些方法在调用时会检查是否只有一个实例的数据。这些方法不会同步，从而提升了性能。

例如，调用 `SystemAPI.GetSingleton<MyComponent>()` 时，会查询是否只有一个实体符合给定条件，如果是，则获取组件 `MyComponent`。它这样做时不会要求作业系统完成所有使用 `MyComponent` 的作业。

这是 `EntityManager.GetComponentData` 的一个有用替代方法，因为后者会同步数据。例如，当你调用 `EntityManager.GetComponentData<MyComponent>` 时，所有写入 `MyComponent` 的作业都会完成。

### 单例数据 API 列表

#### 单例组件数据 (Singleton Component Data)

| API 名称              | 描述                                 |
| ------------------- | ---------------------------------- |
| `GetSingleton`      | 获取单例组件数据。如果没有或有多个实例则抛出异常。          |
| `TryGetSingleton`   | 尝试获取单例组件数据。如果没有或有多个实例则返回 false。    |
| `GetSingletonRW`    | 获取单例组件的读写数据。如果没有或有多个实例则抛出异常。       |
| `TryGetSingletonRW` | 尝试获取单例组件的读写数据。如果没有或有多个实例则返回 false。 |
| `SetSingleton`      | 设置单例组件数据。                          |

#### 单例实体数据 (Singleton Entity Data)

| API 名称                  | 描述                                 |
| ----------------------- | ---------------------------------- |
| `GetSingletonEntity`    | 获取包含单例组件的实体。如果没有或有多个实例则抛出异常。       |
| `TryGetSingletonEntity` | 尝试获取包含单例组件的实体。如果没有或有多个实例则返回 false。 |

#### 单例缓冲区 (Singleton Buffers)

| API 名称                  | 描述                             |
| ----------------------- | ------------------------------ |
| `GetSingletonBuffer`    | 获取单例缓冲区。如果没有或有多个实例则抛出异常。       |
| `TryGetSingletonBuffer` | 尝试获取单例缓冲区。如果没有或有多个实例则返回 false。 |

#### 所有单例 (All Singletons)

| API 名称         | 描述          |
| -------------- | ----------- |
| `HasSingleton` | 检查是否存在单例组件。 |

### 示例

以下示例展示了如何使用 `SystemAPI` 来访问和操作单例数据：

#### 获取单例组件数据

```csharp
using Unity.Entities;

public partial class SingletonExampleSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 获取单例组件数据
        var myComponent = SystemAPI.GetSingleton<MyComponent>();

        // 尝试获取单例组件数据（只读）
        if (SystemAPI.TryGetSingleton<MyComponent>(out var myComponentRO))
        {
            // 使用只读数据进行操作
        }

        // 获取单例组件数据（读写）
        var myComponentRW = SystemAPI.GetSingletonRW<MyComponent>();

        // 设置单例组件数据
        SystemAPI.SetSingleton(new MyComponent { Value = 42 });

        // 检查是否存在单例组件
        bool hasSingleton = SystemAPI.HasSingleton<MyComponent>();
    }
}
```

## Managed 版本的 SystemAPI (Managed Versions of SystemAPI)

`SystemAPI.ManagedAPI` 命名空间暴露了 `SystemAPI` 方法的托管版本，你可以使用这些方法来访问托管组件。

### 数据类型与 API 对应关系

#### 组件数据 (Component Data)

| API 名称                           | 描述          |
| -------------------------------- | ----------- |
| `ManagedAPI.GetComponent`        | 获取托管组件数据    |
| `ManagedAPI.HasComponent`        | 检查是否有托管组件   |
| `ManagedAPI.IsComponentEnabled`  | 检查托管组件是否启用  |
| `ManagedAPI.SetComponentEnabled` | 设置托管组件的启用状态 |

#### 句柄 (Handles)

| API 名称                                    | 描述         |
| ----------------------------------------- | ---------- |
| `ManagedAPI.GetSharedComponentTypeHandle` | 获取共享组件类型句柄 |

#### 单例组件数据 (Singleton Component Data)

| API 名称                       | 描述           |
| ---------------------------- | ------------ |
| `ManagedAPI.GetSingleton`    | 获取单例托管组件数据   |
| `ManagedAPI.TryGetSingleton` | 尝试获取单例托管组件数据 |

#### 单例实体数据 (Singleton Entity Data)

| API 名称                             | 描述              |
| ---------------------------------- | --------------- |
| `ManagedAPI.GetSingletonEntity`    | 获取包含单例托管组件的实体   |
| `ManagedAPI.TryGetSingletonEntity` | 尝试获取包含单例托管组件的实体 |

#### 所有单例 (All Singletons)

| API 名称                    | 描述           |
| ------------------------- | ------------ |
| `ManagedAPI.HasSingleton` | 检查是否存在单例托管组件 |

### 示例

以下示例展示了如何使用 `ManagedAPI` 来访问和操作托管数据：

#### 获取和设置托管组件数据

```csharp
using Unity.Entities;

public partial class ManagedComponentExampleSystem : SystemBase
{
    protected override void OnUpdate()
    {
        Entities.ForEach((Entity entity) =>
        {
            // 获取托管组件数据
            var myManagedComponent = SystemAPI.ManagedAPI.GetComponent<MyManagedComponent>(entity);

            // 检查是否有托管组件
            bool hasMyManagedComponent = SystemAPI.ManagedAPI.HasComponent<MyManagedComponent>(entity);

            // 检查托管组件是否启用
            bool isComponentEnabled = SystemAPI.ManagedAPI.IsComponentEnabled<MyManagedComponent>(entity);

            // 启用或禁用托管组件
            SystemAPI.ManagedAPI.SetComponentEnabled<MyManagedComponent>(entity, true);

        }).WithoutBurst().Run(); // Managed API calls should run without Burst compilation
    }
}
```
