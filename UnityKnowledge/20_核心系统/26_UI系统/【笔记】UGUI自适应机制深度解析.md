---
title: 【笔记】UGUI自适应机制深度解析
tags: ["Unity", "UI", "UGUI", "自适应", "CanvasScaler", "Anchor", "SafeArea", "深度解析"]
category: 核心系统/UI系统
created: "2026-07-02"
updated: "2026-07-02"
description: UGUI 自适应三层模型：CanvasScaler 全局缩放原理（含 Match Width Or Height 对数混合公式完整推导）、Anchor 锚点弹性布局机制、SafeArea 刘海屏适配，含实战代码
unity_version: 2021.3+
status: 待验证
validation: 基于知识库现有文档汇总整理
related:
  - "[[【笔记】UI系统知识脑图]]"
  - "[[【笔记】UGUI性能优化实战总览]]"
  - "[[【踩坑】Android常见问题清单]]"
  - "[[【笔记】UGUI DrawCall影响因素全面测试]]"
  - "[[【设计原理】UGUI合批机制深度解析]]"
  - "[[【笔记】UGUI深度解析]]"
author: llm
sources:
  - "[[【笔记】UGUI性能优化实战总览]]"
  - "[[【踩坑】Android常见问题清单]]"
  - "[[【笔记】UGUI DrawCall影响因素全面测试]]"
  - "[[【设计原理】UGUI合批机制深度解析]]"
  - "[[【笔记】UGUI深度解析]]"
---

# 【笔记】UGUI 自适应机制深度解析

> Unity UI 自适应的核心是 **CanvasScaler + Anchor（锚点）+ SafeArea** 三层协作。本文从公式层面拆解每层原理，把分散在 5+ 篇文档中的适配知识串成一张完整图景。

## 文档定位

知识库中 UI 适配知识分散在性能优化、平台适配、DrawCall 测试等文档中。本文将它们提炼为一篇系统性原理文档，可作为面试讲解和工程实现的入口。

---

## 自适应三层模型

```
┌─────────────────────────────────────────────────┐
│ 第1层：CanvasScaler — 全局缩放                   │
│   设计分辨率 → 任意屏幕尺寸                      │
│   Scale With Screen Size + match 参数            │
├─────────────────────────────────────────────────┤
│ 第2层：Anchor 锚点 — 局部弹性                    │
│   固定位置 / 拉伸填充 / 边距保持                 │
│   anchorMin / anchorMax 决定行为                 │
├─────────────────────────────────────────────────┤
│ 第3层：SafeArea — 安全区规避                     │
│   刘海/挖孔/圆角 → Screen.safeArea               │
│   像素坐标 → 归一化锚点                          │
└─────────────────────────────────────────────────┘
```

---

## 一、CanvasScaler：全局缩放策略

CanvasScaler 挂在 Canvas 根节点上，决定整个 UI 树如何映射到不同分辨率。三种 ScaleMode 原理完全不同。

### 1.1 Constant Pixel Size（固定像素）

**原理**：不做任何缩放，1 pixel = 1 unit。通过手动设 `scaleFactor` 做线性放大。

```
屏幕 1080p → UI 按原始像素绘制
屏幕 4K    → UI 看起来变小（物理屏幕相同但像素更多）
```

- **优点**：性能最优（无缩放计算），CPU 开销最低
- **缺点**：不同分辨率下 UI 大小不一致，不自动适配
- **适用**：PC 端固定分辨率、需要精确像素控制的场景

### 1.2 Scale With Screen Size（随屏幕缩放）— 最常用

**原理**：设定一个「参考分辨率」（如 1920×1080），CanvasScaler 自动计算缩放因子，让 UI 在任意分辨率下保持近似一致的视觉效果。

#### Match Width Or Height 实现原理

##### 要解决的核心问题

给定参考分辨率 `(Rx, Ry)` 和实际屏幕 `(W, H)`，需要算出**一个** `scaleFactor` 应用到整个 Canvas。Canvas 的 `scaleFactor` 是标量而非矢量——它等比缩放整个 UI 画布。所以问题变成：**宽的缩放比和高的缩放比不同时，如何混合为一个值？**

##### 朴素的线性混合（为什么不行）

最直觉的做法是线性加权平均：

```
scaleW = W / Rx
scaleH = H / Ry
scaleFactor = scaleW * (1 - match) + scaleH * match    // 线性混合
```

用参考分辨率 `(1080, 1920)`、match=0.5 测试极端比例：

| 屏幕 | scaleW | scaleH | 线性混合 | 问题 |
|------|--------|--------|---------|------|
| 540×1920 | 0.5 | 1.0 | 0.75 | 缩小到 75% |
| 2160×1920 | 2.0 | 1.0 | 1.5 | 放大到 150% |

0.75 和 1.5 数值上关于 1.0 对称（±0.5），但**倍数关系不对称**：缩小 25% 和放大 50% 的视觉感知差异很大。根本原因是缩放是乘法关系（2 倍 / 0.5 倍），用加法（算术平均）混合乘法关系，数学上不自然。

##### 对数空间混合（Unity 实际做法）

Unity CanvasScaler 源码核心逻辑：

```csharp
// CanvasScaler.cs 关键计算（简化）
float logWidth  = Mathf.Log(screenWidth  / m_ReferenceResolution.x, 2);
float logHeight = Mathf.Log(screenHeight / m_ReferenceResolution.y, 2);
float logWeighted = logWidth * (1 - m_MatchWidthOrHeight)
                  + logHeight * m_MatchWidthOrHeight;
float scaleFactor = Mathf.Pow(2, logWeighted);
```

拆解每一步：

```
                  W/Rx                    H/Ry
                   ↓                       ↓
logWidth  = log₂(W/Rx)     logHeight = log₂(H/Ry)
                   ↓                       ↓
                   └───── 对数加权混合 ──────┘
                   logScale = logWidth × (1-match) + logHeight × match
                                      ↓
                              scaleFactor = 2^logScale
```

##### 为什么用 log₂？

log₂ 把「倍数」变成「指数」：

| 缩放比 | log₂ | 含义 |
|--------|------|------|
| 4.0 | +2 | 放大 2 档（每档 ×2） |
| 2.0 | +1 | 放大 1 档 |
| 1.0 | 0 | 基准 |
| 0.5 | -1 | 缩小 1 档 |
| 0.25 | -2 | 缩小 2 档 |

在对数空间中，放大 1 档 (+1) 和缩小 1 档 (-1) 是**等距**的。对数空间中的线性混合 = 原始空间中的**几何加权平均**，保证 match 参数在 0~1 全范围内的缩放行为均匀对称。

##### 线性 vs 对数的效果对比

参考分辨率 `(1080, 1920)`，match=0.5：

| 屏幕 | scaleW | scaleH | 线性混合 | 对数混合 | 差异 |
|------|--------|--------|---------|---------|------|
| 1080×1920 | 1.0 | 1.0 | 1.0 | 1.0 | — |
| 1080×2400 | 1.0 | 1.25 | 1.125 | 1.118 | 小 |
| 540×1920 | 0.5 | 1.0 | **0.75** | **0.707** | 对数 = 1/√2 |
| 2160×1920 | 2.0 | 1.0 | **1.5** | **1.414** | 对数 = √2 |

对数结果 0.707 = 1/√2，1.414 = √2——完美对称（几何中项）。线性结果 0.75 和 1.5 不对称。

> **结论：对数混合保证 match 值在 0~1 全范围内的缩放行为是均匀的，偏向宽或偏向高的过渡是平滑的。**

##### Unity 源码完整流程

```
CanvasScaler.OnEnable → 注册到 CanvasUpdateRegistry

每帧（或分辨率变化时）HandleScaleWithScreenSize():
  ├── 获取 screenW, screenH（实际屏幕像素）
  ├── 获取 refW, refH（参考分辨率）
  ├── 根据 m_ScreenMatchMode 分支：
  │   ├── Shrink:  scaleFactor = min(screenW/refW, screenH/refH)
  │   ├── Expand:  scaleFactor = max(screenW/refW, screenH/refH)
  │   └── MatchWidthOrHeight:
  │       ├── logW = log₂(screenW / refW)
  │       ├── logH = log₂(screenH / refH)
  │       ├── logBlend = logW × (1-match) + logH × match
  │       └── scaleFactor = 2^logBlend
  │
  ├── 应用 scaleFactor 到 canvas.scaleFactor
  └── 设置 canvas 坐标系 = screenW × screenH / scaleFactor
      （UI 在此逻辑分辨率内布局）
```

##### 一句话总结

**Match Width Or Height 在对数空间中对宽高缩放比做线性插值，再还原回线性空间作为 scaleFactor。** 用对数是因为缩放是乘法关系，对数把乘法变成加法，使得 match 参数的调节在全范围内行为均匀对称。

#### match 参数选择

| match 值 | 含义 | 效果 | 适用 |
|-----------|------|------|------|
| **0** | 完全 Match Width | 以宽度为基准缩放。竖屏高度有余量 | 竖屏游戏 |
| **0.5** | 各 50% 权重 | 宽高都近似匹配，取折中 | 通用方案 |
| **1** | 完全 Match Height | 以高度为基准缩放。横屏宽度有余量 | 横屏游戏 |

**为什么竖屏用 match=0、横屏用 match=1？**

```
竖屏：屏幕比参考分辨率更窄（宽高比更小）
  约束瓶颈在宽度 → match=0 保证宽度铺满，上下自然延伸

横屏：屏幕比参考分辨率更扁（宽高比更大）
  约束瓶颈在高度 → match=1 保证高度铺满，左右自然延伸
```

#### Screen Match Mode 补充

| 模式 | 原理 | 效果 |
|------|------|------|
| `MatchWidthOrHeight` | 宽高对数权重混合 | 最常用，可精细控制 |
| `Shrink` | 取 Width 和 Height 中**较小**的缩放因子 | UI 可能变小但不会被裁切 |
| `Expand` | 取 Width 和 Height 中**较大**的缩放因子 | UI 可能超出但保证覆盖 |

```
举例：参考 1080×1920，实际 1080×2400
  scaleW = 1.0, scaleH = 1.25

  Shrink:  min(1.0, 1.25) = 1.0  → UI 按宽度缩放，上下留白
  Expand:  max(1.0, 1.25) = 1.25 → UI 放大，左右被裁
```

一般 UI 选 Shrink（宁可留白不可裁切关键信息）；全屏背景图选 Expand（宁可裁切不可留白）。

### 1.3 Constant Physical Size（固定物理尺寸）

**原理**：按 DPI（每英寸点数）将物理单位（厘米/英寸）转换为像素。

```
scaleFactor = Screen.dpi / fallbackScreenDPI
```

- **适用**：需要 UI 元素在不同设备上物理大小一致（1cm 按钮在任何手机都是 1cm）
- **缺点**：性能最差（DPI 转换），且设备 DPI 报告不一定准确

### 1.4 三种模式性能对比

来自 [[【笔记】UGUI DrawCall影响因素全面测试]] 的实测数据：

| ScaleMode | DrawCall | 渲染时间 | CPU 时间 |
|-----------|----------|----------|----------|
| Constant Pixel Size | 2 | 0.85ms | 1.2ms |
| Scale With Screen Size | 2 | 1.4ms | 2.3ms |
| Constant Physical Size | 2 | 1.8ms | 3.1ms |

> 结论：ScaleMode **不影响 DrawCall**，但影响 CPU 时间（缩放计算开销）。移动端必须用 Scale With Screen Size，PC 端可用 Constant Pixel Size 省 CPU。

---

## 二、Anchor 锚点系统：局部弹性布局

CanvasScaler 解决了「整体缩放」，但不同宽高比下元素位置怎么变？这就是 Anchor 的工作。

### 2.1 RectTransform 与锚点

每个 RectTransform 有两个锚点属性：

- **anchorMin** / **anchorMax**：定义元素在父容器中的参考位置（归一化 0~1）

```
情况1：anchorMin == anchorMax（点锚）
  → 元素大小固定，位置随屏幕拉伸而偏移
  → 例：右上角小图标 anchorMin = anchorMax = (1,1)

情况2：anchorMin != anchorMax（拉伸锚）
  → 元素随父容器拉伸，大小自适应
  → 例：全屏背景 anchorMin = (0,0), anchorMax = (1,1)
```

### 2.2 锚点计算公式

```
// 锚点为单点时（anchorMin == anchorMax = a）
元素中心位置 = 父容器左下角 + a * 父容器大小 + offset（像素偏移）
元素大小 = sizeDelta（固定尺寸）

// 锚点为拉伸时（anchorMin != anchorMax）
元素左下角 = 父容器左下角 + anchorMin * 父容器大小 + offsetMin
元素右上角 = 父容器右上角 - (1-anchorMax) * 父容器大小 - offsetMax
元素大小 = 随父容器变化而变化
```

### 2.3 常见锚点配置

```
┌─────────────────────────────────────────┐
│  ◇        ◇ ═══════════ ◇        ◇     │
│  左上     顶部拉伸      右上             │
│                                          │
│  ║                                ║     │
│  左侧拉伸                      右侧拉伸  │
│  ║                                ║     │
│                                          │
│  ◇        ◇ ═══════════ ◇        ◇     │
│  左下     底部拉伸      右下             │
└─────────────────────────────────────────┘

全屏拉伸：anchorMin=(0,0) anchorMax=(1,1) → 适应所有宽高比
居中固定：anchorMin=(0.5,0.5) anchorMax=(0.5,0.5) → 大小固定居中
底部条：  anchorMin=(0,0) anchorMax=(1,0) → 宽度拉伸高度固定
```

### 2.4 CanvasScaler + Anchor 的协作

```
CanvasScaler 决定「整体缩放因子」
  └── 让 1080p 的设计在 4K 上看起来差不多大

Anchor 决定「缩放后的布局弹性」
  └── 让不同宽高比的屏幕上元素位置合理
```

**举例**：设计分辨率 1920×1080，设备 2560×1080（超宽屏）

```
CanvasScaler (match=1, MatchHeight):
  → 高度 1080 == 参考高度 1080，scaleFactor = 1
  → UI 整体不变，但屏幕变宽了 640px

Anchor 的作用：
  全屏背景 (0,0)~(1,1) → 自动拉伸覆盖多出来的宽度
  右上角按钮 (1,1)     → 自动靠右，保持 offset 偏移
  居中 Logo (0.5,0.5)  → 居中不变
  底部血条 (0,0)~(1,0) → 宽度自动拉伸
```

### 2.5 锚点与 Rebuild 的关系

修改 Anchor 属性（anchorMin / anchorMax）会触发 Layout Rebuild（[[【笔记】UGUI深度解析]]）。因此：

- **避免在 Update 中修改 Anchor**（每帧触发 Rebuild）
- **避免在 Layout 系统中做动画**（UGUI 第26章「UI 动画实现」）
- 如需动态调整，使用 `LayoutRebuilder.MarkLayoutForRebuild()` 标记后等下一帧统一处理

---

## 三、SafeArea：刘海/挖孔屏适配

### 3.1 原理

`Screen.safeArea` 返回一个 `Rect`，表示屏幕上「不被遮挡的安全区域」。在刘海屏、挖孔屏、圆角屏上，这个区域比 `Screen.width × Screen.height` 小。

```
屏幕物理区域：    ┌─────────────────────┐
                  │     ▓▓▓（刘海）      │
                  │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
                  │                     │
                  │   SafeArea          │
                  │   (真正可用的区域)    │
                  │                     │
                  └─────────────────────┘
```

### 3.2 实战代码

来自 [[【踩坑】Android常见问题清单]]：

```csharp
/// <summary>
/// 安全区适配：将 RectTransform 的锚点设置为 Screen.safeArea 的归一化值
/// </summary>
public class SafeAreaAdapter : MonoBehaviour
{
    [SerializeField] private RectTransform panel;

    private void Start()
    {
        ApplySafeArea();
    }

    private void ApplySafeArea()
    {
        Rect safeArea = Screen.safeArea;

        // 将像素坐标转换为归一化锚点坐标
        Vector2 anchorMin = safeArea.position;
        Vector2 anchorMax = safeArea.position + safeArea.size;

        anchorMin.x /= Screen.width;
        anchorMin.y /= Screen.height;
        anchorMax.x /= Screen.width;
        anchorMax.y /= Screen.height;

        panel.anchorMin = anchorMin;
        panel.anchorMax = anchorMax;
    }
}
```

### 3.3 三种刘海屏方案对比

| 方案 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| 改 ViewPort | 调整 `Camera.rect` | 相机和 UI 都生效 | 改变渲染区域有黑边 |
| 改 SafeArea 锚点 | `Screen.safeArea` → anchor | 精准、无黑边 | 只影响 UI 不影响 3D |
| 缩放根节点 | 整体 `scaleFactor` 下调 | 简单 | 精度差、浪费屏幕空间 |

> **推荐**：方案2（SafeArea 锚点），精准且无副作用。

### 3.4 设备分辨率适配

来自 [[【踩坑】Android常见问题清单]]，处理超长屏比例（18:9 / 19.5:9 / 20:9）：

```csharp
/// <summary>
/// 超长屏分辨率适配
/// </summary>
public class ResolutionAdapter : MonoBehaviour
{
    [SerializeField] private Canvas canvas;

    private void Start()
    {
        AdaptToScreen();
    }

    private void AdaptToScreen()
    {
        float aspectRatio = (float)Screen.width / Screen.height;

        // 常见比例
        // 16:9   = 1.778 (1920x1080)
        // 18:9   = 2.000 (2160x1080)
        // 19.5:9 = 2.167 (2340x1080)
        // 20:9   = 2.222 (2400x1080)

        if (aspectRatio > 2.0f)
        {
            // 长屏适配：调整 scaleFactor 补偿
            canvas.scaleFactor = CalculateLongScreenScale(aspectRatio);
        }
    }

    private float CalculateLongScreenScale(float aspectRatio)
    {
        return 1f + (aspectRatio - 1.778f) * 0.1f;
    }
}
```

---

## 四、横竖屏切换

运行时修改 CanvasScaler 参数 + Screen.orientation（[[【笔记】UGUI性能优化实战总览]]）：

```csharp
/// <summary>
/// 横竖屏切换管理器
/// </summary>
public class OrientationSwitcher : MonoBehaviour
{
    [SerializeField] private CanvasScaler scaler;
    [SerializeField] private Canvas fader;  // 全屏遮罩

    public void SwitchToPortrait()
    {
        StartCoroutine(SwitchRoutine(ScreenOrientation.Portrait,
            new Vector2(1080, 1920), 0));
    }

    public void SwitchToLandscape()
    {
        StartCoroutine(SwitchRoutine(ScreenOrientation.Landscape,
            new Vector2(1920, 1080), 1));
    }

    private IEnumerator SwitchRoutine(
        ScreenOrientation orientation, Vector2 refRes, float match)
    {
        // 1. 全屏遮罩淡入，避免切换瞬间 UI 错位
        fader.gameObject.SetActive(true);

        yield return Fade(0, 1, 0.2f);

        // 2. 切换屏幕方向与 CanvasScaler 参数
        Screen.orientation = orientation;
        scaler.referenceResolution = refRes;
        scaler.matchWidthOrHeight = match;

        yield return null;  // 等一帧让 Layout 重建

        // 3. 遮罩淡出
        yield return Fade(1, 0, 0.2f);
        fader.gameObject.SetActive(false);
    }

    private IEnumerator Fade(float from, float to, float duration)
    {
        // ... DOTween 或手动插值
        yield break;
    }
}
```

### 要点

| 要点 | 说明 |
|------|------|
| 遮罩 fade | 切换瞬间 CanvasScaler 会触发 Layout Rebuild，必须遮盖 |
| 等一帧 | 修改参数后 `yield return null`，等 Layout 重建完成 |
| 竖屏 match=0 | 保证宽度填满 |
| 横屏 match=1 | 保证高度填满 |

---

## 五、自适应背景

背景图要在任意宽高比下**铺满屏幕且不变形**：

| 方案 | 原理 | 效果 |
|------|------|------|
| 拉伸 | Sprite 拉伸至全屏 | 会变形 |
| 裁切 | 保持宽高比，超出部分裁切 | 不变形但会丢边 |
| Shader 采样矩阵 | GPU 级别修改 UV 采样 | 最优，无 CPU 开销 |

### 裁切方案（推荐）

```csharp
/// <summary>
/// 自适应背景：保持图片宽高比，裁切超出部分
/// </summary>
public class AspectRatioBackground : MonoBehaviour
{
    [SerializeField] private Image bgImage;
    [SerializeField] private Vector2 designAspect = new(1920, 1080);

    private void Start()
    {
        AdjustBackground();
    }

    private void AdjustBackground()
    {
        float screenAspect = (float)Screen.width / Screen.height;
        float designRatio = designAspect.x / designAspect.y;

        RectTransform rt = bgImage.rectTransform;

        if (screenAspect > designRatio)
        {
            // 屏幕更宽：以宽度为基准，高度裁切
            rt.anchorMin = new Vector2(0, 0.5f);
            rt.anchorMax = new Vector2(1, 0.5f);
            float height = Screen.width / designRatio;
            rt.sizeDelta = new Vector2(0, height);
        }
        else
        {
            // 屏幕更高：以高度为基准，宽度裁切
            rt.anchorMin = new Vector2(0.5f, 0);
            rt.anchorMax = new Vector2(0.5f, 1);
            float width = Screen.height * designRatio;
            rt.sizeDelta = new Vector2(width, 0);
        }
    }
}
```

---

## 六、ContentSizeFitter：内容自适应尺寸

ContentSizeFitter 是另一层自适应——根据内容自动调整元素大小（[[【笔记】UGUI深度解析]] 源码解析）。

### 原理

```
ContentSizeFitter 工作流程：
  ① Layout 系统收集子元素尺寸
  ② ContentSizeFitter 根据 Layout 结果调整自身大小
  ③ 调用 SetSizeWithCurrentAnchors 更新 RectTransform
```

### 模式

| 模式 | Horizontal Fit | Vertical Fit | 效果 |
|------|----------------|--------------|------|
| Unconstrained | — | — | 不自动调整 |
| Min Size | 最小尺寸 | 最小尺寸 | 收缩到内容最小值 |
| Preferred Size | 最佳尺寸 | 最佳尺寸 | 展开到内容所需大小 |

> 注意：ContentSizeFitter 会触发 Layout Rebuild，大量使用会导致性能问题。滚动列表中可用对象池 + 手动控制替代。

---

## 七、实战速查表

| 场景 | 推荐方案 |
|------|----------|
| 移动端通用 | Scale With Screen Size + match=0.5 |
| 竖屏游戏 | Ref=(1080,1920), match=0 |
| 横屏游戏 | Ref=(1920,1080), match=1 |
| PC 固定分辨率 | Constant Pixel Size |
| 刘海屏 | SafeArea 锚点适配 |
| 超长屏 (18:9+) | ResolutionAdapter 微调 scaleFactor |
| 横竖屏切换 | 运行时改 CanvasScaler + fade 遮罩 |
| 自适应背景 | 裁切方案保持宽高比 |
| 文本自适应 | ContentSizeFitter Preferred Size |
| 列表 Item | Horizontal/Vertical Layout Group + ContentSizeFitter |

---

## 八、性能注意

- **CanvasScaler 每帧计算缩放比例** → 所有 RectTransform 重新计算 → Layout System 重新布局（[[【设计原理】UGUI合批机制深度解析]]）
- Canvas Scaler 开销排序：`Constant Pixel Size < Scale With Screen Size < Constant Physical Size`
- 修改 Anchor 会触发 Layout Rebuild，**不要在 Update 中修改**
- ContentSizeFitter 大量使用会导致 Layout Rebuild 风暴，列表场景谨慎使用
- 横竖屏切换时必须遮罩 + 等一帧，避免 Layout Rebuild 过程中的闪烁

---

## 九、面试自测

- [ ] 能说出 CanvasScaler 三种 ScaleMode 的原理差异
- [ ] 能推导 Match Width Or Height 的对数混合公式
- [ ] 能解释为什么 match=0 适合竖屏、match=1 适合横屏
- [ ] 能解释点锚与拉伸锚的区别及各自的计算公式
- [ ] 能说出 CanvasScaler 与 Anchor 各自负责什么层面的适配
- [ ] 能写出 SafeArea 适配的核心代码（像素坐标 → 归一化锚点）
- [ ] 能说出三种刘海屏方案及推荐选择
- [ ] 能解释 Shrink vs Expand 的区别
- [ ] 知道修改 Anchor 会触发 Layout Rebuild

---

## 相关文档

| 文档 | 覆盖内容 |
|------|----------|
| [[【笔记】UI系统知识脑图]] | UI 系统全知识体系脑图 |
| [[【笔记】UGUI性能优化实战总览]] | 刘海屏 / 横竖屏 / 自适应背景实战方案 |
| [[【踩坑】Android常见问题清单]] | ResolutionAdapter + SafeAreaAdapter 完整代码 |
| [[【笔记】UGUI DrawCall影响因素全面测试]] | CanvasScaler 三种模式性能测试数据 |
| [[【设计原理】UGUI合批机制深度解析]] | FixedResolutionScaler 手动缩放实现 |
| [[【笔记】UGUI深度解析]] | ContentSizeFitter 自适应尺寸源码解析 |
| [[【最佳实践】Android专项]] | Android 平台 SafeArea 简要代码 |
| [[UI系统专题索引]] | UI 系统文档导航 |
