# 场景流式传输（Scene streaming）

为了避免停滞，Entities 中的所有场景加载都是异步进行的。这被称为流式传输。

你可以通过部分（sections）、元实体（meta entities）和实例化（instancing）来控制场景的流式传输。

### 主题概述

<table><thead><tr><th width="418">主题</th><th>描述</th></tr></thead><tbody><tr><td>场景流式传输概述（<a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/manual/streaming-overview.html">Scene streaming overview</a>）</td><td>了解场景流式传输。</td></tr><tr><td>场景部分概述（<a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/manual/streaming-scene-sections.html">Scene section overview</a>）</td><td>处理场景部分。</td></tr><tr><td>加载场景（<a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/manual/streaming-loading-scenes.html">Load a scene</a>）</td><td>在项目中加载场景。</td></tr><tr><td>场景和部分元实体（<a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/manual/streaming-meta-entities.html">Scene and section meta entities</a>）</td><td>理解部分元实体。</td></tr><tr><td>场景实例化（<a href="https://docs.unity3d.com/Packages/com.unity.entities@1.3/manual/streaming-scene-instancing.html">Scene instancing</a>）</td><td>创建场景的多个实例。</td></tr></tbody></table>

### 详细信息

#### 场景流式传输概述

场景流式传输是指异步加载场景数据的过程，以避免编辑器或运行时的卡顿。利用流式传输，可以根据需要动态加载和卸载场景内容，从而优化内存使用和性能。

#### 场景部分概述

场景部分（Scene Sections）将一个大场景划分为多个较小的部分，每个部分可以独立加载和卸载。这样可以更高效地管理和渲染场景内容。

#### 加载场景

使用流式传输功能加载场景时，可以选择按需加载整个场景或者仅加载特定的场景部分。这有助于减少初始加载时间，并确保只加载当前所需的数据。

#### 场景和部分元实体

元实体（Meta Entities）是用于管理和跟踪场景和部分加载状态的特殊实体。它们包含了有关场景或部分的元数据，并负责协调相应的加载和卸载过程。

#### 场景实例化

场景实例化允许你创建单个场景的多个实例。这对于需要在多个位置或以不同配置出现同一场景内容的情况非常有用。通过实例化，可以有效地复用已有的场景数据，从而节省内存和提高性能。

***

这些概念和技术共同构成了一个高效、灵活的场景管理系统，使你能够在大型项目中更好地控制数据流动，优化加载性能。
