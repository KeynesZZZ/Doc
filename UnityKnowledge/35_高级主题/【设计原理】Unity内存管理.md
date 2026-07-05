---
title: 【设计原理】Unity内存管理
tags: ["Unity", "高级主题", "内存管理", "设计原理"]
category: 高级主题
created: "2026-03-05 08:41"
updated: "2026-05-29 00:00"
description: Unity内存管理机制深度分析
unity_version: 2021.3+
status: 待验证
validation: 未经测试
related: ["[[【实战案例】内存泄漏排查实战]]", "[[【教程】打包与热更新]]", "[[../30_性能优化/32_内存管理/【设计原理】GC工作原理深度解析]]"]
author: llm
---

# Unity内存管理

> 专题课程 | 性能优化核心

## 文档定位

深入剖析Unity的Native/Managed/Graphics三层内存架构，详解GC回收机制与增量式GC、纹理内存优化策略、资源生命周期管理以及内存泄漏检测方法。适合需要理解和优化Unity内存使用的开发者阅读。

---

## 1. Unity内存模型

### 1.1 内存架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Unity 内存架构                             │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Native Memory                       │   │
│  │              (原生内存 - C++层)                        │   │
│  │  ├── 纹理 (Textures)                                 │   │
│  │  ├── 网格 (Meshes)                                   │   │
│  │  ├── 音频 (Audio)                                    │   │
│  │  ├── 动画 (Animation)                                │   │
│  │  ├── 渲染资源 (RenderTextures, Shaders)              │   │
│  │  └── 场景数据                                         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Managed Memory                      │   │
│  │               (托管内存 - C#层)                        │   │
│  │  ├── Heap (堆)                                       │   │
│  │  │   ├── 引用类型对象                                 │   │
│  │  │   ├── 类实例、数组、字符串                          │   │
│  │  │   └── 由GC管理                                     │   │
│  │  │                                                    │   │
│  │  └── Stack (栈)                                       │   │
│  │      ├── 值类型                                       │   │
│  │      ├── 方法参数、局部变量                            │   │
│  │      └── 自动释放                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Graphics Memory                        │   │
│  │               (显存 - GPU)                            │   │
│  │  ├── Vertex Buffers                                  │   │
│  │  ├── Index Buffers                                   │   │
│  │  ├── Textures (GPU Copy)                             │   │
│  │  └── Render Targets                                  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 内存分析工具

```csharp
using UnityEngine;
using System;

/// <summary>
/// 内存分析工具
/// </summary>
public static class MemoryAnalyzer
{
    /// <summary>
    /// 输出当前内存状态
    /// </summary>
    public static void LogMemoryStatus()
    {
        Debug.Log("===== Memory Status =====");

        // Unity总内存
        long totalMemory = UnityEngine.Profiling.Profiler.GetTotalAllocatedMemoryLong();
        long reservedMemory = UnityEngine.Profiling.Profiler.GetTotalReservedMemoryLong();
        long unusedReserved = UnityEngine.Profiling.Profiler.GetTotalUnusedReservedMemoryLong();

        Debug.Log($"Total Allocated: {FormatBytes(totalMemory)}");
        Debug.Log($"Total Reserved: {FormatBytes(reservedMemory)}");
        Debug.Log($"Unused Reserved: {FormatBytes(unusedReserved)}");

        // GC内存
        long gcMemory = GC.GetTotalMemory(false);
        Debug.Log($"GC Memory: {FormatBytes(gcMemory)}");

        // 纹理内存
        long textureMemory = UnityEngine.Profiling.Profiler.GetTotalAllocatedMemoryLong(
            UnityEngine.Profiling.Memory.Recorder.Texture);
        Debug.Log($"Texture Memory: {FormatBytes(textureMemory)}");

        // 网格内存
        long meshMemory = UnityEngine.Profiling.Profiler.GetTotalAllocatedMemoryLong(
            UnityEngine.Profiling.Memory.Recorder.Mesh);
        Debug.Log($"Mesh Memory: {FormatBytes(meshMemory)}");

        Debug.Log("=========================");
    }

    /// <summary>
    /// 获取详细内存信息
    /// </summary>
    public static void LogDetailedMemory()
    {
#if UNITY_2020_1_OR_NEWER
        var snapshot = UnityEngine.Profiling.Memory.Experimental.MemoryProfiler.TakeSnapshot();
        Debug.Log("Memory snapshot taken. Use Memory Profiler package to analyze.");
#endif
    }

    private static string FormatBytes(long bytes)
    {
        string[] sizes = { "B", "KB", "MB", "GB" };
        int order = 0;
        double size = bytes;

        while (size >= 1024 && order < sizes.Length - 1)
        {
            order++;
            size /= 1024;
        }

        return $"{size:0.##} {sizes[order]}";
    }

    /// <summary>
    /// 获取所有加载的资源
    /// </summary>
    public static void LogLoadedAssets()
    {
        var allTextures = Resources.FindObjectsOfTypeAll<Texture>();
        var allMeshes = Resources.FindObjectsOfTypeAll<Mesh>();
        var allMaterials = Resources.FindObjectsOfTypeAll<Material>();
        var allAudioClips = Resources.FindObjectsOfTypeAll<AudioClip>();

        Debug.Log($"Loaded Textures: {allTextures.Length}");
        Debug.Log($"Loaded Meshes: {allMeshes.Length}");
        Debug.Log($"Loaded Materials: {allMaterials.Length}");
        Debug.Log($"Loaded AudioClips: {allAudioClips.Length}");
    }
}
```

---

## 2. 托管堆与GC

### 2.1 C#垃圾回收机制

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;

/// <summary>
/// C# GC机制详解
/// </summary>
public class GCFundamentals : MonoBehaviour
{
    /*
    ========== C# GC 基础 ==========

    1. 分代回收 (Generational GC)
       - Gen 0: 短期对象，频繁回收
       - Gen 1: 中期对象
       - Gen 2: 长期对象，完整回收

    2. 回收触发条件
       - Gen 0 满时
       - 手动调用 GC.Collect()
       - 系统内存不足时
       - AppDomain 卸载时

    3. GC对游戏的影响
       - 停顿时间 (Pause Time)
       - CPU峰值
       - 帧率波动

    ========== Unity中的GC ==========

    Unity使用Boehm GC（非分代）
    - Stop-the-world 回收
    - 不压缩内存
    - 可能造成内存碎片
    - 移动端影响更明显

    Unity 2020+ 可切换到增量式GC
    - 分帧执行回收
    - 减少单帧卡顿
    - 启用方式: Player Settings > Incremental GC
    */

    [Header("GC Settings")]
    [SerializeField] private bool enableIncrementalGC = true;

    private void Start()
    {
        // 启用增量式GC
        if (enableIncrementalGC)
        {
            UnityEngine.Scripting.GarbageCollector.incrementalTimeSliceNs = 3000000; // 3ms
        }
    }
}
```

### 2.2 避免GC分配

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Text;

/// <summary>
/// GC分配优化示例
/// </summary>
public class GCAllocationOptimization : MonoBehaviour
{
    // ========== 1. 字符串优化 ==========

    // 错误：每次调用都会产生GC
    public string GetPlayerInfoBad(int score, int level)
    {
        return "Score: " + score + ", Level: " + level; // 产生临时字符串
    }

    // 正确：使用StringBuilder或string.Format
    private StringBuilder sb = new StringBuilder(128);

    public string GetPlayerInfoGood(int score, int level)
    {
        sb.Clear();
        sb.Append("Score: ");
        sb.Append(score);
        sb.Append(", Level: ");
        sb.Append(level);
        return sb.ToString();
    }

    // 更好：完全避免字符串拼接
    public void DisplayPlayerInfo(int score, int level)
    {
        // 直接更新UI组件
        scoreText.text = score.ToString();
        levelText.text = level.ToString();
    }

    [SerializeField] private Text scoreText;
    [SerializeField] private Text levelText;

    // ========== 2. 装箱拆箱优化 ==========

    // 错误：值类型装箱
    public void LogValueBad(int value)
    {
        Debug.Log("Value: " + value); // int被装箱成object
    }

    // 正确：避免装箱
    public void LogValueGood(int value)
    {
        Debug.Log($"Value: {value}"); // 使用字符串插值
    }

    // ========== 3. 数组与List优化 ==========

    // 错误：每次都创建新数组
    public int[] GetScoresBad()
    {
        return new int[] { 1, 2, 3, 4, 5 }; // 每次调用都分配
    }

    // 正确：缓存数组
    private static readonly int[] cachedScores = { 1, 2, 3, 4, 5 };

    public int[] GetScoresGood()
    {
        return cachedScores;
    }

    // 错误：在Update中分配
    private void UpdateBad()
    {
        var enemies = GetEnemies(); // 每帧分配新List
        foreach (var enemy in enemies)
        {
            // ...
        }
    }

    // 正确：复用List
    private List<Enemy> enemyCache = new List<Enemy>(32);

    private void UpdateGood()
    {
        enemyCache.Clear();
        GetEnemiesNonAlloc(enemyCache);
        foreach (var enemy in enemyCache)
        {
            // ...
        }
    }

    private void GetEnemiesNonAlloc(List<Enemy> result)
    {
        // 填充现有List而不创建新的
    }

    // ========== 4. 委托与事件优化 ==========

    // 错误：每次都创建新委托
    public void AddListenerBad()
    {
        button.onClick.AddListener(OnButtonClick); // 如果重复调用会累积
    }

    // 正确：缓存委托
    private Action cachedClickAction;

    public void AddListenerGood()
    {
        if (cachedClickAction == null)
            cachedClickAction = OnButtonClick;
        button.onClick.AddListener(cachedClickAction);
    }

    // 移除时也使用缓存的委托
    public void RemoveListenerGood()
    {
        if (cachedClickAction != null)
            button.onClick.RemoveListener(cachedClickAction);
    }

    [SerializeField] private UnityEngine.UI.Button button;

    private void OnButtonClick() { }

    // ========== 5. 协程优化 ==========

    // 错误：在协程中使用 new WaitForSeconds
    private IEnumerator CoroutineBad()
    {
        while (true)
        {
            yield return new WaitForSeconds(1f); // 每次都分配
        }
    }

    // 正确：缓存WaitForSeconds
    private WaitForSeconds waitOneSecond = new WaitForSeconds(1f);

    private IEnumerator CoroutineGood()
    {
        while (true)
        {
            yield return waitOneSecond;
        }
    }

    // 常用WaitForXXX缓存
    private static readonly WaitForEndOfFrame waitForEndOfFrame = new WaitForEndOfFrame();
    private static readonly WaitForFixedUpdate waitForFixedUpdate = new WaitForFixedUpdate();

    // ========== 6. foreach优化 ==========

    // Unity 5.5+ foreach已优化，但自定义集合需要注意
    // 对于List<T>，foreach是安全的

    // 对于自定义集合，使用for循环更安全
    public void IterateList(List<int> list)
    {
        // 两种方式在Unity 5.5+都可以
        // for循环更可控
        for (int i = 0; i < list.Count; i++)
        {
            int item = list[i];
            // ...
        }
    }

    // ========== 7. LINQ优化 ==========

    // 错误：在热代码路径中使用LINQ
    public int GetTotalScoreBad(List<int> scores)
    {
        return scores.Where(s => s > 0).Sum(); // 产生GC
    }

    // 正确：使用循环
    public int GetTotalScoreGood(List<int> scores)
    {
        int total = 0;
        for (int i = 0; i < scores.Count; i++)
        {
            if (scores[i] > 0)
                total += scores[i];
        }
        return total;
    }
}
```

### 2.3 对象池模式

```csharp
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// 通用对象池 - 避免频繁分配
/// </summary>
public class ObjectPool<T> where T : class, new()
{
    private Stack<T> pool;
    private int maxSize;

    public ObjectPool(int initialSize = 10, int maxSize = 100)
    {
        this.maxSize = maxSize;
        pool = new Stack<T>(initialSize);

        for (int i = 0; i < initialSize; i++)
        {
            pool.Push(new T());
        }
    }

    public T Get()
    {
        if (pool.Count > 0)
            return pool.Pop();
        return new T();
    }

    public void Return(T item)
    {
        if (pool.Count < maxSize)
            pool.Push(item);
    }

    public void Clear()
    {
        pool.Clear();
    }
}

/// <summary>
/// GameObject对象池
/// </summary>
public class GameObjectPool
{
    private GameObject prefab;
    private Transform parent;
    private Queue<GameObject> pool = new Queue<GameObject>();
    private HashSet<GameObject> active = new HashSet<GameObject>();
    private int maxSize;

    public GameObjectPool(GameObject prefab, int initialSize, int maxSize, Transform parent = null)
    {
        this.prefab = prefab;
        this.maxSize = maxSize;
        this.parent = parent;

        for (int i = 0; i < initialSize; i++)
        {
            var go = CreateNew();
            go.SetActive(false);
            pool.Enqueue(go);
        }
    }

    public GameObject Get(Vector3 position, Quaternion rotation)
    {
        GameObject go;

        if (pool.Count > 0)
        {
            go = pool.Dequeue();
            go.transform.position = position;
            go.transform.rotation = rotation;
        }
        else
        {
            go = Object.Instantiate(prefab, position, rotation, parent);
        }

        go.SetActive(true);
        active.Add(go);
        return go;
    }

    public void Return(GameObject go)
    {
        if (!active.Contains(go)) return;

        active.Remove(go);
        go.SetActive(false);
        go.transform.SetParent(parent);
        pool.Enqueue(go);
    }

    public void ReturnAll()
    {
        foreach (var go in active)
        {
            go.SetActive(false);
            go.transform.SetParent(parent);
            pool.Enqueue(go);
        }
        active.Clear();
    }

    private GameObject CreateNew()
    {
        var go = Object.Instantiate(prefab, parent);
        return go;
    }

    public void Clear()
    {
        foreach (var go in pool)
            Object.Destroy(go);
        foreach (var go in active)
            Object.Destroy(go);

        pool.Clear();
        active.Clear();
    }
}

/// <summary>
/// 对象池管理器
/// </summary>
public class PoolManager : MonoSingleton<PoolManager>
{
    private Dictionary<string, GameObjectPool> gameObjectPools = new Dictionary<string, GameObjectPool>();
    private Dictionary<Type, object> objectPools = new Dictionary<Type, object>();

    /// <summary>
    /// 初始化GameObject池
    /// </summary>
    public void InitializeGameObjectPool(string key, GameObject prefab, int initialSize = 10, int maxSize = 50)
    {
        if (!gameObjectPools.ContainsKey(key))
        {
            gameObjectPools[key] = new GameObjectPool(prefab, initialSize, maxSize, transform);
        }
    }

    /// <summary>
    /// 获取GameObject
    /// </summary>
    public GameObject GetGameObject(string key, Vector3 position, Quaternion rotation)
    {
        if (gameObjectPools.TryGetValue(key, out var pool))
            return pool.Get(position, rotation);
        return null;
    }

    /// <summary>
    /// 归还GameObject
    /// </summary>
    public void ReturnGameObject(string key, GameObject go)
    {
        if (gameObjectPools.TryGetValue(key, out var pool))
            pool.Return(go);
        else
            Destroy(go);
    }

    /// <summary>
    /// 获取C#对象池
    /// </summary>
    public ObjectPool<T> GetObjectPool<T>(int initialSize = 10, int maxSize = 100) where T : class, new()
    {
        Type type = typeof(T);
        if (!objectPools.TryGetValue(type, out var pool))
        {
            pool = new ObjectPool<T>(initialSize, maxSize);
            objectPools[type] = pool;
        }
        return (ObjectPool<T>)pool;
    }

    /// <summary>
    /// 清理所有池
    /// </summary>
    public void ClearAll()
    {
        foreach (var pool in gameObjectPools.Values)
            pool.Clear();
        gameObjectPools.Clear();
        objectPools.Clear();
    }
}
```

---

## 3. 纹理内存优化

### 3.1 纹理内存计算

```csharp
using UnityEngine;

/// <summary>
/// 纹理内存分析
/// </summary>
public static class TextureMemoryAnalyzer
{
    /*
    ========== 纹理内存计算公式 ==========

    内存大小 = 宽 × 高 × 像素大小 × Mipmap倍数

    像素大小（字节）：
    - RGB24:      3 bytes
    - RGBA32:     4 bytes
    - RGB565:     2 bytes
    - RGBA4444:   2 bytes
    - ASTC 4x4:   1 byte (压缩)
    - ASTC 8x8:   0.5 byte (压缩)

    Mipmap倍数 ≈ 1.33x（完整Mipmap链）

    示例：
    1024x1024 RGBA32 无Mipmap = 4 MB
    1024x1024 RGBA32 有Mipmap = 5.33 MB
    1024x1024 ASTC 4x4 有Mipmap ≈ 1.33 MB
    */

    /// <summary>
    /// 计算纹理内存大小
    /// </summary>
    public static long CalculateTextureMemory(Texture texture)
    {
        if (texture == null) return 0;

        long memory = 0;

        // 获取基本尺寸
        int width = texture.width;
        int height = texture.height;

        // 计算像素大小
        int bytesPerPixel = GetBytesPerPixel(texture);

        // 基础内存
        memory = (long)width * height * bytesPerPixel;

        // Mipmap
        if (texture.mipmapCount > 1)
        {
            memory = (long)(memory * 1.33);
        }

        return memory;
    }

    private static int GetBytesPerPixel(Texture texture)
    {
        if (texture is Texture2D tex2D)
        {
            var format = tex2D.format;

            return format switch
            {
                TextureFormat.RGB24 => 3,
                TextureFormat.RGBA32 => 4,
                TextureFormat.ARGB32 => 4,
                TextureFormat.RGB565 => 2,
                TextureFormat.RGBA4444 => 2,
                TextureFormat.ETC_RGB4 => 1,
                TextureFormat.ETC2_RGBA8 => 1,
                TextureFormat.ASTC_4x4 => 1,
                TextureFormat.ASTC_6x6 => 1,
                TextureFormat.ASTC_8x8 => 1,
                _ => 4
            };
        }

        return 4;
    }

    /// <summary>
    /// 分析场景中所有纹理
    /// </summary>
    public static void AnalyzeAllTextures()
    {
        var textures = Resources.FindObjectsOfTypeAll<Texture2D>();
        long totalMemory = 0;

        var sortedTextures = new System.Collections.Generic.List<TextureInfo>();

        foreach (var tex in textures)
        {
            long memory = CalculateTextureMemory(tex);
            totalMemory += memory;

            sortedTextures.Add(new TextureInfo
            {
                Name = tex.name,
                Size = $"{tex.width}x{tex.height}",
                Format = tex.format.ToString(),
                Memory = memory
            });
        }

        // 按内存排序
        sortedTextures.Sort((a, b) => b.Memory.CompareTo(a.Memory));

        Debug.Log($"===== Texture Analysis =====");
        Debug.Log($"Total Textures: {textures.Length}");
        Debug.Log($"Total Memory: {FormatBytes(totalMemory)}");
        Debug.Log($"Top 10 Largest:");

        for (int i = 0; i < Mathf.Min(10, sortedTextures.Count); i++)
        {
            var info = sortedTextures[i];
            Debug.Log($"  {info.Name}: {info.Size} ({info.Format}) = {FormatBytes(info.Memory)}");
        }
    }

    private static string FormatBytes(long bytes)
    {
        string[] sizes = { "B", "KB", "MB", "GB" };
        int order = 0;
        double size = bytes;

        while (size >= 1024 && order < sizes.Length - 1)
        {
            order++;
            size /= 1024;
        }

        return $"{size:0.##} {sizes[order]}";
    }

    private struct TextureInfo
    {
        public string Name;
        public string Size;
        public string Format;
        public long Memory;
    }
}
```

### 3.2 纹理优化策略

```csharp
using UnityEngine;
using UnityEditor;

/// <summary>
/// 纹理优化配置
/// </summary>
public class TextureOptimizationSettings : MonoBehaviour
{
    /*
    ========== 纹理格式选择 ==========

    移动端：
    - ASTC 4x4:  高质量，推荐
    - ASTC 6x6:  中质量
    - ASTC 8x8:  低质量，省内存
    - ETC2:      Android备用（不支持ASTC时）

    PC：
    - BC7:       高质量压缩
    - DXT5:      通用压缩
    - RGBA32:    无压缩（UI等需要精确颜色的）

    ========== 优化策略 ==========

    1. 使用压缩格式
    2. 按需启用Mipmap
    3. 合理的Max Size
    4. 使用Sprite Atlas
    5. 异步加载大纹理
    6. 及时卸载不用的纹理
    */

    [Header("Mobile Settings")]
    [SerializeField] private TextureFormat mobileFormat = TextureFormat.ASTC_4x4;

    [Header("Quality Settings")]
    [SerializeField] private bool useMipmapFor3D = true;
    [SerializeField] private bool useMipmapForUI = false;
    [SerializeField] private int maxTextureSize = 2048;

    /// <summary>
    /// 动态调整纹理质量
    /// </summary>
    public static void AdjustTextureQuality(Texture2D texture, int qualityLevel)
    {
        // 根据画质设置调整纹理
        int maxSize = qualityLevel switch
        {
            0 => 512,   // Low
            1 => 1024,  // Medium
            2 => 2048,  // High
            _ => 2048
        };

        // 注意：运行时改变纹理大小需要重新创建
    }
}

/// <summary>
/// 纹理加载优化
/// </summary>
public class TextureLoader : MonoBehaviour
{
    /// <summary>
    /// 异步加载纹理
    /// </summary>
    public static void LoadTextureAsync(string path, System.Action<Texture2D> onComplete)
    {
        var request = Resources.LoadAsync<Texture2D>(path);
        request.completed += (op) =>
        {
            var texture = request.asset as Texture2D;
            onComplete?.Invoke(texture);
        };
    }

    /// <summary>
    /// 卸载纹理资源
    /// </summary>
    public static void UnloadTexture(Texture2D texture)
    {
        if (texture != null)
        {
            Resources.UnloadAsset(texture);
        }
    }

    /// <summary>
    /// 压缩运行时纹理
    /// </summary>
    public static void CompressTexture(Texture2D texture, bool highQuality = true)
    {
        if (texture == null) return;

        // 重新压缩纹理
        texture.Compress(highQuality);
    }
}
```

---

## 4. 资源生命周期管理

### 4.1 资源加载策略

```csharp
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// 资源加载策略
/// </summary>
public class AssetLoadingStrategy : MonoBehaviour
{
    /*
    ========== 资源加载方式对比 ==========

    1. Resources.Load
       - 简单易用
       - 同步加载
       - 增加包体大小
       - 不适合热更新

    2. AssetBundle
       - 支持热更新
       - 异步加载
       - 需要管理依赖
       - 内存管理复杂

    3. Addressables
       - 统一接口
       - 支持热更新
       - 自动管理依赖
       - 推荐使用
    */

    // ========== Resources.Load 示例 ==========

    public T LoadResource<T>(string path) where T : Object
    {
        return Resources.Load<T>(path);
    }

    public void LoadResourceAsync<T>(string path, System.Action<T> onComplete) where T : Object
    {
        StartCoroutine(LoadResourceCoroutine(path, onComplete));
    }

    private System.Collections.IEnumerator LoadResourceCoroutine<T>(string path, System.Action<T> onComplete) where T : Object
    {
        var request = Resources.LoadAsync<T>(path);
        yield return request;

        if (request.asset != null)
            onComplete?.Invoke(request.asset as T);
        else
            Debug.LogError($"Failed to load: {path}");
    }

    // ========== 缓存策略 ==========

    private Dictionary<string, Object> cache = new Dictionary<string, Object>();

    public T LoadWithCache<T>(string path) where T : Object
    {
        if (cache.TryGetValue(path, out var cached))
            return cached as T;

        var asset = Resources.Load<T>(path);
        if (asset != null)
            cache[path] = asset;

        return asset;
    }

    public void ClearCache()
    {
        foreach (var asset in cache.Values)
        {
            if (asset != null)
                Resources.UnloadAsset(asset);
        }
        cache.Clear();
    }
}
```

### 4.2 场景内存管理

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using System.Collections;

/// <summary>
/// 场景内存管理器
/// </summary>
public class SceneMemoryManager : MonoBehaviour
{
    public static SceneMemoryManager Instance { get; private set; }

    private void Awake()
    {
        Instance = this;
        DontDestroyOnLoad(gameObject);
        SceneManager.sceneUnloaded += OnSceneUnloaded;
        SceneManager.sceneLoaded += OnSceneLoaded;
    }

    private void OnSceneLoaded(Scene scene, LoadSceneMode mode)
    {
        Debug.Log($"Scene Loaded: {scene.name}");
        MemoryAnalyzer.LogMemoryStatus();
    }

    private void OnSceneUnloaded(Scene scene)
    {
        Debug.Log($"Scene Unloaded: {scene.name}");

        // 场景卸载后清理资源
        StartCoroutine(CleanupAfterSceneUnload());
    }

    private IEnumerator CleanupAfterSceneUnload()
    {
        // 等待一帧，让Unity完成内部清理
        yield return null;

        // 卸载未使用的资源
        Resources.UnloadUnusedAssets();

        // 触发GC
        GC.Collect();
        GC.WaitForPendingFinalizers();

        Debug.Log("Cleanup completed");
        MemoryAnalyzer.LogMemoryStatus();
    }

    /// <summary>
    /// 切换场景（带加载界面和内存清理）
    /// </summary>
    public void LoadSceneWithCleanup(string sceneName)
    {
        StartCoroutine(LoadSceneCoroutine(sceneName));
    }

    private IEnumerator LoadSceneCoroutine(string sceneName)
    {
        // 显示加载界面
        // LoadingScreen.Show();

        // 异步加载
        var op = SceneManager.LoadSceneAsync(sceneName);
        op.allowSceneActivation = false;

        while (op.progress < 0.9f)
        {
            // 更新进度条
            // LoadingScreen.SetProgress(op.progress);
            yield return null;
        }

        // 允许场景激活
        op.allowSceneActivation = true;

        // 等待场景加载完成
        yield return op;

        // 隐藏加载界面
        // LoadingScreen.Hide();
    }

    private void OnDestroy()
    {
        SceneManager.sceneUnloaded -= OnSceneUnloaded;
        SceneManager.sceneLoaded -= OnSceneLoaded;
    }
}
```

---

## 5. 内存泄漏检测

```csharp
using UnityEngine;
using System.Collections.Generic;
using System;

/// <summary>
/// 内存泄漏检测器
/// </summary>
public class MemoryLeakDetector : MonoBehaviour
{
    private static MemoryLeakDetector instance;

    [Header("Settings")]
    [SerializeField] private float checkInterval = 10f;
    [SerializeField] private long warningThresholdMB = 100;
    [SerializeField] private bool enableLogging = true;

    private long lastMemory;
    private int checkCount;

    // 追踪对象创建
    private Dictionary<Type, int> objectCounts = new Dictionary<Type, int>();
    private Dictionary<Type, int> lastObjectCounts = new Dictionary<Type, int>();

    public static MemoryLeakDetector Instance
    {
        get
        {
            if (instance == null)
            {
                var go = new GameObject("MemoryLeakDetector");
                instance = go.AddComponent<MemoryLeakDetector>();
                DontDestroyOnLoad(go);
            }
            return instance;
        }
    }

    private void Start()
    {
        StartCoroutine(CheckMemoryRoutine());
    }

    private IEnumerator CheckMemoryRoutine()
    {
        while (true)
        {
            yield return new WaitForSeconds(checkInterval);
            CheckMemory();
        }
    }

    private void CheckMemory()
    {
        checkCount++;

        long currentMemory = GC.GetTotalMemory(false);
        long deltaMemory = currentMemory - lastMemory;

        if (enableLogging)
        {
            Debug.Log($"[Memory Check #{checkCount}] " +
                     $"Current: {FormatBytes(currentMemory)}, " +
                     $"Delta: {FormatBytes(deltaMemory)}");
        }

        // 检查内存增长
        if (deltaMemory > warningThresholdMB * 1024 * 1024)
        {
            Debug.LogWarning($"Memory increased by {FormatBytes(deltaMemory)}! " +
                           $"Possible memory leak.");
            AnalyzePotentialLeaks();
        }

        lastMemory = currentMemory;
    }

    private void AnalyzePotentialLeaks()
    {
        // 检查常见泄漏源
        var allObjects = FindObjectsOfType<UnityEngine.Object>();

        // 分类统计
        var typeCounts = new Dictionary<Type, int>();
        foreach (var obj in allObjects)
        {
            var type = obj.GetType();
            if (!typeCounts.ContainsKey(type))
                typeCounts[type] = 0;
            typeCounts[type]++;
        }

        // 对比上次检查
        foreach (var kvp in typeCounts)
        {
            int lastCount = lastObjectCounts.GetValueOrDefault(kvp.Key, 0);
            int delta = kvp.Value - lastCount;

            if (delta > 10) // 显著增加
            {
                Debug.LogWarning($"Object count increased: {kvp.Key.Name} " +
                               $"({lastCount} -> {kvp.Value}, +{delta})");
            }
        }

        lastObjectCounts = new Dictionary<Type, int>(typeCounts);
    }

    /// <summary>
    /// 手动触发内存检查
    /// </summary>
    public static void ForceCheck()
    {
        Instance?.CheckMemory();
    }

    /// <summary>
    /// 注册对象创建（手动追踪）
    /// </summary>
    public static void RegisterObjectCreation<T>()
    {
        var type = typeof(T);
        if (!Instance.objectCounts.ContainsKey(type))
            Instance.objectCounts[type] = 0;
        Instance.objectCounts[type]++;
    }

    /// <summary>
    /// 注册对象销毁
    /// </summary>
    public static void RegisterObjectDestruction<T>()
    {
        var type = typeof(T);
        if (Instance.objectCounts.ContainsKey(type))
            Instance.objectCounts[type]--;
    }

    private string FormatBytes(long bytes)
    {
        string[] sizes = { "B", "KB", "MB", "GB" };
        int order = 0;
        double size = bytes;

        while (size >= 1024 && order < sizes.Length - 1)
        {
            order++;
            size /= 1024;
        }

        return $"{size:0.##} {sizes[order]}";
    }
}
```

---

## 6. 内存优化最佳实践

```
┌─────────────────────────────────────────────────────────────┐
│                  内存优化最佳实践清单                         │
│                                                             │
│  1. GC优化                                                  │
│     ├── 避免在Update中分配内存                              │
│     ├── 缓存常用对象（WaitForSeconds、委托）                 │
│     ├── 使用StringBuilder代替字符串拼接                     │
│     ├── 避免装箱拆箱                                        │
│     ├── 使用对象池                                          │
│     └── 启用增量式GC                                        │
│                                                             │
│  2. 纹理优化                                                │
│     ├── 使用压缩格式（ASTC/ETC2）                           │
│     ├── 按需启用Mipmap                                      │
│     ├── 合理设置Max Size                                    │
│     ├── 使用Sprite Atlas                                    │
│     └── 及时卸载大纹理                                      │
│                                                             │
│  3. 资源管理                                                │
│     ├── 避免使用Resources（用Addressables）                 │
│     ├── 场景切换时清理资源                                  │
│     ├── 合理使用AssetBundle引用计数                         │
│     └── 卸载未使用资源                                      │
│                                                             │
│  4. 代码优化                                                │
│     ├── 避免闭包捕获                                        │
│     ├── 使用NonAlloc版本API                                 │
│     ├── 减少反射使用                                        │
│     └── 避免LINQ在热代码路径                                │
│                                                             │
│  5. 监控与调试                                              │
│     ├── 使用Memory Profiler                                 │
│     ├── 定期检查内存状态                                    │
│     ├── 监控内存泄漏                                        │
│     └── 记录内存峰值                                        │
└─────────────────────────────────────────────────────────────┘
```

### 内存优化检查表

```csharp
/// <summary>
/// 内存优化检查表
/// </summary>
public static class MemoryOptimizationChecklist
{
    public static void RunChecks()
    {
        Debug.Log("===== Memory Optimization Checklist =====");

        CheckRaycastTargets();
        CheckTextureSizes();
        CheckGCAllocations();
        CheckObjectPools();

        Debug.Log("==========================================");
    }

    private static void CheckRaycastTargets()
    {
        var graphics = UnityEngine.Object.FindObjectsOfType<Graphic>();
        int raycastEnabled = 0;

        foreach (var g in graphics)
        {
            if (g.raycastTarget)
                raycastEnabled++;
        }

        Debug.Log($"[UI] Raycast Targets: {raycastEnabled}/{graphics.Length}");

        if (raycastEnabled > graphics.Length * 0.5f)
        {
            Debug.LogWarning("More than 50% of Graphics have Raycast Target enabled!");
        }
    }

    private static void CheckTextureSizes()
    {
        var textures = Resources.FindObjectsOfTypeAll<Texture2D>();
        int largeTextures = 0;

        foreach (var tex in textures)
        {
            if (tex.width > 2048 || tex.height > 2048)
            {
                Debug.LogWarning($"Large texture detected: {tex.name} ({tex.width}x{tex.height})");
                largeTextures++;
            }
        }

        Debug.Log($"[Textures] Large textures (>2048): {largeTextures}");
    }

    private static void CheckGCAllocations()
    {
        // 提示检查常见问题
        Debug.Log("[GC] Remember to check:");
        Debug.Log("  - Avoid string concatenation in Update");
        Debug.Log("  - Cache WaitForSeconds in coroutines");
        Debug.Log("  - Use StringBuilder for dynamic strings");
        Debug.Log("  - Avoid boxing/unboxing");
    }

    private static void CheckObjectPools()
    {
        Debug.Log("[Pools] Consider using object pools for:");
        Debug.Log("  - Bullets/Projectiles");
        Debug.Log("  - Particles");
        Debug.Log("  - UI List Items");
        Debug.Log("  - Frequently instantiated objects");
    }
}
```

---

## 本课小结

### 核心知识点

| 知识点 | 核心要点 |
|--------|----------|
| 内存模型 | Native/Managed/Graphics三层 |
| GC机制 | Boehm GC（非分代、非压缩）、增量式GC |
| GC优化 | 避免分配、对象池、缓存 |
| 纹理优化 | 压缩格式、Mipmap、MaxSize |
| 资源管理 | 加载策略、卸载时机 |
| 泄漏检测 | 内存监控、对象追踪 |

### 内存优化优先级

```
1. 纹理内存（通常占50%+）
   ├── 使用压缩格式
   └── 合理的纹理尺寸

2. GC优化（影响帧率）
   ├── 减少运行时分配
   └── 使用对象池

3. 资源管理（避免泄漏）
   ├── 及时卸载
   └── 正确使用引用计数

4. 代码优化
   ├── 避免闭包
   └── 缓存常用对象
```

---

## 相关链接

- [[【实战案例】内存泄漏排查实战]]
- [Unity Memory Management](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity7.html)
- [Memory Profiler](https://docs.unity3d.com/Packages/com.unity.memoryprofiler@latest)
- [Garbage Collection](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/)
