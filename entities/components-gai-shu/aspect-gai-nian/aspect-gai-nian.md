# Aspect 概念

Aspect 是一种类似对象的包装器，可以将实体的一部分组件组合到一个 C# 结构中。Aspects 对于组织组件代码和简化系统中的查询非常有用。Unity 提供了预定义的 Aspects 用于相关组件组，或者您可以使用 `IAspect` 接口定义自己的 Aspects。

#### Aspects 可以包含以下内容：

* 一个用于存储实体ID的单个 `Entity` 字段。
* `RefRW<T>` 和 `RefRO<T>` 字段，用于访问实现了 `IComponentData` 的类型 T 的组件数据。
* `EnabledRefRW` 和 `EnabledRefRO` 字段，用于访问实现了 `IEnableableComponent` 的组件的启用状态。
* `DynamicBuffer<T>` 字段，用于访问实现了 `IBufferElementData` 的缓冲区元素。
* 任何 `ISharedComponent` 字段，用于以只读方式访问共享组件值。
* 其他 Aspect 类型。
