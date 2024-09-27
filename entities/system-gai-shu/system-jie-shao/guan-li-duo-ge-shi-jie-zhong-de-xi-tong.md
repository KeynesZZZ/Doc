# 管理多个世界中的系统

你可以创建多个世界，并在多个世界中实例化相同的系统类型。你还可以从更新顺序的不同点以不同的速率更新每个系统。例如，Netcode 包使用这种方式在同一进程中为客户端和服务器创建单独的世界。在用户代码中手动执行此操作是不常见的，这是一种高级用例。

#### 使用 `ICustomBootstrap` 接口管理多个世界

要实现这一点，你可以使用 `ICustomBootstrap` 接口来管理多个世界。Netcode 包中包含一个实现示例，你可以参考。

当你实现此接口时，Unity 在默认世界初始化之前调用它，并使用返回值确定是否应运行默认的世界初始化：

```csharp
public interface ICustomBootstrap
{
    // 在此方法中创建你自己的世界集或自定义默认世界。
    // 如果返回 true，则默认的世界引导程序不会运行，也不会创建其他世界。
    bool Initialize(string defaultWorldName);
}
```

### 自定义 Bootstrapper 创建多个世界

你可以使用自定义 bootstrapper 创建多个世界，从 `DefaultWorldInitialization.GetAllSystems` 获取系统的过滤列表，并使用 `DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups` 将一组系统添加到世界中。你不需要添加 `DefaultWorldInitialization.GetAllSystems` 返回的相同系统列表，可以通过添加或删除系统来修改列表。你也可以在不使用 `DefaultWorldInitialization` 的情况下创建自己的系统列表。

#### 自定义 `MyCustomBootstrap.Initialize` 实现的典型过程

1. **创建所需的世界**：创建你的游戏或应用程序需要的世界集。
2. **为每个创建的世界执行以下操作**：
   * 生成你希望在该世界中的系统列表。你可以使用 `DefaultWorldInitialization.GetAllSystems`，但这不是必需的。
   * 调用 `DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups` 将系统列表添加到世界。这也会按照 `CreateAfter/ CreateBefore` 的顺序创建系统。
3. 如果你不想手动更新世界，请调用 `ScriptBehaviourUpdateOrder.AppendWorldToCurrentPlayerLoop` 将世界添加到玩家循环中。
4. 如果你创建了默认世界，请设置 `World.DefaultGameObjectInjectionWorld` 为默认世界并返回 `true`；如果你没有创建默认世界并希望默认引导程序为你创建，则返回 `false`。

#### 示例实现

以下是一个自定义 `ICustomBootstrap` 实现的示例，用于创建和初始化多个世界：

```csharp
using Unity.Entities;
using Unity.Scenes;

public class MyCustomBootstrap : ICustomBootstrap
{
    public bool Initialize(string defaultWorldName)
    {
        // 创建默认世界
        var defaultWorld = new World(defaultWorldName);

        // 创建客户端世界
        var clientWorld = new World("ClientWorld");
        
        // 创建服务器世界
        var serverWorld = new World("ServerWorld");

        // 获取所有系统
        var allSystems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.Default);

        // 过滤系统（例如，只保留特定命名空间中的系统）
        var filteredSystems = allSystems.Where(systemType => systemType.Namespace?.Contains("YourNamespace") == true).ToList();

        // 向默认世界添加系统
        DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(defaultWorld, allSystems);

        // 向客户端世界添加系统
        DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(clientWorld, filteredSystems);

        // 向服务器世界添加系统
        DefaultWorldInitialization.AddSystemsToRootLevelSystemGroups(serverWorld, filteredSystems);

        // 将世界添加到当前玩家循环
        ScriptBehaviourUpdateOrder.AppendWorldToCurrentPlayerLoop(defaultWorld);
        ScriptBehaviourUpdateOrder.AppendWorldToCurrentPlayerLoop(clientWorld);
        ScriptBehaviourUpdateOrder.AppendWorldToCurrentPlayerLoop(serverWorld);

        // 设置默认世界
        World.DefaultGameObjectInjectionWorld = defaultWorld;

        return true;  // 防止默认世界引导程序运行
    }
}
```
