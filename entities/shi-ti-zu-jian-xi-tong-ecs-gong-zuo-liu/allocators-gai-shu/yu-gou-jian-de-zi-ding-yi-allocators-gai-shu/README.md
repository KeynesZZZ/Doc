# 预构建的自定义 allocators 概述

#### 预构建的自定义 allocators 概述

你可以使用预构建的自定义 allocators 来管理在 worlds、entity command buffers 和 system groups 中的分配。以下所有都是可回退的 allocators：

**Allocator 描述**

| Allocator                               | 描述                                                                                                                                  |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| World update allocator                  | <p>一个世界拥有的双重可回退 allocator，快速且线程安全。<br><br>这个 allocator 所做的分配每2帧自动释放，这使其非常适用于持续2帧的 world 内部分配。没有内存泄漏。</p>                            |
| Entity command buffer allocator         | <p>一个实体命令缓冲系统拥有的可回退 allocator，快速且线程安全。<br><br>实体命令缓冲系统使用此 allocator 创建实体命令缓冲。在播放完一个实体命令缓冲后，实体命令缓冲系统会自动释放分配。这种 allocator 没有内存泄漏。</p> |
| System group allocator for rate manager | <p>当设置其速率管理器时，由组件系统组创建的可选双重可回退 allocator。<br><br>它适用于固定或可变速率系统组中进行的分配，这些系统组的 tick 速率不同于世界更新。分配持续2次系统组更新，不需要手动释放分配。</p>              |

这些预构建的 allocators 是自定义 allocators。要分配和释放 `Native-` 集合类型和 `Unsafe-` 集合类型，请参阅 Collections 包文档中的《如何使用自定义 allocator》。
