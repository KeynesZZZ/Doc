# 弱引用场景（Weakly reference a scene）

#### 弱引用场景

对场景的弱引用与对对象的弱引用工作方式相同。有关更多信息，请参阅弱引用对象。

Unity 使用 `UntypedWeakReferenceId` 来弱引用场景，因此要从 C# 脚本中弱引用场景，可以使用与从 C# 脚本弱引用对象描述的相同工作流程。

**从检查器弱引用场景**

`RuntimeContentManager` 有一组特定的 API 来在运行时管理弱引用的场景。这意味着 `WeakObjectReference` 包装器不适用于场景。该包装器的场景等效项是 `WeakObjectSceneReference`，它为场景提供了与 `WeakObjectReference` 相同的运行时和编辑器工作流程优势。

要从检查器弱引用场景，请在“从检查器弱引用对象”工作流程中用 `WeakObjectSceneReference` 替换 `WeakObjectReference`。以下代码示例展示了如何做到这一点。

**示例代码：从检查器弱引用场景**

```csharp
using Unity.Entities;
using Unity.Entities.Content;
using UnityEngine;

public class SceneRefSample : MonoBehaviour
{
    public WeakObjectSceneReference scene;

    class SceneRefSampleBaker : Baker<SceneRefSample>
    {
        public override void Bake(SceneRefSample authoring)
        {
            var entity = GetEntity(TransformUsageFlags.Dynamic);
            AddComponent(entity, new SceneComponentData { scene = authoring.scene });
        }
    }
}

public struct SceneComponentData : IComponentData
{
    public WeakObjectSceneReference scene;
}
```
