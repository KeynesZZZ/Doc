# 场景部分概述（Scene section overview）

Unity 将场景中的所有实体分组为多个部分，默认部分为 0。每个场景中的实体都有一个 `SceneSection` 共享组件，该组件指示该实体所属的部分。

`SceneSection` 包含场景的 GUID（作为 Hash128）和部分号（作为整数）。

### 分配部分

如果你想控制实体被分配到哪个部分，可以执行以下操作：

* 使用作者组件 `SceneSectionComponent`。此作者组件会影响其所在的作者 GameObject 及其所有子对象（递归）。
* 编写一个自定义烘焙系统，直接设置 `SceneSection` 值。不能在 `Baker` 中分配 `SceneSection` 的值。

部分索引不需要是连续的，但默认部分 0 总是存在，即使它是空的。例如，你可以有默认部分 0 和索引为 123 的部分。

在编辑器中，仅当子场景关闭时才会应用场景部分。打开的子场景将所有实体都放在部分 0 中。

### 编辑器中的场景部分

你可以在编辑器的组件检查器中查看场景部分及其 GUID 的详细信息。![](<../../../../.gitbook/assets/image (2).png>)

#### 编辑器检查器中的场景部分组件

<figure><img src="../../../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

当子场景组件关闭时，检查器列出该子场景中的部分。部分 0 总是首先出现（没有部分索引）。

通过检查器显示的 `ConvertedScene` 的默认值，以及另一个索引为 123 的 `ConvertedScene`

### 横跨部分的引用

在子场景中，ECS 组件只能包含对以下内容的引用：

* 与它们相同部分中的实体
* 部分 0 中的实体

#### 重要提示

对来自不同部分的实体的引用，或者不是部分 0 中的实体的引用，在加载时会被设置为 `Entity.Null`。

### 实体预制件和部分

场景中的所有实体都有一个 `SceneSection` 组件，将它们链接到场景中的某个部分。当该部分或场景被卸载时，所有具有匹配 `SceneSection` 组件的实体也将被卸载。这也适用于实体预制件。

当实体预制件被实例化时，其 `SceneSection` 会被添加到实例化的实体。这意味着卸载场景将销毁与之关联的所有预制件实例。如果这不是所需的行为，你可以手动从预制件实例中删除 `SceneSection` 组件。

## 分部分加载

你可以独立加载或卸载场景的各个部分，但部分 0 必须始终首先加载。同样，只有在场景中的其他所有部分都已卸载后，才可以卸载部分 0。

### 加载特定部分的内容

要加载特定部分的内容，请将组件 `Unity.Entities.RequestSceneLoaded` 添加到部分元实体。你可以查询场景元实体上的 `ResolvedSectionEntity` 缓冲区以访问各个部分元实体。

#### 示例代码：加载每隔一个部分

```csharp
// 为了简化示例代码，我们假设这些部分已经解析。
// 确保代码只运行一次的逻辑也未包含在内。
var sectionBuffer = EntityManager.GetBuffer<ResolvedSectionEntity>(sceneEntity);
var sectionEntities = sectionBuffer.ToNativeArray(Allocator.Temp);

for (int i = 0; i < sectionEntities.Length; i += 1)
{
    if (i % 2 == 0)
    {
        // 注意，这个条件包括了部分 0，
        // 如果部分 0 缺失，则其他任何东西都不会加载。
        var sectionEntity = sectionEntities[i].SectionEntity;
        EntityManager.AddComponent<RequestSceneLoaded>(sectionEntity);
    }
}

sectionEntities.Dispose();
```

以类似的方式，要卸载某一部分的内容，请从部分元实体中移除组件 Unity.Entities.RequestSceneLoaded。
