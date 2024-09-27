# Baker 概述

一个简单的烘焙系统使用 baker 从输入创作场景读取数据，并将这些数据作为组件写入实体作为输出。

要创建一个 baker，你需要使用 `Baker<TAuthoringType>` 管理泛型类并传递一个创作组件作为类型参数。创作组件是 UnityEngine 组件，通常是 MonoBehaviour。一个 baker 定义了一个接受该类型的创作组件作为输入的单一 Bake 方法。

Unity 调用每个需要烘焙的创作组件上的 Bake 方法。如果 Unity 执行完整烘焙，它会烘焙创作场景中的所有创作组件。如果执行增量烘焙，它只会烘焙已直接修改或其依赖项已被修改的组件。

### Baker 架构

一个 baker 只实例化一次，其 Bake 方法被多次调用，且顺序不确定。如果 Unity 执行增量烘焙，它会在很长一段时间内执行此操作。这意味着 bakers 需要是无状态的，并且总是通过方法访问数据。特别是，在 baker 中缓存任何值会违反这一不变性，并导致烘焙过程异常。

当一个创作 GameObject 被修改时，Unity 必须先撤销之前任何烘焙的效果，然后才能重新烘焙该 GameObject。

每个 baker 还必须声明其依赖项。默认情况下，一个 baker 只能访问单一创作组件。但当它从其他组件、其他 GameObjects 或各种资源中提取数据时，烘焙过程需要知道这一点。如果这些外部来源之一发生变化，baker 也需要重新运行。

例如，如果一个创作 GameObject 是一个立方体原始体，Unity 会将其烘焙为一个渲染立方体的实体。如果该创作 GameObject 后来被修改为球体，则其 ECS 等效项必须更改为一个渲染球体的实体。这意味着 Unity 必须销毁渲染立方体的实体，并创建一个新的渲染球体的实体。或者，Unity 需要更改实体以显示球体。如果 GameObject 具有依赖于一个可脚本化对象的材质，它会声明对该资源的依赖，以确保每当可脚本化对象被修改时，对象都会被重新烘焙。

因此，Unity 记录 baker 生成的所有内容，以便在重新烘焙之前撤销它。同样，Unity 记录 baker 访问的所有内容，以在需要时触发重新烘焙。这一过程是自动完成的，但这也是为什么大多数 baker 的操作是调用自己的成员函数，因为这些函数执行记录，没有它们逻辑将会破裂。

### 创建一个 baker

在创建一个 baker 时，你需要定义它所针对的 MonoBehaviour 创作组件，然后编写代码，使用创作组件数据创建和附加 ECS 组件到实体上。

Bakers 必须继承自 Baker 类。一个 baker 只能向创作组件的主要实体以及同一个 baker 创建的附加实体添加组件。例如：

```csharp
// RotationSpeedAuthoring 类必须遵循 MonoBehaviour 约定，
// 并且应该放在名为 RotationSpeedAuthoring.cs 的文件中
public class RotationSpeedAuthoring : MonoBehaviour
{
    public float DegreesPerSecond;
}

public struct RotationSpeed : IComponentData
{
    public float RadiansPerSecond;
}

public struct Additional : IComponentData
{
    public float SomeValue;
}

public class SimpleBaker : Baker<RotationSpeedAuthoring>
{
    public override void Bake(RotationSpeedAuthoring authoring)
    {
        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new RotationSpeed
        {
            RadiansPerSecond = math.radians(authoring.DegreesPerSecond)
        });

        var additionalA = CreateAdditionalEntity(TransformUsageFlags.Dynamic, entityName: "Additional A");
        var additionalB = CreateAdditionalEntity(TransformUsageFlags.Dynamic, entityName: "Additional B");

        AddComponent(additionalA, new Additional { SomeValue = 123 });
        AddComponent(additionalB, new Additional { SomeValue = 234 });
    }
}
```

## 在 baker 中访问其他数据源

为了保持增量烘焙功能，你需要跟踪在烘焙 GameObject 时使用了哪些数据。Unity 会自动跟踪创作组件中的任何字段，并在这些数据发生变化时重新运行 baker。

然而，Unity 不会自动跟踪来自其他来源的数据，例如创作组件或资源。你需要为 baker 添加一个依赖项，以便它可以跟踪这种数据。为此，请使用 Baker 类提供的方法来访问其他组件和 GameObjects，而不是使用 GameObject 提供的方法：

```csharp
public struct DependentData : IComponentData
{
    public float Distance;
    public int VertexCount;
}

public class DependentDataAuthoring : MonoBehaviour
{
    public GameObject Other;
    public Mesh Mesh;
}

public class GetComponentBaker : Baker<DependentDataAuthoring>
{
    public override void Bake(DependentDataAuthoring authoring)
    {
        // 在进行任何提前退出之前，声明对外部引用的依赖。
        // 因为即使这些评估结果为 null，它们仍可能是与丢失对象的正确 Unity 引用。
        // 这个依赖确保当那些对象被恢复时，baker 会被触发。

        DependsOn(authoring.Other);
        DependsOn(authoring.Mesh);

        if (authoring.Other == null) return;
        if (authoring.Mesh == null) return;

        var transform = GetComponent<Transform>();
        var transformOther = GetComponent<Transform>(authoring.Other);

        // 以下检查确保组件存在在这种情况下是不必要的，
        // 因为 Transform 始终存在于任何 GameObject 上。
        // 作为一般原则，建议检查是否缺少组件。

        if (transform == null) return;
        if (transformOther == null) return;

        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new DependentData
        {
            Distance = Vector3.Distance(transform.position, transformOther.position),
            VertexCount = authoring.Mesh.vertexCount
        });
    }
}
```

该示例对潜在的更改做出反应，并通过以下方式创建所需的依赖项：

1. **创作组件中的值变化**
   * 如果你更改了创作组件中的任何值，baker 会自动触发。你不需要在 baker 代码中显式表达任何依赖关系。
2. **资产引用丢失后恢复**
   * 如果创作组件引用的资产丢失，然后再次出现，对 `DependsOn` 的调用会触发 baker。这种依赖关系不是自动的，因为存储在创作组件中的数据没有改变：这是由该数据引用的某些东西进出存在（在 Unity 中称为 "fake null"）。这种依赖关系仅在通过这些引用提取数据时才需要。在本例中，这是从另一个 GameObject 获取的位置和从网格获取的顶点数。如果引用只是被传递而不通过它访问数据，则不需要依赖关系，因为 baker 不会以不同的方式处理任何事情。
3. **GetComponent 方法的作用**
   * `GetComponent` 方法还会注册对所需组件存在及其数据的引用。如果该组件缺失，`GetComponent` 返回 null，但它仍会注册一个依赖项。这样，当组件被添加时，baker 会被触发。

### 示例代码

```csharp
public struct DependentData : IComponentData
{
    public float Distance;
    public int VertexCount;
}

public class DependentDataAuthoring : MonoBehaviour
{
    public GameObject Other;
    public Mesh Mesh;
}

public class GetComponentBaker : Baker<DependentDataAuthoring>
{
    public override void Bake(DependentDataAuthoring authoring)
    {
        // 在进行任何提前退出之前，声明对外部引用的依赖。
        DependsOn(authoring.Other);
        DependsOn(authoring.Mesh);

        if (authoring.Other == null) return;
        if (authoring.Mesh == null) return;

        var transform = GetComponent<Transform>();
        var transformOther = GetComponent<Transform>(authoring.Other);

        // 检查确保组件存在，在这种情况下是不必要的，因为 Transform 始终存在于任何 GameObject 上。
        if (transform == null) return;
        if (transformOther == null) return;

        var entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new DependentData
        {
            Distance = Vector3.Distance(transform.position, transformOther.position),
            VertexCount = authoring.Mesh.vertexCount
        });
    }
}
```
