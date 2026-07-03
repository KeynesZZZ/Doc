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
    └── Addressables ── [[【教程】资源管线-Addressables]]
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
