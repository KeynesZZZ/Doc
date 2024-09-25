# 可启用组件 (Enableable components)

使用可启用组件可以在运行时禁用或启用实体上的单个组件。当处理预期频繁且不可预测变化的状态时，这非常有用，因为它们比添加或移除组件产生的结构性更改要少。

#### 主题描述

| 主题      | 描述                      |
| ------- | ----------------------- |
| 可启用组件概述 | 概述可启用组件，您可以在运行时禁用或启用它们。 |
| 使用可启用组件 | 有关使用可启用组件的信息。           |

#### 可启用组件概述 (Enableable components overview)

可启用组件允许您在运行时动态地控制每个组件的启用状态，而无需进行传统的组件添加或移除操作。这种方法减少了结构性更改，提高了应用程序的性能和灵活性。

#### 使用可启用组件 (Using enableable components)

启用或禁用可启用组件非常简单。以下是一些常见操作示例：

<pre class="language-csharp"><code class="lang-csharp">//启用
<strong>EntityManager.SetComponentEnabled&#x3C;MyComponent>(entity, true); 
</strong>//禁用
EntityManager.SetComponentEnabled&#x3C;MyComponent>(entity, false); 
//检查组件是否启用
bool isEnabled = EntityManager.IsComponentEnabled&#x3C;MyComponent>(entity);

</code></pre>
