# 使用实体命令缓冲区修改动态缓冲区 (Modify dynamic buffers with an entity command buffer)

`EntityCommandBuffer`（ECB）记录添加、删除或设置实体的缓冲组件的命令。与普通组件 API 不同，它有特定的动态缓冲区 API。ECB 只能记录将来发生的命令，因此它只能以以下方式操作动态缓冲组件：

#### 操作方法

1. **SetBuffer**:
   * 返回一个 `DynamicBuffer<T>`，录制线程可以用数据填充该缓冲区。在回放时，这些缓冲区内容将覆盖任何现有的缓冲区内容。`SetBuffer` 不会在目标实体已经包含缓冲区组件时失败。
   * 如果多个线程在同一实体上记录 `SetBuffer` 命令，则回放后只有根据 `sortKey` 顺序最后一个命令添加的内容是可见的。
   * `SetBuffer` 的功能与 `AddBuffer<T>` 相同，除了 `AddBuffer` 首先添加缓冲区到组件（如果它不存在）。
2. **AppendToBuffer**:
   * 将单个缓冲区元素附加到实体上的现有缓冲区组件，并保留任何现有的缓冲区内容。多个线程可以安全地附加到同一实体上的相同缓冲区组件，录制命令的 `sortKey` 确定结果元素的顺序。
   * 如果目标实体不包含类型 T 的缓冲区组件，回放时 `AppendToBuffer<T>` 会失败。因此，最好在每个 `AppendToBuffer` 命令之前添加 `AddComponent<T>`，以确保目标缓冲区组件存在。
3. **AddComponent 和 RemoveComponent**:
   * 如果 T 是 `IBufferElementData`，则这些方法可以安全地用于添加空缓冲区或删除现有缓冲区。
   * 这些方法可以安全地从多个线程使用，并且添加现有组件或删除不存在的组件不会导致错误。

### 使用 `EntityCommandBuffer` 的动态缓冲区 API 示例 (Example of dynamic buffer-specific EntityCommandBuffer APIs)

以下代码示例展示了如何使用一些常见的动态缓冲区特定的 `EntityCommandBuffer` API。假设存在一个名为 `MyElement` 的动态缓冲区。

```csharp
private void Example(Entity e, Entity otherEntity)
{
    // 创建一个临时的 EntityCommandBuffer
    EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);

    // 记录一个命令，从实体 e 移除 MyElement 动态缓冲区。
    ecb.RemoveComponent<MyElement>(e);

    // 记录一个命令，向现有实体 e 添加一个 MyElement 动态缓冲区。
    // 如果目标实体已经包含缓冲区组件，这不会失败。
    // 返回的 DynamicBuffer 数据存储在 EntityCommandBuffer 中，
    // 因此对返回缓冲区的更改也会被录制。
    DynamicBuffer<MyElement> myBuff = ecb.AddBuffer<MyElement>(e);

    // 在回放后，实体将具有长度为 20 且包含这些记录值的 MyElement 缓冲区。
    myBuff.Length = 20;
    myBuff[0] = new MyElement { Value = 5 };
    myBuff[3] = new MyElement { Value = -9 };

    // SetBuffer 类似于 AddBuffer，但如果实体没有 MyElement 缓冲区，
    // 在回放时安全检查会抛出异常。
    DynamicBuffer<MyElement> otherBuf = ecb.SetBuffer<MyElement>(otherEntity);

    // 记录一个 MyElement 值以附加到缓冲区。如果实体没有 MyElement 缓冲区，
    // 在回放时会抛出异常。
    // ecb.AddComponent<MyElement>(otherEntity) 是确保缓冲区存在的安全方法，然后再附加。
    ecb.AppendToBuffer(otherEntity, new MyElement { Value = 12 });

    // 回放并释放 EntityCommandBuffer
    ecb.Playback(EntityManager);
    ecb.Dispose();
}

```

当您设置 DynamicBuffer 的长度、容量和内容时，ECS 会将这些更改记录到 EntityCommandBuffer 中。当您回放 EntityCommandBuffer 时，ECS 会对动态缓冲区进行相应的更改。
