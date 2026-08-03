---
title: 【设计原理】GC工作原理深度解析
tags: ["Unity", "性能优化", "内存管理", "设计原理", "GC", "垃圾回收", "IL2CPP"]
category: 性能优化/内存管理
created: "2026-03-05 18:00"
updated: "2026-07-07 00:00"
description: Unity Boehm GC 的真实工作原理，包含 mark-sweep 算法、非分代非压缩特性、触发条件、内存分配底层机制、优化策略
unity_version: 2021.3+
status: 待验证
validation: 未经测试
related: ["[[【最佳实践】GC优化清单]]", "[[【踩坑】内存泄漏模式]]", "[[../../31_代码优化/【最佳实践】Update优化清单]]"]
author: llm
sources:
  - "Jamin. Unity IL2CPP的GC原理. UWA (侑虎科技), 2024. https://mp.weixin.qq.com/s?__biz=MzI3MzA2MzE5Nw==&mid=2668943544&idx=1&sn=a036f9d59db85e299938903d542f01f8"
---

# 【设计原理】GC工作原理深度解析

> 核心价值：理解GC工作原理，才能有效优化内存分配

## 文档定位

Unity Boehm GC 垃圾回收器的完整工作原理深度解析。Unity 的 GC（无论是 Mono 还是 IL2CPP 后端）均使用 **Boehm-Demers-Weiser (BDW) GC**——一种**保守式、非分代、非压缩**的标记-清除收集器。本文涵盖 GC 基础概念、Boehm GC 的真实特性、触发条件、Stop-The-World 卡顿原因、标记-清除两阶段流程，以及增量 GC 的作用。理解这些原理是有效减少 GC 触发频率的前提。

> **重要事实纠正**：Unity 的 Boehm GC **不是分代 GC**，没有 Generation 0/1/2 的概念；也**不做内存压缩**，只有标记(Mark)和清除(Sweep)两个阶段。此前版本的本文档曾错误描述了 .NET CoreCLR 的分代 GC 机制，已于 2026-07-04 修正。

---

## 一、GC基础概念

### 1.1 什么是GC

```
┌─────────────────────────────────────────────────────────────┐
│                    GC（Garbage Collection）                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  定义：自动内存管理系统                                       │
│  ├─ 自动分配内存                                            │
│  ├─ 自动回收不再使用的内存                                   │
│  └─ 减轻开发者负担                                          │
│                                                             │
│  Unity使用的GC：                                            │
│  ├─ Mono GC（IL2CPP之前）                                   │
│  ├─ Boehm GC（默认）                                        │
│  └─ Incremental GC（增量GC）                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 1.2 托管堆 vs 原生堆

```
内存布局：
┌─────────────────────────────────────────────────────────────┐
│                        内存                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  托管堆（Managed Heap）                                      │
│  ├─ 由GC管理                                               │
│  ├─ 存储托管对象（C#对象）                                 │
│  ├─ 自动分配和回收                                         │
│  └─ 大小：几十MB到几百MB                                   │
│                                                             │
│  原生堆（Native Heap）                                      │
│  ├─ 不由GC管理                                            │
│  ├─ 存储原生对象（Texture、Mesh等）                         │
│  └─ 需要手动释放                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、Boehm GC 的核心特性

### 2.1 保守式 Mark-Sweep（非分代、非压缩）

```
┌─────────────────────────────────────────────────────────────┐
│              Unity Boehm GC 真实特性                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  算法：Mark-Sweep（标记-清除）                               │
│  ├─ 标记阶段：从根对象遍历，标记所有可达对象                 │
│  ├─ 清除阶段：释放未标记的对象内存                          │
│  └─ 没有压缩阶段（对象不移动，不整理碎片）                  │
│                                                             │
│  分代：否（NON-GENERATIONAL）                               │
│  ├─ 所有对象在同一个堆上                                    │
│  ├─ 每次 GC 必须扫描整个堆                                  │
│  └─ 没有"快速回收新生代"的优化                              │
│                                                             │
│  压缩：否（NON-COMPACTING）                                 │
│  ├─ 对象地址在 GC 后不改变                                  │
│  ├─ 不移动对象，不整理碎片                                  │
│  └─ 长时间运行会产生堆碎片                                 │
│                                                             │
│  类型：保守式（CONSERVATIVE）                               │
│  ├─ 不精确跟踪每个指针的类型                                │
│  ├─ 把"看起来像指针的值"当作指针                            │
│  ├─ 可能误判：把非指针数据当成引用，阻止回收                │
│  └─ 优点：可以作为 C/C++ 的 drop-in GC，兼容 IL2CPP        │
│                                                             │
│  后端：Mono 和 IL2CPP 均使用 Boehm GC                      │
│  ├─ Mono 后端：默认 Boehm（实验性可选 SGen 分代 GC）        │
│  └─ IL2CPP 后端：Boehm GC（通过 libgc）                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 2.2 为什么 Unity 使用 Boehm GC

```
选择 Boehm GC 的原因：
├─ 兼容性：保守式 GC 可以作为 C/C++ 的 drop-in GC
│   ├─ IL2CPP 将 C# 编译为 C++，需要兼容 C++ 内存模型
│   ├─ Boehm GC 无需精确的类型信息即可工作
│   └─ 不需要修改编译器或运行时的内存分配方式
│
├─ 稳定性：超过十年的生产验证
│   ├─ Boehm-Demers-Weiser GC 是最广泛部署的保守式 GC
│   ├─ 大量第三方库在 Boehm GC 下稳定运行
│   └─ 迁移到精确式 GC 需要大规模改造
│
└─ 代价：
    ├─ 非分代 → 每次 Full GC 扫描全堆，性能差
    ├─ 非压缩 → 堆碎片化，实际内存占用偏高
    └─ 保守式 → 偶尔会"泄漏"（把非指针误判为引用）
```

---

### 2.3 Boehm GC vs 分代 GC 的性能对比

```
为什么 Boehm GC 的 GC 卡顿比 .NET 分代 GC 严重？

分代 GC（如 .NET CoreCLR / Mono SGen）：
├─ Gen0 回收：只扫描新生代，约 0.1-1ms
├─ Gen1 回收：扫描新+中年代，约 1-10ms
├─ Gen2 回收：Full GC，扫描全堆，约 50-200ms
├─ Gen0/Gen1 频率高、耗时短 → 日常几乎无感
└─ Gen2 频率极低 → 大部分时间不受影响

Boehm GC（Unity 默认）：
├─ 每次 GC 都是 Full GC
├─ 必须扫描整个堆
├─ 没有快速回收新生代的优化
├─ 堆越大，单次 GC 耗时越长
└─ 即使大部分垃圾是短命对象，也要全堆扫描

实际影响：
├─ 托管堆 50MB：Full GC 约 10-30ms
├─ 托管堆 100MB：Full GC 约 30-80ms
├─ 托管堆 200MB+：Full GC 可达 100-300ms+
└─ 这就是 Unity GC Spike 的根本原因
```

> **增量 GC 的作用**：Unity 2019+ 引入的 Incremental GC 将 Boehm GC 的标记阶段**分摊到多帧**执行（每帧执行一小段），从而避免单次长卡顿。但本质仍是 Boehm GC——仍然不分代、不压缩，只是把 Full GC 的代价"分期付款"。

---

## 三、GC触发条件

### 3.1 触发条件

```
┌─────────────────────────────────────────────────────────────┐
│                    GC触发条件                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  条件1：堆空间不足                                          │
│  ├─ 托管堆无法满足新分配请求时触发                          │
│  └─ Boehm GC 执行 Full GC，扫描整个堆                      │
│                                                             │
│  条件2：手动调用                                            │
│  ├─ System.GC.Collect()                                    │
│  ├─ Resources.UnloadUnusedAssets()                         │
│  └─ SceneManager.LoadScene()                                │
│                                                             │
│  条件3：系统内存压力                                        │
│  ├─ 移动平台内存告警                                       │
│  ├─ 系统内存不足                                           │
│  └─ Unity主动触发GC                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 3.2 Boehm GC 的触发阈值

```
Boehm GC 没有分代阈值，触发机制基于堆空间使用率：

堆增长策略：
├─ Boehm GC 维护一个动态增长的堆
├─ 核心参数 GC_free_space_divisor（官方默认值 = 3）
│   ├─ 触发公式：当可用空闲空间 < heap_size / GC_free_space_divisor 时触发 GC
│   ├─ 值越大 → 堆越小但 GC 越频繁（省内存费 CPU）
│   ├─ 值越小 → 堆越大但 GC 越稀少（费内存省 CPU）
│   └─ 设为 1 → 几乎禁用 GC，堆无限增长
├─ GC 后如果回收了大量内存，堆可能缩小
├─ GC 后如果回收不够，堆会继续增长
└─ 整体目标是：堆越大，GC 频率越低（代价是单次 GC 更耗时）

实际行为：
├─ 托管堆较小时：GC 频繁但单次耗时短
├─ 托管堆较大时：GC 稀少但单次耗时长
├─ 移动平台：操作系统内存压力会强制触发 GC
└─ 没有"快速回收新生代"的优化路径
```

---

## 四、GC工作流程

### 4.1 GC的三个阶段（Mark-Sweep）

```
┌─────────────────────────────────────────────────────────────┐
│              Boehm GC 工作流程（Mark-Sweep）                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  阶段1：暂停（Stop The World）                               │
│  ├─ 暂停所有托管线程                                        │
│  ├─ 冻结内存状态                                            │
│  └─ 准备开始GC                                              │
│           ↓ 约0.1-1ms                                       │
│                                                             │
│  阶段2：标记（Mark）                                         │
│  ├─ 从根对象（Root）开始扫描                                │
│  ├─ 递归遍历所有引用                                        │
│  ├─ 标记所有可达对象                                        │
│  └─ 未标记对象 = 垃圾                                       │
│           ↓ 主要耗时部分（堆越大越慢）                      │
│                                                             │
│  阶段3：清除（Sweep）                                        │
│  ├─ 遍历堆，释放未标记对象的内存                            │
│  ├─ 恢复线程执行                                            │
│  └─ 完成 GC                                                 │
│           ↓ 约0.1-1ms                                       │
│                                                             │
│  ⚠️ 没有压缩（Compact）阶段！                               │
│  ├─ 对象地址不改变                                          │
│  ├─ 不移动对象                                              │
│  ├─ 不整理碎片                                              │
│  └─ 长时间运行 → 堆碎片化 → 实际内存占用偏高               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **关键区别**：.NET CoreCLR 的 GC 有 Mark-Sweep-Compact 三阶段（甚至会移动对象、更新引用），而 Unity 的 Boehm GC 只有 Mark-Sweep 两个阶段。这就是 Boehm GC 被称为"非压缩(non-compacting)"的原因。

---

### 4.2 根对象（Root）

```
什么是根对象？
├─ GC标记的起点                                              │
├─ 被认为是"总是存活"的对象                                  │
└─ 包括：                                                    │
   ├─ 静态字段（static fields）                              │
   ├─ 栈上的局部变量（local variables）                       │
   ├─ 当前活动线程的栈                                        │
   └─ GC句柄（GCHandle）                                      │

示例：
public class GameManager
{
    private static GameManager instance;  // 根对象
    private List<Enemy> enemies;  // 通过instance可达
}

GC从instance开始标记，enemies也会被标记为存活
```

---

## 五、Boehm GC 内存分配底层机制

> 本节深入 Boehm GC 的内存分配实现，理解这些机制有助于理解 GC 内存碎片和堆扩展的根因。
> 参考：Jamin《Unity IL2CPP的GC原理》(UWA)

### 5.1 分配入口

Boehm GC 的使用非常简单，将 `malloc` 替换为 `GC_malloc` 即可，之后无需手动 `free`：

```c
// 分配链：GC_malloc → GC_malloc_kind → GC_malloc_kind_global
void *GC_malloc(size_t lb) {
    return GC_malloc_kind(lb, NORMAL);
}
```

分配器底层通过平台相关接口向操作系统申请内存，**每次批量申请 4KB 的倍数**以提高效率。

核心思路：根据内存大小归类为**小内存对象**（≤2048 字节）和**大内存对象**（>2048 字节），分别走不同分配路径。

### 5.2 小内存分配

#### 粒度对齐（GRANULE）

Boehm GC 以 **GRANULE（16 字节）** 为基本分配单位：

```
原始大小 → GRANULE 数（通过 GC_size_map 映射表）

示例：
  1 字节  → 1 GRANULE = 16 字节
  18 字节 → 2 GRANULE = 32 字节
  100 字节 → 7 GRANULE = 112 字节

上限：128 GRANULE = 2048 字节（小内存上限）
```

`GC_size_map` 是一个索引映射表，维护原始大小到 GRANULE 数的映射，最多 128 个 GRANULE。

#### 空闲链表 ok_freelist

确定 GRANULE 数后，首先从**空闲链表**中查找可用内存：

```c
struct obj_kind {
    void **ok_freelist;         // 空闲链表（二维指针）
    struct hblk **ok_reclaim_list;
    ...
} GC_obj_kinds[3];  // 三种内存类型
```

三种内存类型：

| 类型 | 说明 | GC 行为 |
|------|------|---------|
| **PTRFREE** | 无指针对象 | GC 时跳过引用扫描 |
| **NORMAL** | 通用对象（保守式） | GC 时扫描可能的指针 |
| **UNCOLLECTABLE** | Boehm 自用 | 不标记、不回收 |

`ok_freelist` 维护了 0~127 个链表索引，每个索引对应一种 GRANULE 大小的内存池：

```
ok_freelist[0] → null（不可能存在 0 GRANULE）
ok_freelist[1] → [16B空闲块] → [16B空闲块] → ...
ok_freelist[2] → [32B空闲块] → [32B空闲块] → ...
  ...
ok_freelist[127] → [2032B空闲块] → ...
```

分配时计算 GRANULE 索引，查对应 freelist，有空闲则直接返回。

### 5.3 核心内存块链表 GC_hblkfreelist

当 `ok_freelist` 无可用内存时，向底层内存池申请。底层维护了 `GC_hblkfreelist`——一个 **60 个元素的分级空闲链表**，基本单位是 **4KB（一个内存页）** 的 `hblk`：

```c
struct hblk {
    char hb_body[HBLKSIZE];  // HBLKSIZE = 4096
};
```

每个 `hblk` 有对应的 header 信息：

```c
struct hblkhdr {
    struct hblk *hb_next;      // 链表后继
    struct hblk *hb_prev;      // 链表前驱
    unsigned char hb_obj_kind; // PTRFREE/NORMAL/UNCOLLECTABLE
    word hb_sz;                // 分配给上层=实际单位；空闲=内存块大小
    word hb_marks[MARK_BITS_SZ]; // GC 标记位
};
```

#### 链表分级规则

`GC_hblkfreelist` 的 60 个链表索引按所需 4KB 块数映射：

```
所需块数          → 链表索引
1~32             → 直接映射（index = blocks_needed）
33~256           → 8 块一组（index = (blocks-32)/8 + 32）
>256             → 全部归入 index 60
```

#### 查找与分割策略

1. **精确查找**：先从 `GC_hblkfreelist[blocks_needed]` 精确匹配
2. **升序查找**：精确失败则逐级增大 index，从更大块链表中查找
3. **分割返回**：找到更大块后，一分为二——前半返回使用，后半加入对应链表

```
示例：申请 1 个 hblk (4KB)
  ├─ 先查 freelist[1]：精确匹配 4KB → 直接返回
  └─ 若无，查 freelist[2]：找到 8KB → 拆为 4KB(使用) + 4KB(加入 freelist[1])
```

#### 新内存块申请

若所有链表都无可用块，通过 `GC_expand_hp_inner` 向操作系统申请内存：

```
GC_expand_hp_inner → GET_MEM(系统调用) → GC_add_to_heap(加入链表)
```

`GC_add_to_heap` 还会检查相邻地址的空闲块，**合并连续内存块**生成更大的块。

### 5.4 大内存分配（>2048 字节）

大对象不走 GRANULE 路径，直接以 hblk(4KB) 为单位分配：

```c
// 9000 字节 → 需要 3 个 hblk (12KB)
n_blocks = OBJ_SZ_TO_BLOCKS(9000);  // = 3
h = GC_allochblk(lb, k, flags);     // 走 hblkfreelist 查找
```

与小型分配不同：大块找到后**不拆分构建 ok_freelist**，直接返回整块地址。

分配失败时会尝试触发 GC 回收内存后重试：

```c
while (0 == h && GC_collect_or_expand(n_blocks, ...)) {
    h = GC_allochblk(lb, k, flags);  // GC 后重试
}
```

### 5.5 完整分配流程

```
GC_malloc(lb)
  │
  ├─ lb ≤ 2048？──→ 小内存路径
  │    │
  │    ├─ GC_size_map[lb] → gran (GRANULE 数)
  │    ├─ 查 ok_freelist[gran]
  │    │    ├─ 有空闲 → 直接返回
  │    │    └─ 无空闲 → GC_allocobj → GC_new_hblk
  │    │         ├─ GC_allochblk: 从 hblkfreelist 查/分割/系统申请
  │    │         └─ GC_build_fl: 将 hblk 拆分为 GRANULE 块，构建 ok_freelist
  │    └─ 返回内存地址
  │
  └─ lb > 2048？──→ 大内存路径
       │
       ├─ OBJ_SZ_TO_BLOCKS(lb) → n_blocks
       ├─ GC_alloc_large → GC_allochblk
       │    ├─ hblkfreelist 查找（不拆分）
       │    └─ 失败 → GC_collect_or_expand → 重试
       └─ 返回 hblk->hb_body
```

### 5.6 为什么 Unity 托管堆只增不减

理解了上述机制，就能解释 Unity 托管堆的几个关键行为：

1. **堆扩展后优先复用而非归还**：`GC_expand_hp_inner` 申请的内存加入 `GC_hblkfreelist`，优先复用。BDW GC v8.0.0+ 在 `USE_MUNMAP` 启用时会归还**完全空闲**的 hblk 给 OS（通过 `munmap`/`VirtualFree(MEM_DECOMMIT)`），但碎片化导致大部分 hblk 有零星存活对象，极少能整块归还
2. **非压缩导致碎片**：Mark-Sweep 只标记+清除，不移动对象，空闲 hblk 可能分散在各处。真实案例：736MB 可达对象因碎片占用 1GB 堆空间 [^BoneFragment]
3. **碎片触发堆扩展**：总空闲空间够但无连续大块 → 分配失败 → GC → 仍失败 → 扩展堆

> 这就是为什么**减少托管堆分配**比优化 GC 频率更根本——不产生垃圾就不会触发 GC，不扩展堆就不会产生碎片。

[^BoneFragment]: Paul Bone. Memory Fragmentation in BDWGC. 2016. https://paul.bone.id.au/blog/2016/10/08/memory-fragmentation-in-boehmgc/

### 5.7 补充：SGen GC

SGen（Simple Generational GC）是 Mono 的**分代 GC**，比 Boehm GC 更先进：

| 特性 | Boehm GC | SGen GC |
|------|----------|---------|
| 分代 | 否 | 是（Nursery + Old Gen） |
| 压缩 | 否 | 是 |
| 回收类型 | 全堆 Full GC | Minor GC（初生代）+ Major GC（全堆） |
| Unity 支持 | IL2CPP 默认 | Mono 实验性（非默认） |

**SGen 内存结构**：

```
初生代 (Nursery)：
├─ 固定大小连续内存（默认 4MB）
├─ 多线程各自 TLAB (4KB) 内指针碰撞分配
├─ 不划分粒度
└─ Minor GC 回收（频率高、耗时短）

旧生代 (Old Generation)：
├─ Section(1MB) → Block(16KB) → Page(4KB) → Slot(不同粒度)
├─ 从 Nursery 晋升的对象按粒度存入对应 Slot
├─ 空闲 Slot 返还 freelist，清空的层级逐级向上返还
└─ Major GC 回收（频率低）

大对象 (>8KB)：
├─ ≤1MB：存于 Mono 托管堆 LOSSection
└─ >1MB：直接向 OS 申请，清理后归还 OS
```

> **注意**：目前 Unity IL2CPP 后端**不支持 SGen**，只能用 Boehm GC。SGen 仅在 Mono 后端作为实验性选项。Unity 官方在探索精确式 GC 但尚无明确时间线。

---

## 六、GC性能影响

### 5.1 GC卡顿原因

```
为什么GC会导致卡顿？

1. Stop The World
   ├─ GC期间暂停所有托管线程
   ├─ 游戏逻辑停止执行
   └─ 表现为帧率下降或冻结

2. 标记阶段耗时
   ├─ 需要遍历所有对象
   ├─ 对象越多，耗时越长
   └─ 可能耗时几十到几百毫秒

3. 全堆扫描（非分代的代价）
   ├─ Boehm GC 没有分代，每次 GC 都扫描整个堆
   ├─ 堆越大，标记阶段越长
   └─ 即使大部分垃圾是短命对象，也要全堆遍历

4. 堆碎片化（非压缩的长期影响）
   ├─ Boehm GC 不移动对象，不整理碎片
   ├─ 长时间运行后堆碎片化严重
   ├─ 实际内存占用比真实需要的多得多
   └─ 最终可能触发操作系统内存告警
```

---

### 5.2 Boehm GC 性能数据

```
Boehm GC 每次 GC 都是 Full GC，没有分代优化：

Full GC 耗时（与托管堆大小正相关）：

托管堆 10-30MB：
├─ 耗时：约 3-10ms
├─ 影响：轻微掉帧
└─ 60FPS 预算：16.6ms/帧，勉强可接受

托管堆 50-100MB：
├─ 耗时：约 15-60ms
├─ 影响：明显卡顿（1-3 帧卡死）
└─ 60FPS 影响：严重

托管堆 200MB+：
├─ 耗时：约 100-300ms+
├─ 影响：严重卡顿（6-18 帧卡死）
└─ 表现：肉眼可见的"冻帧"

关键区别（对比分代 GC）：
├─ 分代 GC 的大部分 GC 只回收 Gen0（~1ms），不卡顿
├─ Boehm GC 每次都全堆扫描，没有"便宜"的 GC
└─ 因此 Unity 的核心策略是：尽量不触发 GC（减少分配）
```

---

### 5.3 GC导致的性能问题示例

```
场景：每帧分配1KB内存

计算：
├─ 每帧分配：1KB
├─ 60FPS分配：60KB/秒
├─ 堆增长到触发阈值（假设当前堆已稳定在约50MB）
├─ 触发频率：取决于堆增长策略，大约几十秒一次
└─ 影响：不频繁的 Full GC，每次约15ms，可接受

场景：每帧分配100KB内存

计算：
├─ 每帧分配：100KB
├─ 60FPS分配：6MB/秒
├─ 堆快速膨胀，频繁触发 Full GC
├─ 触发频率：可能每1-3秒一次
└─ 影响：每次 Full GC 15-60ms+，严重卡顿
```

---

## 七、减少GC触发的策略

### 7.1 核心策略

```
┌─────────────────────────────────────────────────────────────┐
│              减少GC触发的核心策略                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  策略1：减少分配                                            │
│  ├─ 避免在Update中new对象                                   │
│  ├─ 避免在Update中new集合                                   │
│  ├─ 避免字符串拼接                                         │
│  └─ 避免使用LINQ                                            │
│                                                             │
│  策略2：复用对象                                            │
│  ├─ 使用对象池                                             │
│  ├─ 复用临时对象                                           │
│  └─ 预分配集合                                             │
│                                                             │
│  策略3：使用值类型                                          │
│  ├─ struct代替class（适用时）                               │
│  ├─ 避免装箱拆箱                                           │
│  └─ 使用数组代替List（适用时）                              │
│                                                             │
│  策略4：优化字符串                                           │
│  ├─ 使用StringBuilder                                      │
│  ├─ 缓存常用字符串                                         │
│  └─ 避免字符串拼接                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 7.2 具体技巧

#### 技巧1：复用集合

```csharp
// ❌ 错误：每帧new List
void Update()
{
    var enemies = new List<Enemy>();
    // ...
}

// ✅ 正确：复用List
private List<Enemy> enemies = new List<Enemy>();

void Update()
{
    enemies.Clear();
    // ...
}
```

#### 技巧2：使用对象池

```csharp
// ❌ 错误：频繁Instantiate和Destroy
void SpawnBullet()
{
    var bullet = Instantiate(bulletPrefab);
    // ...
}

// ✅ 正确：使用对象池
void SpawnBullet()
{
    var bullet = bulletPool.Get();
    // ...
    bulletPool.Return(bullet);
}
```

#### 技巧3：避免装箱拆箱

```csharp
// ❌ 错误：装箱拆箱
void ProcessValue(int value)
{
    object obj = value;  // 装箱
    int result = (int)obj;  // 拆箱
}

// ✅ 正确：使用泛型
void ProcessValue<T>(T value) where T : struct
{
    // 无装箱拆箱
}
```

---

## 八、GC优化检查清单

### 8.1 快速检查清单

```
□ 没有在Update中new对象
□ 没有在Update中new集合
□ 没有在Update中字符串拼接
□ 没有在Update中使用LINQ
□ 使用了对象池
□ 复用了临时对象
□ 避免了装箱拆箱
□ 使用了StringBuilder
□ 预分配了集合容量
□ 使用了数组代替List（适用时）
```

---

### 8.2 Profiler检查

```
使用Unity Profiler检查GC：

1. 打开Profiler
   Window → Analysis → Profiler

2. 记录游戏运行
   点击Record按钮

3. 查看GC.Alloc
   在Profiler中查找GC.Alloc列
   关注Update中的GC.Alloc

4. 分析GC调用
   查看GC.Column
   关注GC触发频率

5. 定位热点
   Deep Profile查看具体函数
   找到GC分配的源头
```

---

## 九、常见问题

### Q1: 手动调用GC好吗？

**A**: 通常**不推荐**手动调用GC

```csharp
// ❌ 不推荐：每帧调用GC
void Update()
{
    System.GC.Collect();  // 每帧GC，性能灾难
}

// ✅ 正确：只在必要时调用
void OnApplicationPause(bool pause)
{
    if (pause)
    {
        // 应用进入后台时GC
        System.GC.Collect();
    }
}
```

**原因**：
- 手动GC成本高
- GC会自动优化
- 频繁手动GC反而降低性能

---

### Q2: IL2CPP会影响GC吗？

**A**: IL2CPP 不会改变 GC 类型，Mono 和 IL2CPP 默认都使用 Boehm GC

```
Mono vs IL2CPP 的 GC：

Mono 后端：
├─ 默认：Boehm GC（非分代、非压缩）
├─ 实验性：可选 SGen（Mono 的分代 GC，非默认）
├─ JIT 编译
└─ 适合开发阶段

IL2CPP 后端：
├─ 使用 Boehm GC（通过 libgc）
├─ AOT 编译 → C++ → 原生机器码
├─ GC 类型与 Mono 相同（都是 Boehm）
├─ 代码执行效率更高 → 间接减少 GC 分配
└─ 适合发布版本

结论：
├─ IL2CPP 不会改变 GC 算法（仍是 Boehm）
├─ 但 AOT 编译优化可能减少临时分配
└─ IL2CPP 不支持 SGen，只能用 Boehm
```

---

### Q3: 增量GC（Incremental GC）有什么用？

**A**: 增量 GC 将 Boehm GC 的标记阶段分摊到多帧，减少单次卡顿

```
普通 GC（Stop-The-World）：
├─ 一次性完成 Mark + Sweep
├─ 托管堆 100MB 时可能耗时 30-80ms
└─ 导致明显卡顿

增量 GC（Incremental GC）：
├─ 将标记阶段拆分为多个小步
├─ 每帧只执行一小段标记工作
├─ 单帧额外耗时：约 1-3ms
├─ 总耗时不变，但分散到多帧
└─ 大幅减少单帧卡顿

⚠️ 本质仍是 Boehm GC：
├─ 仍然不分代
├─ 仍然不压缩
├─ 只是把 Full GC 的代价"分期付款"
└─ 仍需减少分配来从根本上降低 GC 频率

开启方式：
Player Settings → Other Settings → Incremental GC
```

---

### Q4: Unity GC 是分代的吗？

**A**: **不是。** Unity 默认的 Boehm GC 是**非分代**的。

```
常见误解 vs 事实：

误解：Unity GC 有 Gen0/Gen1/Gen2，Gen0 回收很快
事实：Boehm GC 没有"代"的概念，每次 GC 都是 Full GC

误解：Gen0 GC 只要 1-5ms，对帧率影响小
事实：Boehm GC 每次都扫描整个堆，耗时与堆大小正相关

误解：短命对象在 Gen0 被快速回收，不影响性能
事实：所有对象平等对待，短命对象也要等 Full GC 才被回收

误解：Unity GC 会自动压缩内存、消除碎片
事实：Boehm GC 不做压缩，长时间运行会产生堆碎片

什么时候 Unity 可能用上分代 GC？
├─ Mono 后端实验性支持 SGen（分代），但非默认
├─ IL2CPP 后端只能用 Boehm
├─ Unity 官方在探索精确式 GC，但尚无明确时间线
└─ 目前（Unity 6 / 2024-2025）：默认 Boehm GC
```

---

## 相关链接

- [[【最佳实践】资源卸载指南]] ← 资源卸载
- [[【最佳实践】GC优化清单]] ← GC优化技巧
- [[【踩坑】内存泄漏模式]] ← 内存泄漏案例
- [[../../30_性能优化/31_代码优化/【最佳实践】Update优化清单]] ← Update优化

---

*创建日期: 2026-03-05*
*相关标签: #GC #内存管理 #性能优化 #设计原理*
