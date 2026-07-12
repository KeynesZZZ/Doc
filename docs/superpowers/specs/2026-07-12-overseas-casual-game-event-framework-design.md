# 海外休闲游戏活动开发框架设计

> **创建日期**: 2026-07-12
> **状态**: 设计完成，待审核
> **作者**: llm + human
> **技术栈**: Unity 2022.3 LTS / HybridCLR / YooAssets / UniTask / DOTween / UGUI / Firebase

---

## 1. 文档定位

本文档设计一套面向**海外休闲游戏**的**活动开发框架**，支持完整的活动生态——从日常签到、通行证赛季，到万圣节/圣诞节限时自定义玩法。

框架采用**微内核 + 活动插件架构**，每个活动作为独立的 HybridCLR 热更模块（dll + Prefab + 配置），宿主提供调度、配置、奖励、经济监控等基础设施。

### 1.1 设计目标

| 目标 | 衡量标准 |
|------|---------|
| 新活动开发效率 | 一个常规活动 3 人天完成（不含美术资源） |
| 运营灵活度 | 活动开关、时间调整、AB 分组无需发版 |
| 安全性 | 经济系统不可被客户端篡改，所有奖励经服务端校验 |
| 容错性 | 单个活动崩溃不影响核心游戏和其他活动 |
| 热更能力 | 新活动上线、bug 修复、配置调整均无需应用商店发版 |

### 1.2 设计约束

- 热更方式：HybridCLR 程序集热更（与现有休闲游戏框架一致）
- 配置下发：自研后台（结构化数据）+ Firebase Remote Config（实时开关/AB 分组）
- UI 构建：每活动独立 UGUI Prefab + 热更代码模块
- 奖励系统：统一奖励管线 + 经济监控（客户端预检 + 服务端权威）
- 同时前台活动数：1 个（休闲游戏屏幕空间有限）

---

## 2. 整体架构分层

```
┌──────────────────────────────────────────────────────────┐
│                    Bootstrap Layer                        │
│  (游戏启动时初始化 ActivityHost，注册内置服务)              │
├──────────────────────────────────────────────────────────┤
│                    Host Core (宿主核心)                    │
│                                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │ Scheduler  │ │ Lifecycle  │ │ Config Manager     │   │
│  │ 活动调度器  │ │ 生命周期    │ │ 配置管理器          │   │
│  │            │ │ Manager    │ │ (HTTP+Firebase)    │   │
│  └────────────┘ └────────────┘ └────────────────────┘   │
│                                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │ Reward     │ │ Economy    │ │ Analytics          │   │
│  │ System     │ │ Monitor    │ │ Integration        │   │
│  │ 奖励管线    │ │ 经济监控    │ │ 埋点上报            │   │
│  └────────────┘ └────────────┘ └────────────────────┘   │
│                                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │
│  │ Module     │ │ Resource   │ │ State               │   │
│  │ Loader     │ │ Provider   │ │ Persistence         │   │
│  │ 模块加载器  │ │ 资源桥接    │ │ 状态持久化           │   │
│  └────────────┘ └────────────┘ └────────────────────┘   │
├──────────────────────────────────────────────────────────┤
│              Activity Plugin API (SPI)                    │
│                                                          │
│  IActivityModule    IActivityContext    IHostService     │
│  (活动入口接口)      (宿主上下文)        (宿主服务注册)     │
├──────────────────────────────────────────────────────────┤
│                Activity Modules (插件层)                   │
│                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ SignIn   │ │ Battle   │ │ Xmas2026 │ │ Halloween│   │
│  │ Module   │ │ Pass     │ │ Module   │ │ Module   │   │
│  │ .dll     │ │ .dll     │ │ .dll     │ │ .dll     │   │
│  │ .prefab  │ │ .prefab  │ │ .prefab  │ │ .prefab  │   │
│  │ config   │ │ config   │ │ config   │ │ config   │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
├──────────────────────────────────────────────────────────┤
│              Base Game (基础游戏层)                        │
│  YooAssets / HybridCLR / UniTask / DOTween / UGUI        │
└──────────────────────────────────────────────────────────┘
```

### 各层职责

| 层 | 职责 | 关键设计原则 |
|---|---|---|
| **Bootstrap** | 初始化宿主、注册服务、拉取活动清单 | 简洁的启动序列，失败不阻断游戏 |
| **Host Core** | 9 个核心子系统，提供活动运行的全部基础设施 | 子系统之间通过接口通信，不直接互调 |
| **SPI** | 活动模块与宿主的契约层，3 个核心接口 | 接口稳定后不轻易变动，保证插件兼容性 |
| **Activity Modules** | 每个活动一个独立模块（dll + prefab + config） | 模块之间完全隔离，通过宿主间接交互 |
| **Base Game** | 现有游戏框架和第三方库 | 活动框架是上层消费者，不修改基础层 |

---

## 3. 核心接口契约（SPI）

### 3.1 IActivityModule — 活动入口接口

```csharp
/// <summary>
/// 活动模块入口接口 — 每个活动插件的主类必须实现
/// </summary>
public interface IActivityModule
{
    /// <summary>活动唯一标识（如 "halloween_2026"）</summary>
    string ActivityId { get; }

    /// <summary>活动版本号，用于热更兼容性检查</summary>
    string Version { get; }

    /// <summary>依赖的宿主API最低版本</summary>
    string MinHostVersion { get; }

    /// <summary>
    /// 初始化 — 宿主在加载模块后调用
    /// 传入上下文，模块借此获取宿主服务
    /// </summary>
    UniTask OnInitAsync(IActivityContext context);

    /// <summary>
    /// 活动进入前台 — 用户打开了活动界面
    /// 模块负责加载UI Prefab、开始播放音乐等
    /// </summary>
    UniTask OnEnterAsync();

    /// <summary>
    /// 活动退到后台 — 用户关闭或切出活动界面
    /// 模块负责暂停动画、保存临时状态
    /// </summary>
    UniTask OnExitAsync();

    /// <summary>
    /// 活动被宿主卸载 — 时间到期或运营强制下线
    /// 模块负责释放所有资源、保存进度
    /// </summary>
    UniTask OnDisposeAsync();

    /// <summary>
    /// 宿主每帧调用，模块在此驱动活动逻辑
    /// 未进入前台的模块不会被调用
    /// </summary>
    void OnTick(float deltaTime);

    /// <summary>
    /// 配置热更新通知 — 运营修改配置后由宿主调用
    /// 模块可读取 Context.Config 获取最新配置
    /// </summary>
    void OnConfigUpdated(ResolvedConfig newConfig);
}
```

### 3.2 IActivityContext — 宿主上下文

```csharp
/// <summary>
/// 宿主上下文 — 活动模块与宿主通信的唯一桥梁
/// </summary>
public interface IActivityContext
{
    /// <summary>本活动的已合并配置（双层合并后的运行时配置）</summary>
    ResolvedConfig Config { get; }

    /// <summary>本活动的持久化状态（玩家进度、领取记录等）</summary>
    IActivityState State { get; }

    /// <summary>获取宿主注册的服务（奖励系统、资源系统等）</summary>
    T GetService<T>() where T : IHostService;

    /// <summary>加载本活动的UI Prefab</summary>
    UniTask<GameObject> LoadUIAsync(string prefabName);

    /// <summary>上报埋点事件</summary>
    void TrackEvent(string eventName, Dictionary<string, object> parameters);

    /// <summary>请求打开其他活动（宿主决定是否允许）</summary>
    void RequestNavigate(string targetActivityId);
}
```

### 3.3 IHostService — 宿主服务接口族

```csharp
/// <summary>标记接口 — 所有宿主服务接口必须继承</summary>
public interface IHostService { }

/// <summary>奖励服务 — 活动产出发奖请求</summary>
public interface IRewardService : IHostService
{
    UniTask<RewardResult> GrantRewardsAsync(RewardData[] rewards);
    RewardPreview PreviewRewards(RewardData[] rewards);
}

/// <summary>资源服务 — 活动请求基础游戏资源</summary>
public interface IResourceService : IHostService
{
    UniTask<T> LoadAssetAsync<T>(string address) where T : UnityEngine.Object;
    void ReleaseAsset(UnityEngine.Object asset);
}

/// <summary>玩家信息服务 — 查询玩家基础数据</summary>
public interface IPlayerInfoService : IHostService
{
    long GetCurrency(CurrencyType type);
    int GetPlayerLevel();
    string GetPlayerId();
}

/// <summary>网络服务 — 活动请求服务端接口</summary>
public interface INetworkService : IHostService
{
    UniTask<ResponseData> PostAsync(string path, string jsonBody);
    bool IsOnline { get; }
}
```

### 3.4 SPI 版本兼容策略

采用语义化版本（主版本.次版本.修订号）：

- **主版本不同** → 不兼容，拒绝加载
- **次版本不同** → 兼容（宿主次版本 >= 模块要求即可）
- **修订号** → 不影响兼容性

```csharp
public class ModuleCompatChecker
{
    public bool IsCompatible(IActivityModule module)
    {
        return HostVersionUtil.Satisfies(
            hostVersion: _hostVersion,
            requiredVersion: module.MinHostVersion
        );
    }
}
```

### 3.5 接口调用关系

```
    ┌──────────────────────────────────────────────┐
    │              Activity Module                  │
    │            (Halloween 2026)                   │
    │   只知道接口，不知道具体实现                    │
    └──────┬───────────────────┬────────────────────┘
           │                   │
     OnInitAsync()        GetService<T>()
           │                   │
           ▼                   ▼
    ┌──────────────┐   ┌──────────────────────────┐
    │IActivityCtx  │   │  IHostService 子接口       │
    │ .Config      │   │  IRewardService           │
    │ .State       │   │  IResourceService         │
    │ .LoadUI()    │   │  IPlayerInfoService       │
    │ .TrackEvent()│   │  INetworkService          │
    └──────┬───────┘   └───────────────────────────┘
           │                      │
           ▼                      ▼
    ┌──────────────────────────────────────────────┐
    │              Host Core (宿主)                 │
    │  实现所有接口，管理具体逻辑                     │
    └──────────────────────────────────────────────┘
```

---

## 4. 活动生命周期管理

### 4.1 状态机

```
                    ┌───────────┐
     宿主启动 ──────▶│  Pending  │
    (拉取活动清单)    │  (待激活)  │
                    └─────┬─────┘
                          │ 到达活动开始时间
                          ▼
                    ┌───────────┐
                    │  Loading  │     加载 HybridCLR 程序集
                    │  (加载中)  │     加载 UI Prefab / 配置
                    └─────┬─────┘     调用 OnInitAsync()
                          │ 初始化完成
                          ▼
                    ┌───────────┐
     用户打开活动 ──▶│  Running  │◀── 用户重新进入
                    │  (运行中)  │
                    └─────┬─────┘
                          │ 用户关闭 / 切到其他活动
                          ▼
                    ┌───────────┐
                    │ Suspended │     UI 卸载，调用 OnExitAsync()
                    │ (挂起中)   │     逻辑暂停，状态保留在内存
                    └─────┬─────┘
                          │ 到达结束时间 / 运营强制下线
                          ▼
                    ┌───────────┐
                    │ Disposing │     调用 OnDisposeAsync()
                    │ (卸载中)   │     保存进度、释放资源
                    └─────┬─────┘     卸载 HybridCLR 程序集
                          ▼
                    ┌───────────┐
                    │  Closed   │     从活动列表移除
                    │  (已关闭)  │     清理内存
                    └───────────┘
```

**特殊转换路径**：

| 场景 | 转换 | 处理 |
|------|------|------|
| 加载失败 | Loading → Closed | 记录错误埋点，通知玩家"活动暂不可用" |
| 运行中崩溃 | Running → Suspended | 捕获异常，自动挂起并上报，不阻塞其他活动 |
| 运营紧急下线 | 任意状态 → Disposing | 强制走卸载流程，玩家进度存邮件补偿 |
| 活动时间延长 | Closed → Pending | 运营修改结束时间后重新进入待激活 |

### 4.2 调度器

```csharp
public class ActivityScheduler
{
    private readonly Dictionary<string, ActivityManifest> _manifests;
    private readonly Dictionary<string, LoadedModule> _loaded;

    /// <summary>宿主每帧调用，检查时间窗口并触发状态转换</summary>
    public void Tick(float deltaTime)
    {
        var now = DateTimeOffset.UtcNow;

        // Pending → Loading（到达开始时间）
        foreach (var (id, manifest) in _manifests)
        {
            if (manifest.State == ActivityState.Pending
                && now >= manifest.StartTime
                && now < manifest.EndTime)
            {
                _ = LoadAndInitAsync(id);
            }
        }

        // Running/Suspended → Disposing（到达结束时间）
        foreach (var (id, module) in _loaded)
        {
            if (now >= module.Manifest.EndTime)
            {
                _ = DisposeAsync(id);
            }
        }
    }

    /// <summary>运营推送的活动开关（来自 Firebase Remote Config）</summary>
    public void ApplyRemoteSwitch(ActivitySwitch switchData) { }
}
```

### 4.3 活动清单结构

```json
{
  "manifestVersion": "2026.07.12.01",
  "fetchTime": "2026-07-12T08:00:00Z",
  "activities": [
    {
      "activityId": "halloween_2026",
      "displayName": "Halloween Spooky Festival",
      "assemblyName": "Act.Halloween2026.dll",
      "entryClass": "Halloween2026.HalloweenModule",
      "resourceCatalog": "act_halloween_2026",
      "configUrl": "https://cdn.example.com/configs/halloween_2026.json",
      "startTime": "2026-10-25T00:00:00Z",
      "endTime": "2026-11-05T00:00:00Z",
      "minHostVersion": "1.0.0",
      "moduleVersion": "1.2.0",
      "abTestKey": "halloween_2026_ui_variant",
      "priority": 90,
      "requiredLevel": 10
    }
  ]
}
```

### 4.4 前台/后台切换

同一时刻只有 1 个活动处于 Running（前台），其余活动保持 Suspended（挂起）。用户切换活动时：

1. 挂起当前前台活动：调用 `OnExitAsync()`，卸载 UI Prefab，保留逻辑状态
2. 恢复或首次进入目标活动：重新加载 UI Prefab，调用 `OnEnterAsync()`
3. 埋点上报 `activity_enter`

---

## 5. 配置系统设计

### 5.1 双层配置架构

```
┌───────────────────┐     ┌──────────────────────┐
│  自研活动管理后台   │     │  Firebase Console    │
│                   │     │                      │
│ · 结构化配置编辑   │     │ · 活动总开关 (KV)     │
│ · 奖励表/任务表   │     │ · AB分组比例          │
│ · 时间窗口        │     │ · 紧急下线开关        │
│ · 版本管理/回滚   │     │ · 动态参数微调        │
│                   │     │                      │
│ 产出：ActivityConfig│    │ 产出：RemoteSwitch   │
│ (完整结构化JSON)  │     │ (KV开关+参数)        │
└────────┬──────────┘     └──────────┬───────────┘
         │                           │
    HTTP 拉取                 Firebase SDK 推送
         ▼                           ▼
┌─────────────────────────────────────────────────┐
│         Config Manager (宿主)                    │
│  · 合并两层配置，生成最终运行时配置 (ResolvedConfig)│
│  · RemoteSwitch 可覆盖 ActivityConfig            │
│  · 缓存到本地，离线时使用上次缓存                  │
│  · HMAC-SHA256 签名验证完整性                     │
└─────────────────────────────────────────────────┘
```

### 5.2 结构化配置（ActivityConfig）

```csharp
[System.Serializable]
public class ActivityConfig
{
    public string ActivityId;
    public string DisplayName;
    public string DisplayIcon;

    public long StartTimeUnix;
    public long EndTimeUnix;

    public MissionDef[] Missions;
    public RewardDef[] RewardPool;
    public PriceDef[] Prices;

    public string I18nPrefix;
    public string CustomDataJson;       // 活动专属自由格式数据

    // 配置签名（服务端生成，客户端校验）
    public string Signature;
    public int ConfigVersion;           // 配置版本号，用于热更新检测

    // AB 差异化配置（按 abGroup 名索引）
    public Dictionary<string, ActivityVariant> Variants;
}

[System.Serializable]
public class MissionDef
{
    public string MissionId;
    public MissionType Type;          // Daily, Cumulative, OneTime
    public string ConditionId;
    public long TargetValue;
    public string[] RewardIds;
    public int SortOrder;
}

[System.Serializable]
public class RewardDef
{
    public string RewardId;
    public RewardType Type;           // Currency, Item, Skin, Mail
    public string ItemId;
    public int Amount;
    public Rarity Rarity;
}

/// <summary>AB 分组差异化配置</summary>
[System.Serializable]
public class ActivityVariant
{
    public string GroupName;              // "A", "B", "control"
    public MissionDef[] OverrideMissions; // 覆盖默认任务
    public RewardDef[] OverrideRewards;   // 覆盖默认奖励
    public string OverrideCustomData;     // 覆盖自定义数据
}
```

### 5.3 实时开关（RemoteSwitch）

```csharp
public class RemoteSwitch
{
    public bool IsEnabled;
    public string AbGroup;            // "A", "B", "control"
    public long StartTimeOffset;      // 秒
    public long EndTimeOffset;
    public int MaxDailyRewardCount;   // 客户端预检用（服务端为权威）
    public long MaxDailyRewardValue;  // 客户端预检用
}
```

### 5.4 配置合并规则

```csharp
public class ConfigManager
{
    public ResolvedConfig Resolve(string activityId)
    {
        var baseConfig = _configCache[activityId];
        var remoteSwitch = _switchCache.GetValueOrDefault(activityId);

        // 1. 确定 AB 分组
        var abGroup = remoteSwitch?.AbGroup ?? "default";

        // 2. 如果有该分组的差异化配置，应用覆盖
        var variant = baseConfig.Variants?.GetValueOrDefault(abGroup);

        var resolved = new ResolvedConfig
        {
            ActivityId = baseConfig.ActivityId,
            StartTime = DateTimeOffset.FromUnixTimeSeconds(
                baseConfig.StartTimeUnix + (remoteSwitch?.StartTimeOffset ?? 0)),
            EndTime = DateTimeOffset.FromUnixTimeSeconds(
                baseConfig.EndTimeUnix + (remoteSwitch?.EndTimeOffset ?? 0)),
            IsEnabled = remoteSwitch?.IsEnabled ?? true,
            AbGroup = abGroup,
            Missions = variant?.OverrideMissions ?? baseConfig.Missions,
            RewardPool = variant?.OverrideRewards ?? baseConfig.RewardPool,
            CustomDataJson = variant?.OverrideCustomData ?? baseConfig.CustomDataJson,
            MaxDailyRewardCount = remoteSwitch?.MaxDailyRewardCount ?? 9999,
            MaxDailyRewardValue = remoteSwitch?.MaxDailyRewardValue ?? long.MaxValue,
        };

        return resolved;
    }
}
```

### 5.5 配置完整性校验

```csharp
public class ConfigIntegrityValidator
{
    /// <summary>HMAC-SHA256 签名验证</summary>
    public bool ValidateSignature(ActivityConfig config, string secretKey)
    {
        var payload = SerializeWithoutSignature(config);
        var expectedSig = ComputeHMACSHA256(payload, secretKey);
        return config.Signature == expectedSig;
    }
}
```

### 5.6 配置缓存与离线策略

| 场景 | 策略 |
|------|------|
| 首次启动（无缓存） | 使用随包内置的活动清单（只有常驻活动如签到） |
| 正常启动（有缓存） | 先用缓存秒开游戏，后台异步更新 |
| 弱网/离线 | 使用上次缓存配置运行活动，不阻断游戏 |
| 配置版本不一致 | 以服务端 manifestVersion 为准，客户端缓存自动覆盖 |
| 用户进入不存在的活动 | 友好提示"活动已结束"，从列表移除 |

**延迟加载策略**：活动清单一次性拉取（数据量小），但每个活动的完整 ActivityConfig JSON 在用户即将打开时才拉取，避免一次性下载全部活动配置。

### 5.7 Firebase RC 实际行为与策略

Firebase Remote Config 默认有 12 小时缓存，调 `fetch()` 有节流限制。

- **紧急开关（紧急下线）**：调用 `fetchAndActivate()` 主动拉取 + HTTP 轮询兜底（每 15 分钟检查 kill switch）
- **非紧急参数**：跟随默认缓存周期，不主动 fetch

### 5.8 配置热更新流程

1. Firebase RC 推送时携带新的 `configVersion`
2. 宿主比对版本号，发现变化
3. ConfigManager 重新生成 ResolvedConfig
4. 调用活动模块的 `OnConfigUpdated(resolvedConfig)`
5. 模块自行决定如何响应（刷新 UI、重置任务列表等）

---

## 6. 奖励系统与经济监控

### 6.1 奖励管线全景

```
    活动模块                    宿主核心层                    服务端

┌────────────┐
│ Activity   │  1. 构造 RewardData[]
│ Module     │     (活动定义了"应该发什么")
│            │──────────────┐
└────────────┘              │
                            ▼
                   ┌─────────────────┐
                   │ RewardValidator │  2. 客户端预校验
                   │                 │     (格式/引用合法性)
                   └────────┬────────┘
                            │ 通过
                            ▼
                   ┌─────────────────┐
                   │ EconomyMonitor  │  3. 客户端经济预检
                   │                 │     (日上限/价值上限)
                   │ · 累计今日已发  │     超限则截断或转邮件
                   │ · 比对 RemoteSwitch│
                   └────────┬────────┘
                            │ 通过
                            ▼
                   ┌─────────────────┐
                   │ RewardService   │  4. 发放执行
                   │                 │     客户端即时发放
                   │ · 货币/道具入库 │     (先发后校验)
                   │ · 触发UI飘字    │
                   └────────┬────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │   HTTP API      │  5. 服务端二次校验
                   │ POST /reward    │     (权威经济审计)
                   │                 │
                   │ · 验签          │     不合法 → 回滚
                   │ · 校验活动合法性│     标记异常账号
                   │ · 校验日上限    │     触发风控告警
                   │ · 写入审计日志  │
                   └────────┬────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │ RewardResult    │  6. 最终结果回调
                   │                 │
                   │ · 成功/部分成功 │     活动模块据此
                   │ · 回滚项列表    │     更新 UI 状态
                   │ · 剩余可发额度  │
                   └─────────────────┘
```

### 6.2 核心数据结构

```csharp
[System.Serializable]
public class RewardData
{
    public string SourceActivityId;     // 来源活动（审计用）
    public string SourceMissionId;      // 来源任务（审计用，可选）
    public RewardType Type;
    public string ItemId;
    public int Amount;
    public Rarity Rarity;
    public string MailFallbackId;       // 无法直接发放时的邮件补偿模板
}

public enum RewardType
{
    Currency, Item, Skin, Character, Energy, Mail
}

public enum Rarity
{
    Common, Rare, Epic, Legendary, Mythic
}
```

### 6.3 经济监控器

```csharp
/// <summary>
/// 经济监控器 — 防止单活动/单日产出失控
/// 客户端预检 + 服务端权威校验 双保险
/// </summary>
public class EconomyMonitor
{
    private readonly Dictionary<string, DailyCounter> _counters;

    /// <summary>
    /// 客户端预检 — 在发放前检查是否超限
    /// 返回截断后的可发数量 + 需转邮件的部分
    /// </summary>
    public EconomyCheckResult PreCheck(string activityId, RewardData[] rewards)
    {
        var limit = _resolvedLimits[activityId];
        var counter = GetOrCreateCounter(activityId);
        var result = new EconomyCheckResult();

        foreach (var reward in rewards)
        {
            var key = $"{reward.Type}_{reward.ItemId}";
            var todayCount = counter.GetTodayCount(key);
            var itemLimit = limit.GetPerItemLimit(reward.Type, reward.ItemId);

            int grantable = Math.Min(reward.Amount, itemLimit - todayCount);

            if (grantable <= 0)
            {
                result.MailFallbacks.Add(reward);
            }
            else if (grantable < reward.Amount)
            {
                result.Granted.Add(reward with { Amount = grantable });
                result.MailFallbacks.Add(reward with {
                    Amount = reward.Amount - grantable
                });
            }
            else
            {
                result.Granted.Add(reward);
            }
        }

        return result;
    }

    /// <summary>记录发放成功（服务端确认后调用）</summary>
    public void RecordGrant(string activityId, RewardData[] granted) { }

    /// <summary>每日重置（UTC 0点）</summary>
    public void ResetDaily() { }
}
```

### 6.4 安全策略（修正）

> **经济参数的权威来源是服务端。**
> 客户端 RemoteSwitch 中的 `MaxDailyRewardCount` / `MaxDailyRewardValue` 只用于：
> 1. UI 提示（"今日已达上限"）
> 2. 提前拦截（避免无意义的网络请求）
>
> 实际发放奖励时，`IRewardService.GrantRewardsAsync()` 必须携带活动 ID 发到服务端二次校验。
> 服务端拒绝 → 回滚已发放的奖励 + 上报异常账号。

### 6.5 邮件补偿系统

触发场景：
- 奖励发放时玩家不在线 → 服务端存邮件
- 经济监控截断超出部分 → 超出部分转邮件
- 活动结束时玩家未领取 → 未领奖励转邮件
- 服务端回滚后补偿 → 等价补偿转邮件

邮件结构：标题 + 正文 + 附件奖励列表 + 7天过期时间。

### 6.6 活动结束结算

```csharp
public class ActivitySettlement
{
    public async UniTask SettleAsync(string activityId)
    {
        // 1. 服务端扫描未领取奖励
        var pending = await _network.PostAsync("/api/activity/settle",
            new { activityId });

        // 2. 未领取奖励转邮件（服务端负责）
        // 3. 清理客户端活动状态
        _stateStore.ClearActivityState(activityId);
        // 4. 埋点
        _analytics.Track("activity_settled", new { activityId, pending.MailCount });
    }
}
```

---

## 7. 模块热更部署

### 7.1 打包目录约定

每个活动模块在 YooAssets 中作为一个独立 Catalog：

```
Catalog 名 = activityId (如 "act_halloween_2026")

{activityId}/
  ├── assembly/
  │   ├── HalloweenModule.dll           ← HybridCLR 程序集
  │   └── HalloweenModule.dll.symbols   ← 调试符号(可选)
  ├── ui/                               ← UI Prefab
  ├── atlas/                            ← 图集
  ├── audio/                            ← 音频
  └── config.json                       ← 默认配置(随包版本)
```

### 7.2 构建管线

```
CI/CD 流程：
1. 编译活动独立程序集
   - msbuild Activity.Halloween2026.csproj
   - HybridCLR AOT 补充元数据生成
2. 打包资源（Prefab/图集/音频）
   - YooAssets BuildPipeline
3. 生成版本目录文件
4. 上传到 YooAssets CDN
5. 更新活动清单 manifest（指向新的资源版本）
```

### 7.3 客户端加载流程

```
Loading 状态子流程：

1. 检查活动清单，获取目标活动的 resourceCatalog
2. YooAssets 检查该 catalog 的版本号
   ├─ 版本一致 → 跳过下载
   └─ 版本不一致 → 下载增量包
3. HybridCLR LoadAssembly(assemblyName)
4. 反射创建 entryClass 实例
5. 版本兼容性检查（ModuleCompatChecker）
6. 注入 ActivityContext，调用 OnInitAsync()
```

### 7.4 AOT 补充元数据策略

- 活动模块只引用宿主 SPI 接口，不直接引用宿主内部实现
- 宿主预编译时生成一份通用 AOT 补充元数据（涵盖所有 SPI 接口和基础类型）
- 活动模块构建时声明依赖的 AOT 元数据列表，随模块一起打包下发

---

## 8. 容错与异常处理

### 8.1 三层异常隔离

| 层级 | 范围 | 策略 |
|------|------|------|
| Level 1: 方法级 | 单个方法调用 | try-catch 包裹宿主对模块的每次调用，单方法异常不影响模块其他功能 |
| Level 2: 模块级熔断 | 单个活动模块 | 单次会话异常 > 5 次 → 自动挂起该活动，显示"活动暂时不可用" |
| Level 3: 宿主级兜底 | 整个活动系统 | 活动系统崩溃 → 禁用所有活动，核心游戏不受影响 |

### 8.2 熔断器

```csharp
public class ModuleCircuitBreaker
{
    private readonly string _activityId;
    private readonly int _maxErrors = 5;
    private readonly float _cooldownSeconds = 60f;

    private int _errorCount;
    private float _lastErrorTime;
    private CircuitState _state = CircuitState.Closed;

    public CircuitState State => _state;

    public void RecordException(Exception ex)
    {
        _errorCount++;
        _lastErrorTime = Time.time;

        Analytics.Track("module_exception", new {
            activityId = _activityId,
            errorCount = _errorCount,
            exception = ex.GetType().Name,
            message = ex.Message,
        });

        if (_errorCount >= _maxErrors)
        {
            _state = CircuitState.Open;
            Debug.LogError($"[ActivityHost] 模块 {_activityId} 触发熔断，" +
                          $"累计异常 {_errorCount} 次");
        }
    }

    public bool TryReset()
    {
        if (_state == CircuitState.Open &&
            Time.time - _lastErrorTime > _cooldownSeconds)
        {
            _state = CircuitState.HalfOpen;
            _errorCount = 0;
            return true;
        }
        return _state != CircuitState.Open;
    }
}

public enum CircuitState { Closed, Open, HalfOpen }
```

### 8.3 网络异常分级

| 网络场景 | 处理策略 | 玩家感知 |
|---------|---------|---------|
| 配置拉取超时 | 使用本地缓存，4 小时后重试 | 无感知（降级运行） |
| 奖励发放请求失败 | 客户端已发放，请求加入重试队列 | 无感知（最终一致） |
| 奖励校验请求失败 | 保留本地发放，下次启动补偿校验 | 无感知 |
| 活动清单拉取失败 | 使用随包默认清单 | 无感知 |
| Firebase RC 拉取被限流 | 保持上次值，12 小时后自动重试 | 无感知 |
| 完全离线 | 活动可玩但进度暂存本地，联网后同步 | 弱提示 |

---

## 9. 埋点与分析集成

### 9.1 标准化事件模型

所有活动共用同一套事件模型，方便 BI 做跨活动横向对比。

```csharp
public static class ActivityAnalytics
{
    // 漏斗事件
    public const string Impression      = "activity_impression";    // 入口曝光
    public const string Click           = "activity_click";         // 点击进入
    public const string Enter           = "activity_enter";         // 加载完成
    public const string MissionStart    = "activity_mission_start"; // 开始任务
    public const string MissionComplete = "activity_mission_complete";
    public const string RewardClaim     = "activity_reward_claim";  // 领取奖励
    public const string Purchase        = "activity_purchase";      // 内购
    public const string Exit            = "activity_exit";          // 离开活动

    // 异常事件
    public const string LoadFail        = "activity_load_fail";
    public const string Exception       = "module_exception";
    public const string RewardRollback  = "reward_rollback";
}

public static class StandardParams
{
    public const string ActivityId    = "activity_id";
    public const string ModuleVersion = "module_version";
    public const string AbGroup       = "ab_group";
    public const string SessionId     = "session_id";
    public const string DayIndex      = "day_index";     // 活动第几天
}
```

---

## 10. 测试策略

### 10.1 单元测试

| 测试目标 | 框架 | 覆盖范围 |
|---------|------|---------|
| 宿主核心子系统 | Unity Test Framework | 调度器时间计算、配置合并逻辑、经济监控计数、熔断器状态机 |
| SPI 接口契约 | Unity Test Framework | Mock IActivityContext，验证模块对宿主服务的调用合规性 |
| 活动模块 | Unity Play Mode | Mock 宿主环境，测试模块生命周期流转 |

### 10.2 活动模块测试基类

```csharp
public abstract class ActivityModuleTestBase<TModule> where TModule : IActivityModule, new()
{
    protected MockActivityContext Context { get; private set; }
    protected TModule Module { get; private set; }

    [SetUp]
    public virtual async UniTask SetUp()
    {
        Module = new TModule();
        Context = new MockActivityContext();

        Context.RegisterService<IRewardService>(new MockRewardService());
        Context.RegisterService<IResourceService>(new MockResourceService());
        Context.RegisterService<IPlayerInfoService>(new MockPlayerInfoService());
        Context.RegisterService<INetworkService>(new MockNetworkService());

        await Module.OnInitAsync(Context);
    }

    protected async UniTask SimulateFullLifecycle()
    {
        await Module.OnEnterAsync();
        await UniTask.DelayFrame(10);
        await Module.OnExitAsync();
        await Module.OnDisposeAsync();
    }
}
```

### 10.3 端到端验证清单

每个新活动上线前必须通过：

- [ ] 冷启动 → 活动入口正常显示
- [ ] 点击进入 → 活动加载 < 3 秒
- [ ] 弱网环境(3G) → 加载有 Loading 提示，不卡死
- [ ] 完全离线 → 使用缓存正常运行
- [ ] 杀进程重启 → 进度未丢失
- [ ] 活动时间到 → 自动关闭，奖励转邮件
- [ ] 模拟异常(注入空引用) → 熔断器触发，显示友好提示
- [ ] ABTest A/B 组 → 配置正确分流
- [ ] 同设备切换 3 个活动 → 内存不泄漏
- [ ] 活动结束后入口消失 → 列表正确刷新

---

## 11. 关键设计决策汇总

| 决策 | 选择 | 理由 |
|------|------|------|
| 架构模式 | 微内核 + 活动插件 | 与 HybridCLR 天然契合，隔离性最强，团队协作友好 |
| 热更方式 | HybridCLR 程序集热更 | 与现有休闲游戏框架一致，类型安全，可调试 |
| 配置下发 | 自研后台 + Firebase 双层 | 复杂结构化数据走后台，实时开关和 AB 分组走 Firebase |
| UI 构建 | 每活动独立 Prefab + 热更代码 | 灵活度最高，符合完整活动生态定位 |
| 奖励系统 | 统一管线 + 经济监控 | 安全性最高，防止经济崩溃 |
| 经济安全 | 客户端预检 + 服务端权威 | 先发后校验保证体验，服务端校验保证安全 |
| 前台活动数 | 同时 1 个 | 休闲游戏屏幕空间有限 |
| 配置完整性 | HMAC-SHA256 签名 | 防止 CDN 篡改和中间人攻击 |
| AB 差异化 | Variants 字段按分组覆盖 | 支持不同 AB 组有完全不同的活动配置 |
| 异常隔离 | 三层防线（方法/模块/宿主） | 单活动崩溃不影响核心游戏 |

---

## 相关链接

- [[休闲游戏框架]] — 现有框架（UI/事件系统/资源管理/热更）
- [[HybridCLR 热更新]] — 热更技术栈
- [[YooAsset 与 Addressables 方案对比]] — 资源管理选型
- [[ABTest 基础概念与运营方法论]] — ABTest 集成参考
- [[事件系统实现对比]] — C# 事件系统设计参考
- [[休闲游戏关卡可解性保证]] — 关卡生成算法参考
