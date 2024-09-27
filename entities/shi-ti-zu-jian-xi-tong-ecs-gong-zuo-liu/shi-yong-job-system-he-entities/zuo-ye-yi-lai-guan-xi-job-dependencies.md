# 作业依赖关系（Job dependencies）

Unity 根据系统读取和写入的 ECS 组件来分析每个系统的数据依赖关系。如果一个系统在帧早期更新时读取后续系统写入的数据，或者写入后续系统读取的数据，那么第二个系统依赖于第一个系统。为了防止竞争条件，作业调度器确保所有系统所依赖的作业在运行该系统的作业之前已经完成。

### 作业依赖关系更新顺序

系统的 `Dependency` 属性是一个代表系统 ECS 相关依赖关系的 `JobHandle`。在 `OnUpdate()` 之前，`Dependency` 属性反映了系统对先前作业的传入依赖关系。默认情况下，当你在系统中调度作业时，系统会根据每个作业读取和写入的组件来更新 `Dependency` 属性。

### 覆盖默认顺序

要覆盖此默认行为，可以使用 `Entities.ForEach` 和 `Job.WithCode` 的重载版本，这些版本以作业依赖项作为参数并返回更新后的依赖项作为 `JobHandle`。当你使用这些显式版本的构造时，ECS 不会自动将作业句柄与系统的 `Dependency` 属性组合。你必须在需要时手动组合它们。

`Dependency` 属性不跟踪作业可能对通过 `NativeArray` 或其他类似容器传递的数据的依赖关系。如果你在一个作业中写入 `NativeArray`，并在另一个作业中读取该数组，则必须手动将第一个作业的 `JobHandle` 添加为第二个作业的依赖项。你可以使用 `JobHandle.CombineDependencies` 来实现这一点。

#### 示例代码：使用 `JobHandle.CombineDependencies`

```csharp
public partial class DependencyExampleSystem : SystemBase
{
    protected override void OnUpdate()
    {
        NativeArray<float> array = new NativeArray<float>(10, Allocator.TempJob);
        JobHandle handle1 = Job.WithCode(() =>
        {
            for (int i = 0; i < array.Length; i++)
            {
                array[i] = i;
            }
        }).Schedule(Dependency);

        JobHandle handle2 = Job.WithCode(() =>
        {
            for (int i = 0; i < array.Length; i++)
            {
                array[i] *= 2;
            }
        }).Schedule(handle1);

        // Combine the dependencies manually
        Dependency = JobHandle.CombineDependencies(handle1, handle2);

        // Complete the dependency to safely read the results
        Dependency.Complete();

        UnityEngine.Debug.Log("Array sum is " + array.Sum());

        array.Dispose();
    }
}
```

当你调用 Entities.ForEach.Run() 时，作业调度器会在开始 ForEach 迭代之前完成系统依赖的所有已调度作业。如果你还使用 WithStructuralChanges() 作为构造的一部分，那么作业调度器会完成所有正在运行和已调度的作业。结构更改也会使对组件数据的任何直接引用失效。更多信息请参阅 结构更改文档。

```csharp
public partial class StructuralChangeSystem : SystemBase
{
    protected override void OnUpdate()
    {
        Entities
            .WithAll<SomeComponent>()
            .ForEach((Entity entity, int entityInQueryIndex, ref SomeComponent someComponent) =>
            {
                someComponent.Value += 1;
            })
            .Run(); // Ensures all dependencies are completed

        Entities
            .WithStructuralChanges()
            .ForEach((Entity entity, int entityInQueryIndex, ref SomeOtherComponent otherComponent) =>
            {
                EntityManager.AddComponentData(entity, new AnotherComponent { Value = 42 });
            })
            .Run(); // Ensures all running and scheduled jobs are completed
    }
}

```
