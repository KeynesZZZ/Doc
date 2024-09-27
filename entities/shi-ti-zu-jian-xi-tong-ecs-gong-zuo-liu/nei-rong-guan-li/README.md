# 内容管理

Unity 的实体组件系统 (ECS) 包含其自己的内容管理和交付系统。该系统为数据导向的应用程序提供了一个高性能的接口，用于加载和释放 Unity 对象和场景。该 API 可供 ECS 系统和 MonoBehaviour 代码使用，这意味着你可以在 bakers 中使用它。

> **注意**\
> 此系统构建在 Unity 的 ContentLoadModule 程序集之上。

**主题与描述**

| 主题                                             | 描述                         |
| ---------------------------------------------- | -------------------------- |
| **Introduction to content management**         | 了解内容管理在基于实体的应用程序中的工作原理。    |
| **Weakly reference an object**                 | 获取对对象的弱引用，以便在运行时加载和使用该对象。  |
| **Weakly reference a scene**                   | 获取对场景的弱引用，以便在运行时加载和使用该场景。  |
| **Load a weakly-referenced object at runtime** | 使用系统从内容存档中加载对象。            |
| **Load a weakly-referenced scene at runtime**  | 使用系统从内容存档中加载场景及其包含的所有内容。   |
| **Create custom content archives**             | 创建自己的内容存档，以存储准备交付到应用程序的对象。 |
| **Deliver content to an application**          | 在运行时将内容存档加载到你的应用程序中。       |
