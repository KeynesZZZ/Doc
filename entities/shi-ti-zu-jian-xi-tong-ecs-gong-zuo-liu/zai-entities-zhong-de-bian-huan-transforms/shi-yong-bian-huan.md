# 使用变换

## 使用transforms

要在你的项目中使用变换，可以使用 `Unity.Transforms` 命名空间来控制任何实体的位置、旋转和缩放。

### LocalTransform 组件

`LocalTransform` 表示实体的相对位置、旋转和缩放。如果有父级，变换是相对于该父级的；如果没有父级，变换是相对于世界原点的。你可以读写这个组件。

#### 组件定义

```csharp
public struct LocalTransform : IComponentData
{
    public float3 Position;
    public float Scale;
    public quaternion Rotation;
}
```

## 使用 API

在 API 中没有修改 `LocalTransform` 的方法。所有方法返回一个新值，并且不会改变变换本身。所以如果你想修改变换，必须使用赋值操作符。例如，要围绕 Z 轴旋转一个变换：

```csharp
myTransform = myTransform.RotateZ(someAngle);
```

直接修改 LocalTransform 的唯一方法是写入 Position、Rotation 和 Scale 属性。例如：

```csharp
myTransform.Position += math.up(); 
```

这段代码等价于：

```csharp
 myTransform = myTransform.Translate(math.up());
```

有多种方法可以为你构造一个变换。所以如果你想创建一个具有指定位置但使用默认旋转和缩放的 LocalTransform，可以使用以下代码：

```csharp
 var myTransform = LocalTransform.FromPosition(1, 2, 3);
```

## 使用层次结构

你可以单独使用 `LocalTransform`。然而，如果你想使用实体的层次结构，还必须使用 `Parent` 组件。要设置子实体的父级，可以使用 `Parent` 组件：

### Parent 组件定义

```csharp
public struct Parent : IComponentData
{
    public Entity Value;
}
```

为了确保父级找到其子级，并设置它们的 Child 组件，必须运行 ParentSystem。

对不会移动的所有实体使用静态标志。这将提高性能并减少内存消耗。

变换系统针对根级别的大量层次结构进行了优化。根级别变换是没有父级的变换。避免在单个根下拥有大量层次结构。连接层次变换的工作在根级别被划分到多个作业中。所以最糟糕的情况是将大量非静态实体保持在一个根下。
