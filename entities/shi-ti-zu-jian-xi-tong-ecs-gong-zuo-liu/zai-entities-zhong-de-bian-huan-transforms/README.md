# 在 Entities 中的变换 (Transforms)

## 在 Entities 中的变换 (Transforms)

本节包含有关变换在 Entities 中如何工作的信息，以及如何控制项目中任何实体的世界空间位置、旋转和缩放。

### 主题描述

| 主题           | 描述                                   |
| ------------ | ------------------------------------ |
| Transform 概念 | 了解变换在 Entities 中是如何工作的。              |
| 使用变换         | 在你的项目中使用变换。                          |
| 变换比较         | 比较 ECS 中的变换操作和 `UnityEngine` 中的变换操作。 |
| 变换助手概述       | 使用变换助手处理变换矩阵。                        |
| 变换使用标志       | 了解变换使用标志，这有助于高效转换变换数据。               |
| 自定义变换        | 创建自定义变换系统。                           |

### 详细描述

#### Transform 概念

在 Entities 中，变换由特定的组件来表示，例如 `LocalTransform` 和 `WorldTransform`。这些组件在不同的坐标系中定义实体的位置信息。

#### 使用变换

要在你的项目中使用变换，你可以添加和操作相应的变换组件。例如，可以使用 `Translation`、`Rotation` 和 `Scale` 组件来控制实体的世界空间属性。

#### 变换比较

ECS 中的变换操作与 UnityEngine 中的变换操作不同。在 ECS 中，变换操作更倾向于批处理和并行计算，而不是逐个对象操作。这提高了性能和效率。

#### 变换助手概述

变换助手提供了一组工具，用于简化变换矩阵的操作。这些工具通常包括矩阵乘法、逆矩阵计算等功能。

#### 变换使用标志

变换使用标志帮助确定何时以及如何更新变换数据。这些标志可以优化变换数据的转换和传输。

#### 自定义变换

如果需要非标准的变换逻辑，可以创建自定义变换系统。此系统可以定义特定的规则和行为，以适应特殊需求。

### 示例代码：使用基本变换组件

```csharp
using Unity.Entities;
using Unity.Transforms;

public partial class MoveSystem : SystemBase
{
    protected override void OnUpdate()
    {
        Entities
            .ForEach((ref Translation translation, in Velocity velocity) =>
            {
                translation.Value += velocity.Value * SystemAPI.Time.DeltaTime;
            })
            .ScheduleParallel();
    }
}
```
