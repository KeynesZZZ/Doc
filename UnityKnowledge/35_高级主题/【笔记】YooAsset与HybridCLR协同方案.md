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
