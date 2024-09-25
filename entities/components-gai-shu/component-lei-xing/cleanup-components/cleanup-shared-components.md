# Cleanup shared components

清理共享组件是具有清理组件销毁语义的共享组件。它们对于标记需要相同清理信息的实体非常有用。

### 创建一个清理共享组件

要创建一个清理共享组件，请创建一个继承自 `ICleanupSharedComponentData` 的结构体。

以下代码示例显示了一个空的系统清理组件：

```csharp
public struct ExampleSharedCleanupComponent : ICleanupSharedComponentData
{
    public int Value;
}
```
