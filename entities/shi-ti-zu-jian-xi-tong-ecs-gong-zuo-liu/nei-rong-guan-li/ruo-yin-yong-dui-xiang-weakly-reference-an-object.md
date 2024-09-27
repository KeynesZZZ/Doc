# 弱引用对象（Weakly reference an object）

#### 弱引用对象

弱引用对象是一个对象句柄，无论对象是否已加载或卸载，该句柄都是有效的。如果你对一个对象创建弱引用，Unity 会将该对象包括在内容存档中，使其在运行时可用。然后你可以使用 `RuntimeContentManager` API 来加载、使用和释放这些弱引用对象。

内容管理系统提供通过检查器（Inspector）和 C# 脚本对对象进行弱引用的方法。

**通过检查器弱引用对象**

`WeakObjectReference` 结构体为负责管理弱引用对象的 `RuntimeContentManager` API 提供了一个包装器。它还使得可以通过检查器创建对象的弱引用。`WeakObjectReference` 的检查器属性绘制器是一个对象字段，你可以将对象拖放到这个字段上。在内部，Unity 会生成你分配的对象的弱引用，然后你可以在烘焙过程中将其传递给 ECS 组件。

`WeakObjectReference` 包装器还简化了运行时对单个弱引用对象的管理。它提供了加载、使用和释放其弱引用对象的方法和属性。

以下代码示例展示了如何创建一个 `Baker`，将网格资产的 `WeakObjectReference` 传递给非托管组件。网格属性作为对象字段出现在 `MeshRefSample` 组件的检查器中，你可以将网格资产分配给该字段。

```csharp
using Unity.Entities;
using Unity.Entities.Content;
using UnityEngine;

public class MeshRefSample : MonoBehaviour
{
    public WeakObjectReference<Mesh> mesh;

    class MeshRefSampleBaker : Baker<MeshRefSample>
    {
        public override void Bake(MeshRefSample authoring)
        {
            var entity = GetEntity(TransformUsageFlags.Dynamic);
            AddComponent(entity, new MeshComponentData { mesh = authoring.mesh });
        }
    }
}

public struct MeshComponentData : IComponentData
{
    public WeakObjectReference<Mesh> mesh;
}
```

#### 从 C# 脚本弱引用对象

`RuntimeContentManager` API 通过 `UntypedWeakReferenceId` 管理弱引用对象。

以下代码示例展示了如何获取当前在项目窗口中选定对象的 `UntypedWeakReferenceId`。要使 Unity 将这些对象包括在内容存档中，你必须将弱引用 ID 烘焙到 ECS 组件中。为了将编辑器脚本中创建的弱引用 ID 传递给 baker，你可以使用一个 ScriptableObject。编辑器脚本可以将弱引用 ID 写入 ScriptableObject，然后在烘焙过程中，baker 可以从 ScriptableObject 中读取这些 ID 并将它们写入 ECS 组件。

**示例代码：获取选定对象的 `UntypedWeakReferenceId`**

```csharp
using UnityEngine;
using UnityEditor;
using Unity.Entities.Serialization;

public static class ContentManagementEditorUtility
{
    [MenuItem("Content Management/Log UntypedWeakReferenceId of Selected")]
    private static void LogWeakReferenceIDs()
    {
        Object[] selectedObjects = Selection.GetFiltered(typeof(Object), SelectionMode.Assets);
        for (int i = 0; i < selectedObjects.Length; i++)
        {
            Debug.Log($"{selectedObjects[i].name}: {UntypedWeakReferenceId.CreateFromObjectInstance(selectedObjects[i])}");
        }
    }
}
```
