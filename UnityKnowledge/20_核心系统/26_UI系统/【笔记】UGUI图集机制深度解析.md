---
title: 【笔记】UGUI图集机制深度解析
tags: ["Unity", "UI", "UGUI", "SpriteAtlas", "图集", "合批", "AssetBundle", "深度解析"]
category: 核心系统/UI系统
created: "2026-07-02"
updated: "2026-07-02"
description: UGUI 图集全链路原理：Sprite/Texture2D 关系、SpriteAtlas 运行时纹理替换、sactx 与 AB 冗余规则、动态图集代价、图集与合批关系、TMP 字体 Atlas 优化
unity_version: 2021.3+
status: 待验证
validation: 基于知识库现有文档汇总整理
related:
  - "[[【笔记】UI系统知识脑图]]"
  - "[[【笔记】UGUI性能优化实战总览]]"
  - "[[【笔记】UGUI DrawCall影响因素全面测试]]"
  - "[[【笔记】UGUI自适应机制深度解析]]"
  - "[[【片段】UGUI 性能优化规则清单]]"
  - "[[UI系统专题索引]]"
author: llm
sources:
  - "[[【笔记】UGUI性能优化实战总览]]"
  - "[[【笔记】UGUI DrawCall影响因素全面测试]]"
  - "[[【片段】UGUI 性能优化规则清单]]"
  - "[[【笔记】Unity资源依赖与打包陷阱]]"
  - "UGUI 第12章 UI 资源与图集系统"
  - "UGUI 第22章 批处理优化"
---

# 【笔记】UGUI 图集机制深度解析

> 图集减少的不是图片数量，而是 GPU 的纹理切换次数。本文从 Sprite/Texture2D 底层关系出发，把 sactx 冗余、AB 打包、动态图集、合批影响、TMP Atlas 优化串成一条完整知识链。

## 文档定位

知识库中图集知识分散在 15+ 篇文档中（UGUI 系列书籍、性能优化实战、DrawCall 测试、AB 打包陷阱、UWA 外部文章等）。本文将核心结论提炼为一篇系统性原理文档，可作为面试讲解和工程实现的入口。

---

## 一、图集的本质

### 1.1 Sprite 与 Texture2D 的关系

```
Texture2D              Sprite
┌────────────┐        ┌──────────────────────┐
│  像素数据    │        │  区域描述（壳子）        │
│  2048×2048 │        │  offset + size + UV  │
│  RGBA32    │        │  不保存任何像素         │
└────────────┘        └──────────────────────┘
      ↑                        ↑
  GPU 只认识这个           UGUI Image 引用的是这个
```

一个 2048×2048 的纹理切出 1000 个 Sprite，内存不变——因为像素数据只有一份。图集做的事情就是把很多零散的小 Texture2D 合并成一张大 Texture2D，让 GPU 减少纹理切换。

### 1.2 为什么图集能减少 DrawCall

```
不用图集：                             用图集：
Image1 → Texture_A → DrawCall 1       Image1 → Atlas → DrawCall 1
Image2 → Texture_B → DrawCall 2       Image2 → Atlas ↗
Image3 → Texture_A → DrawCall 3       Image3 → Atlas ↗
（每次切换纹理 = 新的 DrawCall）         （同一张纹理 = 合批为 1 个 DrawCall）
```

来自 [[【笔记】UGUI DrawCall影响因素全面测试]] 的实测数据：

| 场景 | 图集数量 | DrawCall | 渲染时间 | 内存占用 |
|------|----------|----------|----------|----------|
| 单图集（100 个 UI 元素） | 1 | **2** | 0.85ms | 12.4MB |
| 多图集 | 5 | 10 | 4.2ms | 14.1MB |
| 零散纹理 | 100 | **100** | 43.5ms | 28.7MB |

> 结论：图集数量与 DrawCall 呈正相关。单图集 100 个 UI 元素仅需 2 DC。

---

## 二、SpriteAtlas 运行时纹理替换机制

### 2.1 核心原理：Texture Replacement

这是理解后续所有 sactx 冗余问题的基础。

```
构建阶段（Editor Build）：
  Sprite_A（引用 Texture_Small）
  SpriteAtlas（包含 Sprite_A）
        ↓ Packing
  生成图集纹理 sactx（Sprite Atlas Cache Texture）

运行时加载后：
  Sprite_A 的内部 texture 引用被「悄悄替换」
  Sprite_A.texture → 不再指向 Texture_Small
                   → 指向 sactx（图集纹理）
  Sprite_A 的 UV 被重新映射到 sactx 中对应区域
```

**开发者完全不需要修改 `Image.sprite` 赋值**——SpriteAtlas 在幕后做了重定向。

```csharp
// 这行代码不变，但底层纹理已被替换
image.sprite = mySprite;

// mySprite.texture 已经指向 sactx，而非原始小纹理
Texture2D atlasTex = mySprite.texture; // → sactx
```

### 2.2 获取 Sprite 在图集中的 UV

```csharp
// 获取 Sprite 在所在图集中的 Outer UV
// 用于自定义 mesh、RawImage 等场景
Vector4 uv = Sprites.DataUtility.GetOuterUV(activeSprite);
```

### 2.3 Padding 的必要性

图集中 Sprite 之间留 Padding（间距）是**必要保护机制**，不是浪费空间：

- **双线性过滤**：采样时读取相邻像素，无 Padding 会串色
- **Mipmap**：降级时相邻 Sprite 颜色混合（UI 一般不开 Mipmap）
- **缩放**：CanvasScaler 缩放后像素边界模糊

---

## 三、sactx 与 AssetBundle 冗余规则

> 知识库中最有工程价值的结论之一，来自 [[【笔记】UGUI性能优化实战总览]]。

### 3.1 什么是 sactx

SpriteAtlas 构建后生成的图集纹理叫 **sactx**（Sprite Atlas Cache Texture）。这是实际被 GPU 使用的大纹理。

### 3.2 两种打包场景的冗余规则

```
场景 A：SpriteAtlas 不打进 AssetBundle
├── 必须勾选 Include in Build（否则 sactx 消失）
├── 所有小 Sprite 必须打进同一个 AssetBundle
│   └── 否则：每个 AB 各打一份 sactx → 冗余！
└── 结论：Sprite 同 AB + Atlas 勾 Include

场景 B：SpriteAtlas 本身打进 AssetBundle
├── sactx 永不冗余（打包造成的冗余）
├── 小 Sprite 也最好打进 AB
│   └── 否则：小 Sprite 的原始 Texture 被打进包体 → 冗余
└── 结论：Sprite 和 Atlas 同 AB（或 Atlas 独立 AB + Sprite 同 AB）
```

**一句话记忆**：Sprite 和它的 SpriteAtlas **必须在同一个 AssetBundle 里**，否则总有东西会冗余。

### 3.3 Include in Build 的真正含义

勾不勾 `Include in Build` **不影响依赖关系**，唯一区别是：

| Include in Build | 行为 |
|---|---|
| **勾选** | 构建时主动生成 sactx，Sprite 直接显示图集纹理 |
| **不勾** | sactx 不生成，Sprite 退回使用原始小纹理；需运行时脚本 `SpriteAtlasManager.RegisterAtlas()` 手动加载 |

### 3.4 SpriteAtlas 与 AB 的 late binding 问题

来自 [[【笔记】Unity资源依赖与打包陷阱]]：

> Sprite 位于一个 Bundle，Atlas 位于另一个 Bundle 且未提前加载时，Sprite 会退回原始纹理，直接破坏 Batch。

```
正确加载顺序：
  ① 加载 SpriteAtlas 的 AB → sactx 就绪
  ② 加载 Sprite 的 AB → Sprite.texture → sactx（正确指向图集）
  ③ 创建 Image → 使用图集纹理 → 可合批

错误加载顺序：
  ① 加载 Sprite 的 AB → Sprite.texture → Texture_Small（退回原始纹理）
  ② 创建 Image → 使用原始纹理 → 无法与图集元素合批
  ③ 后续加载 Atlas AB → 已创建的 Image 可能不自动刷新
```

---

## 四、RawImage 陷阱

> 用 Sprite Packer 打图集时，图集中的图片**不能被 `RawImage` 引用**。

```
RawImage 引用了图集中的 Sprite
  → Unity 发现 RawImage 需要原始 Texture（不是图集纹理）
  → 额外把原始 Texture2D 再打一份进包体
  → 同一张图存在两处，包体增大
```

**结论**：
- `RawImage` 只用 `Texture2D`，不要用 `Sprite`
- 如果一定要引用某个图，**不要把它打入图集**

---

## 五、图集尺寸策略

来自 [[【笔记】UGUI性能优化实战总览]] 和 [[【最佳实践】UI性能优化速查]]：

### 5.1 分配原则

| 资源类型 | 推荐尺寸 | 说明 |
|----------|----------|------|
| 常驻通用资源（按钮底图、边框） | 2048~4096 | 全局使用，不卸载 |
| 功能独有图集（商城、背包） | **≤1024** | 用完即卸 |
| 特殊大功能 | 达到 3 张 1024 才升 2048 | 不要过早升级 |

### 5.2 权衡逻辑

```
多一张贴图 ≈ 多 1 个 DrawCall（可接受）
强行合并 2 张 1024 → 1 张 2048：
  ├── 空白多
  ├── 合并不合理
  └── 内存反而浪费

Shader 绑定贴图的消耗是 ns 级
主要消耗在渲染面积（填充率）和 DrawCall，而非贴图绑定本身
```

### 5.3 移动端推荐导入配置

```
Max Size:        1024（功能内）/ 2048（通用常驻）
Format:          ASTC 6x6（iOS/Android 推荐）
Read/Write:      false（省一半 CPU 内存！）
Allow Rotation:  false（UI 不建议旋转）
Generate Mipmap: false（UI 不需要）
Tight Packing:   true（节省空间）
Include in Build: 根据打包策略决定
```

> 关闭 Read/Write 非常关键：开启时纹理在 CPU 内存和 GPU 显存各存一份，2048×2048 RGBA32 从 16MB 变成 32MB。

---

## 六、图集与合批的关系

### 6.1 合批前提

合批七大条件中，图集直接影响的是「相同 Texture」：

```
同一 Canvas + 相同 Material + 相同 Texture（图集）→ 可以合批
```

### 6.2 图集正确但 DrawCall 仍然高的原因

来自 UGUI 第12章和第22章的关键结论：

| 原因 | 说明 |
|------|------|
| **层级穿插** | 图集 A 的元素夹在图集 B 元素中间，打断连续性 |
| **Mask / RectMask2D** | 裁剪区域不同，打段合批 |
| **字体图集与 UI 图集不同** | Text 的纹理天然与 Image 不同 |
| **不同 Canvas** | 跨 Canvas 默认不合批 |
| **Material 实例不同** | 代码 new Material 或 MaterialPropertyBlock 创建了新实例 |
| **Atlas 未加载退回原始纹理** | late binding 导致 Sprite.texture 不指向 sactx |

> 图集只解决 Texture 合并问题，**无法解决 Material 分裂问题**。

### 6.3 图集分 Canvas 策略

来自 [[【笔记】UGUI DrawCall影响因素全面测试]]：

```
Canvas_A ← HUD 图集（血条、技能图标、摇杆）
Canvas_B ← 通用图集（按钮底图、边框、背景）
Canvas_C ← 弹窗图集（确认框、Toast）
```

---

## 七、动态图集

来自 UGUI 第12章的深度解析。

### 7.1 原理

```
运行时：
  小纹理 A ──┐
  小纹理 B ──┼──→ 复制到动态图集 Texture ──→ UV 重映射
  小纹理 C ──┘

  注意：原始纹理仍然存在（不是真正合并）
        有额外的纹理复制开销（CopyTexture）
```

### 7.2 限制条件

以下情况无法进入动态图集：

- 尺寸过大（超过动态图集剩余空间）
- 开启了 Mipmap
- Crunch 压缩格式
- Read/Write 异常
- RenderTexture
- 不同 FilterMode / WrapMode

### 7.3 隐藏代价

> **CPU 开销是最大隐藏问题**——尤其第一次打开 UI 时，大量纹理需要 CopyTexture，可能造成明显卡顿。

动态图集大小限制通常为 1024 或 2048，空间不足时创建新的动态图集。

### 7.4 适用场景

- 无法在构建期确定的用户头像、动态内容
- 少量零散小纹理（< 32×32）
- 不适用于大量纹理或大尺寸纹理

---

## 八、TMP 字体 Atlas 优化

来自 UWA 外部文章（TMP 字体 Atlas 内存优化实战）。

### 8.1 问题：动态字体 Atlas 过大

```
优化前（动态字体方案）：
  动态生成 → 2 张 4096×4096 双实例（CPU+GPU）
  Read/Write 开启 → 内存翻倍
  总计 = 64MB
```

### 8.2 优化方案：静态图集 + Fallback 补字

```
优化后：
  ① 预收集常用字符集 → 静态图集 512×512 = 0.25MB
  ② 大字符集 → 4096×4096 静态 ASTC8x8 = 4MB
  ③ 罕见字 → Fallback 小分辨率动态 Atlas 补字
  总计 ≈ 5MB（从 64MB 降至 5MB）
```

### 8.3 关键手段

| 手段 | 效果 |
|------|------|
| 预收集常用字符 → 静态字体 | 消除运行时动态生成开销 |
| Fallback 用小分辨率动态补充 | 兼顾罕见字与内存 |
| 项目结束后将动态图集转静态 | 发布版零动态开销 |
| 关闭 Read/Write | 省一半 CPU 内存 |
| Multi Atlas Textures | 1 张 4096 → 3 张 2048，节省 8MB |

---

## 九、图集设计原则

来自 UGUI 第22章和 [[【片段】UGUI 性能优化规则清单]]：

### 9.1 按功能模块拆分

```
主界面图集    → 按钮底图、导航栏、菜单背景
商城图集      → 商品图标、价格框、分类标签
战斗图集      → 技能图标、血条、伤害数字
通用图集      → 通用按钮、边框、滑块
活动图集      → 限时活动专属（用完即卸）
```

### 9.2 四大常见问题

| 问题 | 后果 | 解法 |
|------|------|------|
| **图集过多** | 纹理切换频繁，DrawCall 高 | 按模块合并，同界面同图集 |
| **图集过大** | 内存常驻开销大 | 功能图集 ≤1024 |
| **跨图集穿插** | 层级交替打断合批 | 同图集元素排列连续 |
| **动态加载不合理** | 纹理状态变化破坏连续性 | Atlas 先于 Sprite 加载 |

### 9.3 规则清单

来自 [[【片段】UGUI 性能优化规则清单]]：

- **R3.2**：同一界面的 UI 图**必须打到同一图集**；文字 atlas 与图形 atlas 分开
- **R6.2**：界面关闭后释放其专属大纹理；图集共享纹理不释放
- **P1 优先级**：`sharedMaterial` 规范 + 图集合并（低开销、高收益、全项目适用）

---

## 十、常见问题速查表

| 症状 | 根因 | 解法 |
|------|------|------|
| 图集打了但 DrawCall 还是高 | 层级穿插 / Mask 打断 / 字体纹理不同 | Frame Debugger 检查断裂点 |
| 包体出现重复资源 | RawImage 引用了图集 Sprite | RawImage 只用 Texture2D |
| sactx 冗余 | Sprite 和 SpriteAtlas 不在同一 AB | Sprite + Atlas 同 AB |
| 图集占内存大 | 过大图集常驻 + Read/Write 开启 | 按功能拆分 + 关 RW |
| 切界面时卡顿 | 动态图集第一次 CopyTexture | 预加载 + 转静态图集 |
| AB 加载后 DrawCall 暴增 | Atlas 未加载，Sprite 退回原始纹理 | 确保 Atlas 先于 Sprite 加载 |
| 内存只增不减 | 图集被多 UI 引用无法释放 | 界面关闭后 Release 专属图集 |
| 字体 Atlas 内存巨大 | 动态字体双实例 + Read/Write | 静态图集 + Fallback 补字 |

---

## 十一、面试自测

- [ ] 能说出图集减少 DrawCall 的底层原因（纹理切换次数）
- [ ] 能解释 SpriteAtlas 运行时 Texture 替换机制
- [ ] 能背出 sactx 冗余的两种场景规则
- [ ] 知道 RawImage 不能引用图集 Sprite 的原因
- [ ] 能说出图集正确但 DrawCall 仍高的 5 种原因
- [ ] 能解释动态图集的 CPU 开销来源
- [ ] 知道关闭 Read/Write 为什么能省一半内存
- [ ] 能描述 TMP Atlas 从 64MB 降至 5MB 的优化路径
- [ ] 能讲清 SpriteAtlas 与 AB 的 late binding 问题及正确加载顺序

---

## 相关文档

| 文档 | 图集覆盖内容 |
|------|-------------|
| [[【笔记】UGUI性能优化实战总览]] | sactx 冗余规则 / RawImage 陷阱 / 尺寸策略 |
| [[【笔记】UGUI DrawCall影响因素全面测试]] | 图集数量与 DrawCall 量化数据 |
| [[【最佳实践】UI性能优化速查]] | 移动端推荐导入配置 |
| [[【片段】UGUI 性能优化规则清单]] | R3.2 / R6.2 图集规则 |
| [[【笔记】Unity资源依赖与打包陷阱]] | 图集 AB 重复打包根因 |
| [[【设计原理】UGUI合批机制深度解析]] | 合批条件 / 图集与 Material 的关系 |
| [[【踩坑】UGUI常见性能陷阱与根因分析]] | 相同图集但层级穿插不打批 |
| [[UI系统专题索引]] | UI 系统文档导航 |
