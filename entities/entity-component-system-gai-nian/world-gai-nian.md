# World概念

一个世界是实体的集合。实体的 ID 号在其所属的世界内是唯一的。一个世界有一个 `EntityManager` 结构体，用于创建、销毁和修改世界中的实体。

一个世界拥有一组系统，这些系统通常只访问同一个世界中的实体。此外，世界中具有相同组件类型集的一组实体存储在一个原型（archetype）中，决定了程序中组件在内存中的组织方式。

#### 初始化

默认情况下，当你进入播放模式时，Unity 会创建一个世界实例，并将每个系统添加到这个默认世界。

如果你更喜欢手动将系统添加到默认世界中，可以创建一个实现 `ICustomBootstrap` 接口的类。

如果你想完全手动控制引导过程，可以使用以下定义来禁用默认世界的创建：

```csharp
#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP_RUNTIME_WORLD: 禁用默认运行时世界的生成。
#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP_EDITOR_WORLD: 禁用默认编辑器世界的生成。
#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP: 禁用两个默认世界的生成。
```

然后你的代码负责创建世界和系统，并将世界的更新插入到 Unity 的可脚本化 PlayerLoop 中。

Unity 在编辑器中使用 WorldFlags 创建专门的世界。
