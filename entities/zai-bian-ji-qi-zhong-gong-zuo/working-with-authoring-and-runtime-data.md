# Working with Authoring and Runtime Data

#### Working with Authoring and Runtime Data

Entities Hierarchy 窗口和 Entity Inspector 提供以下模式以控制不同类型的数据：

**数据模式**

* **Authoring Mode**：包含版本控制的数据（例如，资产、场景中的 GameObjects）。在编辑器中由白色或灰色圆圈表示 ![white/gray circle](https://via.placeholder.com/15/FFFFFF/000000?text=+).
* **Runtime Mode**：包含运行时使用和修改的数据。例如，当你退出播放模式时，Unity 销毁的数据或状态。在编辑器中由橙色或红色圆圈表示 ![orange/red circle](https://via.placeholder.com/15/FFA500/000000?text=+).
* **Mixed Mode**：代表可以同时查看运行时数据和创作数据的视图，但创作数据具有优先权。在编辑器中由白色和橙色或灰色和红色圆圈表示 ![mixed circle](https://via.placeholder.com/15/808080/000000?text=O).

能够在播放模式和编辑模式之间切换数据模式非常有用，这样可以在不进入或退出播放模式的情况下对应用进行永久性更改。例如，可以在播放模式下更改一个关卡的几何形状并保存它，同时保持在播放模式。

**在场景视图显示创作数据**

可以将场景视图设置为仅显示创作数据 (`Preferences > Entities > Baking > Scene View Mode`)。这在 Unity 在运行时生成大量元素且可能导致场景视图混乱时非常有用。有关更多信息，请参阅 Entities Preferences reference。

**运行时数据模式**

也可以在编辑模式下切换到运行时数据模式，以查看 Unity 如何烘焙和优化 GameObjects，而不需要进入播放模式。

Entities Hierarchy 和 Inspector 窗口会使用以下颜色突出显示所有在退出播放模式时 Unity 将销毁的运行时数据：

* **橙色**：如果使用 Editor Dark 主题。
* **红色**：如果使用 Editor Light 主题。

这种高亮使得更容易看到哪些数据不会在模式之间持久化。

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

#### 默认行为

要更改窗口的数据模式，请选择窗口右上角的数据模式圆圈。可以选择：

* **Automatic**
* **Authoring**
* **Mixed**
* **Runtime**

在自动模式下，Unity 会根据你的选择以及你处于编辑模式还是播放模式来自动选择适当的数据模式。

| 操作            | 默认数据模式                                           |
| ------------- | ------------------------------------------------ |
| 选择实体          | Entity Inspector 设置为运行时数据模式。                     |
| 选择 GameObject | 在编辑模式下：Entity Inspector 设置为创作数据模式。               |
|               | 在播放模式的 Sub Scene 内部：Entity Inspector 设置为创作数据模式。  |
|               | 在播放模式的 Sub Scene 外部：Entity Inspector 设置为运行时数据模式。 |
| 进入播放模式        | Entities Hierarchy 和 Entity Inspector 设置为混合数据模式。 |

当选择其他模式时，Unity 会将你的窗口选择锁定到此模式。为了表明模式已锁定，在数据模式圆圈下方会出现下划线。

#### 在播放模式下创作子场景

在播放模式下，可以创作子场景。当退出播放模式时，Unity 会保留你对在运行时转换为实体的子场景 GameObjects 所做的任何更改。

**示例操作**

1. **切换到 Authoring 模式**：
   * 点击窗口右上角的数据模式圆圈，选择 `Authoring`。
   * 现在，所有的修改都会保存在编辑器中。
2. **查看运行时优化**：
   * 切换到 `Runtime` 模式，查看 Unity 如何优化和处理游戏对象，而无需进入播放模式。
3. **混合模式调试**：
   * 使用 `Mixed` 模式，可以同时查看创作和运行时数据，非常适用于调试和优化工作。

通过这些模式和功能，可以在 Unity 编辑器中灵活地管理和调试实体组件系统 (ECS) 项目，从而提升开发效率和产品质量。
