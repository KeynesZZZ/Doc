---
title: 【代码片段】YooAsset常用API速查
tags: ["Unity", "第三方库", "YooAsset", "代码片段", "API"]
category: 第三方库
created: 2026-07-02
updated: 2026-07-02
description: YooAsset 高频 API 速查手册：初始化/资源加载/实例化释放/场景加载/文件下载/资源信息，按场景组织、复制即用
unity_version: 2021.3+
status: 待验证
validation: 基于官方文档与工程实践整理
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【笔记】YooAsset热更新与版本管理]]", "[[【踩坑】YooAsset中大型项目踩坑实录]]"]
author: llm
sources:
  - "YooAsset 官方文档 https://www.yooasset.com/"
  - "YooAsset GitHub https://github.com/tuyoogame/YooAsset"
  - "[[【笔记】YooAsset核心概念与架构]]"
---

# 【代码片段】YooAsset 常用 API 速查

> 按场景组织的 YooAsset API 字典，复制即用。概念解释见 [[【笔记】YooAsset核心概念与架构]]。

## 文档定位

日常开发中的 API 参考，覆盖 8 个高频场景。每个代码块独立可运行。

---

## 1. 初始化

### 1.1 全局初始化

```csharp
using YooAsset;

// 在游戏启动时调用一次
YooAssets.Initialize();

// 如果需要重新初始化，先销毁
YooAssets.DestroyAllPackages();
YooAssets.Initialize();
```

### 1.2 创建 Package 并设置默认

```csharp
// 创建资源包实例
var package = YooAssets.CreatePackage("DefaultPackage");

// 设为默认包（后续可通过 YooAssets.GetPackage() 快速获取）
YooAssets.SetDefaultPackage(package);
```

### 1.3 初始化（四种模式）

```csharp
// 模式 1：编辑器模拟模式（开发阶段）
var editorParams = new EditorSimulateModeParameters();
await package.InitializeAsync(editorParams);

// 模式 2：离线模式（单机游戏 / 首包）
var offlineParams = new OfflinePlayModeParameters();
offlineParams.BuildinFileSystemParameters =
    FileSystemParameters.CreateDefaultBuildinFileSystemParameters();
await package.InitializeAsync(offlineParams);

// 模式 3：在线模式（需要热更）
var hostParams = new HostPlayModeParameters();
hostParams.BuildinFileSystemParameters =
    FileSystemParameters.CreateDefaultBuildinFileSystemParameters();
hostParams.CacheFileSystemParameters =
    FileSystemParameters.CreateDefaultCacheFileSystemParameters();
hostParams.RemoteServices = new MyRemoteServices();
await package.InitializeAsync(hostParams);

// 模式 4：Web 模式（WebGL / 小游戏）
var webParams = new WebPlayModeParameters();
webParams.RemoteServices = new MyRemoteServices();
await package.InitializeAsync(webParams);
```

### 1.4 IRemoteServices 实现

```csharp
public class MyRemoteServices : IRemoteServices
{
    private readonly string _mainHost = "http://your-cdn.com/bundles";
    private readonly string _fallbackHost = "http://backup-cdn.com/bundles";

    public string GetRemoteMainURL(string fileName)
    {
        return $"{_mainHost}/{fileName}";
    }

    public string GetRemoteFallbackURL(string fileName)
    {
        return $"{_fallbackHost}/{fileName}";
    }
}
```

---

## 2. 资源加载

### 2.1 异步加载（推荐）

```csharp
// 加载 GameObject
var handle = package.LoadAssetAsync<GameObject>("Assets/Prefabs/Player.prefab");
await handle.ToUniTask();
if (handle.Status == EOperationStatus.Succeed)
{
    GameObject prefab = handle.AssetObject as GameObject;
    Instantiate(prefab);
}

// 加载 Sprite
var spriteHandle = package.LoadAssetAsync<Sprite>("Assets/Icons/Avatar.png");
await spriteHandle.ToUniTask();
Sprite sprite = spriteHandle.AssetObject as Sprite;

// 加载 TextAsset
var textHandle = package.LoadAssetAsync<TextAsset>("Assets/Configs/items.json");
await textHandle.ToUniTask();
string jsonText = (textHandle.AssetObject as TextAsset).text;

// 加载 AudioClip
var audioHandle = package.LoadAssetAsync<AudioClip>("Assets/Audio/bgm.mp3");
await audioHandle.ToUniTask();
AudioClip clip = audioHandle.AssetObject as AudioClip;
```

### 2.2 同步加载

```csharp
// 注意：同步加载在内部可能仍为异步，会阻塞直到完成
var handle = package.LoadAssetSync<GameObject>("Assets/Prefabs/Enemy.prefab");
if (handle.Status == EOperationStatus.Succeed)
{
    Instantiate(handle.AssetObject);
}
```

### 2.3 批量加载

```csharp
// 通过资源标签批量加载
var handles = package.LoadAssetsAsync(new string[] {
    "Assets/Icons/Icon_01.png",
    "Assets/Icons/Icon_02.png",
    "Assets/Icons/Icon_03.png"
});

await handles.ToUniTask();
// handles.AllAssetObjects 获取所有加载的资源
```

### 2.4 加载子资源（Sprite Atlas / FBX 子物体）

```csharp
// 加载 Sprite Atlas 中的子精灵
var handle = package.LoadSubAssetsAsync<Sprite>("Assets/Atlas/UIAtlas.spriteatlas");
await handle.ToUniTask();
Sprite[] subSprites = handle.AllAssetObjects as Sprite[];
```

---

## 3. 实例化与释放

### 3.1 实例化

```csharp
// 异步实例化
var handle = package.LoadAssetAsync<GameObject>("Assets/Prefabs/UIPanel.prefab");
await handle.ToUniTask();

// 方式 1：手动实例化
GameObject instance = Instantiate(handle.AssetObject as GameObject);
instance.transform.SetParent(canvas.transform, false);

// 方式 2：通过 handle 实例化（自动跟踪）
// handle.InstantiateAsync() — 推荐方式，便于统一管理
```

### 3.2 释放（关键！）

```csharp
// 规则：谁 Load 谁 Release
// 错误：只 Destroy 实例，不 Release handle
Destroy(instance);       // 只销毁了场景实例
// handle.Release() 未调用 → 内存泄漏！

// 正确：先销毁实例，再释放 handle
Destroy(instance);
handle.Release();        // 引用计数 -1，计数归 0 时资源被回收
```

### 3.3 强制卸载

```csharp
// 卸载未使用的资源（类似 Resources.UnloadUnusedAssets）
package.UnloadUnusedAssets();

// 强制卸载所有资源（场景切换时常用）
package.UnloadAllAssets();
```

---

## 4. 场景加载

```csharp
using UnityEngine.SceneManagement;

// 异步加载场景
var handle = package.LoadSceneAsync("Assets/Scenes/Battle.unity");
await handle.ToUniTask();

if (handle.Status == EOperationStatus.Succeed)
{
    // 场景加载完成
    Debug.Log("场景加载成功");
}

// 卸载场景
handle.UnloadScene();
handle.Release();
```

### 场景加载模式

```csharp
// LoadSceneMode.Additive — 叠加加载
var handle = package.LoadSceneAsync(
    "Assets/Scenes/UI.unity",
    LoadSceneMode.Additive
);

// 指定局部物理场景（用于多场景物理隔离）
var handle = package.LoadSceneAsync(
    "Assets/Scenes/Battle.unity",
    LoadSceneMode.Additive,
    LocalPhysicsMode.Physics3D
);
```

---

## 5. 资源文件下载（热更流程）

> 完整热更流程见 [[【笔记】YooAsset热更新与版本管理]]。

### 5.1 获取版本号

```csharp
// 请求远端版本号
var versionOp = package.RequestPackageVersionAsync();
await versionOp.ToUniTask();

if (versionOp.Status == EOperationStatus.Succeed)
{
    string packageVersion = versionOp.PackageVersion;
    Debug.Log($"远端版本: {packageVersion}");
}
```

### 5.2 更新 Manifest

```csharp
// 根据版本号更新 Manifest
var manifestOp = package.UpdatePackageManifestAsync(packageVersion);
await manifestOp.ToUniTask();

if (manifestOp.Status == EOperationStatus.Succeed)
{
    Debug.Log("Manifest 更新成功");
}
```

### 5.3 创建下载器

```csharp
// 比对本地与远端，获取需要下载的资源列表
int downloadingMax = 10;   // 最大并发下载数
int failedRetry = 3;       // 失败重试次数
var downloader = package.CreateResourceDownloader(downloadingMax, failedRetry);

// 检查是否需要下载
if (downloader.TotalDownloadCount == 0)
{
    Debug.Log("无需下载，已是最新版本");
    return;
}

Debug.Log($"需要下载 {downloader.TotalDownloadBytes} 字节，" +
         $"共 {downloader.TotalDownloadCount} 个文件");
```

### 5.4 下载回调与开始下载

```csharp
// 注册回调
downloader.OnStart += () => { Debug.Log("下载开始"); };
downloader.OnDownloadProgressBegin += (context) => {
    Debug.Log($"开始下载: {context.FileName}");
};
downloader.OnDownloadProgressUpdate += (context) => {
    float progress = (float)context.DownloadedBytes / context.TotalDownloadBytes;
    Debug.Log($"进度: {progress:P1}");
};
downloader.OnDownloadFileFinished += (context) => {
    Debug.Log($"文件完成: {context.FileName}");
};
downloader.OnDownloadError += (context) => {
    Debug.LogError($"下载失败: {context.FileName}, 错误: {context.Error}");
};

// 开始下载
downloader.BeginDownload();
await downloader.ToUniTask();

if (downloader.Status == EOperationStatus.Succeed)
{
    Debug.Log("全部下载完成");
}
```

---

## 6. 获取资源信息

### 6.1 检查资源是否存在

```csharp
bool exists = package.CheckLocationValid("Assets/Prefabs/Player.prefab");
if (exists)
{
    Debug.Log("资源存在");
}
```

### 6.2 获取资源信息

```csharp
// 获取单个资源信息
AssetInfo assetInfo = package.GetAssetInfo("Assets/Prefabs/Player.prefab");
Debug.Log($"地址: {assetInfo.Address}");
Debug.Log($"类型: {assetInfo.AssetType}");

// 按标签获取资源信息列表
AssetInfo[] assetInfos = package.GetAssetInfos("UI");
foreach (var info in assetInfos)
{
    Debug.Log($"标签匹配: {info.Address}");
}
```

### 6.3 获取原始文件路径（RawFile 模式）

```csharp
// 用于非 AssetBundle 打包的原始文件（如视频、音频）
var handle = package.LoadRawFileAsync("Assets/Videos/intro.mp4");
await handle.ToUniTask();

// 获取文件路径
string filePath = handle.GetRawFileData();
byte[] fileBytes = handle.GetRawFileData();
```

---

## 7. 解密接口实现

> 打包加密配置见 [[【笔记】YooAsset打包管线与分包策略]]。

```csharp
using YooAsset;

// 运行时解密实现
public class FileOffsetDecryption : IDecryptionServices
{
    public ulong LoadFromFileOffset(DecryptionFileInfo fileInfo)
    {
        // 返回文件偏移量（与打包时的加密偏移一致）
        return 32; // 跳过前 32 字节的加密头
    }

    public byte[] LoadFromMemory(DecryptionFileInfo fileInfo)
    {
        // 内存解密（读取文件后解密返回明文）
        byte[] encryptedData = File.ReadAllBytes(fileInfo.FilePath);
        byte[] decryptedData = DecryptAES(encryptedData);
        return decryptedData;
    }

    public System.IO.Stream LoadFromStream(DecryptionFileInfo fileInfo)
    {
        // 流式解密（适用于大文件）
        var stream = new System.IO.FileStream(
            fileInfo.FilePath,
            System.IO.FileMode.Open,
            System.IO.FileAccess.Read
        );
        stream.Seek(32, System.IO.SeekOrigin.Begin); // 跳过加密头
        return stream;
    }

    private byte[] DecryptAES(byte[] data)
    {
        // 实现 AES 解密逻辑
        // ...
        return data;
    }
}

// 初始化时传入解密服务
var initParams = new HostPlayModeParameters();
initParams.CacheFileSystemParameters =
    FileSystemParameters.CreateDefaultCacheFileSystemParameters(new FileOffsetDecryption());
```

---

## 8. Package 管理

### 8.1 多 Package 管理

```csharp
// 创建多个 Package
var defaultPackage = YooAssets.CreatePackage("DefaultPackage");
var activityPackage = YooAssets.CreatePackage("ActivityPackage");
var texturePackage = YooAssets.CreatePackage("HDTexturePackage");

// 分别初始化...
YooAssets.SetDefaultPackage(defaultPackage);

// 按名称获取
var pkg = YooAssets.GetPackage("ActivityPackage");
```

### 8.2 销毁 Package

```csharp
// 销毁指定 Package
YooAssets.DestroyPackage("ActivityPackage");

// 销毁所有 Package
YooAssets.DestroyAllPackages();
```

### 8.3 清理缓存

```csharp
var package = YooAssets.GetPackage("DefaultPackage");

// 清理所有 Bundle 文件（慎用！会删除所有已下载资源）
var clearOp = package.ClearAllBundleFilesAsync();
await clearOp.ToUniTask();

// 清理未使用的 Bundle 文件（推荐，安全）
var clearUnusedOp = package.ClearUnusedBundleFilesAsync();
await clearUnusedOp.ToUniTask();
```

---

## 速查表

| 场景 | 核心 API | 注意事项 |
|------|----------|----------|
| 初始化 | `YooAssets.Initialize()` → `CreatePackage()` → `InitializeAsync()` | 全局只调一次 |
| 异步加载 | `LoadAssetAsync<T>()` | 推荐，配合 UniTask |
| 同步加载 | `LoadAssetSync<T>()` | 可能阻塞主线程 |
| 释放 | `handle.Release()` | 必须 Load/Release 配对 |
| 场景 | `LoadSceneAsync()` → `UnloadScene()` | 卸载后 Release |
| 版本检查 | `RequestPackageVersionAsync()` | HostPlay 模式必需 |
| Manifest 更新 | `UpdatePackageManifestAsync()` | 版本检查后调用 |
| 下载 | `CreateResourceDownloader()` → `BeginDownload()` | 支持断点续传 |
| 强制卸载 | `UnloadUnusedAssets()` / `UnloadAllAssets()` | 场景切换时 |

---

## 相关文档

- [[【笔记】YooAsset核心概念与架构]] — 概念解释与架构总览
- [[【笔记】YooAsset热更新与版本管理]] — 完整热更流程详解
- [[【笔记】YooAsset打包管线与分包策略]] — 打包配置与加密
- [[【踩坑】YooAsset中大型项目踩坑实录]] — 常见错误与解决方案

## 官方参考

- [YooAsset 官方文档](https://www.yooasset.com/)
- [YooAsset Code Tutorial](https://www.yooasset.com/docs/guide-runtime/CodeTutorial1)
- [YooAsset GitHub](https://github.com/tuyoogame/YooAsset)
