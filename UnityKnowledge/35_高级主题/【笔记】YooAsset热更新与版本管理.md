---
title: 【笔记】YooAsset热更新与版本管理
tags: ["Unity", "YooAsset", "热更新", "版本管理", "CDN"]
category: 高级主题
created: 2026-07-02
updated: 2026-07-02
description: YooAsset 热更新全链路：版本检查→Manifest 更新→差异下载→断点续传→CDN 部署规范→灰度发布→边界情况处理
unity_version: 2021.3+
status: 待验证
validation: 基于官方文档与工程实践整理
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【代码片段】YooAsset常用API速查]]", "[[【笔记】YooAsset打包管线与分包策略]]", "[[【踩坑】YooAsset中大型项目踩坑实录]]", "[[【笔记】YooAsset与HybridCLR协同方案]]"]
author: llm
sources:
  - "YooAsset 官方文档 https://www.yooasset.com/"
  - "YooAsset Code Tutorial https://www.yooasset.com/docs/guide-runtime/CodeTutorial2"
  - "[[【笔记】YooAsset核心概念与架构]]"
  - "[[【笔记】YooAsset打包管线与分包策略]]"
---

# 【笔记】YooAsset 热更新与版本管理

> 从 0 到 1 搭建 YooAsset 热更新流程：版本检查 → Manifest 更新 → 差异下载 → 断点续传，包含 CDN 部署规范和完整代码示例。

## 文档定位

- **概念基础**：[[【笔记】YooAsset核心概念与架构]] 介绍了 HostPlayMode 模式
- **打包前置**：[[【笔记】YooAsset打包管线与分包策略]] 说明了构建产物结构
- **本文**：聚焦运行时热更流程的完整实现

---

## 一、热更流程全链路图

```
App 启动
  │
  ▼
YooAssets 初始化 (HostPlayMode)
  │
  ▼
请求远端版本号 (RequestPackageVersionAsync)
  │
  ├─ 版本号一致 → 直接加载资源
  │
  ▼
版本号不一致 → 更新 Manifest (UpdatePackageManifestAsync)
  │
  ▼
创建下载器 (CreateResourceDownloader)
  │
  ├─ 无需下载 → 加载资源
  │
  ▼
开始下载 (BeginDownload)
  │  ├─ 支持断点续传
  │  ├─ 支持多线程并发
  │  └─ 支持失败重试
  │
  ▼
下载完成 → 加载资源
  │
  ▼
保存版本号到本地（下次启动跳过版本检查）
```

---

## 二、版本号机制

### 2.1 两种版本号

| 版本号 | 来源 | 用途 |
|--------|------|------|
| StaticVersion | 构建时生成 | 用于判断 Manifest 是否需要更新 |
| PackageVersion | Manifest 内部记录 | 用于判断资源内容是否需要更新 |

### 2.2 版本检查流程

```csharp
// 1. 获取本地保存的版本号
string localVersion = PlayerPrefs.GetString("YooAsset_Version", "");

// 2. 请求远端版本号
var versionOp = package.RequestPackageVersionAsync();
await versionOp.ToUniTask();

if (versionOp.Status != EOperationStatus.Succeed)
{
    Debug.LogError($"获取版本号失败: {versionOp.Error}");
    // 处理网络错误：使用本地资源继续运行
    return;
}

string remoteVersion = versionOp.PackageVersion;

// 3. 比较版本号
if (localVersion == remoteVersion)
{
    Debug.Log("版本一致，无需更新");
    // 直接加载资源
}
else
{
    Debug.Log($"版本不一致: {localVersion} → {remoteVersion}");
    // 执行热更流程
}
```

---

## 三、增量更新机制

### 3.1 差异比对

YooAsset 通过 Manifest 记录每个 Bundle 的哈希值。热更时：

```
本地 Manifest                    远端 Manifest
├── BundleA (hash: abc123)       ├── BundleA (hash: abc123)  ← 一致，跳过
├── BundleB (hash: def456)       ├── BundleB (hash: xyz789)  ← 不一致，下载
├── BundleC (hash: ghi789)       ├── BundleC (hash: ghi789)  ← 一致，跳过
└── BundleD (hash: jkl012)       └── BundleE (hash: mno345)  ← 新增，下载
```

### 3.2 下载器工作原理

```csharp
// YooAsset 内部流程（伪代码）：
// 1. 比对本地 Manifest 与远端 Manifest
// 2. 找出需要下载的 Bundle 列表
// 3. 为每个 Bundle 创建下载任务
// 4. 并发下载（受 maxConcurrentCount 限制）
// 5. 支持失败重试（受 failedRetryCount 限制）
// 6. 下载完成后写入本地缓存
```

---

## 四、断点续传

### 4.1 实现

YooAsset 的下载器原生支持断点续传：

```csharp
// 创建下载器
var downloader = package.CreateResourceDownloader(
    downloadingMax: 10,    // 最大并发数
    failedRetry: 3         // 失败重试次数
);

// 注册回调
downloader.OnDownloadError += (context) => {
    Debug.LogError($"下载失败: {context.FileName}");
    // 失败后 YooAsset 会自动重试，重试次数用尽才报错
};

// 开始下载
downloader.BeginDownload();
await downloader.ToUniTask();

// 如果下载中断（网络断开 / App 杀死），下次启动时：
// 1. YooAsset 检查本地缓存中的半成品文件
// 2. 对比已下载大小与远端文件大小
// 3. 从断点继续下载
```

### 4.2 断点续传注意事项

```csharp
// 检查是否有未完成的下载
var downloader = package.CreateResourceDownloader(10, 3);
if (downloader.TotalDownloadCount > 0)
{
    // 有未完成的下载，继续
    Debug.Log($"继续下载 {downloader.TotalDownloadCount} 个文件");
    downloader.BeginDownload();
    await downloader.ToUniTask();
}
```

> **注意**：断点续传依赖 CDN 支持 HTTP Range 请求。详见 [[【踩坑】YooAsset中大型项目踩坑实录]]。

---

## 五、CDN 部署规范

### 5.1 目录结构

```
CDN 根目录 /
├── android/
│   ├── DefaultPackage/
│   │   ├── v1.0.0/
│   │   │   ├── manifest.hash         ← 版本哈希（客户端首先请求）
│   │   │   ├── manifest.bytes        ← Manifest 数据
│   │   │   └── bundles/
│   │   │       ├── 5f3a2b.bundle
│   │   │       └── ...
│   │   └── v1.0.1/
│   │       ├── manifest.hash
│   │       ├── manifest.bytes
│   │       └── bundles/
│   │           ├── 5f3a2b.bundle     ← 未变文件（客户端跳过）
│   │           ├── 9x8y7z.bundle     ← 新增/变更文件
│   │           └── ...
│   ├── ActivityPackage/
│   │   └── v1.0.0/
│   └── HDTexturePackage/
│       └── v1.0.0/
├── ios/
│   └── DefaultPackage/
│       └── v1.0.0/
└── webgl/
    └── DefaultPackage/
        └── v1.0.0/
```

### 5.2 多节点容灾

```csharp
// IRemoteServices 实现多节点 fallback
public class MultiNodeRemoteServices : IRemoteServices
{
    private readonly string[] _cdnNodes = {
        "http://primary-cdn.com/bundles",
        "http://backup-cdn.com/bundles",
        "http://failover-cdn.com/bundles"
    };
    private int _currentNodeIndex = 0;

    public string GetRemoteMainURL(string fileName)
    {
        return $"{_cdnNodes[_currentNodeIndex]}/{fileName}";
    }

    public string GetRemoteFallbackURL(string fileName)
    {
        int fallbackIndex = (_currentNodeIndex + 1) % _cdnNodes.Length;
        return $"{_cdnNodes[fallbackIndex]}/{fileName}";
    }

    public void SwitchToNextNode()
    {
        _currentNodeIndex = (_currentNodeIndex + 1) % _cdnNodes.Length;
        Debug.LogWarning($"切换 CDN 节点: {_cdnNodes[_currentNodeIndex]}");
    }
}
```

### 5.3 灰度发布

```
灰度发布策略：
1. 全量用户 → v1.0.0
2. 灰度 10% 用户 → v1.0.1
   └── 客户端请求时带 User-ID / Device-ID
   └── CDN 服务端根据 ID 返回对应版本
3. 灰度 50% → v1.0.1
4. 全量 → v1.0.1
5. 旧版本 v1.0.0 保留 7 天后清理
```

---

## 六、完整代码示例：热更检查 + 下载 + 进度展示

```csharp
using UnityEngine;
using UnityEngine.UI;
using YooAsset;
using Cysharp.Threading.Tasks;

public class HotUpdateManager : MonoBehaviour
{
    public Slider progressSlider;
    public Text statusText;
    public Text detailText;

    private ResourcePackage _package;

    private async UniTaskVoid Start()
    {
        // 初始化
        YooAssets.Initialize();
        _package = YooAssets.CreatePackage("DefaultPackage");
        YooAssets.SetDefaultPackage(_package);

        var initParams = new HostPlayModeParameters();
        initParams.BuildinFileSystemParameters =
            FileSystemParameters.CreateDefaultBuildinFileSystemParameters();
        initParams.CacheFileSystemParameters =
            FileSystemParameters.CreateDefaultCacheFileSystemParameters();
        initParams.RemoteServices = new RemoteServices();

        var initOp = _package.InitializeAsync(initParams);
        await initOp.ToUniTask();

        if (initOp.Status != EOperationStatus.Succeed)
        {
            statusText.text = "初始化失败";
            return;
        }

        // 开始热更
        await RunHotUpdate();
    }

    private async UniTask RunHotUpdate()
    {
        statusText.text = "检查版本...";

        // 1. 请求远端版本号
        var versionOp = _package.RequestPackageVersionAsync();
        await versionOp.ToUniTask();

        if (versionOp.Status != EOperationStatus.Succeed)
        {
            statusText.text = "版本检查失败，使用本地资源";
            return;
        }

        string packageVersion = versionOp.PackageVersion;

        // 2. 更新 Manifest
        statusText.text = "更新资源清单...";
        var manifestOp = _package.UpdatePackageManifestAsync(packageVersion);
        await manifestOp.ToUniTask();

        if (manifestOp.Status != EOperationStatus.Succeed)
        {
            statusText.text = "Manifest 更新失败";
            return;
        }

        // 3. 创建下载器
        var downloader = _package.CreateResourceDownloader(
            downloadingMax: 10,
            failedRetry: 3
        );

        if (downloader.TotalDownloadCount == 0)
        {
            statusText.text = "已是最新版本";
            progressSlider.value = 1f;
            OnUpdateComplete();
            return;
        }

        // 显示下载信息
        long totalMB = downloader.TotalDownloadBytes / 1024 / 1024;
        statusText.text = $"需要下载 {totalMB} MB";

        // 4. 注册下载回调
        downloader.OnStart += () => {
            statusText.text = "开始下载...";
        };

        downloader.OnDownloadProgressUpdate += (context) => {
            float progress = (float)context.DownloadedBytes / context.TotalDownloadBytes;
            progressSlider.value = progress;
            detailText.text = $"{context.FileName} " +
                $"({context.DownloadedBytes / 1024}KB / {context.TotalDownloadBytes / 1024}KB)";
        };

        downloader.OnDownloadError += (context) => {
            Debug.LogError($"下载错误: {context.FileName}, {context.Error}");
        };

        // 5. 开始下载
        downloader.BeginDownload();
        await downloader.ToUniTask();

        if (downloader.Status == EOperationStatus.Succeed)
        {
            // 6. 保存版本号
            PlayerPrefs.SetString("YooAsset_Version", packageVersion);
            statusText.text = "更新完成";
            progressSlider.value = 1f;
            OnUpdateComplete();
        }
        else
        {
            statusText.text = "下载失败，请检查网络";
        }
    }

    private void OnUpdateComplete()
    {
        // 热更完成，加载游戏
        Debug.Log("热更新完成，进入游戏");
    }
}

// CDN 远端服务
public class RemoteServices : IRemoteServices
{
    private readonly string _host = "http://your-cdn.com/android/DefaultPackage";

    public string GetRemoteMainURL(string fileName)
    {
        return $"{_host}/{fileName}";
    }

    public string GetRemoteFallbackURL(string fileName)
    {
        return $"{_host}/{fileName}";
    }
}
```

---

## 七、边界情况处理

### 7.1 网络中断

```csharp
downloader.OnDownloadError += (context) => {
    // 记录失败信息
    Debug.LogError($"下载失败: {context.FileName}");

    // YooAsset 会自动重试（次数为 CreateResourceDownloader 的 failedRetry 参数）
    // 重试次数用尽后，downloader.Status 会变为 Failed
};

// 下载完成后检查状态
if (downloader.Status != EOperationStatus.Succeed)
{
    // 提示用户网络异常，提供"重试"按钮
    // 重试时重新调用 CreateResourceDownloader → BeginDownload
    // YooAsset 会自动跳过已完成的文件，从未完成处继续
}
```

### 7.2 磁盘空间不足

```csharp
// 在下载前检查磁盘空间
long availableSpace = GetAvailableDiskSpace();
long needDownload = downloader.TotalDownloadBytes;

if (needDownload > availableSpace)
{
    long needMB = needDownload / 1024 / 1024;
    long availableMB = availableSpace / 1024 / 1024;
    Debug.LogError($"磁盘空间不足: 需要 {needMB}MB, 可用 {availableMB}MB");
    // 提示用户清理空间
    return;
}

// 获取磁盘可用空间（平台相关）
long GetAvailableDiskSpace()
{
    // Android: 使用 android.os.StatFs
    // iOS: 使用 NSFileManager
    // 需要 Native Plugin 实现
    return long.MaxValue; // 临时返回
}
```

### 7.3 版本回退

```csharp
// 当新版本有问题需要回退时
// YooAsset 不支持自动回退，但可以通过以下方式处理：

// 方案 1：清理缓存，使用内置版本
package.ClearAllBundleFilesAsync();
// 重新初始化为 OfflinePlayMode（使用首包内置资源）

// 方案 2：CDN 上保留旧版本
// 通过服务端接口获取指定版本号
string targetVersion = "1.0.0"; // 回退到旧版本
var versionOp = package.RequestPackageVersionAsync();
// 需要自定义 IRemoteServices 返回旧版本目录
```

### 7.4 离线模式与在线模式切换

```csharp
// 检测网络状态后选择模式
if (Application.internetReachability == NetworkReachability.NotReachable)
{
    // 无网络 — 使用离线模式（内置资源）
    var offlineParams = new OfflinePlayModeParameters();
    offlineParams.BuildinFileSystemParameters =
        FileSystemParameters.CreateDefaultBuildinFileSystemParameters();
    await package.InitializeAsync(offlineParams);
}
else
{
    // 有网络 — 使用在线模式（支持热更）
    var hostParams = new HostPlayModeParameters();
    // ... 配置在线模式参数
    await package.InitializeAsync(hostParams);
}
```

---

## 八、热更新流程 Checklist

- [ ] 初始化 YooAssets 并创建 Package
- [ ] 配置 HostPlayMode + IRemoteServices
- [ ] 请求远端版本号
- [ ] 更新 Manifest
- [ ] 创建下载器并检查是否有更新
- [ ] 注册下载进度回调
- [ ] 处理下载失败（重试 + 提示）
- [ ] 下载完成后保存版本号
- [ ] 处理边界情况（网络中断 / 磁盘不足 / 版本回退）

---

## 相关文档

- [[【笔记】YooAsset核心概念与架构]] — 四种运行模式概览
- [[【代码片段】YooAsset常用API速查]] — 下载相关 API
- [[【笔记】YooAsset打包管线与分包策略]] — 构建产物结构
- [[【踩坑】YooAsset中大型项目踩坑实录]] — 热更相关坑
- [[【笔记】YooAsset与HybridCLR协同方案]] — 代码+资源双热更

## 官方参考

- [YooAsset Code Tutorial: 资源更新](https://www.yooasset.com/docs/guide-runtime/CodeTutorial2)
- [YooAsset Code Tutorial: 资源清理](https://www.yooasset.com/docs/guide-runtime/CodeTutorial3)
- [YooAsset GitHub](https://github.com/tuyoogame/YooAsset)
