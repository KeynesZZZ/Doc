# Managed components

与非托管组件不同，托管组件可以存储任何类型的属性。然而，它们在存储和访问时更耗资源，并且有以下限制：

* 不能在作业中访问它们。
* 不能在 Burst 编译的代码中使用它们。
* 它们需要垃圾回收。
* 出于序列化目的，它们必须包含一个无参数的构造函数。

### 托管类型属性

如果托管组件中的属性使用了托管类型，您可能需要手动添加克隆、比较和序列化属性的能力。

### 创建一个托管组件

要创建一个托管组件，请创建一个继承自 `IComponentData` 的类，并且没有构造函数或包含一个无参数的构造函数。

以下代码示例显示了一个托管组件：

```csharp
public class ExampleManagedComponent : IComponentData
{
    public int Value;
}
```

### 管理外部资源的生命周期

对于引用外部资源的托管组件，最佳实践是实现 ICloneable 和 IDisposable，例如，对于存储对 ParticleSystem 引用的托管组件。

如果您复制这个托管组件的实体，默认情况下会创建两个托管组件，它们都引用同一个粒子系统。如果您为托管组件实现了 ICloneable，可以为第二个托管组件复制粒子系统。如果您销毁了托管组件，默认情况下粒子系统会保留。如果您为托管组件实现了 IDisposable，可以在销毁组件时销毁粒子系统。

```csharp
public class ManagedComponentWithExternalResource : IComponentData, IDisposable, ICloneable
{
    public ParticleSystem ParticleSystem;

    public void Dispose()
    {
        UnityEngine.Object.Destroy(ParticleSystem);
    }

    public object Clone()
    {
        return new ManagedComponentWithExternalResource { ParticleSystem = UnityEngine.Object.Instantiate(ParticleSystem) };
    }
}

```

### **优化托管组件**&#x20;

与非托管组件不同，Unity 不会直接将托管组件存储在区块中。相反，Unity 会将它们存储在整个世界的大数组中。然后，区块存储相关托管组件的数组索引。这意味着当你访问一个实体的托管组件时，Unity 会进行一次额外的索引查找。这使得托管组件比非托管组件效率低下。

托管组件的性能影响意味着应尽可能使用非托管组件。
