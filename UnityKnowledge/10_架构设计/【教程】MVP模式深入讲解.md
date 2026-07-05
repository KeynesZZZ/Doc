---
title: 【教程】MVP模式深入讲解
tags: ["C#", "Unity", "架构", "教程", "MVP", "UI架构", "设计模式"]
category: 架构设计/教程
created: "2024-01-10 09:00"
updated: "2026-05-29 00:00"
description: MVP模式在Unity UI开发中的深入讲解，包含Model-View-Presenter的职责划分和实现示例
unity_version: 2021.3+
status: 待验证
validation: Demo验证
related: ["[[【架构决策】UI架构-MVP vs MVVM]]", "[[【教程】项目架构设计]]", "[[【教程】依赖注入与服务定位器]]"]
author: llm
---

# 【教程】MVP模式深入讲解

> 补充课程 | 脚本与架构模块

## 文档定位

本文档从**使用角度**讲解MVP模式在Unity UI开发中的应用，涵盖Model-View-Presenter的职责划分、接口设计、数据绑定机制以及与MVC/MVVM的对比分析。

**相关文档**：[[【架构决策】UI架构-MVC vs MVP]]、[[【架构决策】UI架构-MVP vs MVVM]]、[[【教程】项目架构设计]]

---

## 1. MVP的核心概念

**MVP = Model - View - Presenter**

```
┌─────────────────────────────────────────────────────────────┐
│                        MVP 模式                              │
│                                                             │
│    ┌─────────────┐              ┌─────────────┐           │
│    │             │   用户操作    │             │           │
│    │    View     │─────────────→│  Presenter  │           │
│    │   (视图)     │              │  (展示器)    │           │
│    │             │←─────────────│             │           │
│    │             │   更新视图    │             │           │
│    └─────────────┘              └──────┬──────┘           │
│           ↑                           │                   │
│           │                           │                   │
│           │        ┌─────────────┐    │                   │
│           └────────│    Model    │←───┘                   │
│            数据变化  │   (模型)    │   数据操作             │
│                    └─────────────┘                        │
└─────────────────────────────────────────────────────────────┘

关键特点：
1. View 不知道 Model 的存在
2. Presenter 是中间人，协调 View 和 Model
3. View 是被动的（Passive View），只负责显示
4. 所有逻辑都在 Presenter 中
```

---

## 2. MVP vs MVC 的区别

```
┌─────────────────────────────────────────────────────────────┐
│                      MVC vs MVP                              │
│                                                             │
│    MVC:                          MVP:                       │
│    ┌─────┐ ←────────→ ┌─────┐    ┌─────┐ ←────────→ ┌─────┐│
│    │View │             │Ctrl │    │View │             │Pres ││
│    └──┬──┘             └──┬──┘    └──┬──┘             └──┬──┘│
│       │                   │          ↑                    │   │
│       │    ┌─────┐        │          │    ┌─────┐        │   │
│       └───→│Model│←───────┘          └────│Model│←───────┘   │
│            └─────┘                       └─────┘            │
│                                                             │
│    View 可以直接访问 Model        View 不能直接访问 Model    │
│    Controller 处理输入            Presenter 处理所有逻辑     │
│    View 有一定逻辑                View 是纯被动显示          │
└─────────────────────────────────────────────────────────────┘
```

| 对比项 | MVC | MVP |
|--------|-----|-----|
| View访问Model | ✅ 可以 | ❌ 不可以 |
| View的逻辑 | 有一定逻辑 | 完全被动 |
| 中间层 | Controller | Presenter |
| 测试性 | 中等 | 高（易于Mock） |
| 适用场景 | Web应用 | 桌面/移动应用 |

---

## 3. MVP的两种变体

### Passive View（被动视图）

```
┌─────────────────────────────────────────────────────────────┐
│                    Passive View                              │
│                                                             │
│   View 完全被动：                                            │
│   - 没有任何逻辑                                             │
│   - 只响应 Presenter 的指令更新UI                            │
│   - 所有逻辑都在 Presenter 中                                │
│                                                             │
│   优点：                                                     │
│   - View 极其简单，易于测试                                  │
│   - 逻辑集中在 Presenter                                     │
│                                                             │
│   缺点：                                                     │
│   - Presenter 会变得臃肿                                    │
│   - View 和 Presenter 紧密耦合                              │
└─────────────────────────────────────────────────────────────┘
```

### Supervising Controller（监督控制器）

```
┌─────────────────────────────────────────────────────────────┐
│                 Supervising Controller                       │
│                                                             │
│   View 有简单逻辑：                                          │
│   - 简单的数据绑定由 View 自己处理                           │
│   - 复杂的逻辑由 Presenter 处理                              │
│   - View 可以直接绑定 Model 的数据                           │
│                                                             │
│   优点：                                                     │
│   - Presenter 更轻量                                        │
│   - 简单场景下代码更少                                       │
│                                                             │
│   缺点：                                                     │
│   - View 有了逻辑，测试性降低                                │
│   - 数据绑定机制复杂                                        │
└─────────────────────────────────────────────────────────────┘
```

**Unity中推荐使用 Passive View**，因为：
- Unity的UI系统本身就很复杂
- 数据绑定需要额外实现
- Passive View更容易测试和维护

---

## 4. 完整的MVP实现示例

### 场景：三消游戏的分数系统

```csharp
// ============ 第一步：定义 Model ============

/// <summary>
/// 分数模型 - 纯数据，不知道View和Presenter的存在
/// </summary>
public class ScoreModel
{
    // 数据
    private int currentScore;
    private int highScore;
    private int combo;

    // 属性
    public int CurrentScore => currentScore;
    public int HighScore => highScore;
    public int Combo => combo;

    // 事件 - 通知外部数据变化
    public event Action<ScoreChangedEventArgs> ScoreChanged;

    // 业务逻辑方法
    public void AddScore(int basePoints)
    {
        var oldScore = currentScore;
        combo++;
        currentScore += basePoints * combo;

        if (currentScore > highScore)
        {
            highScore = currentScore;
        }

        // 触发事件
        ScoreChanged?.Invoke(new ScoreChangedEventArgs
        {
            OldScore = oldScore,
            NewScore = currentScore,
            Combo = combo,
            IsNewHighScore = currentScore == highScore
        });
    }

    public void ResetCombo()
    {
        combo = 0;
    }

    public void Reset()
    {
        currentScore = 0;
        combo = 0;
        ScoreChanged?.Invoke(new ScoreChangedEventArgs
        {
            OldScore = 0,
            NewScore = 0,
            Combo = 0,
            IsNewHighScore = false
        });
    }

    public void LoadHighScore(int savedHighScore)
    {
        highScore = savedHighScore;
    }
}

public class ScoreChangedEventArgs
{
    public int OldScore { get; set; }
    public int NewScore { get; set; }
    public int Combo { get; set; }
    public bool IsNewHighScore { get; set; }
}
```

```csharp
// ============ 第二步：定义 View Interface ============

/// <summary>
/// 分数视图接口 - Presenter通过接口与View通信
/// </summary>
public interface IScoreView
{
    // 显示方法 - Presenter调用这些方法更新View
    void DisplayScore(int score);
    void DisplayHighScore(int highScore);
    void DisplayCombo(int combo);
    void PlayScoreAnimation(int delta);
    void PlayHighScoreEffect();
    void ShowNewRecordBadge(bool show);

    // 事件 - View通过这些事件通知Presenter用户操作
    // （分数显示场景下可能没有用户操作事件）
}

/// <summary>
/// 游戏主视图接口 - 包含用户操作事件
/// </summary>
public interface IGameView : IScoreView
{
    // 显示方法
    void ShowGameUI();
    void HideGameUI();
    void ShowPauseMenu();
    void HidePauseMenu();
    void ShowGameOver(int finalScore);
    void ShowVictory(int finalScore, int stars);

    // 用户操作事件
    event Action OnPauseButtonClicked;
    event Action OnResumeButtonClicked;
    event Action OnRestartButtonClicked;
    event Action OnQuitButtonClicked;
    event Action<Vector2Int> OnGemSelected;
    event Action<Vector2Int> OnGemSwapped;
}
```

```csharp
// ============ 第三步：实现 View ============

/// <summary>
/// 游戏视图实现 - 只负责显示和转发用户操作
/// </summary>
public class GameView : MonoBehaviour, IGameView
{
    [Header("UI Components")]
    [SerializeField] private Text scoreText;
    [SerializeField] private Text highScoreText;
    [SerializeField] private Text comboText;
    [SerializeField] private GameObject newRecordBadge;
    [SerializeField] private Animator scoreAnimator;

    [Header("Panels")]
    [SerializeField] private GameObject pausePanel;
    [SerializeField] private GameObject gameOverPanel;
    [SerializeField] private GameObject victoryPanel;

    [Header("Buttons")]
    [SerializeField] private Button pauseButton;
    [SerializeField] private Button resumeButton;
    [SerializeField] private Button restartButton;
    [SerializeField] private Button quitButton;

    // 事件
    public event Action OnPauseButtonClicked;
    public event Action OnResumeButtonClicked;
    public event Action OnRestartButtonClicked;
    public event Action OnQuitButtonClicked;
    public event Action<Vector2Int> OnGemSelected;
    public event Action<Vector2Int> OnGemSwapped;

    private void Awake()
    {
        // 绑定按钮事件
        pauseButton.onClick.AddListener(() => OnPauseButtonClicked?.Invoke());
        resumeButton.onClick.AddListener(() => OnResumeButtonClicked?.Invoke());
        restartButton.onClick.AddListener(() => OnRestartButtonClicked?.Invoke());
        quitButton.onClick.AddListener(() => OnQuitButtonClicked?.Invoke());
    }

    private void Update()
    {
        HandleInput();
    }

    private void HandleInput()
    {
        if (Input.GetMouseButtonDown(0))
        {
            var gridPos = ConvertToGridPosition(Input.mousePosition);
            if (gridPos.HasValue)
            {
                OnGemSelected?.Invoke(gridPos.Value);
            }
        }
    }

    private Vector2Int? ConvertToGridPosition(Vector3 mousePosition)
    {
        // 转换鼠标位置到棋盘坐标
        return null;
    }

    // ============ IScoreView 实现 ============

    public void DisplayScore(int score)
    {
        scoreText.text = $"Score: {score:N0}";
    }

    public void DisplayHighScore(int highScore)
    {
        highScoreText.text = $"Best: {highScore:N0}";
    }

    public void DisplayCombo(int combo)
    {
        if (combo > 1)
        {
            comboText.gameObject.SetActive(true);
            comboText.text = $"x{combo} Combo!";
        }
        else
        {
            comboText.gameObject.SetActive(false);
        }
    }

    public void PlayScoreAnimation(int delta)
    {
        scoreAnimator.SetTrigger("ScoreUp");
    }

    public void PlayHighScoreEffect()
    {
        // 播放新纪录特效
    }

    public void ShowNewRecordBadge(bool show)
    {
        newRecordBadge.SetActive(show);
    }

    // ============ IGameView 实现 ============

    public void ShowGameUI() => gameObject.SetActive(true);
    public void HideGameUI() => gameObject.SetActive(false);
    public void ShowPauseMenu() => pausePanel.SetActive(true);
    public void HidePauseMenu() => pausePanel.SetActive(false);

    public void ShowGameOver(int finalScore)
    {
        gameOverPanel.SetActive(true);
    }

    public void ShowVictory(int finalScore, int stars)
    {
        victoryPanel.SetActive(true);
    }
}
```

```csharp
// ============ 第四步：实现 Presenter ============

/// <summary>
/// 游戏展示器 - 处理所有逻辑，协调Model和View
/// </summary>
public class GamePresenter : MonoBehaviour
{
    private IGameView view;
    private ScoreModel scoreModel;
    private GameModel gameModel;

    private IBoardService boardService;
    private IAudioService audioService;
    private ISaveSystem saveSystem;

    private Vector2Int? selectedGem;
    private bool isAnimating;

    private void Awake()
    {
        view = GetComponent<IGameView>();
        scoreModel = new ScoreModel();
        gameModel = new GameModel();

        boardService = Services.Get<IBoardService>();
        audioService = Services.Get<IAudioService>();
        saveSystem = Services.Get<ISaveSystem>();
    }

    private void OnEnable()
    {
        SubscribeViewEvents();
        scoreModel.ScoreChanged += OnScoreChanged;
    }

    private void OnDisable()
    {
        UnsubscribeViewEvents();
        scoreModel.ScoreChanged -= OnScoreChanged;
    }

    private void Start()
    {
        InitializeGame();
    }

    private void InitializeGame()
    {
        var saveData = saveSystem.Load();
        scoreModel.LoadHighScore(saveData.highScore);

        view.DisplayScore(scoreModel.CurrentScore);
        view.DisplayHighScore(scoreModel.HighScore);
        view.DisplayCombo(0);
        view.ShowGameUI();

        boardService.Initialize(7, 7);
    }

    private void SubscribeViewEvents()
    {
        view.OnPauseButtonClicked += OnPauseButtonClicked;
        view.OnResumeButtonClicked += OnResumeButtonClicked;
        view.OnRestartButtonClicked += OnRestartButtonClicked;
        view.OnQuitButtonClicked += OnQuitButtonClicked;
        view.OnGemSelected += OnGemSelected;
    }

    private void UnsubscribeViewEvents()
    {
        view.OnPauseButtonClicked -= OnPauseButtonClicked;
        view.OnResumeButtonClicked -= OnResumeButtonClicked;
        view.OnRestartButtonClicked -= OnRestartButtonClicked;
        view.OnQuitButtonClicked -= OnQuitButtonClicked;
        view.OnGemSelected -= OnGemSelected;
    }

    // ============ View事件处理 ============

    private void OnPauseButtonClicked()
    {
        gameModel.Pause();
        view.ShowPauseMenu();
        audioService.PauseBGM();
    }

    private void OnResumeButtonClicked()
    {
        gameModel.Resume();
        view.HidePauseMenu();
        audioService.ResumeBGM();
    }

    private void OnRestartButtonClicked()
    {
        view.HidePauseMenu();
        RestartGame();
    }

    private void OnQuitButtonClicked()
    {
        SaveGame();
    }

    private void OnGemSelected(Vector2Int position)
    {
        if (isAnimating) return;

        if (!selectedGem.HasValue)
        {
            selectedGem = position;
        }
        else
        {
            if (IsAdjacent(selectedGem.Value, position))
            {
                TrySwap(selectedGem.Value, position);
            }
            else
            {
                selectedGem = position;
            }
        }
    }

    // ============ 游戏逻辑 ============

    private bool IsAdjacent(Vector2Int a, Vector2Int b)
    {
        var dx = Mathf.Abs(a.x - b.x);
        var dy = Mathf.Abs(a.y - b.y);
        return (dx == 1 && dy == 0) || (dx == 0 && dy == 1);
    }

    private void TrySwap(Vector2Int from, Vector2Int to)
    {
        isAnimating = true;
        selectedGem = null;
        boardService.SwapGems(from, to);
        StartCoroutine(CheckMatchesAfterSwap(from, to));
    }

    private IEnumerator CheckMatchesAfterSwap(Vector2Int from, Vector2Int to)
    {
        yield return new WaitForSeconds(0.3f);

        var matches = boardService.DetectMatches();

        if (matches.Count > 0)
        {
            yield return ProcessMatches(matches);
        }
        else
        {
            boardService.SwapGems(to, from);
            scoreModel.ResetCombo();
        }

        isAnimating = false;
    }

    private IEnumerator ProcessMatches(List<Match> matches)
    {
        foreach (var match in matches)
        {
            int basePoints = match.Count * 100;
            scoreModel.AddScore(basePoints);
            boardService.RemoveGems(match.Positions);
            yield return new WaitForSeconds(0.1f);
        }

        yield return boardService.FillBoard();

        var newMatches = boardService.DetectMatches();
        if (newMatches.Count > 0)
        {
            yield return ProcessMatches(newMatches);
        }
        else
        {
            scoreModel.ResetCombo();
        }
    }

    // ============ Model事件处理 ============

    private void OnScoreChanged(ScoreChangedEventArgs e)
    {
        view.DisplayScore(e.NewScore);
        view.DisplayCombo(e.Combo);
        view.PlayScoreAnimation(e.NewScore - e.OldScore);

        if (e.IsNewHighScore)
        {
            view.ShowNewRecordBadge(true);
            view.PlayHighScoreEffect();
            audioService.PlaySFX("new_record");
        }
        else
        {
            audioService.PlaySFX("score_up");
        }
    }

    private void RestartGame()
    {
        scoreModel.Reset();
        boardService.Initialize(7, 7);
        selectedGem = null;
        isAnimating = false;
    }

    private void SaveGame()
    {
        var saveData = saveSystem.Load();
        saveData.highScore = scoreModel.HighScore;
        saveSystem.Save(saveData);
    }
}
```

---

## 5. MVP的测试优势

```csharp
/// <summary>
/// Mock View 用于测试
/// </summary>
public class MockGameView : IGameView
{
    public int DisplayScoreCallCount { get; private set; }
    public int LastDisplayedScore { get; private set; }
    public int DisplayComboCallCount { get; private set; }
    public int LastDisplayedCombo { get; private set; }

    public event Action OnPauseButtonClicked;
    public event Action OnResumeButtonClicked;
    public event Action<Vector2Int> OnGemSelected;

    public void DisplayScore(int score)
    {
        DisplayScoreCallCount++;
        LastDisplayedScore = score;
    }

    public void DisplayCombo(int combo)
    {
        DisplayComboCallCount++;
        LastDisplayedCombo = combo;
    }

    // 测试辅助方法
    public void SimulatePauseButtonClick() => OnPauseButtonClicked?.Invoke();
    public void SimulateGemSelection(Vector2Int pos) => OnGemSelected?.Invoke(pos);
}

/// <summary>
/// Presenter 单元测试
/// </summary>
[Test]
public void ScoreModel_AddScore_UpdatesView()
{
    // Arrange
    var mockView = new MockGameView();
    var presenter = new GamePresenter();
    presenter.SetView(mockView);

    // Act
    presenter.AddScore(100);

    // Assert
    Assert.AreEqual(1, mockView.DisplayScoreCallCount);
    Assert.AreEqual(100, mockView.LastDisplayedScore);
}

[Test]
public void ScoreModel_Combo_IncreasesMultiplier()
{
    // Arrange
    var mockView = new MockGameView();
    var presenter = new GamePresenter();
    presenter.SetView(mockView);

    // Act
    presenter.AddScore(100);  // 1x combo
    presenter.AddScore(100);  // 2x combo
    presenter.AddScore(100);  // 3x combo

    // Assert: 100 + 200 + 300 = 600
    Assert.AreEqual(600, mockView.LastDisplayedScore);
    Assert.AreEqual(3, mockView.LastDisplayedCombo);
}
```

---

## 6. MVP在Unity中的最佳实践

### Presenter 粒度选择

```
┌─────────────────────────────────────────────────────────────┐
│                    Presenter 粒度选择                        │
│                                                             │
│   方案1：单一大 Presenter                                    │
│   ┌─────────────────────────────────────┐                  │
│   │           GamePresenter              │                  │
│   │  - 处理所有游戏逻辑                  │                  │
│   │  - 管理所有View                      │                  │
│   └─────────────────────────────────────┘                  │
│   优点：简单，适合小项目                                     │
│   缺点：代码臃肿，难以维护                                   │
│                                                             │
│   方案2：多个小 Presenter（推荐）                            │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                  │
│   │ Score    │ │ Board    │ │ Combat   │                  │
│   │ Presenter│ │ Presenter│ │ Presenter│                  │
│   └──────────┘ └──────────┘ └──────────┘                  │
│   优点：职责清晰，易于测试和复用                              │
│   缺点：需要协调多个Presenter                                │
└─────────────────────────────────────────────────────────────┘
```

### 多Presenter架构示例

```csharp
/// <summary>
/// 分数展示器 - 只负责分数相关逻辑
/// </summary>
public class ScorePresenter
{
    private readonly IScoreView view;
    private readonly ScoreModel model;

    public ScorePresenter(IScoreView view, ScoreModel model)
    {
        this.view = view;
        this.model = model;
        model.ScoreChanged += OnScoreChanged;
    }

    private void OnScoreChanged(ScoreChangedEventArgs e)
    {
        view.DisplayScore(e.NewScore);
        view.DisplayCombo(e.Combo);
    }

    public void AddScore(int points) => model.AddScore(points);
    public void ResetCombo() => model.ResetCombo();
}

/// <summary>
/// 棋盘展示器 - 只负责棋盘相关逻辑
/// </summary>
public class BoardPresenter
{
    private readonly IBoardView view;
    private readonly IBoardService boardService;

    public BoardPresenter(IBoardView view, IBoardService boardService)
    {
        this.view = view;
        this.boardService = boardService;
        view.OnGemSelected += OnGemSelected;
    }

    private void OnGemSelected(Vector2Int position)
    {
        // 处理棋子选择逻辑
    }
}

/// <summary>
/// 游戏总展示器 - 协调多个子展示器
/// </summary>
public class GamePresenter : MonoBehaviour
{
    private ScorePresenter scorePresenter;
    private BoardPresenter boardPresenter;
    private CombatPresenter combatPresenter;

    private void Awake()
    {
        var scoreView = GetComponentInChildren<IScoreView>();
        var boardView = GetComponentInChildren<IBoardView>();

        scorePresenter = new ScorePresenter(scoreView, new ScoreModel());
        boardPresenter = new BoardPresenter(boardView, Services.Get<IBoardService>());
    }
}
```

---

## 7. MVP常见陷阱

### ❌ 陷阱1：View中包含业务逻辑

```csharp
// ❌ 错误
public class BadScoreView : MonoBehaviour, IScoreView
{
    public void DisplayScore(int score)
    {
        if (score > 1000)
            text.color = Color.red;  // 逻辑在View中
        else
            text.color = Color.white;
    }
}

// ✅ 正确
public class ScorePresenter
{
    private void OnScoreChanged(ScoreChangedEventArgs e)
    {
        view.DisplayScore(e.NewScore);

        if (e.NewScore > 1000)
            view.SetScoreColor(ScoreColor.High);
        else
            view.SetScoreColor(ScoreColor.Normal);
    }
}
```

### ❌ 陷阱2：View直接访问Model

```csharp
// ❌ 错误
public class BadGameView : MonoBehaviour
{
    private void Start()
    {
        var score = ScoreModel.Instance.CurrentScore;  // View直接访问Model
        scoreText.text = score.ToString();
    }
}
```

### ❌ 陷阱3：Presenter直接操作GameObject

```csharp
// ❌ 错误
public class BadPresenter
{
    private void OnScoreChanged(int score)
    {
        GameObject.Find("ScoreText").GetComponent<Text>().text = score.ToString();
    }
}

// ✅ 正确
public class GoodPresenter
{
    private void OnScoreChanged(int score)
    {
        view.DisplayScore(score);
    }
}
```

---

## 本课小结

### 核心知识点

| 知识点 | 核心要点 |
|--------|----------|
| MVP定义 | Model-View-Presenter，View不知道Model |
| Passive View | View完全被动，所有逻辑在Presenter |
| View Interface | Presenter通过接口与View通信 |
| 测试性 | 易于Mock View进行单元测试 |
| 粒度 | 推荐多个小Presenter而非单一大Presenter |

### MVP适用场景

| 场景 | 是否推荐MVP |
|------|-------------|
| 复杂UI系统 | ✅ 推荐 |
| 需要单元测试 | ✅ 推荐 |
| 团队协作 | ✅ 推荐 |
| 简单原型 | ⚠️ 可选 |
| 单人小项目 | ⚠️ 可选 |

---

## 相关链接

- [[【架构决策】UI架构-MVC vs MVP]]
- [[【架构决策】UI架构-MVP vs MVVM]]
- [MVP Pattern - Martin Fowler](https://martinfowler.com/eaaDev/uiArchs.html)
- [Passive View - Martin Fowler](https://martinfowler.com/eaaDev/PassiveScreen.html)
- [MVP in Unity - Unity Blog](https://blog.unity.com/technology/architecture-patterns-for-gameplay)
