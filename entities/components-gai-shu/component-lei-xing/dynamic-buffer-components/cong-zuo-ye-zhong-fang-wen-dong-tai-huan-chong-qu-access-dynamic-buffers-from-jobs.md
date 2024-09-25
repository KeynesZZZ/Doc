# 从作业中访问动态缓冲区 (Access dynamic buffers from jobs)

如果一个作业需要在其代码中查找一个或多个缓冲区，作业需要使用 `BufferLookup` 查找表。你可以在系统中创建这些查找表，然后将它们传递给需要它们的作业。

#### 修改作业 (Modify the job)

在需要随机访问动态缓冲区的作业中：

1. 添加一个只读的 `BufferLookup` 成员变量。
2. 在 `IJobEntity.Execute` 方法中，通过实体索引 `BufferLookup` 查找表。这将提供对附加到实体的动态缓冲区的访问。

#### 定义动态缓冲组件

```csharp
[InternalBufferCapacity(16)]
public struct ExampleBufferComponent : IBufferElementData
{
    public int Value;
}
```

### 修改系统 (Modify the systems)

在创建作业实例的系统中：

1. 添加一个 `BufferLookup` 成员变量。
2. 在 `OnCreate` 中，使用 `SystemState.GetBufferLookup` 来分配 `BufferLookup` 变量。
3. 在 `OnUpdate` 的开头调用 `Update` 方法更新 `BufferLookup` 变量。这会更新查找表。
4. 当你创建作业实例时，将查找表传递给作业。

以下代码示例展示了如何修改系统以支持从作业中访问动态缓冲区。

#### 定义动态缓冲组件

```csharp
[InternalBufferCapacity(16)]
public struct ExampleBufferComponent : IBufferElementData
{
    public int Value;
}
```

```csharp
public partial struct AccessDynamicBufferJob : IJobEntity
{
    [ReadOnly] public BufferLookup<ExampleBufferComponent> BufferLookup;

    public void Execute(Entity entity)
    {
        // 通过实体索引获取实体关联的动态缓冲区
        if (BufferLookup.HasBuffer(entity))
        {
            DynamicBuffer<ExampleBufferComponent> buffer = BufferLookup[entity];

            // 遍历缓冲区中的所有元素并执行操作
            for (int i = 0; i < buffer.Length; i++)
            {
                ExampleBufferComponent element = buffer[i];
                // 执行你的操作，例如打印值
                Console.WriteLine(element.Value);
            }
        }
    }
}

public partial struct AccessDynamicBufferFromJobSystem : ISystem
{
    private BufferLookup<ExampleBufferComponent> _bufferLookup;

    public void OnCreate(ref SystemState state)
    {
        // 获取 BufferLookup
        _bufferLookup = state.GetBufferLookup<ExampleBufferComponent>(true);
    }

    public void OnUpdate(ref SystemState state)
    {
        // 更新 BufferLookup
        _bufferLookup.Update(ref state);

        // 创建并调度作业，将 BufferLookup 作为参数传递给作业
        var exampleBufferAccessJob = new AccessDynamicBufferJob 
        { 
            BufferLookup = _bufferLookup 
        };
        
        exampleBufferAccessJob.ScheduleParallel();
    }
}

```

通过这种方式，可以在作业中安全且高效地访问和操作与实体关联的动态缓冲区数据，确保并行处理的安全性和效率。
