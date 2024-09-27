# 自动回放和释放实体命令缓冲区 (Automatic Playback and Disposal of Entity Command Buffers)

利用 `EntityCommandBufferSystem`，你可以自动回放和释放实体命令缓冲区（ECBs），而无需手动完成这些操作。以下是实现步骤：

1. 获取要执行回放的 `EntityCommandBufferSystem` 的单例实例。
2. 使用该单例创建一个 `EntityCommandBuffer` 实例。
3. 向 `EntityCommandBuffer` 记录命令。

### 示例

```csharp
// ... 在一个系统中

// 假设存在一个名为 FooECBSystem 的 EntityCommandBufferSystem。
// 调用 GetSingleton 会自动注册作业，以便由 ECB 系统完成。
var singleton = SystemAPI.GetSingleton<FooECBSystem.Singleton>();

// 创建一个命令缓冲区，该缓冲区将由 MyECBSystem 回放和释放。
EntityCommandBuffer ecb = singleton.CreateCommandBuffer(state.WorldUnmanaged);

// 一个 IJobEntity，没有参数调度隐式地将返回的 JobHandle 分配给 this.Dependency
new MyParallelRecordingJob() { ecbParallel = ecb.AsParallelWriter() }.Schedule();
```

### 重要提示

**不要手动回放或释放由 `EntityCommandBufferSystem` 创建的 `EntityCommandBuffer`。** 当 `EntityCommandBufferSystem` 运行时，它会为你执行这两个操作。

## `EntityCommandBufferSystem` 的工作方式

在每次更新中，`EntityCommandBufferSystem` 会：

1. **完成所有注册的作业**，以及针对其单例组件调度的所有作业。这确保了所有相关的作业都已完成它们的记录。
2. **按创建顺序回放通过系统创建的所有 ECBs**。
3. **释放 `EntityCommandBuffer` 实例**。

## 默认的 `EntityCommandBufferSystem` 系统

在默认世界中，存在以下默认的 `EntityCommandBufferSystem` 系统：

* `BeginInitializationEntityCommandBufferSystem`
* `EndInitializationEntityCommandBufferSystem`
* `BeginFixedStepSimulationEntityCommandBufferSystem`
* `EndFixedStepSimulationEntityCommandBufferSystem`
* `BeginVariableRateSimulationEntityCommandBufferSystem`
* `EndVariableRateSimulationEntityCommandBufferSystem`
* `BeginSimulationEntityCommandBufferSystem`
* `EndSimulationEntityCommandBufferSystem`
* `BeginPresentationEntityCommandBufferSystem`

由于结构性更改不能发生在 Unity 将渲染数据提供给渲染器之后，因此没有 `EndPresentationEntityCommandBufferSystem` 系统。你可以使用 `BeginInitializationEntityCommandBufferSystem` 代替：一帧的结束是下一帧的开始。

这些 `EntityCommandBufferSystem` 系统在标准系统组的开始和结束以及固定速率和可变速率模拟组的开始和结束进行更新。有关更多信息，请参阅 [系统更新顺序](https://docs.unity3d.com/Manual/system-update-order.html) 的文档。

### 自定义 `EntityCommandBufferSystem`

如果默认系统不适用于你的应用程序，那么你可以创建自己的 `EntityCommandBufferSystem`：

#### 示例代码

```csharp
// 你应该指定这个 ECB 系统应在帧中的确切位置更新。
[UpdateInGroup(typeof(SimulationSystemGroup))]
[UpdateAfter(typeof(FooSystem))]
public partial class MyECBSystem : EntityCommandBufferSystem
{
    // 使用单例组件数据访问模式安全地访问命令缓冲区系统。
    // 该数据将存储在派生的 ECB 系统的系统实体中。

    public unsafe struct Singleton : IComponentData, IECBSingleton
    {
        internal UnsafeList<EntityCommandBuffer>* pendingBuffers;
        internal AllocatorManager.AllocatorHandle allocator;

        public EntityCommandBuffer CreateCommandBuffer(WorldUnmanaged world)
        {
            return EntityCommandBufferSystem
                .CreateCommandBuffer(ref *pendingBuffers, allocator, world);
        }

        // IECBSingleton 所需方法
        public void SetPendingBufferList(ref UnsafeList<EntityCommandBuffer> buffers)
        {
            var ptr = UnsafeUtility.AddressOf(ref buffers);
            pendingBuffers = (UnsafeList<EntityCommandBuffer>*)ptr;
        }

        // IECBSingleton 所需方法
        public void SetAllocator(Allocator allocatorIn)
        {
            allocator = allocatorIn;
        }

        // IECBSingleton 所需方法
        public void SetAllocator(AllocatorManager.AllocatorHandle allocatorIn)
        {
            allocator = allocatorIn;
        }
    }

    protected override void OnCreate()
    {
        base.OnCreate();

        this.RegisterSingleton<Singleton>(ref PendingBuffers, World.Unmanaged);
    }
}
```

## 延迟创建的实体 (Deferred Entities)

`EntityCommandBuffer` 的方法 `CreateEntity` 和 `Instantiate` 用于记录创建实体的命令。这些方法仅记录命令，而不实际创建实体。因此，它们返回带有负索引的 `Entity` 值，这些值表示尚不存在的占位符实体。这些占位符 `Entity` 值仅在同一个 ECB 的记录命令中有意义。

### 示例

#### 创建和使用占位符实体

```csharp
// ... 在一个系统中

EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);

Entity placeholderEntity = ecb.CreateEntity();

// 在同一个 ECB 的后续命令中使用占位符实体是有效的。
ecb.AddComponent<FooComp>(placeholderEntity);

// 实体被创建，并且 FooComp 被添加到真正的实体上。
ecb.Playback(state.EntityManager);

// 异常！占位符实体在创建它的 ECB 之外没有任何意义，即使在回放之后也如此。
state.EntityManager.AddComponent<BarComp>(placeholderEntity);

ecb.Dispose();
```

### 示例

组件中的占位符实体重映射

```csharp
// ... 在一个系统中

EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);

// 对于所有具有 FooComp 组件的实体...
foreach (var (f, e) in SystemAPI.Query<FooComp>().WithEntityAccess())
{
    // 在回放时，将创建一个与此占位符实体对应的实际实体。
    Entity placeholderEntity = ecb.CreateEntity();

    // (假设 BarComp 有一个名为 TargetEnt 的 Entity 字段。)
    BarComp bar = new BarComp { TargetEnt = placeholderEntity };

    // 在回放时，TargetEnt 将分配给对应于占位符实体的实际实体。
    ecb.AddComponent(e, bar);
}

// 回放后，每个具有 FooComp 组件的实体现在都有一个 BarComp 组件，
// 其中的 TargetEnt 引用一个新的实体。
ecb.Playback(state.EntityManager);

ecb.Dispose();

```
