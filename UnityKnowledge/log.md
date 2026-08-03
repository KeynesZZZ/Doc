# UnityKnowledge 维护日志（log.md）

> append-only。每行前缀 `## [YYYY-MM-DD] ingest|query|lint | 标题`，可 `grep "^## \[" log.md | tail`。

## [2026-06-24] lint | P1 地基初始化
- 生成首版 index.md
- 全库回填 author 字段
- 统一 lint 基线跑通

## [2026-06-25] query | 回填 UGUI 性能优化规则清单
- 新增 [[20_核心系统/26_UI系统/【片段】UGUI 性能优化规则清单]]（author:llm + sources）
- 将「UGUI 如何实现 / 如何优化」问答沉淀为 R1–R6 规则集 + ROI 表 + 监控阈值 + Code Review 清单
- 来源：踩坑记录、UI卡顿优化实战案例、合批机制深度解析、UGUI 第4/7章
- 更新 UI系统专题索引（新增「规则清单」分类 + 目录条目 + 相关链接），避免孤儿页

## [2026-06-26] ingest | UGUI性能优化实战总览
- 读源：知乎《Unity性能优化总结-UI》(DragonL, 2025-08) → https://zhuanlan.zhihu.com/p/1941865546690827415
- 新增综述页：20_核心系统/26_UI系统/【笔记】UGUI性能优化实战总览.md（author:llm，sources 含原文 URL + 4 篇既有笔记）
- 新增主题：图集/SpriteAtlas 打包冗余规则、RawImage 陷阱、图集尺寸策略、Mask vs RectMask2D 数量决策、动静分离、界面切换开销数据、刘海屏/横竖屏适配、UI 混合粒子、字体加粗与 Font Texture
- 重叠主题（合批/DrawCall/SendWill/TMP）以链接指向既有深度笔记，未重复展开；并与 [[【片段】UGUI 性能优化规则清单]] 互链
- 更新 README.md、UI系统专题索引.md（综述分类 + 推荐阅读首位 + 目录条目）；重生成 UnityKnowledge/index.md（254 篇）
- lint：新文档零 issue，未引入新断链/孤儿

## [2026-06-26] query | 回填 Playable API 使用指南
- 新增 [[20_核心系统/21_动画系统/【笔记】Playable API 使用指南]]（author:llm + sources）
- 将「讲解 Playable 的使用」问答沉淀为：概念定位 / 数据流与求值顺序 / 三大类节点（PlayableGraph·ScriptPlayable·动画类）/ 最小示例 / 程序化动画（IAnimationJob）/ 选型表 / 高频坑速查
- 来源：[[Clippings/【Unity】Playable使用细则]]（知乎 p/632890306）、Unity 官方 Playables 文档、[[【设计原理】Animator深度解析]]
- 更新 21_动画系统 README（入口）+ 动画系统专题索引（新增「笔记」分类 + 目录条目）+ index.md，避免孤儿页

## [2026-06-26] ingest | 新增 tolua 专题
- 新增专题索引：35_高级主题/tolua专题索引.md（author:llm），聚合 toLua# 绑定机制/性能/踩坑 + 既有热更新选型文档
- 新增 3 篇笔记：[[35_高级主题/【笔记】tolua入门与调用机制]]（Wrap/LuaState/C#↔Lua 互调）、[[35_高级主题/【笔记】tolua性能与GC优化]]（边界开销/GC/LuaJIT 平台差异）、[[35_高级主题/【踩坑】tolua热更新常见坑]]（7 类坑速查表）
- 新增 35_高级主题/README.md（领域入口，避免孤儿页）
- 内容为 LLM 基于训练知识编译（非外部源），统一标 status:待验证，未编造 sources
- 重生成 UnityKnowledge/index.md（260 篇）

## [2026-06-26] ingest | 补充 Lua 语法速查
- 新增 [[35_高级主题/【笔记】Lua语法速查]]（author:llm，status:待验证）：以 Lua 5.1/LuaJIT（tolua 默认）为准
- 覆盖：8 种数据类型、运算符（~= / and or / ..）、控制流（含无 continue）、函数（多返回值/可变参/`:` 糖）、table（1-based 索引、pairs/ipairs）、metatable 面向对象、闭包/upvalue、require 模块、给 C# 开发者的易错点速查表
- 更新 tolua专题索引（收录数 5→6、阅读顺序新增第 2 位、笔记分类/目录/相关链接）与 35_高级主题/README 入口
- 重生成 UnityKnowledge/index.md（261 篇）

## [2026-06-26] ingest | 补全热更新方案对比（机制+选型总览）
- 扩写 [[40_工具链/【设计原理】热更新方案对比]]（author:llm，status:待验证）：60 → ~217 行，定位为热更专题的「机制+选型总览页」
- 核心修正：拆分「资源热更(AssetBundle/Addressables) vs 代码热更」两条正交轴（旧版混在同一张表）；新增根因（iOS W^X 禁 JIT → 只能解释器/AOT）、三类方案执行机制剖析（Lua 系 / ILRuntime / HybridCLR + puerts）、多维对比表、选型决策矩阵、共性工程问题、DO/DON'T
- 纠偏：HybridCLR 由「侵入性强」改为「低侵入，代价在构建管线 + 泛型补充 AOT 元数据」；指明 LuaJIT iOS 无 JIT、基准以 iOS 解释器模式为准；区分 xLua Hotfix（可注入式热修已上线 C#）与 toLua#（不可热修 C# 接口）
- 「怎么做」交给周边教程（Addressables / 打包与热更新），本文仅 wikilink 不重复
- 内容为 LLM 基于训练知识编译（非外部源），未编造 sources；官方链接（HybridCLR/xLua/ILRuntime/puerts/Addressables）放正文「相关链接」作参考
- 重生成 UnityKnowledge/index.md（261 篇，仅本文 description 同步）；lint：本文档零 issue，工具链目录 64 链接全有效（既有 44 ERROR 均为模板/Clippings 作者标签断链，与本次无关）

## [2026-06-27] ingest | 热更新方案对比迁入 35_高级主题
- [[35_高级主题/【设计原理】热更新方案对比]] 由 40_工具链 迁入 35_高级主题（与 tolua 专题/打包与热更聚合，图谱更连通）
- 同步本文：category 工具链→高级主题、updated→2026-06-27；文件内相对 wikilink 改同目录 basename（tolua 系列 / 打包与热更），Addressables 链接补 ../40_工具链/ 跨目录前缀
- 入链同步：tolua专题索引（推荐顺序/分类标签/按目录段重组，覆盖目录数 2→1）、tolua入门 related 去旧路径前缀、Addressables教程 markdown 链接改 ../35_高级主题/
- 重生成 UnityKnowledge/index.md（262 篇，本文归入 35_高级主题 段）；lint ERROR=44 不变（零新断链），35_高级主题(133)/40_工具链(43) 目录链接全有效
- 附注：lint 新增 1 WARN 孤儿页 [[35_高级主题/【实战案例】tolua热更新系统完整落地]] 为外部新建未跟踪文件，与本次无关

## [2026-06-27] ingest | 新增 Lua 面向对象深入笔记
- 新增 [[35_高级主题/【笔记】Lua面向对象深入]]（author:llm，status:待验证）：把 Lua OOP 从语法糖讲到底层机制
- 覆盖：`__index` 读路径查找算法、self 本质（类方法 vs 实例方法）、生产级 `class()` 逐行剖析、继承链两条委托方向、super 正确写法（`.` + 手动 self）、`__index` vs `__newindex` 读写不对称、原型委托 vs 闭包范式权衡、访问控制/接口的 duck typing、`.` vs `:` 深坑、metatable 链查找的性能代价（呼应 iOS 解释器模式）、tolua 热更语境三点、C# vs Lua OOP 速查表
- 定位为 [[35_高级主题/【笔记】Lua语法速查]] 第七节（基础）的机制深入篇，不重复基础语法
- 更新 tolua专题索引（收录数 6→7、笔记分布 3→4、推荐阅读顺序第 3 位、笔记分类/按目录/相关链接）；互链避免孤儿
- 内容为 LLM 基于训练知识编译（非外部源），未编造 sources
- 重生成 UnityKnowledge/index.md（263 篇）；lint：新文档零 issue，35_高级主题 157 链接全有效（ERROR=44 不变）

## [2026-06-27] ingest | Lua OOP 笔记补充云风 class/单例/重载（首次引入 sources）
- 读源：博客园《Lua 面向对象》（Fflyqaq, 2020）→ https://www.cnblogs.com/Fflyqaq/p/13292388.html
- 增量补充 [[35_高级主题/【笔记】Lua面向对象深入]]：云风 class.lua 虚表实现（类表代理独立虚表 + 继承查找缓存 `vtb[k]=ret`，呼应第十节性能）、构造链基类优先递归（对比第五节手动 super）、单例模式（Instance+m_instance）、重载模拟（可变参+nil 判断）
- 该源基础部分（类/封装/继承/多态）与既有笔记重叠且笔记更深入机制（如读写不对称），仅取增量；文章 1.3「boy.name 不影响 person」正印证第六节读委托/写实例
- 首次为该笔记加 sources（引用即契约）；基础机制部分仍为 LLM 基于训练知识编译，非整篇 ingest
- 重生成 UnityKnowledge/index.md；lint：本文档零 issue

## [2026-06-27] ingest | 新增 tolua 完整实战案例
- 新增 [[35_高级主题/【实战案例】tolua热更新系统完整落地]]（author:llm，status:待验证）：以休闲游戏为例的 toLua# 热更新系统端到端参考实现
- 覆盖：C# 宿主层（LuaEnv/HotUpdateManager/稳定接口契约）、Lua 业务层（main + 模块）、版本比对→下载→校验→Reload 时序、对接踩坑文档的问题处置表、效果权衡与换方案建议
- 明确标注为参考实现（非某次真实项目复盘），未编造实测性能数据
- 更新 tolua专题索引（收录数→8、推荐阅读顺序 #8、新增「实战案例」分类/目录/相关链接）与 35_高级主题/README
- 注：用户同期新增 [[35_高级主题/【笔记】Lua面向对象深入]] 并将 [[【设计原理】热更新方案对比]] 由 40_工具链 移入本目录，索引已同步
- 重生成 UnityKnowledge/index.md（264 篇）

## [2026-06-27] ingest | 新增 Unity 客户端面试题综述（按域补全答案）
- 读源：知乎《Unity客户端面试题记录》（Ray小铭）→ https://zhuanlan.zhihu.com/p/449331086
- 新增 [[35_高级主题/【笔记】Unity客户端面试题]]（author:llm，status:待验证，sources 含原文 URL + 6 篇互链笔记）：跨域面试速查 + 复习清单
- 覆盖 29 道题，按域分 7 章：C#/.NET（GC/闭包/unsafe/值引用类型/il2cpp vs Mono）、渲染（渲染管线/UGUI渲染流程/合批/UGUI优化/粒子层级/实时阴影）、算法（A*/DFS-BFS/快排/三消）、Lua（数据类型/字符串拼接/sort/OOP/GC与泄漏）、3D数学（叉乘判左右/2D射线相交/AABB Slab法）、网络（GET vs POST/TCP握手挥手）、Unity工程（AB打包/Animator优化/骨骼动画/背包虚拟化）
- 原文已给答案的题目整理自原文（如渲染管线、合批、AB策略、叉乘、TCP）；原文只列题目无答案的由 LLM 补充并明确标注「（LLM 补充）」（如 A*、il2cpp/Mono、实时阴影、AABB Slab、骨骼动画、Lua sort/OOP/GC 等），未编造数据
- 强调作者经验：中级主考算法/Lua/3D数学，3D数学公式原理为高频失分点
- 交叉互链既有真相层笔记（Lua面向对象深入/UGUI性能优化综述/打包与热更新/Unity内存管理/tolua GC/三消架构），避免孤儿且可下钻
- 更新 35_高级主题/README 入口；重生成 UnityKnowledge/index.md（264→266 篇）
- lint：新文档与 35_高级主题 目录零 issue；总计 ERROR=48/WARN=25 均为既有 UGUI/、Clippings/、性能优化/ 书籍摘录的作者署名 token（黑客不黑/洛桑/MrLiu 等），非本次引入

## [2026-06-27] query | 回填 AB 压缩格式选型（LZMA vs LZ4）
- 用户追问综述第 26 题 AB 打包的压缩选择，对话答案回填 [[35_高级主题/【笔记】Unity客户端面试题]] 第 26 题新增「压缩格式选型」子节
- 内容：LZMA/LZ4/不压缩 三方对比表（压缩率/解压速度/随机访问）+ 原理（LZMA 流式全局字典 vs LZ4 块压缩可 seek）+ 选型决策（热更用 LZ4、首包用 LZMA、CDN 传 LZMA+本地缓存转可随机访问）
- 未改 frontmatter（updated 仍 2026-06-27）；无新增断链

## [2026-06-28] ingest | 新增 Unity 资源依赖与打包陷阱综述（原文反爬截断，据片段+官方文档撰写）
- 读源：知乎《为什么你的包体总是莫名增大？深度解析 Unity 资源依赖关系与打包陷阱》（【Unity 底层与原理向】03）→ https://zhuanlan.zhihu.com/p/1962283012189292152
  - ⚠️ 反爬限制：web reader / curl / Googlebot / 知乎 API / Wayback / RSSHub 全部受阻，仅获取到原文**开头约 1/3**（标题、开篇四问、图书馆引用链类比、Resources 模式开头），正文后半未获取
  - 用户决策：基于「已获取片段 + Unity 官方 AssetBundle 依赖文档 + 工程经验」撰写综述，文内严格区分标注
- 新增 [[35_高级主题/【笔记】Unity资源依赖与打包陷阱]]（author:llm，status:待验证，sources 含知乎原文 + Unity 官方文档 ×3 + 互链笔记 ×6）
- 内容：问题四问（原文）→ 图书馆类比（原文，截断处标注）→ 依赖系统底层机制（LLM 补充，据 Unity 官方）→ 打包冗余成因（LLM 补充，含官方 383KB+377KB→359KB+2×20KB 算账）→ 灵异事件归因 → 依赖分析工具（Browser/Analyzer/Analyze/GetDependencies/BuildReport）→ 优化策略（公共依赖单独成包/加载顺序/Shader 图集字体/分包粒度/Resources 清理）→ 避坑清单
- 来源分离：`（原文）` 标注整理自原文已获取片段；`（LLM 补充，据 Unity 官方文档）` 标注 LLM 编译部分，并在文首声明"可能与原文后半不一致，引用前对照原文"，未编造 sources
- 聚焦增量：通用 AB 依赖树/构建期冗余/分析工具/打包策略；不重述 UI 图集冗余（→UGUI性能优化综述）、Addressables 用法（→资源管线-Addressables）、热更选型（→热更新方案对比）
- 交叉互链既有笔记（打包与热更新/Unity客户端面试题/资源管线-Addressables/资源卸载指南/内存分析工具使用指南/Unity内存管理/资源预加载策略/UGUI性能优化实战总览），避免孤儿且可下钻
- 更新 35_高级主题/README 入口；重生成 UnityKnowledge/index.md（273 篇）
- lint：新文档与 35_高级主题 目录零 issue；总计 ERROR=50/WARN=32 均为既有 UGUI/、Clippings/、性能优化/ 书籍摘录的作者署名 token 断链及其在 index.md 的镜像，非本次引入

## [2026-06-30] query | 新增高级 Unity 游戏开发面试复习地图
- 用户诉求：整理北京高级 Unity 游戏开发岗面试复习知识点（聚焦游戏方向）
- 先采岗：Boss 直聘列表页动态加载/反爬，仅得 SEO 通用推荐位；改综合 BOSS直聘/猎聘/智联/职友集/企业官网公开 JD 与聚合数据（2026-06），得四方向（游戏/座舱智驾/AR-VR数字孪生/通用仿真）+ 薪资带 + 学历经验技能要求
- 新增 [[35_高级主题/【笔记】高级Unity游戏开发面试复习地图]]（author:llm，status:待验证，sources 含 JD 来源 + 互链笔记）
- 定位：与已有 [[【笔记】Unity客户端面试题]]（中级「题目+答案」速查）互补——本文是「高级岗能力域 + 笔记导航 + 重要度 + 自评」地图，不重复题库
- 以 10 个能力域组织（性能优化⭐⭐⭐/渲染Shader⭐⭐⭐/资源热更⭐⭐⭐/UGUI⭐⭐⭐/架构/C#/动画/算法3D数学/DOTS等加分/项目STAR），每个考点 wikilink 关联 vault 既有笔记，突出高级岗增量
- 更新 35_高级主题/README（入口）、index.md（新增行）；与【笔记】Unity客户端面试题 互链
- 内容为 LLM 基于 JD 数据 + 既有笔记编译，未编造 sources；建议用户结合自评 Checklist 与 STAR 项目话术复习

## [2026-06-30] query | 面试复习问答系列 4 篇（性能/STAR/热更/渲染）
- 承 [[【笔记】高级Unity游戏开发面试复习地图]]，按用户多选依次产出 4 份独立问答/话术，全部落 35_高级主题
- 新增：[[【笔记】性能优化面试问答]]（CPU/内存GC/渲染/启动/工具，Q1-Q17 + 自测）、[[【片段】项目面试STAR话术]]（云存档/休闲框架两项目的 STAR 骨架 + 高频追问）、[[【笔记】热更新面试问答]]（代码热更方案对比 / 资源热更 / toLua 深入 / 工程流程，严格对齐 [[【设计原理】热更新方案对比]]）、[[【笔记】渲染与Shader面试问答]]（管线 / URP选型 / HLSL / 光照阴影 / 后处理 / Compute / 移动端 TBDR）
- 诚实处理：项目 STAR 话术因 vault 项目复盘为骨架（无实测数据），量化指标全部 [待填]，未编造任何项目指标；性能/热更/渲染问答的关联数据均来自 vault 既有笔记或标注 LLM 编译
- 互链：4 篇与复习地图 / 已有面试题综述 / 性能·渲染·热更专题索引双向关联
- 修复：热更文档「相关文档」节 5 个 wikilink 笔误（【综述）→【笔记】）、渲染文档目录型链接 [[30_性能优化/33_渲染优化]] 改指 README，避免断链
- 更新 35_高级主题/README（入口 +4）、index.md（新增行 +4）；lint 见本次结果

## [2026-06-30] ingest | 新增 Entities 1.4 + Entities Graphics 1.4 官方文档综述
- 读源：Unity 官方手册 com.unity.entities@1.4（1.4.7）+ com.unity.entities.graphics@1.4（1.4.17）
  - https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/index.html
  - https://docs.unity3d.com/Packages/com.unity.entities.graphics@1.4/manual/index.html
  - 抓取方式：3 个并行 agent 分域深读手册子页（Baking/内容管理、Entities Graphics、编程模型）
- 新增综述页：25_DOTS技术栈/【笔记】Entities 1.4 与 Entities Graphics 1.4 官方文档.md（author:llm，sources 含 15 条官方手册 URL，互链 6 篇既有笔记）
- 定位：补全 [[【教程】ECS架构入门]] 未覆盖的 1.4 量产栈三块拼图——Baking 烘焙管线、SubScene/Entity Scene/内容管理、Entities Graphics 渲染；ECS 基础概念不重复，仅修正 1.4 现代 API（LocalTransform 取代旧 Position、MaterialMeshInfo 取代 RenderMesh）
- 核心内容：①范式转移（Conversion→Baking、ISystem 默认、LocalTransform+TransformUsageFlags）；②组件类型全表（含 managed/cleanup/enableable/shared）；③ISystem vs SystemBase（据 systems-comparison.html）+ 生命周期；④Baker 工作流 + TransformUsageFlags 取值表 + IBaker vs Baking System；⑤SubScene/SceneSection 流式 + Resolve→Load + SceneSystem API（LoadSubScene 已弃用）+ Content Management；⑥Entities Graphics 定位（非管线，构建于 BatchRendererGroup）/ Requirements（不支持 Built-in RP 与 WebGL，URP 仅 Forward+）/ 渲染 6 阶段 / MaterialMeshInfo+RenderMeshUtility / Companion / Material Overrides / 性能要点 / 限制速查
- 关键版本事实：**IAspect 在 Entities 1.4 已废弃**（据 aspects-concepts.html），新代码用裸组件查询；Entity Scene 文件格式变更需重建缓存
- 诚实标注：Baking/SubScene/Graphics/ISystem对比/托管组件/Aspects废弃 据官方页面编译；SystemAPI 与系统更新顺序两页官方站点持续 500 抓取失败，相关小节据其它页引用与 ECS 通用机制归纳并已在文内标注「建议对照原文」
- 更新 DOTS专题索引（收录数 6→7、类型分布 +综述、推荐阅读顺序 #3、新增「综述」分类 + 目录条目）；重生成 UnityKnowledge/index.md（289 篇）
- lint：新文档零 issue；总计 ERROR=50/WARN=38 均为既有 UGUI/、Clippings/、性能优化/ 书籍摘录作者署名断链，非本次引入

## [2026-06-30] query | 回填同屏大规模单位渲染方案
- 用户追问「如何实现同屏 10w 单位渲染」，对话答案沉淀为 [[25_DOTS技术栈/【笔记】同屏大规模单位渲染方案]]（author:llm，sources 含 Entities Graphics 综述 + performance/runtime-entity-creation 官方页 + 3 篇互链笔记）
- 核心论点：瓶颈不在「画」而在 CPU 提交（DrawCall）与每帧更新；方案 = ECS + Entities Graphics + Jobs/Burst
- 覆盖：①最少 archetype（每 archetype 一个 batch）②BRG + DOTS Instancing 合批（澄清 per-instance 属性不拆 batch，拆 batch 的是材质/网格/设置/archetype）③IJobEntity+Burst+ScheduleParallel 更新 ④LOD/剔除 ⑤避坑（IEnableableComponent 替代频繁增删、SharedComponent 不每帧改）⑥最小骨架代码（RenderMeshUtility+Instantiate+MoveJob）⑦性能预算（经验估算 10w 位移 ~1-2ms）⑧GPU-driven 进阶与「ECS 非最优解」边界
- 性能数字明确标注为经验估算（非实测基准），建议上线前 Profiler 实测，未编造精确基准
- 更新 DOTS专题索引（收录 7→8、类型分布 +笔记、推荐阅读顺序 #4、新增「笔记」分类 + 目录条目）；重生成 UnityKnowledge/index.md（290 篇）

## [2026-06-30] query | 回填大规模单位动画方案
- 用户追问「10w 单位 × 12 怪 × 每怪 death/move/attack1/attack2/idle 动画如何实现」，对话答案沉淀为 [[25_DOTS技术栈/【笔记】大规模单位动画方案]]（author:llm，sources 含渲染笔记 + Entities Graphics 综述 + material-overrides/mesh-deformations 官方页 + 互链笔记）
- 核心论点：Animator/SkinnedMeshRenderer 撑不到 10w；标准答案是 GPU 顶点动画(VAT) + ECS 状态机驱动，CPU 零骨骼
- 覆盖：①方案选型表（Animator/DOTS Animation/Mesh Deformations/VAT/Sprite 对比，VAT 为 10w 标配）②VAT 原理（顶点×帧纹理 + LUT，12 怪 ×5 动画=60 段烘焙）③ECS 状态机（UnitAnim 组件 + IJobEntity 优先级 死亡>攻击>移动>待机 + MaterialPropertyOverride 传参 + shader 采样伪代码）④12 怪数据组织（MonsterType byte + 索引区分，保持 1 archetype，禁不同组件）⑤VAT 烘焙管线 ⑥性能预算（状态机 10w ~1-3ms，overdraw 才是真瓶颈）⑦避坑（IEnableableComponent 替代 DestroyEntity、事件触发不扫全场、2D 用 sprite 序列）
- 诚实标注：DOTS Animation 标为 1.4 期 preview/千级；性能数字为经验估算非实测基准
- 更新 DOTS专题索引（收录 8→9、笔记分布 1→2、推荐阅读顺序 #5、笔记分类/目录新增条目）；重生成 UnityKnowledge/index.md（291 篇）；lint ERROR=50/WARN=38 不变，新文档零 issue

## [2026-06-30] ingest | 大规模单位工程化三件套（VAT脚本/Demo/AI寻路）
- 承用户「1,2,3 都需要」，新增 3 篇互链笔记（均 author:llm + sources），落 25_DOTS技术栈：
- [[25_DOTS技术栈/【片段】VAT顶点动画烘焙脚本]]：可复用 Editor 脚本骨架——逐帧 `AnimationMode.SampleAnimationClip`+`SkinnedMeshRenderer.BakeMesh` 采样 → 写 RGBAFloat `Texture2D` + `VatLUT`(ScriptableObject)，配套 URP shader 端（SV_VertexID + DOTS Instanced `_AnimIndex/_AnimTime` + LUT StructuredBuffer 采样）。标注多 mesh/法线/half 精度等工程化要点
- [[25_DOTS技术栈/【实战案例】10w单位渲染与动画最小Demo]]：把渲染+VAT动画+状态机串成可跑最小工程骨架——manifest 依赖（URP Forward+）、目录结构、Components/Authoring/Baker、SpawnSystem(`Instantiate(prefab,count)` 批量克隆)、Movement/Anim 系统、Profiler 验证清单（archetype/batch 数、各系统 ms、内存）与已知坑表（batch 爆炸/VAT 错乱/动画同步播放等）。status:草稿，明确标注为参考实现非真实复盘，性能数字为经验估算/待实测
- [[25_DOTS技术栈/【笔记】大规模单位AI决策与寻路]]：10w 寻路/决策——为何 NavMesh/独立 A* 撑不住 → Flow Field 流场（Cost→Integration→Vector 三场，同目标预计算单位 O(1) 查表）、Burst Job 化(桶式 BFS 替代优先队列)、UniformGrid(`NativeMultiHashMap<int3,Entity>`)邻域查询、RVO/boids 避障、分帧决策(分桶轮询/事件驱动/AI LOD)；诚实标注 1.4 期无稳定官方 DOTS NavMesh 包
- 诚实标注：三篇性能数字均为经验估算（非实测基准）；DOTS Animation 标 1.4 期 preview/千级；官方 DOTS NavMesh 缺位；代码为骨架需按项目调整
- 更新 DOTS专题索引（收录 9→12、类型分布 +笔记3/片段1/实战案例1、推荐阅读顺序 #6-#8、新增「片段」「实战案例」分类 + 笔记分类补条目 + 目录条目）；重生成 UnityKnowledge/index.md（294 篇）；lint ERROR=50/WARN=38 不变，三篇零 issue

## [2026-06-30] ingest | 大规模单位计算内核三件套（FlowField/RVO2/战斗结算）
- 承用户「继续深挖」，新增 3 篇互链笔记（均 author:llm + sources），落 25_DOTS技术栈，补全大规模单位的「计算内核」：
- [[25_DOTS技术栈/【片段】FlowField流场Job实现]]：Flow Field 三场完整 Burst Job——CostField + IntegrationField(wavefront Dijkstra 松弛，NativeQueue 并发，标注 Burst 无堆故用桶式/wavefront 替代优先队列) + VectorField(8 邻域梯度 IJobParallelFor)，含对角线代价、穿墙检查、桶式 Dijkstra 扩展要点、跨系统 NativeArray 共享的 JobHandle 依赖；标注可读版用托管数组需改 NativeArray 才能 Burst
- [[25_DOTS技术栈/【片段】RVO2局部避障ECS移植]]：RVO2/ORCA 忠实 Burst 移植（来源 gamma.cs.unc.edu/RVO2 BSD）——XZ 2D 投影、UniformGrid 邻域查询(ComponentLookup)、computeORCALines(无碰撞截断圆锥/leg/碰撞三种分支 + ORCA 半责任 0.5)、linearProgram1/2/3 完整数学(Det2/dot/判别式)、AvoidanceJob(IJobEntity 串联)、参数调优表(NeighborDist/TimeHorizon/MaxNeighbors)与避坑(写竞争/Grid 一致性/MaxNeighbors 截断)
- [[25_DOTS技术栈/【笔记】大规模单位战斗结算]]：战斗 DOTS 事件化——为何 OnTriggerEnter/SendMessage 不行 → 碰撞只产出事件(Unity.Physics ITriggerEventsJob + NativeQueue.ParallelWriter，或自建 UniformGrid) → 伤害 DynamicBuffer<DamageEvent>+ECB 单点结算(无写竞争) → 死亡 IEnableableComponent(AliveTag) 软禁用 + ECB 延迟回收(禁每帧 DestroyEntity) → AOE 复用伤害管线 → 系统时序(碰撞→伤害→回收→动画 UpdateAfter + Lookup 每帧刷新)
- 诚实标注：FlowField 可读版托管数组需改 NativeArray 才能 Burst；RVO2 数学忠实移植自公开 RVO2 库；Unity.Physics API 据 1.4 手册；性能无编造基准，代码为骨架需按项目调
- 更新 DOTS专题索引（收录 12→15、类型分布 +笔记4/片段3、推荐阅读顺序 #9-#11、片段分类补 2 条 + 笔记分类补 1 条 + 目录条目）；重生成 UnityKnowledge/index.md（297 篇）；lint ERROR=50/WARN=38 不变，三篇零 issue

## [2026-07-06] ingest | YooAsset 弱联网与海外安卓分包方案
- 读源：YooAsset 官方文档 ×2
  - 弱联网方案A https://www.yooasset.com/docs/solution/WeakNetworkA
  - 海外安卓分包 https://www.yooasset.com/docs/solution/AndroidInstall
- 新增 [[60_第三方库/【笔记】YooAsset弱联网与海外分包方案]]（author:llm，sources 含两条官方 URL）
- 覆盖：①弱联网方案A——远端优先→本地兜底流程、两个关键初始化参数（`CopyBuiltinPackageManifest=true` 拷贝内置清单、`InstallCleanupMode=None` 覆盖安装不清）、版本号持久化契约（仅完整下载成功才写 PlayerPrefs）、首装离线走 `GetBuildinPackageVersionOperation`、本地完整性校验（`TotalDownloadCount > 0` 即不完整）②海外安卓分包——Play Asset Delivery install-time 模式、目录对齐（`YooFolderName` 与 asset pack 名称必须一致）、业务层零改动
- 与既有 [[【笔记】YooAsset核心概念与架构]] 和 [[【代码片段】YooAsset常用API速查]] 互链
