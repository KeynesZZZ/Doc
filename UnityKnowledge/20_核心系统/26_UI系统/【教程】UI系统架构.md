---
title: 【教程】UI系统架构
tags: ["Unity", "游戏系统", "UI", "教程", "架构"]
category: 核心系统/游戏系统
created: "2026-03-05 08:32"
updated: "2026-05-29 00:00"
description: 游戏UI系统架构设计
unity_version: 2021.3+
status: 待验证
validation: Demo验证
related: ["【笔记】UGUI深度解析"]
author: llm
---

# 【教程】UI系统架构

> 第1课 | 游戏系统开发模块

## 文档定位

讲解游戏UI系统的架构设计，包括Canvas分层策略、UI面板管理系统、MVVM模式在UI中的应用、UI动画系统和分辨率适配方案。

**相关文档**：[[【笔记】UGUI深度解析]]

---

## 1. UGUI核心概念

### Canvas渲染模式

```
┌─────────────────────────────────────────────────────────────┐
│                    Canvas 渲染模式                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Screen Space - Overlay                               │   │
│  │ ├── 覆盖在场景最上层                                   │   │
│  │ ├── 不受相机影响                                      │   │
│  │ └── 适合：HUD、固定UI                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Screen Space - Camera                                │   │
│  │ ├── 相对于相机渲染                                    │   │
│  │ ├── 可设置距离（产生透视效果）                         │   │
│  │ └── 适合：需要3D效果的UI                              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ World Space                                          │   │
│  │ ├── 作为3D物体存在于场景中                            │   │
│  │ ├── 可被遮挡、有深度                                  │   │
│  │ └── 适合：血条、对话框、物品标签                       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### UI事件系统

```csharp
using UnityEngine;
using UnityEngine.EventSystems;

/// <summary>
/// UI事件监听器 - 扩展UGUI事件
/// </summary>
public class UIEventListener : MonoBehaviour, IPointerClickHandler, IPointerEnterHandler, IPointerExitHandler
{
    public System.Action<PointerEventData> OnClick;
    public System.Action<PointerEventData> OnEnter;
    public System.Action<PointerEventData> OnExit;

    public static UIEventListener Get(GameObject go)
    {
        var listener = go.GetComponent<UIEventListener>();
        if (listener == null) listener = go.AddComponent<UIEventListener>();
        return listener;
    }

    public void OnPointerClick(PointerEventData eventData) => OnClick?.Invoke(eventData);
    public void OnPointerEnter(PointerEventData eventData) => OnEnter?.Invoke(eventData);
    public void OnPointerExit(PointerEventData eventData) => OnExit?.Invoke(eventData);
}

// 使用示例
public class ExampleUsage : MonoBehaviour
{
    public Button button;

    private void Start()
    {
        UIEventListener.Get(button.gameObject).OnClick += (data) =>
        {
            Debug.Log($"点击了按钮: {data.pointerCurrentRaycast.gameObject.name}");
        };
    }
}
```

---

## 2. UI管理器设计

### 基础UI管理器

```csharp
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// UI层级
/// </summary>
public enum UILayer
{
    Background = 0,   // 背景层
    Game = 100,       // 游戏主界面
    Popup = 200,      // 弹窗
    Top = 300,        // 顶层UI（提示、Toast）
    System = 400      // 系统UI（加载、网络）
}

/// <summary>
/// UI基类
/// </summary>
public abstract class UIScreen : MonoBehaviour
{
    public UILayer Layer = UILayer.Game;
    public bool IsOpen { get; protected set; }

    public virtual void Open(object param = null)
    {
        IsOpen = true;
        gameObject.SetActive(true);
        OnOpen(param);
    }

    public virtual void Close()
    {
        IsOpen = false;
        OnClose();
        gameObject.SetActive(false);
    }

    protected virtual void OnOpen(object param) { }
    protected virtual void OnClose() { }
}

/// <summary>
/// UI管理器 - 管理所有UI界面
/// </summary>
public class UIManager : MonoSingleton<UIManager>
{
    [Header("Canvas Settings")]
    [SerializeField] private Canvas mainCanvas;
    [SerializeField] private Transform[] layerParents;

    private Dictionary<string, UIScreen> screens = new Dictionary<string, UIScreen>();
    private Stack<UIScreen> screenStack = new Stack<UIScreen>();

    private void Awake()
    {
        InitializeLayers();
    }

    private void InitializeLayers()
    {
        // 为每个层级创建父物体
        layerParents = new Transform[System.Enum.GetValues(typeof(UILayer)).Length];

        foreach (UILayer layer in System.Enum.GetValues(typeof(UILayer)))
        {
            var go = new GameObject($"Layer_{layer}");
            go.transform.SetParent(mainCanvas.transform, false);
            go.AddComponent<Canvas>();

            var rectTransform = go.GetComponent<RectTransform>();
            rectTransform.anchorMin = Vector2.zero;
            rectTransform.anchorMax = Vector2.one;
            rectTransform.sizeDelta = Vector2.zero;
            rectTransform.anchoredPosition = Vector2.zero;

            var canvas = go.GetComponent<Canvas>();
            canvas.overrideSorting = true;
            canvas.sortingOrder = (int)layer;

            layerParents[(int)layer / 100] = go.transform;
        }
    }

    /// <summary>
    /// 注册UI界面
    /// </summary>
    public void RegisterScreen<T>(T screen) where T : UIScreen
    {
        string key = typeof(T).Name;
        if (screens.ContainsKey(key))
        {
            Debug.LogWarning($"Screen {key} already registered");
            return;
        }

        screens[key] = screen;
        screen.transform.SetParent(layerParents[(int)screen.Layer / 100], false);
    }

    /// <summary>
    /// 获取UI界面
    /// </summary>
    public T GetScreen<T>() where T : UIScreen
    {
        string key = typeof(T).Name;
        if (screens.TryGetValue(key, out var screen))
            return screen as T;
        return null;
    }

    /// <summary>
    /// 打开UI界面
    /// </summary>
    public T OpenScreen<T>(object param = null) where T : UIScreen
    {
        var screen = GetScreen<T>();
        if (screen != null)
        {
            screen.Open(param);
            screenStack.Push(screen);
        }
        return screen;
    }

    /// <summary>
    /// 关闭当前界面
    /// </summary>
    public void CloseCurrent()
    {
        if (screenStack.Count > 0)
        {
            var screen = screenStack.Pop();
            screen.Close();
        }
    }

    /// <summary>
    /// 关闭所有界面
    /// </summary>
    public void CloseAll()
    {
        foreach (var screen in screens.Values)
        {
            if (screen.IsOpen)
                screen.Close();
        }
        screenStack.Clear();
    }

    /// <summary>
    /// 返回上一个界面
    /// </summary>
    public void Back()
    {
        if (screenStack.Count > 1)
        {
            var current = screenStack.Pop();
            current.Close();

            var previous = screenStack.Peek();
            previous.Open();
        }
    }
}
```

---

## 3. MVP模式在UI中的应用

### MVP架构

```
┌─────────────────────────────────────────────────────────────┐
│                      MVP 架构模式                             │
│                                                             │
│   ┌─────────────┐         ┌─────────────┐                  │
│   │    View     │ ←─────→ │  Presenter  │                  │
│   │  (UI显示)   │         │  (逻辑控制)  │                  │
│   └─────────────┘         └──────┬──────┘                  │
│         ↑                        │                          │
│         │                        ↓                          │
│         │                ┌─────────────┐                    │
│         └─────────────── │    Model    │                    │
│                          │  (数据模型)  │                    │
│                          └─────────────┘                    │
└─────────────────────────────────────────────────────────────┘

View: 负责UI显示和用户交互
Presenter: 处理业务逻辑，协调View和Model
Model: 数据模型，不依赖UI
```

### 实战示例：背包界面

```csharp
// ========== Model: 物品数据模型 ==========

[System.Serializable]
public class ItemData
{
    public int Id;
    public string Name;
    public string Description;
    public int Count;
    public Sprite Icon;
}

/// <summary>
/// 背包数据模型
/// </summary>
public class InventoryModel
{
    private List<ItemData> items = new List<ItemData>();

    public IReadOnlyList<ItemData> Items => items;

    public event System.Action<ItemData> OnItemAdded;
    public event System.Action<int> OnItemRemoved;
    public event System.Action OnChanged;

    public void AddItem(ItemData item)
    {
        items.Add(item);
        OnItemAdded?.Invoke(item);
        OnChanged?.Invoke();
    }

    public void RemoveItem(int itemId)
    {
        var item = items.Find(i => i.Id == itemId);
        if (item != null)
        {
            items.Remove(item);
            OnItemRemoved?.Invoke(itemId);
            OnChanged?.Invoke();
        }
    }

    public ItemData GetItem(int index)
    {
        return index >= 0 && index < items.Count ? items[index] : null;
    }
}

// ========== View: 背包视图 ==========

/// <summary>
/// 物品格子视图
/// </summary>
public class ItemSlotView : MonoBehaviour
{
    [SerializeField] private Image iconImage;
    [SerializeField] private Text countText;
    [SerializeField] private Button button;

    public int Index { get; private set; }
    public event System.Action<int> OnClicked;

    public void Initialize(int index)
    {
        Index = index;
        button.onClick.AddListener(() => OnClicked?.Invoke(Index));
    }

    public void UpdateView(ItemData item)
    {
        if (item != null)
        {
            iconImage.sprite = item.Icon;
            iconImage.enabled = true;
            countText.text = item.Count > 1 ? item.Count.ToString() : "";
        }
        else
        {
            iconImage.enabled = false;
            countText.text = "";
        }
    }

    public void SetHighlight(bool active)
    {
        // 高亮效果
    }
}

/// <summary>
/// 背包界面视图
/// </summary>
public class InventoryView : UIScreen
{
    [Header("UI Components")]
    [SerializeField] private Transform gridParent;
    [SerializeField] private GameObject slotPrefab;
    [SerializeField] private Button closeButton;

    [Header("Details Panel")]
    [SerializeField] private GameObject detailsPanel;
    [SerializeField] private Image detailIcon;
    [SerializeField] private Text detailName;
    [SerializeField] private Text detailDesc;
    [SerializeField] private Button useButton;

    private List<ItemSlotView> slots = new List<ItemSlotView>();

    public event System.Action<int> OnSlotClicked;
    public event System.Action OnCloseClicked;
    public event System.Action OnUseClicked;

    public override void Open(object param = null)
    {
        base.Open(param);
        InitializeSlots((int)param);
    }

    private void InitializeSlots(int slotCount)
    {
        // 清理旧的
        foreach (var slot in slots)
            Destroy(slot.gameObject);
        slots.Clear();

        // 创建新的
        for (int i = 0; i < slotCount; i++)
        {
            var go = Instantiate(slotPrefab, gridParent);
            var slot = go.GetComponent<ItemSlotView>();
            slot.Initialize(i);
            slot.OnClicked += (index) => OnSlotClicked?.Invoke(index);
            slots.Add(slot);
        }
    }

    public void UpdateSlot(int index, ItemData item)
    {
        if (index >= 0 && index < slots.Count)
            slots[index].UpdateView(item);
    }

    public void UpdateAllSlots(List<ItemData> items)
    {
        for (int i = 0; i < slots.Count; i++)
        {
            var item = i < items.Count ? items[i] : null;
            slots[i].UpdateView(item);
        }
    }

    public void ShowItemDetails(ItemData item)
    {
        detailsPanel.SetActive(true);
        detailIcon.sprite = item.Icon;
        detailName.text = item.Name;
        detailDesc.text = item.Description;
    }

    public void HideItemDetails()
    {
        detailsPanel.SetActive(false);
    }

    private void OnEnable()
    {
        closeButton.onClick.AddListener(() => OnCloseClicked?.Invoke());
        useButton.onClick.AddListener(() => OnUseClicked?.Invoke());
    }

    private void OnDisable()
    {
        closeButton.onClick.RemoveAllListeners();
        useButton.onClick.RemoveAllListeners();
    }
}

// ========== Presenter: 背包逻辑控制 ==========

/// <summary>
/// 背包界面Presenter
/// </summary>
public class InventoryPresenter
{
    private InventoryModel model;
    private InventoryView view;

    private int selectedSlotIndex = -1;

    public InventoryPresenter(InventoryView view, InventoryModel model)
    {
        this.view = view;
        this.model = model;

        BindEvents();
    }

    private void BindEvents()
    {
        view.OnSlotClicked += OnSlotClicked;
        view.OnCloseClicked += OnCloseClicked;
        view.OnUseClicked += OnUseClicked;

        model.OnItemAdded += OnItemAdded;
        model.OnItemRemoved += OnItemRemoved;
    }

    public void Initialize()
    {
        view.Open(20); // 20个格子
        RefreshView();
    }

    private void RefreshView()
    {
        view.UpdateAllSlots(model.Items.ToList());
    }

    private void OnSlotClicked(int index)
    {
        selectedSlotIndex = index;
        var item = model.GetItem(index);

        if (item != null)
        {
            view.ShowItemDetails(item);
        }
        else
        {
            view.HideItemDetails();
        }

        // 高亮选中格子
        for (int i = 0; i < 20; i++)
        {
            // view.SetSlotHighlight(i, i == index);
        }
    }

    private void OnCloseClicked()
    {
        view.Close();
    }

    private void OnUseClicked()
    {
        if (selectedSlotIndex >= 0)
        {
            var item = model.GetItem(selectedSlotIndex);
            if (item != null)
            {
                // 使用物品逻辑
                Debug.Log($"使用物品: {item.Name}");
            }
        }
    }

    private void OnItemAdded(ItemData item)
    {
        RefreshView();
    }

    private void OnItemRemoved(int itemId)
    {
        RefreshView();
        view.HideItemDetails();
    }

    public void Dispose()
    {
        view.OnSlotClicked -= OnSlotClicked;
        view.OnCloseClicked -= OnCloseClicked;
        view.OnUseClicked -= OnUseClicked;

        model.OnItemAdded -= OnItemAdded;
        model.OnItemRemoved -= OnItemRemoved;
    }
}

// ========== 使用示例 ==========

public class GameUIManager : MonoBehaviour
{
    [SerializeField] private InventoryView inventoryView;

    private InventoryModel inventoryModel;
    private InventoryPresenter inventoryPresenter;

    private void Start()
    {
        // 创建Model
        inventoryModel = new InventoryModel();

        // 创建Presenter，连接View和Model
        inventoryPresenter = new InventoryPresenter(inventoryView, inventoryModel);

        // 初始化
        inventoryPresenter.Initialize();
    }

    private void OnDestroy()
    {
        inventoryPresenter?.Dispose();
    }
}
```

---

## 4. UI优化策略

### 4.1 图集优化

```
┌─────────────────────────────────────────────────────────────┐
│                      图集优化策略                             │
│                                                             │
│  1. 打包规则：                                              │
│     ├── 同一界面使用的图片放在同一图集                       │
│     ├── 小图（< 32x32）单独处理                              │
│     └── 动态图集用于运行时加载                               │
│                                                             │
│  2. 图集大小：                                              │
│     ├── 移动端：1024x1024 或 2048x2048                      │
│     ├── PC端：最大 4096x4096                                │
│     └── 2的幂次方                                           │
│                                                             │
│  3. 格式选择：                                              │
│     ├── ASTC (iOS/Android推荐)                              │
│     ├── ETC2 (Android备用)                                  │
│     └── RGBA32 (高质量需求)                                 │
│                                                             │
│  4. Sprite Atlas配置：                                      │
│     ├── Include in Build: 是                                │
│     ├── Allow Rotation: 否（避免旋转问题）                   │
│     └── Tight Packing: 是（节省空间）                        │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 DrawCall优化

```csharp
/// <summary>
/// UI DrawCall优化工具
/// </summary>
public static class UIDrawCallOptimizer
{
    /// <summary>
    /// 批量设置UI元素的Raycast Target
    /// </summary>
    public static void SetRaycastTargetRecursive(Transform parent, bool enable)
    {
        foreach (Transform child in parent)
        {
            var graphic = child.GetComponent<Graphic>();
            if (graphic != null)
            {
                // 只对需要交互的UI开启Raycast Target
                graphic.raycastTarget = enable;
            }
            SetRaycastTargetRecursive(child, enable);
        }
    }

    /// <summary>
    /// 优化UI层级，减少DrawCall
    /// </summary>
    public static void OptimizeHierarchy(Transform parent)
    {
        // 按材质/图集排序子物体
        var children = new List<Transform>();
        foreach (Transform child in parent)
            children.Add(child);

        children.Sort((a, b) =>
        {
            var graphicA = a.GetComponent<Graphic>();
            var graphicB = b.GetComponent<Graphic>();

            if (graphicA == null || graphicB == null) return 0;

            // 按材质排序
            return graphicA.material.GetInstanceID().CompareTo(
                   graphicB.material.GetInstanceID());
        });

        for (int i = 0; i < children.Count; i++)
            children[i].SetSiblingIndex(i);
    }

    /// <summary>
    /// 检查DrawCall问题
    /// </summary>
    public static void CheckDrawCallIssues(GameObject root)
    {
        var graphics = root.GetComponentsInChildren<Graphic>(true);
        int currentAtlas = -1;
        int breakCount = 0;

        foreach (var graphic in graphics)
        {
            if (!graphic.gameObject.activeInHierarchy) continue;

            var sprite = (graphic as Image)?.sprite;
            if (sprite != null)
            {
                int atlasId = sprite.texture.GetInstanceID();
                if (currentAtlas != -1 && currentAtlas != atlasId)
                {
                    breakCount++;
                    Debug.LogWarning($"DrawCall中断: {graphic.name}", graphic);
                }
                currentAtlas = atlasId;
            }
        }

        Debug.Log($"总计DrawCall中断次数: {breakCount}");
    }
}
```

### 4.3 UI对象池

```csharp
using UnityEngine.Pool;

/// <summary>
/// UI对象池 - 优化频繁创建/销毁
/// </summary>
public class UIObjectPool : MonoSingleton<UIObjectPool>
{
    private Dictionary<string, ObjectPool<GameObject>> pools = new Dictionary<string, ObjectPool<GameObject>>();
    private Dictionary<GameObject, string> poolKeys = new Dictionary<GameObject, string>();

    /// <summary>
    /// 初始化对象池
    /// </summary>
    public void InitializePool(string key, GameObject prefab, int defaultCapacity = 10, int maxSize = 50)
    {
        if (pools.ContainsKey(key)) return;

        var pool = new ObjectPool<GameObject>(
            () =>
            {
                var go = Instantiate(prefab);
                poolKeys[go] = key;
                return go;
            },
            go => go.SetActive(true),
            go => go.SetActive(false),
            go => Destroy(go),
            true, defaultCapacity, maxSize
        );

        pools[key] = pool;
    }

    /// <summary>
    /// 获取对象
    /// </summary>
    public GameObject Get(string key, Transform parent = null)
    {
        if (!pools.TryGetValue(key, out var pool))
        {
            Debug.LogError($"Pool {key} not initialized");
            return null;
        }

        var go = pool.Get();
        if (parent != null)
            go.transform.SetParent(parent, false);
        return go;
    }

    /// <summary>
    /// 归还对象
    /// </summary>
    public void Release(GameObject go)
    {
        if (!poolKeys.TryGetValue(go, out var key))
        {
            Destroy(go);
            return;
        }

        if (pools.TryGetValue(key, out var pool))
            pool.Release(go);
        else
            Destroy(go);
    }

    /// <summary>
    /// 清空指定池
    /// </summary>
    public void ClearPool(string key)
    {
        if (pools.TryGetValue(key, out var pool))
        {
            pool.Clear();
            pools.Remove(key);
        }
    }

    /// <summary>
    /// 清空所有池
    /// </summary>
    public void ClearAll()
    {
        foreach (var pool in pools.Values)
            pool.Clear();
        pools.Clear();
        poolKeys.Clear();
    }
}

// 使用示例：列表项复用
public class UIListView : MonoBehaviour
{
    [SerializeField] private GameObject itemPrefab;
    [SerializeField] private Transform contentParent;

    private List<GameObject> activeItems = new List<GameObject>();
    private const string POOL_KEY = "ListItem";

    private void Awake()
    {
        UIObjectPool.Instance.InitializePool(POOL_KEY, itemPrefab);
    }

    public void SetData(int count)
    {
        // 归还旧的
        foreach (var item in activeItems)
            UIObjectPool.Instance.Release(item);
        activeItems.Clear();

        // 创建新的
        for (int i = 0; i < count; i++)
        {
            var item = UIObjectPool.Instance.Get(POOL_KEY, contentParent);
            activeItems.Add(item);
        }
    }
}
```

---

## 5. 三消游戏UI架构

### 完整的UI层级设计

```
┌─────────────────────────────────────────────────────────────┐
│                   三消游戏UI层级                              │
│                                                             │
│  Layer_System (400)                                         │
│  ├── LoadingScreen        加载界面                          │
│  ├── NetworkIndicator     网络状态                          │
│  └── Toast                提示信息                          │
│                                                             │
│  Layer_Top (300)                                            │
│  ├── EnergyBar            体力条                            │
│  ├── CoinDisplay          金币显示                          │
│  └── SettingsPopup        设置弹窗                          │
│                                                             │
│  Layer_Popup (200)                                          │
│  ├── ResultPopup          结算弹窗                          │
│  ├── RewardPopup          奖励弹窗                          │
│  └── PausePopup           暂停弹窗                          │
│                                                             │
│  Layer_Game (100)                                           │
│  ├── GameHUD               游戏主界面                       │
│  │   ├── ScoreDisplay      分数显示                         │
│  │   ├── ComboDisplay      连击显示                         │
│  │   ├── TurnIndicator     回合指示                         │
│  │   └── SkillBar          技能栏                           │
│  ├── BoardUI              棋盘UI层                          │
│  └── HeroStatus           英雄状态                          │
│                                                             │
│  Layer_Background (0)                                       │
│  └── MainMenu             主菜单背景                        │
└─────────────────────────────────────────────────────────────┘
```

### 游戏HUD界面

```csharp
/// <summary>
/// 游戏HUD界面
/// </summary>
public class GameHUD : UIScreen
{
    [Header("Score")]
    [SerializeField] private Text scoreText;
    [SerializeField] private Animator scoreAnimator;

    [Header("Combo")]
    [SerializeField] private Text comboText;
    [SerializeField] private GameObject comboPanel;
    [SerializeField] private Animator comboAnimator;

    [Header("Turn")]
    [SerializeField] private Text turnText;
    [SerializeField] private Image turnFill;

    [Header("Skills")]
    [SerializeField] private SkillSlotUI[] skillSlots;

    [Header("Buttons")]
    [SerializeField] private Button pauseButton;
    [SerializeField] private Button hintButton;

    private int currentScore;
    private int currentCombo;

    protected override void OnOpen(object param)
    {
        // 绑定事件
        pauseButton.onClick.AddListener(OnPauseClicked);
        hintButton.onClick.AddListener(OnHintClicked);

        // 订阅游戏事件
        EventBus.Subscribe<ScoreChangedEvent>(OnScoreChanged);
        EventBus.Subscribe<ComboChangedEvent>(OnComboChanged);
        EventBus.Subscribe<TurnChangedEvent>(OnTurnChanged);
    }

    protected override void OnClose()
    {
        pauseButton.onClick.RemoveListener(OnPauseClicked);
        hintButton.onClick.RemoveListener(OnHintClicked);

        EventBus.Unsubscribe<ScoreChangedEvent>(OnScoreChanged);
        EventBus.Unsubscribe<ComboChangedEvent>(OnComboChanged);
        EventBus.Unsubscribe<TurnChangedEvent>(OnTurnChanged);
    }

    private void OnScoreChanged(ScoreChangedEvent e)
    {
        currentScore = e.Score;
        scoreText.text = e.Score.ToString("N0");
        scoreAnimator.SetTrigger("Pulse");
    }

    private void OnComboChanged(ComboChangedEvent e)
    {
        if (e.Combo > 1)
        {
            comboPanel.SetActive(true);
            comboText.text = $"{e.Combo}x Combo!";
            comboAnimator.SetInteger("Combo", e.Combo);
        }
        else
        {
            comboPanel.SetActive(false);
        }
    }

    private void OnTurnChanged(TurnChangedEvent e)
    {
        turnText.text = $"Turn {e.CurrentTurn}/{e.MaxTurns}";
        turnFill.fillAmount = (float)e.CurrentTurn / e.MaxTurns;
    }

    private void OnPauseClicked()
    {
        UIManager.Instance.OpenScreen<PausePopup>();
    }

    private void OnHintClicked()
    {
        EventBus.Publish(new HintRequestedEvent());
    }
}

// 事件定义
public struct ScoreChangedEvent { public int Score; }
public struct ComboChangedEvent { public int Combo; }
public struct TurnChangedEvent { public int CurrentTurn; public int MaxTurns; }
public struct HintRequestedEvent { }
```

---

## 本课小结

### 核心知识点

| 知识点 | 核心要点 |
|--------|----------|
| Canvas渲染模式 | Overlay/Camera/World Space |
| UI事件系统 | IPointerClickHandler等接口 |
| UI管理器 | 层级管理、界面栈、统一入口 |
| MVP模式 | Model-View-Presenter分离 |
| 图集优化 | 打包规则、格式选择 |
| DrawCall优化 | 层级排序、Raycast Target关闭 |
| UI对象池 | 频繁创建销毁的UI元素复用 |

### UI架构最佳实践

```
1. 分层管理：按功能划分UI层级
2. MVP模式：逻辑与视图分离
3. 事件驱动：使用EventBus解耦
4. 对象池：列表项复用
5. 图集打包：减少DrawCall
6. Raycast Target：按需开启
```

---

## 相关链接

- [Unity UGUI优化指南](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity7.html)
- [Unity UI Extensions](https://github.com/Unity-UI-Extensions/com.unity.ui.extensions)
- [MVP Pattern](https://en.wikipedia.org/wiki/Model–view–presenter)
