# Aspect 源代码生成

源代码生成器通过分析现有代码，在编译期间生成代码。Entities 包会生成方法和类型，以便你可以将 Aspects 与 Unity API 的其他部分一起使用。有关 Unity 中源代码生成器的更多信息，请参阅用户手册中关于 Roslyn 分析器和源代码生成器的文档。

#### Aspect 生成的方法

Aspect 源代码生成器会在你的 Aspect 结构体中生成额外的方法，使其可以与 `IJobEntity` 和 `SystemAPI.Query<MyAspect>()` 等 API 一起使用。

**AddComponentRequirementsTo**

将此 Aspect 的组件要求添加到原型列表中。如果列表中已经存在某个组件，它不会添加重复组件。但是，如果此 Aspect 需要，则会将只读要求覆盖为读写要求。

### 声明

```csharp
public void AddComponentRequirementsTo(
    ref UnsafeList<ComponentType> all, 
    ref UnsafeList<ComponentType> any, 
    ref UnsafeList<ComponentType> none, 
    bool isReadOnly)
```

### 参数

* **all**：原型必须匹配所有组件要求。
* **any**：原型必须匹配任何一个组件要求。
* **none**：原型必须不匹配任何一个组件要求。
* **isReadOnly**：设置为 `true` 将使所有组件变为只读。

### CreateAspect

为特定实体的组件数据创建 Aspect 结构体实例。

#### 声明

```csharp
public AspectT CreateAspect(
    Entity entity, 
    ref SystemState systemState, 
    bool isReadOnly)
```

#### 参数

* **entity**：从该实体创建 Aspect 结构体。
* **systemState**：从中提取数据的系统状态。
* **isReadOnly**：设置为 `true` 将使对数据的所有引用变为只读。

#### 返回值

一个指向实体组件数据的 Aspect 结构体。

示例代码&#x20;

以下是如何使用 CreateAspect 方法的示例：

```csharp
using Unity.Entities;

public class AspectExample : SystemBase
{
    protected override void OnUpdate()
    {
        // 假设你有一个实体实例
        Entity myEntity = ...;

        // 获取系统状态
        var state = World.DefaultGameObjectInjectionWorld.GetExistingSystem<SystemState>();

        // 创建 CannonBallAspect 的实例
        var cannonBallAspect = state.EntityManager.CreateAspect<CannonBallAspect>(myEntity, ref state, false);

        // 使用该 Aspect 实例
        cannonBallAspect.Position = new float3(1, 2, 3);
        cannonBallAspect.Speed = new float3(0, 10, 0);
    }
}

```

### Query

创建一个 `IEnumerable<AspectT>`，你可以用它来遍历查询到的实体 Aspect。

#### 声明

```csharp
public static Enumerator Query(EntityQuery query, TypeHandle typeHandle)
```

#### 参数

* **query**：用于枚举的 `EntityQuery` 实例。
* **typeHandle**：Aspect 的类型句柄。

#### 示例代码

以下是如何使用 `Query` 方法的示例：

```csharp
using Unity.Entities;

public partial struct MySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // 创建 EntityQuery
        EntityQuery query = state.EntityManager.CreateEntityQuery(typeof(CannonBall));

        // 获取 Aspect 的 TypeHandle（假设 CannonBallAspect 已定义）
        var typeHandle = state.GetComponentTypeHandle<CannonBallAspect>();

        // 使用 Query 方法获取 Enumerator
        var enumerator = SystemAPI.Query(query, typeHandle);

        // 遍历查询到的实体 Aspects
        foreach (var cannonball in enumerator)
        {
            // 在这里使用 cannonball aspect
        }
    }
}
```

### CompleteDependencyBeforeRO

完成此 Aspect 所需的依赖链，以便进行只读访问。这将完成组件、缓冲区等的所有写入依赖关系，从而允许读取。

#### 声明

```csharp
public static void CompleteDependencyBeforeRO(ref SystemState systemState)
```

#### 参数

* **state**：包含所有依赖关系的 `SystemState`，其中包含一个 `EntityManager`。

#### 示例代码

以下是如何使用 `CompleteDependencyBeforeRO` 方法的示例：

```csharp
using Unity.Entities;

public partial struct MySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // 完成所需的依赖链以便进行只读访问
        CannonBallAspect.CompleteDependencyBeforeRO(ref state);

        // 创建 EntityQuery
        EntityQuery query = state.EntityManager.CreateEntityQuery(typeof(CannonBall));

        // 获取 Aspect 的 TypeHandle（假设 CannonBallAspect 已定义）
        var typeHandle = state.GetComponentTypeHandle<CannonBallAspect>(isReadOnly: true);

        // 使用 Query 方法获取 Enumerator
        var enumerator = SystemAPI.Query(query, typeHandle);

        // 遍历查询到的实体 Aspects
        foreach (var cannonball in enumerator)
        {
            // 在这里使用 cannonball aspect，只读操作
        }
    }
}
```

### CompleteDependencyBeforeRW

完成此组件所需的依赖链以便进行读写访问。这将完成组件、缓冲区等的所有写入依赖关系，从而允许读取，并完成所有读取依赖关系，以便你可以写入数据。

#### 声明

```csharp
public static void CompleteDependencyBeforeRW(ref SystemState state)
```

#### 参数

* state：包含所有依赖关系的 SystemState，其中包含一个 EntityManager。

#### 示例代码

以下是如何使用 ompleteDependencyBeforeRW 方法的示例：

```csharp
using Unity.Entities;

public partial struct MySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // 完成所需的依赖链以便进行读写访问
        CannonBallAspect.CompleteDependencyBeforeRW(ref state);

        // 创建 EntityQuery
        EntityQuery query = state.EntityManager.CreateEntityQuery(typeof(CannonBall));

        // 获取 Aspect 的 TypeHandle（假设 CannonBallAspect 已定义）
        var typeHandle = state.GetComponentTypeHandle<CannonBallAspect>(isReadOnly: false);

        // 使用 Query 方法获取 Enumerator
        var enumerator = SystemAPI.Query(query, typeHandle);

        // 遍历查询到的实体 Aspects
        foreach (var cannonball in enumerator)
        {
            // 在这里使用 cannonball aspect，读写操作
            cannonball.Position += new float3(0, 1, 0); // 修改位置
            cannonball.Speed += new float3(1, 0, 0);    // 修改速度
        }
    }
}

```

### Aspect 生成的类型

Aspect 源代码生成器在所有实现 `IAspect` 的 Aspect 结构体中声明了新的嵌套类型。

#### MyAspect.Lookup

一个结构体，用于访问给定实体上的 Aspect。它由必要的结构体（如 `ComponentLookup` 和 `BufferLookup`）组成，以提供对所有 Aspect 字段数据的访问。

**生成的方法**

* **Lookup(ref SystemState state)**：从 `SystemState` 构建查找。
* **Update(ref SystemState state)**：在使用此实体之前更新 Aspect。
* **MyAspect this\[Entity entity]**：从该实体查找 `MyAspect`。

#### MyAspect.TypeHandle

一个结构体，用于从 `ArchetypeChunk` 访问 Aspect。它由必要的结构体（如 `ComponentTypeHandle` 和 `BufferTypeHandle`）组成，以提供对块中所有 Aspect 字段数据的访问。

**生成的方法**

* **TypeHandle(ref SystemState state)**：从 `SystemState` 构建 `TypeHandle`。
* **Update(ref SystemState state)**：在使用 `Resolve` 之前更新 Aspect。
* **ResolvedChunk Resolve(ArchetypeChunk chunk)**：获取 Aspect 的块数据。

#### MyAspect.ResolvedChunk

一个结构体，表示块中 Aspect 的所有实例。它由必要的块结构体（如 `NativeArray` 和 `BufferAccessor`）组成，并提供索引器和长度。

**生成的字段**

* **NativeArray #AspectPath#\_#FieldName#NaE;**：表示一个实体 aspect 字段。
* **NativeArray #AspectPath#\_#FieldName#NaC;**：表示每个 `RefRO/RW<ComponentT>` aspect 字段。
* **BufferAccessor #AspectPath#\_#FieldName#Ba;**：表示每个 `DynamicBuffer<BufferElementT>` aspect 字段。
* **SharedComponentT #AspectPath#\_#FieldName#Sc;**：表示每个共享组件 aspect 字段。
* **int Length;**：此块中所有 `NativeArray` 和 `BufferAccessor` 的长度。

**字段名称**

* **#AspectPath#**：封闭 aspect 的名称。对于嵌套 aspects，这是从根 aspect 开始的全路径，用下划线分隔。例如，`MyRootAspect_MyNestedAspect`。
* **#FieldName#**：字段的名称，具有相同的大小写。

#### MyAspect.Enumerator

一个结构体，用于迭代 `EntityQuery` 中 aspect 的所有实例。

**实现**

* **System.Collections.Generic.IEnumerator**
* **System.Collections.Generic.IEnumerable**
