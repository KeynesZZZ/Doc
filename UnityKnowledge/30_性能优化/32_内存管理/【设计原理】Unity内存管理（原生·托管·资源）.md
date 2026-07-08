---
title: 【设计原理】Unity内存管理（原生·托管·资源）
tags: ["Unity", "性能优化", "内存管理", "设计原理", "IL2CPP", "GC"]
category: 性能优化/内存管理
created: 2026-07-08
description: Unity 内存架构原理，涵盖官方三层管理模型、C#↔C++ 桥接机制、Asset 引用图、IL2CPP 编译布局、碎片化与不归还根因
unity_version: 2021.3+
status: 待验证
validation: 未经测试
related:
  - "[[【设计原理】GC工作原理深度解析]]"
  - "[[【最佳实践】资源卸载指南]]"
  - "[[【踩坑】内存泄漏模式]]"
  - "[[【实战案例】内存泄漏排查实战]]"
author: llm
sources:
  - "Unity Memory in Unity introduction. Unity 6 官方文档. https://docs.unity3d.com/6000.4/Documentation/Manual/performance-memory-overview.html"
  - "Unity Profiler counters reference. Unity 6000.3 官方文档. https://docs.unity3d.com/6000.3/Documentation/Manual/profiler-counters-reference.html"
  - "Unity Memory Profiler module. Unity 2022.3 官方文档. https://docs.unity3d.com/2022.3/Documentation/Manual/ProfilerMemory.html"
  - "Unity iOS Hardware Guide — Unified Memory Architecture. Unity 官方文档. https://docs.unity3d.com/460/Documentation/Manual/iphone-Hardware.html"
  - "Unity Asynchronous Texture Upload. Unity 2019.2 官方文档. https://docs.unity3d.com/2019.2/Documentation/Manual/AsyncTextureUpload.html"
  - "Memory Profiler — Memory usage on devices. Unity 包文档. https://docs.unity3d.com/Packages/com.unity.memoryprofiler@1.1/manual/memory-on-device.html"
  - "IL2CPP Il2CppObject 源码定义. il2cpp-object-internals.h. https://github.com/dreamanlan/il2cpp_ref/blob/master/libil2cpp/il2cpp-object-internals.h"
  - "Boehm-Demers-Weiser Garbage Collector 源码. GitHub. https://github.com/bdwgc/bdwgc"
  - "Jamin. Unity IL2CPP的GC原理. UWA (侑虎科技), 2024. https://mp.weixin.qq.com/s?__biz=MzI3MzA2MzE5Nw==&mid=2668943544&idx=1&sn=a036f9d59db85e299938903d542f01f8"
  - "[[【设计原理】GC工作原理深度解析]] — Boehm GC 内部数据结构与分配流程"
---

# 【设计原理】Unity内存管理（原生·托管·资源）

> 核心价值：建立 Unity 内存的正确心智模型——三个区域、两层桥接、一套生命周期

## 文档定位

本文从**架构原理**层面解析 Unity 内存管理，回答三个核心问题：
1. Unity 的内存到底分几个区域？各自如何分配和回收？
2. C# 对象和引擎 C++ 对象之间的桥接机制是什么？
3. 为什么 Unity 内存会"只增不减"？

> **与现有文档的关系**：本文聚焦原理层。Boehm GC 的 mark-sweep 算法和分配器底层细节见 [[【设计原理】GC工作原理深度解析]]；资源卸载的实践操作见 [[【最佳实践】资源卸载指南]]；GC 优化清单见 [[【最佳实践】GC优化清单]]。

---

## 一、Unity 内存管理模型

> **来源**：Unity 6 官方文档将内存划分为三个管理层（Memory Management Layers）[^UnityMemoryOverview]。以下基于官方模型展开。

Unity 官方将进程内存分为**三个管理层**（注意：不是"堆"的划分，而是**管理机制**的划分）：

```
┌──────────────────────────────────────────────────────────────────┐
│                       Unity 进程内存                              │
├────────────────────┬──────────────────────┬─────────────────────┤
│  Managed Memory    │ C# Unmanaged Memory  │  Native Memory      │
│  (托管内存)         │ (C# 非托管内存)       │  (原生内存)          │
├────────────────────┼──────────────────────┼─────────────────────┤
│ ● Managed Heap     │ ● NativeArray<T>     │ ● 引擎 C++ 核心      │
│   (GC 管理的堆)     │ ● NativeList<T>      │ ● Asset 数据         │
│ ● Scripting Stack  │ ● UnsafeUtility      │   (Texture/Mesh/    │
│   (脚本栈)          │   .Malloc / Free     │    Audio/Anim)      │
│ ● Native VM Memory │ ● Collections 包     │ ● 子系统              │
│   (VM 运行时元数据)  │                      │   (物理/渲染/动画)   │
│                    │                      │ ● 图形/GPU 资源       │
│                    │                      │   (Profiler 中用     │
│                    │                      │    Gfx Used/Reserved │
│                    │                      │    Memory 追踪)      │
├────────────────────┼──────────────────────┼─────────────────────┤
│ Boehm GC 自动管理  │ 手动管理(Dispose)     │ 引擎手动管理          │
│ 非分代、非压缩      │ 不受 GC 管理          │ Destroy/UnloadUnused │
│                    │ 绕过 GC              │ Assets 引擎内部释放   │
├────────────────────┼──────────────────────┼─────────────────────┤
│ 日常可控但受限      │ DOTS/ECS 的核心      │ 占大头(50%+)         │
│                    │                      │ 通常不可通过 C# 访问  │
└────────────────────┴──────────────────────┴─────────────────────┘
```

**关键澄清**：

| 常见社区说法 | Unity 官方术语 |
|-------------|---------------|
| "Native Heap"（社区常用说法） | **Native Memory**（官方术语）— 官方不用 "Heap" 这个词 |
| "GPU Memory"（显存）作为顶层分类 | GPU/图形内存属于 **Native Memory** 的一部分，Profiler 用 **Gfx Used/Reserved Memory** 追踪 |
| 三分法（Native/Managed/GPU） | 官方三层管理模型：**Managed / C# Unmanaged / Native** |

> **本文后续使用说明**：为了便于理解，本文有时仍会使用 "托管堆"(Managed Heap) 指代 Managed Memory 中的 GC 堆部分，使用 "原生内存" 指代 Native Memory。这些是社区通用说法，但读者应知道官方的精确术语。

### 1.1 移动平台的统一内存架构（UMA）

> **来源**：Unity iOS Hardware Guide："Both the CPU and GPU on the iPhone/iPad share the same memory" [^iOSHardware]。Memory Profiler 文档也区分了"有独立 VRAM 的系统"和 UMA 系统 [^MemoryProfilerPkg]。

在移动平台（iOS / 大多数 Android）上，CPU 和 GPU **物理上共享同一块系统内存**（统一内存架构，UMA）。

```
PC / 主机（独立显存）：
├─ GPU 有独立 VRAM
├─ 纹理上传 = 从系统内存复制到 VRAM（PCIe 传输）
└─ CPU 和 GPU 各有一份物理副本

移动平台 (iOS/Android UMA)：
├─ CPU 和 GPU 共享同一物理内存
├─ 纹理"上传"= 建立虚拟地址映射，无大规模物理复制
├─ 纹理只占一份物理内存
└─ Profiler 中 "Gfx Used Memory" 是估算值（非精确），
   官方文档："Unity doesn't have access to information on the
   exact usage of graphics resources" [^MemoryProfilerPkg]
```

**Read/Write Enabled 的内存影响**：

> **来源**：Memory Profiler 模块文档："Read/Write enabled graphics assets need to store a copy in CPU-accessible memory, which doubles their total memory usage" [^ProfilerMemory]。

开启 `Texture.isReadable = true`（Read/Write Enabled）会保留一份 CPU 侧可读取副本，**翻倍**该纹理的内存占用。非 Read/Write 纹理使用**异步纹理上传**（Async Texture Upload），在渲染线程上分时间片上传到 GPU [^AsyncTexture]。

### 1.2 纹理何时上传 GPU

> **来源**：Unity Asynchronous Texture Upload 文档 [^AsyncTexture]。

```
非 Read/Write 纹理（默认推荐）：
├─ 构建时数据存储在 resS（Streaming Resource）文件中
├─ 运行时通过异步纹理上传在渲染线程分时间片上传
├─ 纹理在首次需要渲染时上传
└─ AwakeFromLoad 调用时保证纹理已可用

Read/Write 启用的纹理：
├─ 不使用异步上传
├─ 使用同步方法加载
├─ 保留 CPU 侧可读取副本（内存翻倍）
└─ 仅在需要运行时读写像素数据时开启
```

---

## 二、Native Memory（原生内存）

### 2.1 原生内存中有什么

Unity 引擎本身是 C++ 应用，所有引擎内部对象分配在原生内存中。Unity 官方指出："In most situations, you can't access this memory through your C# code, but because it's usually the biggest chunk of your application's memory footprint" [^UnityMemoryOverview]。

```
Native Memory 主要内容：
├─ 场景图（GameObject、Transform 层级）
├─ Asset 原始数据
│   ├─ 纹理像素数据（未上传 GPU 的部分）
│   ├─ Mesh 顶点/索引数组
│   ├─ AudioClip PCM 数据
│   └─ AnimationClip 关键帧数据
├─ 子系统内部数据
│   ├─ 物理引擎（PhysX / Box2D 世界数据）
│   ├─ 动画系统（Animator 状态机）
│   ├─ 粒子系统（ParticleSystem 模拟数据）
│   └─ 导航系统（NavMesh 寻路数据）
├─ 渲染管线
│   ├─ DrawCall 队列
│   ├─ 光照贴图
│   └─ 静态/动态批处理数据
└─ 第三方原生库
    ├─ FMOD / Wwise（音频引擎）
    ├─ Lua VM（如使用 xLua/sLua）
    └─ 平台 SDK（iOS/Android 原生插件）
```

### 2.2 原生内存的分配与回收

原生内存**不受 GC 管理**，Unity 引擎内部通过 C++ 的手动分配/释放管理：

```
分配：引擎内部 malloc / new
回收：Destroy() / Resources.UnloadUnusedAssets() / 场景卸载

关键 API 对应关系：
├─ Instantiate(prefab)
│   ├─ Native Memory: +新的 C++ GameObject 及 Component 链
│   └─ Managed Heap: +C# 包装器（见第三节）
│
├─ Destroy(obj)
│   ├─ Native Memory: 标记销毁 → 引擎在安全时机释放 C++ 内存
│   └─ Managed Heap: C# 包装器等 GC 回收（延迟）
│
├─ Resources.UnloadUnusedAssets()
│   ├─ 扫描所有已加载 Asset 的引用
│   ├─ 无引用的 Asset → 从 Native Memory 释放
│   └─ 耗时操作（可能几十 ms），适合在 Loading 时调用
│
└─ SceneManager.UnloadSceneAsync()
    ├─ 释放该场景的 C++ 对象
    └─ 不自动释放跨场景共享的 Asset（需配合 UnloadUnusedAssets）
```

> **注意**：`Destroy()` 是**延迟执行**的。引擎在调用后的安全时机（通常是当前帧结束）才释放 C++ 内存。这是引擎内部的设计选择，避免在游戏逻辑执行过程中释放正在使用的对象。

---

## 三、Managed Heap（托管堆）

### 3.1 托管堆的核心特性

> **来源**：Boehm-Demers-Weiser GC 官方仓库 [^BDWGC]；GC 内部分配器细节见 [[【设计原理】GC工作原理深度解析]]。

Unity 使用 **Boehm-Demers-Weiser (BDW) GC** 管理托管堆，无论使用 Mono 还是 IL2CPP 后端。核心特性：

| 特性 | Boehm GC（Unity 默认） | .NET CoreCLR 分代 GC |
|------|----------------------|---------------------|
| 算法 | Mark-Sweep（标记-清除） | Mark-Sweep-Compact（标记-清除-压缩） |
| 分代 | **否** | 是（Gen0/Gen1/Gen2） |
| 压缩 | **否** | 是 |
| 扫描类型 | 保守式 | 精确式 |
| 单次 GC | Full GC（全堆扫描） | Gen0 极快（~0.1ms），Gen2 慢 |
| 堆碎片 | **产生碎片** | 压缩消除碎片 |

> **常见误解纠正**：Unity 的 Boehm GC **不是分代 GC**，没有 Gen0/Gen1/Gen2。每次 GC 都是 Full GC，必须扫描整个堆。

### 3.2 托管堆的分配/回收机制

```
分配（C# new 关键字）：
├─ 小对象（≤2048B）→ GRANULE(16B) 粒度分配 → 查 ok_freelist
├─ 大对象（>2048B）→ 以 4KB hblk 为单位分配
└─ 空间不足 → 触发 GC → 回收垃圾 → 仍不足 → 扩展堆

回收（GC 触发）：
├─ Mark 阶段：从根对象遍历所有引用，标记可达对象
├─ Sweep 阶段：释放未标记对象的内存（加入 freelist）
└─ 无 Compact 阶段：对象地址不变，不整理碎片
```

> GC 内部数据结构（GRANULE / ok_freelist / hblkfreelist / size_map）的详细分析见 [[【设计原理】GC工作原理深度解析]] 第五节。

### 3.3 不压缩导致的碎片化

**碎片化的形成**（逐步推演）：

```
初始状态（紧凑排列）：
[A][A][B][C][C][C][D][____free____]

GC 回收 B 和 D 后（Mark-Sweep，不移动）：
[A][A][__free__][C][C][C][__free__][__free__]

问题：请求分配 3-GRANULE（48B）的连续块
├─ 索引 1-2 处有 2-GRANULE 空闲 → 不够
├─ 索引 6-7 处有 2-GRANULE 空闲 → 不够
├─ 总空闲 = 4 GRANULE (64B) > 48B，但没有连续 3-GRANULE
└─ 分配失败 → 触发 GC → 仍无连续块 → 扩展堆
```

**碎片化的长期趋势**：

```
T=0     启动：堆 10MB（紧凑），实际使用 8MB，碎片率 0%
T=60s   堆 12MB，实际 8MB，碎片率 ~33%
T=10min 堆 18MB，实际 8MB，碎片率 ~55%
T=30min 堆 25MB，实际 8MB，碎片率 ~68%

恶性循环：
碎片增多 → 分配失败 → 扩展堆 → 堆变大 → Full GC 更慢 → 卡顿更严重
```

### 3.4 GC 回收的内存为什么"不归还 OS"

> **来源**：BDW GC 源码中存在 `USE_MUNMAP` 宏控制的内存归还机制 [^BDWGC]。但在 Unity 的实际配置中，归还条件极其苛刻。

```
Boehm GC 的内存归还链：

Sweep 阶段释放对象 → 内存进入 freelist（不是还给 OS）
                           ↓
某个 hblk(4KB) 上所有对象都被回收 → 理论上可整块归还
                           ↓
                    但实际很少发生：
                    ├─ 不压缩导致存活对象散布在各 hblk
                    ├─ 大部分 hblk 有零星存活对象 → 无法整块归还
                    └─ 只有场景切换等大清理后才可能批量释放

即使启用了 USE_MUNMAP：
├─ 归还以 hblk(4KB) 为最小单位
├─ 需要系统调用(munmap/VirtualFree)，有内核开销
├─ Boehm GC 倾向于保留内存供复用，而非频繁归还再申请
└─ 结果：托管堆总大小呈"只增不减"趋势
```

**为什么 Boehm GC 选择不压缩**：

```
根因：保守式 GC 无法安全地移动对象

精确式 GC（如 .NET CoreCLR）：
├─ 知道每个指针的确切类型和位置
├─ 移动对象后可以更新所有引用 → 安全
└─ 需要编译器生成 GC 信息表

保守式 GC（Boehm GC）：
├─ 不区分"指针"和"恰好长得像指针的整数值"
├─ 如果移动对象并更新"指针" → 可能破坏原本是数据的值
├─ 选择：不移动对象 → 不需要更新引用 → 安全但碎片化
└─ 这就是 "conservative"（保守）的含义：宁可浪费内存，不冒数据损坏风险

IL2CPP 的局限：
├─ IL2CPP 理论上可以生成精确类型信息
├─ 但 Boehm GC 设计假设不需要类型信息即可工作
├─ 改造为精确式需要大规模重写 GC 内部逻辑
└─ Unity 官方在探索但尚无明确时间线
```

---

## 四、C# 对象与 C++ 对象的桥接

### 4.1 双实体模型

每个继承自 `UnityEngine.Object` 的 C# 对象在内存中都有**两个实体**：一个 C# 包装器（Managed Heap）和一个 C++ 原生对象（Native Memory）。

```
┌─ Managed Heap ──────────────────────┐
│  C# 包装器（约 40-60 bytes）          │
│  ┌──────────────────────────┐        │
│  │ m_CachedPtr (IntPtr)     │───────┼──→ 指向 Native Memory
│  │ m_InstanceID (int)       │        │    上的 C++ 对象
│  └──────────────────────────┘        │
└──────────────────────────────────────┘

┌─ Native Memory ────────────────────────────────────────┐
│  C++ 原生对象（数百字节 ~ 数 KB）                      │
│  ┌─────────────────────────────────────────┐         │
│  │ 名称、层级、组件列表、引擎内部状态          │         │
│  └─────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────┘
```

**关键认识**：Profiler 中看到的一个 GameObject 占了几 KB，绝大部分在 Native Memory 上。C# 包装器只是一个小指针壳。

### 4.2 IL2CPP 对象的内存布局

> **来源**：IL2CPP 源码 `il2cpp-object-internals.h` 中的结构体定义 [^Il2CppObject]。以下基于 64-bit 平台。

```c
// IL2CPP 源码中的实际定义（简化）
struct Il2CppObject {
    union {
        Il2CppClass *klass;     // 类型信息指针（8 bytes on 64-bit）
        Il2CppVTable *vtable;   // 或虚函数表指针
    };
    MonitorData *monitor;       // 线程同步信息（8 bytes on 64-bit）
};
// 对象头总计：16 bytes（64-bit）/ 8 bytes（32-bit）
```

**class vs struct 的内存差异**：

```
struct Vector3Data { float x, y, z; }
→ 纯数据，sizeof = 12 bytes，无额外开销
→ 嵌入在宿主对象内部或分配在栈上

class EnemyData { float x, y, z; string name; }
→ Il2CppObject 头(16B) + x/y/z(12B) + name 指针(8B) + 对齐(2B) ≈ 40+ bytes
→ 分配在 Managed Heap，受 GC 管理

存储 1000 个 3D 位置：
├─ class 数组：1000 × (40+8) ≈ 48KB，非连续内存，GC 管理 1000 个对象
└─ struct 数组：1000 × 12 = 12KB，连续内存，缓存友好，无 GC 开销
```

> 这就是 DOTS/ECS 选择 struct + NativeArray 的根本原因：连续内存布局带来缓存友好性，且绕过 GC。

### 4.3 "Fake Null" 问题

```csharp
GameObject enemy = Instantiate(prefab);
Destroy(enemy);

// enemy == null 输出 true（Unity 重载了 == 运算符）
// 但 C# 包装器对象仍存在于 Managed Heap 上
```

```
Destroy(enemy) 后的时间线：

T+0帧: Destroy() 调用
       ├─ C++ 侧：标记 m_IsDestroying = true
       ├─ C# 侧：enemy == null → Unity 重载 == 检查 C++ 状态 → 返回 true
       └─ C# 包装器仍引用 C++ 地址（此刻 C++ 对象可能尚未释放）

T+0帧结束或 T+1帧: 引擎释放 C++ 对象
       ├─ Native Memory: C++ 内存被释放
       ├─ m_CachedPtr 现在指向已释放的内存
       └─ 此时访问 enemy.transform → 行为未定义

T+???帧: GC 终于运行
       ├─ 回收 C# 包装器
       └─ m_CachedPtr 随之消失
```

**实践原则**：`Destroy()` 后立即将引用置为 `null`，帮助 GC 更快识别为可回收。

### 4.4 跨语言调用的内存开销

```csharp
// 这行代码的完整调用链：
Rigidbody rb = GetComponent<Rigidbody>();

C# GetComponent<T>()
  → IL2CPP 生成的 C++ 包装
    → C++ 层遍历组件链表
    → 找到 C++ Rigidbody 对象
    → 检查缓存表：此 C++ 指针是否已有 C# 包装器？
      → 有 → 返回缓存的 C# 对象（零分配）
      → 无 → 在 Managed Heap 新建 C# 包装器
  → 返回 C# Rigidbody 引用
```

这就是为什么应缓存组件引用——不仅省 CPU，还避免重复创建 C# 包装器。

---

## 五、Asset 引用图与内存共享

### 5.1 Asset 与 Instance 的内存关系

```
一个 Enemy.prefab 引用的 Asset：

Enemy.prefab
├─ Mesh: EnemyMesh (5000 verts)    → ~300KB
├─ Material: EnemyMat
│   ├─ Shader: Standard
│   ├─ Texture: Diffuse (2048²)    → ~16MB (RGBA32)
│   └─ Texture: Normal (2048²)     → ~16MB
├─ AnimationClip[] (5 clips)       → ~2MB
└─ AudioClip: Roar (3 sec)         → ~300KB

Instantiate(prefab) 后的内存模型：

          ┌──────────────────┐
          │  Asset (共享)     │  ← 只加载一次
          │  Mesh: 300KB     │
          │  Texture: 32MB   │  ← 所有 Instance 共享，不复制
          │  AnimClip: 2MB   │
          │  Audio: 300KB    │
          └────────┬─────────┘
                   │ 共享引用
     ┌─────────────┼─────────────┐
     ↓             ↓             ↓
┌─────────┐  ┌─────────┐  ┌─────────┐
│Instance1│  │Instance2│  │Instance3│
│ ~2-5KB  │  │ ~2-5KB  │  │ ~2-5KB  │  ← 每个 Instance 独立的 C++ + C# 对象
└─────────┘  └─────────┘  └─────────┘

1000 个 Enemy 实例 ≈ 35MB(Asset) + 1000×5KB ≈ 40MB
                    不是 1000 × 35MB
```

**关键认识**：Instantiate 共享 Asset 数据，只创建场景实例。同理，Destroy 实例不会释放 Asset。

### 5.2 正确的资源卸载顺序

```
常见错误（导致内存泄漏或粉色材质）：

错误1：先 UnloadUnusedAssets，再 Destroy 实例
├─ 实例还引用 Asset → UnloadUnusedAssets 认为 Asset "仍在使用" → 不释放
└─ 结果：什么都没卸载

错误2：Destroy 实例后立即 UnloadUnusedAssets
├─ C++ 对象已释放，但 C# 包装器等 GC 回收
├─ GC 未运行 → C# 引用仍在 → UnloadUnusedAssets 认为 Asset 仍被引用
└─ 结果：Asset 无法释放

正确顺序：
1. Destroy 所有实例
2. 将所有 C# 引用置为 null
3. GC.Collect()              → 强制回收 C# 包装器，解除引用
4. Resources.UnloadUnusedAssets()  → 扫描确认无引用后释放 Asset
5.（可选）AssetBundle.Unload(true)
```

### 5.3 AssetBundle 跨包引用陷阱

```
BundleA "core.bundle":
└─ SharedTexture.png

BundleB "level1.bundle":
└─ Level1.prefab → 引用 SharedTexture（跨 bundle 依赖）

如果先卸载 BundleA.Unload(true)：
├─ SharedTexture 被释放
├─ Level1.prefab 的材质引用 SharedTexture → 变成野指针
└─ 下次渲染 → 粉色材质（Shader fallback）或渲染异常

这就是 Addressables 存在的价值：
├─ 手动管理 AssetBundle 依赖极易出错
├─ Addressables 通过引用计数自动管理依赖
├─ 只有所有引用者都释放后，才真正卸载
└─ 但 Addressables 本身也有内存开销（引用图 + 计数表）
```

---

## 六、完整内存生命周期

追踪一行代码从加载到释放的全过程：

```
步骤1: AssetBundle.LoadFromFile("enemies.bundle")
  Native Memory:  +AB 头信息 + 压缩数据解压
  Managed Heap:   +C# AssetBundle 包装器(~60B)

步骤2: bundle.LoadAsset<GameObject>("Enemy")
  Native Memory:  +Mesh/Material/Texture/AnimClip/AudioClip 原始数据(~35MB)
  Managed Heap:   +C# 包装器(几百 bytes)
  Gfx Memory:     (非 Read/Write 纹理可能通过异步上传稍后到 GPU)

步骤3: Instantiate(prefab)
  Native Memory:  +C++ GameObject + Component 链(~2-5KB)
  Managed Heap:   +每个 MonoBehaviour 的 C# 包装器
  Gfx Memory:     +纹理上传(如果之前没传) + DrawCall 常量缓冲
  注意: Mesh/Texture/Material 不复制（共享 Asset）

步骤4: 使用（Update 等逻辑帧）
  Managed Heap:   每帧临时分配（List、字符串拼接等）
  → 这些是未来 GC 的对象，但此刻堆积在堆上

步骤5: Destroy(enemy)
  Native Memory:  标记销毁 → 引擎在帧末释放 C++ 对象(~2-5KB)
  Managed Heap:   C# 包装器等 GC 回收（此刻仍在，fake null）
  Gfx Memory:     DrawCall 注册移除（共享 Asset 仍在 GPU）

步骤6: bundle.Unload(true)
  Native Memory:  释放 AssetBundle 本身 + 所有从该 bundle 加载的 Asset
                  (前提：步骤5的实例确实已全部 Destroy)
  Managed Heap:   C# 包装器等 GC
  Gfx Memory:     纹理从 GPU 卸载（前提：无其他引用）

步骤7: GC.Collect() + Resources.UnloadUnusedAssets()
  Native Memory:  扫描并释放无引用的残留 Asset
  Managed Heap:   Boehm GC Full GC（Mark-Sweep）
    → 回收包装器和临时对象
    → 不压缩（碎片保留）
    → 不归还 OS（空闲块进入 freelist）
```

---

## 七、IL2CPP 的额外内存开销

> **来源**：Unity IL2CPP 文档 [^IL2CPP]；社区逆向工程资料 [^Il2CppReverse]。

```
IL2CPP 比 Mono 多出的常驻内存：

1. 类型信息表（Il2CppClass）
   ├─ 每个 C# 类型生成对应的 C++ 类型信息结构
   ├─ 包含方法表、字段布局、接口映射、GC 引用图
   ├─ 常驻 Native Memory，进程生命周期内不可释放
   └─ 项目越大，类型越多，占用越大

2. 全局元数据（Global-Metadata.dat）
   ├─ IL2CPP 序列化的所有 C# 类型/方法/字段信息
   ├─ 运行时反射依赖这些数据
   ├─ 典型大小：10-50 MB（与项目规模正相关）
   ├─ 可通过 Player Settings → Managed Stripping Level 减小
   └─ Low/Medium/High 级别影响保留多少类型信息

3. 代码段（Text Segment）
   ├─ IL2CPP 把所有 C# 方法编译为 C++ → 机器码
   ├─ 全量 AOT 编译，比 Mono JIT 产生更多代码
   ├─ 常驻内存，不可释放
   └─ 大项目可达几十 MB

4. GC 元数据
   ├─ Boehm GC 扫描需要的辅助数据
   ├─ 标记哪些内存区域需要扫描
   └─ 与 IL2CPP 类型信息配合
```

---

## 八、Profiling：如何观测各区域

> **来源**：Unity Profiler counters reference（Unity 6000.3 官方文档）[^ProfilerCounters]。

```
Unity Profiler > Memory 模块关键计数器（官方名称）：

┌───────────────────────────────────────────────────────────┐
│  官方计数器名（Unity 6 / 6000.3）                          │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  Total Used Memory      ← 应用使用的总内存                 │
│  Total Reserved Memory  ← OS 为应用保留的总内存            │
│  System Used Memory     ← OS 报告的常驻内存总量             │
│                                                           │
│  GC Used Memory         ← 托管堆实际使用量（GC 管理）       │
│  GC Reserved Memory     ← 托管堆保留总量（含空闲和碎片）    │
│  GC Allocated In Frame  ← 当帧托管分配字节数               │
│                                                           │
│  Gfx Used Memory        ← 驱动使用的 GPU 估算内存           │
│  Gfx Reserved Memory    ← 驱动保留的 GPU 估算内存           │
│  Texture Memory         ← 已加载纹理的内存                 │
│  Mesh Memory            ← Mesh 数据                       │
│  Audio Used/Reserved    ← 音频系统估算内存                 │
│                                                           │
└───────────────────────────────────────────────────────────┘

⚠️ 注意事项：
├─ Gfx Used/Reserved Memory 是"估算值"
│   官方："Unity doesn't have access to information on the
│   exact usage of graphics resources" [^MemoryProfilerPkg]
├─ Texture Memory 与 Gfx Used Memory 不是直接映射关系
│   "Texture and Mesh memory doesn't map directly to the
│   Graphics & Graphics Driver stat" [^ProfilerMemory]
└─ GC Allocated In Frame 仅在 Editor/Development Build 可用

碎片判断方法：
碎片率 ≈ (GC Reserved Memory - GC Used Memory) / GC Reserved Memory

经验阈值（仅参考，需结合实际项目）：
├─ < 20%    健康
├─ 20-40%   可接受
├─ 40-60%   需要关注
└─ > 60%    碎片严重，长时间运行有 OOM 风险
```

**通过代码获取内存指标**：

```csharp
// Unity Profiler API 获取内存数据
long totalAllocated = Profiler.GetTotalAllocatedMemoryLong();   // ≈ Total Used
long totalReserved  = Profiler.GetTotalReservedMemoryLong();    // Total Reserved
long unusedReserved = Profiler.GetTotalUnusedReservedMemoryLong();
long gcMemory       = System.GC.GetTotalMemory(false);          // ≈ GC Used

// Unity 2020+ 可使用 ProfilerRecorder 获取精确计数器
// 参考: https://docs.unity3d.com/6000.3/Documentation/Manual/profiler-counters-reference.html
```

---

## 九、Unity 内存管理的核心矛盾

```
根本矛盾：引擎(C++手动管理) × 脚本(C# GC管理) × Boehm GC(保守式)

保守式 GC 的连锁代价：
├─ 不能精确移动对象 → 不压缩 → 碎片化 → 堆膨胀
├─ 不能快速回收新生代 → 每次 Full GC → 卡顿
├─ 不归还 OS → 托管堆只增不减 → 内存虚高
└─ 保守扫描 → 偶尔误判（非指针被当作引用，阻止回收）

开发者的实际处境：
├─ 必须手动管理 Asset 生命周期（C++ 风格）
├─ 必须避免 Managed Heap 分配（因为 GC 不可靠）
├─ 等于同时承受 C++ 和 GC 的复杂性
└─ 这就是 Unity 内存优化"难"的根本原因

应对策略优先级：
1. 不分配 > 减少分配 > 高效回收
   └─ 对象池、预分配容器、值类型优先、零分配热路径

2. 控制 Asset 生命周期
   └─ 正确的加载/卸载顺序，Addressables 引用计数

3. 在安全时机主动清理
   └─ 场景切换/Loading 时 GC.Collect() + UnloadUnusedAssets()

4. 终极方案：DOTS/ECS
   └─ struct + NativeContainer 绕过 Managed Heap
   └─ 手动管理生命周期(Dispose)，可控但需要纪律
```

---

## 相关链接

- [[【设计原理】GC工作原理深度解析]] ← Boehm GC 内部数据结构与分配器底层
- [[【最佳实践】资源卸载指南]] ← 资源卸载的实践操作和代码示例
- [[【最佳实践】GC优化清单]] ← GC 优化检查清单
- [[【踩坑】内存泄漏模式]] ← 常见内存泄漏模式汇总
- [[【实战案例】内存泄漏排查实战]] ← 480MB→120MB 排查实战
- [[../../35_高级主题/【设计原理】Unity内存管理]] ← 代码示例版（含对象池、纹理优化等实践代码）

---

## 参考文献

[^UnityMemoryOverview]: Memory in Unity introduction. Unity 6 官方文档. https://docs.unity3d.com/6000.4/Documentation/Manual/performance-memory-overview.html

[^ProfilerCounters]: Profiler counters reference. Unity 6000.3 官方文档. https://docs.unity3d.com/6000.3/Documentation/Manual/profiler-counters-reference.html

[^ProfilerMemory]: Memory Profiler module. Unity 2022.3 官方文档. https://docs.unity3d.com/2022.3/Documentation/Manual/ProfilerMemory.html

[^MemoryProfilerPkg]: Memory usage on devices. Memory Profiler 包文档. https://docs.unity3d.com/Packages/com.unity.memoryprofiler@1.1/manual/memory-on-device.html

[^iOSHardware]: Unity iOS Hardware Guide. Unity 官方文档. https://docs.unity3d.com/460/Documentation/Manual/iphone-Hardware.html

[^AsyncTexture]: Asynchronous Texture Upload. Unity 2019.2 官方文档. https://docs.unity3d.com/2019.2/Documentation/Manual/AsyncTextureUpload.html

[^Il2CppObject]: Il2CppObject 结构体定义. il2cpp-object-internals.h. https://github.com/dreamanlan/il2cpp_ref/blob/master/libil2cpp/il2cpp-object-internals.h

[^BDWGC]: Boehm-Demers-Weiser Garbage Collector. GitHub 仓库. https://github.com/bdwgc/bdwgc

[^IL2CPP]: Introduction to IL2CPP. Unity 官方文档. https://docs.unity3d.com/6000.4/Documentation/Manual/il2cpp-introduction.html

[^Il2CppReverse]: IL2CPP Reverse Engineering Guide. https://github.com/jadis0x/il2cpp-reverse-engineering-guide

[^UWA_GC]: Jamin. Unity IL2CPP的GC原理. UWA (侑虎科技), 2024. https://mp.weixin.qq.com/s?__biz=MzI3MzA2MzE5Nw==&mid=2668943544&idx=1&sn=a036f9d59db85e299938903d542f01f8

---

*创建日期: 2026-07-08*
*相关标签: #内存管理 #设计原理 #IL2CPP #GC #性能优化*
