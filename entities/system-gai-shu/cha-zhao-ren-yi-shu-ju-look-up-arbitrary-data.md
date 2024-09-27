# 查找任意数据（Look up arbitrary data）

访问和更改数据的最有效方法是使用带有实体查询和 Job 的系统。这种方法利用了 CPU 资源，以最小的内存缓存未命中实现最高效的处理。最佳实践是使用最快速、最高效的路径来执行大部分数据转换。然而，有时可能需要在程序的任意点访问任意实体的任意组件。

你可以查找实体的 `IComponentData` 和其动态缓冲区中的数据。查找数据的方法取决于代码是使用 `Entities.ForEach`，还是 `IJobChunk` job，或是在主线程上以其他方法在系统中执行。

## 在系统中查找实体数据

要在系统的 `Entities.ForEach` 或 `Job.WithCode` 方法内部查找存储在任意实体组件中的数据，可以使用 `GetComponent<T>(Entity)`。

例如，以下代码使用 `GetComponent<T>(Entity)` 获取 `Target` 组件，该组件具有一个标识目标实体的字段。然后，它将跟踪实体旋转到其目标方向：

```csharp
[RequireMatchingQueriesForUpdate]
public partial class TrackingSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        Entities
            .ForEach((ref Rotation orientation,
                      in LocalToWorld transform,
                      in Target target) =>
            {
                // 确认目标实体仍然存在并且具有所需的组件
                if (!SystemAPI.HasComponent<LocalToWorld>(target.entity))
                    return;

                // 查找实体数据
                LocalToWorld targetTransform = SystemAPI.GetComponent<LocalToWorld>(target.entity);
                float3 targetPosition = targetTransform.Position;

                // 计算旋转
                float3 displacement = targetPosition - transform.Position;
                float3 upReference = new float3(0, 1, 0);
                quaternion lookRotation = quaternion.LookRotationSafe(displacement, upReference);

                orientation.Value = math.slerp(orientation.Value, lookRotation, deltaTime);
            })
            .ScheduleParallel();
    }
}
```

如果你想访问存储在动态缓冲区中的数据，需要在 `SystemBase` 的 `OnUpdate` 方法中声明一个类型为 `BufferLookup<T>` 的局部变量。然后可以在 lambda 表达式中捕获该局部变量。例如：

```csharp
public struct BufferData : IBufferElementData
{
    public float Value;
}

[RequireMatchingQueriesForUpdate]
public partial class BufferLookupSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 声明 BufferLookup 变量
        BufferLookup<BufferData> buffersOfAllEntities = this.GetBufferLookup<BufferData>(true);

        Entities
            .ForEach((ref Rotation orientation,
                      in LocalToWorld transform,
                      in Target target) =>
            {
                // 确认目标实体仍然存在并且具有此缓冲区类型
                if (!buffersOfAllEntities.HasBuffer(target.entity))
                    return;

                // 获取缓冲区引用
                DynamicBuffer<BufferData> bufferOfOneEntity = buffersOfAllEntities[target.entity];

                // 使用缓冲区中的数据
                float avg = 0;
                for (var i = 0; i < bufferOfOneEntity.Length; i++)
                {
                    avg += bufferOfOneEntity[i].Value;
                }
                if (bufferOfOneEntity.Length > 0)
                    avg /= bufferOfOneEntity.Length;

                // 执行额外的逻辑...
            })
            .ScheduleParallel();
    }
}
```

## 在 Job 中查找实体数据

若要在 `IJobChunk` 等 Job 结构中随机访问组件数据，可以使用以下类型：

* `ComponentLookup<T>`
* `BufferLookup<T>`

这些类型提供类似数组的接口来根据 `Entity` 对象索引组件。你还可以使用 `ComponentLookup` 来确定实体的可启用组件是否已启用或禁用，或切换这些组件的状态。

要使用它们，请声明一个 `ComponentLookup` 或 `BufferLookup` 类型的字段，设置该字段的值，然后调度 Job。

### 示例代码：在 Job 中查找组件数据

#### 使用 `ComponentLookup` 查找实体位置

```csharp
public struct MyComponent : IComponentData
{
    public int Value;
}

public partial class LookupInJobSystem : SystemBase
{
    private EntityQuery _query;

    protected override void OnCreate()
    {
        // 创建包含 MyComponent 和 LocalToWorld 的查询
        _query = GetEntityQuery(
            ComponentType.ReadOnly<MyComponent>(),
            ComponentType.ReadOnly<LocalToWorld>());
    }

    protected override void OnUpdate()
    {
        var job = new MyJob
        {
            DeltaTime = SystemAPI.Time.DeltaTime,
            EntityPositions = GetComponentLookup<LocalToWorld>(true)
        };

        this.Dependency = job.ScheduleParallel(_query, this.Dependency);
    }

    private struct MyJob : IJobChunk
    {
        public float DeltaTime;
        [ReadOnly]
        public ComponentLookup<LocalToWorld> EntityPositions;

        public void Execute(ArchetypeChunk chunk, int chunkIndex, int firstEntityIndex)
        {
            var entities = chunk.GetNativeArray(EntityTypeHandle);
            for (int i = 0; i < chunk.Count; i++)
            {
                Entity entity = entities[i];
                if (!EntityPositions.HasComponent(entity))
                    continue;

                LocalToWorld position = EntityPositions[entity];
                // 使用位置数据执行逻辑，例如调整组件值...
            }
        }
    }
}
```

## 注意事项

在声明 `ComponentLookup` 对象时，使用 `[ReadOnly]` 属性。除非你需要写入所访问的组件，否则应始终将 `ComponentLookup` 对象声明为只读。



以下示例演示了如何设置数据字段并调度 Job：

```csharp
protected override void OnUpdate()
{
    var job = new ChaserSystemJob();

    // 设置非 ECS 的数据字段
    job.deltaTime = SystemAPI.Time.DeltaTime;

    // 使用 Dependency 属性调度 Job
    Dependency = job.ScheduleParallel(query, this.Dependency);
}
```

在 Job 的 Execute 方法中查找组件的值

```csharp
float3 targetPosition = entityPosition.Position;
float3 chaserPosition = transform.Position;
float3 displacement = targetPosition - chaserPosition;
float3 newPosition = chaserPosition + displacement * deltaTime;
transform.Position = newPosition;

```

以下完整示例展示了一个系统，该系统移动具有 Target 字段的实体，使其位置靠近当前目标的位置：

```csharp
[RequireMatchingQueriesForUpdate]
public partial class MoveTowardsEntitySystem : SystemBase
{
    private EntityQuery query;

    [BurstCompile]
    private partial struct MoveTowardsJob : IJobEntity
    {
        // 只读数据存储在（可能是）其他 Chunk 中
        [ReadOnly]
        public ComponentLookup<LocalToWorld> EntityPositions;

        // 非实体数据
        public float deltaTime;

        public void Execute(ref LocalTransform transform, in Target target, in LocalToWorld entityPosition)
        {
            // 获取目标 Entity 对象
            Entity targetEntity = target.entity;

            // 检查目标是否仍然存在
            if (!EntityPositions.HasComponent(targetEntity))
                return;

            // 更新位移以使追踪实体朝向目标
            float3 targetPosition = EntityPositions[targetEntity].Position;
            float3 chaserPosition = transform.Position;

            float3 displacement = targetPosition - chaserPosition;
            transform.Position = chaserPosition + displacement * deltaTime;
        }
    }

    protected override void OnCreate()
    {
        // 选择所有具有 LocalTransform 和 Target 组件的实体
        query = GetEntityQuery(
            typeof(LocalTransform),
            ComponentType.ReadOnly<Target>()
        );
    }

    protected override void OnUpdate()
    {
        // 创建 Job
        var job = new MoveTowardsJob();

        // 设置组件数据查找字段
        job.EntityPositions = GetComponentLookup<LocalToWorld>(true);

        // 设置非 ECS 的数据字段
        job.deltaTime = SystemAPI.Time.DeltaTime;

        // 使用 Dependency 属性调度 Job
        Dependency = job.ScheduleParallel(query, Dependency);
    }
}

```

## 数据访问错误

如果你查找的数据与在 Job 中读取和写入的数据重叠，那么随机访问可能导致竞争条件。

在确认没有要直接读取或写入的实体数据与随机读取或写入的特定实体数据之间存在重叠时，你可以用 `NativeDisableParallelForRestriction` 属性标记访问器对象
