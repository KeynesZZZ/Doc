# World update allocator

World update allocator 是一个可回退的 allocator，ECS 在每次 world 更新时都会回退它。每个 world 包含在初始化时创建的双重可回退 allocators。

`WorldUpdateAllocatorResetSystem` 系统在每次 world 更新时切换双重可回退 allocators。当一个 allocator 交换进入时，它会回退 allocator。因此，从 world update allocator 分配的生命周期跨越两帧。你不需要手动释放分配，因此没有内存泄漏。

你可以将 world update allocator 的分配传递到一个作业中。你可以通过以下方式访问 world update allocator：

* `World.UpdateAllocator`
* `ComponentSystemBase.WorldUpdateAllocator`
* `SystemState.WorldUpdateAllocator`

**示例：通过 World 访问 world update allocator**

```csharp
// 通过 World.UpdateAllocator 访问 world update allocator.
public void WorldUpdateAllocatorFromWorld_works()
{
    // 创建一个测试 world。
    World world = new World("Test World");

    // 使用 world update allocator 创建一个 native array。
    var nativeArray = CollectionHelper.CreateNativeArray<int>(5, world.UpdateAllocator.ToAllocator);
    for (int i = 0; i < 5; i++)
    {
        nativeArray[i] = i;
    }

    Assert.AreEqual(nativeArray[3], 3);

    // 释放测试 world。
    world.Dispose();
}
```

示例：通过 SystemBase 访问 world update allocator

```csharp
// 通过 SystemBase.WorldUpdateAllocator 访问 world update allocator。
unsafe partial class AllocateNativeArraySystem : SystemBase
{
    public NativeArray<int> nativeArray = default;

    protected override void OnUpdate()
    {
        // 通过 SystemBase.WorldUpdateAllocator 获取 world update allocator。
        var allocator = WorldUpdateAllocator;

        // 使用 world update allocator 创建一个 native array。
        nativeArray = CollectionHelper.CreateNativeArray<int>(5, allocator);
        for (int i = 0; i < 5; i++)
        {
            nativeArray[i] = i;
        }
    }
}

```

示例：通过 SystemState 访问 world update allocator

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
