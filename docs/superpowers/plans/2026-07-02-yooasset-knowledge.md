# YooAsset 知识体系文档 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 UnityKnowledge 仓库创建完整的 YooAsset 知识体系（7 篇文档 + 1 索引页），覆盖核心架构、API 速查、打包管线、热更新、踩坑实录、HybridCLR 协同、方案对比。

**Architecture:** 基础教程（架构 + API）放 `60_第三方库/`，高级实战（打包/热更/踩坑/协同/对比 + 索引）放 `35_高级主题/`，通过 `[[]]` 双向链接串联，索引页汇总导航。

**Tech Stack:** Markdown + YAML frontmatter + Obsidian WikiLinks + YooAsset 2.x/3.x API

## Global Constraints

- **文件命名**：使用 V2 前缀 — `【笔记】`/`【代码片段】`/`【踩坑】`
- **YAML frontmatter**：必须包含 `title`、`tags`（最少 2 个）、`created: 2026-07-02`、`description`、`author: llm`、`status: 待验证`
- **LLM-Wiki 模式**：`author: llm` 的文档必须带 `sources:` 字段
- **代码注释**：使用中文
- **链接格式**：Obsidian `[[]]` 双向链接
- **Unity 版本**：覆盖 2021.3 LTS / 2022.3 LTS / Unity 6
- **YooAsset 版本**：基于 2.x（主流稳定）撰写，涉及 3.x 新特性时标注版本差异
- **中文**：所有内容使用简体中文

---

## File Structure

```
UnityKnowledge/
├── 60_第三方库/
│   ├── 【笔记】YooAsset核心概念与架构.md          ← Task 1
│   └── 【代码片段】YooAsset常用API速查.md          ← Task 2
└── 35_高级主题/
    ├── 【笔记】YooAsset打包管线与分包策略.md       ← Task 3
    ├── 【笔记】YooAsset热更新与版本管理.md         ← Task 4
    ├── 【踩坑】YooAsset中大型项目踩坑实录.md       ← Task 5
    ├── 【笔记】YooAsset与HybridCLR协同方案.md      ← Task 6
    ├── 【笔记】YooAsset与Addressables方案对比.md   ← Task 7
    └── YooAsset专题索引.md                         ← Task 8
```

**跨文档链接关系**：
- 文档 1 ↔ 文档 2（架构 ↔ API 速查互链）
- 文档 3~7 → 文档 1（全部链回入口页）
- 文档 6 → `[[【踩坑】HybridCLR接入常见坑]]`、`[[【笔记】HybridCLR构建管线与Generate工具链]]`
- 文档 7 → `[[【教程】资源管线-Addressables]]`、`[[【最佳实践】Addressables性能优化]]`
- 索引页 → 所有 7 篇文档

---

## Task 1: 【笔记】YooAsset核心概念与架构

**Files:**
- Create: `UnityKnowledge/60_第三方库/【笔记】YooAsset核心概念与架构.md`

**Interfaces:**
- Produces: YooAsset 知识体系入口页，后续 6 篇文档全部链接回此页

- [ ] **Step 1: 创建文档，写入完整内容**

文件路径：`UnityKnowledge/60_第三方库/【笔记】YooAsset核心概念与架构.md`

完整内容：

```markdown
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
```

- [ ] **Step 2: 验证文档格式**

检查 YAML frontmatter 格式正确，`[[]]` 链接语法正确，代码块语法正确。

- [ ] **Step 3: 提交**

```bash
cd "D:/GitHub/Doc"
git add "UnityKnowledge/60_第三方库/【笔记】YooAsset核心概念与架构.md"
git commit -m "docs(UnityKnowledge): 新增 YooAsset 核心概念与架构入口页

涵盖四层模型(Package/Collection/Bundle/Asset)、三大管线(Collector/Packer/Loader)、
四种运行模式(EditorSimulate/OfflinePlay/HostPlay/WebPlay)、关键概念速览表、
完整初始化代码示例"
```

---

## Task 2: 【代码片段】YooAsset常用API速查

**Files:**
- Create: `UnityKnowledge/60_第三方库/【代码片段】YooAsset常用API速查.md`

**Interfaces:**
- Consumes: Task 1 的概念体系
- Produces: API 速查字典，后续文档引用的代码来源

- [ ] **Step 1: 创建文档，写入完整内容**

文件路径：`UnityKnowledge/60_第三方库/【代码片段】YooAsset常用API速查.md`

完整内容：

```markdown
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
```

- [ ] **Step 2: 提交**

```bash
cd "D:/GitHub/Doc"
git add "UnityKnowledge/60_第三方库/【代码片段】YooAsset常用API速查.md"
git commit -m "docs(UnityKnowledge): 新增 YooAsset 常用 API 速查手册

覆盖初始化/资源加载/实例化释放/场景加载/文件下载/资源信息/
解密接口/多 Package 管理 8 个高频场景，代码块复制即用"
```

---

## Task 3: 【笔记】YooAsset打包管线与分包策略

**Files:**
- Create: `UnityKnowledge/35_高级主题/【笔记】YooAsset打包管线与分包策略.md`

**Interfaces:**
- Consumes: Task 1 的概念体系
- Produces: 打包配置指南，Task 4/5/6 引用

- [ ] **Step 1: 创建文档，写入完整内容**

文件路径：`UnityKnowledge/35_高级主题/【笔记】YooAsset打包管线与分包策略.md`

完整内容：

```markdown
---
title: 【笔记】YooAsset打包管线与分包策略
tags: ["Unity", "YooAsset", "打包管线", "分包策略", "AssetBundle"]
category: 高级主题
created: 2026-07-02
updated: 2026-07-02
description: YooAsset Collector 收集系统、Packer 打包模式、中大型项目分包策略设计、加密方案、构建产物结构解读
unity_version: 2021.3+
status: 待验证
validation: 基于官方文档与工程实践整理
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【代码片段】YooAsset常用API速查]]", "[[【笔记】YooAsset热更新与版本管理]]", "[[【踩坑】YooAsset中大型项目踩坑实录]]"]
author: llm
sources:
  - "YooAsset 官方文档 https://www.yooasset.com/"
  - "YooAsset AssetBundleCollector 文档 https://www.yooasset.com/docs/guide-editor/AssetBundleCollector"
  - "YooAsset AssetBundleBuilder 文档 https://www.yooasset.com/docs/guide-editor/AssetBundleBuilder"
  - "[[【笔记】YooAsset核心概念与架构]]"
---

# 【笔记】YooAsset 打包管线与分包策略

> 从 Collector 收集配置到 Packer 打包产物，完整拆解 YooAsset 的构建管线，并给出中大型项目的分包策略设计。

## 文档定位

- **概念基础**：[[【笔记】YooAsset核心概念与架构]] 已介绍三大管线的概览，本文深入**收集管线**和**打包管线**的配置细节。
- **实战场景**：面向中大型项目（多模块、分包需求、CDN 部署），给出可落地的配置实例。

---

## 一、Collector 收集系统

### 1.1 Collector Group（收集组）

Collector Group 是 YooAsset 收集系统的核心组织单位。每个 Group 配置：

| 配置项 | 作用 | 示例 |
|--------|------|------|
| Collect Path | 资源源目录 | `Assets/GameRes/UI` |
| Collector Type | 收集类型 | MainAsset（主资源）/ StaticAsset（静态资源，不独立打包） |
| Address Rule | 寻址规则 | AddressByFileName / AddressByGroupPath |
| Pack Rule | 打包规则 | PackSeparately / PackDirectory / PackGroup |
| Filter Rule | 过滤规则 | CollectAll / CollectSprite / CollectScene |
| Active Rule | 激活规则 | ActiveByName / ActiveByGroup |

### 1.2 Address Rule（寻址规则）

寻址规则决定了加载资源时使用的地址字符串：

```csharp
// AddressByFileName — 按文件名寻址（最常用）
// 文件路径: Assets/Prefabs/UI/PlayerPanel.prefab
// 寻址地址: PlayerPanel
package.LoadAssetAsync<GameObject>("PlayerPanel");

// AddressByFilePath — 按完整路径寻址
// 寻址地址: Assets/Prefabs/UI/PlayerPanel.prefab
package.LoadAssetAsync<GameObject>("Assets/Prefabs/UI/PlayerPanel.prefab");

// AddressByGroupPath — 按组路径寻址（适合模块化项目）
// 寻址地址: UI/PlayerPanel
package.LoadAssetAsync<GameObject>("UI/PlayerPanel");
```

### 1.3 Pack Rule（打包规则）

打包规则决定了资源如何被组合进 AssetBundle：

| 规则 | 说明 | 适用场景 |
|------|------|----------|
| PackSeparately | 每个资源单独一个 Bundle | UI Prefab、独立配置表 |
| PackDirectory | 同一目录的资源打成一个 Bundle | 同类纹理、同模块音频 |
| PackGroup | 整个 Group 打成一个 Bundle | 小文件集、共享依赖 |
| PackTopDirectory | 顶层目录分别打包 | 按模块分包 |

### 1.4 Filter Rule（过滤规则）

```csharp
// 只收集 Sprite 类型
// CollectSprite — 过滤 .png/.jpg/.tga 等

// 只收集场景
// CollectScene — 过滤 .unity

// 收集全部
// CollectAll — 不过滤

// 自定义过滤规则（通过 IFilterRule 接口）
public class CollectPrefab : IFilterRule
{
    public bool IsCollectAsset(FileInfo fileInfo)
    {
        return fileInfo.Extension == ".prefab";
    }
}
```

---

## 二、Packer 打包模式

### 2.1 三种构建管线对比

| 构建管线 | 说明 | 优势 | 适用场景 |
|----------|------|------|----------|
| BuiltinBuildPipeline | 模拟 Unity 原生管线 | 兼容性最好 | 老项目迁移 |
| ScriptableBuildPipeline | Unity 官方 SBP | 可定制性强、依赖处理优 | **推荐新项目** |
| RawFileBuildPipeline | 原始文件管线 | 不压缩不打包 | 视频等大文件 |

### 2.2 压缩格式选择

| 压缩格式 | 说明 | 适用场景 |
|----------|------|----------|
| Uncompressed | 无压缩 | 开发调试、WebGL |
| LZ4 | 快速压缩 | **推荐**，加载速度与体积的平衡 |
| LZ4HC | 高压缩 LZ4 | 正式发布 |
| LZMA | 最高压缩比 | 包体敏感场景（加载较慢） |

### 2.3 打包配置示例

```
Build Pipeline: ScriptableBuildPipeline
Package Name: DefaultPackage
Compress Option: LZ4
File Name Style: HashName（使用 Hash 命名，避免路径泄露）
Copy Buildin File Option: None（不内置到首包）
```

---

## 三、分包策略设计（核心）

### 3.1 首包最小化策略

**目标**：将首包体积控制在平台限制内（Android 150MB / iOS 4GB）。

**方案**：

```
DefaultPackage（首包必需）
├── 启动场景
├── 核心 UI 框架
├── 基础角色 Prefab
└── 新手引导资源

ActivityPackage（活动包，按需下载）
├── 活动场景
├── 活动 UI
└── 活动专属资源

HDTexturePackage（高清纹理包，可选）
├── 高清角色贴图
└── 高清场景贴图

LevelPackage_01（关卡分包 1）
├── 关卡 1-50 场景
└── 关卡专属资源
```

### 3.2 分包版本独立性

每个 Package 有独立的版本号，可以独立热更：

```
CDN 目录结构:
/cdn-server/
├── DefaultPackage/
│   ├── v1.0.0/
│   │   ├── manifest.hash
│   │   ├── manifest.bytes
│   │   └── bundles/
│   └── v1.0.1/
│       └── ...（增量补丁）
├── ActivityPackage/
│   ├── v1.0.0/
│   └── v1.1.0/（活动更新，不影响主包）
└── HDTexturePackage/
    └── v1.0.0/
```

### 3.3 分包粒度设计原则

| 分包类型 | 粒度建议 | 理由 |
|----------|----------|------|
| 核心包 | 必需的最小集 | 首包体积控制 |
| 关卡包 | 50 关 / 包 | 玩家不会一次打所有关卡 |
| 活动包 | 一个活动 / 包 | 活动结束可整体卸载 |
| 纹理包 | 按场景或角色 | 高清纹理按需下载 |

### 3.4 分包间依赖处理

```csharp
// 多 Package 加载共享资源的正确姿势
// 方案 A：共享资源放到 DefaultPackage，其他 Package 通过路径引用
var defaultPkg = YooAssets.GetPackage("DefaultPackage");
var handle = defaultPkg.LoadAssetAsync<GameObject>("SharedEffect");

// 方案 B：使用 YooAssets 全局接口（自动路由到正确的 Package）
var handle = YooAssets.LoadAssetAsync<GameObject>("SharedEffect");
```

> **注意**：跨 Package 引用时，地址必须是全局唯一的。详见 [[【踩坑】YooAsset中大型项目踩坑实录]]。

---

## 四、加密方案

### 4.1 加密接口

YooAsset 通过 `IEncryptionServices` 接口支持自定义加密：

```csharp
// 打包时加密
public class FileOffsetEncryption : IEncryptionServices
{
    public EncryptResult Encrypt(EncryptFileInfo fileInfo)
    {
        // 方案 1：文件偏移加密（最简单）
        // 在文件头插入随机字节，加载时跳过
        byte[] fileData = File.ReadAllBytes(fileInfo.FilePath);
        byte[] encryptData = new byte[32 + fileData.Length]; // 32 字节随机头

        // 填充随机字节
        for (int i = 0; i < 32; i++)
            encryptData[i] = (byte)UnityEngine.Random.Range(0, 256);

        // 拷贝原始数据
        Buffer.BlockCopy(fileData, 0, encryptData, 32, fileData.Length);

        return new EncryptResult
        {
            Encrypted = true,
            EncryptedData = encryptData,
            FileOffset = 32  // 加载时需要跳过的偏移量
        };
    }
}
```

### 4.2 三种加载解密方式

| 方式 | 接口方法 | 说明 | 适用场景 |
|------|----------|------|----------|
| 文件偏移 | `LoadFromFileOffset` | 指定偏移量读取 | 简单加密，性能最好 |
| 内存解密 | `LoadFromMemory` | 完整读入内存后解密 | AES 等加密算法 |
| 流式解密 | `LoadFromStream` | 返回解密流 | 大文件 |

> 运行时解密实现见 [[【代码片段】YooAsset常用API速查]] §7。

---

## 五、构建产物结构

### 5.1 产物目录

```
BundlesOutput/
├── DefaultPackage/
│   ├── BuildReport.html          ← 构建报告（人读）
│   ├── BuildReport.json          ← 构建报告（机读）
│   ├── manifest.hash             ← Manifest 文件哈希（用于版本对比）
│   ├── manifest.json             ← Manifest 数据（依赖关系、Bundle 信息）
│   ├── manifest.json.bytes       ← Manifest 二进制版本（运行时加载）
│   ├── Setting.json              ← 构建配置快照
│   └── Bundles/
│       ├── 5f3a2b...bundle       ← AssetBundle 文件（Hash 命名）
│       ├── 8c1d4e...bundle
│       └── ...
```

### 5.2 BuildReport 解读

```json
{
  "BuildResult": {
    "BuildSuccess": true,
    "TotalBundleCount": 156,
    "TotalBundleSize": 67108864,
    "AverageBundleSize": 430000,
    "MaxBundleSize": 5242880,
    "MinBundleSize": 1024
  },
  "AssetList": [
    {
      "Address": "PlayerPanel",
      "AssetPath": "Assets/Prefabs/UI/PlayerPanel.prefab",
      "BundleName": "5f3a2b...bundle",
      "Size": 24576
    }
  ],
  "BundleList": [
    {
      "BundleName": "5f3a2b...bundle",
      "Size": 24576,
      "ReferencedCount": 3,
      "MainAssetSize": 20480,
      "Dependencies": ["8c1d4e...bundle"]
    }
  ]
}
```

**关键指标**：
- `TotalBundleSize` — 总包体大小，直接影响首包体积
- `MaxBundleSize` — 最大单个 Bundle，过大需要考虑拆分
- `ReferencedCount` — 被引用次数，>1 说明被多个 Bundle 共享
- `MainAssetSize` vs `Size` 差值 — 依赖冗余大小

---

## 六、实操示例：中大型项目 Collector 配置

### 6.1 配置总览

| Package | Collector Group | Address Rule | Pack Rule | 说明 |
|---------|-----------------|--------------|-----------|------|
| DefaultPackage | UI/ | AddressByFileName | PackSeparately | UI Panel 独立打包 |
| DefaultPackage | UI/Atlas/ | AddressByFileName | PackDirectory | 图集按目录打包 |
| DefaultPackage | Config/ | AddressByFileName | PackGroup | 配置表打成一个包 |
| DefaultPackage | Core/ | AddressByFilePath | PackSeparately | 核心资源路径寻址 |
| LevelPackage | Levels/ | AddressByFileName | PackSeparately | 关卡资源独立打包 |
| ActivityPackage | Events/ | AddressByFileName | PackDirectory | 活动按目录打包 |

### 6.2 Jenkins 自动化构建

```python
# Python 脚本：调用 Unity 执行 YooAsset 打包
import subprocess
import os

def build_yooasset(unity_path, project_path, package_name, output_path):
    """执行 YooAsset 打包"""
    args = [
        unity_path,
        "-batchmode",
        "-projectPath", project_path,
        "-executeMethod", "BuildPipeline.BuildYooAsset",
        "-packageName", package_name,
        "-outputPath", output_path,
        "-quit"
    ]
    result = subprocess.run(args, capture_output=True, text=True)
    if result.returncode != 0:
        print(f"构建失败: {result.stderr}")
        return False

    # 检查构建产物
    manifest_path = os.path.join(output_path, package_name, "manifest.hash")
    if not os.path.exists(manifest_path):
        print("构建产物不完整")
        return False

    print(f"构建成功: {output_path}/{package_name}")
    return True

# 使用示例
build_yooasset(
    "C:/Program Files/Unity/Hub/Editor/2022.3.20f1/Editor/Unity.exe",
    "D:/MyProject",
    "DefaultPackage",
    "D:/BuildOutput"
)
```

---

## 七、打包优化 Checklist

### Bundle 大小优化

- [ ] 单个 Bundle 不超过 5MB（超过考虑拆分）
- [ ] 共享依赖单独打包（避免冗余）
- [ ] 静态资源（材质、Shader）使用 StaticAsset 类型
- [ ] 使用 LZ4 压缩而非无压缩

### 分包策略优化

- [ ] 首包只包含启动必需资源
- [ ] 关卡/活动按模块分包
- [ ] 高清资源独立分包
- [ ] 分包版本独立可更

### 构建流程优化

- [ ] 构建流程脚本化（Jenkins / Python / Shell）
- [ ] 每次构建自动校验 BuildReport
- [ ] 构建产物与版本号绑定
- [ ] CDN 上传后校验文件完整性

---

## 相关文档

- [[【笔记】YooAsset核心概念与架构]] — 四层模型与三大管线概览
- [[【代码片段】YooAsset常用API速查]] — 加密/解密 API 实现
- [[【笔记】YooAsset热更新与版本管理]] — 打包产物如何用于热更
- [[【踩坑】YooAsset中大型项目踩坑实录]] — 分包相关的坑

## 官方参考

- [YooAsset AssetBundleCollector](https://www.yooasset.com/docs/guide-editor/AssetBundleCollector)
- [YooAsset AssetBundleBuilder](https://www.yooasset.com/docs/guide-editor/AssetBundleBuilder)
- [YooAsset GitHub](https://github.com/tuyoogame/YooAsset)
```

- [ ] **Step 2: 提交**

```bash
cd "D:/GitHub/Doc"
git add "UnityKnowledge/35_高级主题/【笔记】YooAsset打包管线与分包策略.md"
git commit -m "docs(UnityKnowledge): 新增 YooAsset 打包管线与分包策略

涵盖 Collector 收集系统(AddressRule/PackRule/FilterRule)、三种构建管线对比、
中大型项目分包策略设计(首包最小化/分包独立性/依赖处理)、
加密方案(文件偏移/内存/流式)、构建产物结构解读、Jenkins 自动化示例"
```

---

## Task 4: 【笔记】YooAsset热更新与版本管理

**Files:**
- Create: `UnityKnowledge/35_高级主题/【笔记】YooAsset热更新与版本管理.md`

**Interfaces:**
- Consumes: Task 1 的概念体系，Task 3 的打包产物结构
- Produces: 完整热更流程指南，Task 6 引用

- [ ] **Step 1: 创建文档，写入完整内容**

文件路径：`UnityKnowledge/35_高级主题/【笔记】YooAsset热更新与版本管理.md`

完整内容：

```markdown
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
    // 重试次数用尽后， downloader.Status 会变为 Failed
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
```

- [ ] **Step 2: 提交**

```bash
cd "D:/GitHub/Doc"
git add "UnityKnowledge/35_高级主题/【笔记】YooAsset热更新与版本管理.md"
git commit -m "docs(UnityKnowledge): 新增 YooAsset 热更新与版本管理

涵盖版本号机制(StaticVersion/PackageVersion)、增量更新差异比对、
断点续传实现、CDN 部署规范(目录结构/多节点容灾/灰度发布)、
完整热更代码示例(版本检查+下载+进度展示)、边界情况处理
(网络中断/磁盘不足/版本回退/离线在线切换)"
```

---

## Task 5: 【踩坑】YooAsset中大型项目踩坑实录

**Files:**
- Create: `UnityKnowledge/35_高级主题/【踩坑】YooAsset中大型项目踩坑实录.md`

**Interfaces:**
- Consumes: Task 1~4 的概念和流程
- Produces: 踩坑速查表，Task 6/7 引用

- [ ] **Step 1: 创建文档，写入完整内容**

文件路径：`UnityKnowledge/35_高级主题/【踩坑】YooAsset中大型项目踩坑实录.md`

完整内容：

```markdown
---
title: 【踩坑】YooAsset中大型项目踩坑实录
tags: ["Unity", "YooAsset", "踩坑记录", "资源管理", "热更新", "内存管理"]
category: 高级主题
created: 2026-07-02
updated: 2026-07-02
description: YooAsset 中大型项目高频坑：AssetHandle 泄漏、引用计数误用、多 Package 依赖冲突、并发加载瓶颈、平台兼容问题
unity_version: 2021.3+
status: 待验证
validation: 社区高频反馈整理
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【代码片段】YooAsset常用API速查]]", "[[【笔记】YooAsset打包管线与分包策略]]", "[[【笔记】YooAsset热更新与版本管理]]", "[[【踩坑】HybridCLR接入常见坑]]"]
author: llm
sources:
  - "YooAsset GitHub Issues https://github.com/tuyoogame/YooAsset/issues"
  - "YooAsset 官方文档 https://www.yooasset.com/"
  - "[[【笔记】YooAsset核心概念与架构]]"
---

# 【踩坑】YooAsset 中大型项目踩坑实录

> YooAsset 在中大型项目落地中的高频坑，按类型分为内存管理、依赖加载、并发性能、平台兼容四类。每条按「现象 → 原因分析 → 解决方案 → 验证方式」组织。

## 文档定位

集中 YooAsset 工程实践中最容易踩的坑，帮助接入者快速定位与规避。与 [[【踩坑】HybridCLR接入常见坑]] 互补——前者针对 HybridCLR，本文针对 YooAsset。

---

## 一、内存管理坑

### 坑 1：AssetHandle 未释放导致内存泄漏

**现象**：反复加载/销毁 UI Panel 后，内存持续增长不回落，Profiler 中 AssetBundle 数量不断增加。

**原因分析**：

```csharp
// 错误代码 — 只 Destroy 了 GameObject，没有 Release handle
private void OpenPanel()
{
    var handle = package.LoadAssetAsync<GameObject>("UIPanel");
    handle.Completed += op =>
    {
        var panel = Instantiate(op.AssetObject);
        // handle 从未 Release！
    };
}

private void ClosePanel(GameObject panel)
{
    Destroy(panel);
    // handle 已经超出作用域，但引用计数没归零
    // AssetBundle 无法被卸载 → 内存泄漏
}
```

**解决方案**：

```csharp
// 正确做法 — 将 handle 存为字段，Close 时 Release
private AssetHandle _panelHandle;
private GameObject _panelInstance;

private async void OpenPanel()
{
    _panelHandle = package.LoadAssetAsync("UIPanel");
    await _panelHandle.ToUniTask();
    _panelInstance = Instantiate(_panelHandle.AssetObject) as GameObject;
}

private void ClosePanel()
{
    if (_panelInstance != null)
    {
        Destroy(_panelInstance);
        _panelInstance = null;
    }
    if (_panelHandle != null)
    {
        _panelHandle.Release();  // 引用计数 -1
        _panelHandle = null;
    }
}
```

**验证方式**：
1. 打开 Memory Profiler，记录加载/卸载前后的 AssetBundle 数量
2. 反复打开关闭 Panel 10 次，确认 AssetBundle 计数不持续增长
3. 确认 `Resources.UnloadUnusedAssets()` 后内存回落到基线

---

### 坑 2：Bundle 依赖引用计数理解偏差

**现象**：显式 Release 了某个资源的 handle，但关联的 Bundle 仍然占用内存。

**原因分析**：

Bundle 之间有依赖关系。Bundle A 依赖 Bundle B（因为 A 中的 Prefab 引用了 B 中的材质）。只有当 A 和 B 的引用计数**都归零**时，B 才会被卸载。

```
BundleA (主资源) ──依赖──> BundleB (共享材质)
RefCount: 0 (已 Release)     RefCount: 1 (被 BundleA 引用)
                              ↑ 此时 BundleB 不会被卸载！

等 BundleA 也 Release 后:
BundleA RefCount: 0
BundleB RefCount: 0 → 才会被卸载
```

实际中 BundleB 可能被多个资源引用（如共享 UI 材质），只要还有任何一个资源持有引用，BundleB 就不会被卸载。

**解决方案**：

```csharp
// 确保所有引用同一 Bundle 的资源都被正确 Release
// 使用引用计数管理器统一跟踪

public class AssetRefManager
{
    private Dictionary<string, AssetHandle> _handles = new();

    public async UniTask<T> LoadAsync<T>(string address) where T : Object
    {
        if (_handles.ContainsKey(address))
        {
            // 已加载，返回已有 handle（引用计数由 YooAsset 内部管理）
            return _handles[address].AssetObject as T;
        }

        var handle = YooAssets.LoadAssetAsync<T>(address);
        await handle.ToUniTask();
        _handles[address] = handle;
        return handle.AssetObject as T;
    }

    public void Release(string address)
    {
        if (_handles.TryGetValue(address, out var handle))
        {
            handle.Release();
            _handles.Remove(address);
        }
    }

    public void ReleaseAll()
    {
        foreach (var handle in _handles.Values)
        {
            handle.Release();
        }
        _handles.Clear();
    }
}
```

**验证方式**：
1. 使用 Memory Profiler 检查 Bundle 卸载情况
2. 调用 `package.UnloadUnusedAssets()` 后检查内存变化
3. 确保场景切换时所有旧场景资源被正确 Release

---

### 坑 3：Resources.UnloadUnusedAssets 与 YooAsset 的协调

**现象**：调用 `Resources.UnloadUnusedAssets()` 后，YooAsset 加载的资源被意外卸载，再次使用时出现粉色材质或空引用。

**原因分析**：

`Resources.UnloadUnusedAssets()` 会卸载所有"无引用"的资源。但 YooAsset 通过 AssetHandle 持有的资源，如果只被 C# 字段引用（而非被场景中的 GameObject 引用），可能被判定为"无引用"而卸载。

**解决方案**：

```csharp
// 规则：YooAsset 资源的生命周期由 AssetHandle 管理
// 不要在持有 AssetHandle 的资源时调用 Resources.UnloadUnusedAssets()

// 正确的清理时机：场景切换时，先 Release 所有旧场景 handle
foreach (var handle in _sceneAssetHandles)
{
    handle.Release();
}
_sceneAssetHandles.Clear();

// 然后再调用 UnloadUnusedAssets
await UniTask.DelayFrame(1); // 等一帧让引用关系更新
Resources.UnloadUnusedAssets();
// 或者使用 YooAsset 提供的封装
package.UnloadUnusedAssets();
```

**验证方式**：
1. 调用 `UnloadUnusedAssets` 后立即检查关键资源是否完好
2. 在场景切换流程中加入资源完整性校验

---

## 二、依赖与加载坑

### 坑 4：多 Package 共享资源的依赖冲突

**现象**：两个 Package 中都有名为 `"HealthBar"` 的资源，加载时总是返回错误的那个。

**原因分析**：

当 Address Rule 使用 `AddressByFileName` 时，不同 Package 中同名文件会产生地址冲突。`YooAssets.LoadAssetAsync("HealthBar")` 通过默认 Package 加载，可能加载到错误的资源。

**解决方案**：

```csharp
// 方案 1：使用全路径寻址（AddressByFilePath），避免重名
package.LoadAssetAsync<GameObject>("Assets/UI/HealthBar.prefab");
activityPackage.LoadAssetAsync<GameObject>("Assets/Activity/HealthBar.prefab");

// 方案 2：使用 AddressByGroupPath，加模块前缀
// UI 模块的 Group: 寻址为 "UI/HealthBar"
// Activity 模块的 Group: 寻址为 "Activity/HealthBar"

// 方案 3：显式指定 Package 加载
var defaultPkg = YooAssets.GetPackage("DefaultPackage");
var activityPkg = YooAssets.GetPackage("ActivityPackage");

var uiHandle = defaultPkg.LoadAssetAsync<GameObject>("UI_HealthBar");
var actHandle = activityPkg.LoadAssetAsync<GameObject>("Activity_HealthBar");
```

**验证方式**：
1. 在 Collector 配置中检查是否有地址冲突警告
2. 加载后检查资源路径是否符合预期
3. 使用 BuildReport 检查 Address 列表中是否有重复

---

### 坑 5：Address 规则变更导致 Manifest 不匹配

**现象**：修改了 Collector 的 Address Rule（如从 `AddressByFileName` 改为 `AddressByFilePath`）后，客户端热更报错"资源地址不存在"。

**原因分析**：

Address Rule 变更后，新版本 Manifest 中的地址与旧版本完全不同。客户端如果不更新 Manifest 就找不到资源。

**解决方案**：

```csharp
// Address Rule 变更属于破坏性变更，必须全量更新
// 1. 重新打包所有资源
// 2. 客户端必须更新到新版本 Manifest
// 3. 不能通过增量更新处理地址变更

// 在版本管理中加入 BreakingChange 标记
public class VersionManager
{
    public bool IsBreakingChange(string oldVersion, string newVersion)
    {
        // 大版本变更视为 Breaking Change
        var oldParts = oldVersion.Split('.');
        var newParts = newVersion.Split('.');
        return oldParts[0] != newParts[0];
    }

    public async UniTask HandleBreakingChange()
    {
        // 清理所有缓存
        var package = YooAssets.GetPackage("DefaultPackage");
        await package.ClearAllBundleFilesAsync();

        // 重新初始化
        // ... 重新执行完整初始化 + 热更流程
    }
}
```

**验证方式**：
1. 升级后全量测试资源加载路径
2. 在 Manifest 中记录 Address Rule 版本，运行时做兼容检查

---

### 坑 6：异步加载时序问题

**现象**：在协程/异步方法中加载资源，获取到的结果为 null 或状态为 `None`。

**原因分析**：

```csharp
// 错误：没有等待完成就读取结果
private void WrongUsage()
{
    var handle = package.LoadAssetAsync<GameObject>("Player");
    // handle.Status 此时还是 None！异步操作还没完成
    var prefab = handle.AssetObject; // null!
}

// 错误：Completed 回调中忘记检查状态
private void WrongUsage2()
{
    var handle = package.LoadAssetAsync<GameObject>("Player");
    handle.Completed += op =>
    {
        // 没检查 op.Status，可能失败了
        Instantiate(op.AssetObject); // 如果失败，AssetObject 是 null
    };
}
```

**解决方案**：

```csharp
// 正确：使用 await（推荐 UniTask）
private async UniTaskVoid CorrectUsage()
{
    var handle = package.LoadAssetAsync<GameObject>("Player");
    await handle.ToUniTask();

    if (handle.Status == EOperationStatus.Succeed)
    {
        Instantiate(handle.AssetObject);
    }
    else
    {
        Debug.LogError($"加载失败: {handle.LastError}");
    }
}

// 正确：使用 Completed 回调（如果不使用 UniTask）
private void CorrectUsage2()
{
    var handle = package.LoadAssetAsync<GameObject>("Player");
    handle.Completed += op =>
    {
        if (op.Status == EOperationStatus.Succeed)
        {
            Instantiate(op.AssetObject);
        }
        else
        {
            Debug.LogError($"加载失败: {op.LastError}");
        }
    };
}
```

**验证方式**：
1. 所有异步加载代码统一使用 await 模式
2. 加载后必须检查 `handle.Status == EOperationStatus.Succeed`

---

## 三、并发与性能坑

### 坑 7：同帧大量并发加载的性能瓶颈

**现象**：打开一个复杂的 UI 界面（包含 20+ 个图片和 Prefab），一次性发起所有加载请求，导致掉帧严重。

**原因分析**：

同时发起大量异步加载请求，虽然 YooAsset 内部有并发限制，但所有加载回调集中在一帧内执行，导致主线程卡顿。

**解决方案**：

```csharp
// 方案 1：分帧加载（推荐）
private async UniTask LoadUIElements(List<string> addresses)
{
    foreach (var address in addresses)
    {
        var handle = package.LoadAssetAsync<GameObject>(address);
        await handle.ToUniTask();
        Instantiate(handle.AssetObject);
        await UniTask.DelayFrame(1); // 每个元素间隔一帧
    }
}

// 方案 2：使用 UniTask 的批量等待
private async UniTask LoadUIBatch(List<string> addresses)
{
    var tasks = addresses.Select(addr =>
    {
        var handle = package.LoadAssetAsync<GameObject>(addr);
        return handle.ToUniTask();
    });

    // 分批等待，每批 5 个
    foreach (var batch in tasks.Chunk(5))
    {
        await UniTask.WhenAll(batch);
        await UniTask.DelayFrame(1); // 批次间间隔一帧
    }
}
```

**验证方式**：
1. 使用 Unity Profiler 检查加载时的帧时间
2. 目标：单帧时间不超过 33ms（30fps 最低要求）

---

### 坑 8：Android IL2CPP 的 GC 压力

**现象**：Android 真机上频繁加载资源时 GC 频繁触发，导致卡顿。

**原因分析**：

YooAsset 的异步操作对象（Operation）会产生 Managed Heap 分配。大量短生命周期对象导致 GC 压力。IL2CPP 的 GC 不如 Mono 高效。

**解决方案**：

```csharp
// 1. 减少不必要的重复加载/卸载
// 使用缓存池管理频繁使用的资源
public class AssetCachePool
{
    private Dictionary<string, AssetHandle> _cache = new();

    public async UniTask<GameObject> GetAsync(string address)
    {
        if (_cache.TryGetValue(address, out var cached))
        {
            return cached.AssetObject as GameObject;
        }

        var handle = package.LoadAssetAsync<GameObject>(address);
        await handle.ToUniTask();
        _cache[address] = handle;
        return handle.AssetObject as GameObject;
    }
}

// 2. 批量加载替代逐个加载
// 减少 Operation 对象的创建数量

// 3. 避免在热路径上创建闭包
// 不要在 Update/FixedUpdate 中创建 Lambda
```

**验证方式**：
1. Android Profiler 检查 GC Alloc/帧
2. 目标：单帧 GC Alloc < 1KB（加载场景除外）

---

## 四、平台兼容坑

### 坑 9：Android 文件路径大小写问题

**现象**：在 Windows/macOS 开发正常，Android 真机上报"文件不存在"。

**原因分析**：

Android 文件系统区分大小写（Windows 不区分）。如果 Collector 配置中的路径大小写与实际文件不一致，在 Android 上会找不到文件。

**解决方案**：

```csharp
// 1. 统一使用小写路径命名规范
// 资源文件名: player_panel.prefab (不是 PlayerPanel.prefab)

// 2. 在 CI 流程中加入大小写检查
// 脚本示例：
// 遍历 Assets 目录，检查是否存在仅大小写不同的文件名

// 3. 路径常量集中管理，避免硬编码
public static class AssetPaths
{
    public const string PlayerPanel = "Assets/Prefabs/UI/player_panel.prefab";
    public const string EnemyPrefab = "Assets/Prefabs/Game/enemy.prefab";
}
```

**验证方式**：
1. 在 Android 真机上进行完整资源加载测试
2. CI 流程中加入路径大小写校验脚本

---

### 坑 10：iOS 文件写入权限限制

**现象**：iOS 真机上下载资源后写入失败，报权限错误。

**原因分析**：

iOS 的沙盒机制限制了文件写入位置。`StreamingAssets` 目录在 iOS 上是只读的，不能直接写入。

**解决方案**：

```csharp
// YooAsset 默认使用 persistentDataPath 作为缓存目录
// 确保 CacheFileSystemParameters 使用正确的路径

var cacheParams = FileSystemParameters.CreateDefaultCacheFileSystemParameters();
// YooAsset 内部使用 Application.persistentDataPath，在 iOS 上是可写的

// 自定义缓存路径时注意：
string cachePath;
#if UNITY_IOS
    // iOS: 使用 Library 目录（NoBackup 避免被 iCloud 备份）
    cachePath = Path.Combine(Application.persistentDataPath, "bundles");
#elif UNITY_ANDROID
    // Android: 使用外部存储
    cachePath = Path.Combine(Application.persistentDataPath, "bundles");
#else
    cachePath = Path.Combine(Application.persistentDataPath, "bundles");
#endif
```

**验证方式**：
1. iOS 真机完整下载测试
2. 检查下载后文件确实存在于 persistentDataPath

---

### 坑 11：WebGL 不支持本地文件 IO

**现象**：WebGL 平台上 YooAsset 的 OfflinePlayMode 和 HostPlayMode 的本地缓存功能不工作。

**原因分析**：

WebGL 运行在浏览器沙盒中，不支持直接的文件系统 IO。不能像原生平台那样写入本地缓存文件。

**解决方案**：

```csharp
// WebGL 必须使用 WebPlayMode
#if UNITY_WEBGL && !UNITY_EDITOR
    var webParams = new WebPlayModeParameters();
    webParams.RemoteServices = remoteServices;
    await package.InitializeAsync(webParams);
#else
    // 原生平台使用 HostPlayMode
    var hostParams = new HostPlayModeParameters();
    // ...
    await package.InitializeAsync(hostParams);
#endif
```

**验证方式**：
1. WebGL 平台单独测试
2. 确认所有资源加载路径在 WebGL 上有效

---

## 速查表

| 症状 | 最可能原因 | 首步排查 |
|------|-----------|---------|
| 内存持续增长 | AssetHandle 未 Release（坑 1） | 检查所有 Load 是否有配对 Release |
| 共享资源不卸载 | Bundle 依赖引用计数（坑 2） | 检查所有引用同一共享资源的 handle |
| 资源被意外卸载 | UnloadUnusedAssets 误杀（坑 3） | 检查调用时机，先 Release 再 Unload |
| 加载到错误资源 | 多 Package 地址冲突（坑 4） | 检查 Address Rule，使用唯一地址 |
| 地址不存在 | Address Rule 变更（坑 5） | 确认 Manifest 版本匹配 |
| 异步返回 null | 没有等待完成（坑 6） | 使用 await + 检查 Status |
| 打开 UI 掉帧 | 同帧大量并发加载（坑 7） | 分帧加载或分批加载 |
| Android GC 卡顿 | 大量 Operation 对象（坑 8） | 使用缓存池，减少重复加载 |
| Android 文件找不到 | 路径大小写（坑 9） | 统一小写命名 |
| iOS 写入失败 | 沙盒权限（坑 10） | 使用 persistentDataPath |
| WebGL 不工作 | 不支持文件 IO（坑 11） | 使用 WebPlayMode |

---

## 最佳实践 DO / DON'T

### DO

- **Load/Release 严格配对**，建立引用计数管理器
- **异步加载后必须检查 Status**
- **同帧加载控制在 10 个以内**
- **Android 路径统一小写**
- **WebGL 使用 WebPlayMode**
- **场景切换时先 Release 再 UnloadUnusedAssets**

### DON'T

- 不要只 Destroy GameObject 不 Release handle
- 不要在持有资源时随意调用 `Resources.UnloadUnusedAssets()`
- 不要修改 Address Rule 后增量更新
- 不要在 Update 中创建 Lambda 闭包
- 不要假设 Editor 能跑真机就能跑

---

## 相关文档

- [[【笔记】YooAsset核心概念与架构]] — 概念基础
- [[【代码片段】YooAsset常用API速查]] — 正确的 API 使用方式
- [[【笔记】YooAsset打包管线与分包策略]] — Collector 配置
- [[【笔记】YooAsset热更新与版本管理]] — 热更流程
- [[【踩坑】HybridCLR接入常见坑]] — HybridCLR 踩坑（姊妹篇）

## 官方参考

- [YooAsset GitHub Issues](https://github.com/tuyoogame/YooAsset/issues)
- [YooAsset 官方文档](https://www.yooasset.com/)
```

- [ ] **Step 2: 提交**

```bash
cd "D:/GitHub/Doc"
git add "UnityKnowledge/35_高级主题/【踩坑】YooAsset中大型项目踩坑实录.md"
git commit -m "docs(UnityKnowledge): 新增 YooAsset 中大型项目踩坑实录

11 个高频坑，按内存管理/依赖加载/并发性能/平台兼容四类组织，
每条包含现象→原因分析→解决方案→验证方式四要素，
附速查表与 DO/DON'T 最佳实践"
```

---

## Task 6: 【笔记】YooAsset与HybridCLR协同方案

**Files:**
- Create: `UnityKnowledge/35_高级主题/【笔记】YooAsset与HybridCLR协同方案.md`

**Interfaces:**
- Consumes: Task 1~5 的 YooAsset 体系，现有 HybridCLR 文档
- Produces: 代码+资源双热更完整方案

- [ ] **Step 1: 创建文档，写入完整内容**

文件路径：`UnityKnowledge/35_高级主题/【笔记】YooAsset与HybridCLR协同方案.md`

完整内容：

```markdown
---
title: 【笔记】YooAsset与HybridCLR协同方案
tags: ["Unity", "YooAsset", "HybridCLR", "热更新", "代码热更", "资源热更"]
category: 高级主题
created: 2026-07-02
updated: 2026-07-02
description: YooAsset 资源热更 + HybridCLR 代码热更的完整协同方案：DLL 分发、AOT 元数据加载、版本对齐、打包集成、双热更启动流程
unity_version: 2021.3+
status: 待验证
validation: 基于官方文档与工程实践整理
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【笔记】YooAsset热更新与版本管理]]", "[[【踩坑】YooAsset中大型项目踩坑实录]]", "[[【踩坑】HybridCLR接入常见坑]]", "[[【笔记】HybridCLR构建管线与Generate工具链]]"]
author: llm
sources:
  - "YooAsset 官方文档 https://www.yooasset.com/"
  - "HybridCLR 官方文档 https://hybridclr.doc.code-philosophy.com/"
  - "[[【笔记】YooAsset热更新与版本管理]]"
  - "[[【踩坑】HybridCLR接入常见坑]]"
  - "[[【笔记】HybridCLR构建管线与Generate工具链]]"
---

# 【笔记】YooAsset 与 HybridCLR 协同方案

> HybridCLR 负责代码热更（DLL 级别），YooAsset 负责资源热更（AssetBundle 级别）+ DLL 文件分发。两者协同形成完整的"代码 + 资源"双热更方案。

## 文档定位

- **YooAsset 基础**：[[【笔记】YooAsset热更新与版本管理]] 已覆盖资源热更流程
- **HybridCLR 基础**：[[【笔记】HybridCLR构建管线与Generate工具链]] 已覆盖构建工具链
- **本文**：聚焦两者**如何协同工作** — DLL 分发、版本对齐、打包集成

---

## 一、整体架构

```
┌─────────────────────────────────────────────────────────┐
│                      CDN 服务器                           │
│  ├── DefaultPackage/                                    │
│  │   ├── 热更资源 Bundle（场景/UI/贴图/音频）            │
│  │   └── DLL Bundle（HotUpdate.dll + AOT 元数据 DLL）   │
│  └── manifest.hash / manifest.bytes                     │
└─────────────────────────────────────────────────────────┘
                          │
                    YooAsset 下载
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    客户端运行时                           │
│                                                         │
│  ① YooAsset 初始化 + 热更检查 + 下载                    │
│  ② 从 YooAsset 加载 HotUpdate.dll 字节                  │
│  ③ HybridCLR 加载 AOT 元数据 DLL                        │
│  ④ HybridCLR 加载 HotUpdate.dll                         │
│  ⑤ 通过反射调用热更入口                                  │
│  ⑥ 热更代码正常运行，通过 YooAsset 加载资源              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 二、DLL 热更流程

### 2.1 从 YooAsset 加载 DLL 字节

```csharp
using UnityEngine;
using YooAsset;
using HybridCLR;
using System.Reflection;
using System.Collections.Generic;
using Cysharp.Threading.Tasks;

public class HybridCLRDLLLoader
{
    private ResourcePackage _package;

    public async UniTask<bool> LoadHotUpdateDLLs()
    {
        _package = YooAssets.GetPackage("DefaultPackage");

        // 1. 加载 AOT 元数据 DLL
        var aotDllNames = new string[]
        {
            "mscorlib.dll",
            "System.dll",
            "System.Core.dll",
            "UnityEngine.CoreModule.dll",
            "Unity.Collections.dll"
        };

        foreach (var dllName in aotDllNames)
        {
            // 从 YooAsset 加载 DLL 的 TextAsset 或 RawFile
            var handle = _package.LoadAssetAsync<TextAsset>($"DLL_{dllName}");
            await handle.ToUniTask();

            if (handle.Status != EOperationStatus.Succeed)
            {
                Debug.LogError($"加载 AOT 元数据失败: {dllName}");
                return false;
            }

            byte[] dllBytes = (handle.AssetObject as TextAsset).bytes;

            // 加载到 HybridCLR
            int err = RuntimeApi.LoadMetadataForAOTAssembly(
                dllBytes,
                HomologousImageMode.SuperSet
            );

            if (err != 0)
            {
                Debug.LogError($"AOT 元数据加载失败: {dllName}, error={err}");
                return false;
            }

            Debug.Log($"AOT 元数据加载成功: {dllName}");
            handle.Release();
        }

        // 2. 加载热更 DLL
        var hotUpdateHandle = _package.LoadAssetAsync<TextAsset>("DLL_HotUpdate");
        await hotUpdateHandle.ToUniTask();

        if (hotUpdateHandle.Status != EOperationStatus.Succeed)
        {
            Debug.LogError("加载热更 DLL 失败");
            return false;
        }

        byte[] hotUpdateBytes = (hotUpdateHandle.AssetObject as TextAsset).bytes;

        // 3. Assembly.Load 加载热更 DLL
        Assembly hotUpdateAssembly = Assembly.Load(hotUpdateBytes);
        Debug.Log($"热更 DLL 加载成功: {hotUpdateAssembly.GetName().Name}");

        hotUpdateHandle.Release();

        // 4. 调用热更入口
        InvokeHotUpdateEntry(hotUpdateAssembly);
        return true;
    }

    private void InvokeHotUpdateEntry(Assembly assembly)
    {
        var entryType = assembly.GetType("HotUpdate.GameEntry");
        var entryMethod = entryType.GetMethod("Start");
        entryMethod?.Invoke(null, null);
    }
}
```

---

## 三、版本对齐策略

### 3.1 代码版本与资源版本绑定

```
问题：代码版本 v1.2 引用了新 Prefab "NewBoss"，但资源版本还停留在 v1.1
      → 加载 "NewBoss" 时找不到资源

方案：代码版本和资源版本必须同步更新
```

### 3.2 版本对齐实现

```csharp
public class VersionManager
{
    // 版本配置（服务端下发）
    [System.Serializable]
    public class VersionConfig
    {
        public string appVersion;           // App 整体版本
        public string resourceVersion;      // 资源版本
        public string hotUpdateVersion;     // 代码版本
        public int minAppVersionCode;       // 最低兼容 App 版本号
    }

    public async UniTask<bool> CheckVersion()
    {
        // 1. 请求服务端版本配置
        VersionConfig remoteConfig = await FetchVersionConfig();

        // 2. 检查 App 版本是否兼容
        if (remoteConfig.minAppVersionCode > GetCurrentAppVersionCode())
        {
            // App 版本过低，需要强制更新整包
            ShowForceUpdateDialog();
            return false;
        }

        // 3. 检查资源版本
        string localResourceVersion = PlayerPrefs.GetString("ResourceVersion", "");
        if (localResourceVersion != remoteConfig.resourceVersion)
        {
            // 执行 YooAsset 资源热更
            await RunResourceHotUpdate(remoteConfig.resourceVersion);
            PlayerPrefs.SetString("ResourceVersion", remoteConfig.resourceVersion);
        }

        // 4. 检查代码版本
        string localCodeVersion = PlayerPrefs.GetString("CodeVersion", "");
        if (localCodeVersion != remoteConfig.hotUpdateVersion)
        {
            // 加载新的热更 DLL
            await LoadNewHotUpdateDLL();
            PlayerPrefs.SetString("CodeVersion", remoteConfig.hotUpdateVersion);
        }

        return true;
    }
}
```

### 3.3 版本不一致的处理策略

| 场景 | 处理方式 |
|------|----------|
| 资源版本超前于代码版本 | 清理多余资源，使用旧版本 |
| 代码版本超前于资源版本 | 提示用户重试 / 回退代码版本 |
| 版本差距过大（跨大版本） | 强制更新整包 |
| AOT 元数据版本不匹配 | 强制更新整包（详见 [[【踩坑】HybridCLR接入常见坑]] 坑 3） |

---

## 四、打包集成

### 4.1 将 HybridCLR 产物接入 YooAsset Collector

在 YooAsset 的 Collector 配置中添加 DLL 收集组：

| Collector Group | Collect Path | Address Rule | Pack Rule | 说明 |
|-----------------|--------------|--------------|-----------|------|
| DLLGroup | `HybridCLRData/HotUpdateDlls/` | AddressByFileName | PackGroup | 热更 DLL |
| AOTDLLGroup | `HybridCLRData/AOTDlls/` | AddressByFileName | PackGroup | AOT 元数据 |

### 4.2 构建流程脚本

```python
# build_yooasset_hybridclr.py
# 完整构建流程：HybridCLR 编译 → YooAsset 打包 → CDN 上传

import subprocess
import os
import shutil

def full_build(unity_path, project_path, output_path):
    """一步到位的构建脚本"""

    # Step 1: HybridCLR Generate (Link.xml + MethodBridge + AOTDlls)
    print("=== Step 1: HybridCLR Generate ===")
    run_unity_method(unity_path, project_path,
        "HybridCLR.Editor.Commands.PrebuildCommand.GenerateAll")

    # Step 2: IL2CPP 编译首发包
    print("=== Step 2: IL2CPP Build ===")
    run_unity_method(unity_path, project_path,
        "UnityEditor.BuildPipeline.BuildPlayer")

    # Step 3: Generate AOTDlls
    print("=== Step 3: Generate AOTDlls ===")
    run_unity_method(unity_path, project_path,
        "HybridCLR.Editor.commands.AOTDllGenerator.Generate")

    # Step 4: 编译热更 DLL
    print("=== Step 4: Compile HotUpdate DLL ===")
    run_unity_method(unity_path, project_path,
        "HybridCLR.Editor.Commands.CompileDllCommand.CompileDll")

    # Step 5: YooAsset 打包
    print("=== Step 5: YooAsset Build ===")
    run_unity_method(unity_path, project_path,
        "YooAsset.Editor.AssetBundleBuilder.BuildPackage",
        extra_args=["-packageName", "DefaultPackage",
                     "-outputPath", output_path])

    # Step 6: 验证产物
    print("=== Step 6: Verify ===")
    verify_build_output(output_path)

    # Step 7: 上传 CDN
    print("=== Step 7: Upload CDN ===")
    upload_to_cdn(output_path)

def run_unity_method(unity_path, project_path, method, extra_args=None):
    args = [
        unity_path, "-batchmode", "-projectPath", project_path,
        "-executeMethod", method, "-quit"
    ]
    if extra_args:
        args.extend(extra_args)
    subprocess.run(args, check=True)

def verify_build_output(output_path):
    """验证产物完整性"""
    package_path = os.path.join(output_path, "DefaultPackage")
    required_files = ["manifest.hash", "manifest.bytes"]
    for f in required_files:
        path = os.path.join(package_path, f)
        if not os.path.exists(path):
            raise Exception(f"缺少必要文件: {path}")

    bundles_path = os.path.join(package_path, "Bundles")
    if not os.path.exists(bundles_path):
        raise Exception("缺少 Bundles 目录")

    print("构建产物验证通过")

def upload_to_cdn(output_path):
    """上传到 CDN（示例使用 rsync）"""
    subprocess.run([
        "rsync", "-avz", "--delete",
        f"{output_path}/DefaultPackage/",
        "user@cdn-server:/var/www/cdn/android/DefaultPackage/"
    ], check=True)
    print("CDN 上传完成")

full_build(
    "C:/Program Files/Unity/Hub/Editor/2022.3.20f1/Editor/Unity.exe",
    "D:/MyProject",
    "D:/BuildOutput"
)
```

---

## 五、完整代码示例：双热更启动流程

```csharp
using UnityEngine;
using YooAsset;
using HybridCLR;
using System.Reflection;
using Cysharp.Threading.Tasks;

public class GameBootstrap : MonoBehaviour
{
    private ResourcePackage _package;

    private async UniTaskVoid Start()
    {
        Debug.Log("=== 游戏启动 ===");

        // 阶段 1: YooAsset 初始化
        await InitYooAsset();

        // 阶段 2: YooAsset 热更检查
        await CheckAndUpdateResources();

        // 阶段 3: HybridCLR 加载 DLL
        await LoadHybridCLRDLLs();

        // 阶段 4: 进入游戏
        EnterGame();
    }

    /// <summary>
    /// 阶段 1: YooAsset 初始化
    /// </summary>
    private async UniTask InitYooAsset()
    {
        YooAssets.Initialize();
        _package = YooAssets.CreatePackage("DefaultPackage");
        YooAssets.SetDefaultPackage(_package);

        var initParams = new HostPlayModeParameters();
        initParams.BuildinFileSystemParameters =
            FileSystemParameters.CreateDefaultBuildinFileSystemParameters();
        initParams.CacheFileSystemParameters =
            FileSystemParameters.CreateDefaultCacheFileSystemParameters();
        initParams.RemoteServices = new GameRemoteServices();

        var op = _package.InitializeAsync(initParams);
        await op.ToUniTask();

        Debug.Log($"YooAsset 初始化: {op.Status}");
    }

    /// <summary>
    /// 阶段 2: 资源热更检查
    /// </summary>
    private async UniTask CheckAndUpdateResources()
    {
        // 获取远端版本号
        var versionOp = _package.RequestPackageVersionAsync();
        await versionOp.ToUniTask();
        if (versionOp.Status != EOperationStatus.Succeed) return;

        // 更新 Manifest
        var manifestOp = _package.UpdatePackageManifestAsync(versionOp.PackageVersion);
        await manifestOp.ToUniTask();
        if (manifestOp.Status != EOperationStatus.Succeed) return;

        // 检查是否需要下载
        var downloader = _package.CreateResourceDownloader(10, 3);
        if (downloader.TotalDownloadCount == 0)
        {
            Debug.Log("资源已是最新");
            return;
        }

        // 开始下载
        downloader.BeginDownload();
        await downloader.ToUniTask();

        Debug.Log($"资源更新完成: {downloader.Status}");
    }

    /// <summary>
    /// 阶段 3: 加载 HybridCLR DLL
    /// </summary>
    private async UniTask LoadHybridCLRDLLs()
    {
        // 3.1 加载 AOT 元数据
        string[] aotDlls = {
            "mscorlib.dll", "System.dll", "System.Core.dll",
            "UnityEngine.CoreModule.dll", "Unity.Collections.dll"
        };

        foreach (var dllName in aotDlls)
        {
            var handle = _package.LoadAssetAsync<TextAsset>($"AOT_{dllName}");
            await handle.ToUniTask();

            if (handle.Status == EOperationStatus.Succeed)
            {
                RuntimeApi.LoadMetadataForAOTAssembly(
                    (handle.AssetObject as TextAsset).bytes,
                    HomologousImageMode.SuperSet
                );
                Debug.Log($"AOT 元数据加载: {dllName}");
            }
            handle.Release();
        }

        // 3.2 加载热更 DLL
        var hotHandle = _package.LoadAssetAsync<TextAsset>("HotUpdate.dll");
        await hotHandle.ToUniTask();

        if (hotHandle.Status == EOperationStatus.Succeed)
        {
            Assembly asm = Assembly.Load((hotHandle.AssetObject as TextAsset).bytes);
            Debug.Log($"热更 DLL 加载: {asm.GetName().Version}");
        }
        hotHandle.Release();
    }

    /// <summary>
    /// 阶段 4: 进入游戏
    /// </summary>
    private void EnterGame()
    {
        // 调用热更代码入口
        var assembly = AppDomain.CurrentDomain.GetAssemblies()
            .First(a => a.GetName().Name == "HotUpdate");
        var entryType = assembly.GetType("HotUpdate.GameEntry");
        entryType.GetMethod("Start")?.Invoke(null, null);
    }
}
```

---

## 六、踩坑要点

> 详细踩坑见 [[【踩坑】HybridCLR接入常见坑]] 和 [[【踩坑】YooAsset中大型项目踩坑实录]]。

| 问题 | 原因 | 解法 |
|------|------|------|
| DLL 加载后泛型崩溃 | AOT 元数据不完整 | 补充所有必需的 AOT 元数据 DLL |
| DLL 版本不匹配 | 首发包更新后未同步元数据 | 每次打首发包重新生成 AOT 元数据 |
| 资源找不到 | 代码版本和资源版本不同步 | 版本绑定，同时更新 |
| 构建产物中缺少 DLL | Collector 未配置 DLL 收集 | 添加 DLLGroup 到 Collector |
| DLL 被反编译 | 未加密 | 使用 IEncryptionServices 加密 DLL |

---

## 七、落地 Checklist

### 打包阶段
- [ ] HybridCLR Generate/All 已执行
- [ ] AOT 元数据 DLL 已生成
- [ ] 热更 DLL 已编译
- [ ] YooAsset Collector 包含 DLL 收集组
- [ ] YooAsset 打包完成
- [ ] 构建产物中包含 DLL Bundle
- [ ] CDN 上传完成并校验

### 运行阶段
- [ ] YooAsset 初始化成功（HostPlayMode）
- [ ] 资源热更检查完成
- [ ] AOT 元数据全部加载成功
- [ ] 热更 DLL Assembly.Load 成功
- [ ] 热更入口方法调用成功
- [ ] 真机测试热更代码正常运行

### 版本管理
- [ ] 代码版本与资源版本绑定
- [ ] 版本配置服务端可控制
- [ ] 强制更新机制可用
- [ ] 版本回退方案已准备

---

## 相关文档

- [[【笔记】YooAsset核心概念与架构]] — YooAsset 架构基础
- [[【笔记】YooAsset热更新与版本管理]] — 资源热更详解
- [[【踩坑】YooAsset中大型项目踩坑实录]] — YooAsset 踩坑
- [[【踩坑】HybridCLR接入常见坑]] — HybridCLR 13 个高频坑
- [[【笔记】HybridCLR构建管线与Generate工具链]] — Generate 工具链详解

## 官方参考

- [YooAsset 官方文档](https://www.yooasset.com/)
- [HybridCLR 官方文档](https://hybridclr.doc.code-philosophy.com/)
- [YooAsset GitHub](https://github.com/tuyoogame/YooAsset)
- [HybridCLR GitHub](https://github.com/focus-creative-games/hybridclr)
```

- [ ] **Step 2: 提交**

```bash
cd "D:/GitHub/Doc"
git add "UnityKnowledge/35_高级主题/【笔记】YooAsset与HybridCLR协同方案.md"
git commit -m "docs(UnityKnowledge): 新增 YooAsset 与 HybridCLR 协同方案

涵盖整体架构图、DLL 热更流程(从 YooAsset 加载 DLL)、
版本对齐策略(代码版本与资源版本绑定)、
打包集成(HybridCLR 产物接入 YooAsset Collector)、
完整双热更启动流程代码示例、落地 Checklist"
```

---

## Task 7: 【笔记】YooAsset与Addressables方案对比

**Files:**
- Create: `UnityKnowledge/35_高级主题/【笔记】YooAsset与Addressables方案对比.md`

**Interfaces:**
- Consumes: Task 1~5 的 YooAsset 体系，现有 Addressables 文档
- Produces: 技术选型参考文档

- [ ] **Step 1: 创建文档，写入完整内容**

文件路径：`UnityKnowledge/35_高级主题/【笔记】YooAsset与Addressables方案对比.md`

完整内容：

```markdown
---
title: 【笔记】YooAsset与Addressables方案对比
tags: ["Unity", "YooAsset", "Addressables", "资源管理", "方案对比", "技术选型"]
category: 高级主题
created: 2026-07-02
updated: 2026-07-02
description: YooAsset 与 Unity Addressables 横向对比：架构设计、工作流、性能、热更新、团队成本与适用场景推荐
unity_version: 2021.3+
status: 待验证
validation: 基于官方文档与工程实践整理
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【代码片段】YooAsset常用API速查]]", "[[【踩坑】YooAsset中大型项目踩坑实录]]", "[[【教程】资源管线-Addressables]]"]
author: llm
sources:
  - "YooAsset 官方文档 https://www.yooasset.com/"
  - "Unity Addressables 官方文档 https://docs.unity3d.com/Packages/com.unity.addressables@latest"
  - "[[【笔记】YooAsset核心概念与架构]]"
  - "[[【教程】资源管线-Addressables]]"
---

# 【笔记】YooAsset 与 Addressables 方案对比

> 从架构设计、工作流、性能、热更新、团队成本五个维度横向对比 YooAsset 和 Addressables，帮助技术选型。

## 文档定位

- 已有 Addressables 文档：[[【教程】资源管线-Addressables]]
- 已有 YooAsset 文档：[[【笔记】YooAsset核心概念与架构]]
- 本文聚焦两者的**差异对比**和**选型建议**

---

## 一、架构对比

### 1.1 设计哲学

| 维度 | Addressables | YooAsset |
|------|-------------|----------|
| 设计理念 | "Address"（地址）驱动，资源与位置解耦 | "Package"（资源包）驱动，逻辑分组优先 |
| 核心抽象 | AssetReference / Address | ResourcePackage / AssetHandle |
| 管理粒度 | 全局 Catalog 统一管理 | 多 Package 独立管理 |
| 配置方式 | Addressables Groups 窗口（GUI） | AssetBundleCollector 配置（GUI/代码） |

### 1.2 资源寻址

```
Addressables:
  地址: "Assets/Prefabs/Player" 或 "Player"
  通过 AddressableAssetSettings 配置
  运行时: Addressables.LoadAssetAsync<GameObject>("Player")

YooAsset:
  地址: 由 Address Rule 决定（FileName / FilePath / GroupPath）
  通过 Collector 配置
  运行时: package.LoadAssetAsync<GameObject>("Player")
```

### 1.3 依赖管理

| 维度 | Addressables | YooAsset |
|------|-------------|----------|
| 依赖解析 | 自动解析，基于 Catalog | 自动解析，基于 Manifest |
| 依赖冗余 | SBP（Scriptable Build Pipeline）处理 | Pack Rule + Collector Group 处理 |
| 冲突处理 | 全局唯一地址 | 需注意多 Package 地址冲突 |

---

## 二、工作流对比

### 2.1 打包配置

```
Addressables 工作流：
  1. 在 Addressables Groups 窗口创建 Group
  2. 将资源拖入 Group
  3. 配置 Bundle 模式（Pack Together / Pack Separately）
  4. New Build → Build → New Build → Default Build Script

YooAsset 工作流：
  1. 在 AssetBundleCollector 窗口创建 Collector Group
  2. 配置 Collect Path / Address Rule / Pack Rule
  3. 在 AssetBundleBuilder 窗口配置构建参数
  4. Build
```

### 2.2 调试体验

| 维度 | Addressables | YooAsset |
|------|-------------|----------|
| Editor 模拟 | Play Mode → Use Asset Database | EditorSimulateMode |
| 分析工具 | Addressables Analyze | BuildReport |
| 资源引用查看 | Addressables Groups 窗口 | BuildReport + Collector 窗口 |
| 加载日志 | 需开启 Diagnostic | 内建 Operation 日志 |

### 2.3 发布流程

```
Addressables:
  Build → 上传到 CDN（手动或 CCD）
  需要自己管理 Catalog 版本

YooAsset:
  Build → 自动生成 manifest.hash / manifest.bytes
  上传整个 Package 目录到 CDN
  版本管理内建
```

---

## 三、性能对比

### 3.1 加载速度

| 场景 | Addressables | YooAsset |
|------|-------------|----------|
| 首次加载（需读取 AB） | 基准 | 相近 |
| 重复加载（已缓存） | 基准 | 相近 |
| 批量加载 | 无特殊优化 | 支持 LoadAssetsAsync |
| 场景加载 | 封装 SceneManager | 原生封装 LoadSceneAsync |

> 两者的加载性能本质都依赖 AssetBundle，差异不大。优化空间主要在打包策略和引用管理。

### 3.2 内存占用

| 维度 | Addressables | YooAsset |
|------|-------------|----------|
| 引用管理 | ReferenceCounting（内建） | AssetHandle 引用计数 |
| 卸载粒度 | 按 Group 卸载 | 按 Package / Asset 卸载 |
| 内存泄漏风险 | 中（需配对 Load/Release） | 中（同上，见 [[【踩坑】YooAsset中大型项目踩坑实录]]） |

### 3.3 包体大小

| 维度 | Addressables | YooAsset |
|------|-------------|----------|
| 默认压缩 | LZ4 | LZ4 |
| 依赖冗余 | SBP 优化 | Pack Rule 控制 |
| Manifest 大小 | Catalog JSON | manifest.bytes（更紧凑） |

---

## 四、热更新对比

### 4.1 Addressables CCD（Cloud Content Delivery）

| 维度 | 说明 |
|------|------|
| 基础设施 | Unity 云服务，海外节点多 |
| 成本 | 按流量计费，大项目成本较高 |
| 国内访问 | 海外 CDN，延迟高 |
| 灰度发布 | 需自行配置 |
| 断点续传 | 不原生支持 |

### 4.2 YooAsset CDN

| 维度 | 说明 |
|------|------|
| 基础设施 | 自建 CDN，任意服务商 |
| 成本 | CDN 流量费（可控） |
| 国内访问 | 可选国内 CDN，延迟低 |
| 灰度发布 | 自定义灵活 |
| 断点续传 | 原生支持 |
| 版本管理 | manifest.hash + manifest.bytes，内建版本对比 |

### 4.3 热更能力对比

| 能力 | Addressables | YooAsset |
|------|-------------|----------|
| 增量更新 | 支持（需要手动管理 Catalog diff） | 支持（Manifest 自动 diff） |
| 断点续传 | ❌ 不原生支持 | ✅ 原生支持 |
| 分包热更 | 有限支持（多 Catalog） | 原生支持（多 Package） |
| 版本回退 | 困难 | 可实现 |
| CDN 自由度 | 低（CCD 或自行搭建） | 高（任意 CDN） |

---

## 五、团队成本对比

### 5.1 学习曲线

| 维度 | Addressables | YooAsset |
|------|-------------|----------|
| 上手难度 | 中（概念较多） | 中低（概念直观） |
| 中文文档 | 少（主要为英文） | 丰富（官方中文文档） |
| 社区支持 | Unity 官方论坛 | 国内社区活跃 |
| 教程资源 | Unity Learn + 社区 | 官方文档 + B 站教程 |

### 5.2 维护成本

| 维度 | Addressables | YooAsset |
|------|-------------|----------|
| 版本升级 | Unity 版本绑定 | 独立版本，可跨 Unity 版本 |
| 兼容性 | 高（Unity 官方维护） | 高（积极维护） |
| 工具链 | 集成在 Unity 中 | 独立 Package |
| 自动化 | 支持 CI/CD | 支持 Jenkins / Python |

---

## 六、适用场景推荐表

| 场景 | 推荐 | 理由 |
|------|------|------|
| 海外上线 | Addressables | CCD 海外节点 + Unity 官方支持 |
| 国内上线 | **YooAsset** | 自由选择国内 CDN，延迟低 |
| 单机游戏 | Addressables | 无热更需求，Unity 原生方案足够 |
| 需要热更的网游 | **YooAsset** | 热更链路完整，断点续传，分包灵活 |
| HybridCLR 协同 | **YooAsset** | DLL 分发通过 YooAsset 统一管理 |
| 小型团队 | Addressables | Unity 官方，学习资料多 |
| 中大型团队 | **YooAsset** | 分包策略灵活，CI/CD 集成好 |
| WebGL | 两者均可 | Addressables Web 模式 / YooAsset WebPlayMode |

---

## 七、结论与推荐

### 7.1 选型决策树

```
是否需要热更新？
├── 否 → Addressables（Unity 官方，简单直接）
└── 是
    ├── 海外项目 → Addressables + CCD（官方一体化）
    └── 国内项目 → YooAsset（热更链路完整，CDN 自由）
```

### 7.2 综合评价

| 维度 | Addressables | YooAsset | 优势方 |
|------|-------------|----------|--------|
| 架构设计 | 优秀 | 优秀 | 平手 |
| 工作流 | 好 | 好 | 平手 |
| 性能 | 好 | 好 | 平手 |
| 热更新 | 一般 | 优秀 | YooAsset |
| 国内生态 | 一般 | 优秀 | YooAsset |
| 团队学习 | 好 | 好 | 平手 |
| 官方背书 | Unity 官方 | 社区 | Addressables |

### 7.3 迁移建议

如果从 Addressables 迁移到 YooAsset：
1. 资源寻址方式相近，迁移成本低
2. Group → Collector Group 概念映射直观
3. 主要工作在 Collector 配置和 CDN 搭建
4. 参见 [[【笔记】YooAsset打包管线与分包策略]] 快速上手

如果从 YooAsset 迁移到 Addressables：
1. Package → Group 映射
2. 需要自行搭建热更流程（如果之前依赖 YooAsset 热更）
3. 热更能力可能降级（断点续传等需自行实现）

---

## 相关文档

- [[【笔记】YooAsset核心概念与架构]] — YooAsset 架构详解
- [[【代码片段】YooAsset常用API速查]] — API 对比参考
- [[【踩坑】YooAsset中大型项目踩坑实录]] — YooAsset 坑点
- [[【教程】资源管线-Addressables]] — Addressables 教程

## 官方参考

- [YooAsset 官方文档](https://www.yooasset.com/)
- [Unity Addressables 文档](https://docs.unity3d.com/Packages/com.unity.addressables@latest)
- [YooAsset GitHub](https://github.com/tuyoogame/YooAsset)
```

- [ ] **Step 2: 提交**

```bash
cd "D:/GitHub/Doc"
git add "UnityKnowledge/35_高级主题/【笔记】YooAsset与Addressables方案对比.md"
git commit -m "docs(UnityKnowledge): 新增 YooAsset 与 Addressables 方案对比

从架构设计、工作流、性能、热更新、团队成本五个维度横向对比，
包含热更能力差异表、适用场景推荐表、选型决策树、迁移建议"
```

---

## Task 8: YooAsset专题索引

**Files:**
- Create: `UnityKnowledge/35_高级主题/YooAsset专题索引.md`

**Interfaces:**
- Consumes: All 7 documents from Task 1~7
- Produces: 导航页，汇总所有 YooAsset 文档

- [ ] **Step 1: 创建文档，写入完整内容**

文件路径：`UnityKnowledge/35_高级主题/YooAsset专题索引.md`

完整内容：

```markdown
---
title: YooAsset专题索引
tags: ["Unity", "YooAsset", "资源管理", "热更新", "专题索引"]
category: 高级主题/专题索引
created: 2026-07-02
updated: 2026-07-02
description: YooAsset 专题索引，汇总核心架构、API 速查、打包管线、热更新、踩坑实录、HybridCLR 协同、方案对比文档。
status: 待验证
validation: 未实测
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【代码片段】YooAsset常用API速查]]", "[[【笔记】YooAsset打包管线与分包策略]]", "[[【笔记】YooAsset热更新与版本管理]]", "[[【踩坑】YooAsset中大型项目踩坑实录]]", "[[【笔记】YooAsset与HybridCLR协同方案]]", "[[【笔记】YooAsset与Addressables方案对比]]"]
author: llm
---

# YooAsset 专题索引

> YooAsset 专题索引，汇总核心架构、API 速查、打包管线、热更新、踩坑实录、HybridCLR 协同、方案对比文档。

## 文档定位

本文档从知识索引角度组织 YooAsset 相关文档，适合在资源管理选型、热更新落地、踩坑定位时快速导航。

## 专题概览

- 收录文档数：7
- 覆盖目录数：2（`60_第三方库` + `35_高级主题`）
- 文档类型分布：笔记 5 篇，代码片段 1 篇，踩坑记录 1 篇

## 推荐阅读顺序

1. [【笔记】YooAsset核心概念与架构](../60_第三方库/【笔记】YooAsset核心概念与架构.md) - 入口页，建立 YooAsset 四层模型和三大管线的全局认知。
2. [【代码片段】YooAsset常用API速查](../60_第三方库/【代码片段】YooAsset常用API速查.md) - 日常开发 API 字典，按场景组织、复制即用。
3. [【笔记】YooAsset打包管线与分包策略](./【笔记】YooAsset打包管线与分包策略.md) - Collector 收集系统、Packer 打包模式、中大型项目分包策略。
4. [【笔记】YooAsset热更新与版本管理](./【笔记】YooAsset热更新与版本管理.md) - 热更全链路：版本检查→差异下载→断点续传→CDN 部署。
5. [【踩坑】YooAsset中大型项目踩坑实录](./【踩坑】YooAsset中大型项目踩坑实录.md) - 11 个高频坑：内存泄漏/引用计数/并发瓶颈/平台兼容。
6. [【笔记】YooAsset与HybridCLR协同方案](./【笔记】YooAsset与HybridCLR协同方案.md) - 代码热更 + 资源热更的完整双热更方案。
7. [【笔记】YooAsset与Addressables方案对比](./【笔记】YooAsset与Addressables方案对比.md) - 横向技术选型评测与推荐。

## 按文档类型

### 笔记

- [【笔记】YooAsset核心概念与架构](../60_第三方库/【笔记】YooAsset核心概念与架构.md) - `第三方库` - 四层模型(Package/Collection/Bundle/Asset)、三大管线(Collector/Packer/Loader)、四种运行模式。
- [【笔记】YooAsset打包管线与分包策略](./【笔记】YooAsset打包管线与分包策略.md) - `高级主题` - Collector 配置详解、Packer 打包模式、中大型项目分包策略设计、加密方案。
- [【笔记】YooAsset热更新与版本管理](./【笔记】YooAsset热更新与版本管理.md) - `高级主题` - 版本号机制、增量更新、断点续传、CDN 部署规范、完整热更代码示例。
- [【笔记】YooAsset与HybridCLR协同方案](./【笔记】YooAsset与HybridCLR协同方案.md) - `高级主题` - DLL 热更流程、版本对齐、打包集成、双热更启动流程。
- [【笔记】YooAsset与Addressables方案对比](./【笔记】YooAsset与Addressables方案对比.md) - `高级主题` - 架构/工作流/性能/热更新/团队成本五维对比。

### 代码片段

- [【代码片段】YooAsset常用API速查](../60_第三方库/【代码片段】YooAsset常用API速查.md) - `第三方库` - 初始化/加载/释放/场景/下载/资源信息 8 个场景的代码模板。

### 踩坑记录

- [【踩坑】YooAsset中大型项目踩坑实录](./【踩坑】YooAsset中大型项目踩坑实录.md) - `高级主题` - 11 个高频坑（内存管理/依赖加载/并发性能/平台兼容），每条含现象→原因→解决→验证。

## 按目录

### 60_第三方库

- [【笔记】YooAsset核心概念与架构](../60_第三方库/【笔记】YooAsset核心概念与架构.md)
- [【代码片段】YooAsset常用API速查](../60_第三方库/【代码片段】YooAsset常用API速查.md)

### 35_高级主题

- [【笔记】YooAsset打包管线与分包策略](./【笔记】YooAsset打包管线与分包策略.md)
- [【笔记】YooAsset热更新与版本管理](./【笔记】YooAsset热更新与版本管理.md)
- [【踩坑】YooAsset中大型项目踩坑实录](./【踩坑】YooAsset中大型项目踩坑实录.md)
- [【笔记】YooAsset与HybridCLR协同方案](./【笔记】YooAsset与HybridCLR协同方案.md)
- [【笔记】YooAsset与Addressables方案对比](./【笔记】YooAsset与Addressables方案对比.md)

## 知识图谱

```
YooAsset 知识体系
│
├── 基础层
│   ├── 核心概念 ──── [[【笔记】YooAsset核心概念与架构]]
│   └── API 速查 ──── [[【代码片段】YooAsset常用API速查]]
│
├── 构建层
│   ├── 收集配置 ──── [[【笔记】YooAsset打包管线与分包策略]] §1
│   ├── 打包模式 ──── [[【笔记】YooAsset打包管线与分包策略]] §2
│   ├── 分包策略 ──── [[【笔记】YooAsset打包管线与分包策略]] §3
│   └── 加密方案 ──── [[【笔记】YooAsset打包管线与分包策略]] §4
│
├── 运行层
│   ├── 热更流程 ──── [[【笔记】YooAsset热更新与版本管理]]
│   ├── 版本管理 ──── [[【笔记】YooAsset热更新与版本管理]] §2
│   ├── 断点续传 ──── [[【笔记】YooAsset热更新与版本管理]] §4
│   └── CDN 部署 ──── [[【笔记】YooAsset热更新与版本管理]] §5
│
├── 实战层
│   ├── 踩坑实录 ──── [[【踩坑】YooAsset中大型项目踩坑实录]]
│   ├── HybridCLR ── [[【笔记】YooAsset与HybridCLR协同方案]]
│   └── 方案选型 ──── [[【笔记】YooAsset与Addressables方案对比]]
│
└── 关联专题
    ├── HybridCLR ── [[【踩坑】HybridCLR接入常见坑]]
    ├── HybridCLR ── [[【笔记】HybridCLR构建管线与Generate工具链]]
    ├── Addressables ── [[【教程】资源管线-Addressables]]
    └── 热更选型 ──── [[【设计原理】热更新方案对比]]
```

## 相关链接

- [[【笔记】YooAsset核心概念与架构]]
- [[【代码片段】YooAsset常用API速查]]
- [[【笔记】YooAsset打包管线与分包策略]]
- [[【笔记】YooAsset热更新与版本管理]]
- [[【踩坑】YooAsset中大型项目踩坑实录]]
- [[【笔记】YooAsset与HybridCLR协同方案]]
- [[【笔记】YooAsset与Addressables方案对比]]
- [[【踩坑】HybridCLR接入常见坑]]
- [[【笔记】HybridCLR构建管线与Generate工具链]]
- [[【教程】资源管线-Addressables]]
- [[【设计原理】热更新方案对比]]
- [[../00_元数据与模板/学习路径导航]]
```

- [ ] **Step 2: 提交**

```bash
cd "D:/GitHub/Doc"
git add "UnityKnowledge/35_高级主题/YooAsset专题索引.md"
git commit -m "docs(UnityKnowledge): 新增 YooAsset 专题索引

汇总 7 篇 YooAsset 文档导航，包含推荐阅读顺序、按类型/目录分类、
知识图谱（基础层/构建层/运行层/实战层/关联专题）"
```

---

## 完成后：更新 index.md

所有 8 篇文档创建完成后，更新 `UnityKnowledge/index.md` 添加 YooAsset 相关文档条目。

- [ ] **Final Step: 更新 index.md 并提交**

```bash
cd "D:/GitHub/Doc"
# 手动在 index.md 中添加 YooAsset 文档条目后
git add "UnityKnowledge/index.md"
git commit -m "docs(UnityKnowledge): 更新 index.md 添加 YooAsset 文档条目"
```
