# 使用实体命令缓冲区 (Use an Entity Command Buffer)

你可以在作业中和主线程上记录实体命令缓冲区（ECBs）。

### 在作业中使用实体命令缓冲区 (Use an Entity Command Buffer in a Job)

你不能在作业中执行结构性更改，除非是在 `ExclusiveEntityTransaction` 中，因此你可以使用 ECB 来记录结构性更改，并在作业完成后回放这些更改。例如：

```csharp
protected override void OnUpdate()
{
    // 你不需要指定大小，因为缓冲区会根据需要增长。
    EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);

    // ECB 被 ForEach 作业捕获。
    // 在作业完成之前，作业拥有 ECB 的作业安全句柄。
    Entities
        .ForEach((Entity e, in FooComp foo) =>
        {
            if (foo.Value > 0)
            {
                // 记录一个稍后会向实体添加 BarComp 的命令。
                ecb.AddComponent<BarComp>(e);
            }
        }).Schedule();
    
    // 确保作业已经完成。
    Dependency.Complete();

    // 现在作业已经完成，你可以执行这些更改。
    // 注意，Playback 只能在主线程上调用。
    ecb.Playback(EntityManager);

    // 你有责任释放任何创建的 ECB。
    ecb.Dispose();
}
```

## 并行作业 (Parallel Jobs)

如果你想在并行作业中使用 ECB，请使用 `EntityCommandBuffer.ParallelWriter`，它可以以线程安全的方式并发记录到命令缓冲区。

### 示例

```csharp
// 创建一个实体命令缓冲区
EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);

// 这个 writer 的方法可以以线程安全的方式记录命令到 EntityCommandBuffer 中
EntityCommandBuffer.ParallelWriter parallelEcb = ecb.AsParallelWriter();

protected override void OnUpdate()
{
    // 捕获 ParallelWriter 到 ForEach 作业中
    Entities
        .ForEach((Entity entity, int entityInQueryIndex, in FooComp foo) =>
        {
            if (foo.Value > 0)
            {
                // 使用 parallelEcb 记录一个命令，该命令稍后会向实体添加 BarComp
                parallelEcb.AddComponent<BarComp>(entityInQueryIndex, entity);
            }
        }).ScheduleParallel();
    
    // 确保作业已经完成
    Dependency.Complete();

    // 回放命令缓冲区中的所有操作（只能在主线程上进行）
    ecb.Playback(EntityManager);

    // 释放命令缓冲区
    ecb.Dispose();
}
```

注意：只有记录需要是线程安全的，以便在并行作业中实现并发。回放总是在主线程上单线程进行。

有关并行作业中确定性回放的信息，请参阅文档中的 实体命令缓冲区回放 部分。

## 在主线程上使用实体命令缓冲区 (Use an Entity Command Buffer on the Main Thread)

你可以在主线程上记录 ECB 更改，适用于以下情况：

* **延迟你的更改**。
* **多次回放一组更改**。要实现这一点，请参阅有关[多次回放](https://docs.unity3d.com/Manual/entity-command-buffer-multi-playback.html)的信息。
* **在一个整合的地方回放大量不同种类的更改**。这比在帧的不同部分中间插入更改更加高效。

每个结构性更改操作都会触发一个同步点，这意味着该操作必须等待某些或所有已调度的作业完成。如果将这些结构性更改合并到一个 ECB 中，则该帧的同步点会更少。

### 示例

```csharp
protected override void OnUpdate()
{
    // 创建一个实体命令缓冲区
    EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);

    Entities
        .ForEach((Entity entity, in FooComp foo) =>
        {
            if (foo.Value > 0)
            {
                // 在 ECB 中记录添加 BarComp 的命令
                ecb.AddComponent<BarComp>(entity);
            }
        }).WithoutBurst().Run(); // Managed API calls should run without Burst compilation

    // 回放命令缓冲区中的所有操作（只能在主线程上进行）
    ecb.Playback(EntityManager);

    // 释放命令缓冲区
    ecb.Dispose();
}
```
