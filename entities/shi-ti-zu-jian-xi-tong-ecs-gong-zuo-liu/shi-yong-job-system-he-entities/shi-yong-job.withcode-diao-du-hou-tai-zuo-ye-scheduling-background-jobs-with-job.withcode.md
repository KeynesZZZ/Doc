# 使用 Job.WithCode 调度后台作业（Scheduling background jobs with Job.WithCode）

`SystemBase` 类中的 `Job.WithCode` 构造可以作为单个后台作业运行一个方法。你也可以在主线程上运行 `Job.WithCode` 并利用 Burst 编译来加速执行。

### 使用 Job.WithCode

以下示例使用一个 `Job.WithCode` lambda 表达式填充一个原生数组（NativeArray）随机数，并使用另一个作业将这些数字相加：

#### 示例代码：使用 Job.WithCode

```csharp
public partial class RandomSumJob : SystemBase
{
    private uint seed = 1;

    protected override void OnUpdate()
    {
        Random randomGen = new Random(seed++);
        NativeArray<float> randomNumbers = new NativeArray<float>(500, Allocator.TempJob);

        Job.WithCode(() =>
        {
            for (int i = 0; i < randomNumbers.Length; i++)
            {
                randomNumbers[i] = randomGen.NextFloat();
            }
        }).Schedule();

        // 要从作业中获取数据，必须使用 NativeArray，即使只有一个值
        NativeArray<float> result = new NativeArray<float>(1, Allocator.TempJob);

        Job.WithCode(() =>
        {
            for (int i = 0; i < randomNumbers.Length; i++)
            {
                result[0] += randomNumbers[i];
            }
        }).Schedule();

        // 这将完成调度的作业以立即获取结果，但为了更高效，
        // 应该在帧早期的系统中调度作业，并在帧后期的不同系统中获取结果。
        this.CompleteDependency();
        UnityEngine.Debug.Log("The sum of " + randomNumbers.Length + " numbers is " + result[0]);

        randomNumbers.Dispose();
        result.Dispose();
    }
}
```

要运行并行作业，可以实现 IJobFor 接口。你可以使用 ScheduleParallel() 在系统的 OnUpdate() 函数中调度并行作业。



## 捕获变量

你不能将参数传递给 `Job.WithCode` lambda 表达式或返回一个值。相反，你必须在系统的 `OnUpdate()` 函数中捕获局部变量。

### 在使用 Schedule() 调度作业时的限制

* 你必须将捕获的变量声明为 `NativeArray`、本机容器（native container）或可直接复制的类型（blittable type）。
* 要返回数据，必须将返回值写入捕获的 `NativeArray`，即使数据是一个单一值。然而，如果你使用 `Run()` 执行作业，可以写入任何捕获变量。

`Job.WithCode` 有一组方法可以应用只读和安全属性到捕获的本机容器变量。例如，你可以使用 `WithReadOnly` 将变量访问限制为只读。你也可以使用 `WithDisposeOnCompletion` 在作业完成后自动释放容器。更多信息请参阅 [Job.WithCode 文档](https://docs.unity3d.com/Manual/JobWithCode.html) 的捕获变量部分。

### 执行 Job.WithCode Lambda 表达式

你可以使用以下方法执行 `Job.WithCode` lambda 表达式：

#### Schedule()

以单个、非并行作业的方式执行方法。调度作业会在后台线程上运行代码，更好地利用所有可用的 CPU 资源。你可以显式传递一个 `JobHandle` 给 `Schedule()`，或者如果你不传递任何依赖项，系统假设当前系统的 `Dependency` 属性代表该作业的依赖项。或者，如果作业没有依赖项，你可以传入一个新的 `JobHandle`。

#### Run()

在主线程上执行方法。你可以 Burst 编译 `Job.WithCode`，因此如果你使用 `Run()` 执行代码，即使它在主线程上运行，这也可能更快。当你调用 `Run()` 时，Unity 会自动完成 `Job.WithCode` 构造的所有依赖项。

### 依赖关系

默认情况下，系统使用其 `Dependency` 属性来管理其依赖关系。系统将你在 `OnUpdate()` 方法中创建的每个 `Entities.ForEach` 和 `Job.WithCode` 作业按顺序添加到 `Dependency` 作业句柄中。

要手动管理作业依赖关系，请将 `JobHandle` 传递给 `Schedule` 方法，然后返回结果依赖项。更多信息请参阅 [依赖关系 API 文档](https://docs.unity3d.com/Manual/JobDependencies.html)。

了解作业依赖关系的常规信息，请参阅 [作业依赖关系文档](https://docs.unity3d.com/Manual/JobDependencies.html)。

#### 示例代码：捕获变量和调度作业

```csharp
public partial class RandomSumJob : SystemBase
{
    private uint seed = 1;

    protected override void OnUpdate()
    {
        Random randomGen = new Random(seed++);
        NativeArray<float> randomNumbers = new NativeArray<float>(500, Allocator.TempJob);

        Job.WithCode(() =>
        {
            for (int i = 0; i < randomNumbers.Length; i++)
            {
                randomNumbers[i] = randomGen.NextFloat();
            }
        }).Schedule();

        // 要从作业中获取数据，必须使用 NativeArray，即使只有一个值
        NativeArray<float> result = new NativeArray<float>(1, Allocator.TempJob);

        Job.WithCode(() =>
        {
            for (int i = 0; i < randomNumbers.Length; i++)
            {
                result[0] += randomNumbers[i];
            }
        }).Schedule();

        // 完成调度的作业以立即获取结果，但为了更高效，
        // 应该在帧早期的系统中调度作业，并在帧后期的不同系统中获取结果。
        this.CompleteDependency();
        UnityEngine.Debug.Log("The sum of " + randomNumbers.Length + " numbers is " + result[0]);

        randomNumbers.Dispose();
        result.Dispose();
    }
}
```
