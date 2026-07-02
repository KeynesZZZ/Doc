---
title: 【最佳实践】Addressables性能优化
tags: ["Unity", "性能优化", "加载优化", "Addressables", "最佳实践"]
category: 性能优化/加载优化
created: "2026-03-07 10:00"
updated: "2026-05-29 00:00"
description: Unity Addressables 资源管理系统性能优化指南，涵盖异步加载、内存管理、AssetBundle优化
unity_version: 2021.3+
status: 待验证
validation: Demo验证
related: ["[[【最佳实践】资源预加载策略]]", "[[【实战案例】加载时间优化实战]]", "[[../../32_内存管理/【最佳实践】资源卸载指南]]"]
author: llm
---

# 【最佳实践】Addressables性能优化

> Addressables 资源管理系统的性能优化实战 `#性能优化` `#加载优化` `#Addressables` `#最佳实践`

## 文档定位

Unity Addressables资源管理系统的性能优化指南，涵盖异步加载优化（避免同步加载阻塞）、内存管理（正确Release资源、引用计数）、AssetBundle优化（分组策略、压缩格式选择、缓存策略），以及加载性能监控和常见性能陷阱排查。

---

## 1. Addressables vs Resources

### 1.1 为什么选择 Addressables

```
┌─────────────────────────────────────────────────────────────┐
│              Resources.Load vs Addressables                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Resources.Load 问题：                                      │
│  ├── 同步阻塞主线程                                        │
│  ├── 无法精确控制内存                                      │
│  ├── 增加初始包体大小                                      │
│  └── 无法热更新                                            │
│                                                             │
│  Addressables 优势：                                        │
│  ├── ✅ 异步加载，不阻塞主线程                             │
│  ├── ✅ 引用计数内存管理                                   │
│  ├── ✅ 本地/远程资源统一接口                              │
│  ├── ✅ 支持热更新和 DLC                                   │
│  └── ✅ 自动依赖管理                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 性能对比

| 方案 | 首包大小 | 加载时间 | 灵活性 | 推荐场景 |
|------|---------|---------|--------|---------|
| Resources.Load | 大（全打入首包） | 快（同步阻塞主线程） | 低 | 小型项目、原型 |
| AssetBundle | 小（按需下载） | 中（异步） | 高 | 大型项目、热更 |
| Addressables | 小（按需下载） | 中（异步） | 高 | 所有项目（推荐） |

---

## 2. 异步加载优化

### 2.0 初始化前提（必须）

> **在使用任何 Addressables API 之前，必须确保 `Addressables.InitializeAsync()` 已完成。**

首次调用 `LoadAssetAsync` 时会自动触发惰性初始化，但这可能导致不可预期的延迟。建议在启动流程中显式调用：

```csharp
/// <summary>
/// Addressables 初始化管理器
/// </summary>
public class AddressablesInitializer : MonoBehaviour
{
    private static bool isInitialized;

    /// <summary>
    /// 确保初始化完成（可多次调用，只初始化一次）
    /// </summary>
    public static async Task EnsureInitialized()
    {
        if (isInitialized) return;

        var handle = Addressables.InitializeAsync();
        await handle.Task;

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            isInitialized = true;
            Debug.Log("[Addressables] 初始化完成");
        }
        else
        {
            Debug.LogError("[Addressables] 初始化失败!");
        }
    }
}
```

**初始化内容包括**：

| 内容 | 说明 | 耗时 |
|------|------|------|
| 加载 Catalog | 资源地址映射表（address → path） | 50-200ms（取决于资源数量） |
| 初始化 Resource Manager | 内部资源管理器 | < 10ms |
| 检查缓存目录 | 确认本地缓存可用 | < 5ms |

**注意事项**：
- Catalog 文件随包发布（`catalog_*.json` + `.hash`），大型项目可能达到数 MB
- Catalog 解析在主线程进行，可在 Splash Screen 或加载界面阶段完成
- 初始化失败会导致后续所有加载 API 抛异常，必须处理失败情况

### 2.1 基础异步加载

```csharp
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;
using System.Collections.Generic;
using System.Threading.Tasks;

/// <summary>
/// Addressables 异步加载管理器
/// </summary>
public class AddressablesLoader : MonoBehaviour
{
    private Dictionary<string, AsyncOperationHandle> loadedHandles = new Dictionary<string, AsyncOperationHandle>();
    private Dictionary<string, int> referenceCounts = new Dictionary<string, int>();

    /// <summary>
    /// 异步加载单个资源
    /// </summary>
    public async Task<T> LoadAssetAsync<T>(string address) where T : Object
    {
        // 检查缓存
        if (loadedHandles.TryGetValue(address, out var existingHandle))
        {
            if (existingHandle.IsDone)
            {
                referenceCounts[address]++;
                return existingHandle.Result as T;
            }
            await existingHandle.Task;
            referenceCounts[address]++;
            return existingHandle.Result as T;
        }

        // 异步加载
        var handle = Addressables.LoadAssetAsync<T>(address);
        loadedHandles[address] = handle;
        referenceCounts[address] = 1;

        await handle.Task;

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            return handle.Result;
        }

        Debug.LogError($"Failed to load asset: {address}");
        loadedHandles.Remove(address);
        referenceCounts.Remove(address);
        return null;
    }

    /// <summary>
    /// 批量异步加载
    /// </summary>
    public async Task<List<T>> LoadAssetsAsync<T>(IList<string> addresses) where T : Object
    {
        var results = new List<T>();
        var tasks = new List<Task<T>>();

        foreach (var address in addresses)
        {
            tasks.Add(LoadAssetAsync<T>(address));
        }

        var loadedAssets = await Task.WhenAll(tasks);
        results.AddRange(loadedAssets.Where(a => a != null));

        return results;
    }

    /// <summary>
    /// 释放资源
    /// </summary>
    public void ReleaseAsset(string address)
    {
        if (!referenceCounts.TryGetValue(address, out var count))
            return;

        count--;
        referenceCounts[address] = count;

        if (count <= 0)
        {
            if (loadedHandles.TryGetValue(address, out var handle))
            {
                Addressables.Release(handle);
                loadedHandles.Remove(address);
            }
            referenceCounts.Remove(address);
        }
    }

    /// <summary>
    /// 释放所有资源
    /// </summary>
    public void ReleaseAll()
    {
        foreach (var handle in loadedHandles.Values)
        {
            Addressables.Release(handle);
        }
        loadedHandles.Clear();
        referenceCounts.Clear();
    }

    private void OnDestroy()
    {
        ReleaseAll();
    }
}
```

### 2.2 带进度的异步加载

```csharp
/// <summary>
/// 带进度回调的异步加载
/// </summary>
public class ProgressLoader : MonoBehaviour
{
    /// <summary>
    /// 加载资源并报告进度
    /// </summary>
    public async Task<T> LoadWithProgress<T>(
        string address,
        System.Action<float> onProgress = null) where T : Object
    {
        var handle = Addressables.LoadAssetAsync<T>(address);

        // 直接 await handle.Task，无需手动轮询
        // 如需进度回调，用协程方式（见下方补充）
        await handle.Task;

        onProgress?.Invoke(1f);

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            return handle.Result;
        }

        return null;
    }

    /// <summary>
    /// 批量加载并报告总进度
    /// </summary>
    public async Task<List<T>> LoadMultipleWithProgress<T>(
        IList<string> addresses,
        System.Action<float> onProgress = null) where T : Object
    {
        var results = new List<T>();
        int totalCount = addresses.Count;
        int loadedCount = 0;

        foreach (var address in addresses)
        {
            var asset = await LoadWithProgress<T>(address, progress =>
            {
                // 计算总进度
                float overallProgress = (loadedCount + progress) / totalCount;
                onProgress?.Invoke(overallProgress);
            });

            if (asset != null)
            {
                results.Add(asset);
            }
            loadedCount++;
        }

        return results;
    }
}
```

### 2.3 预加载策略

```csharp
/// <summary>
/// 智能预加载管理器
/// </summary>
public class SmartPreloader : MonoBehaviour
{
    [System.Serializable]
    public class PreloadGroup
    {
        public string groupName;
        public string[] assetAddresses;
        public bool preloadOnStart;
        public bool showProgress;
    }

    [Header("Preload Groups")]
    [SerializeField] private PreloadGroup[] preloadGroups;

    private Dictionary<string, Object> preloadedAssets = new Dictionary<string, Object>();
    private bool isPreloading;

    /// <summary>
    /// 预加载指定组
    /// </summary>
    public async Task<bool> PreloadGroup(string groupName, System.Action<float> onProgress = null)
    {
        var group = System.Array.Find(preloadGroups, g => g.groupName == groupName);
        if (group == null)
        {
            Debug.LogError($"Preload group not found: {groupName}");
            return false;
        }

        return await PreloadAssets(group.assetAddresses, onProgress);
    }

    /// <summary>
    /// 预加载资源列表
    /// </summary>
    public async Task<bool> PreloadAssets(string[] addresses, System.Action<float> onProgress = null)
    {
        if (isPreloading)
        {
            Debug.LogWarning("Already preloading");
            return false;
        }

        isPreloading = true;
        int loadedCount = 0;
        int totalCount = addresses.Length;

        foreach (var address in addresses)
        {
            if (preloadedAssets.ContainsKey(address))
            {
                loadedCount++;
                continue;
            }

            var handle = Addressables.LoadAssetAsync<Object>(address);
            await handle.Task;

            if (handle.Status == AsyncOperationStatus.Succeeded)
            {
                preloadedAssets[address] = handle.Result;
            }
            else
            {
                Debug.LogWarning($"Failed to preload: {address}");
            }

            loadedCount++;
            onProgress?.Invoke((float)loadedCount / totalCount);
        }

        isPreloading = false;
        return true;
    }

    /// <summary>
    /// 获取预加载的资源
    /// </summary>
    public T GetPreloadedAsset<T>(string address) where T : Object
    {
        if (preloadedAssets.TryGetValue(address, out var asset))
        {
            return asset as T;
        }
        return null;
    }

    /// <summary>
    /// 后台预加载（不阻塞游戏）
    /// </summary>
    public void PreloadInBackground(string groupName)
    {
        StartCoroutine(PreloadCoroutine(groupName));
    }

    private System.Collections.IEnumerator PreloadCoroutine(string groupName)
    {
        var group = System.Array.Find(preloadGroups, g => g.groupName == groupName);
        if (group == null) yield break;

        foreach (var address in group.assetAddresses)
        {
            if (preloadedAssets.ContainsKey(address))
                continue;

            var handle = Addressables.LoadAssetAsync<Object>(address);
            yield return handle;

            if (handle.Status == AsyncOperationStatus.Succeeded)
            {
                preloadedAssets[address] = handle.Result;
            }

            // 每加载一个资源，等待一帧
            yield return null;
        }
    }
}
```

---

## 3. 内存管理优化

### 3.1 引用计数管理

```csharp
/// <summary>
/// 引用计数资源管理器
/// </summary>
public class ReferenceCountedLoader : MonoBehaviour
{
    private class AssetInfo
    {
        public AsyncOperationHandle Handle;
        public int ReferenceCount;
        public float LastAccessTime;
    }

    private Dictionary<string, AssetInfo> loadedAssets = new Dictionary<string, AssetInfo>();

    [Header("Memory Management")]
    [SerializeField] private float unloadUnusedAfterSeconds = 60f;
    [SerializeField] private int maxCachedAssets = 100;

    /// <summary>
    /// 获取资源（增加引用计数）
    /// </summary>
    public async Task<T> GetAsset<T>(string address) where T : Object
    {
        if (loadedAssets.TryGetValue(address, out var info))
        {
            info.ReferenceCount++;
            info.LastAccessTime = Time.time;
            return info.Handle.Result as T;
        }

        var handle = Addressables.LoadAssetAsync<T>(address);
        await handle.Task;

        if (handle.Status == AsyncOperationStatus.Failed)
        {
            Debug.LogError($"Failed to load: {address}");
            return null;
        }

        loadedAssets[address] = new AssetInfo
        {
            Handle = handle,
            ReferenceCount = 1,
            LastAccessTime = Time.time
        };

        return handle.Result;
    }

    /// <summary>
    /// 释放资源引用
    /// </summary>
    public void ReleaseAsset(string address)
    {
        if (!loadedAssets.TryGetValue(address, out var info))
            return;

        info.ReferenceCount--;

        if (info.ReferenceCount <= 0)
        {
            // 不立即卸载，等待自动清理
            info.LastAccessTime = Time.time;
        }
    }

    /// <summary>
    /// 定期清理未使用的资源
    /// </summary>
    private void Update()
    {
        if (Time.frameCount % 300 != 0) // 每5秒检查一次
            return;

        var keysToRemove = new List<string>();
        float currentTime = Time.time;

        foreach (var kvp in loadedAssets)
        {
            if (kvp.Value.ReferenceCount <= 0 &&
                currentTime - kvp.Value.LastAccessTime > unloadUnusedAfterSeconds)
            {
                keysToRemove.Add(kvp.Key);
            }
        }

        foreach (var key in keysToRemove)
        {
            Addressables.Release(loadedAssets[key].Handle);
            loadedAssets.Remove(key);
            Debug.Log($"Unloaded unused asset: {key}");
        }
    }

    /// <summary>
    /// 强制清理超出限制的缓存
    /// </summary>
    public void EnforceCacheLimit()
    {
        if (loadedAssets.Count <= maxCachedAssets)
            return;

        // 按 LastAccessTime 排序，移除最旧的
        var sortedAssets = loadedAssets
            .OrderBy(x => x.Value.LastAccessTime)
            .ToList();

        int toRemove = loadedAssets.Count - maxCachedAssets;
        for (int i = 0; i < toRemove; i++)
        {
            var kvp = sortedAssets[i];
            if (kvp.Value.ReferenceCount <= 0)
            {
                Addressables.Release(kvp.Value.Handle);
                loadedAssets.Remove(kvp.Key);
            }
        }
    }

    /// <summary>
    /// 获取内存使用统计
    /// </summary>
    public string GetMemoryStats()
    {
        int totalAssets = loadedAssets.Count;
        int activeAssets = loadedAssets.Count(x => x.Value.ReferenceCount > 0);

        return $"Total: {totalAssets}, Active: {activeAssets}";
    }
}
```

### 3.2 场景切换清理

```csharp
/// <summary>
/// 场景切换时的资源清理
/// </summary>
public class SceneResourceCleaner : MonoBehaviour
{
    [Header("Cleanup Settings")]
    [SerializeField] private string[] persistentLabels; // 跨场景保留的资源标签
    [SerializeField] private bool cleanupOnSceneChange = true;

    private HashSet<string> persistentAssets = new HashSet<string>();

    private void Awake()
    {
        UnityEngine.SceneManagement.SceneManager.sceneUnloaded += OnSceneUnloaded;
    }

    private void OnDestroy()
    {
        UnityEngine.SceneManagement.SceneManager.sceneUnloaded -= OnSceneUnloaded;
    }

    private async void OnSceneUnloaded(UnityEngine.SceneManagement.Scene scene)
    {
        if (!cleanupOnSceneChange)
            return;

        await CleanupSceneAssets(scene);
    }

    private async Task CleanupSceneAssets(UnityEngine.SceneManagement.Scene unloadedScene)
    {
        // 1. 标记场景相关资源
        var sceneAssets = await Addressables
            .LoadResourceLocationsAsync(unloadedScene.name)
            .Task;

        // 2. 清理非持久化资源
        foreach (var location in sceneAssets)
        {
            if (persistentAssets.Contains(location.PrimaryKey))
                continue;

            // 检查是否为持久化标签
            bool isPersistent = false;
            foreach (var label in persistentLabels)
            {
                if (location.Labels.Contains(label))
                {
                    isPersistent = true;
                    break;
                }
            }

            if (!isPersistent)
            {
                Debug.Log($"Should cleanup: {location.PrimaryKey}");
            }
        }

        // 3. 强制 GC
        Resources.UnloadUnusedAssets();
        System.GC.Collect();
    }

    /// <summary>
    /// 标记资源为持久化
    /// </summary>
    public void MarkAsPersistent(string address)
    {
        persistentAssets.Add(address);
    }
}
```

### 3.3 内存监控

```csharp
/// <summary>
/// Addressables 内存监控
/// </summary>
public class AddressablesMemoryMonitor : MonoBehaviour
{
    [Header("Monitor Settings")]
    [SerializeField] private bool showDebugUI = true;
    [SerializeField] private float warningThresholdMB = 200f;

    private Rect windowRect = new Rect(10, 10, 300, 200);

    private void OnGUI()
    {
        if (!showDebugUI)
            return;

        windowRect = GUILayout.Window(12345, windowRect, DrawWindow, "Addressables Monitor");
    }

    private void DrawWindow(int windowId)
    {
        // 显示内存使用（GetTotalAllocatedMemoryLong 包含托管堆 + Native 分配）
        long totalMemory = UnityEngine.Profiling.Profiler.GetTotalAllocatedMemoryLong() / 1024 / 1024;
        long monoMemory = UnityEngine.Profiling.Profiler.GetMonoUsedSizeLong() / 1024 / 1024;

        GUILayout.Label($"Total Allocated: {totalMemory}MB | Mono Heap: {monoMemory}MB");

        if (totalMemory > warningThresholdMB)
        {
            GUI.color = Color.red;
            GUILayout.Label($"WARNING: Memory exceeds threshold {warningThresholdMB}MB!");
            GUI.color = Color.white;
        }

        // 清理按钮
        if (GUILayout.Button("Cleanup Unused"))
        {
            Resources.UnloadUnusedAssets();
            System.GC.Collect();
        }

        GUI.DragWindow();
    }
}
```

---

## 4. AssetBundle 优化策略

### 4.1 打包策略

```
┌─────────────────────────────────────────────────────────────┐
│                 AssetBundle 打包策略                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 按功能模块分组                                          │
│     ├── UI_Atlas_Main (UI主图集)                           │
│     ├── UI_Atlas_Popup (弹窗图集)                          │
│     ├── Characters_Player (玩家角色)                       │
│     ├── Characters_Enemies (敌人)                          │
│     ├── Audio_BGM (背景音乐)                               │
│     └── Audio_SFX (音效)                                   │
│                                                             │
│  2. 按加载时机分组                                          │
│     ├── Core (启动加载)                                    │
│     ├── Gameplay (游戏中加载)                              │
│     └── Optional (可选内容)                                │
│                                                             │
│  3. 按更新频率分组                                          │
│     ├── Static (不更新)                                    │
│     ├── Frequent (经常更新)                                │
│     └── Hotfix (热更内容)                                  │
│                                                             │
│  Bundle 大小建议：                                          │
│  ├── UI 图集：2-10MB                                       │
│  ├── 3D 模型：1-5MB                                        │
│  ├── 音频：按类型分组，每包 < 10MB                         │
│  └── 避免超大 Bundle (> 50MB)                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Group 配置建议

**操作路径**：`Window > Asset Management > Addressables > Groups` → 选中 Group → Inspector

**Compression 选项**：选中 Group → Inspector → Advanced → Bundle Compression

**Bundle Mode 选项**：选中 Group → Inspector → Advanced → Bundle Mode

**Build Path / Load Path**：选中 Group → Inspector → Schema > Content Packing & Loading

```
┌─────────────────────────────────────────────────────────────┐
│           Addressables Group 配置建议                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  UI 资源组：                                                │
│  ├── Compression: LZ4 (快速解压)                           │
│  ├── Bundle Mode: Pack Together (合并打包)                 │
│  └── 适合需要快速加载的UI元素                              │
│                                                             │
│  3D 资源组：                                                │
│  ├── Compression: LZ4 (Addressables 默认，无需选 LZMA)     │
│  ├── Bundle Mode: Pack Separately (独立打包)               │
│  └── 每个资源独立地址                                      │
│                                                             │
│  音频资源组：                                                │
│  ├── Compression: LZ4                                      │
│  └── 按类型分组 (BGM/SFX/Voice)                            │
│                                                             │
│  热更资源组：                                                │
│  ├── Build Path: Remote                                    │
│  ├── Load Path: Remote                                     │
│  └── 启用 CRC 校验                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 下载优化

```csharp
/// <summary>
/// 远程资源下载管理器
/// </summary>
public class RemoteAssetDownloader : MonoBehaviour
{
    [Header("Download Settings")]
    [SerializeField] private int maxConcurrentDownloads = 4;
    [SerializeField] private float timeoutSeconds = 30f;

    /// <summary>
    /// 下载指定 Label 的所有资源
    /// </summary>
    public async Task<DownloadResult> DownloadLabelAsync(
        string label,
        System.Action<float> onProgress = null)
    {
        var result = new DownloadResult();

        // 获取下载大小
        var sizeHandle = Addressables.GetDownloadSizeAsync(label);
        await sizeHandle.Task;

        result.TotalSize = sizeHandle.Result;
        result.TotalSizeMB = result.TotalSize / 1024f / 1024f;

        if (result.TotalSize <= 0)
        {
            result.AlreadyDownloaded = true;
            return result;
        }

        // 开始下载
        var downloadHandle = Addressables.DownloadDependenciesAsync(label);

        while (!downloadHandle.IsDone)
        {
            result.DownloadedBytes = downloadHandle.GetDownloadStatus().DownloadedBytes;
            result.Progress = downloadHandle.GetDownloadStatus().Percent;

            onProgress?.Invoke(result.Progress);

            await Task.Delay(100);
        }

        if (downloadHandle.Status == AsyncOperationStatus.Succeeded)
        {
            result.Success = true;
        }
        else
        {
            result.Success = false;
            result.ErrorMessage = "Download failed";
        }

        Addressables.Release(downloadHandle);
        return result;
    }

    public class DownloadResult
    {
        public bool Success;
        public bool AlreadyDownloaded;
        public long TotalSize;
        public float TotalSizeMB;
        public long DownloadedBytes;
        public float Progress;
        public string ErrorMessage;
    }

    /// <summary>
    /// 清理下载缓存
    /// 注意: 清空全部缓存用 Caching.ClearCache()，不是 ClearDependencyCacheAsync
    /// </summary>
    public async Task ClearCacheAsync()
    {
        // 清空所有 AssetBundle 缓存
        if (Caching.ClearCache())
        {
            Debug.Log("Cache cleared successfully");
        }

        // 如需按 label 清理特定资源的缓存:
        // await Addressables.ClearDependencyCacheAsync(new List<string> { "boss_assets" }, true);
    }
}
```

---

## 5. 性能监控与调试

### 5.1 加载性能分析

```csharp
/// <summary>
/// Addressables 性能分析器
/// </summary>
public class AddressablesProfiler : MonoBehaviour
{
    private class LoadRecord
    {
        public string Address;
        public float StartTime;
        public float Duration;
        public bool Success;
    }

    private List<LoadRecord> loadRecords = new List<LoadRecord>();
    private int maxRecords = 100;

    /// <summary>
    /// 带性能追踪的加载
    /// </summary>
    public async Task<T> LoadWithProfiling<T>(string address) where T : Object
    {
        var record = new LoadRecord
        {
            Address = address,
            StartTime = Time.realtimeSinceStartup
        };

        var handle = Addressables.LoadAssetAsync<T>(address);
        await handle.Task;

        record.Duration = Time.realtimeSinceStartup - record.StartTime;
        record.Success = handle.Status == AsyncOperationStatus.Succeeded;

        loadRecords.Add(record);
        if (loadRecords.Count > maxRecords)
        {
            loadRecords.RemoveAt(0);
        }

        if (record.Duration > 0.5f) // 超过500ms警告
        {
            Debug.LogWarning($"Slow load: {address} took {record.Duration * 1000:F0}ms");
        }

        return record.Success ? handle.Result : null;
    }

    /// <summary>
    /// 获取性能报告
    /// </summary>
    public string GetPerformanceReport()
    {
        if (loadRecords.Count == 0)
            return "No load records";

        var sb = new System.Text.StringBuilder();
        sb.AppendLine("=== Addressables Performance Report ===");

        float avgDuration = loadRecords.Average(r => r.Duration);
        float maxDuration = loadRecords.Max(r => r.Duration);
        int slowLoads = loadRecords.Count(r => r.Duration > 0.5f);

        sb.AppendLine($"Total loads: {loadRecords.Count}");
        sb.AppendLine($"Average duration: {avgDuration * 1000:F1}ms");
        sb.AppendLine($"Max duration: {maxDuration * 1000:F1}ms");
        sb.AppendLine($"Slow loads (>500ms): {slowLoads}");

        return sb.ToString();
    }
}
```

### 5.2 内存泄漏检测

```csharp
#if UNITY_EDITOR
/// <summary>
/// Addressables 内存泄漏检测工具（Editor Only）
/// </summary>
public static class AddressablesLeakDetector
{
    private static Dictionary<string, int> loadCounts = new Dictionary<string, int>();

    [UnityEditor.MenuItem("Tools/Addressables/Check Leaks")]
    public static void CheckLeaks()
    {
        var unreleased = loadCounts
            .Where(kvp => kvp.Value > 0)
            .Select(kvp => $"{kvp.Key} (refs: {kvp.Value})")
            .ToList();

        if (unreleased.Count > 0)
        {
            Debug.LogWarning($"Potential leaks detected:\n{string.Join("\n", unreleased)}");
        }
        else
        {
            Debug.Log("No leaks detected");
        }
    }

    /// <summary>
    /// 记录加载
    /// </summary>
    public static void RecordLoad(string address)
    {
        if (!loadCounts.ContainsKey(address))
            loadCounts[address] = 0;

        loadCounts[address]++;
    }

    /// <summary>
    /// 记录释放
    /// </summary>
    public static void RecordRelease(string address)
    {
        if (loadCounts.ContainsKey(address))
        {
            loadCounts[address]--;
            if (loadCounts[address] < 0)
            {
                Debug.LogWarning($"Possible over-release: {address}");
            }
        }
    }

    /// <summary>
    /// 获取未释放的资源
    /// </summary>
    public static List<string> GetUnreleasedAssets()
    {
        return loadCounts
            .Where(kvp => kvp.Value > 0)
            .Select(kvp => $"{kvp.Key} (refs: {kvp.Value})")
            .ToList();
    }
}
#endif
```

---

## 6. 性能优化检查清单

```markdown
## Addressables 性能优化检查清单

### 加载优化
- [ ] 使用异步加载而非同步
- [ ] 实现加载进度反馈
- [ ] 预加载关键资源
- [ ] 批量加载优化

### 内存管理
- [ ] 正确释放资源（Release handle）
- [ ] 实现引用计数
- [ ] 场景切换时清理资源
- [ ] 定期清理未使用资源
- [ ] 监控内存使用

### AssetBundle 优化
- [ ] 合理分组打包
- [ ] 选择适当的压缩格式
- [ ] 控制单个 Bundle 大小
- [ ] 按更新频率分组

### 下载优化
- [ ] 显示下载进度
- [ ] 实现断点续传（如需要）
- [ ] 后台下载
- [ ] 错误重试机制

### 调试与监控
- [ ] 加载性能分析
- [ ] 内存泄漏检测
- [ ] 性能报告生成
```

---

## 相关链接

- [[【教程】资源管线-Addressables]] - Addressables 基础教程
- [[【最佳实践】资源卸载指南]] - 资源卸载最佳实践
- [[【实战案例】加载时间优化实战]] - 加载优化实战案例
- [Unity Addressables 官方文档](https://docs.unity3d.com/Packages/com.unity.addressables@latest)

---

*创建日期: 2026-03-07*
*相关标签: #性能优化 #加载优化 #Addressables #最佳实践*
