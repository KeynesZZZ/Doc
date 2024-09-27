# 自定义变换系统(Custom transforms)

## 自定义变换系统

你可以定制内置的变换系统，以满足项目中特定的变换功能需求。本节解释如何创建一个自定义变换系统，并使用 2D 自定义变换系统作为具体示例。

### 写入组 (Write Groups)

写入组允许你使用自己的变换覆盖内置的变换系统。内置变换系统在内部使用写入组，你可以配置它们，使内置变换系统忽略你希望使用自定义变换系统处理的实体。

更准确地说，写入组从查询中排除特定实体，这些查询传递给内置变换系统使用的作业。你可以在某些组件上使用写入组，从而将这些组件的实体排除在内置变换系统的作业之外，而这些实体可以由你自己的变换系统处理。关于更多信息，可以参考写入组的文档。

### 创建自定义变换系统

以下步骤概述了如何创建自定义变换系统：

1. 替换 `LocalTransform` 组件。
2. 创建一个作者组件以接收你的自定义变换。
3. 替换 `LocalToWorldSystem`。

#### 替换 LocalTransform 组件

内置变换系统默认会为每个实体添加一个 `LocalTransform` 组件。它存储表示实体位置、旋转和缩放的数据，并且还定义了一系列静态帮助方法。

要创建自己的自定义变换系统，你必须用自己的组件替换 `LocalTransform` 组件。

* 创建一个 `.cs` 文件来定义内置 `LocalTransform` 组件的替代品。你可以从 `Entities` 包中复制内置的 `LocalTransform.cs` 文件到你的资源文件夹，然后编辑内容。去项目中的 `Packages > Entities > Unity.Transforms` 路径下，复制 `LocalTransform.cs` 文件并重命名。
* 更改属性和方法以满足你的需求。请参见下面的自定义 `LocalTransform2D` 组件示例：

```csharp
using System.Globalization;
using Unity.Entities;
using Unity.Mathematics;

[WriteGroup(typeof(LocalToWorld))]
public struct LocalTransform2D : IComponentData
{
    public float2 Position;
    public float Scale;
    public float Rotation;

    public override string ToString()
    {
        return $"Position={Position.ToString()} Rotation={Rotation.ToString()} Scale={Scale.ToString(CultureInfo.InvariantCulture)}";
    }

    public float4x4 ToMatrix()
    {
        quaternion rotation = quaternion.RotateZ(math.radians(Rotation));
        return float4x4.TRS(new float3(Position.xy, 0f), rotation, new float3(Scale, Scale, 1f));
    }
}


//示例代码：替换 LocalTransform 组件
using Unity.Entities;
using Unity.Transforms;

public class CustomTransformSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 自定义变换系统逻辑
        Entities
            .ForEach((ref LocalTransform2D localTransform2D, ref LocalToWorld localToWorld) =>
            {
                // 更新世界矩阵
                localToWorld.Value = localTransform2D.ToMatrix();
            })
            .ScheduleParallel();
    }
}

public class CustomTransformAuthoring : MonoBehaviour
{
    public float2 initialPosition;
    public float initialScale;
    public float initialRotation;

    public class Baker : Baker<CustomTransformAuthoring>
    {
        public override void Bake(CustomTransformAuthoring authoring)
        {
            var entity = GetEntity(TransformUsageFlags.Dynamic);
            AddComponent(entity, new LocalTransform2D
            {
                Position = authoring.initialPosition,
                Scale = authoring.initialScale,
                Rotation = authoring.initialRotation
            });
        }
    }
}

```

## 自定义变换系统修改细节

在上述示例中，自定义 `LocalTransform2D` 组件对内置的 `LocalTransform` 进行了以下修改：

1. **添加 `WriteGroup(typeof(LocalToWorld))` 属性**：
   * 通过添加 `[WriteGroup(typeof(LocalToWorld))]` 属性，确保带有 `LocalTransform2D` 的实体不会被内置变换系统处理。
2. **将 `Position` 字段从 `float3` 减少为 `float2`**：
   * 在 2D 示例中，实体仅沿 XY 平面移动，因此将 `Position` 字段从 `float3` 改为 `float2`。
3. **将 `Rotation` 字段减少为表示绕 Z 轴旋转角度的 `float`**：
   * 内置变换系统的 `Rotation` 属性是一个表示三维空间旋转的四元数。在 2D 示例中，仅需表示绕 Z 轴的旋转角度，因此将 `Rotation` 字段改为 `float`。
4. **移除所有方法，除了 `ToMatrix` 和 `ToString`**：
   * 保留并修改 `ToMatrix` 方法以适应 2D 需求。删除其他不必要的方法，以简化自定义 2D 变换系统。

注意：LocalTransform2D 位于全局命名空间。在上述链接的示例项目中，它位于子命名空间中，以确保不会干扰同一项目中的其他示例。只要自定义变换系统的所有文件都在相同的命名空间内，这两种选项都可以正常工作。

## 创建作者组件

每个需要由自定义变换系统处理的实体必须满足以下条件：

1. 具有一个自定义替代 `LocalTransform` 组件，并且名称不同。
2. 具有 `LocalToWorld` 组件。
3. 如果实体有父实体，则它必须有一个指向父实体的 `Parent` 组件。

为了满足这些条件，需要在每个实体上添加一个作者组件，并使用变换使用标志来防止实体接收来自内置变换系统的任何组件。下面是一个示例代码，展示了如何创建这样的作者组件：

```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;
using UnityEngine;

public class Transform2DAuthoring : MonoBehaviour
{
    class Baker : Baker<Transform2DAuthoring>
    {
        public override void Bake(Transform2DAuthoring authoring)
        {
            // 确保不添加标准变换组件。
            var entity = GetEntity(TransformUsageFlags.ManualOverride);
            
            AddComponent(entity, new LocalTransform2D
            {
                Scale = 1
            });
            
            AddComponent(entity, new LocalToWorld
            {
                Value = float4x4.Scale(1)
            });

            var parentGO = authoring.transform.parent;
            if (parentGO != null)
            {
                AddComponent(entity, new Parent
                {
                    Value = GetEntity(parentGO, TransformUsageFlags.None)
                });
            }
        }
    }
}
```

## 注意事项

### 启用不安全代码

`LocalToWorldSystem` 使用了不安全的本机代码，为了避免错误，需要在项目中启用“允许不安全代码”属性。以下是启用该属性的步骤：

1. **打开项目设置**:
   * 在 Unity 编辑器中，导航到 `Edit > Project Settings > Player > Other Settings`
2. **选择 "Allow unsafe code"**:
   * 在“其他设置”（Other Settings）部分中，勾选“允许不安全代码”（Allow unsafe code）

这将允许你的项目使用不安全代码，从而避免与 `LocalToWorldSystem` 相关的错误允许不安全代码” ![Step 3](https://docs.unity3d.com/uploads/Main/UnsafeCodeOption.png)

通过启用此属性，可以确保自定义的变换系统在使用 `LocalToWorldSystem` 时不会遇到因禁用不安全代码而导致的问题。
