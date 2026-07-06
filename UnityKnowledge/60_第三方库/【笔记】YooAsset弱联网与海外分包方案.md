---
title: 【笔记】YooAsset弱联网与海外分包方案
tags: ["Unity", "第三方库", "YooAsset", "资源管理", "弱联网", "Android", "热更新"]
category: 第三方库
created: 2026-07-06
updated: 2026-07-06
description: YooAsset 弱联网方案A（离线兜底流程）与 Play Asset Delivery install-time 海外安卓分包方案的实现要点与关键配置
unity_version: 2021.3+
status: 待验证
validation: 基于YooAsset官方文档整理
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【代码片段】YooAsset常用API速查]]"]
author: llm
sources:
  - "YooAsset 弱联网方案A https://www.yooasset.com/docs/solution/WeakNetworkA"
  - "YooAsset 海外安卓分包方案 https://www.yooasset.com/docs/solution/AndroidInstall"
---

# 【笔记】YooAsset 弱联网与海外分包方案

> 本文整理 YooAsset 两个独立但可组合使用的方案：**弱联网方案A**解决离线可玩问题，**海外安卓分包**解决 Google Play 包体限制。

## 文档定位

- **弱联网方案A**：偏单机+热更需求的项目，无网络时跳过更新直接进游戏
- **海外安卓分包**：上 Google Play 的 AAB 包体超限，用 Play Asset Delivery 拆资源

---

## 一、弱联网环境方案A（离线可进游戏）

### 1.1 适用场景

偏单机但有资源热更需求的项目。玩家无网时不希望被卡在资源更新步骤，需要跳过更新直接进游戏。

### 1.2 核心策略：远端优先 → 本地兜底

```
初始化 → 尝试远端更新 → 成功则进游戏
                     ↓ 失败
                  尝试本地清单 → 完整则进游戏
                              ↓ 不完整/失败
                           弹错误框
```

### 1.3 关键初始化参数（决定弱联网能否成立）

| 参数 | 值 | 作用 |
|------|-----|------|
| `CopyBuiltinPackageManifest` | `true` | 初始化时把内置清单拷贝到沙盒，离线才能找到清单 |
| `InstallCleanupMode` | `None` | 覆盖安装时不清掉已拷贝的内置清单，否则老玩家覆盖装后离线不可用 |

```csharp
var builtinFileSystemParams = FileSystemParameters.CreateDefaultBuiltinFileSystemParameters();
// 初始化时拷贝内置清单到沙盒目录
builtinFileSystemParams.AddParameter(EFileSystemParameter.CopyBuiltinPackageManifest, true);

var cacheFileSystemParams = FileSystemParameters.CreateDefaultSandboxFileSystemParameters(remoteServices);
// 覆盖安装时不清理已拷贝的内置清单
cacheFileSystemParams.AddParameter(EFileSystemParameter.InstallCleanupMode, EInstallCleanupMode.None);
```

> **坑点**：这两个参数是方案A能成立的"地基"，缺任一都会在覆盖安装/首装离线场景下出问题。

### 1.4 远端更新流程 `TryUpdateRemotePackage`

```csharp
private IEnumerator TryUpdateRemotePackage(ResourcePackage package, System.Action<bool, string> callback)
{
    // 1. 获取远端最新版本（超时30秒）
    var versionOp = package.RequestPackageVersionAsync(new RequestPackageVersionOptions(true, 30));
    yield return versionOp;
    if (versionOp.Status != EOperationStatus.Succeeded)
    {
        callback.Invoke(false, versionOp.Error);
        yield break;
    }

    // 2. 加载远端清单（超时60秒）
    var manifestOp = package.LoadPackageManifestAsync(new LoadPackageManifestOptions(versionOp.PackageVersion, 60));
    yield return manifestOp;
    if (manifestOp.Status != EOperationStatus.Succeeded)
    {
        callback.Invoke(false, manifestOp.Error);
        yield break;
    }

    // 3. 创建下载器，有需要下载的资源则下载
    var downloader = package.CreateResourceDownloader(new ResourceDownloaderOptions(10, 3));
    if (downloader.TotalDownloadCount > 0)
    {
        downloader.StartDownload();
        yield return downloader;
        if (downloader.Status != EOperationStatus.Succeeded)
        {
            callback.Invoke(false, downloader.Error);
            yield break;
        }
    }

    // 4. 关键：只有下载完整成功后才保存版本号
    PlayerPrefs.SetString("GAME_VERSION", versionOp.PackageVersion);
    PlayerPrefs.Save();

    callback.Invoke(true, string.Empty);
}
```

**契约要点**：版本号只在远端下载**全部成功**后才写入 `PlayerPrefs`，半成品状态不保存版本号，避免下次误用残缺数据。

### 1.5 本地兜底流程 `TryLoadLocalPackage`

```csharp
private IEnumerator TryLoadLocalPackage(ResourcePackage package, System.Action<bool, string> callback)
{
    // 1. 先读上次成功保存的版本号
    string version = PlayerPrefs.GetString("GAME_VERSION", string.Empty);
    if (string.IsNullOrEmpty(version))
    {
        // 首次安装且无网络 → 使用包体内置版本
        var builtinVersionOp = new GetBuildinPackageVersionOperation(package.PackageName);
        OperationHelper.StartOperation(package.PackageName, builtinVersionOp);
        yield return builtinVersionOp;
        if (builtinVersionOp.Status == EOperationStatus.Succeeded)
        {
            version = builtinVersionOp.PackageVersion;
        }
        else
        {
            callback.Invoke(false, builtinVersionOp.Error);
            yield break;
        }
    }

    // 2. 加载本地缓存的清单（HostPlayMode下优先用沙盒里已有的Hash和Manifest）
    var manifestOp = package.LoadPackageManifestAsync(new LoadPackageManifestOptions(version, 60));
    yield return manifestOp;
    if (manifestOp.Status != EOperationStatus.Succeeded)
    {
        callback.Invoke(false, manifestOp.Error);
        yield break;
    }

    // 3. 完整性校验：若仍需下载说明本地资源不完整
    var downloader = package.CreateResourceDownloader(new ResourceDownloaderOptions(1, 1));
    if (downloader.TotalDownloadCount > 0)
    {
        callback.Invoke(false, "本地资源内容不完整。");
        yield break;
    }

    callback.Invoke(true, string.Empty);
}
```

### 1.6 流程入口

```csharp
private IEnumerator Start()
{
    var package = YooAssets.CreatePackage("DefaultPackage");

    // ... 初始化参数设置（见1.3）...
    var initOperation = package.InitializePackageAsync(playModeParameters);
    yield return initOperation;
    if (initOperation.Status != EOperationStatus.Succeeded)
    {
        ShowMessageBox($"资源系统初始化失败：{initOperation.Error}");
        yield break;
    }

    // 优先尝试远端更新
    bool remoteSucceed = false;
    string remoteError = string.Empty;
    yield return TryUpdateRemotePackage(package, (succeed, error) =>
    {
        remoteSucceed = succeed;
        remoteError = error;
    });
    if (remoteSucceed) { StartGame(); yield break; }

    // 远端失败，回退到本地
    bool localSucceed = false;
    string localError = string.Empty;
    yield return TryLoadLocalPackage(package, (succeed, error) =>
    {
        localSucceed = succeed;
        localError = error;
    });
    if (localSucceed) { StartGame(); yield break; }

    ShowMessageBox($"资源更新失败：{remoteError}\n本地资源不可用：{localError}");
}
```

### 1.7 关键设计要点总结

| 要点 | 说明 |
|------|------|
| 版本号持久化时机 | 只在远端下载完整成功后保存，防止"半更"状态被当成可用版本 |
| 本地完整性校验 | 通过 `TotalDownloadCount > 0` 判断，若仍需下载说明离线不可用 |
| 首装离线 | 走 `GetBuildinPackageVersionOperation` 读包体内置版本 |
| HostPlayMode 清单复用 | 优先用沙盒里已有的 Hash/Manifest，本地兜底走同一套 API |

---

## 二、海外安卓分包方案（Play Asset Delivery install-time）

### 2.1 适用场景

上 Google Play，AAB（Android App Bundle）格式要求单个 APK 下载体积有限制。内置资源较大时需要用 Play Asset Delivery（PAD）把资源拆到独立 asset pack。

### 2.2 方案选择：install-time 模式

| 交付模式 | 时机 | 是否需要改业务 |
|---------|------|--------------|
| **install-time**（推荐） | 随应用安装一起下发 | 否，访问方式与 StreamingAssets 一致 |
| fast-follow | 安装后立即开始下载 | 是 |
| on-demand | 运行时按需下载 | 是 |

推荐 **install-time**：安装完成即可用，业务初始化流程无需改动。

### 2.3 核心问题：目录对齐

```
install-time asset pack "assetpack" 解压 → assets/assetpack/
Yoo 默认内置目录                              assets/yoo/
                                         ↑ 不一致，找不到资源
```

### 2.4 两种对齐方式（任选其一）

| 方式 | 操作 | 影响 |
|------|------|------|
| 方式A | 改 `YooAssetSettings.YooFolderName` 为 `assetpack` | 内置资源目录变为 `StreamingAssets/assetpack/` |
| 方式B | 把 asset pack 命名为 `yoo` | 保持 Yoo 默认配置不变 |

> 两者只要对齐即可，无本质区别。

### 2.5 操作步骤

1. 修改 `YooAssetSettings` 资源里的 `YooFolderName` 为 `assetpack`
2. 正常构建资源包，内置资源会落到 `StreamingAssets/assetpack/`
3. 在 Android 工程中配置 install-time 的 asset pack，打包目录为 `assetpack`
4. 以 AAB 形式构建并上传 Google Play

### 2.6 初始化

完成目录对齐后，资源初始化流程**无需特殊处理**，仍使用 `BuiltinFileSystem` 按常规方式读取，与本地运行一致。

### 2.7 注意事项

1. `YooFolderName` 与 asset pack 名称**必须严格一致**，否则无法定位内置资源
2. install-time 模式不增加运行时下载逻辑，适合"安装后即用"的内置资源

---

## 三、两套方案的联系与差异

| 维度 | 弱联网方案A | 安卓分包 |
|------|------------|---------|
| 解决的问题 | 离线能进游戏 | 规避 GP 包体限制 |
| 作用层 | 初始化/更新流程 | 包体打包/分发 |
| 是否改业务流程 | 是（远端→本地兜底分支） | 否（业务无感） |
| 关键配置 | `CopyBuiltinPackageManifest` + `InstallCleanupMode` | `YooFolderName` 对齐 |

### 组合使用

两套方案可叠加使用：分包后的 install-time 资源同样可作为弱联网方案A的"内置兜底"来源。

**组合坑点**：同时使用时，确保 install-time 的 asset pack 名字、`YooFolderName`、`CopyBuiltinPackageManifest` 三者协调，否则离线兜底会读不到内置清单。

---

## 来源

- [弱联网环境方案A - YooAsset 官方文档](https://www.yooasset.com/docs/solution/WeakNetworkA)
- [海外安卓分包方案 - YooAsset 官方文档](https://www.yooasset.com/docs/solution/AndroidInstall)
