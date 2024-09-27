# Subscenes overview

## 子场景概述

实体组件系统 (ECS) 使用子场景来管理应用程序的内容，这是因为 Unity 的核心场景系统与 ECS 不兼容。

### 子场景的使用

#### 添加 GameObjects 和 MonoBehaviour 组件

你可以将 GameObjects 和 MonoBehaviour 组件添加到子场景中，通过烘焙过程将这些 GameObjects 和 MonoBehaviour 组件转换为实体和 ECS 组件。有关更多信息，请参考 `Scenes overview` 文档。

#### 自定义 Bakers

你还可以选择创建自己的 bakers 以将 ECS 组件附加到转换后的实体上。有关更多信息，请参考 `Bakers overview` 文档。

### 创建子场景

#### 步骤：

1. **打开要添加子场景的场景。**
2. 在 `Hierarchy` 窗口中，右键单击并选择 `New Sub Scene > Empty Scene`。



<figure><img src="../../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

**从现有 GameObjects 创建子场景：**

1. 打开包含你想要创建子场景的 GameObjects 的场景。
2. 在 `Hierarchy` 窗口中，选择要移动到新子场景的 GameObjects。
3. 在同一窗口中，右键单击并选择 `New Sub Scene > From Selection`。

**将现有子场景添加到场景中：**

1. 打开要添加子场景的场景。
2. 创建一个空的 `GameObject`。
3. 添加 `SubScene` 组件。
4. 在 `SubScene` 组件中，设置 `Scene Asset` 属性为你想用作子场景的场景。

### 子场景组件

`SubScene` 组件是一个 Unity 组件，用于触发引用场景的烘焙和流式传输。启用 `SubScene` 组件时，如果将 `AutoLoadScene` 字段设置为 `true`，Unity 会流式传输引用的场景。

你还可以在编辑器中启用 `Auto Load Scene` 字段。方法如下：

1. 在 `Hierarchy` 窗口中选择子场景。
2. 在 `Inspector` 中，找到 `Sub Scene` 脚本并启用 `Auto Load Scene` 复选框。

#### 子场景模式

`SubScene` 组件有两种模式，取决于子场景是打开还是关闭状态。你可以通过以下方式之一打开或关闭子场景：

* 在 `Hierarchy` 窗口中，启用或禁用子场景名称旁边的复选框。
* 选择子场景，在 `Inspector` 中，在 `Open SubScenes` 下选择 `Open/Close`。

#### 子场景打开时

* 在 `Hierarchy` 窗口中，Unity 会在具有 `SubScene` 组件的 `GameObject` 下显示子场景中的所有作者 GameObjects。
* `Scene View` 根据 `Entities` 部分中的 `Scene View Mode` 设置显示运行时数据（实体）或作者数据（GameObjects）。
* 对子场景中的所有作者组件进行初步烘焙。
* 对作者组件所做的任何更改都会触发增量烘焙。

#### 子场景关闭时

* Unity 会流式传输烘焙场景的内容。当进入播放模式时，关闭子场景中的实体需要几个帧才能可用。在构建中，子场景的行为与编辑器中的关闭子场景相同，因此它们的实体不会立即可用。

### 重要提示

Unity 不会流式传输打开的子场景的内容。当进入播放模式时，打开子场景中的实体立即可用。
