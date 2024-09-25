# Unmanaged components

非托管组件存储最常见的数据类型，这意味着它们在大多数使用场景中都很有用。

非托管组件可以存储以下类型的字段：

* Blittable 类型
* bool
* char
* `BlobAssetReference<T>`（对 Blob 数据结构的引用）
* `Collections.FixedString`（固定大小的字符缓冲区）
* `Collections.FixedList`
* 固定数组（仅在不安全的上下文中允许）
* 其他符合这些限制的结构体

### 创建一个非托管组件

要创建一个非托管组件，请创建一个继承自 `IComponentData` 的结构体。

以下代码示例显示了一个非托管组件：

```csharp
public struct ExampleUnmanagedComponent : IComponentData
{
    public int Value;
}
```

向结构体添加使用兼容类型的字段，以定义组件的数据。如果您没有向组件添加任何字段，它将作为一个标签组件。
