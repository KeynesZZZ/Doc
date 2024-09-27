# 创建 Blob 资源

## 创建 Blob 资源

要创建 Blob 资源，请执行以下步骤：

1. **创建一个 `BlobBuilder`**：这需要在内部分配一些内存。
2. **使用 `BlobBuilder.ConstructRoot` 构造 Blob 资源的根节点**。
3. **用你的数据填充结构**。
4. **使用 `BlobBuilder.CreateBlobAssetReference` 创建一个 `BlobAssetReference`**。这会将 Blob 资源复制到其最终位置。
5. **释放 `BlobBuilder`**。

以下示例将具有基本成员的结构存储为 Blob 资源：

```csharp
using Unity.Entities;
using Unity.Collections;

// 定义 MarketData 结构体
struct MarketData
{
    public float PriceOranges;
    public float PriceApples;
}

// 创建 MarketData 的 Blob 资源
BlobAssetReference<MarketData> CreateMarketData()
{
    // 创建一个新的 BlobBuilder，将使用临时内存来构造 Blob 资源
    var builder = new BlobBuilder(Allocator.Temp);

    // 构造 Blob 资源的根对象。注意使用 `ref` 关键字。
    ref MarketData marketData = ref builder.ConstructRoot<MarketData>();

    // 用数据填充构造的根对象：
    // 苹果与橘子的普遍接受比例为 2 : 1 。
    marketData.PriceApples = 2f;
    marketData.PriceOranges = 4f;

    // 将数据从构建器复制到它的最终位置，这将使用持久性分配器
    var result = builder.CreateBlobAssetReference<MarketData>(Allocator.Persistent);

    // 确保释放构建器本身，以便所有内部内存被释放。
    builder.Dispose();
    return result;
}
```

`BlobBuilder` 构造存储在 `Blob 资源` 中的数据，确保所有内部引用都存储为偏移量，然后将完成的 `Blob 资源` 复制到一个单一分配中，由返回的 `BlobAssetReference<T>` 引用。

## Blob 资源中的数组

你必须使用 `BlobArray` 类型在 Blob 资源中创建数组。这是因为数组在内部通过相对偏移实现。以下是如何分配 Blob 数据数组并填充它的示例：

#### 定义结构体

```csharp
using Unity.Entities;
using Unity.Collections;

struct Hobby
{
    public float Excitement;
    public int NumOrangesRequired;
}

struct HobbyPool
{
    public BlobArray<Hobby> Hobbies;
}

BlobAssetReference<HobbyPool> CreateHobbyPool()
{
    var builder = new BlobBuilder(Allocator.Temp);
    ref HobbyPool hobbyPool = ref builder.ConstructRoot<HobbyPool>();

    // 为池中的两个爱好分配足够的空间。使用返回的 BlobBuilderArray 填充数据。
    const int numHobbies = 2;
    BlobBuilderArray<Hobby> arrayBuilder = builder.Allocate(
        ref hobbyPool.Hobbies,
        numHobbies
    );

    // 初始化爱好数据。

    // 一个令人兴奋的爱好，需要大量橘子。
    arrayBuilder[0] = new Hobby
    {
        Excitement = 1.0f,
        NumOrangesRequired = 7
    };

    // 一个不那么令人兴奋的爱好，节约使用橘子。
    arrayBuilder[1] = new Hobby
    {
        Excitement = 0.2f,
        NumOrangesRequired = 2
    };

    var result = builder.CreateBlobAssetReference<HobbyPool>(Allocator.Persistent);
    builder.Dispose();
    return result;
}

```

## Blob 资源中的字符串

你必须使用 `BlobString` 类型在 Blob 资源中创建字符串。以下是如何使用 `BlobBuilder` API 分配字符串的示例：

#### 定义结构体

```csharp
using Unity.Entities;
using Unity.Collections;
using Unity.Collections.LowLevel.Unsafe;

struct CharacterSetup
{
    public float Loveliness;
    public BlobString Name;
}

BlobAssetReference<CharacterSetup> CreateCharacterSetup(string name)
{
    var builder = new BlobBuilder(Allocator.Temp);
    ref CharacterSetup character = ref builder.ConstructRoot<CharacterSetup>();

    character.Loveliness = 9001; // 这是一个非常可爱的角色

    // 创建一个新的 BlobString 并将其设置为给定的名称。
    builder.AllocateString(ref character.Name, name);

    var result = builder.CreateBlobAssetReference<CharacterSetup>(Allocator.Persistent);
    builder.Dispose();
    return result;
}

```

## 内部指针

要手动设置内部指针，请使用 `BlobPtr<T>` 类型。

#### 定义结构体

```csharp
using Unity.Entities;
using Unity.Collections;

struct FriendList
{
    public BlobPtr<BlobString> BestFriend;
    public BlobArray<BlobString> Friends;
}

BlobAssetReference<FriendList> CreateFriendList()
{
    var builder = new BlobBuilder(Allocator.Temp);
    ref FriendList friendList = ref builder.ConstructRoot<FriendList>();

    const int numFriends = 3;
    var arrayBuilder = builder.Allocate(ref friendList.Friends, numFriends);
    builder.AllocateString(ref arrayBuilder[0], "Alice");
    builder.AllocateString(ref arrayBuilder[1], "Bob");
    builder.AllocateString(ref arrayBuilder[2], "Joachim");

    // 设置最佳朋友指针，指向数组中的第二个元素。
    builder.SetPointer(ref friendList.BestFriend, ref arrayBuilder[2]);

    var result = builder.CreateBlobAssetReference<FriendList>(Allocator.Persistent);
    builder.Dispose();
    return result;
}

```

## 在组件上访问 Blob 资源

一旦你创建了 `BlobAssetReference<T>`，你可以将该引用存储在组件上并进行访问。必须通过引用访问包含内部指针的所有 Blob 资源部分。

#### 定义结构体和组件

```csharp
using Unity.Entities;
using Unity.Collections;

struct Hobby
{
    public float Excitement;
    public int NumOrangesRequired;
}

struct HobbyPool
{
    public BlobArray<Hobby> Hobbies;
}

struct Hobbies : IComponentData
{
    public BlobAssetReference<HobbyPool> Blob;
}
float GetExcitingHobby(ref Hobbies component, int numOranges)
{
    // 获取可用爱好池的引用。注意它需要通过引用传递，
    // 否则 BlobArray 中的内部引用将无效。
    ref HobbyPool pool = ref component.Blob.Value;

    // 找到我们可以参与且橘子数量足够的最令人兴奋的爱好。
    float mostExcitingHobby = 0;
    for (int i = 0; i < pool.Hobbies.Length; i++)
    {
        // 这种操作是安全的，因为 Hobby 结构体不包含内部引用。
        var hobby = pool.Hobbies[i];
        if (hobby.NumOrangesRequired > numOranges)
            continue;
        if (hobby.Excitement >= mostExcitingHobby)
            mostExcitingHobby = hobby.Excitement;
    }

    return mostExcitingHobby;
}

```

## 释放 Blob 资源引用

任何在运行时通过 `BlobBuilder.CreateBlobAssetReference` 分配的 Blob 资源都需要手动释放。

然而，作为从磁盘加载的实体场景的一部分加载的任何 Blob 资源则不需要手动释放。这些 Blob 资源会被引用计数管理，一旦没有组件再引用它们，它们将自动释放。

#### 示例代码：在系统中分配和释放 Blob 资源

```csharp
using Unity.Entities;
using Unity.Collections;

struct MarketData
{
    public float PriceOranges;
    public float PriceApples;
}

public partial struct BlobAssetInRuntimeSystem : ISystem
{
    private BlobAssetReference<MarketData> _blobAssetReference;

    public void OnCreate(ref SystemState state)
    {
        using (var builder = new BlobBuilder(Allocator.Temp))
        {
            ref MarketData marketData = ref builder.ConstructRoot<MarketData>();
            marketData.PriceApples = 2f;
            marketData.PriceOranges = 4f;
            _blobAssetReference =
                builder.CreateBlobAssetReference<MarketData>(Allocator.Persistent);
        }
    }

    public void OnDestroy(ref SystemState state)
    {
        // 调用 Dispose 方法将销毁引用的 BlobAsset 并释放其内存
        _blobAssetReference.Dispose();
    }
}
```

## 调试 Blob 资源内容

Blob 资源使用相对偏移来实现内部引用。这意味着复制 `BlobString` 结构或其他包含这些内部引用的类型时，只会复制包含的相对偏移，而不是它指向的内容。结果是一个不可用的 `BlobString`，表示一个随机字符字符串。尽管这在你自己的代码中容易避免，但调试工具通常会执行此操作。因此，`BlobString` 的内容在调试器中不会正确显示。

然而，支持显示 `BlobAssetReference<T>` 及其所有内容的值。如果你想查看 `BlobString` 的内容，请导航到包含该 `BlobString` 的 `BlobAssetReference<T>` 并从那里开始调试。

### 示例代码：调试 Blob 资源

```csharp
using Unity.Entities;
using Unity.Collections;
using UnityEngine;

struct CharacterSetup
{
    public float Loveliness;
    public BlobString Name;
}

public class DebugBlobAsset : MonoBehaviour
{
    private BlobAssetReference<CharacterSetup> _blobAssetReference;

    void Start()
    {
        using (var builder = new BlobBuilder(Allocator.Temp))
        {
            ref CharacterSetup character = ref builder.ConstructRoot<CharacterSetup>();
            character.Loveliness = 9001;
            builder.AllocateString(ref character.Name, "SampleName");

            _blobAssetReference = builder.CreateBlobAssetReference<CharacterSetup>(Allocator.Persistent);
        }

        // For debugging purposes: Use the BlobAssetReference to access and inspect the contents.
        ref CharacterSetup characterSetup = ref _blobAssetReference.Value;
        Debug.Log($"Loveliness: {characterSetup.Loveliness}, Name: {characterSetup.Name.ToString()}");
    }

    void OnDestroy()
    {
        _blobAssetReference.Dispose();
    }
}
```

## 在烘焙中使用 Blob 资源

可以使用 bakers 和烘焙系统来离线创建 Blob 资源，并在运行时使用它们。

### 处理 Blob 资源：`BlobAssetStore`

`BlobAssetStore` 用于管理 Blob 资源。它保持内部引用计数，并确保如果没有引用它们，Blob 资源将被释放。Bakers 内部访问 `BlobAssetStore`，但要在烘焙系统中创建 Blob 资源，需要从烘焙系统中检索 `BlobAssetStore`。

#### 示例代码：在烘焙系统中创建和管理 Blob 资源

```csharp
using Unity.Entities;
using Unity.Collections;

struct MarketData
{
    public float PriceOranges;
    public float PriceApples;
}

class MarketDataAuthoring : MonoBehaviour
{
    public float PriceOranges = 4f;
    public float PriceApples = 2f;
}

class MarketDataBaker : Baker<MarketDataAuthoring>
{
    public override void Bake(MarketDataAuthoring authoring)
    {
        var blobAssetStore = World.DefaultGameObjectInjectionWorld.GetOrCreateSystemManaged<BakingSystemGroup>().BlobAssetStore;
        
        using (var builder = new BlobBuilder(Allocator.Temp))
        {
            ref MarketData marketData = ref builder.ConstructRoot<MarketData>();
            marketData.PriceApples = authoring.PriceApples;
            marketData.PriceOranges = authoring.PriceOranges;

            var blobAssetReference = builder.CreateBlobAssetReference<MarketData>(Allocator.Persistent);
            // 将 Blob 资产存储在 BlobAssetStore 中。
            blobAssetStore.AddUniqueBlobAsset(ref blobAssetReference);

            // 添加到实体上
            AddBlobAsset(ref blobAssetReference);
        }
    }
}

[UpdateInGroup(typeof(BakingSystemGroup))]
public partial class MarketDataBakingSystem : SystemBase
{
    private BlobAssetStore _blobAssetStore;

    protected override void OnCreate()
    {
        base.OnCreate();
        _blobAssetStore = new BlobAssetStore();
    }

    protected override void OnDestroy()
    {
        // 确保释放 BlobAssetStore 以清理未引用的 Blob 资源
        _blobAssetStore.Dispose();
        base.OnDestroy();
    }

    protected override void OnUpdate()
    {
        // 实际上在 BakingSystem 中不需要进行操作，因为 BlobAssetStore 管理引用计数和清理
    }
}
```



## 在 Baker 中注册 Blob 资源

由于 bakers 是确定性和增量式的，因此在烘焙中使用 Blob 资源需要遵循一些额外的步骤。在使用 `BlobBuilder` 创建 `BlobAssetReference` 的同时，还需要将 Blob 资源注册到 baker。

### 示例代码：在 Baker 中注册 Blob 资源

#### 定义结构体和组件

```csharp
using Unity.Entities;

struct MarketData
{
    public float PriceOranges;
    public float PriceApples;
}

struct MarketDataComponent : IComponentData
{
    public BlobAssetReference<MarketData> Blob;
}

using UnityEngine;

public class MarketDataAuthoring : MonoBehaviour
{
    public float PriceOranges;
    public float PriceApples;
}

using Unity.Entities;
using Unity.Collections;

class MarketDataBaker : Baker<MarketDataAuthoring>
{
    public override void Bake(MarketDataAuthoring authoring)
    {
        // 创建一个将使用临时内存构建 Blob 资源的新 builder
        var builder = new BlobBuilder(Allocator.Temp);

        // 构造 Blob 资源的根对象。注意使用 `ref`。
        ref MarketData marketData = ref builder.ConstructRoot<MarketData>();

        // 填充构造的根数据：
        marketData.PriceApples = authoring.PriceApples;
        marketData.PriceOranges = authoring.PriceOranges;

        // 将数据从 builder 复制到其最终位置，将使用持久化分配器
        var blobReference = builder.CreateBlobAssetReference<MarketData>(Allocator.Persistent);

        // 确保释放 builder 本身以释放所有内部内存
        builder.Dispose();

        // 将 Blob 资源注册到 Baker 以进行去重和回滚
        AddBlobAsset<MarketData>(ref blobReference, out var hash);
        
        // 获取实体并添加组件
        var entity = GetEntity(TransformUsageFlags.None);
        AddComponent(entity, new MarketDataComponent() { Blob = blobReference });
    }
}

```

重要 如果不将 Blob 资源注册到 baker，引用计数不会更新，并且 Blob 资源可能会意外被释放。

baker 使用 BlobAssetStore 来去重和引用计数 Blob 资源。它还会减少关联 Blob 资源的引用计数，以在 baker 重新运行时回滚 Blob 资源。如果没有这一步，bakers 的增量行为将中断。因此，BlobAssetStore 不能直接从 baker 获取，只能通过 baker 方法访问。



### 使用自定义哈希进行去重&#x20;

前面的示例让 baker 处理所有的去重，但这意味着你必须先创建 Blob 资源，然后 baker 才能去重并释放多余的 Blob 资源。在某些情况下，你可能希望在 baker 中创建 Blob 资源之前进行去重。

要做到这一点，可以使用自定义哈希，而不是让 baker 生成哈希。如果多个 bakers 要么可以访问，要么为相同的 Blob 资源生成相同的哈希，则可以使用此哈希在生成 Blob 资源之前进行去重。使用 `TryGetBlobAssetReference` 检查自定义哈希是否已注册到 baker：

```csharp
class MarketDataCustomHashBaker : Baker<MarketDataAuthoring>
{
    public override void Bake(MarketDataAuthoring authoring)
    {
        var customHash = new Unity.Entities.Hash128(
            (uint)authoring.PriceOranges.GetHashCode(),
            (uint)authoring.PriceApples.GetHashCode(), 
            0, 
            0);

        if (!TryGetBlobAssetReference(customHash, out BlobAssetReference<MarketData> blobReference))
        {
            // 创建一个将使用临时内存构建 Blob 资源的新 builder
            var builder = new BlobBuilder(Allocator.Temp);

            // 构造 Blob 资源的根对象。注意使用 `ref`。
            ref MarketData marketData = ref builder.ConstructRoot<MarketData>();

            // 填充构造的根数据：
            marketData.PriceApples = authoring.PriceApples;
            marketData.PriceOranges = authoring.PriceOranges;

            // 将数据从 builder 复制到其最终位置，将使用持久化分配器
            blobReference =
                builder.CreateBlobAssetReference<MarketData>(Allocator.Persistent);

            // 确保释放 builder 本身以释放所有内部内存
            builder.Dispose();

            // 将 Blob 资源与自定义哈希一起注册到 Baker，以进行去重和回滚
            AddBlobAssetWithCustomHash<MarketData>(ref blobReference, customHash);
        }

        var entity = GetEntity(TransformUsageFlags.None);
        AddComponent(entity, new MarketDataComponent() { Blob = blobReference });
    }
}
```
