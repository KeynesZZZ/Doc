# 创建自定义内容档案

#### 创建自定义内容档案

Unity 自动创建内容存档（content archive）以存储构建中引用的所有对象。这涵盖了许多应用程序的内容管理需求，但你也可以创建其他内容存档，以便在运行时独立加载。这对于结构化可下载内容、减少应用程序的初始安装大小或加载针对最终用户平台优化的资产非常有用。

创建可以在 Unity 中运行时加载的自定义内容存档的过程包括以下步骤：

1. **构建**：Unity 查找应放入内容存档的所有对象，并将对象文件复制到目标目录。
2. **发布**：Unity 将目录中的所有文件组织成一种结构，使 Unity 能够直接将文件安装到目标平台。为此，Unity 将文件重命名为文件内容的哈希值。然后，它在内容档案旁边构建一个目录，以映射文件到它们的原始对象。

这个过程的结果是用于存储文件的内容档案和一个目录，用于使 Unity 能够找到对象的正确文件。

**从 Unity 菜单重建玩家的内容**

在播放器构建过程中，Unity 自动生成内容存档以存储构建中引用的所有对象。为了提高迭代时间，Unity 可以在不构建新播放器的情况下生成相同的内容存档集。然后，你可以使用新内容存档更新应用程序中的内容。要执行此操作：

1. 选择 `Assets > Publish > Content Update`。这将开始内容存档的构建和发布过程。在完成后，结果内容存档和目录将位于你的 Streaming Assets 文件夹中。

**从 C# 脚本构建内容存档**

如果菜单项无法提供足够的控制来决定将哪些对象包含在自定义内容存档中，你可以使用内容存档构建和发布 API 从 C# 脚本创建内容存档。

以下代码示例展示了如何构建和发布内容更新。

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using Unity.Build.Classic;
using Unity.Collections;
using Unity.Scenes.Editor;
using UnityEditor.Experimental;
using UnityEngine;
using UnityEditor;
using System.Linq;
using Unity.Build;
using Unity.Build.Common;
using Unity.Entities.Build;
using Unity.Entities.Content;

static class BuildUtilities
{
    // 准备要发布的内容文件。通过更改 PublishContent 调用的最后一个参数，可以删除或保留原始文件。
    static void PublishExistingBuild()
    {
        var buildFolder = EditorUtility.OpenFolderPanel("Select Build To Publish",
        Path.GetDirectoryName(Application.dataPath), "Builds");
        if (!string.IsNullOrEmpty(buildFolder))
        {
            var streamingAssetsPath = $"{buildFolder}/{PlayerSettings.productName}_Data/StreamingAssets";
            // 内容集由传递的 functor 定义。
            RemoteContentCatalogBuildUtility.PublishContent(streamingAssetsPath, 
                $"{buildFolder}-RemoteContent", 
                f => new string[] { "all" }, true);
        }
    }

    // 这个方法较为复杂，因为它将从玩家构建中构建场景但不会完全构建玩家。
    static void CreateContentUpdate()
    {
        var buildFolder = EditorUtility.OpenFolderPanel("Select Build To Publish",
        Path.GetDirectoryName(Application.dataPath), "Builds");
        if (!string.IsNullOrEmpty(buildFolder))
        {
            var buildTarget = EditorUserBuildSettings.activeBuildTarget;
            var tmpBuildFolder = Path.Combine(Path.GetDirectoryName(Application.dataPath),
                        $"/Library/ContentUpdateBuildDir/{PlayerSettings.productName}");

            var instance = DotsGlobalSettings.Instance;
            var playerGuid = instance.GetPlayerType() == DotsGlobalSettings.PlayerType.Client ? instance.GetClientGUID() : instance.GetServerGUID();
            if (!playerGuid.IsValid)
                throw new Exception("Invalid Player GUID");

            var subSceneGuids = new HashSet<Unity.Entities.Hash128>();
            for (int i = 0; i < EditorBuildSettings.scenes.Length; i++)
            {
                var ssGuids = EditorEntityScenes.GetSubScenes(EditorBuildSettings.scenes[i].guid);
                foreach (var ss in ssGuids)
                    subSceneGuids.Add(ss);
            }
            RemoteContentCatalogBuildUtility.BuildContent(subSceneGuids, playerGuid, buildTarget, tmpBuildFolder);

            var publishFolder = Path.Combine(Path.GetDirectoryName(Application.dataPath), "Builds", $"{buildFolder}-RemoteContent");
            RemoteContentCatalogBuildUtility.PublishContent(tmpBuildFolder, publishFolder, f => new string[] { "all" });
        }
    }
}
```

#### 交付内容

在创建自定义内容存档（content archive）后，可以在运行时将其交付到应用程序中。内容存档构建和发布过程将内容组织成与本地设备缓存结构相同的形式。这使得内容交付过程更加简单，因为可以直接从本地设备上的内容存档或通过在线内容交付服务加载内容到你的应用程序中。有关如何执行此操作的信息，请参阅将内容交付到应用程序。
