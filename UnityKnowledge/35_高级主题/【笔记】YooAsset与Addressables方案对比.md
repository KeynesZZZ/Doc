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
| 断点续传 | 不原生支持 | 原生支持 |
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
