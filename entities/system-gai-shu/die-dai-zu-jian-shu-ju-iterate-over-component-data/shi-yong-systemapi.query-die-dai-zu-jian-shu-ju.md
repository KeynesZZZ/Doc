# 使用 SystemAPI.Query 迭代组件数据

要在主线程上迭代数据集，可以在 `ISystem` 和 `SystemBase` 系统类型中使用 `SystemAPI.Query<T>` 方法。它使用 C# 习惯的 `foreach` 语法。

你可以用多达七个类型参数重载该方法。支持的类型参数包括：

* `IAspect`
* `IComponentData`
* `ISharedComponentData`
* `DynamicBuffer<T>`
* `RefRO<T>`
* `RefRW<T>`
* `EnabledRefRO<T>` (其中 `T` 为 `IEnableableComponent`, `IComponentData`)
* `EnabledRefRW<T>` (其中 `T` 为 `IEnableableComponent`, `IComponentData`)

### `SystemAPI.Query` 实现

每当调用 `SystemAPI.Query<T>` 时，源生成器解决方案会在系统本身上创建一个 `EntityQuery` 字段。它还会缓存一个由查询类型及其相应的读写/只读访问模式组成的 `EntityQuery` 在这个字段中。编译期间，源生成解决方案将 `foreach` 语句中的 `SystemAPI.Query<T>` 调用替换为一个枚举器，该枚举器通过缓存的查询数据进行迭代。

此外，源生成解决方案缓存所有必需的类型句柄，并在每个 `foreach` 之前根据需要自动注入 `TypeHandle.Update(SystemBase system)` 或 `TypeHandle.Update(ref SystemState state)`。这确保了类型句柄的安全性。

源生成器还生成代码以在每个 `foreach` 语句之前自动完成所有必要的读和读写依赖项。

## 查询数据

以下是一个使用 `SystemAPI.Query` 迭代每个具有 `LocalTransform` 和 `RotationSpeed` 组件的实体的示例：

```csharp
public partial struct MyRotationSpeedSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // 使用 SystemAPI.Query 迭代具有 LocalTransform 和 RotationSpeed 组件的实体
        foreach (var (transform, speed) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<RotationSpeed>>())
        {
            // 更新 transform 的旋转值
            transform.ValueRW = transform.ValueRO.RotateY(speed.ValueRO.RadiansPerSecond * deltaTime);
        }
    }
}
```

因为示例修改了 `LocalTransform` 数据，所以它被包裹在 `RefRW<T>` 中，作为读写引用。然而，因为只读取 `RotationSpeed` 数据，所以使用 `RefRO<T>`。`RefRO<T>` 的使用完全是可选的，你可以使用以下代码作为有效的替代：

```csharp
float deltaTime = SystemAPI.Time.DeltaTime;

foreach (var (transform, speed) in SystemAPI.Query<RefRW<LocalTransform>, RotationSpeed>())
    transform.ValueRW = transform.ValueRO.RotateY(speed.RadiansPerSecond * deltaTime);
```

RefRW.ValueRW, RefRW.ValueRO 和 RefRO.ValueRO 都返回组件的引用。调用时，ValueRW 会进行读写访问的安全检查，而 ValueRO 也会进行相应的只读访问安全检查。

## 在 `foreach` 语句中访问实体

`Unity.Entities.Entity` 不是支持的类型参数。每个查询已经隐式地过滤了所有存在的实体。要访问实体，请使用 `WithEntityAccess`。例如：

```csharp
foreach (var (transform, speed, entity) in SystemAPI.Query<RefRW<LocalToWorld>, RefRO<RotationSpeed>>().WithEntityAccess())
{
    // 处理代码
}
```

请注意，Entity 参数在返回的元组中是最后一个。

## 已知限制

`SystemAPI.Query` 有以下限制，如下所述。

### 动态缓冲区只读限制

在 `SystemAPI.Query<T>` 中，`DynamicBuffer<T>` 类型参数默认是读写访问。然而，如果你想要只读访问，你需要创建自己的实现，类似于以下示例：

```csharp
var bufferHandle = state.GetBufferTypeHandle<MyBufferElement>(isReadOnly: true);
var myBufferElementQuery = SystemAPI.QueryBuilder().WithAll<MyBufferElement>().Build();
var chunks = myBufferElementQuery.ToArchetypeChunkArray(Allocator.Temp);

foreach (var chunk in chunks)
{
    var numEntities = chunk.Count;
    var bufferAccessor = chunk.GetBufferAccessor(ref bufferHandle);

    for (int j = 0; j < numEntities; j++)
    {
        var dynamicBuffer = bufferAccessor[j];
        // 从 dynamicBuffer 中读取并执行各种操作
    }
}
```

## 重用 `SystemAPI.Query`

你不能将 `SystemAPI.Query<T>` 存储在变量中，然后在多个 `foreach` 语句中使用：没有办法重用 `SystemAPI.Query`。这是因为 API 的实现依赖于在编译时知道查询类型。源生成解决方案在编译时不知道要生成和缓存哪个 `EntityQuery`，也不知道要调用哪些类型句柄的更新，也不知道要完成哪些依赖关系。

### 示例说明

下面的代码示例展示了为什么你不能重用 `SystemAPI.Query<T>`：

```csharp
public partial struct MySystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // 无法将 SystemAPI.Query<T> 存储在变量中并重用
        var query = SystemAPI.Query<RefRW<LocalTransform>, RefRO<RotationSpeed>>();

        foreach (var (transform, speed) in query)
        {
            transform.ValueRW = transform.ValueRO.RotateY(speed.ValueRO.RadiansPerSecond * deltaTime);
        }

        // 再次使用同一个 query 会导致编译错误
        foreach (var (transform, speed) in query)
        {
            // 其他处理逻辑
        }
    }
}
```

**原因**&#x20;

* 编译时确定性：SystemAPI.Query 依赖于编译时确定查询类型。源生成解决方案需要在编译时知道要生成和缓存哪个 EntityQuery。
* 类型句柄更新：每个查询都需要相应的类型句柄，并且这些句柄必须在每次 foreach 之前更新。
* &#x20;依赖关系管理：每个查询需要自动完成所有必要的读和读写依赖关系，这要求在每次查询时重新确定和管理这些依赖关系。

**解决方法**&#x20;

如果你需要在多个地方使用相同的查询，可以直接在每个 foreach 语句中重新调用 SystemAPI.Query，而不是试图存储和重用它。这种方式虽然可能看起来有些冗余，但确保了正确的类型安全和性能优化。

```
public partial struct MySystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        // 第一次查询并处理数据
        foreach (var (transform, speed) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<RotationSpeed>>())
        {
            transform.ValueRW = transform.ValueRO.RotateY(speed.ValueRO.RadiansPerSecond * deltaTime);
        }

        // 第二次查询并处理数据
        foreach (var (transform, speed) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<RotationSpeed>>())
        {
            // 其他处理逻辑
        }
    }
}

```

