# 创建 Aspect

要创建一个 Aspect，请使用 `IAspect` 接口。必须将 Aspect 声明为只读的部分结构（readonly partial struct），并且该结构必须实现 `IAspect` 接口：

```csharp
using Unity.Entities;

readonly partial struct MyAspect : IAspect
{
    // 您的 Aspect 代码
}
```

### 字段

你可以使用 `RefRW<T>` 或 `RefRO<T>` 将组件声明为 Aspect 的一部分。要声明一个缓冲区，使用 `DynamicBuffer<T>`。有关可用字段的更多信息，请参阅 `IAspect` 文档。

在 Aspect 内声明的字段定义了为了使 Aspect 实例对特定实体有效必须查询哪些数据。

要使字段成为可选的，可以使用 `[Optional]` 属性。要将 `DynamicBuffer` 和嵌套的 Aspects 声明为只读，可以使用 `[ReadOnly]` 属性。

#### 只读和读写访问

使用 `RefRO` 和 `RefRW` 字段来提供对 Aspect 中组件的只读或读写访问。当你想在代码中引用一个 Aspect 时，使用 `in` 来将所有引用覆盖为只读，或者使用 `ref` 来遵循在 Aspect 中声明的只读或读写访问权限。

如果你使用 `in` 引用一个具有读写权限的 Aspect，在尝试写入时可能会抛出异常。

### 在系统中创建 Aspect 实例

要在系统中创建 Aspect 实例，调用 `SystemAPI.GetAspect`：

```csharp
// 如果实体缺少任何 MyAspect 所需的组件，将抛出异常。
MyAspect asp = SystemAPI.GetAspect<MyAspect>(myEntity);
```

要在系统外创建 Aspect 实例，使用 EntityManager.GetAspect。

### 遍历 Aspect

如果你想遍历一个 Aspect，可以使用 `SystemAPI.Query`：

```csharp
public partial struct MySystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var cannonball in SystemAPI.Query<CannonBallAspect>())
        {
            // 在这里使用 cannonball aspect
        }
    }
}
```

### 示例

在这个例子中，`CannonBallAspect` 设置了坦克主题游戏中炮弹组件的变换、位置和速度。

```csharp
using Unity.Entities;
using Unity.Mathematics;

struct CannonBall : IComponentData
{
    public float3 Speed;
}

// Aspects 必须声明为只读部分结构体（readonly partial struct）
readonly partial struct CannonBallAspect : IAspect
{
    // Aspect 中的 Entity 字段提供对实体本身的访问。
    // 例如，这在使用 EntityCommandBuffer 注册命令时是必需的。
    public readonly Entity Self;

    // Aspects 可以包含其他 aspects。

    // RefRW 字段提供对组件的读写访问。如果 aspect 被作为 "in" 参数传递，
    // 该字段表现得像 RefRO，并在尝试写入时抛出异常。
    readonly RefRW<LocalTransform> Transform;
    readonly RefRW<CannonBall> CannonBall;

    // 像这样的属性不是必须的。Transform 字段可以公开。
    // 但通过避免 "aspect.aspect.aspect.component.value.value" 的链条，属性提高了可读性。
    public float3 Position
    {
        get => Transform.ValueRO.Position;
        set => Transform.ValueRW.Position = value;
    }

    public float3 Speed
    {
        get => CannonBall.ValueRO.Speed;
        set => CannonBall.ValueRW.Speed = value;
    }
}
```

### 在其他代码中使用这个 Aspect

你可以像请求组件一样请求 `CannonBallAspect`：

```csharp
using Unity.Entities;
using Unity.Burst;

// 最佳实践是对你的代码进行 Burst 编译
[BurstCompile]
partial struct CannonBallJob : IJobEntity
{
    void Execute(ref CannonBallAspect cannonBall)
    {
        // 你的游戏逻辑
    }
}
```
