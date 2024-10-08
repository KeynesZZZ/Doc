# 内容管理介绍

对于使用 Entities 包构建的应用程序，Unity 将资产、场景和其他对象存储在内容存档中。Unity 在 Player 构建过程中自动生成这些内容存档，并在运行时根据需要加载、使用和释放其中的对象。所有这些操作都是内部进行的，但 Unity 也提供了 API，使你可以自行与内容管理系统进行接口。

这个基于内容存档的内容管理系统旨在为数据导向的基于实体的应用程序提供最佳性能，并且无需太多设置。它提供了可在 ECS 系统、多线程作业代码中使用的 Burst 编译和线程安全的 API，有时也可用于 MonoBehaviour 组件。Unity 的其他内容管理解决方案，如 Resources、AssetBundles 和 Addressables，要么无法与 Entities 一起工作，要么对于数据导向的应用程序来说不是最佳解决方案。因此，最佳实践是对于使用 Entities 的应用程序，使用这种基于内容存档的内容管理系统。

**Unity 如何生成内容存档**

在 Player 构建过程中，Unity 生成内容存档以存储构建中包含的子场景中引用的每个对象。Unity 至少为每个引用对象的子场景生成一个内容存档。如果多个子场景引用同一个对象，Unity 会将该对象移到共享的内容存档中。你可以通过以下方式引用对象：

* **强引用**：当你直接将对象分配给子场景中的 MonoBehaviour 组件属性时。
* **弱引用**：当你在烘焙过程中将对象的 `UntypedWeakReferenceId` 传递给 ECS 组件时。

Unity 将强引用和弱引用的对象捆绑到相同的内容存档中，但在运行时处理方式不同。

**强引用对象**

强引用对象是指你直接分配给子场景中 MonoBehaviour 组件属性的对象。例如，如果你将一个网格对象分配给 Mesh Filter 组件，Unity 会认为该网格对象是强引用的。

在运行时，当需要时 Unity 会自动加载强引用对象，跟踪它们的使用位置，并在不再需要时卸载它们。你不需要担心通过这种方式引用的任何对象的资产生命周期。

**弱引用对象**

弱引用对象是在烘焙过程中 Unity 检测到 `UntypedWeakReferenceId` 的对象。要弱引用一个对象，你需要从 Baker 将 `UntypedWeakReferenceId` 或一个 `WeakObjectReference`（其封装了 `UntypedWeakReferenceId`）传递给 ECS 组件。有关更多信息，请参阅弱引用对象。

在运行时，你负责弱引用对象的生命周期。你必须在需要使用它们之前使用内容管理 API 加载它们，并在不再需要时释放它们。Unity 会计算对每个弱引用对象的引用次数，当你释放最后一个实例时，Unity 会卸载该对象。

**内容管理工作流程**

在最简单的内容管理工作流程中，你需要：

1. 弱引用一个对象并将引用烘焙到组件中。有关如何执行此操作的信息，请参阅弱引用对象。
2. 使用弱引用在运行时加载、使用和释放对象。有关更多信息，请参阅在运行时加载弱引用对象和在运行时加载弱引用场景。

Unity 会自动将你从子场景中弱引用的任何对象添加到内容存档，因此你无需进行任何额外工作即可使对象在运行时可用。然而，为了支持更高级的用例，你还可以创建自定义内容存档，并从本地设备或远程内容交付服务加载它们。这可以帮助你构建额外的可下载内容，减少应用程序的初始安装大小，或者加载针对最终用户平台优化的资产。有关更多信息，请参阅创建自定义内容存档。
