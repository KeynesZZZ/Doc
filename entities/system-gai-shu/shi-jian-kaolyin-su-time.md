# 时间考虑因素（Time）

一个 World 控制其内部系统的 `Time` 属性值。系统的 `Time` 属性是当前世界时间的别名。

### 默认行为

默认情况下，Unity 为每个 World 创建一个 `TimeData` 实体，并由一个 `UpdateWorldTimeSystem` 实例更新。这反映了自上一个帧以来的经过时间。

### 固定步长仿真系统组中的时间处理

位于 `FixedStepSimulationSystemGroup` 中的系统对时间的处理不同于其他系统组。这些系统在固定间隔内更新，而不是在当前的 delta time 上更新。如果固定间隔足够小，它们可能会在每帧内更新多次。

#### 示例：固定步长仿真系统组

```csharp
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
public partial class FixedStepExampleSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float fixedDeltaTime = Time.DeltaTime;
        // 执行固定步长仿真的逻辑
    }
}
```

<pre class="language-csharp"><code class="lang-csharp"><strong>//如果需要对一个 World 内的时间进行更多控制，可以使用 World.SetTime 
</strong><strong>//来直接指定时间值。你还可以使用 PushTime 临时更改世界时间，
</strong><strong>//并使用 PopTime 返回到上一个时间（在时间堆栈中）。
</strong><strong>public partial class CustomTimeSystem : SystemBase
</strong>{
    protected override void OnUpdate()
    {
        // 创建一个新的 TimeData
        TimeData newTime = new TimeData(elapsedTime: 10.0f, deltaTime: 0.1f);
        
        // 设置世界时间
        World.SetTime(newTime);

        // 使用新的时间进行某些操作
    }
}


//压入和弹出时间
public partial class PushPopTimeSystem : SystemBase
{
    protected override void OnUpdate()
    {
        // 保存当前时间
        World.PushTime(new TimeData(elapsedTime: 20.0f, deltaTime: 0.05f));

        // 临时使用新的时间进行操作
        PerformTemporaryTimeOperation();

        // 恢复到之前的时间
        World.PopTime();
    }

    private void PerformTemporaryTimeOperation()
    {
        // 在临时时间上下文中执行的操作
    }
}


</code></pre>
