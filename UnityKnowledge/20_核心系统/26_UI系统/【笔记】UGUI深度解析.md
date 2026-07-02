---
title: 【笔记】UGUI深度解析
tags: ["Unity", "游戏系统", "UGUI", "源码解析", "设计原理", "UI"]
category: 核心系统/游戏系统
created: "2026-03-05 08:32"
updated: "2026-05-29 00:00"
description: UGUI系统源码深度解析
unity_version: 2021.3+
status: 待验证
validation: 未经测试
related: ["【教程】UI系统架构"]
author: llm
---

# 【笔记】UGUI深度解析

> 专题课程 | UI系统进阶 | 源码分析版

## 文档定位

基于UnityCsReference源码深度分析UGUI系统内部实现，包括Canvas重构流程、Layout System布局计算、Graphic渲染管线、ClipRect裁剪机制和EventSystem事件传递链。

**相关文档**：[[【教程】UI系统架构]]

---

---

## 0. UGUI 源码架构总览

### 0.1 核心命名空间

```
┌─────────────────────────────────────────────────────────────┐
│                   UGUI 核心命名空间                           │
│                                                             │
│  UnityEngine.UI                                             │
│  ├── Graphic (抽象基类)                                      │
│  │   ├── Image                                             │
│  │   ├── Text (Legacy)                                     │
│  │   ├── RawImage                                          │
│  │   └── Mask                                              │
│  ├── CanvasRenderer                                        │
│  ├── LayoutGroup (抽象基类)                                  │
│  │   ├── HorizontalLayoutGroup                             │
│  │   ├── VerticalLayoutGroup                               │
│  │   └── GridLayoutGroup                                   │
│  ├── GraphicRaycaster                                      │
│  └── RectMask2D                                            │
│                                                             │
│  UnityEngine.EventSystems                                   │
│  ├── EventSystem                                           │
│  ├── BaseInputModule (抽象基类)                              │
│  │   ├── StandaloneInputModule                             │
│  │   └── TouchInputModule                                  │
│  ├── BaseRaycaster (抽象基类)                               │
│  │   ├── GraphicRaycaster                                  │
│  │   ├── PhysicsRaycaster                                  │
│  │   └── Physics2DRaycaster                                │
│  └── ExecuteEvents                                         │
└─────────────────────────────────────────────────────────────┘
```

### 0.2 核心类关系图

```
                    ┌──────────────┐
                    │  UIBehaviour │ (抽象基类)
                    │  UnityEngine │
                    └──────┬───────┘
                           │ 继承
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
    │   Graphic   │ │ LayoutGroup │ │ Selectable  │
    │   (抽象)    │ │   (抽象)    │ │   (抽象)    │
    └──────┬──────┘ └─────────────┘ └─────────────┘
           │
    ┌──────┼──────┬──────────┬──────────┐
    │      │      │          │          │
  Image  Text  RawImage    Mask    InputField

                    ┌──────────────────┐
                    │ ICanvasElement   │ (接口)
                    └────────┬─────────┘
                             │ 实现
           ┌─────────────────┼─────────────────┐
           │                 │                 │
       Graphic         LayoutRebuilder    MaskUtilities

                    ┌──────────────────┐
                    │ IMaterialModifier│ (接口)
                    └────────┬─────────┘
                             │ 实现
                    ┌────────┼────────┐
                    │                 │
                  Mask           RectMask2D
```

### 0.3 源码获取与调试

```csharp
/// <summary>
/// UGUI 源码获取方式
/// </summary>
public static class UGUISourceAccess
{
    /*
    ========== 方式一：GitHub 官方仓库 ==========

    地址：https://github.com/Unity-Technologies/uGUI

    克隆命令：
    git clone https://github.com/Unity-Technologies/uGUI.git

    ========== 方式二：Unity 安装目录 ==========

    路径（Windows）：
    Unity安装目录/Editor/Data/unity-managed/UnityEngine.UI.dll

    路径（macOS）：
    /Applications/Unity/Unity.app/Contents/unity-managed/UnityEngine.UI.dll

    ========== 方式三：Package Manager ==========

    Unity 2018.3+ 可通过 Package Manager 查看 UI 包源码：
    Window > Package Manager > Unity UI (uGUI)

    ========== 调试技巧 ==========

    1. 将源码放入项目的 Plugins 或自定义文件夹
    2. 删除或重命名原来的 UnityEngine.UI.dll
    3. 可在源码中添加 Debug.Log 进行调试

    注意：调试完成后记得还原，避免版本冲突
    */
}
```

### 0.4 源码阅读路径

```
推荐阅读顺序（由浅入深）：

1. 入口点
   └── Graphic.cs           // UI 元素基类，理解渲染入口

2. 渲染管线
   ├── CanvasUpdateRegistry.cs  // 更新注册中心
   ├── VertexHelper.cs          // 顶点辅助类
   └── Image.cs                 // 最常用的 Graphic 实现

3. 布局系统
   ├── LayoutGroup.cs           // 布局基类
   ├── LayoutRebuilder.cs       // 布局重建器
   └── ContentSizeFitter.cs     // 自适应尺寸

4. 事件系统
   ├── EventSystem.cs           // 事件管理器
   ├── GraphicRaycaster.cs      // UI 射线检测
   └── ExecuteEvents.cs         // 事件执行器

5. 高级特性
   ├── Mask.cs                  // 模板遮罩
   ├── RectMask2D.cs            // 矩形裁剪
   └── ScrollRect.cs            // 滚动视图
```

---

## 1. UGUI渲染机制

### 1.1 渲染架构

```
┌─────────────────────────────────────────────────────────────┐
│                    UGUI 渲染架构                              │
│                                                             │
│   Canvas                                                    │
│     │                                                       │
│     ├── CanvasRenderer (每个UI元素)                          │
│     │     ├── 网格生成                                       │
│     │     └── 材质设置                                       │
│     │                                                       │
│     ├── Graphic (基类)                                       │
│     │     ├── Image                                         │
│     │     ├── Text                                          │
│     │     └── RawImage                                      │
│     │                                                       │
│     └── 批处理规则                                           │
│           ├── 相同材质                                       │
│           ├── 相同纹理（图集）                                │
│           └── 层级连续                                       │
│                                                             │
│   渲染流程:                                                  │
│   Update() → Rebuild() → BatchBuild() → Draw()              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Canvas渲染模式详解

```csharp
using UnityEngine;

/// <summary>
/// Canvas渲染模式分析
/// </summary>
public class CanvasModeAnalysis : MonoBehaviour
{
    [Header("Overlay模式")]
    [SerializeField] private Canvas overlayCanvas;

    [Header("Camera模式")]
    [SerializeField] private Canvas cameraCanvas;
    [SerializeField] private Camera uiCamera;

    [Header("World模式")]
    [SerializeField] private Canvas worldCanvas;

    /*
    ========== Screen Space - Overlay ==========
    特点：
    - 渲染在场景最上层，不受相机影响
    - 直接渲染到屏幕
    - 适合：HUD、固定UI

    性能：
    - 最快的渲染模式
    - 不需要相机渲染
    - DrawCall合并效率最高

    ========== Screen Space - Camera ==========
    特点：
    - 相对于相机渲染
    - 可设置Plane Distance产生透视效果
    - 适合：需要3D效果的UI

    性能：
    - 需要额外相机
    - 可与场景物体产生遮挡关系

    ========== World Space ==========
    特点：
    - 作为3D物体存在于场景中
    - 可被遮挡、有深度
    - 适合：血条、对话框、物品标签

    性能：
    - 与场景物体一起渲染
    - 每个Canvas独立批处理
    */
}
```

### 1.3 批处理规则

```csharp
/// <summary>
/// UGUI批处理优化
/// </summary>
public static class UGUIBatchOptimizer
{
    /*
    ========== 合批条件 ==========

    1. 相同材质
       - 使用相同的Shader
       - 使用相同的材质属性

    2. 相同纹理（图集）
       - 同一Sprite Atlas
       - 纹理格式一致

    3. 层级连续
       - 中间没有其他材质的UI元素
       - 正确的Hierarchy顺序

    ========== 打断合批的因素 ==========

    1. 不同材质/纹理
    2. 不同的渲染层
    3. 遮罩（RectMask2D/Mask）
    4. 文字（每个字体纹理可能不同）
    5. 材质属性修改（颜色、透明度）
    */

    /// <summary>
    /// 分析Canvas的DrawCall
    /// </summary>
    public static void AnalyzeDrawCalls(Canvas canvas)
    {
        var graphics = canvas.GetComponentsInChildren<Graphic>(true);
        var batches = new List<BatchInfo>();

        BatchInfo currentBatch = null;

        foreach (var graphic in graphics)
        {
            if (!graphic.gameObject.activeInHierarchy) continue;

            var material = graphic.material;
            var texture = graphic.mainTexture;

            // 检查是否可以合并
            bool canBatch = currentBatch != null &&
                           currentBatch.Material == material &&
                           currentBatch.Texture == texture;

            if (!canBatch)
            {
                currentBatch = new BatchInfo
                {
                    Material = material,
                    Texture = texture,
                    Graphics = new List<Graphic>()
                };
                batches.Add(currentBatch);
            }

            currentBatch.Graphics.Add(graphic);
        }

        // 输出分析结果
        Debug.Log($"Canvas: {canvas.name}");
        Debug.Log($"Total DrawCalls: {batches.Count}");

        for (int i = 0; i < batches.Count; i++)
        {
            var batch = batches[i];
            Debug.Log($"Batch {i}: {batch.Graphics.Count} elements, " +
                     $"Texture: {batch.Texture?.name ?? "null"}");
        }
    }

    private class BatchInfo
    {
        public Material Material;
        public Texture Texture;
        public List<Graphic> Graphics;
    }

    /// <summary>
    /// 优化UI层级顺序
    /// </summary>
    public static void OptimizeHierarchyOrder(Transform parent)
    {
        var graphics = new List<Graphic>();

        foreach (Transform child in parent)
        {
            var graphic = child.GetComponent<Graphic>();
            if (graphic != null)
                graphics.Add(graphic);
        }

        // 按材质和纹理排序
        graphics.Sort((a, b) =>
        {
            int materialCompare = a.material.GetInstanceID().CompareTo(b.material.GetInstanceID());
            if (materialCompare != 0) return materialCompare;

            int textureCompare = (a.mainTexture?.GetInstanceID() ?? 0)
                .CompareTo(b.mainTexture?.GetInstanceID() ?? 0);
            return textureCompare;
        });

        // 重新排序
        for (int i = 0; i < graphics.Count; i++)
        {
            graphics[i].transform.SetSiblingIndex(i);
        }
    }
}
```

### 1.4 渲染系统源码分析

#### 1.4.1 Graphic 基类源码

```csharp
// ===== 源码文件: UnityEngine.UI/Core/Graphic.cs =====
// Graphic 是所有 UI 可视元素的抽象基类

namespace UnityEngine.UI
{
    /// <summary>
    /// Graphic 基类 - 所有 UI 可视元素的基类
    /// 源码路径: UnityEngine.UI/Core/Graphic.cs
    /// </summary>
    public abstract partial class Graphic :
        UIBehaviour,           // Unity UI 行为基类
        ICanvasElement         // Canvas 元素接口，支持重建
    {
        // ========== 核心成员变量 ==========

        // 顶点脏标记 - 当顶点数据需要重建时设置
        [NonSerialized] private bool m_VertsDirty;

        // 材质脏标记 - 当材质需要重建时设置
        [NonSerialized] private bool m_MaterialDirty;

        // 顶点颜色
        [SerializeField] private Color m_Color = Color.white;

        // 射线检测目标
        [SerializeField] private bool m_RaycastTarget = true;

        // 缓存的 RectTransform
        [NonSerialized] private RectTransform m_RectTransform;

        // 缓存的 CanvasRenderer
        [NonSerialized] private CanvasRenderer m_CanvasRenderer;

        // 主纹理（由子类重写）
        public virtual Texture mainTexture => null;

        // ========== 核心属性 ==========

        // RectTransform 缓存属性
        public RectTransform rectTransform
        {
            get
            {
                if (m_RectTransform == null)
                    m_RectTransform = GetComponent<RectTransform>();
                return m_RectTransform;
            }
        }

        // CanvasRenderer 缓存属性
        public CanvasRenderer canvasRenderer
        {
            get
            {
                if (m_CanvasRenderer == null)
                    m_CanvasRenderer = GetComponent<CanvasRenderer>();
                return m_CanvasRenderer;
            }
        }

        // depth 表示在 Canvas 中的层级深度
        // -1 表示不在任何 Canvas 中或被禁用
        public int depth
        {
            get
            {
                if (canvas == null)
                    return -1;
                return canvasRenderer.absoluteDepth;
            }
        }

        // ========== 脏标记设置 ==========

        // 标记顶点需要重建
        // 当 RectTransform 尺寸变化时自动调用
        protected override void OnRectTransformDimensionsChange()
        {
            if (gameObject.activeInHierarchy)
            {
                SetVerticesDirty();  // 触发顶点重建
            }
        }

        // 设置顶点脏标记
        public virtual void SetVerticesDirty()
        {
            if (!IsActive())
                return;

            m_VertsDirty = true;

            // 注册到 CanvasUpdateRegistry 等待重建
            CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);
        }

        // 设置材质脏标记
        public virtual void SetMaterialDirty()
        {
            if (!IsActive())
                return;

            m_MaterialDirty = true;

            // 注册到 CanvasUpdateRegistry 等待重建
            CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);
        }

        // ========== ICanvasElement 接口实现 ==========

        // 重建回调 - 在 Canvas.willRenderCanvases 时被调用
        public virtual void Rebuild(CanvasUpdate executing)
        {
            // 只在 PreRender 阶段处理
            if (executing == CanvasUpdate.PreRender)
            {
                // 处理顶点重建
                if (m_VertsDirty)
                {
                    m_VertsDirty = false;
                    DoMeshGeneration();  // 执行网格生成
                }

                // 处理材质重建
                if (m_MaterialDirty)
                {
                    m_MaterialDirty = false;
                    UpdateMaterial();  // 更新材质
                }
            }
        }

        // ========== 网格生成流程 ==========

        private void DoMeshGeneration()
        {
            // 重置 CanvasRenderer 的网格
            canvasRenderer.Clear();

            // 调用子类的 OnPopulateMesh 生成网格
            var vh = new VertexHelper();
            OnPopulateMesh(vh);  // 子类实现

            // 填充网格到 CanvasRenderer
            var m = new Mesh();
            vh.FillMesh(m);
            canvasRenderer.SetMesh(m);
        }

        // 子类重写此方法生成自定义网格
        // Image、Text 等组件都重写了此方法
        protected virtual void OnPopulateMesh(VertexHelper vh)
        {
            var r = GetPixelAdjustedRect();
            var v = new Vector4(r.x, r.y, r.x + r.width, r.y + r.height);

            // 默认生成一个简单的四边形
            vh.AddUIVertexQuad(new[]
            {
                new UIVertex { position = new Vector3(v.x, v.y), color = color },
                new UIVertex { position = new Vector3(v.x, v.w), color = color },
                new UIVertex { position = new Vector3(v.z, v.w), color = color },
                new UIVertex { position = new Vector3(v.z, v.y), color = color }
            });
        }

        // ========== OnEnable/OnDisable ==========

        protected override void OnEnable()
        {
            base.OnEnable();

            // 注册到 GraphicRegistry，用于射线检测
            GraphicRegistry.RegisterGraphicForCanvas(canvas, this);

            // 标记需要重建
            SetVerticesDirty();
            SetMaterialDirty();
        }

        protected override void OnDisable()
        {
            // 从 GraphicRegistry 注销
            GraphicRegistry.UnregisterGraphicForCanvas(canvas, this);

            // 取消重建注册
            CanvasUpdateRegistry.UnRegisterCanvasElementForRebuild(this);

            // 清理 CanvasRenderer
            canvasRenderer.Clear();

            base.OnDisable();
        }
    }
}

/*
========== 源码要点总结 ==========

1. Graphic 继承自 UIBehaviour，实现 ICanvasElement 接口
2. 使用脏标记模式 (m_VertsDirty, m_MaterialDirty) 避免重复重建
3. SetVerticesDirty() 触发注册到 CanvasUpdateRegistry
4. Rebuild() 在 Canvas 渲染前被调用，执行网格生成
5. OnPopulateMesh() 是子类生成网格的入口点
*/
```

#### 1.4.2 CanvasUpdateRegistry 源码

```csharp
// ===== 源码文件: UnityEngine.UI/Core/CanvasUpdateRegistry.cs =====
// Canvas 更新注册中心 - 管理 UI 元素的重建队列

namespace UnityEngine.UI
{
    /// <summary>
    /// CanvasUpdateRegistry - 管理 UI 元素的批量重建
    /// 源码路径: UnityEngine.UI/Core/CanvasUpdateRegistry.cs
    /// </summary>
    public static class CanvasUpdateRegistry
    {
        // ========== 核心成员 ==========

        // 图形重建队列（按 Canvas 分组）
        private static readonly IndexedSet<ICanvasElement> m_ElementsForGraphicRebuild =
            new IndexedSet<ICanvasElement>();

        // 布局重建队列（按 Canvas 分组）
        private static readonly IndexedSet<ICanvasElement> m_ElementsForLayoutRebuild =
            new IndexedSet<ICanvasElement>();

        // 是否正在执行重建
        private static bool m_PerformingRebuild;

        // ========== 初始化 ==========

        static CanvasUpdateRegistry()
        {
            // 注册到 Canvas 的 willRenderCanvases 回调
            // 这个回调在每帧渲染 Canvas 之前触发
            Canvas.willRenderCanvases += PerformUpdate;
        }

        // ========== 注册方法 ==========

        /// <summary>
        /// 注册图形重建元素
        /// </summary>
        public static void RegisterCanvasElementForGraphicRebuild(ICanvasElement element)
        {
            if (element == null || m_PerformingRebuild)
                return;

            m_ElementsForGraphicRebuild.AddUnique(element);
        }

        /// <summary>
        /// 注册布局重建元素
        /// </summary>
        public static void RegisterCanvasElementForLayoutRebuild(ICanvasElement element)
        {
            if (element == null || m_PerformingRebuild)
                return;

            m_ElementsForLayoutRebuild.AddUnique(element);
        }

        // ========== 核心更新逻辑 ==========

        /// <summary>
        /// 执行更新 - 每帧渲染前调用
        /// </summary>
        private static void PerformUpdate()
        {
            // 设置重建标志，防止在重建过程中修改队列
            m_PerformingRebuild = true;

            try
            {
                // ===== 阶段 1: 布局重建 =====
                // 布局重建先于图形重建，因为布局会影响顶点位置
                ProcessLayoutRebuild();

                // ===== 阶段 2: 裁剪 =====
                // 处理 RectMask2D 的裁剪区域
                ClipperRegistry.instance.Cull();

                // ===== 阶段 3: 图形重建 =====
                // 重建顶点和材质
                ProcessGraphicRebuild();
            }
            finally
            {
                m_PerformingRebuild = false;
            }
        }

        /// <summary>
        /// 处理布局重建
        /// </summary>
        private static void ProcessLayoutRebuild()
        {
            // 按 hierarchy 深度排序，确保父级先处理
            SortLayoutRebuildQueue();

            // 按阶段执行重建
            for (int i = 0; i < m_ElementsForLayoutRebuild.Count; i++)
            {
                var element = m_ElementsForLayoutRebuild[i];

                try
                {
                    // 先计算布局
                    element.Rebuild(CanvasUpdate.Prelayout);
                    element.Rebuild(CanvasUpdate.Layout);
                    element.Rebuild(CanvasUpdate.PostLayout);
                }
                catch (Exception e)
                {
                    Debug.LogException(e);
                }
            }

            m_ElementsForLayoutRebuild.Clear();
        }

        /// <summary>
        /// 处理图形重建
        /// </summary>
        private static void ProcessGraphicRebuild()
        {
            for (int i = 0; i < m_ElementsForGraphicRebuild.Count; i++)
            {
                var element = m_ElementsForGraphicRebuild[i];

                try
                {
                    element.Rebuild(CanvasUpdate.PreRender);
                }
                catch (Exception e)
                {
                    Debug.LogException(e);
                }
            }

            m_ElementsForGraphicRebuild.Clear();
        }

        // ========== 排序 ==========

        private static void SortLayoutRebuildQueue()
        {
            m_ElementsForLayoutRebuild.Sort(
                (a, b) => a.transform.GetHierarchyDepth().CompareTo(b.transform.GetHierarchyDepth()));
        }
    }
}

/*
========== 源码要点总结 ==========

1. CanvasUpdateRegistry 是静态类，通过 Canvas.willRenderCanvases 回调触发更新
2. 使用 IndexedSet 保证元素唯一性，避免重复注册
3. 更新顺序：布局重建 → 裁剪 → 图形重建
4. 布局重建按 hierarchy 深度排序，确保父级先处理
5. m_PerformingRebuild 标志防止在重建过程中修改队列

========== CanvasUpdate 枚举阶段 ==========

public enum CanvasUpdate
{
    Prelayout = 0,    // 布局前
    Layout = 1,       // 布局计算
    PostLayout = 2,   // 布局后
    PreRender = 3,    // 渲染前（图形重建）
    LatePreRender = 4 // 延迟渲染前
}
*/
```

#### 1.4.3 VertexHelper 网格构建

```csharp
// ===== 源码文件: UnityEngine.UI/Core/VertexHelper.cs =====
// VertexHelper - UI 网格顶点辅助类

namespace UnityEngine.UI
{
    /// <summary>
    /// VertexHelper - 高效的 UI 网格构建辅助类
    /// 源码路径: UnityEngine.UI/Core/VertexHelper.cs
    /// </summary>
    public class VertexHelper : IDisposable
    {
        // ========== 内部数据结构 ==========

        // 使用 List 存储顶点数据
        private List<UIVertex> m_Verts = new List<UIVertex>();

        // 使用 List 存储三角形索引
        private List<int> m_Indices = new List<int>();

        // 对象池缓存
        private static readonly ObjectPool<VertexHelper> s_Pool =
            new ObjectPool<VertexHelper>(null, null);

        // ========== 核心方法 ==========

        /// <summary>
        /// 添加一个四边形（2个三角形）
        /// </summary>
        public void AddUIVertexQuad(UIVertex[] verts)
        {
            int startIndex = m_Verts.Count;

            // 添加 4 个顶点
            for (int i = 0; i < 4; i++)
                m_Verts.Add(verts[i]);

            // 添加 2 个三角形（6 个索引）
            m_Indices.Add(startIndex);
            m_Indices.Add(startIndex + 1);
            m_Indices.Add(startIndex + 2);

            m_Indices.Add(startIndex + 2);
            m_Indices.Add(startIndex + 3);
            m_Indices.Add(startIndex);
        }

        /// <summary>
        /// 添加一个三角形流
        /// </summary>
        public void AddUIVertexTriangleStream(List<UIVertex> stream)
        {
            if (stream == null || stream.Count == 0)
                return;

            int startIndex = m_Verts.Count;
            m_Verts.AddRange(stream);

            // 每 3 个顶点构成一个三角形
            for (int i = 0; i < stream.Count; i += 3)
            {
                m_Indices.Add(startIndex + i);
                m_Indices.Add(startIndex + i + 1);
                m_Indices.Add(startIndex + i + 2);
            }
        }

        /// <summary>
        /// 填充到 Mesh
        /// </summary>
        public void FillMesh(Mesh mesh)
        {
            if (mesh == null)
                return;

            mesh.Clear();

            if (m_Verts.Count == 0)
                return;

            // 设置顶点
            mesh.SetVertices(m_Verts);

            // 设置三角形
            mesh.SetTriangles(m_Indices, 0);

            // 重新计算包围盒
            mesh.RecalculateBounds();
        }

        // ========== 工具方法 ==========

        /// <summary>
        /// 获取顶点数量
        /// </summary>
        public int currentVertCount => m_Verts.Count;

        /// <summary>
        /// 获取索引数量
        /// </summary>
        public int currentIndexCount => m_Indices.Count;

        /// <summary>
        /// 清空所有数据
        /// </summary>
        public void Clear()
        {
            m_Verts.Clear();
            m_Indices.Clear();
        }

        // ========== 对象池支持 ==========

        public static VertexHelper Get()
        {
            return s_Pool.Get();
        }

        public void Dispose()
        {
            Clear();
            s_Pool.Release(this);
        }
    }
}

/*
========== 源码要点总结 ==========

1. VertexHelper 封装了 UI 网格构建的常用操作
2. 内部使用 List<UIVertex> 和 List<int> 存储数据
3. AddUIVertexQuad() 是最常用的方法，添加一个四边形
4. 支持对象池，减少 GC 分配
5. FillMesh() 将数据填充到 Mesh 对象

========== UIVertex 结构 ==========

public struct UIVertex
{
    public Vector3 position;    // 顶点位置
    public Vector3 normal;      // 法线
    public Vector4 tangent;     // 切线
    public Color32 color;       // 顶点颜色
    public Vector2 uv0;         // UV0
    public Vector2 uv1;         // UV1
    public Vector2 uv2;         // UV2
    public Vector2 uv3;         // UV3
}
*/
```

### 1.5 批处理系统源码分析

```csharp
// ===== 批处理机制分析 =====
// CanvasRenderer 的批处理逻辑在 C++ 层实现
// 以下是 C# 层可观察到的行为和接口

namespace UnityEngine.UI
{
    /// <summary>
    /// 批处理系统分析（基于 C# 公开接口推断）
    /// </summary>
    public static class BatchingAnalysis
    {
        /*
        ========== 批处理原理 ==========

        UGUI 的批处理在 CanvasRenderer 的 C++ 实现中进行。
        每个 Canvas 独立进行批处理。

        批处理流程：
        1. 收集 Canvas 下所有激活的 CanvasRenderer
        2. 按 hierarchy 顺序排序
        3. 检查相邻元素是否可以合并
        4. 生成 Draw Call

        ========== 合批条件（源码推断） ==========

        两个相邻元素可以合批的条件：

        1. 相同材质 (Material)
           - GetMaterial() 返回相同的材质实例
           - 或者使用相同的 Shader 且属性一致

        2. 相同纹理 (Texture)
           - mainTexture 指向同一纹理
           - 使用 Sprite Atlas 时，多个 Sprite 共享同一图集纹理

        3. 无打断因素
           - 没有 Mask 组件（修改 stencil buffer）
           - 没有 RectMask2D（启用矩形裁剪）
           - 不需要重新计算顶点

        ========== 打断合批的因素 ==========

        1. Mask 组件
           - Mask 修改 stencil buffer
           - 子元素需要额外的 stencil 测试
           - 源码位置: Mask.cs GetModifiedMaterial()

        2. 不同材质实例
           - 即使是同一个 Shader
           - Material.Instantiate() 创建新实例会打断

        3. 动态字体
           - Text 组件使用动态字体
           - 字体纹理可能在不同帧更新
           - 每次更新都可能导致纹理变化

        4. 材质属性修改
           - 修改 color 属性会创建新的材质实例
           - Graphic 的 color 通过 MaterialPropertyBlock 实现
        */

        /// <summary>
        /// 分析 Canvas 的合批情况
        /// </summary>
        public static void AnalyzeBatching(Canvas canvas)
        {
            var renderers = canvas.GetComponentsInChildren<CanvasRenderer>(true);
            var graphics = canvas.GetComponentsInChildren<Graphic>(true);

            Debug.Log($"=== Canvas: {canvas.name} ===");
            Debug.Log($"CanvasRenderer 数量: {renderers.Length}");
            Debug.Log($"Graphic 数量: {graphics.Length}");

            // 分析每个 Graphic 的材质和纹理
            foreach (var graphic in graphics)
            {
                if (!graphic.IsActive()) continue;

                var mat = graphic.material;
                var tex = graphic.mainTexture;

                Debug.Log($"[{graphic.name}] " +
                         $"Material: {mat?.name ?? "null"} " +
                         $"Texture: {tex?.name ?? "null"} " +
                         $"Depth: {graphic.depth}");
            }
        }
    }
}

// ===== Image 组件的 OnPopulateMesh 源码片段 =====
namespace UnityEngine.UI
{
    public partial class Image : Graphic
    {
        /// <summary>
        /// Image 的网格生成 - 源码简化版
        /// </summary>
        protected override void OnPopulateMesh(VertexHelper vh)
        {
            // 清空顶点
            vh.Clear();

            // 获取绘制区域
            var r = GetDrawingDimensions(false);
            var v = new Vector4(r.x, r.y, r.x + r.width, r.y + r.height);

            // 获取 UV 坐标
            var uv = (activeSprite != null)
                ? UnityEngine.Sprites.DataUtility.GetOuterUV(activeSprite)
                : new Vector4(0, 0, 1, 1);

            // 顶点颜色
            var color32 = color;

            // 添加 4 个顶点
            var vert = new UIVertex();
            vert.color = color32;

            // 左下
            vert.position = new Vector3(v.x, v.y);
            vert.uv0 = new Vector2(uv.x, uv.y);
            vh.AddVert(vert);

            // 左上
            vert.position = new Vector3(v.x, v.w);
            vert.uv0 = new Vector2(uv.x, uv.w);
            vh.AddVert(vert);

            // 右上
            vert.position = new Vector3(v.z, v.w);
            vert.uv0 = new Vector2(uv.z, uv.w);
            vh.AddVert(vert);

            // 右下
            vert.position = new Vector3(v.z, v.y);
            vert.uv0 = new Vector2(uv.z, uv.y);
            vh.AddVert(vert);

            // 添加 2 个三角形
            vh.AddTriangle(0, 1, 2);
            vh.AddTriangle(2, 3, 0);
        }
    }
}

/*
========== 批处理优化建议（基于源码分析） ==========

1. 使用 Sprite Atlas
   - 同一 Atlas 下的 Sprite 共享纹理
   - 减少纹理切换，提高合批效率

2. 避免动态修改颜色
   - graphic.color 修改会触发 SetMaterialDirty
   - 考虑使用 CanvasGroup.alpha 替代批量修改

3. 合理使用 Mask
   - Mask 会打断合批
   - 矩形裁剪使用 RectMask2D（不打断合批）

4. 分离动静 UI
   - 频繁变化的 UI 放在独立 Canvas
   - 静态 UI 不需要重建，减少性能开销
*/
```

---

## 2. 事件系统

### 2.1 事件系统架构

```csharp
using UnityEngine;
using UnityEngine.EventSystems;
using System.Collections.Generic;

/// <summary>
/// UGUI事件系统详解
/// </summary>
public class EventSystemGuide : MonoBehaviour
{
    /*
    ========== 事件系统组件 ==========

    EventSystem
    ├── 管理输入事件
    ├── 处理射线检测
    └── 维护当前选中对象

    StandaloneInputModule
    ├── 处理鼠标/触摸输入
    ├── 处理键盘导航
    └── 处理控制器输入

    ========== 射线检测器 ==========

    GraphicRaycaster (Canvas)
    ├── 检测UI元素
    └── 只对Canvas内的Graphic有效

    PhysicsRaycaster (3D)
    ├── 检测3D物体
    └── 需要物体有Collider

    Physics2DRaycaster (2D)
    ├── 检测2D物体
    └── 需要物体有Collider2D
    */

    /// <summary>
    /// 自定义事件监听器
    /// </summary>
    public class CustomUIEventListener :
        MonoBehaviour,
        IPointerClickHandler,
        IPointerDownHandler,
        IPointerUpHandler,
        IPointerEnterHandler,
        IPointerExitHandler,
        IBeginDragHandler,
        IDragHandler,
        IEndDragHandler,
        IScrollHandler,
        ISelectHandler,
        IDeselectHandler
    {
        public System.Action<PointerEventData> OnClickEvent;
        public System.Action<PointerEventData> OnDownEvent;
        public System.Action<PointerEventData> OnUpEvent;
        public System.Action<PointerEventData> OnEnterEvent;
        public System.Action<PointerEventData> OnExitEvent;
        public System.Action<PointerEventData> OnBeginDragEvent;
        public System.Action<PointerEventData> OnDragEvent;
        public System.Action<PointerEventData> OnEndDragEvent;
        public System.Action<PointerEventData> OnScrollEvent;
        public System.Action<BaseEventData> OnSelectEvent;
        public System.Action<BaseEventData> OnDeselectEvent;

        public static CustomUIEventListener Get(GameObject go)
        {
            var listener = go.GetComponent<CustomUIEventListener>();
            if (listener == null)
                listener = go.AddComponent<CustomUIEventListener>();
            return listener;
        }

        public void OnPointerClick(PointerEventData eventData) => OnClickEvent?.Invoke(eventData);
        public void OnPointerDown(PointerEventData eventData) => OnDownEvent?.Invoke(eventData);
        public void OnPointerUp(PointerEventData eventData) => OnUpEvent?.Invoke(eventData);
        public void OnPointerEnter(PointerEventData eventData) => OnEnterEvent?.Invoke(eventData);
        public void OnPointerExit(PointerEventData eventData) => OnExitEvent?.Invoke(eventData);
        public void OnBeginDrag(PointerEventData eventData) => OnBeginDragEvent?.Invoke(eventData);
        public void OnDrag(PointerEventData eventData) => OnDragEvent?.Invoke(eventData);
        public void OnEndDrag(PointerEventData eventData) => OnEndDragEvent?.Invoke(eventData);
        public void OnScroll(PointerEventData eventData) => OnScrollEvent?.Invoke(eventData);
        public void OnSelect(BaseEventData eventData) => OnSelectEvent?.Invoke(eventData);
        public void OnDeselect(BaseEventData eventData) => OnDeselectEvent?.Invoke(eventData);
    }
}
```

### 2.2 射线检测优化

```csharp
using UnityEngine;
using UnityEngine.UI;
using System.Collections.Generic;

/// <summary>
/// 射线检测优化
/// </summary>
public static class RaycastOptimizer
{
    /*
    ========== Raycast Target优化 ==========

    原则：只对需要交互的UI元素开启Raycast Target

    需要开启的：
    - Button
    - Toggle
    - 滑动区域
    - 可点击的Image

    不需要开启的：
    - 纯显示的Image
    - 纯显示的Text
    - 装饰性元素
    - 背景图
    */

    /// <summary>
    /// 批量设置Raycast Target
    /// </summary>
    public static void SetRaycastTargetRecursive(GameObject root, bool enabled)
    {
        var graphics = root.GetComponentsInChildren<Graphic>(true);
        foreach (var graphic in graphics)
        {
            // 只有在有交互组件时才开启
            var hasInteractable = graphic.GetComponent<IPointerClickHandler>() != null ||
                                 graphic.GetComponent<IPointerDownHandler>() != null ||
                                 graphic.GetComponent<IPointerUpHandler>() != null ||
                                 graphic.GetComponent<IDragHandler>() != null;

            graphic.raycastTarget = enabled && hasInteractable;
        }
    }

    /// <summary>
    /// 统计Raycast Target数量
    /// </summary>
    public static int CountRaycastTargets(Canvas canvas)
    {
        int count = 0;
        var graphics = canvas.GetComponentsInChildren<Graphic>(true);

        foreach (var graphic in graphics)
        {
            if (graphic.raycastTarget)
                count++;
        }

        return count;
    }

    /// <summary>
    /// 输出所有Raycast Target
    /// </summary>
    public static void LogRaycastTargets(Canvas canvas)
    {
        var graphics = canvas.GetComponentsInChildren<Graphic>(true);
        int count = 0;

        foreach (var graphic in graphics)
        {
            if (graphic.raycastTarget)
            {
                Debug.Log($"Raycast Target: {graphic.gameObject.name}", graphic.gameObject);
                count++;
            }
        }

        Debug.Log($"Total Raycast Targets: {count}");
    }
}

/// <summary>
/// 自定义GraphicRaycaster优化版
/// </summary>
public class OptimizedGraphicRaycaster : GraphicRaycaster
{
    [Header("Optimization")]
    [SerializeField] private bool ignoreReversedGraphics = true;
    [SerializeField] private bool blockAllOnFirstHit = false;

    // 缓存
    private List<Graphic> m_RaycastResults = new List<Graphic>();

    public override void Raycast(PointerEventData eventData, List<RaycastResult> resultAppendList)
    {
        if (canvas == null) return;

        var eventCamera = eventData.pressEventCamera;
        if (eventCamera == null && canvas.renderMode != RenderMode.ScreenSpaceOverlay)
            return;

        // 获取射线
        Vector2 localPoint;
        if (!GetLocalPoint(eventData, out localPoint)) return;

        // 检测
        m_RaycastResults.Clear();
        Raycast(canvas, eventCamera, localPoint, m_RaycastResults);

        // 转换结果
        foreach (var graphic in m_RaycastResults)
        {
            if (blockAllOnFirstHit && resultAppendList.Count > 0)
                break;

            resultAppendList.Add(new RaycastResult
            {
                gameObject = graphic.gameObject,
                module = this,
                distance = 0,
                index = resultAppendList.Count,
                depth = graphic.depth,
                sortingLayer = canvas.sortingLayerID,
                sortingOrder = canvas.sortingOrder,
                worldPosition = Vector3.zero,
                worldNormal = Vector3.zero
            });
        }
    }

    private bool GetLocalPoint(PointerEventData eventData, out Vector2 localPoint)
    {
        return RectTransformUtility.ScreenPointToLocalPointInRectangle(
            canvas.transform as RectTransform,
            eventData.position,
            eventData.pressEventCamera,
            out localPoint);
    }

    private void Raycast(Canvas canvas, Camera eventCamera, Vector2 pointerPosition, List<Graphic> results)
    {
        var graphics = GraphicRegistry.GetRaycastableGraphicsForCanvas(canvas);

        for (int i = 0; i < graphics.Count; i++)
        {
            var graphic = graphics[i];
            if (graphic.depth == -1 || !graphic.raycastTarget || !graphic.IsActive())
                continue;

            if (!RectTransformUtility.RectangleContainsScreenPoint(
                graphic.rectTransform, pointerPosition, eventCamera))
                continue;

            if (ignoreReversedGraphics && eventCamera != null)
            {
                var dir = graphic.rectTransform.position - eventCamera.transform.position;
                if (Vector3.Dot(dir, eventCamera.transform.forward) <= 0)
                    continue;
            }

            results.Add(graphic);
        }

        results.Sort((g1, g2) => g2.depth.CompareTo(g1.depth));
    }
}
```

### 2.3 事件系统源码分析

#### 2.3.1 EventSystem 核心流程

```csharp
// ===== 源码文件: UnityEngine.EventSystems/EventSystem.cs =====
// EventSystem - 事件系统的核心管理器

namespace UnityEngine.EventSystems
{
    /// <summary>
    /// EventSystem - 管理输入事件、射线检测和事件分发
    /// 源码路径: UnityEngine.EventSystems/EventSystem.cs
    /// </summary>
    public class EventSystem : UIBehaviour
    {
        // ========== 核心成员 ==========

        // 当前选中的游戏对象
        private GameObject m_CurrentSelected;

        // 当前悬停的对象
        private GameObject m_FirstSelected;

        // 射线检测结果
        private List<RaycastResult> m_RaycastResultCache = new List<RaycastResult>();

        // 指针数据
        private List<PointerInputModule> m_PointerInputModules = new List<PointerInputModule>();

        // 当前输入模块
        private BaseInputModule m_CurrentInputModule;

        // ========== 核心属性 ==========

        // 当前 EventSystem 单例（场景中应该只有一个）
        public static EventSystem current { get; set; }

        // 是否有选中对象
        public bool alreadySelecting => m_CurrentSelected != null;

        // ========== 主循环 ==========

        protected virtual void Update()
        {
            // 1. 更新输入模块
            if (m_CurrentInputModule != null)
            {
                m_CurrentInputModule.Process();
            }

            // 2. 处理导航事件（键盘、手柄）
            ProcessNavigationEvents();
        }

        // ========== 射线检测 ==========

        /// <summary>
        /// 执行所有射线检测器的检测
        /// </summary>
        public void RaycastAll(PointerEventData eventData, List<RaycastResult> raycastResults)
        {
            raycastResults.Clear();

            // 获取所有射线检测器
            var modules = RaycasterManager.GetRaycasters();

            foreach (var module in modules)
            {
                // 每个检测器独立检测
                module.Raycast(eventData, raycastResults);
            }

            // 按优先级排序
            raycastResults.Sort((a, b) =>
            {
                // 1. 先按 sortingLayer 排序
                if (a.sortingLayer != b.sortingLayer)
                    return a.sortingLayer.CompareTo(b.sortingLayer);

                // 2. 再按 sortingOrder 排序
                if (a.sortingOrder != b.sortingOrder)
                    return a.sortingOrder.CompareTo(b.sortingOrder);

                // 3. 最后按 depth 排序
                return b.depth.CompareTo(a.depth);
            });
        }

        // ========== 选择管理 ==========

        /// <summary>
        /// 设置当前选中对象
        /// </summary>
        public void SetSelectedGameObject(GameObject selected)
        {
            if (m_CurrentSelected == selected)
                return;

            // 发送取消选中事件
            if (m_CurrentSelected != null)
            {
                ExecuteEvents.Execute(m_CurrentSelected,
                    new BaseEventData(this), ExecuteEvents.deselectHandler);
            }

            m_CurrentSelected = selected;

            // 发送选中事件
            if (m_CurrentSelected != null)
            {
                ExecuteEvents.Execute(m_CurrentSelected,
                    new BaseEventData(this), ExecuteEvents.selectHandler);
            }
        }
    }
}

/*
========== 源码要点总结 ==========

1. EventSystem 是单例模式，场景中应该只有一个
2. Update() 中调用输入模块的 Process() 处理输入
3. RaycastAll() 收集所有射线检测器的结果并排序
4. SetSelectedGameObject() 管理选中状态，触发 select/deselect 事件
5. 射线检测结果排序优先级：sortingLayer > sortingOrder > depth
*/
```

#### 2.3.2 GraphicRaycaster 源码

```csharp
// ===== 源码文件: UnityEngine.UI/Core/GraphicRaycaster.cs =====
// GraphicRaycaster - UI 元素射线检测器

namespace UnityEngine.UI
{
    /// <summary>
    /// GraphicRaycaster - Canvas 专用的射线检测器
    /// 源码路径: UnityEngine.UI/Core/GraphicRaycaster.cs
    /// </summary>
    [RequireComponent(typeof(Canvas))]
    public class GraphicRaycaster : BaseRaycaster
    {
        // ========== 配置项 ==========

        [SerializeField] private bool m_IgnoreReversedGraphics = true;
        [SerializeField] private bool m_BlockingObjects = BlockingObjects.None;

        // ========== 核心属性 ==========

        // 关联的 Canvas
        private Canvas m_Canvas;
        public override Camera eventCamera => m_Canvas.worldCamera;

        // ========== 射线检测实现 ==========

        /// <summary>
        /// 执行射线检测 - BaseRaycaster 的抽象方法实现
        /// </summary>
        public override void Raycast(PointerEventData eventData, List<RaycastResult> resultAppendList)
        {
            if (m_Canvas == null)
                return;

            // 获取检测用的相机
            var eventCamera = eventData.pressEventCamera;
            if (eventCamera == null && m_Canvas.renderMode != RenderMode.ScreenSpaceOverlay)
                return;

            // 1. 将屏幕坐标转换为 Canvas 本地坐标
            Vector2 localPoint;
            if (!RectTransformUtility.ScreenPointToLocalPointInRectangle(
                m_Canvas.transform as RectTransform,
                eventData.position,
                eventCamera,
                out localPoint))
                return;

            // 2. 获取可检测的 Graphic 列表
            var graphics = GraphicRegistry.GetRaycastableGraphicsForCanvas(m_Canvas);

            // 3. 遍历检测每个 Graphic
            for (int i = 0; i < graphics.Count; i++)
            {
                var graphic = graphics[i];

                // 跳过条件检查
                if (graphic.depth == -1)      // 不在任何 Canvas 中
                    continue;
                if (!graphic.raycastTarget)   // 不参与射线检测
                    continue;
                if (!graphic.IsActive())      // 未激活
                    continue;

                // 检查点是否在 Graphic 矩形内
                if (!RectTransformUtility.RectangleContainsScreenPoint(
                    graphic.rectTransform,
                    eventData.position,
                    eventCamera))
                    continue;

                // 检查是否被翻转（背对相机）
                if (m_IgnoreReversedGraphics && eventCamera != null)
                {
                    var dir = graphic.rectTransform.position - eventCamera.transform.position;
                    if (Vector3.Dot(dir, eventCamera.transform.forward) <= 0)
                        continue;
                }

                // 4. 调用 Graphic 自身的射线检测
                if (!graphic.Raycast(eventData.position, eventCamera))
                    continue;

                // 5. 添加到结果列表
                resultAppendList.Add(new RaycastResult
                {
                    gameObject = graphic.gameObject,
                    module = this,
                    distance = 0,  // UI 使用 depth 排序，distance 为 0
                    index = resultAppendList.Count,
                    depth = graphic.depth,
                    sortingLayer = m_Canvas.sortingLayerID,
                    sortingOrder = m_Canvas.sortingOrder,
                    worldPosition = graphic.rectTransform.position,
                    worldNormal = graphic.rectTransform.forward
                });
            }
        }

        // ========== 初始化 ==========

        protected override void Awake()
        {
            base.Awake();
            m_Canvas = GetComponent<Canvas>();
        }
    }
}

// ===== Graphic.Raycast 源码片段 =====
namespace UnityEngine.UI
{
    public abstract partial class Graphic : UIBehaviour
    {
        /// <summary>
        /// Graphic 自身的射线检测
        /// </summary>
        public virtual bool Raycast(Vector2 sp, Camera eventCamera)
        {
            if (!raycastTarget)
                return false;

            // 使用 RectTransformUtility 检测
            return RectTransformUtility.RectangleContainsScreenPoint(
                rectTransform, sp, eventCamera);
        }
    }
}

/*
========== 源码要点总结 ==========

1. GraphicRaycaster 只对关联 Canvas 内的 Graphic 有效
2. 从 GraphicRegistry 获取可检测的 Graphic 列表
3. 检测流程：坐标转换 → 矩形检测 → 背面剔除 → Graphic 自检
4. raycastTarget 属性控制是否参与检测
5. 结果包含 depth 用于排序

========== 性能优化建议 ==========

1. 关闭不需要交互的 Graphic 的 raycastTarget
2. 减少可检测 Graphic 数量可提升性能
3. m_IgnoreReversedGraphics 可跳过背对相机的 UI
*/
```

#### 2.3.3 事件分发机制

```csharp
// ===== 源码文件: UnityEngine.EventSystems/ExecuteEvents.cs =====
// ExecuteEvents - 事件执行器

namespace UnityEngine.EventSystems
{
    /// <summary>
    /// ExecuteEvents - 事件执行和分发的静态工具类
    /// 源码路径: UnityEngine.EventSystems/ExecuteEvents.cs
    /// </summary>
    public static class ExecuteEvents
    {
        // ========== 事件处理器委托 ==========

        // 泛型事件处理器委托
        public delegate void EventFunction<T1>(T1 handler, BaseEventData eventData);

        // ========== 预定义事件处理器 ==========

        // 指针点击
        private static readonly EventFunction<IPointerClickHandler> s_PointerClickHandler =
            (handler, eventData) =>
            {
                handler.OnPointerClick((PointerEventData)eventData);
            };

        // 指针按下
        private static readonly EventFunction<IPointerDownHandler> s_PointerDownHandler =
            (handler, eventData) =>
            {
                handler.OnPointerDown((PointerEventData)eventData);
            };

        // 指针抬起
        private static readonly EventFunction<IPointerUpHandler> s_PointerUpHandler =
            (handler, eventData) =>
            {
                handler.OnPointerUp((PointerEventData)eventData);
            };

        // 拖拽开始
        private static readonly EventFunction<IBeginDragHandler> s_BeginDragHandler =
            (handler, eventData) =>
            {
                handler.OnBeginDrag((PointerEventData)eventData);
            };

        // 拖拽中
        private static readonly EventFunction<IDragHandler> s_DragHandler =
            (handler, eventData) =>
            {
                handler.OnDrag((PointerEventData)eventData);
            };

        // 拖拽结束
        private static readonly EventFunction<IEndDragHandler> s_EndDragHandler =
            (handler, eventData) =>
            {
                handler.OnEndDrag((PointerEventData)eventData);
            };

        // ========== 公开静态属性 ==========

        public static EventFunction<IPointerClickHandler> pointerClickHandler => s_PointerClickHandler;
        public static EventFunction<IPointerDownHandler> pointerDownHandler => s_PointerDownHandler;
        public static EventFunction<IPointerUpHandler> pointerUpHandler => s_PointerUpHandler;
        public static EventFunction<IBeginDragHandler> beginDragHandler => s_BeginDragHandler;
        public static EventFunction<IDragHandler> dragHandler => s_DragHandler;
        public static EventFunction<IEndDragHandler> endDragHandler => s_EndDragHandler;

        // ========== 核心执行方法 ==========

        /// <summary>
        /// 在指定对象上执行事件
        /// </summary>
        public static bool Execute<T>(GameObject target, BaseEventData eventData,
            EventFunction<T> functor) where T : IEventSystemHandler
        {
            if (target == null)
                return false;

            // 获取所有实现了该接口的组件
            var handlers = ListPool<T>.Get();
            target.GetComponents<T>(handlers);

            bool handled = false;

            for (int i = 0; i < handlers.Count; i++)
            {
                try
                {
                    // 调用事件处理器
                    functor(handlers[i], eventData);
                    handled = true;
                }
                catch (Exception e)
                {
                    Debug.LogException(e);
                }
            }

            ListPool<T>.Release(handlers);
            return handled;
        }

        /// <summary>
        /// 向上冒泡执行事件（遍历父级）
        /// </summary>
        public static GameObject ExecuteHierarchy<T>(GameObject root, BaseEventData eventData,
            EventFunction<T> callbackFunction) where T : IEventSystemHandler
        {
            Transform current = root.transform;

            while (current != null)
            {
                // 在当前对象上执行
                if (Execute(current.gameObject, eventData, callbackFunction))
                    return current.gameObject;

                // 向上查找父级
                current = current.parent;
            }

            return null;
        }
    }
}

// ===== 事件接口定义示例 =====
namespace UnityEngine.EventSystems
{
    /// <summary>
    /// 事件处理器基接口
    /// 所有事件接口都继承自 IEventSystemHandler
    /// </summary>
    public interface IEventSystemHandler { }

    public interface IPointerClickHandler : IEventSystemHandler
    {
        void OnPointerClick(PointerEventData eventData);
    }

    public interface IPointerDownHandler : IEventSystemHandler
    {
        void OnPointerDown(PointerEventData eventData);
    }

    public interface IPointerUpHandler : IEventSystemHandler
    {
        void OnPointerUp(PointerEventData eventData);
    }

    public interface IDragHandler : IEventSystemHandler
    {
        void OnDrag(PointerEventData eventData);
    }
}

/*
========== 源码要点总结 ==========

1. ExecuteEvents 使用委托和泛型实现事件分发
2. Execute() 在单个对象上查找所有实现了接口的组件
3. ExecuteHierarchy() 向上冒泡查找处理者
4. 使用 ListPool<T> 对象池减少 GC
5. 所有事件接口都继承自 IEventSystemHandler

========== 事件处理流程 ==========

1. 用户点击屏幕
2. StandaloneInputModule 检测输入
3. GraphicRaycaster 执行射线检测
4. EventSystem 排序结果，确定目标对象
5. ExecuteEvents.Execute() 调用目标对象的接口方法
*/
```

---

## 3. 布局系统

### 3.1 布局组件原理

```csharp
using UnityEngine;
using UnityEngine.UI;

/// <summary>
/// 布局系统详解
/// </summary>
public class LayoutSystemGuide : MonoBehaviour
{
    /*
    ========== 布局组件层级 ==========

    LayoutGroup (基类)
    ├── HorizontalLayoutGroup    水平排列
    ├── VerticalLayoutGroup      垂直排列
    └── GridLayoutGroup          网格排列

    LayoutElement
    └── 覆盖子物体的布局属性

    ContentSizeFitter
    └── 根据子物体调整自身大小

    ScrollRect
    └── 滚动视图容器

    ========== 布局计算流程 ==========

    1. LayoutRebuilder.MarkLayoutForRebuild(rectTransform)
       标记需要重建

    2. LayoutRebuilder.Rebuild()
       执行重建（在Canvas.willRenderCanvases）

    3. 计算顺序：
       - CalcAlongInvisibliAxis (水平/垂直)
       - SetLayoutAlongAxis
       - SetLayoutInput
    */

    /// <summary>
    /// 自定义布局组
    /// </summary>
    public class FlowLayoutGroup : LayoutGroup
    {
        [Header("Flow Layout Settings")]
        [SerializeField] private float spacing = 10f;
        [SerializeField] private float cellWidth = 100f;
        [SerializeField] private float cellHeight = 100f;

        public override void CalculateLayoutInputHorizontal()
        {
            base.CalculateLayoutInputHorizontal();
            CalculateLayout();
        }

        public override void CalculateLayoutInputVertical()
        {
            CalculateLayout();
        }

        public override void SetLayoutHorizontal()
        {
            SetLayout();
        }

        public override void SetLayoutVertical()
        {
            SetLayout();
        }

        private void CalculateLayout()
        {
            int itemCount = rectChildren.Count;
            if (itemCount == 0) return;

            float containerWidth = rectTransform.rect.width - padding.horizontal;
            int cellsPerRow = Mathf.Max(1, Mathf.FloorToInt((containerWidth + spacing) / (cellWidth + spacing)));
            int rows = Mathf.CeilToInt((float)itemCount / cellsPerRow);

            float totalWidth = Mathf.Min(cellsPerRow, itemCount) * cellWidth + (Mathf.Min(cellsPerRow, itemCount) - 1) * spacing;
            float totalHeight = rows * cellHeight + (rows - 1) * spacing;

            SetLayoutInputForAxis(totalWidth + padding.horizontal, totalWidth + padding.horizontal, -1, 0);
            SetLayoutInputForAxis(totalHeight + padding.vertical, totalHeight + padding.vertical, -1, 1);
        }

        private void SetLayout()
        {
            int itemCount = rectChildren.Count;
            if (itemCount == 0) return;

            float containerWidth = rectTransform.rect.width - padding.horizontal;
            int cellsPerRow = Mathf.Max(1, Mathf.FloorToInt((containerWidth + spacing) / (cellWidth + spacing)));

            float startX = padding.left;
            float startY = padding.top;

            for (int i = 0; i < itemCount; i++)
            {
                int row = i / cellsPerRow;
                int col = i % cellsPerRow;

                float x = startX + col * (cellWidth + spacing);
                float y = startY + row * (cellHeight + spacing);

                var child = rectChildren[i];
                SetChildAlongAxis(child, 0, x, cellWidth);
                SetChildAlongAxis(child, 1, y, cellHeight);
            }
        }
    }
}
```

### 3.2 虚拟列表（性能优化）

```csharp
using UnityEngine;
using UnityEngine.UI;
using System.Collections.Generic;

/// <summary>
/// 虚拟列表 - 大数据量优化
/// </summary>
[RequireComponent(typeof(ScrollRect))]
public class VirtualList : MonoBehaviour
{
    [Header("Settings")]
    [SerializeField] private GameObject itemPrefab;
    [SerializeField] private RectTransform content;
    [SerializeField] private float itemHeight = 100f;
    [SerializeField] private float spacing = 10f;
    [SerializeField] private int bufferCount = 2;

    private ScrollRect scrollRect;
    private List<RectTransform> itemPool = new List<RectTransform>();
    private List<object> dataList = new List<object>();

    private int firstVisibleIndex;
    private int lastVisibleIndex;
    private int poolSize;

    public System.Action<int, GameObject, object> OnUpdateItem;

    private void Awake()
    {
        scrollRect = GetComponent<ScrollRect>();
        scrollRect.onValueChanged.AddListener(OnScroll);
    }

    /// <summary>
    /// 设置数据
    /// </summary>
    public void SetData(List<object> data)
    {
        dataList = data;

        // 计算content高度
        float totalHeight = data.Count * itemHeight + (data.Count - 1) * spacing;
        content.sizeDelta = new Vector2(content.sizeDelta.x, totalHeight);

        // 计算需要的池大小
        float viewportHeight = scrollRect.viewport.rect.height;
        poolSize = Mathf.CeilToInt(viewportHeight / (itemHeight + spacing)) + bufferCount * 2;

        // 初始化池
        InitializePool();

        // 初始显示
        UpdateVisibleItems();
    }

    private void InitializePool()
    {
        // 清理旧的
        foreach (var item in itemPool)
        {
            if (item != null)
                Destroy(item.gameObject);
        }
        itemPool.Clear();

        // 创建新的
        for (int i = 0; i < poolSize; i++)
        {
            var go = Instantiate(itemPrefab, content);
            var rect = go.GetComponent<RectTransform>();
            rect.anchorMin = new Vector2(0, 1);
            rect.anchorMax = new Vector2(1, 1);
            rect.pivot = new Vector2(0.5f, 1);
            rect.sizeDelta = new Vector2(0, itemHeight);
            go.SetActive(false);
            itemPool.Add(rect);
        }
    }

    private void OnScroll(Vector2 position)
    {
        UpdateVisibleItems();
    }

    private void UpdateVisibleItems()
    {
        if (dataList.Count == 0) return;

        float contentY = content.anchoredPosition.y;
        float viewportHeight = scrollRect.viewport.rect.height;

        // 计算可见范围
        firstVisibleIndex = Mathf.FloorToInt(contentY / (itemHeight + spacing));
        lastVisibleIndex = Mathf.CeilToInt((contentY + viewportHeight) / (itemHeight + spacing));

        // 添加缓冲
        firstVisibleIndex = Mathf.Max(0, firstVisibleIndex - bufferCount);
        lastVisibleIndex = Mathf.Min(dataList.Count - 1, lastVisibleIndex + bufferCount);

        // 隐藏所有
        foreach (var item in itemPool)
            item.gameObject.SetActive(false);

        // 显示可见的
        int poolIndex = 0;
        for (int i = firstVisibleIndex; i <= lastVisibleIndex && poolIndex < itemPool.Count; i++)
        {
            var item = itemPool[poolIndex];
            float y = -i * (itemHeight + spacing);
            item.anchoredPosition = new Vector2(0, y);
            item.gameObject.SetActive(true);

            // 更新内容
            OnUpdateItem?.Invoke(i, item.gameObject, dataList[i]);

            poolIndex++;
        }
    }

    /// <summary>
    /// 刷新单个项
    /// </summary>
    public void RefreshItem(int index)
    {
        if (index >= firstVisibleIndex && index <= lastVisibleIndex)
        {
            int poolIndex = index - firstVisibleIndex;
            if (poolIndex >= 0 && poolIndex < itemPool.Count)
            {
                OnUpdateItem?.Invoke(index, itemPool[poolIndex].gameObject, dataList[index]);
            }
        }
    }

    /// <summary>
    /// 滚动到指定项
    /// </summary>
    public void ScrollTo(int index)
    {
        float y = index * (itemHeight + spacing);
        content.anchoredPosition = new Vector2(content.anchoredPosition.x, y);
    }

    private void OnDestroy()
    {
        if (scrollRect != null)
            scrollRect.onValueChanged.RemoveListener(OnScroll);
    }
}

// 使用示例
public class VirtualListExample : MonoBehaviour
{
    [SerializeField] private VirtualList virtualList;

    private void Start()
    {
        // 创建测试数据
        var data = new List<object>();
        for (int i = 0; i < 1000; i++)
        {
            data.Add(new ItemData { Index = i, Name = $"Item {i}" });
        }

        virtualList.OnUpdateItem = OnUpdateItem;
        virtualList.SetData(data);
    }

    private void OnUpdateItem(int index, GameObject item, object data)
    {
        var itemData = (ItemData)data;
        var text = item.GetComponentInChildren<Text>();
        if (text != null)
            text.text = itemData.Name;
    }

    private class ItemData
    {
        public int Index;
        public string Name;
    }
}
```

### 3.3 布局系统源码分析

#### 3.3.1 LayoutGroup 基类源码

```csharp
// ===== 源码文件: UnityEngine.UI/Layout/LayoutGroup.cs =====
// LayoutGroup - 布局组件的抽象基类

namespace UnityEngine.UI
{
    /// <summary>
    /// LayoutGroup - 所有布局组的抽象基类
    /// 源码路径: UnityEngine.UI/Layout/LayoutGroup.cs
    /// </summary>
    public abstract class LayoutGroup : UIBehaviour, ILayoutElement, ILayoutGroup
    {
        // ========== 核心成员 ==========

        [SerializeField] protected RectOffset m_Padding = new RectOffset();

        // 子物体列表（只包含参与布局的子物体）
        [NonSerialized] protected List<RectTransform> m_RectChildren = new List<RectTransform>();

        // 布局属性
        protected float m_TotalMinWidth;
        protected float m_TotalMinHeight;
        protected float m_TotalPreferredWidth;
        protected float m_TotalPreferredHeight;

        // ========== 核心属性 ==========

        public RectOffset padding
        {
            get => m_Padding;
            set
            {
                m_Padding = value;
                SetDirty();
            }
        }

        // 只读的子物体列表
        public List<RectTransform> rectChildren => m_RectChildren;

        // ========== ILayoutElement 接口实现 ==========

        public virtual float minWidth => m_TotalMinWidth;
        public virtual float minHeight => m_TotalMinHeight;
        public virtual float preferredWidth => m_TotalPreferredWidth;
        public virtual float preferredHeight => m_TotalPreferredHeight;
        public virtual float flexibleWidth => -1;
        public virtual float flexibleHeight => -1;
        public virtual int layoutPriority => 0;

        // ========== 布局计算接口 ==========

        /// <summary>
        /// 计算水平方向布局
        /// </summary>
        public abstract void CalculateLayoutInputHorizontal();

        /// <summary>
        /// 计算垂直方向布局
        /// </summary>
        public abstract void CalculateLayoutInputVertical();

        /// <summary>
        /// 应用水平方向布局
        /// </summary>
        public abstract void SetLayoutHorizontal();

        /// <summary>
        /// 应用垂直方向布局
        /// </summary>
        public abstract void SetLayoutVertical();

        // ========== 子物体管理 ==========

        /// <summary>
        /// 更新子物体列表
        /// </summary>
        protected virtual void UpdateChildren()
        {
            m_RectChildren.Clear();

            for (int i = 0; i < rectTransform.childCount; i++)
            {
                var child = rectTransform.GetChild(i) as RectTransform;
                if (child == null)
                    continue;

                // 检查是否被 ILayoutIgnorer 忽略
                var ignorer = child.GetComponent<ILayoutIgnorer>();
                if (ignorer != null && ignorer.ignoreLayout)
                    continue;

                m_RectChildren.Add(child);
            }
        }

        // ========== 布局重建 ==========

        protected override void OnEnable()
        {
            base.OnEnable();
            SetDirty();
        }

        protected override void OnDisable()
        {
            SetDirty();
            base.OnDisable();
        }

        /// <summary>
        /// 标记布局需要重建
        /// </summary>
        protected void SetDirty()
        {
            if (!IsActive())
                return;

            LayoutRebuilder.MarkLayoutForRebuild(rectTransform);
        }

        // ========== 辅助方法 ==========

        /// <summary>
        /// 设置子物体在轴上的位置和大小
        /// </summary>
        protected void SetChildAlongAxis(RectTransform rect, int axis, float pos, float size)
        {
            if (rect == null)
                return;

            // 设置锚点为左上角（axis=0 水平，axis=1 垂直）
            if (axis == 0)
            {
                rect.anchorMin = new Vector2(0, rect.anchorMin.y);
                rect.anchorMax = new Vector2(0, rect.anchorMax.y);
                rect.SetInsetAndSizeFromParentEdge(RectTransform.Edge.Left, pos + m_Padding.left, size);
            }
            else
            {
                rect.anchorMin = new Vector2(rect.anchorMin.x, 1);
                rect.anchorMax = new Vector2(rect.anchorMax.x, 1);
                rect.SetInsetAndSizeFromParentEdge(RectTransform.Edge.Top, pos + m_Padding.top, size);
            }
        }
    }
}

// ===== HorizontalOrVerticalLayoutGroup 源码片段 =====
namespace UnityEngine.UI
{
    /// <summary>
    /// 水平/垂直布局组的共享实现
    /// </summary>
    public abstract class HorizontalOrVerticalLayoutGroup : LayoutGroup
    {
        [SerializeField] protected float m_Spacing = 0;
        [SerializeField] protected bool m_ChildForceExpandWidth = true;
        [SerializeField] protected bool m_ChildForceExpandHeight = true;
        [SerializeField] protected bool m_ChildControlWidth = true;
        [SerializeField] protected bool m_ChildControlHeight = true;

        /// <summary>
        /// 计算子元素的总大小
        /// </summary>
        protected void CalcAlongAxis(int axis)
        {
            float combinedPadding = (axis == 0 ? m_Padding.horizontal : m_Padding.vertical);
            float totalMin = combinedPadding;
            float totalPreferred = combinedPadding;
            float totalFlexible = 0;

            bool alongOtherAxis = (axis == 1 ^ IsReverse);

            for (int i = 0; i < m_RectChildren.Count; i++)
            {
                var child = m_RectChildren[i];
                float min, preferred, flexible;

                // 获取子元素的布局属性
                GetChildSizes(child, axis, alongOtherAxis, out min, out preferred, out flexible);

                if (alongOtherAxis)
                {
                    // 垂直方向取最大值
                    totalMin = Mathf.Max(min + combinedPadding, totalMin);
                    totalPreferred = Mathf.Max(preferred + combinedPadding, totalPreferred);
                    totalFlexible = Mathf.Max(flexible, totalFlexible);
                }
                else
                {
                    // 水平方向累加
                    totalMin += min + m_Spacing;
                    totalPreferred += preferred + m_Spacing;
                    totalFlexible += flexible;
                }
            }

            if (axis == 0)
            {
                m_TotalMinWidth = totalMin;
                m_TotalPreferredWidth = totalPreferred;
            }
            else
            {
                m_TotalMinHeight = totalMin;
                m_TotalPreferredHeight = totalPreferred;
            }
        }

        private void GetChildSizes(RectTransform child, int axis, bool controlSize,
            out float min, out float preferred, out float flexible)
        {
            if (!controlSize)
            {
                min = child.rect.size[axis];
                preferred = child.rect.size[axis];
                flexible = 0;
            }
            else
            {
                min = LayoutUtility.GetMinSize(child, axis);
                preferred = LayoutUtility.GetPreferredSize(child, axis);
                flexible = LayoutUtility.GetFlexibleSize(child, axis);
            }
        }
    }
}

/*
========== 源码要点总结 ==========

1. LayoutGroup 实现 ILayoutElement 和 ILayoutGroup 接口
2. m_RectChildren 只包含参与布局的子物体（排除 ILayoutIgnorer）
3. 计算分两个阶段：CalculateLayoutInput* 和 SetLayout*
4. SetDirty() 触发 LayoutRebuilder.MarkLayoutForRebuild
5. SetChildAlongAxis() 设置子物体的锚点和位置

========== 布局计算顺序 ==========

1. UpdateChildren() - 更新子物体列表
2. CalcAlongAxis() - 计算总大小
3. SetLayoutInputForAxis() - 设置布局属性
4. SetChildAlongAxis() - 设置子物体位置
*/
```

#### 3.3.2 LayoutRebuilder 源码

```csharp
// ===== 源码文件: UnityEngine.UI/Layout/LayoutRebuilder.cs =====
// LayoutRebuilder - 布局重建器

namespace UnityEngine.UI
{
    /// <summary>
    /// LayoutRebuilder - 延迟执行布局重建
    /// 源码路径: UnityEngine.UI/Layout/LayoutRebuilder.cs
    /// </summary>
    public class LayoutRebuilder : ICanvasElement
    {
        // ========== 静态成员 ==========

        // 重建队列
        private static readonly IndexedSet<LayoutRebuilder> s_RebuildQueue =
            new IndexedSet<LayoutRebuilder>();

        // 对象池
        private static readonly ObjectPool<LayoutRebuilder> s_LayoutRebuilderPool =
            new ObjectPool<LayoutRebuilder>(() => new LayoutRebuilder(), null, x => x.Clear());

        // ========== 实例成员 ==========

        // 关联的 RectTransform
        private RectTransform m_ToRebuild;

        // ========== 核心静态方法 ==========

        /// <summary>
        /// 标记布局需要重建
        /// </summary>
        public static void MarkLayoutForRebuild(RectTransform rect)
        {
            if (rect == null || rect.gameObject.activeInHierarchy)
                return;

            // 查找需要重建的根节点
            var layoutRoot = rect;
            while (true)
            {
                var parent = layoutRoot.parent as RectTransform;
                if (parent == null)
                    break;

                // 检查父级是否有布局组件
                if (parent.GetComponent<ILayoutGroup>() != null ||
                    parent.GetComponent<ILayoutSelfController>() != null)
                {
                    layoutRoot = parent;
                }
                else
                {
                    break;
                }
            }

            // 检查是否已存在
            if (s_RebuildQueue.Count > 0)
            {
                foreach (var element in s_RebuildQueue)
                {
                    if (element.m_ToRebuild == layoutRoot)
                        return;  // 已存在，不需要重复添加
                }
            }

            // 从对象池获取并添加到队列
            var rebuilder = s_LayoutRebuilderPool.Get();
            rebuilder.m_ToRebuild = layoutRoot;
            s_RebuildQueue.AddUnique(rebuilder);

            // 注册到 CanvasUpdateRegistry
            CanvasUpdateRegistry.RegisterCanvasElementForLayoutRebuild(rebuilder);
        }

        // ========== ICanvasElement 接口实现 ==========

        /// <summary>
        /// 重建 - 由 CanvasUpdateRegistry 调用
        /// </summary>
        public void Rebuild(CanvasUpdate executing)
        {
            switch (executing)
            {
                case CanvasUpdate.Layout:
                    // 执行布局
                    PerformLayoutCalculation(m_ToRebuild);
                    PerformLayoutControl(m_ToRebuild);
                    break;
            }
        }

        /// <summary>
        /// 执行布局计算
        /// </summary>
        private void PerformLayoutCalculation(RectTransform rect)
        {
            // 先递归处理子级
            for (int i = 0; i < rect.childCount; i++)
            {
                var child = rect.GetChild(i) as RectTransform;
                if (child == null || !child.gameObject.activeInHierarchy)
                    continue;

                PerformLayoutCalculation(child);
            }

            // 计算当前层级的布局
            var ignoredIgnorers = ListPool<ILayoutIgnorer>.Get();
            var components = ListPool<Behaviour>.Get();

            rect.GetComponents(components);

            foreach (var comp in components)
            {
                if (comp is ILayoutElement layoutElement && comp.enabled)
                {
                    layoutElement.CalculateLayoutInputHorizontal();
                    layoutElement.CalculateLayoutInputVertical();
                }
            }

            ListPool<ILayoutIgnorer>.Release(ignoredIgnorers);
            ListPool<Behaviour>.Release(components);
        }

        /// <summary>
        /// 执行布局控制
        /// </summary>
        private void PerformLayoutControl(RectTransform rect)
        {
            var components = ListPool<Behaviour>.Get();
            rect.GetComponents(components);

            foreach (var comp in components)
            {
                if (comp is ILayoutController controller && comp.enabled)
                {
                    controller.SetLayoutHorizontal();
                    controller.SetLayoutVertical();
                }
            }

            ListPool<Behaviour>.Release(components);
        }

        // ========== 清理和回收 ==========

        private void Clear()
        {
            m_ToRebuild = null;
        }

        public void LayoutComplete()
        {
            s_LayoutRebuilderPool.Release(this);
        }
    }
}

/*
========== 源码要点总结 ==========

1. LayoutRebuilder 使用对象池减少 GC
2. MarkLayoutForRebuild() 会向上查找布局根节点
3. 使用 IndexedSet 保证元素唯一性
4. Rebuild() 分两步：PerformLayoutCalculation 和 PerformLayoutControl
5. 递归处理子级，确保子级先计算

========== 布局重建流程 ==========

1. 布局组件调用 SetDirty()
2. LayoutRebuilder.MarkLayoutForRebuild() 注册到队列
3. CanvasUpdateRegistry 在 Canvas.willRenderCanvases 时触发
4. LayoutRebuilder.Rebuild() 执行实际布局计算
5. 计算完成后回收到对象池
*/
```

#### 3.3.3 ContentSizeFitter 源码

```csharp
// ===== 源码文件: UnityEngine.UI/Layout/ContentSizeFitter.cs =====
// ContentSizeFitter - 根据子物体调整自身大小

namespace UnityEngine.UI
{
    /// <summary>
    /// ContentSizeFitter - 内容自适应尺寸控制器
    /// 源码路径: UnityEngine.UI/Layout/ContentSizeFitter.cs
    /// </summary>
    public class ContentSizeFitter : UIBehaviour, ILayoutSelfController
    {
        // ========== 配置枚举 ==========

        public enum FitMode
        {
            Unconstrained,  // 不约束，使用原始大小
            MinSize,        // 使用最小尺寸
            PreferredSize   // 使用首选尺寸（最常用）
        }

        // ========== 配置项 ==========

        [SerializeField] private FitMode m_HorizontalFit = FitMode.Unconstrained;
        [SerializeField] private FitMode m_VerticalFit = FitMode.Unconstrained;

        // ========== 属性 ==========

        public FitMode horizontalFit
        {
            get => m_HorizontalFit;
            set { m_HorizontalFit = value; SetDirty(); }
        }

        public FitMode verticalFit
        {
            get => m_VerticalFit;
            set { m_VerticalFit = value; SetDirty(); }
        }

        // ========== ILayoutController 接口实现 ==========

        /// <summary>
        /// 设置水平布局
        /// </summary>
        public void SetLayoutHorizontal()
        {
            if (m_HorizontalFit == FitMode.Unconstrained)
                return;

            // 获取子元素的首选尺寸
            var rect = rectTransform.rect;
            float size = m_HorizontalFit == FitMode.MinSize
                ? LayoutUtility.GetMinSize(rectTransform, 0)
                : LayoutUtility.GetPreferredSize(rectTransform, 0);

            // 设置宽度
            rectTransform.SetSizeWithCurrentAnchors(RectTransform.Axis.Horizontal, size);
        }

        /// <summary>
        /// 设置垂直布局
        /// </summary>
        public void SetLayoutVertical()
        {
            if (m_VerticalFit == FitMode.Unconstrained)
                return;

            // 获取子元素的首选尺寸
            float size = m_VerticalFit == FitMode.MinSize
                ? LayoutUtility.GetMinSize(rectTransform, 1)
                : LayoutUtility.GetPreferredSize(rectTransform, 1);

            // 设置高度
            rectTransform.SetSizeWithCurrentAnchors(RectTransform.Axis.Vertical, size);
        }

        // ========== 脏标记 ==========

        protected override void OnRectTransformDimensionsChange()
        {
            base.OnRectTransformDimensionsChange();
            SetDirty();
        }

        protected void SetDirty()
        {
            if (!IsActive())
                return;

            LayoutRebuilder.MarkLayoutForRebuild(rectTransform);
        }
    }
}

// ===== LayoutUtility 辅助方法 =====
namespace UnityEngine.UI
{
    /// <summary>
    /// LayoutUtility - 布局计算工具类
    /// </summary>
    public static class LayoutUtility
    {
        /// <summary>
        /// 获取最小尺寸
        /// </summary>
        public static float GetMinSize(RectTransform rect, int axis)
        {
            if (axis == 0)
                return GetMinWidth(rect);
            return GetMinHeight(rect);
        }

        /// <summary>
        /// 获取首选尺寸
        /// </summary>
        public static float GetPreferredSize(RectTransform rect, int axis)
        {
            if (axis == 0)
                return GetPreferredWidth(rect);
            return GetPreferredHeight(rect);
        }

        /// <summary>
        /// 获取首选宽度
        /// </summary>
        public static float GetPreferredWidth(RectTransform rect)
        {
            float max = 0;
            var components = ListPool<ILayoutElement>.Get();
            rect.GetComponents(components);

            foreach (var elem in components)
            {
                if (elem.enabled)
                    max = Mathf.Max(max, elem.preferredWidth);
            }

            ListPool<ILayoutElement>.Release(components);
            return max;
        }

        /// <summary>
        /// 获取首选高度
        /// </summary>
        public static float GetPreferredHeight(RectTransform rect)
        {
            float max = 0;
            var components = ListPool<ILayoutElement>.Get();
            rect.GetComponents(components);

            foreach (var elem in components)
            {
                if (elem.enabled)
                    max = Mathf.Max(max, elem.preferredHeight);
            }

            ListPool<ILayoutElement>.Release(components);
            return max;
        }
    }
}

/*
========== 源码要点总结 ==========

1. ContentSizeFitter 实现 ILayoutSelfController 接口
2. 三种 FitMode：Unconstrained、MinSize、PreferredSize
3. 使用 LayoutUtility.GetPreferredSize 获取子元素尺寸
4. SetSizeWithCurrentAnchors 设置 RectTransform 大小
5. 修改配置后调用 SetDirty() 触发重建

========== 使用场景 ==========

- 文本框根据文字内容自动调整大小
- 容器根据子元素自动调整大小
- 配合 LayoutGroup 实现自适应布局
*/
```

---

## 4. 遮罩与裁剪

```csharp
using UnityEngine;
using UnityEngine.UI;

/// <summary>
/// 遮罩系统详解
/// </summary>
public class MaskSystemGuide : MonoBehaviour
{
    /*
    ========== 遮罩类型 ==========

    1. Mask (Stencil Buffer)
       - 使用模板缓冲区
       - 支持任意形状
       - 打断合批
       - 性能开销较大

    2. RectMask2D (Clipping)
       - 只支持矩形
       - 不打断合批
       - 性能好
       - 适合列表项裁剪

    ========== 选择建议 ==========

    - 矩形裁剪 → RectMask2D
    - 圆形/不规则 → Mask
    - 列表滚动 → RectMask2D
    - 头像圆角 → Mask (或Shader)
    */

    /// <summary>
    /// 圆形遮罩（Shader实现，不打断合批）
    /// </summary>
    public class CircleMask : MonoBehaviour, IMaterialModifier
    {
        [SerializeField] private float radius = 50f;
        [SerializeField] private Vector2 center = new Vector2(0.5f, 0.5f);
        [SerializeField] private bool softEdge = true;
        [SerializeField] private float softness = 5f;

        private Material maskMaterial;
        private static Shader maskShader;

        public Material GetModifiedMaterial(Material baseMaterial)
        {
            if (maskShader == null)
                maskShader = Shader.Find("UI/CircleMask");

            if (maskMaterial == null)
                maskMaterial = new Material(maskShader);

            maskMaterial.SetFloat("_Radius", radius);
            maskMaterial.SetVector("_Center", center);
            maskMaterial.SetFloat("_SoftEdge", softEdge ? 1f : 0f);
            maskMaterial.SetFloat("_Softness", softness);

            return maskMaterial;
        }
    }
}
```

### 4.2 遮罩系统源码分析

#### 4.2.1 Mask (Stencil) 源码实现

```csharp
// ===== 源码文件: UnityEngine.UI/Core/Mask.cs =====
// Mask - 使用模板缓冲区实现遮罩

namespace UnityEngine.UI
{
    /// <summary>
    /// Mask - 使用 Stencil Buffer 实现遮罩效果
    /// 源码路径: UnityEngine.UI/Core/Mask.cs
    /// </summary>
    public class Mask : UIBehaviour, ICanvasRaycastFilter, IMaterialModifier
    {
        // ========== 核心成员 ==========

        [SerializeField] private bool m_ShowMaskGraphic = true;

        // Stencil 相关
        private int m_StencilDepth;

        // 材质缓存
        private Material m_MaskMaterial;
        private Material m_UnmaskMaterial;

        // ========== 属性 ==========

        public bool showMaskGraphic
        {
            get => m_ShowMaskGraphic;
            set { m_ShowMaskGraphic = value; SetDirty(); }
        }

        // ========== IMaterialModifier 接口实现 ==========

        /// <summary>
        /// 修改材质 - 核心：添加 Stencil 操作
        /// </summary>
        public virtual Material GetModifiedMaterial(Material baseMaterial)
        {
            if (!MaskEnabled())
                return baseMaterial;

            // 1. 获取 stencil 深度
            var stencilDepth = MaskUtilities.GetStencilDepth(transform, FindRootSortOverrideCanvas());

            if (stencilDepth >= 8)
            {
                Debug.LogWarning("Mask depth too deep! Maximum 8 nested masks allowed.");
                return baseMaterial;
            }

            m_StencilDepth = stencilDepth;

            // 2. 计算 stencil 值
            // 每层 Mask 使用不同的 stencil 值
            int desiredStencilBit = 1 << stencilDepth;
            int stencilID = desiredStencilBit | (desiredStencilBit - 1);

            // 3. 创建遮罩材质
            // 遮罩本身：写入 stencil
            var maskMaterial = StencilMaterial.Add(
                baseMaterial,
                stencilID,                // stencil 值
                StencilOp.Replace,        // 操作：替换
                CompareFunction.Equal,    // 比较：相等
                ColorWriteMask.All,       // 颜色写入
                desiredStencilBit         // stencil 写入掩码
            );

            // 4. 创建反遮罩材质
            // 子元素：测试 stencil
            var unmaskMaterial = StencilMaterial.Add(
                baseMaterial,
                stencilID,               // stencil 值
                StencilOp.Keep,          // 操作：保持
                CompareFunction.Equal,   // 比较：相等
                ColorWriteMask.All,      // 颜色写入
                0                        // 不写入 stencil
            );

            m_MaskMaterial = maskMaterial;
            m_UnmaskMaterial = unmaskMaterial;

            // 5. 如果不显示遮罩图形，使用反遮罩材质
            if (!m_ShowMaskGraphic)
                return m_UnmaskMaterial;

            return m_MaskMaterial;
        }

        // ========== 辅助方法 ==========

        private bool MaskEnabled()
        {
            return IsActive() && graphic != null;
        }

        private Graphic graphic => GetComponent<Graphic>();

        private Canvas FindRootSortOverrideCanvas()
        {
            Canvas canvas = null;
            var list = ListPool<Canvas>.Get();
            gameObject.GetComponentsInParent(false, list);

            foreach (var c in list)
            {
                if (c.overrideSorting)
                {
                    canvas = c;
                    break;
                }
            }

            ListPool<Canvas>.Release(list);
            return canvas;
        }

        // ========== 生命周期 ==========

        protected override void OnEnable()
        {
            base.OnEnable();

            // 通知子元素更新材质
            MaskUtilities.NotifyStencilStateChanged(this);

            SetDirty();
        }

        protected override void OnDisable()
        {
            // 释放材质
            StencilMaterial.Remove(m_MaskMaterial);
            StencilMaterial.Remove(m_UnmaskMaterial);

            m_MaskMaterial = null;
            m_UnmaskMaterial = null;

            // 通知子元素更新材质
            MaskUtilities.NotifyStencilStateChanged(this);

            base.OnDisable();
        }

        private void SetDirty()
        {
            if (graphic != null)
                graphic.SetMaterialDirty();
        }
    }
}

// ===== MaskUtilities 辅助类 =====
namespace UnityEngine.UI
{
    /// <summary>
    /// MaskUtilities - 遮罩工具类
    /// </summary>
    public static class MaskUtilities
    {
        /// <summary>
        /// 获取 Stencil 深度
        /// </summary>
        public static int GetStencilDepth(Transform transform, Canvas stopAfterCanvas)
        {
            int depth = 0;
            Transform current = transform;

            while (current != null)
            {
                var canvas = current.GetComponent<Canvas>();
                if (canvas != null && canvas.overrideSorting)
                    break;

                if (current != transform)
                {
                    var mask = current.GetComponent<Mask>();
                    if (mask != null && mask.MaskEnabled())
                        depth++;
                }

                if (current.parent == null)
                    break;

                current = current.parent;

                if (stopAfterCanvas != null && current == stopAfterCanvas.transform)
                    break;
            }

            return depth;
        }

        /// <summary>
        /// 通知子元素 Stencil 状态改变
        /// </summary>
        public static void NotifyStencilStateChanged(Mask mask)
        {
            var graphics = mask.transform.GetComponentsInChildren<Graphic>(true);
            foreach (var graphic in graphics)
            {
                if (graphic.gameObject == mask.gameObject)
                    continue;

                graphic.SetMaterialDirty();
            }
        }
    }
}

/*
========== 源码要点总结 ==========

1. Mask 使用 Stencil Buffer 实现遮罩
2. GetModifiedMaterial() 是核心，修改材质的 Stencil 操作
3. 支持嵌套，最多 8 层（Stencil Buffer 8 位）
4. 遮罩材质写入 Stencil，子元素材质测试 Stencil
5. showMaskGraphic 控制是否显示遮罩图形本身

========== Stencil Buffer 原理 ==========

1. GPU 的 Stencil Buffer 是一个每像素的整数缓冲区
2. 渲染时可以测试和修改 Stencil 值
3. Mask 写入 Stencil 值，子元素只有通过测试才渲染
4. 嵌套 Mask 使用不同的 Stencil 位

========== 为什么 Mask 打断合批 ==========

1. 每层 Mask 需要不同的 Stencil 操作
2. 无法将不同 Stencil 设置的元素合并到一个 DrawCall
3. 材质被修改（GetModifiedMaterial）导致无法合批
*/
```

#### 4.2.2 RectMask2D 裁剪机制

```csharp
// ===== 源码文件: UnityEngine.UI/Core/RectMask2D.cs =====
// RectMask2D - 矩形裁剪（不打断合批）

namespace UnityEngine.UI
{
    /// <summary>
    /// RectMask2D - 矩形裁剪组件
    /// 源码路径: UnityEngine.UI/Core/RectMask2D.cs
    /// 特点：不打断合批，性能优于 Mask
    /// </summary>
    public class RectMask2D : UIBehaviour, IClipper, IMaterialModifier
    {
        // ========== 核心成员 ==========

        [SerializeField] private Vector4 m_Padding = Vector4.zero;
        [SerializeField] private Vector2 m_Softness = Vector2.zero;

        // 裁剪区域缓存
        private Rect m_LastClipRectCanvasSpace;
        private bool m_ShouldRecalculateClipRects = true;

        // ========== 属性 ==========

        public Vector2 softness
        {
            get => m_Softness;
            set { m_Softness = value; SetDirty(); }
        }

        // ========== IClipper 接口实现 ==========

        /// <summary>
        /// 执行裁剪 - 由 ClipperRegistry 调用
        /// </summary>
        public void PerformClipping()
        {
            if (!IsActive())
                return;

            // 1. 计算裁剪区域
            if (m_ShouldRecalculateClipRects)
            {
                m_ShouldRecalculateClipRects = false;
                m_LastClipRectCanvasSpace = CalculateClipRect();
            }

            // 2. 获取需要裁剪的子元素
            var canvasRect = GetCanvasRect();

            // 3. 启用 CanvasRenderer 的矩形裁剪
            // 注意：这是 C++ 实现，C# 层只是调用
            foreach (var graphic in GetClippableGraphics())
            {
                if (graphic.canvasRenderer != null)
                {
                    // 设置裁剪矩形到 CanvasRenderer
                    // 这会启用 GPU 层面的矩形裁剪
                    graphic.canvasRenderer.EnableRectClipping(canvasRect);
                }
            }
        }

        // ========== IMaterialModifier 接口实现 ==========

        /// <summary>
        /// 修改材质 - RectMask2D 不修改材质！
        /// </summary>
        public Material GetModifiedMaterial(Material baseMaterial)
        {
            // 重要：RectMask2D 返回原始材质
            // 这就是为什么它不打断合批！
            return baseMaterial;
        }

        // ========== 辅助方法 ==========

        private Rect CalculateClipRect()
        {
            var canvas = GetComponentInParent<Canvas>();
            if (canvas == null)
                return new Rect();

            // 计算在 Canvas 空间中的裁剪矩形
            var rectTransform = this.rectTransform;
            var corners = new Vector3[4];
            rectTransform.GetWorldCorners(corners);

            // 转换到 Canvas 空间
            var canvasTransform = canvas.transform as RectTransform;
            for (int i = 0; i < 4; i++)
            {
                corners[i] = canvasTransform.InverseTransformPoint(corners[i]);
            }

            // 计算包围盒
            float minX = corners[0].x;
            float minY = corners[0].y;
            float maxX = corners[0].x;
            float maxY = corners[0].y;

            for (int i = 1; i < 4; i++)
            {
                minX = Mathf.Min(minX, corners[i].x);
                minY = Mathf.Min(minY, corners[i].y);
                maxX = Mathf.Max(maxX, corners[i].x);
                maxY = Mathf.Max(maxY, corners[i].y);
            }

            // 应用 padding
            return new Rect(
                minX + m_Padding.x,
                minY + m_Padding.y,
                maxX - minX - m_Padding.x - m_Padding.z,
                maxY - minY - m_Padding.y - m_Padding.w
            );
        }

        private List<Graphic> GetClippableGraphics()
        {
            var results = new List<Graphic>();
            var graphics = GetComponentsInChildren<Graphic>(true);

            foreach (var graphic in graphics)
            {
                if (graphic.IsActive() && graphic.rectTransform.parent == transform)
                {
                    results.Add(graphic);
                }
            }

            return results;
        }

        private Rect GetCanvasRect()
        {
            var canvas = GetComponentInParent<Canvas>();
            if (canvas == null)
                return new Rect();

            return RectTransformUtility.CalculateRelativeRectTransformBounds(
                canvas.transform, transform).ToRect();
        }

        // ========== 生命周期 ==========

        protected override void OnEnable()
        {
            base.OnEnable();

            // 注册到裁剪器注册表
            ClipperRegistry.Register(this);

            SetDirty();
        }

        protected override void OnDisable()
        {
            // 从裁剪器注册表注销
            ClipperRegistry.Unregister(this);

            // 清理 CanvasRenderer 的裁剪
            foreach (var graphic in GetClippableGraphics())
            {
                if (graphic.canvasRenderer != null)
                {
                    graphic.canvasRenderer.DisableRectClipping();
                }
            }

            base.OnDisable();
        }

        private void SetDirty()
        {
            m_ShouldRecalculateClipRects = true;
        }
    }
}

// ===== ClipperRegistry 裁剪器注册表 =====
namespace UnityEngine.UI
{
    /// <summary>
    /// ClipperRegistry - 管理所有裁剪器
    /// </summary>
    public class ClipperRegistry
    {
        private static ClipperRegistry s_Instance;
        public static ClipperRegistry instance
        {
            get
            {
                if (s_Instance == null)
                    s_Instance = new ClipperRegistry();
                return s_Instance;
            }
        }

        private readonly List<IClipper> m_Clippers = new List<IClipper>();

        public static void Register(IClipper clipper)
        {
            if (clipper == null)
                return;
            instance.m_Clippers.AddUnique(clipper);
        }

        public static void Unregister(IClipper clipper)
        {
            if (clipper == null)
                return;
            instance.m_Clippers.Remove(clipper);
        }

        /// <summary>
        /// 执行所有裁剪 - 由 CanvasUpdateRegistry 调用
        /// </summary>
        public void Cull()
        {
            foreach (var clipper in m_Clippers)
            {
                clipper.PerformClipping();
            }
        }
    }
}

/*
========== 源码要点总结 ==========

1. RectMask2D 不修改材质（GetModifiedMaterial 返回原材质）
2. 使用 CanvasRenderer.EnableRectClipping() 启用 GPU 裁剪
3. 裁剪在 GPU 层面执行，不影响合批
4. 只支持矩形裁剪，不支持任意形状
5. 性能优于 Mask，适合列表滚动等场景

========== RectMask2D vs Mask 对比 ==========

| 特性          | RectMask2D          | Mask               |
|--------------|---------------------|---------------------|
| 裁剪形状      | 矩形                | 任意（Graphic 形状）|
| 打断合批      | 否                  | 是                  |
| 实现方式      | GPU 矩形裁剪        | Stencil Buffer      |
| 嵌套支持      | 有限                | 支持（最多 8 层）   |
| 性能          | 高                  | 中                  |
| 适用场景      | 列表滚动            | 圆形头像、不规则遮罩|

========== 为什么 RectMask2D 不打断合批 ==========

1. GetModifiedMaterial() 返回原始材质
2. 裁剪通过 CanvasRenderer.EnableRectClipping() 实现
3. 这是 GPU 层面的操作，不影响材质和纹理
4. 相同材质的元素仍然可以合并渲染
*/
```

/*
CircleMask Shader:

Shader "UI/CircleMask"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Radius ("Radius", Float) = 50
        _Center ("Center", Vector) = (0.5, 0.5, 0, 0)
        _SoftEdge ("Soft Edge", Float) = 1
        _Softness ("Softness", Float) = 5
    }

    SubShader
    {
        Tags { "Queue" = "Transparent" }
        Blend SrcAlpha OneMinusSrcAlpha

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;
            float _Radius;
            float2 _Center;
            float _SoftEdge;
            float _Softness;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 col = tex2D(_MainTex, i.uv);

                float2 pixelPos = i.uv * _ScreenParams.xy;
                float2 centerPos = _Center * _ScreenParams.xy;
                float dist = distance(pixelPos, centerPos);

                float alpha = 1;
                if (_SoftEdge > 0.5)
                {
                    alpha = smoothstep(_Radius + _Softness, _Radius - _Softness, dist);
                }
                else
                {
                    alpha = step(dist, _Radius);
                }

                col.a *= alpha;
                return col;
            }
            ENDCG
        }
    }
}
*/
```

---

## 5. 图集与Sprite优化

```csharp
using UnityEngine;
using UnityEngine.U2D;
using UnityEngine.UI;
using System.Collections.Generic;

/// <summary>
/// 图集管理
/// </summary>
public class AtlasManager : MonoBehaviour
{
    /*
    ========== Sprite Atlas ==========

    优点：
    - 自动合并Sprite
    - 减少DrawCall
    - 支持多分辨率
    - 运行时加载

    配置：
    - Include in Build: 是
    - Allow Rotation: 否（避免旋转问题）
    - Tight Packing: 是（节省空间）
    - Padding: 2-4像素（防止 bleeding）
    */

    [Header("Atlases")]
    [SerializeField] private SpriteAtlas uiAtlas;
    [SerializeField] private SpriteAtlas iconAtlas;

    private Dictionary<string, Sprite> spriteCache = new Dictionary<string, Sprite>();

    /// <summary>
    /// 获取Sprite
    /// </summary>
    public Sprite GetSprite(string spriteName)
    {
        if (spriteCache.TryGetValue(spriteName, out var sprite))
            return sprite;

        // 从图集获取
        sprite = uiAtlas.GetSprite(spriteName);
        if (sprite == null)
            sprite = iconAtlas.GetSprite(spriteName);

        if (sprite != null)
            spriteCache[spriteName] = sprite;

        return sprite;
    }

    /// <summary>
    /// 预加载Sprite
    /// </summary>
    public void PreloadSprites(string[] spriteNames)
    {
        foreach (var name in spriteNames)
        {
            GetSprite(name);
        }
    }

    /// <summary>
    /// 清除缓存
    /// </summary>
    public void ClearCache()
    {
        spriteCache.Clear();
        Resources.UnloadUnusedAssets();
    }
}

/// <summary>
/// 动态图集（运行时合并）
/// </summary>
public class DynamicAtlas
{
    private Texture2D atlasTexture;
    private Rect[] uvRects;
    private Dictionary<string, int> spriteIndexMap = new Dictionary<string, int>();
    private int currentIndex = 0;

    public DynamicAtlas(int size)
    {
        atlasTexture = new Texture2D(size, size, TextureFormat.RGBA32, false);
        atlasTexture.filterMode = FilterMode.Bilinear;
        atlasTexture.wrapMode = TextureWrapMode.Clamp;
    }

    /// <summary>
    /// 添加Sprite到动态图集
    /// </summary>
    public bool AddSprite(string name, Texture2D texture)
    {
        if (spriteIndexMap.ContainsKey(name))
            return true;

        // 简化实现，实际需要更复杂的打包算法
        // 这里假设是固定大小的格子
        int cellSize = 64;
        int cellsPerRow = atlasTexture.width / cellSize;

        int x = (currentIndex % cellsPerRow) * cellSize;
        int y = (currentIndex / cellsPerRow) * cellSize;

        if (y + cellSize > atlasTexture.height)
            return false; // 图集已满

        // 复制像素
        Graphics.CopyTexture(texture, 0, 0, 0, 0, texture.width, texture.height,
                           atlasTexture, 0, 0, x, y);

        spriteIndexMap[name] = currentIndex;
        currentIndex++;

        return true;
    }

    public Texture2D GetAtlasTexture() => atlasTexture;

    public Rect GetUVRect(string name)
    {
        if (!spriteIndexMap.TryGetValue(name, out int index))
            return Rect.zero;

        int cellSize = 64;
        int cellsPerRow = atlasTexture.width / cellSize;

        float x = (index % cellsPerRow) * cellSize / (float)atlasTexture.width;
        float y = 1f - ((index / cellsPerRow + 1) * cellSize / (float)atlasTexture.height);
        float w = cellSize / (float)atlasTexture.width;
        float h = cellSize / (float)atlasTexture.height;

        return new Rect(x, y, w, h);
    }
}
```

### 5.2 Image 组件源码分析

```csharp
// ===== 源码文件: UnityEngine.UI/Core/Image.cs =====
// Image - 最常用的 UI 图形组件

namespace UnityEngine.UI
{
    /// <summary>
    /// Image - 显示 Sprite 或纹理的 UI 组件
    /// 源码路径: UnityEngine.UI/Core/Image.cs
    /// </summary>
    public class Image : Graphic
    {
        // ========== 核心成员 ==========

        [SerializeField] private Sprite m_Sprite;
        [SerializeField] private Type m_Type = Type.Simple;
        [SerializeField] private bool m_PreserveAspect = false;
        [SerializeField] private bool m_FillCenter = true;
        [SerializeField] private FillMethod m_FillMethod = FillMethod.Radial360;
        [SerializeField] private float m_FillAmount = 1f;
        [SerializeField] private bool m_FillClockwise = true;
        [SerializeField] private int m_FillOrigin = 0;
        [SerializeField] private float m_PixelsPerUnitMultiplier = 1f;

        // 图片类型枚举
        public enum Type
        {
            Simple,      // 简单模式
            Sliced,      // 九宫格切片
            Tiled,       // 平铺
            Filled       // 填充（用于进度条等）
        }

        public enum FillMethod
        {
            Horizontal,   // 水平填充
            Vertical,     // 垂直填充
            Radial90,     // 90度径向填充
            Radial180,    // 180度径向填充
            Radial360     // 360度径向填充
        }

        // ========== 核心属性 ==========

        public override Texture mainTexture
        {
            get
            {
                // 优先使用 Sprite 的纹理
                if (m_Sprite != null)
                    return m_Sprite.texture;

                // 回退到 Graphic 的默认纹理
                return base.mainTexture;
            }
        }

        public Sprite sprite
        {
            get => m_Sprite;
            set
            {
                if (m_Sprite != value)
                {
                    m_Sprite = value;
                    SetVerticesDirty();  // 顶点需要重建
                    SetMaterialDirty();  // 材质需要重建（纹理可能变化）
                }
            }
        }

        public Type type
        {
            get => m_Type;
            set { m_Type = value; SetVerticesDirty(); }
        }

        // ========== 网格生成 ==========

        /// <summary>
        /// OnPopulateMesh - 生成网格（重写 Graphic 的方法）
        /// </summary>
        protected override void OnPopulateMesh(VertexHelper vh)
        {
            if (activeSprite == null)
            {
                // 没有 Sprite，使用 Graphic 默认生成
                base.OnPopulateMesh(vh);
                return;
            }

            switch (m_Type)
            {
                case Type.Simple:
                    GenerateSimpleSprite(vh);
                    break;
                case Type.Sliced:
                    GenerateSlicedSprite(vh);
                    break;
                case Type.Tiled:
                    GenerateTiledSprite(vh);
                    break;
                case Type.Filled:
                    GenerateFilledSprite(vh);
                    break;
            }
        }

        /// <summary>
        /// 生成简单 Sprite 网格
        /// </summary>
        private void GenerateSimpleSprite(VertexHelper vh)
        {
            vh.Clear();

            // 获取绘制尺寸
            var r = GetDrawingDimensions(false);

            // 获取 UV 坐标
            var uv = (activeSprite != null)
                ? UnityEngine.Sprites.DataUtility.GetOuterUV(activeSprite)
                : new Vector4(0, 0, 1, 1);

            // 顶点颜色
            var color32 = color;

            // 添加 4 个顶点
            var vert = new UIVertex();
            vert.color = color32;

            // 左下
            vert.position = new Vector3(r.x, r.y);
            vert.uv0 = new Vector2(uv.x, uv.y);
            vh.AddVert(vert);

            // 左上
            vert.position = new Vector3(r.x, r.y + r.height);
            vert.uv0 = new Vector2(uv.x, uv.w);
            vh.AddVert(vert);

            // 右上
            vert.position = new Vector3(r.x + r.width, r.y + r.height);
            vert.uv0 = new Vector2(uv.z, uv.w);
            vh.AddVert(vert);

            // 右下
            vert.position = new Vector3(r.x + r.width, r.y);
            vert.uv0 = new Vector2(uv.z, uv.y);
            vh.AddVert(vert);

            // 添加三角形
            vh.AddTriangle(0, 1, 2);
            vh.AddTriangle(2, 3, 0);
        }

        /// <summary>
        /// 生成九宫格 Sprite 网格
        /// </summary>
        private void GenerateSlicedSprite(VertexHelper vh)
        {
            vh.Clear();

            if (activeSprite == null)
                return;

            // 获取九宫格边框
            var border = activeSprite.border;

            // 如果没有边框，回退到简单模式
            if (border == Vector4.zero)
            {
                GenerateSimpleSprite(vh);
                return;
            }

            // 九宫格产生 9 个区域，需要 16 个顶点
            // 计算每个区域的尺寸和 UV
            var outerUV = UnityEngine.Sprites.DataUtility.GetOuterUV(activeSprite);
            var innerUV = UnityEngine.Sprites.DataUtility.GetInnerUV(activeSprite);

            var r = GetDrawingDimensions(false);

            // 计算顶点位置（考虑 border）
            float x0 = r.x;
            float x1 = r.x + border.x / pixelsPerUnit;
            float x2 = r.x + r.width - border.z / pixelsPerUnit;
            float x3 = r.x + r.width;

            float y0 = r.y;
            float y1 = r.y + border.w / pixelsPerUnit;
            float y2 = r.y + r.height - border.y / pixelsPerUnit;
            float y3 = r.y + r.height;

            // ... 省略详细的顶点生成代码
            // 实际源码会生成 16 个顶点和对应的三角形
        }

        // ========== 尺寸计算 ==========

        /// <summary>
        /// 计算绘制尺寸（考虑 PixelsPerUnit）
        /// </summary>
        private Rect GetDrawingDimensions(bool shouldPreserveAspect)
        {
            var r = GetPixelAdjustedRect();

            if (activeSprite == null)
                return r;

            var spriteSize = new Vector2(activeSprite.rect.width, activeSprite.rect.height);

            if (shouldPreserveAspect && spriteSize.sqrMagnitude > 0.001f)
            {
                // 保持宽高比
                var spriteRatio = spriteSize.x / spriteSize.y;
                var rectRatio = r.width / r.height;

                if (spriteRatio > rectRatio)
                {
                    r.height = r.width * (1f / spriteRatio);
                }
                else
                {
                    r.width = r.height * spriteRatio;
                }
            }

            return r;
        }

        // ========== 辅助属性 ==========

        public float pixelsPerUnit
        {
            get
            {
                float spritePixelsPerUnit = 100;
                if (activeSprite != null)
                    spritePixelsPerUnit = activeSprite.pixelsPerUnit;

                float referencePixelsPerUnit = 100;
                if (canvas != null)
                    referencePixelsPerUnit = canvas.referencePixelsPerUnit;

                return spritePixelsPerUnit / referencePixelsPerUnit;
            }
        }
    }
}

/*
========== 源码要点总结 ==========

1. Image 继承自 Graphic，重写 OnPopulateMesh 生成网格
2. mainTexture 属性返回 Sprite 的纹理，影响合批
3. 四种类型：Simple、Sliced、Tiled、Filled
4. 修改 sprite 属性会触发 SetVerticesDirty 和 SetMaterialDirty
5. pixelsPerUnit 影响 Sprite 的实际渲染尺寸

========== 合批与图集的关系 ==========

1. 同一 Sprite Atlas 下的 Sprite 共享同一纹理
2. 使用相同纹理的 Image 可以合批
3. mainTexture 返回值决定纹理是否相同
4. Sprite.texture 指向图集纹理，不是原始纹理
*/
```

### 5.3 SpriteAtlas 运行时行为

```csharp
// ===== SpriteAtlas 运行时行为分析 =====
// SpriteAtlas 是 Unity 2017.2+ 引入的图集系统

namespace UnityEngine.U2D
{
    /// <summary>
    /// SpriteAtlas 运行时行为分析
    /// </summary>
    public static class SpriteAtlasAnalysis
    {
        /*
        ========== SpriteAtlas 编译时行为 ==========

        1. 打包时，Unity 将 SpriteAtlas 内的 Sprite 合并到一张纹理
        2. 每个 Sprite 的纹理区域被记录为 UV 坐标
        3. 原始 Sprite 引用被重定向到图集纹理

        ========== 运行时加载流程 ==========

        1. SpriteAtlas 作为 AssetBundle 或 Resources 加载
        2. 调用 atlas.GetSprite("spriteName") 获取 Sprite
        3. 返回的 Sprite.texture 指向图集纹理
        4. Sprite.rect 指定在图集中的区域

        ========== 对 DrawCall 的影响 ==========

        不使用图集：
        Image A (texture1) → Draw Call 1
        Image B (texture2) → Draw Call 2
        Image C (texture3) → Draw Call 3
        总计: 3 Draw Calls

        使用图集：
        Image A (atlas_texture, region A) →
        Image B (atlas_texture, region B) → Draw Call 1 (合批)
        Image C (atlas_texture, region C) →
        总计: 1 Draw Call
        */

        /// <summary>
        /// 演示图集对纹理的影响
        /// </summary>
        public static void AnalyzeSpriteTexture(Sprite sprite)
        {
            if (sprite == null)
            {
                Debug.Log("Sprite is null");
                return;
            }

            Debug.Log($"=== Sprite 分析: {sprite.name} ===");
            Debug.Log($"纹理: {sprite.texture?.name ?? "null"}");
            Debug.Log($"区域: {sprite.rect}");
            Debug.Log($"边界: {sprite.border}");
            Debug.Log($"Pivot: {sprite.pivot}");
            Debug.Log($"Pixels Per Unit: {sprite.pixelsPerUnit}");

            // 检查是否来自图集
            if (sprite.texture != null && sprite.texture.name != sprite.name)
            {
                Debug.Log("此 Sprite 来自图集（纹理名与 Sprite 名不同）");
            }
        }
    }
}

// ===== Image 与图集的关系 =====
namespace UnityEngine.UI
{
    public static class ImageAtlasRelation
    {
        /*
        ========== Image 如何使用图集 ==========

        1. Image.sprite = spriteFromAtlas
        2. Image.mainTexture 返回 sprite.texture（图集纹理）
        3. OnPopulateMesh 使用 Sprite 的 UV 坐标
        4. 渲染时只绘制图集纹理的对应区域

        ========== UV 坐标计算 ==========

        简单模式：
        - GetOuterUV() 获取 Sprite 在图集中的 UV 范围
        - 4 个顶点使用对应的 UV 坐标

        九宫格模式：
        - GetInnerUV() 获取内部区域的 UV
        - 16 个顶点对应不同的 UV 区域

        ========== 合批检查 ==========

        两个 Image 可以合批的条件：
        1. mainTexture 相同（同一图集）
        2. material 相同
        3. 层级连续

        图集保证了条件 1 自动满足。
        */

        /// <summary>
        /// 检查两个 Image 是否可以合批
        /// </summary>
        public static bool CanBatch(Image a, Image b)
        {
            if (a == null || b == null)
                return false;

            // 检查纹理
            if (a.mainTexture != b.mainTexture)
                return false;

            // 检查材质
            if (a.material != b.material)
                return false;

            return true;
        }
    }
}

/*
========== 图集优化最佳实践 ==========

1. 按 UI 界面组织图集
   - 每个界面的 Sprite 放入同一图集
   - 避免跨界面引用导致 Draw Call 增加

2. 控制图集大小
   - 推荐尺寸：1024x1024 或 2048x2048
   - 过大会增加内存和加载时间

3. 使用 Tight Packing
   - 紧凑排列，节省空间
   - 注意 Padding 防止 bleeding

4. 多图集策略
   - 高频使用：小图集，快速加载
   - 低频使用：大图集，按需加载

5. 运行时动态图集
   - 对于网络下载的图片
   - 使用 DynamicAtlas 或 SpriteAtlas.Add()
*/
```

---

## 6. UGUI性能优化清单

```
┌─────────────────────────────────────────────────────────────┐
│                   UGUI 性能优化清单                           │
│                                                             │
│  1. Canvas优化                                              │
│     ├── 分离动态/静态UI到不同Canvas                          │
│     ├── 避免深层嵌套                                        │
│     └── 使用合适的渲染模式                                   │
│                                                             │
│  2. DrawCall优化                                            │
│     ├── 使用Sprite Atlas                                   │
│     ├── 减少材质数量                                        │
│     ├── 优化层级顺序                                        │
│     └── 避免打断合批（Mask、文字）                           │
│                                                             │
│  3. Raycast优化                                             │
│     ├── 关闭不必要的Raycast Target                          │
│     └── 使用RectMask2D替代Mask                              │
│                                                             │
│  4. 布局优化                                                │
│     ├── 避免过多LayoutGroup嵌套                             │
│     ├── 大列表使用虚拟列表                                   │
│     └── 缓存布局计算结果                                    │
│                                                             │
│  5. 内存优化                                                │
│     ├── 合理使用图集大小                                    │
│     ├── 及时卸载不用的Sprite                                │
│     └── 避免大纹理                                          │
│                                                             │
│  6. 文字优化                                                │
│     ├── 使用TextMeshPro                                    │
│     ├── 限制字体纹理大小                                    │
│     └── 缓存文字网格                                        │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 性能优化源码级分析

#### 6.2.1 Canvas 脏标记机制

```csharp
// ===== Canvas 脏标记机制分析 =====
// 理解脏标记是理解 UGUI 性能优化的关键

namespace UnityEngine.UI
{
    /// <summary>
    /// Canvas 脏标记机制分析
    /// </summary>
    public static class CanvasDirtyFlagAnalysis
    {
        /*
        ========== 脏标记类型 ==========

        UGUI 使用脏标记避免不必要的重建：

        1. VertexDirty（顶点脏）
           - 触发条件：RectTransform 尺寸变化、Sprite 变化
           - 调用：SetVerticesDirty()
           - 重建：OnPopulateMesh()

        2. MaterialDirty（材质脏）
           - 触发条件：材质变化、颜色变化
           - 调用：SetMaterialDirty()
           - 重建：UpdateMaterial()

        3. LayoutDirty（布局脏）
           - 触发条件：子物体增减、尺寸变化
           - 调用：LayoutRebuilder.MarkLayoutForRebuild()
           - 重建：布局系统重建

        ========== Canvas 重建代价 ==========

        当 Canvas 中任何元素变脏时：
        1. 整个 Canvas 的所有元素都需要重新遍历
        2. 重新计算批处理
        3. 重新生成网格

        这就是为什么要分离动态/静态 UI！

        ========== 源码流程 ==========

        Graphic.SetVerticesDirty()
        └── CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild()
            └── 添加到 m_ElementsForGraphicRebuild

        Canvas.willRenderCanvases (每帧渲染前)
        └── CanvasUpdateRegistry.PerformUpdate()
            ├── ProcessLayoutRebuild()      // 布局重建
            ├── ClipperRegistry.Cull()      // 裁剪
            └── ProcessGraphicRebuild()     // 图形重建
                └── Graphic.Rebuild()
                    └── DoMeshGeneration()
        */

        /// <summary>
        /// 分析 Canvas 重建频率
        /// </summary>
        public static void AnalyzeCanvasRebuild(Canvas canvas)
        {
            var graphics = canvas.GetComponentsInChildren<Graphic>(true);
            int dirtyCount = 0;

            foreach (var graphic in graphics)
            {
                // 检查是否有脏标记（通过反射）
                var type = graphic.GetType();
                var vertsDirtyField = type.GetField("m_VertsDirty",
                    System.Reflection.BindingFlags.NonPublic |
                    System.Reflection.BindingFlags.Instance);

                if (vertsDirtyField != null)
                {
                    bool isDirty = (bool)vertsDirtyField.GetValue(graphic);
                    if (isDirty)
                    {
                        dirtyCount++;
                        Debug.Log($"Dirty Graphic: {graphic.name}", graphic.gameObject);
                    }
                }
            }

            Debug.Log($"Canvas {canvas.name}: {dirtyCount}/{graphics.Length} graphics are dirty");
        }
    }
}

// ===== 动静分离原理 =====
namespace UnityEngine.UI
{
    /// <summary>
    /// 动静分离原理说明
    /// </summary>
    public static class StaticDynamicSeparation
    {
        /*
        ========== 问题场景 ==========

        单 Canvas：
        ┌─────────────────────────────────┐
        │ Canvas (静态 + 动态)             │
        │  ├── 背景图 (静态)               │
        │  ├── 标题文字 (静态)             │
        │  ├── 装饰元素 (静态)             │
        │  ├── 动画图标 (动态, 每帧更新)   │
        │  └── 进度条 (动态, 频繁更新)     │
        └─────────────────────────────────┘

        问题：动画图标每帧触发 Canvas 重建
        → 背景图、标题文字等静态元素也被重建
        → 性能浪费！

        ========== 解决方案 ==========

        分离 Canvas：
        ┌─────────────────────────────────┐
        │ Canvas_Static                   │
        │  ├── 背景图                     │
        │  ├── 标题文字                   │
        │  └── 装饰元素                   │
        └─────────────────────────────────┘
        ┌─────────────────────────────────┐
        │ Canvas_Dynamic                  │
        │  ├── 动画图标                   │
        │  └── 进度条                     │
        └─────────────────────────────────┘

        效果：
        - 动态元素更新只重建 Canvas_Dynamic
        - Canvas_Static 永不重建（除非首次）
        - 性能提升显著！

        ========== 分离原则 ==========

        1. 按更新频率分离
           - 永不更新 → 静态 Canvas
           - 偶尔更新 → 半静态 Canvas
           - 频繁更新 → 动态 Canvas

        2. 按功能模块分离
           - 背景层
           - 内容层
           - 弹窗层

        3. 注意：Canvas 越多，Draw Call 可能越多
           - 需要在合批和重建之间平衡
        */
    }
}
```

#### 6.2.2 GC 优化要点

```csharp
// ===== UGUI GC 优化分析 =====

namespace UnityEngine.UI
{
    /// <summary>
    /// UGUI 常见 GC 分配点及优化
    /// </summary>
    public static class UGUIGCOptimization
    {
        /*
        ========== 常见 GC 分配点 ==========

        1. List<T> 扩容
           位置：多处使用 List<T>
           问题：当 Count 超过 Capacity 时扩容
           解决：预分配容量

        2. foreach 循环
           位置：遍历 List、Dictionary
           问题：Unity 旧版本 foreach 会产生 GC
           解决：使用 for 循环或缓存迭代器

        3. 委托/事件
           位置：按钮点击、事件监听
           问题：lambda 表达式产生闭包
           解决：使用方法引用代替 lambda

        4. 字符串拼接
           位置：UI 文字更新
           问题：string 不可变，每次拼接产生新字符串
           解决：使用 StringBuilder

        5. 协程 yield
           位置：UI 动画、延迟操作
           问题：new WaitForSeconds() 产生 GC
           解决：缓存 WaitForSeconds 对象
        */

        /// <summary>
        /// List 预分配优化
        /// </summary>
        public static class ListOptimization
        {
            // 错误写法
            public static void BadExample()
            {
                var list = new List<Graphic>();  // 初始容量 0
                // 添加元素时多次扩容
                for (int i = 0; i < 100; i++)
                {
                    list.Add(null);  // 可能触发扩容 GC
                }
            }

            // 正确写法
            public static void GoodExample()
            {
                var list = new List<Graphic>(100);  // 预分配容量
                for (int i = 0; i < 100; i++)
                {
                    list.Add(null);  // 无扩容 GC
                }
            }
        }

        /// <summary>
        /// 事件监听优化
        /// </summary>
        public static class EventOptimization
        {
            // 错误写法：lambda 产生闭包 GC
            public static void BadExample(Button button, int id)
            {
                button.onClick.AddListener(() =>
                {
                    Debug.Log($"Clicked: {id}");  // 闭包捕获 id，产生 GC
                });
            }

            // 正确写法：使用类成员变量
            private static int s_ClickId;
            public static void GoodExample(Button button, int id)
            {
                s_ClickId = id;
                button.onClick.AddListener(OnButtonClick);  // 方法引用，无 GC
            }

            private static void OnButtonClick()
            {
                Debug.Log($"Clicked: {s_ClickId}");
            }
        }

        /// <summary>
        /// 协程优化
        /// </summary>
        public static class CoroutineOptimization
        {
            // 错误写法：每次 yield 产生 GC
            public static System.Collections.IEnumerator BadExample()
            {
                while (true)
                {
                    yield return new WaitForSeconds(1f);  // 每次产生 GC
                }
            }

            // 正确写法：缓存 WaitForSeconds
            private static WaitForSeconds s_WaitOneSecond = new WaitForSeconds(1f);

            public static System.Collections.IEnumerator GoodExample()
            {
                while (true)
                {
                    yield return s_WaitOneSecond;  // 无 GC
                }
            }
        }

        /// <summary>
        /// 字符串优化
        /// </summary>
        public static class StringOptimization
        {
            private static System.Text.StringBuilder s_SB = new System.Text.StringBuilder();

            // 错误写法：字符串拼接
            public static string BadExample(int score, int level)
            {
                return "Score: " + score + " Level: " + level;  // 产生多个临时字符串
            }

            // 正确写法：使用 StringBuilder
            public static string GoodExample(int score, int level)
            {
                s_SB.Clear();
                s_SB.Append("Score: ");
                s_SB.Append(score);
                s_SB.Append(" Level: ");
                s_SB.Append(level);
                return s_SB.ToString();  // 只产生一个字符串
            }
        }
    }
}

// ===== UGUI 源码中的 GC 优化技巧 =====
namespace UnityEngine.UI
{
    /// <summary>
    /// UGUI 源码中的 GC 优化模式
    /// </summary>
    public static class UGUIGCPatterns
    {
        /*
        ========== 1. 对象池模式 ==========

        VertexHelper 使用对象池：
        public static VertexHelper Get()
        {
            return s_Pool.Get();
        }

        public void Dispose()
        {
            Clear();
            s_Pool.Release(this);
        }

        优点：复用对象，减少 new 和 GC

        ========== 2. ListPool<T> 模式 ==========

        临时列表使用 ListPool：
        var list = ListPool<Graphic>.Get();
        // 使用 list
        ListPool<Graphic>.Release(list);

        优点：避免频繁 new List<T>

        ========== 3. 缓存组件引用 ==========

        Graphic 缓存 RectTransform：
        private RectTransform m_RectTransform;

        public RectTransform rectTransform
        {
            get
            {
                if (m_RectTransform == null)
                    m_RectTransform = GetComponent<RectTransform>();
                return m_RectTransform;
            }
        }

        优点：避免每次调用 GetComponent<T>()

        ========== 4. 静态事件处理器 ==========

        ExecuteEvents 使用静态委托：
        private static readonly EventFunction<IPointerClickHandler> s_PointerClickHandler =
            (handler, eventData) => { handler.OnPointerClick((PointerEventData)eventData); };

        优点：委托只创建一次
        */
    }
}
```

---

## UGUI 源码关键类速查表

| 类名 | 职责 | 关键方法 | 源码路径 |
|------|------|----------|----------|
| **Graphic** | UI 可视元素基类 | `Rebuild()`, `OnPopulateMesh()`, `SetVerticesDirty()` | `UI/Core/Graphic.cs` |
| **Image** | 图片显示组件 | `OnPopulateMesh()`, `mainTexture` | `UI/Core/Image.cs` |
| **Text** | 文本显示组件 | `OnPopulateMesh()`, `font` | `UI/Core/Text.cs` |
| **RawImage** | 原始纹理显示 | `OnPopulateMesh()`, `texture` | `UI/Core/RawImage.cs` |
| **CanvasRenderer** | Canvas 渲染器 | `SetMesh()`, `SetMaterial()` | `UI/Core/CanvasRenderer.cs` |
| **CanvasUpdateRegistry** | 更新注册中心 | `RegisterCanvasElementForGraphicRebuild()`, `PerformUpdate()` | `UI/Core/CanvasUpdateRegistry.cs` |
| **VertexHelper** | 顶点辅助类 | `AddUIVertexQuad()`, `FillMesh()` | `UI/Core/VertexHelper.cs` |
| **GraphicRegistry** | Graphic 注册表 | `RegisterGraphicForCanvas()`, `GetRaycastableGraphicsForCanvas()` | `UI/Core/GraphicRegistry.cs` |
| **LayoutGroup** | 布局组基类 | `CalculateLayoutInputHorizontal()`, `SetLayoutHorizontal()` | `UI/Layout/LayoutGroup.cs` |
| **LayoutRebuilder** | 布局重建器 | `MarkLayoutForRebuild()`, `Rebuild()` | `UI/Layout/LayoutRebuilder.cs` |
| **ContentSizeFitter** | 自适应尺寸 | `SetLayoutHorizontal()`, `SetLayoutVertical()` | `UI/Layout/ContentSizeFitter.cs` |
| **HorizontalLayoutGroup** | 水平布局 | `CalcAlongAxis()`, `SetChildrenAlongAxis()` | `UI/Layout/HorizontalLayoutGroup.cs` |
| **VerticalLayoutGroup** | 垂直布局 | `CalcAlongAxis()`, `SetChildrenAlongAxis()` | `UI/Layout/VerticalLayoutGroup.cs` |
| **GridLayoutGroup** | 网格布局 | `CalculateLayoutInputHorizontal()`, `SetLayoutHorizontal()` | `UI/Layout/GridLayoutGroup.cs` |
| **EventSystem** | 事件系统核心 | `Update()`, `RaycastAll()`, `SetSelectedGameObject()` | `EventSystem/EventSystem.cs` |
| **GraphicRaycaster** | UI 射线检测 | `Raycast()` | `UI/Core/GraphicRaycaster.cs` |
| **ExecuteEvents** | 事件执行器 | `Execute()`, `ExecuteHierarchy()` | `EventSystem/ExecuteEvents.cs` |
| **BaseInputModule** | 输入模块基类 | `Process()` | `EventSystem/InputModules/BaseInputModule.cs` |
| **StandaloneInputModule** | 标准输入模块 | `ProcessMouseEvent()`, `ProcessTouchEvent()` | `EventSystem/InputModules/StandaloneInputModule.cs` |
| **Mask** | 模板遮罩 | `GetModifiedMaterial()` | `UI/Core/Mask.cs` |
| **RectMask2D** | 矩形裁剪 | `PerformClipping()`, `GetModifiedMaterial()` | `UI/Core/RectMask2D.cs` |
| **MaskUtilities** | 遮罩工具类 | `GetStencilDepth()`, `NotifyStencilStateChanged()` | `UI/Core/MaskUtilities.cs` |
| **ClipperRegistry** | 裁剪器注册表 | `Register()`, `Cull()` | `UI/Core/ClipperRegistry.cs` |
| **ScrollRect** | 滚动视图 | `OnScroll()`, `SetContentAnchoredPosition()` | `UI/Controls/ScrollRect.cs` |
| **Button** | 按钮控件 | `OnPointerClick()`, `Press()` | `UI/Controls/Button.cs` |
| **Toggle** | 开关控件 | `OnPointerClick()`, `Set()` | `UI/Controls/Toggle.cs` |
| **Slider** | 滑动条 | `UpdateVisuals()`, `Set()` | `UI/Controls/Slider.cs` |
| **InputField** | 输入框 | `OnSubmit()`, `Append()` | `UI/Controls/InputField.cs` |
| **Selectable** | 可交互组件基类 | `OnPointerDown()`, `OnPointerUp()`, `DoStateTransition()` | `UI/Controls/Selectable.cs` |

---

## 本课小结

### 核心知识点

| 知识点 | 核心要点 |
|--------|----------|
| Canvas渲染 | Overlay/Camera/World模式选择 |
| 批处理 | 相同材质+纹理+连续层级 |
| 事件系统 | IPointerHandler接口族 |
| 射线检测 | 减少Raycast Target |
| 布局系统 | LayoutGroup、虚拟列表 |
| 遮罩 | RectMask2D优先于Mask |
| 图集 | Sprite Atlas、动态图集 |

### 源码分析要点

| 系统 | 核心类 | 关键流程 |
|------|--------|----------|
| 渲染 | Graphic, CanvasUpdateRegistry | SetDirty → Register → Rebuild |
| 批处理 | CanvasRenderer (Native) | 材质/纹理检查 → 合并 DrawCall |
| 事件 | EventSystem, GraphicRaycaster | Process → Raycast → Execute |
| 布局 | LayoutGroup, LayoutRebuilder | Mark → Calculate → SetLayout |
| 遮罩 | Mask, RectMask2D | Stencil/Clipping |

### 性能优化优先级

```
1. 减少DrawCall（图集、层级）
2. 关闭不必要的Raycast Target
3. 使用虚拟列表处理大数据
4. 分离动态/静态UI（Canvas脏标记机制）
5. 使用RectMask2D替代Mask
6. 使用TextMeshPro
7. 预分配List容量，减少GC
```

---

## 相关链接

### 官方资源
- [Unity UI Best Practices](https://docs.unity3d.com/Manual/UIBestPracticeGuides.html)
- [UGUI Source Code (GitHub)](https://github.com/Unity-Technologies/uGUI)
- [TextMeshPro](https://docs.unity3d.com/Packages/com.unity.textmeshpro@latest)

### 源码分析
- `UnityEngine.UI/Core/` - 核心组件实现
- `UnityEngine.UI/Layout/` - 布局系统实现
- `UnityEngine.EventSystems/` - 事件系统实现

### 相关文档
- [UI系统架构](./【教程】UI系统架构.md)
- [Unity内存管理](../04-性能优化/Unity内存管理.md)
- [渲染管线基础](../02-渲染与图形/渲染管线基础.md)
