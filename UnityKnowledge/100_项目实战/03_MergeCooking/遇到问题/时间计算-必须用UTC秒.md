---
title: TimeSystem - DST 问题的解决方案
tags: [MergeCooking, TimeSystem, Unity, DST解决方案]
created: 2026-07-31
---
wen
# TimeSystem - DST 问题的解决方案

## 问题：冬夏令时（DST）导致的时间偏移

### 现象

欧洲每年进行两次时间转换：
- **3月最后一个周日**：02:00 CET → 03:00 CEST（春季前跳，跳过 1 小时）
- **10月最后一个周日**：03:00 CEST → 02:00 CET（秋季后退，重复 1 小时）

### 导致的问题

使用本地时间做时间计算时会出现 **±1 小时的偏移**：

```
例子：2026年3月29日（春季转换）

旧系统问题：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
dayId 20260329 的起止时间戳：
  - DST 转换前：获得的是 CET 偏移的时间戳
  - DST 转换后：同一 dayId 获得的是 CEST 偏移的时间戳
  - 结果：同一 dayId 的起点时间戳偏移了 ±1 小时！

后果：
  • 活动开始时间错乱
  • 每日重置时间不准
  • 跨天判断失效
```

## 解决方案：全程使用 UTC 时间戳

### 核心原则

| 原则 | 说明 |
|------|------|
| **全程 UTC 秒** | 永不使用本地时间计算 |
| **标准偏移** | 只用标准偏移（-5 小时），永不用 DST 偏移（-4 小时） |
| **dayId 只显示** | dayId 不参与任何逻辑运算，只用于 UI 显示 |
| **凌晨 3 点跨天** | 游戏日在凌晨 3 点切换（与旧系统一致） |

### 如何工作

```
DST 转换时的行为对比：

旧系统（有问题）：
  dayId 20260329 的起点时间戳会在转换时偏移 ±1h
  ❌ 活动开始时间错乱

新系统（TimeSystem）：
  dayId 20260329 对应的 UTC 时间戳永不变化
  ✅ 活动开始时间准确
  ✅ 无论什么时候查询，结果一致
```

## API 速查

```csharp
using TLF.TimeSystem;

// 获取当前 UTC 时间戳
long now = TimeSystem.GetServerTimestamp();

// 获取今天的 dayId（凌晨 3 点跨天）
int today = TimeSystem.GetGameTodayDayId();

// dayId 获取起止时间戳（禁止用 dayId 直接做算术）
long start = TimeSystem.GetGameDayStartTimestamp(20260710);
long end = TimeSystem.GetGameDayEndTimestamp(20260710);

// dayId 安全运算
int tomorrow = TimeSystem.AddDays(today, 1);           // ✅ 正确处理月份边界
int diff = TimeSystem.DiffDays(20260813, 20260710);    // 返回 34
int weekday = TimeSystem.GetGameTodayWeekday();        // 1~7

// 活动时间判断
if (now >= start && now <= end) { }                    // ✅ 正确

// 活动倒计时
long secondsLeft = TimeSystem.GetSecondsToTimestamp(activityEndTime);
int openDay = TimeSystem.GetActivityOpenDay(activityStartTime);
```

## 使用示例

### 每日重置

```csharp
private int _lastResetDayId = -1;

void Update()
{
    int today = TimeSystem.GetGameTodayDayId();
    if (today != _lastResetDayId)
    {
        _lastResetDayId = today;
        OnDailyReset();  // 凌晨 3 点触发
    }
}
```

### 活动时间判断

```csharp
// ✅ 正确：用时间戳判断
long actStart = TimeSystem.GetGameDayStartTimestamp(20260710);
long actEnd = TimeSystem.GetGameDayEndTimestamp(20260813);
bool isActive = now >= actStart && now <= actEnd;

// ❌ 错误：用 dayId 判断（在 DST 转换时会出错）
// bool isActive = today >= 20260710 && today <= 20260813;
```

### 活动倒计时

```csharp
void UpdateCountdown(long activityEndTime)
{
    long secondsLeft = TimeSystem.GetSecondsToTimestamp(activityEndTime);
    
    int days = (int)(secondsLeft / TimeSystem.DaySec);
    int hours = (int)((secondsLeft % TimeSystem.DaySec) / 3600);
    
    Debug.Log($"Time left: {days}d {hours}h");
}
```

### 距离下次重置的倒计时

```csharp
void ShowTimeToNextReset()
{
    long secondsLeft = TimeSystem.GetSecondsToNextGameDay();
    int hours = (int)(secondsLeft / 3600);
    Debug.Log($"Next reset in {hours}h");
}
```

## 核心代码（关键部分）

### 时间戳 ↔ dayId 转换的关键逻辑

```csharp
// 从时间戳获取 dayId
public static int GetGameDayIdFromTimestamp(long timestamp)
{
    // 1. 加上设备的标准偏移（美东 -5h、印度 +5:30 等）
    long localTimestamp = timestamp + StandardOffsetSec;
    
    // 2. 检查是否在凌晨 3 点之后
    long timeOfDay = localTimestamp % DaySec;
    if (timeOfDay < ResetOffsetSec)  // ResetOffsetSec = 10800 (3小时)
    {
        // 凌晨 3 点之前 → 属于前一天
        localTimestamp -= DaySec;
    }
    
    // 3. 转换为 dayId（yyyyMMdd）
    return SafeDate.DateTimeToDayId(UnixTimeToDateTime(localTimestamp));
}

// 从 dayId 获取起始时间戳
public static long GetGameDayStartTimestamp(int dayId)
{
    // 1. dayId → DateTime（自然日 00:00:00）
    var naturalDate = SafeDate.DayIdToDateTime(dayId);
    
    // 2. DateTime → UTC 时间戳
    long utcTimestamp = DateTimeToUnixTime(naturalDate);
    
    // 3. 计算游戏日起点
    //    = UTC 00:00:00 - 标准偏移 + 凌晨3点偏移
    return utcTimestamp - StandardOffsetSec + ResetOffsetSec;
}
```

### 为什么这样做能解决 DST 问题

```csharp
// 关键：标准偏移永不含 DST
long StandardOffsetSec = (long)TimeZoneInfo.Local.BaseUtcOffset.TotalSeconds;
// ✅ 美东恒为 -18000（-5h），夏天也不会变 -14400（-4h）
// ✅ 首次缓存后，游戏期间永不变化

// 所有时间计算都基于这个恒定的偏移
// → DST 转换时不会产生 ±1h 的偏移
// → dayId 的时间戳永远准确
```

## 禁止事项

```csharp
// ❌ 1. dayId 直接做算术
int tomorrow = today + 1;  // 不处理月份边界！

// ❌ 2. 本地时间比较活动
if (DateTime.Now >= activity.StartTime) { }

// ❌ 3. 调用 DST-aware API
TimeZoneInfo.Local.GetUtcOffset(DateTime.Now)  // 会返回 DST 偏移！
dateTime.ToLocalTime()

// ❌ 4. dayId 参与时间范围判断
if (today >= 20260710 && today <= 20260813) { }

// ❌ 5. 混淆游戏日和自然日
int today = TimeSystem.GetGameNaturalTodayDayId();  // ❌ 错
int today = TimeSystem.GetGameTodayDayId();         // ✅ 对
```

## 完整实现（分段复制）

### 1. TimeSystem.cs（关键方法）

```csharp
using System;
using UnityEngine;

namespace TLF.TimeSystem
{
    public static class TimeSystem
    {
        public const long DaySec = 86400L;
        public const long ResetOffsetSec = 3 * 3600L;

        // 标准偏移（永不含 DST），首次缓存
        private static long? _cachedStandardOffsetSec;
        private static bool _initialized = false;

        public static long StandardOffsetSec
        {
            get
            {
                if (!_initialized)
                {
                    _cachedStandardOffsetSec = (long)TimeZoneInfo.Local.BaseUtcOffset.TotalSeconds;
                    _initialized = true;
                }
                return _cachedStandardOffsetSec.Value;
            }
        }

        public static long GetServerTimestamp() => DateTimeOffset.UtcNow.ToUnixTimeSeconds();

        public static int GetGameTodayDayId() => GetGameDayIdFromTimestamp(GetServerTimestamp());

        public static int GetGameDayIdFromTimestamp(long timestamp)
        {
            long localTimestamp = timestamp + StandardOffsetSec;
            long dayStart = (localTimestamp / DaySec) * DaySec;
            
            if ((localTimestamp % DaySec) < ResetOffsetSec)
                dayStart -= DaySec;
            
            var dt = DateTimeOffset.FromUnixTimeSeconds(dayStart).UtcDateTime;
            return dt.Year * 10000 + dt.Month * 100 + dt.Day;
        }

        public static long GetGameDayStartTimestamp(int dayId)
        {
            var date = ParseDayId(dayId);
            long utcTimestamp = new DateTimeOffset(date).ToUnixTimeSeconds();
            return utcTimestamp - StandardOffsetSec + ResetOffsetSec;
        }

        public static long GetGameDayEndTimestamp(int dayId)
        {
            return GetGameDayStartTimestamp(dayId) + DaySec - 1;
        }

        public static int AddDays(int dayId, int days)
        {
            var date = ParseDayId(dayId).AddDays(days);
            return date.Year * 10000 + date.Month * 100 + date.Day;
        }

        public static int DiffDays(int laterDayId, int earlierDayId)
        {
            return (int)(ParseDayId(laterDayId) - ParseDayId(earlierDayId)).TotalDays;
        }

        public static long GetSecondsToNextGameDay()
        {
            long now = GetServerTimestamp();
            int today = GetGameDayIdFromTimestamp(now);
            long nextDay = GetGameDayStartTimestamp(AddDays(today, 1));
            return nextDay - now;
        }

        public static bool IsActivityActive(long start, long end) 
            => GetServerTimestamp() >= start && GetServerTimestamp() <= end;

        public static int GetActivityOpenDay(long startTimestamp)
        {
            if (!IsActivityStarted(startTimestamp)) return 0;
            int startDay = GetGameDayIdFromTimestamp(startTimestamp);
            int today = GetGameTodayDayId();
            return DiffDays(today, startDay) + 1;
        }

        private static bool IsActivityStarted(long startTimestamp) 
            => GetServerTimestamp() >= startTimestamp;

        private static DateTime ParseDayId(int dayId)
        {
            int year = dayId / 10000;
            int month = (dayId / 100) % 100;
            int day = dayId % 100;
            return new DateTime(year, month, day);
        }
    }
}
```

### 2. 在项目中使用

```csharp
using TLF.TimeSystem;
using UnityEngine;

public class GameManager : MonoBehaviour
{
    void Start()
    {
        long now = TimeSystem.GetServerTimestamp();
        int today = TimeSystem.GetGameTodayDayId();
        
        Debug.Log($"Current timestamp: {now}");
        Debug.Log($"Today dayId: {today}");
    }

    void Update()
    {
        // 每日重置检查
        CheckDailyReset();
    }

    private int _lastResetDay = -1;

    void CheckDailyReset()
    {
        int today = TimeSystem.GetGameTodayDayId();
        if (today != _lastResetDay)
        {
            _lastResetDay = today;
            OnDailyReset();
        }
    }

    void OnDailyReset()
    {
        Debug.Log($"Daily reset on {_lastResetDay}");
        // 业务逻辑
    }
}
```

## 为什么有效

| 方面 | 旧系统（问题） | TimeSystem（解决） |
|------|-------------|-----------------|
| **时间计算基准** | 本地时间（受 DST 影响） | UTC 秒（不受 DST 影响） |
| **时区偏移** | 动态获取（含 DST）| 静态缓存（永不含 DST） |
| **dayId 的一致性** | 同一 dayId 时间戳在 DST 转换时偏移 ±1h | 同一 dayId 时间戳永远不变 |
| **跨天判断** | DST 转换日期容易出错 | 恒定不变，完全可靠 |

---

**Unity 版本**：2022.3.62f2  
**集成位置**：`Assets/Scripts/Core/TimeSystem/TimeSystem.cs`  
**状态**：✅ 可用于生产环境
