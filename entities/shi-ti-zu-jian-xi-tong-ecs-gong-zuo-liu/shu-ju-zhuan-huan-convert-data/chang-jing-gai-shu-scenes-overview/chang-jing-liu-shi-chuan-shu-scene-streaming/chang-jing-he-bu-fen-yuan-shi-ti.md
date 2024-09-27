# 场景和部分元实体

##

烘焙一个作者场景会生成一个实体场景文件。每个实体场景文件的头包含：

* 部分列表，包括文件名、文件大小和边界体积等数据。
* AssetBundle 依赖项列表（GUIDs）。
* 可选的自定义元数据。

部分和包的列表决定了加载场景时 Unity 需要加载的文件列表。你可以选择使用自定义元数据用于游戏特定目的。例如，你可以将 PVS（潜在可见集）信息存储为自定义元数据，以决定何时流式加载场景，或者可以存储某些条件以决定何时加载场景。

### 加载实体场景的步骤

1. **解析阶段**：加载头并为每个场景和每个部分创建一个元实体。
2. **内容加载**：加载部分的内容。

Unity 使用这些场景和部分元实体来控制流式加载。一旦场景加载完成，你可以查询场景元实体上的 `ResolvedSectionEntity` 缓冲区以访问各个部分元实体。

### 自定义部分元数据

即使部分的内容尚未加载，部分元实体仍然可用，因此可以在该部分中存储自定义元数据。例如，可以存储玩家必须进入的边界框的维度，以便加载该部分。

要存储自定义元数据，请在烘焙期间向部分元实体添加常规 ECS 组件。这只能在烘焙系统中完成，而不能在 `Baker` 中完成。要在烘焙期间访问部分元实体，需要使用 `SerializeUtility.GetSceneSectionEntity` 方法，如下例所示：

#### 示例代码：存储自定义元数据

**定义组件**

```csharp
// 将存储元数据的组件
public struct RadiusSectionMetadata : IComponentData
{
    // 考虑加载部分的半径
    public float Radius;

    // 部分的中心
    public float3 Position;
}

[WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)]
partial struct RadiusSectionMetadataBakingSystem : ISystem
{
    private EntityQuery sectionEntityQuery;

    public void OnCreate(ref SystemState state)
    {
        // 提前创建 SerializeUtility.GetSceneSectionEntity 的 EntityQuery
        sectionEntityQuery = new EntityQueryBuilder(Allocator.Temp)
            .WithAll<SectionMetadataSetup>().Build(ref state);
    }

    public void OnUpdate(ref SystemState state)
    {
        int section = 3;
        float radius = 10f;
        float3 position = new float3(0f);

        // 访问部分元实体
        var sectionEntity = SerializeUtility.GetSceneSectionEntity(section,
            state.EntityManager, ref sectionEntityQuery, true);
        // 将 RadiusSectionMetadata 添加为元数据到部分
        state.EntityManager.AddComponentData(sectionEntity, new RadiusSectionMetadata
        {
            Radius   = radius,
            Position = position
        });
    }
}

```

在上面的示例中，变量 section、radius 和 position 是局部变量，但在实际应用场景中，这些信息将通过 Baker 从作者组件中获取。

如示例中所示，SerializeUtility.GetSceneSectionEntity 具有一个 EntityQuery 参数。如果没有提供，该方法会内部创建查询，但外部创建查询并传递给方法效率更高。

通过这种方式，你可以有效地管理和扩展各个部分的元数据，从而使场景加载和资源管理更加灵活和高效。
