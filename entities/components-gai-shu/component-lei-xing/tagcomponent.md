# TagComponent

TagComponent是存储无数据且不占用空间的非托管组件。

从概念上讲，标签组件的作用类似于 GameObject 的标签，它们在查询中非常有用，因为你可以根据实体是否具有标签组件进行过滤。例如，你可以将它们与清理组件一起使用，并通过过滤实体来执行清理。

### 创建一个标签组件

要创建一个标签组件，请创建一个没有任何属性的非托管组件。

以下代码示例显示了一个标签组件：

```csharp
public struct ExampleTagComponent : IComponentData
{

}
```
