# 烘焙阶段(Baking phases)

烘焙有多个阶段，但有两个关键步骤：

1. **Bakers**：在这个阶段，bakers 将 GameObjects 上的创作组件转换为实体和组件。
2. **烘焙系统**：在这个阶段，烘焙系统对这些实体进行额外处理。

### 实体创建

在 bakers 运行之前，Unity 会为子场景中的每个创作 GameObject 创建一个实体。在此阶段，实体除了某些内部元数据外，不包含任何组件。

### Baker 阶段

在 Unity 创建实体之后，它会运行 bakers。每个 baker 处理特定的创作组件类型，并且多个 bakers 可以消费相同类型的创作组件。

Entities 包和使用实体的任何包都会提供一组默认的 bakers。例如，Entities Graphics 包有用于渲染器的 bakers，Unity Physics 有用于刚体的 bakers。这并不妨碍你添加更多的 bakers 来进一步处理这些相同类型。

> **注意**： Unity 运行 bakers 的顺序没有保证。因此，不允许 bakers 之间存在相互依赖关系。这意味着 bakers 不能读取或更改实体的组件，它们只能添加新的组件。
>
> 此外，每个 baker 只允许更改其自己的实体或它生成的实体。在此阶段访问和修改其他实体会破坏逻辑并导致未定义行为。

### 烘焙系统阶段

在所有 bakers 运行之后，Unity 会运行烘焙系统。烘焙系统是具有 BakingSystem 属性的 ECS 系统，用于指定它们只能在烘焙过程中运行。你可以使用 `UpdateAfter`、`UpdateBefore` 和 `UpdateInGroup` 属性来排序烘焙系统。以下默认组按以下顺序运行：

1. **PreBakingSystemGroup**（在 bakers 之前执行）
2. **TransformBakingSystemGroup**
3. **BakingSystemGroup**（默认烘焙系统组）
4. **PostBakingSystemGroup**

> **注意**： 所有烘焙系统都在任何 bakers 之后运行，除了 PreBakingSystemGroup 的烘焙系统。PreBakingSystemGroup 不仅在 bakers 之前运行，还在实体创建之前运行。

在 Unity 运行所有烘焙系统组之后，它会将实体数据存储在实体场景中并序列化到磁盘，或者在实时烘焙的情况下直接反映到主 ECS 世界中。
