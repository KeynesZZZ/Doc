---
title: 【架构决策】UI 架构 - MVP vs MVVM
tags: ["C#", "Unity", "架构", "UI", "MVP", "MVVM", "架构决策", "界面架构"]
category: 架构设计/架构决策
created: "2024-01-22 10:00"
updated: "2026-05-29 00:00"
description: Unity UI开发中两种主流架构模式的对比与选择，包含实现示例和最佳实践
unity_version: 2021.3+
status: 待验证
validation: Demo验证
related: ["[[【教程】MVP模式深入讲解]]", "[[【教程】项目架构设计]]", "[[【教程】依赖注入与服务定位器]]"]
author: llm
---

# 【架构决策】UI 架构 - MVP vs MVVM

> 核心问题：Unity UI开发应该选择 MVP 还是 MVVM？

## 文档定位

本文档从**方案对比角度**分析Unity UI开发中MVP和MVVM两种主流架构模式的优缺点、实现复杂度和适用场景，帮助团队做出架构选型决策。

**相关文档**：[[【架构决策】UI架构-MVC vs MVP]]、[[【教程】MVP模式深入讲解]]、[[【教程】项目架构设计]]

---

## 概述

MVP 和 MVVM 是UI开发中最常用的架构模式，本篇对比它们在Unity中的实现。

## MVP 模式

### 架构图

```
┌─────────────────────────────────────┐
│              View                    │
│  ┌─────────────────────────────┐    │
│  │   UI Components (UGUI)      │    │
│  │   - Button, Text, Image     │    │
│  └─────────────────────────────┘    │
│              ↑↓                      │
│  ┌─────────────────────────────┐    │
│  │   IView Interface           │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
              ↑↓
┌─────────────────────────────────────┐
│           Presenter                  │
│  - 处理用户输入                      │
│  - 调用Model                         │
│  - 更新View                          │
└─────────────────────────────────────┘
              ↑↓
┌─────────────────────────────────────┐
│             Model                    │
│  - 数据逻辑                          │
│  - 业务规则                          │
│  - 状态管理                          │
└─────────────────────────────────────┘
```

### 实现示例

```csharp
// IView - 接口定义
public interface IMainMenuView
{
    void SetScore(int score);
    void Show();
    void Hide();
    event Action OnPlayClicked;
    event Action OnSettingsClicked;
}

// View - MonoBehaviour实现
public class MainMenuView : MonoBehaviour, IMainMenuView
{
    [SerializeField] private Text scoreText;
    [SerializeField] private Button playButton;
    [SerializeField] private Button settingsButton;

    public event Action OnPlayClicked;
    public event Action OnSettingsClicked;

    private void Awake()
    {
        playButton.onClick.AddListener(() => OnPlayClicked?.Invoke());
        settingsButton.onClick.AddListener(() => OnSettingsClicked?.Invoke());
    }

    public void SetScore(int score) => scoreText.text = score.ToString();
    public void Show() => gameObject.SetActive(true);
    public void Hide() => gameObject.SetActive(false);
}

// Presenter - 逻辑处理
public class MainMenuPresenter
{
    private readonly IMainMenuView view;
    private readonly GameModel model;

    public MainMenuPresenter(IMainMenuView view, GameModel model)
    {
        this.view = view;
        this.model = model;

        view.OnPlayClicked += HandlePlayClicked;
        view.OnSettingsClicked += HandleSettingsClicked;

        UpdateView();
    }

    private void HandlePlayClicked()
    {
        // 处理开始游戏逻辑
        view.Hide();
    }

    private void UpdateView()
    {
        view.SetScore(model.Score);
    }
}
```

## MVVM 模式

### 架构图

```
┌─────────────────────────────────────┐
│              View                    │
│  - XAML/UI绑定                       │
│  - 数据绑定到ViewModel               │
└─────────────────────────────────────┘
              ↕ (数据绑定)
┌─────────────────────────────────────┐
│           ViewModel                  │
│  - Observable属性                    │
│  - ICommand                          │
│  - 无View引用                        │
└─────────────────────────────────────┘
              ↕
┌─────────────────────────────────────┐
│             Model                    │
└─────────────────────────────────────┘
```

### 实现示例 (使用UniRx)

```csharp
// ViewModel
public class MainMenuViewModel : IDisposable
{
    // Observable属性
    public ReactiveProperty<int> Score { get; } = new();
    public ReactiveCommand PlayCommand { get; } = new();
    public ReactiveCommand SettingsCommand { get; } = new();

    private readonly GameModel model;

    public MainMenuViewModel(GameModel model)
    {
        this.model = model;

        // 绑定Model到ViewModel
        model.OnScoreChanged += score => Score.Value = score;

        // 绑定命令
        PlayCommand.Subscribe(_ => HandlePlay());
    }

    private void HandlePlay()
    {
        // 处理开始游戏
    }

    public void Dispose()
    {
        Score?.Dispose();
        PlayCommand?.Dispose();
        SettingsCommand?.Dispose();
    }
}

// View - 绑定到ViewModel
public class MainMenuView : MonoBehaviour
{
    [SerializeField] private Text scoreText;
    [SerializeField] private Button playButton;

    private MainMenuViewModel viewModel;
    private CompositeDisposable disposables = new();

    public void Bind(MainMenuViewModel vm)
    {
        viewModel = vm;

        // 数据绑定
        vm.Score
            .SubscribeToText(scoreText)
            .AddTo(disposables);

        // 命令绑定
        playButton
            .OnClickAsObservable()
            .BindTo(vm.PlayCommand)
            .AddTo(disposables);
    }

    private void OnDestroy()
    {
        disposables.Dispose();
    }
}
```

## 对比分析

| 维度 | MVP | MVVM |
|------|-----|------|
| **学习曲线** | 较低 | 较高 |
| **代码量** | 较多 | 较少 |
| **测试性** | 优秀 | 优秀 |
| **绑定方式** | 手动 | 自动(需框架) |
| **Unity适配** | 原生支持 | 需UniRx/MVVM框架 |
| **调试难度** | 简单 | 中等 |

## 选择指南

### 选择 MVP 当:

- 团队不熟悉响应式编程
- 需要精确控制UI更新时机
- 项目规模中等
- 没有MVVM框架支持

### 选择 MVVM 当:

- 团队熟悉UniRx/响应式编程
- 需要大量UI与数据绑定
- 项目UI复杂度高
- 有现成MVVM框架

## 混合方案

```csharp
// 在MVP中引入响应式特性
public class HybridPresenter
{
    private readonly IView view;
    private readonly CompositeDisposable disposables = new();

    public HybridPresenter(IView view, ViewModel vm)
    {
        this.view = view;

        // 响应式绑定
        vm.Score
            .Subscribe(score => view.SetScore(score))
            .AddTo(disposables);
    }
}
```

## 相关链接

- [[【架构决策】UI架构-MVC vs MVP]]
- [[【教程】MVP模式深入讲解]]
- [UI系统架构](../20_核心系统/26_UI系统/【教程】UI系统架构.md)
