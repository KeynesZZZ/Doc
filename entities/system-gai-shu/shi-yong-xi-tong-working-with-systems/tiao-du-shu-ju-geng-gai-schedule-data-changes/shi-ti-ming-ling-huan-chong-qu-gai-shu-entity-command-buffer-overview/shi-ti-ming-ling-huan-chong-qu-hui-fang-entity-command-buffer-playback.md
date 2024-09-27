# 实体命令缓冲区回放 (Entity Command Buffer Playback)

如果你在并行作业中跨多个线程分割实体命令缓冲区（ECB）中的命令记录，那么这些命令的顺序是非确定性的，因为它们依赖于作业调度。

### 非确定性与确定性

确定性并不总是必需的，但生成确定性结果的代码更容易调试。在某些网络场景下，不同机器之间需要一致的结果。然而，确定性会对性能产生影响，因此在某些项目中你可能愿意接受非确定性。

### 并行作业中的确定性回放 (Deterministic Playback in Parallel Jobs)

你无法避免并行作业中记录的非确定性顺序，但可以通过以下方式使命令的回放顺序变得确定：

1. 记录一个 `int` 类型的排序键作为每个 ECB 方法的第一个参数。
2. 在回放之前使用排序键对命令进行排序，然后再执行命令。

如果记录的排序键与调度无关，那么排序使回放顺序确定。Unity 总是先回放较小排序键的命令，再回放较大排序键的命令。

#### 示例

以下示例代码展示了在并行作业中如何使用 ECB 并实现确定性回放：

```csharp
[RequireMatchingQueriesForUpdate]
partial struct MultiThreadedSchedule_ECB : ISystem
{
    partial struct ParallelRecordingJob : IJobEntity
    {
        internal EntityCommandBuffer.ParallelWriter ecbParallel;

        // ChunkIndexInQuery 对查询中的每个 chunk 是唯一的，并且无论调度如何都会保持一致。
        // 这将导致 ECB 的确定性回放。
        void Execute(Entity entity, [ChunkIndexInQuery] int sortKey, in FooComp foo)
        {
            if (foo.Value > 0)
            {
                // 第一个参数是记录命令的 'sort key'。
                ecbParallel.AddComponent<BarComp>(sortKey, entity);
            }
        }
    }

    public void OnUpdate(ref SystemState state)
    {
        EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);

        // 我们需要跨线程并发写入 ECB。
        new ParallelRecordingJob { ecbParallel = ecb.AsParallelWriter() }.Schedule();

        // 回放是单线程的。注意，显式完成会添加一个同步点，这里仅用于演示目的。
        // （让现有的 EntityCommandBufferSystem 进行回放不会引入额外的同步点。）
        state.Dependency.Complete();

        // 为了确保确定性的回放顺序，
        // 命令首先根据它们的排序键进行排序。
        ecb.Playback(state.EntityManager);

        ecb.Dispose();
    }
}
```

## 多次回放 (Multi Playback)

如果你多次调用 `Playback` 方法，会抛出异常。为了避免这种情况，可以使用 `PlaybackPolicy.MultiPlayback` 选项创建一个 `EntityCommandBuffer` 实例：

### 示例

```csharp
// ... 在系统更新中

EntityCommandBuffer ecb = 
    new EntityCommandBuffer(Allocator.TempJob, PlaybackPolicy.MultiPlayback);

// ... 记录命令

ecb.Playback(state.EntityManager);

// 多次回放是可以的，因为这个 ECB 是 MultiPlayback。
ecb.Playback(state.EntityManager);

ecb.Dispose();
```

**注意事项**&#x20;

多次回放只能在带有 PlaybackPolicy.MultiPlayback 的 EntityCommandBuffer 上进行，否则会引发异常。 确保在完成所有操作后正确释放命令缓冲区，以避免内存泄漏。

#### 应用场景&#x20;

你可以使用多次回放来重复生成一组实体。要实现这一点，可以用 EntityCommandBuffer 创建并配置一组新实体，然后重复回放以重新生成另一组匹配的实体。

**示例：反复生成实体**

```csharp
protected override void OnUpdate()
{
    // 创建一个带有多次回放策略的实体命令缓冲区
    EntityCommandBuffer ecb = 
        new EntityCommandBuffer(Allocator.TempJob, PlaybackPolicy.MultiPlayback);

    // 记录命令来创建和配置一组实体
    Entities
        .ForEach((Entity entity, in FooComp foo) =>
        {
            if (foo.Value > 0)
            {
                ecb.AddComponent<BarComp>(entity);
            }
        }).WithoutBurst().Run();

    // 第一次回放
    ecb.Playback(EntityManager);

    // 重复生成另一组匹配的实体
    ecb.Playback(EntityManager);

    // 释放命令缓冲区
    ecb.Dispose();
}

```
