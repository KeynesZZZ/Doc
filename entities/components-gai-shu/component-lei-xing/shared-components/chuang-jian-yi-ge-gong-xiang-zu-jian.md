# 创建一个共享组件

你可以创建托管和非托管共享组件。

### 创建一个非托管共享组件

要创建一个非托管共享组件，请创建一个实现标记接口 `ISharedComponentData` 的结构体。

以下代码示例显示了一个非托管共享组件：

```csharp
public struct ExampleUnmanagedSharedComponent : ISharedComponentData
{
    public int Value;
}
```

### 创建一个托管共享组件

要创建一个托管共享组件，请创建一个实现标记接口 `ISharedComponentData` 和 `IEquatable<>` 的结构体，并确保实现了 `public override int GetHashCode()`。这些相等性方法是必要的，以确保比较不会由于使用默认的 `Equals` 和 `GetHashCode` 实现时的隐式装箱而生成托管分配。

以下代码示例显示了一个托管共享组件：

```csharp
public struct ExampleManagedSharedComponent : ISharedComponentData, IEquatable<ExampleManagedSharedComponent>
{
    public string Value; // 一个托管字段类型

    public bool Equals(ExampleManagedSharedComponent other)
    {
        return Value.Equals(other.Value);
    }

    public override int GetHashCode()
    {
        return Value.GetHashCode();
    }
}
```
