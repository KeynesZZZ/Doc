# 变换使用标志(Transform usage flags )

## 变换使用标志

变换使用标志控制 Unity 如何将 Transform MonoBehaviour 组件转换为实体数据。你可以使用 `TransformUsageFlags` 中的值来定义在烘焙过程中添加到实体的变换组件。

这些标志帮助减少烘焙实体中的不必要变换组件。可用的标志如下：

* **None**: 表示没有特定的变换组件需求。然而，其他烘焙器可以向实体添加 `TransformUsageFlags` 值。
* **Renderable**: 表示实体需要渲染所需的变换组件，但不需要在运行时移动实体的变换组件。
* **Dynamic**: 表示实体需要在运行时移动的变换组件。
* **WorldSpace**: 表示实体必须处于世界空间，即使它有一个动态的父实体。
* **NonUniformScale**: 表示实体需要表示非均匀缩放的变换组件。
* **ManualOverride**: 忽略同一 GameObject 上其他烘焙器的所有 `TransformUsageFlags` 值。不向实体添加任何变换组件。

Unity 在烘焙器访问实体时需要这些标志。此外，默认 GameObject 组件的烘焙器会自动向烘焙实体添加适当的变换使用标志。例如，`MeshRenderer` 的烘焙器会添加 `Renderable` 作为变换使用标志。

### 使用变换使用标志

你可以在一个实体上使用多个标志，Unity 会在添加变换组件之前组合这些标志。例如，如果一个实体上有 `Dynamic` 和 `WorldSpace` 标志，Unity 会认为该实体在运行时是动态且处于世界空间的。

变换使用标志有助于减少烘焙实体中不必要的变换组件。例如，如果你有一个表示建筑物的 GameObject，并且建筑物有一个表示窗户的子 GameObject，因为这些 GameObject 在运行时不会移动，它们都有 `Renderable` 标志。当烘焙过程运行时，这些 GameObject 的实体不需要处于层次结构中，它们的变换信息可以结合在一个世界空间的 `LocalToWorld` 组件中。Unity 不会为这些组件生成 `LocalTransform` 或 `Parent`，从而节省不必要的数据。

类似地，如果窗户 GameObject 位于一个表示船的 GameObject 上，你可以将船标记为 `Dynamic`，并保持窗口为 `Renderable`。在运行时，Unity 提供正确的变换组件（`LocalToWorld`、`LocalTransform` 和 `Parent`）给窗口，以确保它跟随船移动：

#### 示例代码：使用变换使用标志

以下是如何在实体上使用变换使用标志的示例：

```csharp
using Unity.Entities;
using Unity.Transforms;

public struct Ship : IComponentData
{
    public float speed;
    // 其他数据
}

public class ShipAuthoring : MonoBehaviour
{
    public float speed;
    // 其他作者数据

    public class Baker : Baker<ShipAuthoring>
    {
        public override void Bake(ShipAuthoring authoring)
        {
            // 将变换使用标志设置为 Dynamic
            var entity = GetEntity(TransformUsageFlags.Dynamic);
            AddComponent(entity, new Ship
            {
                speed = authoring.speed
                // 分配其他数据
            });
        }
    }
}
```

## 重要提示

实体预制件会自动标记为 `Dynamic`，以便实例可以被放置在世界中。
