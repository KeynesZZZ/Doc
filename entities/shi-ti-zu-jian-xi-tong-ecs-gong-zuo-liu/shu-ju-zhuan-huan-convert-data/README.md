# 数据转换 （Convert data）

在你的 Unity 项目中，数据通过烘焙过程转换为实体和组件。本节包含有关 Unity 如何转换数据的信息。

### 主题和描述

| 主题             | 描述                             |
| -------------- | ------------------------------ |
| **使用烘焙进行数据转换** | 了解如何将 GameObject 数据转换为 ECS 数据。 |
| **场景概述**       | 了解不同类型的场景及其如何与烘焙过程协同工作。        |

#### 使用烘焙进行数据转换

了解如何将 GameObject 数据转换为 ECS 数据。

* **详细描述**：
  * 在 Unity 中，通过烘焙过程将传统的 GameObject 和 MonoBehaviour 数据转换为 ECS 的实体和组件。
  * 这个过程通常在构建或运行时发生，确保在 ECS 环境下以高效的方式处理和管理游戏对象的数据。

#### 场景概述

了解不同类型的场景及其如何与烘焙过程协同工作。

* **详细描述**：
  * Unity 支持多种类型的场景，例如编辑器场景、子场景等。
  * 每种场景类型在烘焙过程中有不同的处理方式。例如，子场景可以预先烘焙并在运行时直接加载，以提高性能和减少加载时间。

### 示例：使用烘焙进行数据转换

以下示例展示了如何设置一个简单的烘焙脚本，将 GameObject 数据转换为 ECS 数据。

#### 示例代码

**GameObject 转换为 ECS 实体**

```csharp
using Unity.Entities;
using UnityEngine;

public class ConvertToEntity : MonoBehaviour
{
    void Start()
    {
        // 获取当前的世界
        var world = World.DefaultGameObjectInjectionWorld;

        // 创建 EntityManager
        var entityManager = world.EntityManager;

        // 将当前 GameObject 转换为 ECS 实体
        var entity = entityManager.CreateEntity();

        // 添加组件到实体
        entityManager.AddComponentData(entity, new Translation { Value = transform.position });
        entityManager.AddComponentData(entity, new Rotation { Value = transform.rotation });
    }
}
```

###

### 场景概述

编辑器场景：

这些场景在 Unity 编辑器中创建和编辑，并在运行时通过烘焙过程转换为 ECS 数据。

&#x20;子场景：

子场景允许你将大场景分割成更小的部分，以便更好地管理和优化。 子场景可以单独烘焙，并在需要时动态加载，从而减少内存占用和加载时间。

### 示例：使用子场景进行数据管理

```csharp
using Unity.Entities;
using Unity.Scenes;
using UnityEngine;

public class SubSceneExample : MonoBehaviour
{
    public SubScene subScene;

    void Start()
    {
        // 加载子场景
        var sceneSystem = World.DefaultGameObjectInjectionWorld.GetOrCreateSystem<SceneSystem>();

        // 指定路径加载子场景
        var sceneGUID = SceneSystem.GetSceneGUID(subScene.ScenePath);
        sceneSystem.LoadSceneAsync(sceneGUID, new SceneSystem.LoadParameters { AutoLoad = true });
    }
}

```
