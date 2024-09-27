# Entity command buffer allocator

Entity command buffer allocator 是一个自定义的可回退 allocator。每个实体命令缓冲系统在创建系统时都会创建一个实体命令缓冲 allocator。从一个实体命令缓冲 allocator 进行分配的生命周期与实体命令缓冲相同。

如果使用 `EntityCommandBufferSystem.CreateCommandBuffer()` 创建实体命令缓冲，实体命令缓冲 allocator 在记录实体命令缓冲期间分配内存，并在缓冲播放后释放内存。

你可以通过 `ECBExtensionMethods.RegisterSingleton` 注册一个实现 `IECBSingleton` 的非托管单例。

在注册期间，父实体命令缓冲系统的实体命令缓冲 allocator 会设置为单例的 allocator。因此，单例的实体命令缓冲都从这个 allocator 分配，并在缓冲播放后全部释放。

实体命令缓冲 allocator 在后台工作，你不需要进行特定的代码更改来使用它。
