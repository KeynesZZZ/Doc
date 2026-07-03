---
title: 【笔记】YooAsset核心概念与架构
tags: ["Unity", "第三方库", "YooAsset", "资源管理", "热更新"]
category: 第三方库
created: 2026-07-02
updated: 2026-07-02
description: YooAsset 资源管理框架核心架构：Package/Bundle/Asset 四层模型、三大管线（Collector/Packer/Loader）、四种运行模式、关键概念速览
unity_version: 2021.3+
status: 待验证
validation: 基于官方文档与工程实践整理
related: ["[[【代码片段】YooAsset常用API速查]]", "[[【笔记】YooAsset打包管线与分包策略]]", "[[【笔记】YooAsset热更新与版本管理]]", "[[【踩坑】YooAsset中大型项目踩坑实录]]", "[[【笔记】YooAsset与HybridCLR协同方案]]", "[[【笔记】YooAsset与Addressables方案对比]]"]
author: llm
sources:
  - "YooAsset 官方文档 https://www.yooasset.com/"
  - "YooAsset GitHub https://github.com/tuyoogame/YooAsset"
  - "YooAsset Unity Package Documentation"
---

# 【笔记】YooAsset 核心概念与架构

> YooAsset 是针对 Unity 开发的资源管理 + 热更新框架。本文从架构层面梳理其核心概念、四层模型、三大管线与四种运行模式，作为 YooAsset 知识体系的入口页。

## 文档定位

- **入门**：如果你第一次接触 YooAsset，从这里开始建立全局认知。
- **导航**：底部专题索引链接到所有 YooAsset 文档。
- **API 查询**：日常开发请配合 [[【代码片段】YooAsset常用API速查]]。

---

## 一、YooAsset 是什么

### 定位

YooAsset 是一款 Unity 资源管理框架，核心解决两个问题：

1. **资源管理** — 统一 AssetBundle 的构建、加载、版本管理、内存管理
2. **热更新** — 提供从版本检查 → 差异下载 → 断点续传的完整热更链路

### 基本信息

| 项目 | 说明 |
|------|------|
| 开源协议 | MIT |
| GitHub | https://github.com/tuyoogame/YooAsset |
| 支持版本 | Unity 2019.4 ~ Unity 6.0 |
| 平台 | Windows / macOS / Android / iOS / WebGL |
| 当前主线 | YooAsset 2.x（稳定），3.x（预览） |

### 与其他方案的关系

| 方案 | 定位 |
|------|------|
| Addressables | Unity 官方，编辑器集成好，热更需 CCD |
| YooAsset | 社区方案，热更链路完整，国内项目广泛使用 |
| 传统 AssetBundle | 手工管理，灵活但维护成本高 |

> 详细对比见 [[【笔记】YooAsset与Addressables方案对比]]。

---

## 二、核心架构：四层模型

YooAsset 的数据从"项目资源文件"到"运行时加载的 Asset"，经过四层抽象：

```
项目资源（Raw Assets）
    │
    ▼  Collector 收集
Package（资源包 — 逻辑分组）
    │
    ▼  Packer 打包
Bundle（AssetBundle — 物理文件）
    │
    ▼  Loader 加载
Asset（资源对象 — 运行时实例）
```

### 2.1 Package（资源包）

- **逻辑概念**：一个 Package 代表一组相关资源的集合
- **典型用法**：`DefaultPackage`（主包）、`ActivityPackage`（活动包）、`HDTexturePackage`（高清纹理包）
- **独立性**：每个 Package 有独立的版本号和 Manifest，支持独立热更
- **生命周期**：`Initialize()` → 加载资源 → `Destroy()`

### 2.2 Collection（资源收集）

- Collector 系统负责把项目中的资源文件纳入 YooAsset 管理
- 每个 Collector Group 配置：资源路径、Address Rule、Pack Rule、Filter Rule
- 详见 [[【笔记】YooAsset打包管线与分包策略]]

### 2.3 Bundle（AssetBundle）

- 打包产物，实际的 `.bundle` 文件
- 包含资源数据 + Manifest 信息
- 支持加密（IEncryptionServices）
- 存储位置：StreamingAssets（内置）/ CDN（在线）/ 本地缓存（下载后）

### 2.4 Asset（运行时资源）

- 通过 `AssetHandle` 加载和持有
- 引用计数管理生命周期
- 支持同步 / 异步加载

---

## 三、三大管线

### 3.1 收集管线（Collector）

**职责**：将项目中的资源文件按照规则收集，生成资源清单。

| 配置项 | 作用 |
|--------|------|
| Collector Group | 一组资源收集规则，指定源目录 |
| Address Rule | 资源寻址规则（按文件名 / 按路径 / 自定义） |
| Pack Rule | 打包规则（单独打包 / 共享打包 / 目录打包） |
| Filter Rule | 过滤规则（只收集特定类型） |

### 3.2 打包管线（Packer）

**职责**：根据 Collector 收集结果，构建 AssetBundle 并生成 Manifest。

| 构建管线 | 说明 |
|----------|------|
| BuiltinBuildPipeline | 模拟 Unity 原生管线，兼容性好 |
| ScriptableBuildPipeline | Unity 官方 SBP 管线，可定制性强 |
| RawFileBuildPipeline | 原始文件管线，不压缩不打包 |

### 3.3 加载管线（Loader）

**职责**：运行时从 Bundle 中加载 Asset，管理引用计数和内存。

- 同步加载：`LoadAssetSync()`
- 异步加载：`LoadAssetAsync()`
- 场景加载：`SceneManager.LoadSceneAsync()`

---

## 四、四种运行模式

YooAsset 提供四种运行模式，覆盖开发到发布的全流程：

### 4.1 EditorSimulateMode（编辑器模拟模式）

```csharp
var buildResult = EditorSimulateModeSimulator.SimulateBuild(buildPipelineName, packageName);
var package = YooAssets.CreatePackage(packageName);
await package.InitializeAsync(new EditorSimulateModeParameters { EditorFileSystemParameters });
```

- **适用**：开发阶段，无需打 AB
- **特点**：直接从源文件加载，修改即生效
- **平台**：仅 Editor

### 4.2 OfflinePlayMode（离线模式）

```csharp
var package = YooAssets.CreatePackage(packageName);
await package.InitializeAsync(new OfflinePlayModeParameters {
    BuildinFileSystemParameters = FileSystemParameters.CreateDefaultBuildinFileSystemParameters()
});
```

- **适用**：首发包、无网络热更需求
- **特点**：从 StreamingAssets 加载，无需下载
- **平台**：全平台

### 4.3 HostPlayMode（在线模式）

```csharp
var package = YooAssets.CreatePackage(packageName);
await package.InitializeAsync(new HostPlayModeParameters {
    BuildinFileSystemParameters = FileSystemParameters.CreateDefaultBuildinFileSystemParameters(),
    CacheFileSystemParameters = FileSystemParameters.CreateDefaultCacheFileSystemParameters(),
    RemoteServices = remoteServices
});
```

- **适用**：正式上线、需要热更
- **特点**：内置资源 + CDN 远端资源 + 本地缓存
- **流程**：检查版本 → 下载 Manifest → 比对差异 → 下载更新

### 4.4 WebPlayMode（Web 模式）

```csharp
var package = YooAssets.CreatePackage(packageName);
await package.InitializeAsync(new WebPlayModeParameters {
    RemoteServices = remoteServices
});
```

- **适用**：WebGL / 微信小游戏
- **特点**：无本地文件 IO，直接从 CDN 加载

### 模式选择速查

| 模式 | 开发阶段 | 需要打 AB | 需要 CDN | 支持热更 |
|------|----------|----------|----------|----------|
| EditorSimulate | 日常开发 | 否 | 否 | 模拟 |
| OfflinePlay | 单机/首包测试 | 是 | 否 | 否 |
| HostPlay | 正式上线 | 是 | 是 | 是 |
| WebPlay | WebGL/小游戏 | 是 | 是 | 是 |

---

## 五、关键概念速览表

| 概念 | 类型 | 说明 |
|------|------|------|
| `ResourcePackage` | 运行时 | 资源包实例，加载资源的入口 |
| `AssetHandle` | 运行时 | 资源句柄，持有加载结果，负责释放 |
| `Manifest` | 数据 | 资源清单，记录 Bundle 的依赖关系和版本信息 |
| `Operation` | 异步 | 所有异步操作的基类，提供进度/状态/回调 |
| `IRemoteServices` | 接口 | 远端服务接口，提供 CDN 地址 |
| `IEncryptionServices` | 接口 | 加密接口，打包时对 Bundle 加密 |
| `IDecryptionServices` | 接口 | 解密接口，加载时对 Bundle 解密 |

### AssetHandle 生命周期

```
LoadAssetAsync() → AssetHandle（引用计数 +1）
    │
    ├── Instantiate() → 场景中的实例
    │
    └── Release()（引用计数 -1）
        └── 计数归 0 → 资源被回收
```

> **关键原则**：谁 Load 谁 Release，必须配对。详见 [[【踩坑】YooAsset中大型项目踩坑实录]]。

---

## 六、初始化流程完整示例

```csharp
using UnityEngine;
using YooAsset;
using Cysharp.Threading.Tasks;

public class YooAssetInitializer : MonoBehaviour
{
    public string PackageName = "DefaultPackage";

    private ResourcePackage _package;

    private async UniTaskVoid Start()
    {
        // 1. 初始化 YooAssets
        YooAssets.Initialize();
        YooAssets.DestroyPackage(PackageName);

        // 2. 创建资源包
        _package = YooAssets.CreatePackage(PackageName);
        YooAssets.SetDefaultPackage(_package);

        // 3. 根据模式初始化（以 HostPlayMode 为例）
        var initParameters = new HostPlayModeParameters();
        initParameters.BuildinFileSystemParameters =
            FileSystemParameters.CreateDefaultBuildinFileSystemParameters();
        initParameters.CacheFileSystemParameters =
            FileSystemParameters.CreateDefaultCacheFileSystemParameters();
        initParameters.RemoteServices = new RemoteServices();

        var initOperation = _package.InitializeAsync(initParameters);
        await initOperation.ToUniTask();

        if (initOperation.Status == EOperationStatus.Succeed)
        {
            Debug.Log("YooAsset 初始化成功");
            // 4. 开始热更检查（详见热更新文档）
        }
        else
        {
            Debug.LogError($"YooAsset 初始化失败: {initOperation.Error}");
        }
    }
}

// CDN 远端服务实现
public class RemoteServices : IRemoteServices
{
    public string GetRemoteMainURL(string fileName)
    {
        return $"http://your-cdn.com/bundles/{fileName}";
    }

    public string GetRemoteFallbackURL(string fileName)
    {
        return $"http://backup-cdn.com/bundles/{fileName}";
    }
}
```

> 完整 API 速查见 [[【代码片段】YooAsset常用API速查]]。

---

## 七、YooAsset 知识体系导航

| 文档 | 目录 | 定位 |
|------|------|------|
| **本文** — 核心概念与架构 | `60_第三方库` | 入口页，架构总览 |
| [[【代码片段】YooAsset常用API速查]] | `60_第三方库` | 日常开发代码字典 |
| [[【笔记】YooAsset打包管线与分包策略]] | `35_高级主题` | Collector/Packer/分包设计 |
| [[【笔记】YooAsset热更新与版本管理]] | `35_高级主题` | 增量更新/断点续传/CDN |
| [[【踩坑】YooAsset中大型项目踩坑实录]] | `35_高级主题` | 内存/依赖/并发/平台兼容 |
| [[【笔记】YooAsset与HybridCLR协同方案]] | `35_高级主题` | 代码热更+资源热更全链路 |
| [[【笔记】YooAsset与Addressables方案对比]] | `35_高级主题` | 横向技术选型评测 |
| [[YooAsset专题索引]] | `35_高级主题` | 完整导航与知识图谱 |

---

## 官方参考

- [YooAsset 官方文档](https://www.yooasset.com/)
- [YooAsset GitHub](https://github.com/tuyoogame/YooAsset)
- [YooAsset Quick Start](https://www.yooasset.com/docs/guide-editor/QuickStart)
- [YooAsset Code Tutorial](https://www.yooasset.com/docs/guide-runtime/CodeTutorial1)
