---
title: 【架构决策】UI 架构 - MVC vs MVP
tags: ["C#", "Unity", "架构", "UI", "MVC", "MVP", "架构决策", "界面架构"]
category: 架构设计/架构决策
created: "2026-07-03 10:00"
updated: "2026-07-03 00:00"
description: Unity UI开发中MVC与MVP两种架构模式的深度对比，包含MVC完整实现、演化逻辑、迁移路径和选型决策框架
unity_version: 2021.3+
status: 待验证
validation: Demo验证
related: ["[[【架构决策】UI架构-MVP vs MVVM]]", "[[【教程】MVP模式深入讲解]]", "[[【教程】项目架构设计]]", "[[【教程】依赖注入与服务定位器]]"]
author: llm
sources:
  - https://martinfowler.com/eaaDev/uiArchs.html
  - https://martinfowler.com/eaaDev/PassiveScreen.html
  - https://blog.unity.com/technology/architecture-patterns-for-gameplay
---

# 【架构决策】UI 架构 - MVC vs MVP

> 核心问题：Unity UI 开发中，MVC 和 MVP 到底差在哪？该选哪一个？

## 文档定位

本文档从**架构演化**视角分析 MVC 到 MVP 的演进逻辑，在 Unity 场景下对比两种模式的实现差异、测试能力和适用边界。与 [[【架构决策】UI架构-MVP vs MVVM]] 互补——那篇聚焦 MVP vs MVVM 的选型，本篇聚焦 MVC vs MVP 的选型。

**相关文档**：[[【教程】MVP模式深入讲解]]、[[【架构决策】UI架构-MVP vs MVVM]]、[[【教程】项目架构设计]]

---

## 1. 为什么需要这篇文档

很多 Unity 开发者的架构认知路径是：

```
面条代码（所有逻辑堆在 MonoBehaviour 里）
    ↓
隐式 MVC（Model + View 混合，Controller 散落各处）
    ↓
显式 MVP（View 接口化，Presenter 解耦）
    ↓
按需 MVVM（数据绑定 + 响应式）
```

中间这个「隐式 MVC → 显式 MVP」的跨越，恰恰是最容易踩坑的阶段。常见疑问：

- MVC 不就是 Model + View + Controller 吗，为什么 Unity 里用起来总是变形？
- 我已经在用 MVC 了，有必要迁移到 MVP 吗？
- Unity 的 MonoBehaviour 天然像 Controller，这算 MVC 吗？

本篇逐一回答。

---

## 2. MVC 模式详解

### 2.1 经典 MVC 三角形

```
┌──────────────────────────────────────────────────────────────┐
│                      经典 MVC                                 │
│                                                              │
│                    ┌───────────┐                            │
│         用户操作    │           │      通知状态变化           │
│       ──────────→  │  Model    │  ────────────────┐         │
│       │            │  (模型)    │                   │         │
│       │            └───────────┘                   │         │
│       │                    ↑                        ↓         │
│       │                    │   查询/更新状态         │         │
│       │                    │                        ↓         │
│  ┌────┴──────┐      ┌───────────┐         ┌───────────┐     │
│  │           │←────│  Controller │─────────→│           │     │
│  │   View    │     │  (控制器)   │  更新视图  │   View    │     │
│  │  (视图)   │      └───────────┘           │  (视图)   │     │
│  └───────────┘                              └───────────┘     │
│                                                              │
│  关键特征：                                                   │
│  ① View 可以直接读 Model（不是只能通过 Controller）           │
│  ② Controller 负责处理输入，但不一定负责所有 View 更新        │
│  ③ Model 不应该知道 View/Controller 的存在                    │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 MVC 在 Unity 中的典型（变形）实现

Unity 没有原生的 MVC 框架，MonoBehaviour 的特殊性导致 MVC 几乎总会变形：

```csharp
// ============ Model - 纯数据 + 业务逻辑 ============

/// <summary>
/// 玩家数据模型
/// </summary>
public class PlayerModel
{
    public int Hp { get; private set; }
    public int MaxHp { get; private set; }
    public int Level { get; private set; }
    public int Exp { get; private set; }

    // 事件 - 通知 View 数据变化（这已经模糊了和 MVP 的边界）
    public event Action<PlayerModel> OnDataChanged;

    public PlayerModel(int maxHp)
    {
        MaxHp = maxHp;
        Hp = maxHp;
        Level = 1;
        Exp = 0;
    }

    public void TakeDamage(int damage)
    {
        Hp = Mathf.Max(0, Hp - damage);
        OnDataChanged?.Invoke(this);
    }

    public void Heal(int amount)
    {
        Hp = Mathf.Min(MaxHp, Hp + amount);
        OnDataChanged?.Invoke(this);
    }

    public void AddExp(int amount)
    {
        Exp += amount;
        while (Exp >= Level * 100)
        {
            Exp -= Level * 100;
            Level++;
            MaxHp += 10;
            Hp = MaxHp;
        }
        OnDataChanged?.Invoke(this);
    }
}
```

```csharp
// ============ View - 直接持有 Model 引用 ============

/// <summary>
/// HUD 视图 - 直接读取 Model 数据来显示
/// 注意：这是 MVC 风格，View 知道 Model 的存在
/// </summary>
public class HUDView : MonoBehaviour
{
    [SerializeField] private Slider hpSlider;
    [SerializeField] private Text hpText;
    [SerializeField] private Text levelText;
    [SerializeField] private Text expText;

    // View 持有 Model 的引用 —— 这是 MVC 的标志特征
    private PlayerModel model;

    /// <summary>
    /// 注入 Model 并订阅变化
    /// </summary>
    public void Initialize(PlayerModel playerModel)
    {
        model = playerModel;
        model.OnDataChanged += UpdateView;
        UpdateView();
    }

    /// <summary>
    /// View 直接从 Model 读取数据
    /// </summary>
    private void UpdateView()
    {
        hpSlider.maxValue = model.MaxHp;
        hpSlider.value = model.Hp;
        hpText.text = $"{model.Hp} / {model.MaxHp}";
        levelText.text = $"Lv.{model.Level}";
        expText.text = $"EXP: {model.Exp}/{model.Level * 100}";
    }

    private void OnDestroy()
    {
        model.OnDataChanged -= UpdateView;
    }
}
```

```csharp
// ============ Controller - 处理输入和游戏逻辑 ============

/// <summary>
/// 游戏控制器 - 处理用户输入，调用 Model
/// 在 MVC 中 Controller 不负责「通知 View 更新」，View 自己监听 Model
/// </summary>
public class GameController : MonoBehaviour
{
    [SerializeField] private HUDView hudView;
    [SerializeField] private Button attackButton;
    [SerializeField] private Button healButton;

    private PlayerModel playerModel;

    private void Awake()
    {
        playerModel = new PlayerModel(maxHp: 100);
    }

    private void Start()
    {
        // View 直接绑定 Model
        hudView.Initialize(playerModel);

        attackButton.onClick.AddListener(() =>
        {
            playerModel.TakeDamage(10);
        });

        healButton.onClick.AddListener(() =>
        {
            playerModel.Heal(20);
        });
    }
}
```

### 2.3 MVC 的隐含问题

| 问题 | 表现 | 后果 |
|------|------|------|
| **View 知道 Model** | `HUDView` 持有 `PlayerModel` 引用 | View 与 Model 紧耦合，Model 结构变化直接波及 View |
| **View 包含显示逻辑** | `UpdateView()` 里的格式化、条件判断 | View 难以测试，逻辑分散 |
| **测试困难** | 想测试 HP 格式化逻辑，必须构造整个 PlayerModel | 单元测试成本高，容易放弃 |
| **多 View 协调混乱** | 多个 View 监听同一个 Model，更新顺序不确定 | 出 Bug 时难以定位是哪个 View 的问题 |
| **Controller 定位模糊** | Unity 中 MonoBehaviour 既像 View 又像 Controller | 开发者分不清职责边界，代码逐渐腐化 |

---

## 3. MVP 模式：MVC 的改良

### 3.1 核心变化：切断 View → Model 的直接依赖

```
┌──────────────────────────────────────────────────────────────┐
│                    MVC → MVP 演化                             │
│                                                              │
│   MVC（View 直接读 Model）:                                   │
│                                                              │
│        View ←──────→ Model                                   │
│         ↑                   ↑                                │
│         └──── Controller ───┘                                │
│                                                              │
│   MVP（View 不认识 Model）:                                   │
│                                                              │
│        View ←──────→ Presenter ←──────→ Model                │
│                  (通过 IView 接口)                            │
│                                                              │
│   切断的箭头：View → Model                                    │
│   新增的中间层：Presenter（替 Controller 拿到对 View 的       │
│   完全控制权，且通过接口操作 View，实现可测试）                │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 同一个功能用 MVP 实现

```csharp
// ============ Model - 完全不变 ============
// PlayerModel 与 MVC 版本完全相同，纯数据 + 业务逻辑

// ============ View Interface - 新增！这是 MVP 的核心 ============

/// <summary>
/// HUD 视图接口 - Presenter 通过接口操作 View
/// </summary>
public interface IHUDView
{
    void SetHpDisplay(int currentHp, int maxHp);
    void SetLevelDisplay(int level);
    void SetExpDisplay(int currentExp, int expToNext);
    void PlayDamageEffect();
    void PlayHealEffect();
    void PlayLevelUpEffect();
}
```

```csharp
// ============ View - 实现接口，不再持有 Model ============

/// <summary>
/// HUD 视图 - 只负责显示，不知道 Model 的存在
/// </summary>
public class HUDView : MonoBehaviour, IHUDView
{
    [SerializeField] private Slider hpSlider;
    [SerializeField] private Text hpText;
    [SerializeField] private Text levelText;
    [SerializeField] private Text expText;
    [SerializeField] private Image damageFlash;

    // 注意：View 不持有 PlayerModel，也不知道 PlayerModel 是什么

    public void SetHpDisplay(int currentHp, int maxHp)
    {
        hpSlider.maxValue = maxHp;
        hpSlider.value = currentHp;
        hpText.text = $"{currentHp} / {maxHp}";
    }

    public void SetLevelDisplay(int level)
    {
        levelText.text = $"Lv.{level}";
    }

    public void SetExpDisplay(int currentExp, int expToNext)
    {
        expText.text = $"EXP: {currentExp}/{expToNext}";
    }

    public void PlayDamageEffect()
    {
        // 播放受击特效
        damageFlash.color = new Color(1, 0, 0, 0.3f);
        StartCoroutine(FadeFlash());
    }

    public void PlayHealEffect()
    {
        damageFlash.color = new Color(0, 1, 0, 0.3f);
        StartCoroutine(FadeFlash());
    }

    public void PlayLevelUpEffect()
    {
        // 播放升级特效
    }

    private IEnumerator FadeFlash()
    {
        var color = damageFlash.color;
        while (color.a > 0)
        {
            color.a -= Time.deltaTime * 2f;
            damageFlash.color = color;
            yield return null;
        }
    }
}
```

```csharp
// ============ Presenter - 替代 Controller ============

/// <summary>
/// HUD 展示器 - 协调 Model 和 View，所有显示逻辑集中于此
/// </summary>
public class HUDPresenter
{
    private readonly IHUDView view;
    private readonly PlayerModel model;

    public HUDPresenter(IHUDView view, PlayerModel model)
    {
        this.view = view;
        this.model = model;

        // Presenter 监听 Model 变化（而不是 View 监听）
        model.OnDataChanged += OnModelChanged;

        // 初始更新
        UpdateView();
    }

    private void OnModelChanged(PlayerModel changedModel)
    {
        UpdateView();
    }

    /// <summary>
    /// 所有 View 更新逻辑集中在这里
    /// </summary>
    private void UpdateView()
    {
        view.SetHpDisplay(model.Hp, model.MaxHp);
        view.SetLevelDisplay(model.Level);
        view.SetExpDisplay(model.Exp, model.Level * 100);
    }

    // 对外暴露的操作方法
    public void TakeDamage(int damage)
    {
        model.TakeDamage(damage);
        view.PlayDamageEffect();
    }

    public void Heal(int amount)
    {
        model.Heal(amount);
        view.PlayHealEffect();
    }

    public void Dispose()
    {
        model.OnDataChanged -= OnModelChanged;
    }
}
```

```csharp
// ============ 装配 - 在场景中连接各组件 ============

/// <summary>
/// 场景装配器 - 创建并连接 View、Model、Presenter
/// </summary>
public class GameBootstrap : MonoBehaviour
{
    [SerializeField] private HUDView hudView;
    [SerializeField] private Button attackButton;
    [SerializeField] private Button healButton;

    private PlayerModel playerModel;
    private HUDPresenter hudPresenter;

    private void Awake()
    {
        playerModel = new PlayerModel(maxHp: 100);

        // Presenter 持有 View 接口，而非具体 View 类
        hudPresenter = new HUDPresenter(hudView, playerModel);
    }

    private void Start()
    {
        attackButton.onClick.AddListener(() => hudPresenter.TakeDamage(10));
        healButton.onClick.AddListener(() => hudPresenter.Heal(20));
    }

    private void OnDestroy()
    {
        hudPresenter?.Dispose();
    }
}
```

---

## 4. MVC vs MVP 逐项对比

### 4.1 依赖关系对比

```
MVC:
  View ──→ Model        ← 直接依赖！
  View ←──→ Controller
  Controller ──→ Model

MVP:
  View ──→ (nothing)    ← View 不知道 Model
  View ←── Presenter    ← 通过 IView 接口
  Presenter ──→ Model
  Presenter ──→ IView   ← 通过接口，不依赖具体 View
```

### 4.2 全维度对比表

| 维度 | MVC | MVP | 差异说明 |
|------|-----|-----|----------|
| **View 能否访问 Model** | 可以 | 不可以 | MVP 的 View 只暴露接口方法，数据由 Presenter 推送 |
| **中间层名称** | Controller | Presenter | Controller 处理输入；Presenter 负责所有 View↔Model 协调 |
| **View 的逻辑量** | 中等（含显示逻辑） | 极少（纯展示） | MVC 的 View 要决定怎么显示数据；MVP 的 View 只管执行 |
| **View 可测试性** | 差（依赖 Model + MonoBehaviour） | 好（通过 Mock IView） | MVP 可以完全脱离 Unity 测试 Presenter |
| **View 可替换性** | 低（换 View 要改数据绑定） | 高（新 View 实现 IView 即可） | MVP 的 Presenter 不关心 View 是 UGUI 还是 UI Toolkit |
| **代码量** | 较少 | 较多（多一层接口） | MVP 的接口定义是额外成本 |
| **适合团队规模** | 1-3 人 | 3+ 人 | 小团队 MVC 足够；大团队需要 MVP 的接口契约 |
| **适合项目阶段** | 原型/Demo | 正式项目 | 快速验证用 MVC；量产需要 MVP 的可维护性 |

### 4.3 同一个功能：MVC vs MVP 代码对比

```
场景：按钮点击扣减 HP，更新血条

MVC 流程：
  按钮 → Controller.TakeDamage() → Model.TakeDamage()
                                        ↓
                                  OnDataChanged 事件
                                        ↓
                                  View.UpdateView()  ← View 自己监听

MVP 流程：
  按钮 → Presenter.TakeDamage()
              ↓         ↓
         Model.Take   View.SetHpDisplay()
         Damage()     View.PlayDamageEffect()
                        ↑ Presenter 主动推送

关键差异：
  - MVC: View 监听 Model 变化，自己决定怎么显示
  - MVP: Presenter 监听 Model 变化，主动告诉 View 怎么显示
```

---

## 5. 测试能力对比（关键差异）

这是 MVP 相比 MVC 最大的优势。

### 5.1 MVC 的测试困境

```csharp
// ❌ MVC: 想测试 "HP 低于 30% 时血条变红" 这个逻辑
// 问题：这个逻辑在 HUDView.UpdateView() 里，而 HUDView 是 MonoBehaviour
//       必须在 Unity Editor 中运行才能测试

// HUDView.cs 中的显示逻辑（无法脱离 Unity 测试）
private void UpdateView()
{
    hpSlider.value = model.Hp;

    // 这个逻辑无法单元测试！
    if ((float)model.Hp / model.MaxHp < 0.3f)
        hpText.color = Color.red;
    else
        hpText.color = Color.white;
}
```

### 5.2 MVP 的测试优势

```csharp
// ✅ MVP: 显示逻辑移到 Presenter 中，可以完全脱离 Unity 测试

[Test]
public void Presenter_LowHp_DisplaysRedColor()
{
    // Arrange - 创建 Mock View
    var mockView = new MockHUDView();
    var model = new PlayerModel(maxHp: 100);
    var presenter = new HUDPresenter(mockView, model);

    // Act - 扣血到低血量
    model.TakeDamage(75); // Hp = 25, 低于 30%

    // Assert - Presenter 应该通知 View 显示红色
    Assert.IsTrue(mockView.LastHpColorRequest == HpColor.Low);
}

[Test]
public void Presenter_TakeDamage_PlaysDamageEffect()
{
    var mockView = new MockHUDView();
    var model = new PlayerModel(maxHp: 100);
    var presenter = new HUDPresenter(mockView, model);

    presenter.TakeDamage(10);

    Assert.AreEqual(1, mockView.DamageEffectCallCount);
    Assert.AreEqual(90, mockView.LastDisplayedHp);
}

// Mock View - 不依赖 Unity 的纯 C# 类
public class MockHUDView : IHUDView
{
    public int LastDisplayedHp { get; private set; }
    public int LastDisplayedMaxHp { get; private set; }
    public HpColor LastHpColorRequest { get; private set; }
    public int DamageEffectCallCount { get; private set; }

    public void SetHpDisplay(int currentHp, int maxHp)
    {
        LastDisplayedHp = currentHp;
        LastDisplayedMaxHp = maxHp;

        // 显示颜色逻辑可以通过 Presenter 来决定
        float ratio = (float)currentHp / maxHp;
        LastHpColorRequest = ratio < 0.3f ? HpColor.Low : HpColor.Normal;
    }

    public void SetLevelDisplay(int level) { }
    public void SetExpDisplay(int currentExp, int expToNext) { }
    public void PlayDamageEffect() => DamageEffectCallCount++;
    public void PlayHealEffect() { }
    public void PlayLevelUpEffect() { }
}

public enum HpColor { Normal, Low }
```

---

## 6. Unity 特有的考量

### 6.1 MonoBehaviour 的身份困惑

Unity 开发者最常见的困惑：**MonoBehaviour 到底是 View 还是 Controller？**

```
┌──────────────────────────────────────────────────────────────┐
│              MonoBehaviour 的身份困惑                          │
│                                                              │
│  在 MVC 中：                                                 │
│  ┌─────────────────────────────────┐                        │
│  │  HUDController : MonoBehaviour   │                        │
│  │  ─ 持有 UI 组件引用 (像 View)    │                        │
│  │  ─ 处理按钮事件 (像 Controller)  │                        │
│  │  ─ 直接修改 UI (像 View)         │                        │
│  │  身份：既是 View 又是 Controller  │ ← 问题根源              │
│  └─────────────────────────────────┘                        │
│                                                              │
│  在 MVP 中：                                                 │
│  ┌─────────────────────────────────┐                        │
│  │  HUDView : MonoBehaviour,        │                        │
│  │            IHUDView              │                        │
│  │  ─ 实现 IView 接口 (纯 View)     │                        │
│  │  ─ 不处理任何业务逻辑             │                        │
│  │  身份：明确的 View                │                        │
│  └─────────────────────────────────┘                        │
│  ┌─────────────────────────────────┐                        │
│  │  HUDPresenter (纯 C# 类)         │                        │
│  │  ─ 不继承 MonoBehaviour          │                        │
│  │  ─ 处理所有逻辑                  │                        │
│  │  身份：明确的 Presenter           │                        │
│  └─────────────────────────────────┘                        │
│  ┌─────────────────────────────────┐                        │
│  │  HUDBootstrap : MonoBehaviour    │                        │
│  │  ─ 负责创建 Presenter 并注入     │                        │
│  │  ─ 极少代码，只做装配            │                        │
│  └─────────────────────────────────┘                        │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 生命周期管理的差异

```csharp
// MVC: View 和 Controller 混在一起，生命周期简单但耦合
public class HUDController_MVC : MonoBehaviour
{
    // View 部分寿命 = GameObject 寿命
    // Controller 部分寿命 = GameObject 寿命
    // 无法分离，无法独立测试
}

// MVP: 各组件可以独立管理
public class HUDBootstrap : MonoBehaviour
{
    private HUDPresenter presenter; // 非 MonoBehaviour，可独立创建/销毁

    private void Awake()
    {
        var view = GetComponent<IHUDView>();
        var model = new PlayerModel(100);
        presenter = new HUDPresenter(view, model);
    }

    private void OnDestroy()
    {
        presenter?.Dispose(); // 显式释放，避免内存泄漏
    }
}
```

### 6.3 prefab  prefab 差异

| 方面 | MVC | MVP |
|------|-----|-----|
| **Prefab 结构** | 简单（一个脚本搞定） | 复杂（View + Bootstrap 两个脚本） |
| **Inspector 配置** | 绑定 UI 组件 + 配置参数 | View 只绑定 UI 组件；参数在 Presenter |
| **Prefab 复用** | 难（View 和逻辑耦合） | 容易（换 Presenter 即可改变行为） |

---

## 7. 选型决策框架

### 7.1 决策树

```
                    ┌─ 项目是原型/Demo？
                    │     ├─ YES → MVC（快速出结果）
                    │     └─ NO ↓
                    │
                    ├─ 团队规模 < 3 人？
                    │     ├─ YES → MVC（够用）
                    │     └─ NO ↓
                    │
                    ├─ 需要单元测试？
                    │     ├─ YES → MVP（必须可 Mock）
                    │     └─ NO ↓
                    │
                    ├─ UI 需要在多平台/多皮肤间复用？
                    │     ├─ YES → MVP（接口隔离）
                    │     └─ NO ↓
                    │
                    ├─ 项目预期维护 > 6 个月？
                    │     ├─ YES → MVP（长期可维护）
                    │     └─ NO → MVC（短期够用）
```

### 7.2 按项目阶段的推荐

| 阶段 | 推荐模式 | 原因 |
|------|----------|------|
| **Game Jam / 原型** | MVC 或面条 | 速度优先，架构不重要 |
| **垂直切片** | MVC → MVP 过渡 | 开始建立架构基线 |
| **正式开发** | MVP | 需要可测试、可维护、可并行开发 |
| **长期运营** | MVP + 局部 MVVM | 核心系统 MVP；数据密集 UI 用 MVVM |

### 7.3 按系统复杂度的推荐

| UI 类型 | 复杂度 | 推荐 |
|---------|--------|------|
| 单页简单 HUD | 低 | MVC |
| 多面板交互（设置、背包） | 中 | MVP |
| 数据驱动 UI（排行榜、商店） | 高 | MVP 或 MVVM |
| 复杂表单（角色编辑器） | 高 | MVP |

---

## 8. 从 MVC 迁移到 MVP 的实操路径

### 8.1 迁移四步法

```
Step 1: 识别 View 中的显示逻辑
  └─ 找到所有 "根据 Model 数据决定怎么显示" 的代码

Step 2: 提取 IView 接口
  └─ 把 View 的 public 方法抽象为接口

Step 3: 创建 Presenter
  └─ 把 View 中的显示逻辑移到 Presenter

Step 4: 切断 View → Model 引用
  └─ View 不再持有 Model，所有数据由 Presenter 推送
```

### 8.2 迁移示例

```csharp
// ============ 迁移前：MVC 风格 ============

public class InventoryView_MVC : MonoBehaviour
{
    [SerializeField] private Transform itemContainer;
    [SerializeField] private GameObject itemPrefab;

    private InventoryModel model; // ← View 直接持有 Model

    public void Initialize(InventoryModel inventoryModel)
    {
        model = inventoryModel;
        model.OnItemsChanged += RefreshUI; // ← View 监听 Model
    }

    private void RefreshUI()
    {
        // 显示逻辑混在 View 里
        foreach (Transform child in itemContainer)
            Destroy(child.gameObject);

        foreach (var item in model.Items)
        {
            var go = Instantiate(itemPrefab, itemContainer);
            go.GetComponent<ItemCell>().Setup(item);
        }
    }
}

// ============ 迁移后：MVP 风格 ============

// Step 1: 提取接口
public interface IInventoryView
{
    void ClearItems();
    void AddItem(ItemData item, int index);
    void ShowEmptyMessage(bool show);
}

// Step 2: View 实现接口，不再持有 Model
public class InventoryView : MonoBehaviour, IInventoryView
{
    [SerializeField] private Transform itemContainer;
    [SerializeField] private GameObject itemPrefab;
    [SerializeField] private GameObject emptyMessage;

    // 注意：没有 Model 引用！

    public void ClearItems()
    {
        foreach (Transform child in itemContainer)
            Destroy(child.gameObject);
    }

    public void AddItem(ItemData item, int index)
    {
        var go = Instantiate(itemPrefab, itemContainer);
        go.GetComponent<ItemCell>().Setup(item);
    }

    public void ShowEmptyMessage(bool show)
    {
        emptyMessage.SetActive(show);
    }
}

// Step 3: Presenter 承载逻辑
public class InventoryPresenter
{
    private readonly IInventoryView view;
    private readonly InventoryModel model;

    public InventoryPresenter(IInventoryView view, InventoryModel model)
    {
        this.view = view;
        this.model = model;
        model.OnItemsChanged += RefreshUI;
    }

    private void RefreshUI()
    {
        view.ClearItems();

        if (model.Items.Count == 0)
        {
            view.ShowEmptyMessage(true);
            return;
        }

        view.ShowEmptyMessage(false);
        for (int i = 0; i < model.Items.Count; i++)
        {
            view.AddItem(model.Items[i], i);
        }
    }

    public void Dispose()
    {
        model.OnItemsChanged -= RefreshUI;
    }
}
```

---

## 9. 常见误区辨析

### 误区 1：「Unity 的 MVC 就是 Model + View + MonoBehaviour」

```
❌ 错误认知：
   "我的脚本叫 XXXController，所以我在用 MVC"

   实际上这个 "Controller" 可能：
   - 直接操作 UI 组件（实际是 View）
   - 包含业务逻辑（实际是 Model）
   - 三者混在一起（实际是面条代码）

✅ 正确认知：
   MVC 是一种职责分离的架构模式，不是命名约定
   关键检验：Model 是否独立于 Unity？（不继承 MonoBehaviour）
```

### 误区 2：「MVP 一定比 MVC 好」

```
❌ 错误认知：
   "所有 UI 都必须用 MVP，否则就是架构不规范"

✅ 正确认知：
   - 加载界面、简单弹出框 → MVC 甚至直接写就够
   - 背包、战斗 HUD、社交面板 → MVP 更合适
   - 模式是工具，不是目的
```

### 误区 3：「Controller 和 Presenter 是一回事」

```
❌ 错误认知：
   "只是改了个名字"

✅ 正确认知：
   - Controller: 接收输入 → 调用 Model → 不主动更新 View
   - Presenter: 接收输入 → 调用 Model → 主动通过 IView 接口更新 View
   - Presenter 比 Controller 承担更多 View 协调职责
   - Presenter 通过接口操作 View，Controller 可能直接操作
```

---

## 10. 与其他架构决策文档的关系

```
                    本文档
               MVC vs MVP（职责分离对比）
                 ↑              ↑
                 │              │
    [[【教程】MVP模式深入讲解]]  [[【架构决策】UI架构-MVP vs MVVM]]
     （MVP 的完整教程）           （MVP vs MVVM 选型）
                 │              │
                 ↓              ↓
            [[【教程】项目架构设计]]
             （整体架构决策）
```

| 文档 | 聚焦 |
|------|------|
| **本文** | MVC vs MVP：View 能否访问 Model？中间层职责如何？ |
| [[【架构决策】UI架构-MVP vs MVVM]] | MVP vs MVVM：手动更新 vs 数据绑定？ |
| [[【教程】MVP模式深入讲解]] | MVP 的完整实现和最佳实践 |
| [[【教程】项目架构设计]] | Unity 项目整体架构选型 |

---

## 总结

### 一句话

> **MVC 允许 View 直接读 Model；MVP 切断这条线，用 Presenter + IView 接口做中间人。**

### 选型速查

| 如果你的情况是... | 推荐 |
|-------------------|------|
| 快速原型，3 人以下团队 | MVC |
| 需要单元测试，团队 3+ 人 | MVP |
| UI 逻辑简单，只是展示数据 | MVC |
| UI 逻辑复杂，有条件显示/多状态 | MVP |
| 短期项目（< 3 个月） | MVC |
| 长期维护项目 | MVP |

---

## 相关链接

- [[【架构决策】UI架构-MVP vs MVVM]]
- [[【教程】MVP模式深入讲解]]
- [[【教程】项目架构设计]]
- [[【教程】依赖注入与服务定位器]]
- [GUI Architectures - Martin Fowler](https://martinfowler.com/eaaDev/uiArchs.html)
- [Passive View - Martin Fowler](https://martinfowler.com/eaaDev/PassiveScreen.html)
- [Unity Architecture Patterns - Unity Blog](https://blog.unity.com/technology/architecture-patterns-for-gameplay)
