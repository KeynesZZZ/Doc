# YooAsset 知识体系文档设计

> 日期：2026-07-02
> 作者：LLM + 用户协作
> 状态：待审核

## 1. 背景与目标

### 1.1 背景

当前 UnityKnowledge 仓库中：
- **无**专门的 YooAsset 文档，仅在 `100_项目实战/02_休闲游戏框架/README.md` 中作为资源管理方案被引用
- 已有 Addressables 相关文档（`40_工具链/` 和 `30_性能优化/`）
- 已有完整的 HybridCLR 和 tolua 热更新专题（`35_高级主题/`）
- `60_第三方库/` 已有 DOTween、UniTask、Zenject、Odin Inspector 教程

### 1.2 目标

为 UnityKnowledge 仓库补充 YooAsset 完整知识体系，覆盖：
- **体系化深度专题**（方案 B）— 从核心架构到打包管线、热更新、版本管理
- **聚焦实战痛点**（方案 C）— 中大型项目场景的踩坑、分包策略、HybridCLR 协同

### 1.3 项目场景

中大型项目，关注：分包策略、CDN 部署、首包缩小、按需下载、断点续传。

## 2. 目录布局

采用混合放置策略：基础教程放 `60_第三方库/`，热更新实战/踩坑放 `35_高级主题/`，互相用 `[[]]` 链接串联。

```
60_第三方库/
├── 【笔记】YooAsset核心概念与架构.md          ← 架构总览，入口页
├── 【代码片段】YooAsset常用API速查.md          ← 高频代码模板
└── (现有 DOTween / UniTask / Zenject / Odin)

35_高级主题/
├── 【笔记】YooAsset打包管线与分包策略.md       ← Collector/Packer/分包设计
├── 【笔记】YooAsset热更新与版本管理.md         ← 增量更新/断点续传/CDN
├── 【踩坑】YooAsset中大型项目踩坑实录.md       ← 内存/依赖/并发/平台兼容
├── 【笔记】YooAsset与HybridCLR协同方案.md      ← 代码热更+资源热更全链路
├── 【笔记】YooAsset与Addressables方案对比.md   ← 横向评测
├── YooAsset专题索引.md                         ← 串联以上所有文档的导航页
└── (现有 HybridCLR / tolua / 面试等文档)
```

## 3. 跨文档链接策略

- `60_第三方库/` 的两篇文档互链（架构 ↔ API 速查）
- `35_高级主题/` 的 5 篇全部链回架构总览页作为"返回入口"
- HybridCLR 协同篇双向链接到现有的：
  - `[[【踩坑】HybridCLR接入常见坑]]`
  - `[[【笔记】HybridCLR构建管线与Generate工具链]]`
- 方案对比篇双向链接到现有的 Addressables 文档：
  - `[[【教程】资源管线-Addressables]]`
  - `[[【最佳实践】Addressables性能优化]]`
- YooAsset 专题索引页汇总所有 7 篇文档的导航

## 4. 各文档详细大纲

### 文档 1：`【笔记】YooAsset核心概念与架构`

- **定位**：YooAsset 知识体系入口页
- **目录**：`60_第三方库/`
- **大纲**：
  - YooAsset 是什么 — 定位（Unity 资源管理 + 热更新框架）、版本（YooAsset 2.x）、开源协议
  - 核心架构 — Package → Collection → Bundle → Asset 四层模型，数据流描述
  - 三大管线概述：
    - 收集管线（Collector）：项目资源纳入管理
    - 打包管线（Packer）：构建 AssetBundle
    - 加载管线（Loader）：运行时加载和卸载
  - 关键概念速览表：AssetHandle / BundleHandle / Package / Manifest / Operation
  - 与 Addressables 的一句话定位差异（详细对比留到文档 7）
  - 底部：专题索引链接

### 文档 2：`【代码片段】YooAsset常用API速查`

- **定位**：日常开发的代码字典，按场景组织，复制即用
- **目录**：`60_第三方库/`
- **大纲**：
  - 初始化 — `YooAssets.Initialize()`、创建 Package、配置
  - 资源加载 — 同步/异步加载 GameObject / Sprite / TextAsset / AudioClip
  - 加载变体 — 针对 Android/iOS 分辨率变体
  - 实例化与释放 — `InstantiateAsync`、`ReleaseHandle`、引用计数
  - 场景加载 — `SceneManager.LoadSceneAsync`
  - 资源卸载 — 卸载指定 Package、释放未使用资源
  - 文件下载 — `UpdateStaticVersionOperation`、`UpdateManifestOperation`、`DownloaderOperation`
  - 获取资源信息 — `GetAssetInfo`、`GetRawFile`、检查资源是否存在

### 文档 3：`【笔记】YooAsset打包管线与分包策略`

- **定位**：中大型项目的打包设计指南
- **目录**：`35_高级主题/`
- **大纲**：
  - Collector 系统详解：
    - Collector Group / Address Rule / Pack Rule 配置逻辑
    - 常用 Address Rule 选择策略
  - Packer 打包模式：
    - 单独打包 / 共享打包 / 依赖打包 的适用场景
    - Group 共享对依赖冗余的影响
  - 分包策略设计（核心）：
    - 首包最小化策略
    - 按需下载包：关卡 / 活动 / 高清纹理的分包粒度
    - 分包版本独立性
  - 加密方案 — 离线模式 / 内置模式
  - 构建产物结构 — BuildReport 解读、Manifest 文件作用
  - 实操示例：中大型项目的 Collector + Packer 配置实例

### 文档 4：`【笔记】YooAsset热更新与版本管理`

- **定位**：从 0 到 1 搭建热更新流程的完整指南
- **目录**：`35_高级主题/`
- **大纲**：
  - 热更流程全链路图 — 启动 → 检查版本 → 下载 Manifest → 比对差异 → 下载资源 → 切换 Package → 完成
  - 版本号机制 — StaticVersion 与 PackageVersion 的区别
  - 增量更新 — 差异比对算法简述、差异补丁生成
  - 断点续传 — DownloaderOperation 实现
  - CDN 部署：目录结构规范、多节点容灾、灰度发布
  - 离线模式与在线模式切换
  - 完整代码示例：热更检查 + 下载 + 进度展示流程
  - 边界情况处理 — 网络中断、磁盘空间不足、版本回退

### 文档 5：`【踩坑】YooAsset中大型项目踩坑实录`

- **定位**：真实项目中遇到的坑和解决方案
- **目录**：`35_高级主题/`
- **大纲**：
  - 内存管理坑：
    - AssetHandle 未释放导致内存泄漏
    - Bundle 依赖引用计数理解偏差
    - Resources.UnloadUnusedAssets 与 YooAsset 的协调
  - 依赖与加载坑：
    - 多 Package 共享资源的依赖冲突
    - Address 规则变化导致 Manifest 不匹配
    - 异步加载时序问题
  - 并发与性能坑：
    - 同帧大量并发加载的性能瓶颈
    - 主线程卡顿：加载回调集中触发
    - Android IL2CPP 的 GC 压力
  - 平台兼容坑：
    - Android 文件路径大小写问题
    - iOS 文件写入权限限制
    - WebGL 不支持本地文件 IO
  - 每个坑结构：现象 → 原因分析 → 解决方案 → 验证方式

### 文档 6：`【笔记】YooAsset与HybridCLR协同方案`

- **定位**：代码热更 + 资源热更的完整落地方案
- **目录**：`35_高级主题/`
- **大纲**：
  - 整体架构 — HybridCLR 负责热更 DLL，YooAsset 负责文件分发和资源管理
  - DLL 热更流程 — YooAsset 下载 DLL → HybridCLR LoadMetadataForAOTAssembly → 完成
  - 版本对齐策略 — 代码版本与资源版本关联管理
  - 打包集成 — HybridCLR 编译产物接入 YooAsset Collector
  - 完整代码示例 — 启动检查到双热更完整流程
  - 踩坑要点 — 链接到现有 HybridCLR 踩坑文档和文档 5
  - 落地 Checklist

### 文档 7：`【笔记】YooAsset与Addressables方案对比`

- **定位**：技术选型参考
- **目录**：`35_高级主题/`
- **大纲**：
  - 架构对比 — 设计哲学、核心抽象差异
  - 工作流对比 — 打包配置、发布流程、调试体验
  - 性能对比 — 加载速度、内存占用、包体大小
  - 热更新对比 — Addressables CCD vs YooAsset CDN
  - 团队成本对比 — 学习曲线、社区生态、文档质量
  - 适用场景推荐表
  - 结论与推荐

### 索引页：`YooAsset专题索引.md`

- **定位**：导航页，仿照 `tolua专题索引.md` 风格
- **目录**：`35_高级主题/`
- **内容**：汇总 7 篇文档导航、阅读顺序建议、知识图谱

## 5. 元数据规范

所有文档统一使用：
- **V2 前缀**：`【笔记】`/`【代码片段】`/`【踩坑】`
- **YAML frontmatter**：
  ```yaml
  ---
  title: 【笔记】YooAsset核心概念与架构
  tags: ["Unity", "第三方库", "YooAsset", "资源管理", "热更新"]
  created: 2026-07-02
  description: (各文档具体描述)
  author: llm
  status: 待验证
  sources:
    - "[YooAsset 官方文档](https://www.yooasset.com/)"
    - "[YooAsset GitHub](https://github.com/tuyoogame/YooAsset)"
  ---
  ```
- **最低 2 个标签**，包含技术域 + 文档类型
- **代码注释使用中文**

## 6. 写作顺序

1. 文档 1（核心概念与架构）— 入口页，其他文档的基础
2. 文档 2（API 速查）— 与文档 1 互链
3. 文档 3（打包管线与分包策略）
4. 文档 4（热更新与版本管理）
5. 文档 5（踩坑实录）
6. 文档 6（HybridCLR 协同方案）
7. 文档 7（方案对比）
8. 索引页 — 汇总所有文档

## 7. 质量标准

- 每篇文档的代码示例可直接参考使用
- 关键概念配有文字版架构图或表格
- 踩坑文档的每个问题都有完整的"现象→原因→解决→验证"四要素
- 与现有文档体系无缝衔接，不产生信息孤岛
- 遵守 LLM-Wiki 模式：`author: llm` + `sources:` 必填
