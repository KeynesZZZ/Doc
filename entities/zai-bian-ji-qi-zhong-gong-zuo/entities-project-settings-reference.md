# Entities Project Settings Reference

#### Entities Project Settings Reference

使用 Entities 项目设置可以为你的项目定义特定于实体系统的设置。要打开 Entities 项目设置，请转到 `Edit > Project Settings > Entities`。

**Setting Descriptions**

| 设置                                    | 描述                                             |
| ------------------------------------- | ---------------------------------------------- |
| **Excluded Baking System Assemblies** | 添加要从烘焙系统中排除的程序集定义资产。你可以使用此属性来防止在构建项目时运行特定的烘焙器。 |

#### Excluded Baking System Assemblies

此设置允许你指定哪些程序集在烘焙过程中应被排除。通过将某些程序集添加到排除列表中，可以确保这些程序集中的烘焙器不会在项目构建时运行。这在以下场景中特别有用：

* **优化构建时间**：通过排除不必要的烘焙系统，可以减少项目的整体构建时间。
* **避免冲突**：防止某些烘焙器干扰或破坏其他重要的烘焙过程。
* **按需运行**：仅在需要时启用特定的烘焙器，确保更精细化的控制和优化。

要使用 `Excluded Baking System Assemblies` 设置：

1. 打开 `Entities Project Settings` 窗口 (`Edit > Project Settings > Entities`)。
2. 在 `Excluded Baking System Assemblies` 列表中添加你希望排除的程序集定义资产（Assembly Definition Assets）。

以下是一个简短的示例，说明如何配置该设置：

```csharp
[assembly: AssemblyDefine("MySpecialBaker")]
namespace MyGame.Baking
{
    public class SpecialBaker : IComponentData
    {
        // Custom baking logic here
    }
}
```
