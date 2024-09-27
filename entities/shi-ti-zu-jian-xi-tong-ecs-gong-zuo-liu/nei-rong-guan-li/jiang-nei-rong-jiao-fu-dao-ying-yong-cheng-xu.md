# 将内容交付到应用程序

#### 将内容交付到应用程序

为了向你的应用程序提供内容更新，你可以创建自定义内容存档，并从内容交付服务或本地设备加载它们。`ContentDeliveryService` APIs 提供了一个统一的工作流，用于按需加载本地和在线内容。你可以使用它来加载内容、检查交付状态，并根据可用内容有条件地运行代码。

**设置内容交付**

要启用项目的内容交付，请设置 `ENABLE_CONTENT_DELIVERY` 脚本符号。有关如何执行此操作的信息，请参阅自定义脚本符号。

**加载内容**

`ContentDeliveryService` APIs 通过 URL 加载内容。你可以传递存储在在线内容交付服务上的内容 URL，或设备上内容存档的本地 URL。在开始下载内容之后，必须等到 Unity 完成安装和缓存这些内容。然后，可以像访问本地内容一样，通过弱引用 ID 从已安装的内容中加载对象。有关更多信息，请参阅在运行时加载弱引用对象。

以下代码示例展示了如何通过 URL 加载内容、等待内容交付完成，然后继续应用程序逻辑。由于示例代码仅在应用程序启动前执行一次完整的内容更新，因此可以使用 `MonoBehaviour` 而不是系统。你也可以使用系统的更新方法来完成此操作，但内容交付系统 API 并未进行 Burst 编译，无法从作业中使用这些 API，因此没有性能优势。

**示例代码：内容交付**

```csharp
using System;
using Unity.Entities.Content;
using UnityEngine;

public class GameStarter : MonoBehaviour
{
    public string remoteUrlRoot;
    public string initialContentSet;

    void Start()
    {
#if ENABLE_CONTENT_DELIVERY
        ContentDeliveryGlobalState.Initialize(remoteUrlRoot, Application.persistentDataPath + "/content-cache", initialContentSet, s =>
        {
            if (s >= ContentDeliveryGlobalState.ContentUpdateState.ContentReady)
                LoadMainScene();
        });
#else
        LoadMainScene();
#endif
    }

    void LoadMainScene()
    {
        // 在这里内容已经准备好，可以加载主场景...
    }
}
```
