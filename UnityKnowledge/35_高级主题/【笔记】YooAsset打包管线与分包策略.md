---
title: 【笔记】YooAsset打包管线与分包策略
tags: ["Unity", "YooAsset", "打包管线", "分包策略", "AssetBundle"]
category: 高级主题
created: 2026-07-02
updated: 2026-07-02
description: YooAsset Collector 收集系统、Packer 打包模式、中大型项目分包策略设计、加密方案、构建产物结构解读
unity_version: 2021.3+
status: 待验证
validation: 基于官方文档与工程实践整理
related: ["[[【笔记】YooAsset核心概念与架构]]", "[[【代码片段】YooAsset常用API速查]]", "[[【笔记】YooAsset热更新与版本管理]]", "[[【踩坑】YooAsset中大型项目踩坑实录]]"]
author: llm
sources:
  - "YooAsset 官方文档 https://www.yooasset.com/"
  - "YooAsset AssetBundleCollector 文档 https://www.yooasset.com/docs/guide-editor/AssetBundleCollector"
  - "YooAsset AssetBundleBuilder 文档 https://www.yooasset.com/docs/guide-editor/AssetBundleBuilder"
  - "[[【笔记】YooAsset核心概念与架构]]"
---

# 【笔记】YooAsset 打包管线与分包策略

> 从 Collector 收集配置到 Packer 打包产物，完整拆解 YooAsset 的构建管线，并给出中大型项目的分包策略设计。

## 文档定位

- **概念基础**：[[【笔记】YooAsset核心概念与架构]] 已介绍三大管线的概览，本文深入**收集管线**和**打包管线**的配置细节。
- **实战场景**：面向中大型项目（多模块、分包需求、CDN 部署），给出可落地的配置实例。

---

## 一、Collector 收集系统

### 1.1 Collector Group（收集组）

Collector Group 是 YooAsset 收集系统的核心组织单位。每个 Group 配置：

| 配置项 | 作用 | 示例 |
|--------|------|------|
| Collect Path | 资源源目录 | `Assets/GameRes/UI` |
| Collector Type | 收集类型 | MainAsset（主资源）/ StaticAsset（静态资源，不独立打包） |
| Address Rule | 寻址规则 | AddressByFileName / AddressByGroupPath |
| Pack Rule | 打包规则 | PackSeparately / PackDirectory / PackGroup |
| Filter Rule | 过滤规则 | CollectAll / CollectSprite / CollectScene |
| Active Rule | 激活规则 | ActiveByName / ActiveByGroup |

### 1.2 Address Rule（寻址规则）

寻址规则决定了加载资源时使用的地址字符串：

```csharp
// AddressByFileName — 按文件名寻址（最常用）
// 文件路径: Assets/Prefabs/UI/PlayerPanel.prefab
// 寻址地址: PlayerPanel
package.LoadAssetAsync<GameObject>("PlayerPanel");

// AddressByFilePath — 按完整路径寻址
// 寻址地址: Assets/Prefabs/UI/PlayerPanel.prefab
package.LoadAssetAsync<GameObject>("Assets/Prefabs/UI/PlayerPanel.prefab");

// AddressByGroupPath — 按组路径寻址（适合模块化项目）
// 寻址地址: UI/PlayerPanel
package.LoadAssetAsync<GameObject>("UI/PlayerPanel");
```

### 1.3 Pack Rule（打包规则）

打包规则决定了资源如何被组合进 AssetBundle：

| 规则 | 说明 | 适用场景 |
|------|------|----------|
| PackSeparately | 每个资源单独一个 Bundle | UI Prefab、独立配置表 |
| PackDirectory | 同一目录的资源打成一个 Bundle | 同类纹理、同模块音频 |
| PackGroup | 整个 Group 打成一个 Bundle | 小文件集、共享依赖 |
| PackTopDirectory | 顶层目录分别打包 | 按模块分包 |

### 1.4 Filter Rule（过滤规则）

```csharp
// 只收集 Sprite 类型
// CollectSprite — 过滤 .png/.jpg/.tga 等

// 只收集场景
// CollectScene — 过滤 .unity

// 收集全部
// CollectAll — 不过滤

// 自定义过滤规则（通过 IFilterRule 接口）
public class CollectPrefab : IFilterRule
{
    public bool IsCollectAsset(FileInfo fileInfo)
    {
        return fileInfo.Extension == ".prefab";
    }
}
```

---

## 二、Packer 打包模式

### 2.1 三种构建管线对比

| 构建管线 | 说明 | 优势 | 适用场景 |
|----------|------|------|----------|
| BuiltinBuildPipeline | 模拟 Unity 原生管线 | 兼容性最好 | 老项目迁移 |
| ScriptableBuildPipeline | Unity 官方 SBP | 可定制性强、依赖处理优 | **推荐新项目** |
| RawFileBuildPipeline | 原始文件管线 | 不压缩不打包 | 视频等大文件 |

### 2.2 压缩格式选择

| 压缩格式 | 说明 | 适用场景 |
|----------|------|----------|
| Uncompressed | 无压缩 | 开发调试、WebGL |
| LZ4 | 快速压缩 | **推荐**，加载速度与体积的平衡 |
| LZ4HC | 高压缩 LZ4 | 正式发布 |
| LZMA | 最高压缩比 | 包体敏感场景（加载较慢） |

### 2.3 打包配置示例

```
Build Pipeline: ScriptableBuildPipeline
Package Name: DefaultPackage
Compress Option: LZ4
File Name Style: HashName（使用 Hash 命名，避免路径泄露）
Copy Buildin File Option: None（不内置到首包）
```

---

## 三、分包策略设计（核心）

### 3.1 首包最小化策略

**目标**：将首包体积控制在平台限制内（Android 150MB / iOS 4GB）。

**方案**：

```
DefaultPackage（首包必需）
├── 启动场景
├── 核心 UI 框架
├── 基础角色 Prefab
└── 新手引导资源

ActivityPackage（活动包，按需下载）
├── 活动场景
├── 活动 UI
└── 活动专属资源

HDTexturePackage（高清纹理包，可选）
├── 高清角色贴图
└── 高清场景贴图

LevelPackage_01（关卡分包 1）
├── 关卡 1-50 场景
└── 关卡专属资源
```

### 3.2 分包版本独立性

每个 Package 有独立的版本号，可以独立热更：

```
CDN 目录结构:
/cdn-server/
├── DefaultPackage/
│   ├── v1.0.0/
│   │   ├── manifest.hash
│   │   ├── manifest.bytes
│   │   └── bundles/
│   └── v1.0.1/
│       └── ...（增量补丁）
├── ActivityPackage/
│   ├── v1.0.0/
│   └── v1.1.0/（活动更新，不影响主包）
└── HDTexturePackage/
    └── v1.0.0/
```

### 3.3 分包粒度设计原则

| 分包类型 | 粒度建议 | 理由 |
|----------|----------|------|
| 核心包 | 必需的最小集 | 首包体积控制 |
| 关卡包 | 50 关 / 包 | 玩家不会一次打所有关卡 |
| 活动包 | 一个活动 / 包 | 活动结束可整体卸载 |
| 纹理包 | 按场景或角色 | 高清纹理按需下载 |

### 3.4 分包间依赖处理

```csharp
// 多 Package 加载共享资源的正确姿势
// 方案 A：共享资源放到 DefaultPackage，其他 Package 通过路径引用
var defaultPkg = YooAssets.GetPackage("DefaultPackage");
var handle = defaultPkg.LoadAssetAsync<GameObject>("SharedEffect");

// 方案 B：使用 YooAssets 全局接口（自动路由到正确的 Package）
var handle = YooAssets.LoadAssetAsync<GameObject>("SharedEffect");
```

> **注意**：跨 Package 引用时，地址必须是全局唯一的。详见 [[【踩坑】YooAsset中大型项目踩坑实录]]。

---

## 四、加密方案

### 4.1 加密接口

YooAsset 通过 `IEncryptionServices` 接口支持自定义加密：

```csharp
// 打包时加密
public class FileOffsetEncryption : IEncryptionServices
{
    public EncryptResult Encrypt(EncryptFileInfo fileInfo)
    {
        // 方案 1：文件偏移加密（最简单）
        // 在文件头插入随机字节，加载时跳过
        byte[] fileData = File.ReadAllBytes(fileInfo.FilePath);
        byte[] encryptData = new byte[32 + fileData.Length]; // 32 字节随机头

        // 填充随机字节
        for (int i = 0; i < 32; i++)
            encryptData[i] = (byte)UnityEngine.Random.Range(0, 256);

        // 拷贝原始数据
        Buffer.BlockCopy(fileData, 0, encryptData, 32, fileData.Length);

        return new EncryptResult
        {
            Encrypted = true,
            EncryptedData = encryptData,
            FileOffset = 32  // 加载时需要跳过的偏移量
        };
    }
}
```

### 4.2 三种加载解密方式

| 方式 | 接口方法 | 说明 | 适用场景 |
|------|----------|------|----------|
| 文件偏移 | `LoadFromFileOffset` | 指定偏移量读取 | 简单加密，性能最好 |
| 内存解密 | `LoadFromMemory` | 完整读入内存后解密 | AES 等加密算法 |
| 流式解密 | `LoadFromStream` | 返回解密流 | 大文件 |

> 运行时解密实现见 [[【代码片段】YooAsset常用API速查]] §7。

---

## 五、构建产物结构

### 5.1 产物目录

```
BundlesOutput/
├── DefaultPackage/
│   ├── BuildReport.html          ← 构建报告（人读）
│   ├── BuildReport.json          ← 构建报告（机读）
│   ├── manifest.hash             ← Manifest 文件哈希（用于版本对比）
│   ├── manifest.json             ← Manifest 数据（依赖关系、Bundle 信息）
│   ├── manifest.json.bytes       ← Manifest 二进制版本（运行时加载）
│   ├── Setting.json              ← 构建配置快照
│   └── Bundles/
│       ├── 5f3a2b...bundle       ← AssetBundle 文件（Hash 命名）
│       ├── 8c1d4e...bundle
│       └── ...
```

### 5.2 BuildReport 解读

```json
{
  "BuildResult": {
    "BuildSuccess": true,
    "TotalBundleCount": 156,
    "TotalBundleSize": 67108864,
    "AverageBundleSize": 430000,
    "MaxBundleSize": 5242880,
    "MinBundleSize": 1024
  },
  "AssetList": [
    {
      "Address": "PlayerPanel",
      "AssetPath": "Assets/Prefabs/UI/PlayerPanel.prefab",
      "BundleName": "5f3a2b...bundle",
      "Size": 24576
    }
  ],
  "BundleList": [
    {
      "BundleName": "5f3a2b...bundle",
      "Size": 24576,
      "ReferencedCount": 3,
      "MainAssetSize": 20480,
      "Dependencies": ["8c1d4e...bundle"]
    }
  ]
}
```

**关键指标**：
- `TotalBundleSize` — 总包体大小，直接影响首包体积
- `MaxBundleSize` — 最大单个 Bundle，过大需要考虑拆分
- `ReferencedCount` — 被引用次数，>1 说明被多个 Bundle 共享
- `MainAssetSize` vs `Size` 差值 — 依赖冗余大小

---

## 六、实操示例：中大型项目 Collector 配置

### 6.1 配置总览

| Package | Collector Group | Address Rule | Pack Rule | 说明 |
|---------|-----------------|--------------|-----------|------|
| DefaultPackage | UI/ | AddressByFileName | PackSeparately | UI Panel 独立打包 |
| DefaultPackage | UI/Atlas/ | AddressByFileName | PackDirectory | 图集按目录打包 |
| DefaultPackage | Config/ | AddressByFileName | PackGroup | 配置表打成一个包 |
| DefaultPackage | Core/ | AddressByFilePath | PackSeparately | 核心资源路径寻址 |
| LevelPackage | Levels/ | AddressByFileName | PackSeparately | 关卡资源独立打包 |
| ActivityPackage | Events/ | AddressByFileName | PackDirectory | 活动按目录打包 |

### 6.2 Jenkins 自动化构建

```python
# Python 脚本：调用 Unity 执行 YooAsset 打包
import subprocess
import os

def build_yooasset(unity_path, project_path, package_name, output_path):
    """执行 YooAsset 打包"""
    args = [
        unity_path,
        "-batchmode",
        "-projectPath", project_path,
        "-executeMethod", "BuildPipeline.BuildYooAsset",
        "-packageName", package_name,
        "-outputPath", output_path,
        "-quit"
    ]
    result = subprocess.run(args, capture_output=True, text=True)
    if result.returncode != 0:
        print(f"构建失败: {result.stderr}")
        return False

    # 检查构建产物
    manifest_path = os.path.join(output_path, package_name, "manifest.hash")
    if not os.path.exists(manifest_path):
        print("构建产物不完整")
        return False

    print(f"构建成功: {output_path}/{package_name}")
    return True

# 使用示例
build_yooasset(
    "C:/Program Files/Unity/Hub/Editor/2022.3.20f1/Editor/Unity.exe",
    "D:/MyProject",
    "DefaultPackage",
    "D:/BuildOutput"
)
```

---

## 七、打包优化 Checklist

### Bundle 大小优化

- [ ] 单个 Bundle 不超过 5MB（超过考虑拆分）
- [ ] 共享依赖单独打包（避免冗余）
- [ ] 静态资源（材质、Shader）使用 StaticAsset 类型
- [ ] 使用 LZ4 压缩而非无压缩

### 分包策略优化

- [ ] 首包只包含启动必需资源
- [ ] 关卡/活动按模块分包
- [ ] 高清资源独立分包
- [ ] 分包版本独立可更

### 构建流程优化

- [ ] 构建流程脚本化（Jenkins / Python / Shell）
- [ ] 每次构建自动校验 BuildReport
- [ ] 构建产物与版本号绑定
- [ ] CDN 上传后校验文件完整性

---

## 相关文档

- [[【笔记】YooAsset核心概念与架构]] — 四层模型与三大管线概览
- [[【代码片段】YooAsset常用API速查]] — 加密/解密 API 实现
- [[【笔记】YooAsset热更新与版本管理]] — 打包产物如何用于热更
- [[【踩坑】YooAsset中大型项目踩坑实录]] — 分包相关的坑

## 官方参考

- [YooAsset AssetBundleCollector](https://www.yooasset.com/docs/guide-editor/AssetBundleCollector)
- [YooAsset AssetBundleBuilder](https://www.yooasset.com/docs/guide-editor/AssetBundleBuilder)
- [YooAsset GitHub](https://github.com/tuyoogame/YooAsset)
