# Cleanup Component介绍

清理组件类似于常规组件，但当你销毁包含一个清理组件的实体时，Unity 会删除所有非清理组件。实体仍然存在，直到你从中移除所有清理组件。这对于标记需要在销毁时进行清理的实体非常有用。如需了解如何使用，请参阅使用清理组件。

### 清理组件生命周期

以下代码示例解释了包含清理组件的实体的生命周期：

```csharp
// 创建一个包含清理组件的实体。
Entity e = EntityManager.CreateEntity(
    typeof(Translation), typeof(Rotation), typeof(ExampleCleanup));

// 尝试销毁该实体，但由于实体具有清理组件，Unity 并没有真正销毁该实体。
// 相反，Unity 只删除了 Translation 和 Rotation 组件。
EntityManager.DestroyEntity(e);

// 实体仍然存在，这表明你仍然可以正常使用该实体。
EntityManager.AddComponent<Translation>(e);

// 从实体中移除所有剩余组件。
// 移除最后的清理组件 (ExampleCleanup) 会自动销毁该实体。
EntityManager.RemoveComponent(e, new ComponentTypeSet(typeof(ExampleCleanup), typeof(Translation)));

// 表明该实体不再存在。entityExists 为 false。
bool entityExists = EntityManager.Exists(e);
```

### 注意事项

清理组件是非托管的，并且具有与所有非托管理组件相同的限制。

以下限制也适用：

* 将实体在不同的世界间复制时，不包含清理组件。
* 因此，在烘焙（baking）时添加的清理组件不会被序列化。
* 预制件实体上的清理组件不会包含在该预制件实例化的实例上。
