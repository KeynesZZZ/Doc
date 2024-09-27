# Define and manage system data

在定义如何构建系统级别数据时，你应该将其组织为组件级别的数据，而不是作为系统类型内的字段。

在系统中使用公共数据不是最佳实践。这是因为在系统中访问公共数据需要直接引用或指向系统实例，这会导致以下后果：

* 它在系统之间创建依赖关系，这与面向数据的方式相冲突。
* 它无法保证访问系统实例时的线程或生命周期安全。
* 它无法保证访问系统数据时的线程或生命周期安全，即使系统仍然存在并以线程安全的方式访问它。

### 将系统数据存储在组件中

你应该将系统中的公开可访问数据存储在组件中，而不是作为系统类型内的字段。例如，`World` 命名空间中的 `Get` 和 `Create` 系统 API（如 `GetExistingSystem<T>`）返回一个不透明的 `SystemHandle` 句柄，而不是直接访问系统实例。这适用于托管的 `SystemBase` 和非托管的 `ISystem` 系统。

#### 示例

在典型的面向对象代码实现中，一个类型的数据是该类型定义的一部分：

```csharp
/// 面向对象代码示例
public partial struct PlayerInputSystem : ISystem
{
    public float AxisX;
    public float AxisY;

    public void OnCreate(ref SystemState state) { }

    public void OnUpdate(ref SystemState state)
    {
        AxisX = [... read controller input];
        AxisY = [... read controller input];
    }

    public void OnDestroy(ref SystemState state) { }
}
```

### 数据导向的 PlayerInputSystem 实现

下面是上述 `PlayerInputSystem` 的一个替代数据导向版本：

```csharp
public struct PlayerInputData : IComponentData
{
    public float AxisX;
    public float AxisY;
}

public partial struct PlayerInputSystem : ISystem
{
    public void OnCreate(ref SystemState state)
    {
        // 在系统创建时添加组件 PlayerInputData
        state.EntityManager.AddComponent<PlayerInputData>(state.SystemHandle);
    }

    public void OnUpdate(ref SystemState state)
    {
        // 每次更新时设置组件数据
        SystemAPI.SetComponent(state.SystemHandle, new PlayerInputData
        {
            AxisX = [...read controller data],  // 读取控制器输入
            AxisY = [...read controller data]   // 读取控制器输入
        });
    }

    // 组件数据在系统销毁时自动销毁。
    // 如果组件中存在原生容器（Native Container），可以在 OnDestroy 中确保内存被释放。
    public void OnDestroy(ref SystemState state) { }
}
```

### 数据协议与系统功能分离

这种方法为系统定义了一个数据协议，与系统功能分离。这些组件可以存在于单例实体中，或者通过 `EntityManager.GetComponentData<T>(SystemHandle)` 和类似方法属于一个与系统关联的实体。当你希望数据的生命周期与系统生命周期绑定时，应使用后者。

#### 好处

* **访问方式一致**：使用此技术，你可以像访问任何其他实体组件数据一样访问系统的数据。不再需要对系统实例的引用或指针。
* **生命周期管理**：系统关联的实体保证了数据的生命周期与系统同步，从而简化了内存管理和清理过程。

### 选择系统或单例实体组件

使用单例 API 类似于在实体组件中使用系统数据，但有以下区别：

* **生命周期**：单例与系统的生命周期无关。
* **实例限制**：单例只能存在于每个系统类型，而不是每个系统实例。
