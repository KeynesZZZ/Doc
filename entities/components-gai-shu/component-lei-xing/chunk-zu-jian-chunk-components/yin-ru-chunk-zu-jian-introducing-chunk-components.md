# 引入 Chunk 组件 (Introducing chunk components)

Chunk 组件在每个 chunk 中存储值，而不是在每个实体中存储值。它们的主要目的是作为一种优化，因为您可以在每个 chunk 级别运行代码，以检查是否需要处理每个 chunk 中所有实体的某些行为。例如，Chunk 组件可以存储其中所有实体的边界。您可以检查这些边界是否在屏幕上，并仅在边界在屏幕上时处理该 chunk 中的实体。

Chunk 组件提供与共享组件类似的功能，但在以下方面有所不同：

* Chunk 组件值从概念上属于 chunk 本身，而不是 chunk 中的各个实体。
* 设置 Chunk 组件值不是结构性更改。
* 与共享组件不同，Unity 不会去重唯一的 Chunk 组件值：具有相同 Chunk 组件值的 chunk 存储它们自己的独立副本。
* Chunk 组件始终是非托管的：您不能创建一个托管的 Chunk 组件。
* 当实体的原型（archetype）改变或实体的共享组件值改变时，Unity 会将实体移动到一个新的 chunk，但这些移动不会修改源 chunk 或目标 chunk 的 Chunk 组件值。
