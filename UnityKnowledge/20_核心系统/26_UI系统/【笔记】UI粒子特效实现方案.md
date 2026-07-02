---
title: 【笔记】UI粒子特效实现方案
tags: ["Unity", "UI", "UI系统", "粒子特效", "渲染管线"]
category: 核心系统/UI系统
created: "2026-07-01"
description: UI粒子特效三种主流方案（Screen Space Camera / RenderTexture / UIParticles组件）的原理、代码与选型
unity_version: 2021.3+
status: 待验证
validation: Demo验证
related: ["【笔记】UGUI DrawCall影响因素全面测试", "【设计原理】UGUI合批机制深度解析"]
author: llm
---

# 【笔记】UI粒子特效实现方案

> UI 粒子特效是 UGUI 开发中的高频痛点：Particle System 默认无法显示在 Screen Space Overlay Canvas 上 `#UI特效` `#粒子系统` `#渲染排序`

## 文档定位

解决"粒子如何在 UI 上正确显示并排序"这一核心问题，提供三种主流方案的原理、代码与选型决策。

**相关文档**：[[【笔记】UGUI DrawCall影响因素全面测试]]、[[【设计原理】UGUI合批机制深度解析]]、[[【最佳实践】UI性能优化速查]]

---

## 核心问题：为什么粒子不显示在 UI 上？

UGUI 的 `Screen Space Overlay` 模式在场景所有渲染完成之后直接绘制，**跳过整个场景渲染管线**，所以放在场景中的 ParticleSystem 不会出现在 Overlay Canvas 之上。

```
渲染管线顺序：

1. 场景几何体 (不透明 → 透明)
2. Screen Space Camera Canvas
3. Screen Space Overlay Canvas    ← 跳过场景里的 ParticleSystem
4. Gizmos / Editor
```

本质问题：**怎么让粒子出现在正确的渲染阶段，且能和 UI 控件正确排序**。

---

## 三种方案总览

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| **Screen Space Camera** | Canvas 切到 Camera 模式，粒子放场景里 | 原生支持，零额外依赖 | Overlay Canvas 要改架构 | 大部分项目首选 |
| **RenderTexture** | 独立相机渲染粒子到 RT，UI 用 RawImage 显示 | 不改 Canvas 模式，支持 3D 粒子 | 额外内存 + 性能开销 | 复杂 3D 模型+粒子预览 |
| **UIParticles 组件** | 截获 ParticleSystem 数据，注入 CanvasRenderer | 真正合批进 UI，性能最优 | 不支持所有粒子特性 | 纯 2D UI 特效 |

---

## 方案一：Screen Space Camera + sortingOrder（推荐）

最常用、最稳定的做法。

### 设置步骤

```
1. Canvas → Render Mode: Screen Space - Camera
2. Canvas → Render Camera: 指定 UI Camera（正交相机）
3. 场景中放 ParticleSystem，Layer 设为 UI
4. ParticleSystem → Renderer → Sorting Layer / Order in Layer 设为比 Canvas 大的值
```

### 排序控制工具

```csharp
/// <summary>
/// UI粒子排序工具：让粒子显示在指定UI面板之上
/// </summary>
public static class UIParticleSorter
{
    /// <summary>
    /// 设置粒子的渲染顺序
    /// </summary>
    /// <param name="particle">粒子系统</param>
    /// <param name="order">目标 sortingOrder</param>
    public static void SetSortingOrder(ParticleSystem particle, int order)
    {
        var renderer = particle.GetComponent<ParticleSystemRenderer>();
        if (renderer != null)
        {
            renderer.sortingOrder = order;
        }
    }

    /// <summary>
    /// 让粒子显示在某个 Canvas 之上
    /// </summary>
    /// <param name="particle">粒子系统</param>
    /// <param name="targetCanvas">目标 Canvas</param>
    /// <param name="offset">排序偏移量，默认 +1</param>
    public static void ShowAbove(ParticleSystem particle, Canvas targetCanvas, int offset = 1)
    {
        var renderer = particle.GetComponent<ParticleSystemRenderer>();
        if (renderer != null)
        {
            renderer.sortingLayerName = targetCanvas.sortingLayerName;
            renderer.sortingOrder = targetCanvas.sortingOrder + offset;
        }
    }

    /// <summary>
    /// 让粒子显示在某个 Canvas 之下
    /// </summary>
    public static void ShowBelow(ParticleSystem particle, Canvas targetCanvas, int offset = 1)
    {
        var renderer = particle.GetComponent<ParticleSystemRenderer>();
        if (renderer != null)
        {
            renderer.sortingLayerName = targetCanvas.sortingLayerName;
            renderer.sortingOrder = targetCanvas.sortingOrder - offset;
        }
    }
}
```

### 实际使用示例

```csharp
// 在背包面板播放"获得物品"特效
public class InventoryPanel : UIPanelBase
{
    [SerializeField] private ParticleSystem obtainEffect;  // 获得物品粒子
    [SerializeField] private Transform effectAnchor;       // 粒子挂载点

    public void PlayObtainEffect()
    {
        // 设置粒子排序在当前面板之上
        UIParticleSorter.ShowAbove(obtainEffect, GetComponentInParent<Canvas>());

        obtainEffect.gameObject.SetActive(true);
        obtainEffect.Play();
    }
}
```

### sortingOrder 规划表

与 [[【笔记】UGUI DrawCall影响因素全面测试]] 中的 Canvas 分层方案配合：

```
sortingOrder 分配：

  0  → Canvas_A (HUD层)        背景 + 血条
  5  → HUD粒子特效              sortingOrder = 5
 10  → Canvas_B (面板层)        背包/商店
 15  → 面板粒子特效             sortingOrder = 15
 20  → Canvas_C (弹窗层)        确认框
 30  → Canvas_D (全屏层)        结算界面
 35  → 全屏粒子特效             sortingOrder = 35
```

**关键原则**：粒子的 `sortingOrder` 要落在对应 Canvas 层的 `sortingOrder` 和下一层 Canvas 之间。

---

## 方案二：RenderTexture（复杂特效用）

适合需要 3D 模型 + 粒子 + 后处理混合的 UI 特效（如抽卡动画、角色展示）。

### 原理

```
独立 Camera (只看粒子)
    ↓ 渲染到
RenderTexture (GPU 纹理)
    ↓ 绑定到
UI RawImage (显示在 Canvas 上)
```

### 完整实现

```csharp
/// <summary>
/// RenderTexture UI粒子方案
/// 独立相机渲染粒子到 RT，UI 上用 RawImage 显示
/// </summary>
public class UIParticleRenderer : MonoBehaviour
{
    [Header("渲染配置")]
    [SerializeField] private Camera effectCamera;     // 专用渲染相机
    [SerializeField] private int textureSize = 512;   // RT 分辨率
    [SerializeField] private int depthBuffer = 24;    // 深度缓冲位数
    [SerializeField] private bool autoDisableCamera = true;  // 无粒子时自动关相机

    private RenderTexture renderTexture;
    private RawImage displayImage;

    private void Awake()
    {
        // 创建 RenderTexture
        renderTexture = new RenderTexture(textureSize, textureSize, depthBuffer);
        effectCamera.targetTexture = renderTexture;
        effectCamera.gameObject.SetActive(false);  // 默认关闭，按需开启

        // UI 上用 RawImage 显示
        displayImage = GetComponent<RawImage>();
        displayImage.texture = renderTexture;
    }

    /// <summary>
    /// 播放粒子特效
    /// </summary>
    /// <param name="effect">要播放的粒子系统</param>
    public void PlayEffect(ParticleSystem effect)
    {
        effectCamera.gameObject.SetActive(true);
        effect.Play();

        if (autoDisableCamera)
        {
            StartCoroutine(StopCameraAfterEffect(effect));
        }
    }

    /// <summary>
    /// 粒子播放结束后关闭相机，节省性能
    /// </summary>
    private IEnumerator StopCameraAfterEffect(ParticleSystem effect)
    {
        yield return new WaitWhile(() => effect != null && effect.isPlaying);
        effectCamera.gameObject.SetActive(false);
    }

    private void OnDestroy()
    {
        if (renderTexture != null)
        {
            renderTexture.Release();
            Destroy(renderTexture);
        }
    }
}
```

### 内存开销参考

| RT 分辨率 | 内存占用（RGBA32） | 适用场景 |
|-----------|-------------------|----------|
| 256×256 | 256 KB | 小型 UI 特效（金币飞溅） |
| 512×512 | 1 MB | 中型特效（获得物品） |
| 1024×1024 | 4 MB | 大型特效（抽卡动画） |
| 2048×2048 | 16 MB | 不推荐，用方案一替代 |

**注意**：RT 用完即 `Release()`，不要常驻。多个同类特效可复用同一张 RT。

---

## 方案三：UIParticles 组件（极致合批）

将粒子顶点直接注入 `CanvasRenderer`，实现与 UI 的真正合批。适合纯 2D UI 特效（按钮点击粒子、金币飘落等）。

### 原理

```
ParticleSystem 模拟
    ↓ 读取粒子数据 (GetParticles)
手动构建 Mesh
    ↓ 注入
CanvasRenderer.SetMesh()
    ↓ 随 Canvas 一起合批提交
GPU 渲染 (与 UI 元素在同一个 DC)
```

### 简化实现

```csharp
/// <summary>
/// 简化版 UI 粒子组件
/// 核心思路：每帧从 ParticleSystem 读模拟数据，手动构建 Mesh 注入 CanvasRenderer
/// 生产环境推荐使用成熟开源方案（见下方推荐）
/// </summary>
[RequireComponent(typeof(CanvasRenderer))]
[RequireComponent(typeof(ParticleSystem))]
public class UIParticle : MonoBehaviour
{
    private ParticleSystem ps;
    private ParticleSystemRenderer psRenderer;
    private CanvasRenderer canvasRenderer;
    private Material renderMaterial;

    // 粒子数据缓冲
    private readonly List<ParticleSystem.Particle> particles = new();
    private readonly List<UIVertex> vertices = new();
    private readonly List<int> triangles = new();
    private Mesh mesh;

    private void Awake()
    {
        ps = GetComponent<ParticleSystem>();
        psRenderer = GetComponent<ParticleSystemRenderer>();
        canvasRenderer = GetComponent<CanvasRenderer>();

        // 禁用原始渲染器，由 CanvasRenderer 接管
        psRenderer.enabled = false;
        renderMaterial = psRenderer.sharedMaterial;

        mesh = new Mesh { name = "UIParticle" };
        mesh.MarkDynamic();
    }

    private void Update()
    {
        SimulateAndBuildMesh();
    }

    private void SimulateAndBuildMesh()
    {
        int count = ps.GetParticles(particles);
        if (count == 0)
        {
            canvasRenderer.SetMesh(null);
            return;
        }

        vertices.Clear();
        triangles.Clear();

        for (int i = 0; i < count; i++)
        {
            var particle = particles[i];
            var size = particle.GetCurrentSize(ps);
            var center = particle.position;

            var halfW = size.x * 0.5f;
            var halfH = size.y * 0.5f;
            int baseIndex = vertices.Count;

            // 构建面向相机的四边形（两个三角形）
            vertices.Add(CreateVertex(new Vector3(center.x - halfW, center.y - halfH), particle));
            vertices.Add(CreateVertex(new Vector3(center.x + halfW, center.y - halfH), particle));
            vertices.Add(CreateVertex(new Vector3(center.x + halfW, center.y + halfH), particle));
            vertices.Add(CreateVertex(new Vector3(center.x - halfW, center.y + halfH), particle));

            triangles.Add(baseIndex);
            triangles.Add(baseIndex + 1);
            triangles.Add(baseIndex + 2);
            triangles.Add(baseIndex);
            triangles.Add(baseIndex + 2);
            triangles.Add(baseIndex + 3);
        }

        // 重建 Mesh 并注入 CanvasRenderer
        mesh.Clear();
        mesh.SetVertices(vertices.ConvertAll(v => v.position));
        mesh.SetTriangles(triangles, 0);
        mesh.SetUVs(0, vertices.ConvertAll(v => v.uv0));
        mesh.SetColors(vertices.ConvertAll(v => v.color));

        canvasRenderer.SetMesh(mesh);
        canvasRenderer.SetMaterial(renderMaterial, null);
    }

    private static UIVertex CreateVertex(Vector3 pos, ParticleSystem.Particle particle)
    {
        return new UIVertex
        {
            position = pos,
            color = particle.GetCurrentColor(ps: default),
            uv0 = new Vector4(0, 0, 0, 0),
            normal = new Vector3(0, 0, -1),
            tangent = new Vector4(1, 0, 0, -1)
        };
    }

    private void OnDestroy()
    {
        if (mesh != null) Destroy(mesh);
    }
}
```

### 开源方案推荐

生产环境不建议自己手写，推荐成熟开源项目：

| 项目 | 特点 | 地址 |
|------|------|------|
| **UIEffect** (mob-sakai) | 功能全面，支持粒子、扭曲、发光等 UI 效果 | github.com/mob-sakai/UIEffect |
| **UI Particles** (Unity 官方) | 官方包，部分 Unity 版本可用 | 包名 `com.unity.ui.particles` |
| **X-Particles** | 国人开源，轻量易用 | GitHub 搜索 `X-Particles Unity` |

---

## 方案选型决策树

```
需要 3D 粒子 / 复杂模型混合？
├─ 是 → RenderTexture (方案二)
└─ 否 → 纯 2D UI 粒子？
    │
    ├─ Canvas 已是 Screen Space Camera？
    │   └─ 是 → 方案一 (sortingOrder 直接控制) ← 推荐
    │
    └─ Canvas 是 Screen Space Overlay 且不想改？
        └─ 方案三 (UIParticles 组件)
```

### 典型场景对照

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 按钮点击粒子 | 方案一 或 方案三 | 粒子少，方案三可合批 |
| 获得物品特效 | 方案一 | 面板层级明确，sortingOrder 控制 |
| 抽卡动画（3D模型+粒子） | 方案二 | 需要 3D 渲染混合 |
| 金币飞溅 | 方案三 | 高频生成，合批优势大 |
| 战斗技能全屏特效 | 方案一 | 场景内直接渲染，性能最优 |
| 聊天框表情特效 | 方案三 | 轻量 2D 粒子，合批进 UI |

---

## 性能注意事项

| 关注点 | 建议 | 说明 |
|--------|------|------|
| **粒子数量** | UI 粒子 ≤ 50~100，移动端更少 | 过多粒子导致模拟和顶点开销激增 |
| **Simulate 开销** | 用 `Play()`，不用 `Simulate()` | 手动模拟有额外计算 |
| **RT 内存** | 用完即 `Release()`，可复用 | 512×512 = 1MB 常驻不可接受 |
| **材质 Shader** | 方案三需 UI 兼容 Shader | 必须支持 `_MainTex` + UI vertex stream |
| **Sorting Layer** | 提前规划 sortingOrder 分配表 | 避免层级混乱和反复调试 |
| **粒子池** | 高频特效用对象池复用 | 避免 `Instantiate`/`Destroy` 的 GC |

---

## 与 Canvas 分层架构的配合

结合 [[【笔记】UGUI DrawCall影响因素全面测试]] 中的 Canvas 分层设计：

```
Root Canvas (Screen Space Camera)
├── [Canvas_A] HUD层       sortingOrder=0  → HUD粒子 sortingOrder=5
├── [Canvas_B] 面板层      sortingOrder=10 → 面板粒子 sortingOrder=15
├── [Canvas_C] 弹窗层      sortingOrder=20 → 弹窗粒子 sortingOrder=25
└── [Canvas_D] 全屏层      sortingOrder=30 → 全屏粒子 sortingOrder=35
```

粒子的 `sortingOrder` 落在对应 Canvas 和下一层 Canvas 之间，确保层级正确。

---

## 相关链接

- 测试数据：[[【笔记】UGUI DrawCall影响因素全面测试]]
- 合批原理：[[【设计原理】UGUI合批机制深度解析]]
- 性能优化：[[【最佳实践】UI性能优化速查]]
- 性能陷阱：[[【踩坑】UGUI常见性能陷阱与根因分析]]

---

*创建日期: 2026-07-01*
*适用版本: Unity 2021.3 LTS+*
