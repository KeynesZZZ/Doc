# System group allocator

每个 `ComponentSystemGroup` 在设置其速率管理器时都有一个创建系统组 allocator 的选项。为此，使用 `ComponentSystemGroup.SetRateManagerCreateAllocator`。如果你通过属性 `RateManager` 设置系统组中的速率管理器，则组件系统组不会创建系统组 allocator。

以下示例使用 `ComponentSystemGroup.SetRateManagerCreateAllocator` 设置速率管理器并创建系统组 allocator：

```csharp
[WorldSystemFilter(WorldSystemFilterFlags.Default | WorldSystemFilterFlags.Editor | WorldSystemFilterFlags.ThinClientSimulation)]
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup), OrderFirst = true)]
public partial class FixedStepTestSimulationSystemGroup : ComponentSystemGroup
{
    // 设置该组使用的时间步长（以秒为单位）。默认值是 1/60 秒。
    // 此值将被限制在 [0.0001f ... 10.0f] 范围内。
    public float Timestep
    {
        get => RateManager != null ? RateManager.Timestep : 0;
        set
        {
            if (RateManager != null)
                RateManager.Timestep = value;
        }
    }

    // 默认构造函数
    public FixedStepTestSimulationSystemGroup()
    {
        float defaultFixedTimestep = 1.0f / 60.0f;

        // 设置 FixedRateSimpleManager 为速率管理器并创建系统组 allocator
        SetRateManagerCreateAllocator(new RateUtils.FixedRateSimpleManager(defaultFixedTimestep));
    }
}
```

#### 使用 `World.SetGroupAllocator` 和 `World.RestoreGroupAllocator`

创建系统组 allocator 的组件系统组包含双重可回退 allocators。`World.SetGroupAllocator` 和 `World.RestoreGroupAllocator` 用于在 `IRateManager.ShouldGroupUpdate` 中将 world update allocator 替换为系统组 allocator，并在稍后恢复 world update allocator。

以下示例展示了如何使用 `World.SetGroupAllocator` 和 `World.RestoreGroupAllocator`：

```csharp
public unsafe class FixedRateSimpleManager : IRateManager
{
    const float MinFixedDeltaTime = 0.0001f;
    const float MaxFixedDeltaTime = 10.0f;

    float m_FixedTimestep;
    public float Timestep
    {
        get => m_FixedTimestep;
        set => m_FixedTimestep = math.clamp(value, MinFixedDeltaTime, MaxFixedDeltaTime);
    }

    double m_LastFixedUpdateTime;
    bool m_DidPushTime;

    DoubleRewindableAllocators* m_OldGroupAllocators = null;

    public FixedRateSimpleManager(float fixedDeltaTime)
    {
        Timestep = fixedDeltaTime;
    }

    public bool ShouldGroupUpdate(ComponentSystemGroup group)
    {
        // 如果为 true，表示我们在循环中被第二次或更多次调用。
        if (m_DidPushTime)
        {
            group.World.PopTime();
            m_DidPushTime = false;

            // 更新组 allocators 并恢复旧的 allocator。
            group.World.RestoreGroupAllocator(m_OldGroupAllocators);

            return false;
        }

        group.World.PushTime(new TimeData(
            elapsedTime: m_LastFixedUpdateTime,
            deltaTime: m_FixedTimestep));

        m_LastFixedUpdateTime += m_FixedTimestep;

        m_DidPushTime = true;

        // 备份当前的 world 或组 allocator。
        m_OldGroupAllocators = group.World.CurrentGroupAllocators;
        // 用这个系统组 allocator 替换当前的 world 或组 allocator。
        group.World.SetGroupAllocator(group.RateGroupAllocators);

        return true;
    }
}
```

#### System group allocator

系统组 allocator 包含双重可回退 allocators，其工作方式与 world update allocator 类似。在一个系统组进行更新之前，它的系统组 allocator 被放入 world update allocator 中，并且从 world update allocator 分配的内存实际上是从系统组 allocator 分配的。

如果系统组跳过了它的更新，它会切换系统组 allocator 的双重可回退 allocators，回退交换进入的那个，然后恢复 world update allocator。由于这是一个双重可回退 allocator，从系统组 allocator 进行分配的生命周期持续两个系统组更新周期。你不需要手动释放这些分配，因此没有内存泄漏。

在下面的示例中，系统组 allocator 被用于 `ExampleSystemGroupAllocatorSystem`，该系统位于具有速率管理器 `FixedRateSimpleManager` 的固定速率系统组中，如上所示。

**示例：通过 SystemState 使用系统组 allocator**

```csharp
// 通过 SystemState.WorldUpdateAllocator 访问 world update allocator。
unsafe partial struct AllocateNativeArrayISystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // 通过 SystemState.WorldUpdateAllocator 获取 world update allocator。
        var allocator = state.WorldUpdateAllocator;

        // 使用 world update allocator 创建一个 native array。
        var nativeArray = CollectionHelper.CreateNativeArray<int>(10, allocator);

        for (int i = 0; i < 10; i++)
        {
            nativeArray[i] = i;
        }
    }
}
```
