---
title: 【笔记】global-metadata.dat文件加密
tags: [Unity, IL2CPP, 安全, iOS, Android, 深度解析]
category: 高级主题/安全
created: 2026-07-02
description: 通过加密 global-metadata.dat 并重新编译 libil2cpp 静态库实现 IL2CPP 保护，涵盖 iOS/Android 两端编译方案
unity_version: "2019.4+"
status: 待验证
validation: 项目实战
author: human
sources:
  - "https://github.com/kkusdoit/CompileLibil2cpp/tree/main"
---

# global-metadata.dat 文件加密

> Unity 游戏包体逆向会用到 global-metadata.dat，因此通过对它进行加密操作，然后在 il2cpp 代码中解密，从而完成 App 加密。这涉及到对 il2cpp 静态库的编译，分 iOS 和 Android 两种情况。

## 平台差异

### Android

- 直接修改 il2cpp 源码，Unity 打包时自动编译

### iOS

- 只能将 il2cpp 源码编译生成静态库 `libil2cpp.a` 文件，然后在 Xcode 工程中替换

---

## Xcode 编译 libil2cpp.a

### 源码位置

```
/Applications/Unity/Hub/Editor/<Your Unity Version>/Unity.app/Contents/il2cpp
```

例如：`/Applications/Unity/Hub/Editor/2019.4.28f1c1/Unity.app/Contents/il2cpp`

### 方案一：自己动手新建 Xcode 工程

新建 Xcode Static Library 工程，Build Setting 修改：

- Search headers
- Other Linker Flags
- 宏定义
- 等

Unity2019 版本经过各种尝试设置，最终可以编译成功。Xcode 静态库工程示例参考：[unity il2cpp ios 构建xcode工程编译](https://git.youle.game/tkw/client/il2cppxcodeproject)

### 方案二：利用 HybridCLR build_libil2cpp.sh 脚本

当切换到 Unity 2021 时，由于 il2cpp 代码结构变化，原 Xcode 工程无法使用。最终借助 HybridCLR（华佗）的编译脚本成功解决。

#### HybridCLR 与 libil2cpp 编译的关系

HybridCLR 是一个 Unity C# 热更方案，其原理涉及到修改 il2cpp 并重新编译。当 `com.code-philosophy.hybridclr` 版本 < v3.2.0 时：

> 除了 iOS 以外平台都是根据 libil2cpp 源码编译出目标程序，iOS 平台使用提前编译好的 `libil2cpp.a` 文件。Unity 导出的 Xcode 工程引用了提前生成好的 `libil2cpp.a`，而不包含 libil2cpp 源码，直接打包无法支持热更新。因此编译 iOS 程序时需要自己单独编译 `libil2cpp.a`，再**替换 Xcode 工程的 `libil2cpp.a` 文件**，接着再打包。

#### 操作步骤

1. 找到 < v3.2.0 的低版本 HybridCLR
2. 找到编译 il2cpp 的模块
3. 借助 AI 研究其 `build_libil2cpp.sh` 脚本和 `CMakeLists.txt`，确保编出来的是原汁原味的（不加料）
4. 把自己的 il2cpp 源码放到指定位置
5. 跑脚本构建

#### 遇到的链接错误

构建出的 `libil2cpp.a` 放到 Xcode 工程后，出现以下报错：

```
Undefined symbol: _SystemNative_ConvertErrorPalToPlatform
Undefined symbol: Il2WriteBarrier(void**, void*)
Undefined symbol: Il2WriteBarrierForType(Il2CppType const*, void**, void*)
Undefined symbol: Il2WriteBarrierForClass(Il2CppClass*, void**, void*)
Undefined symbol: il2cpp_codegen_get_generic_virtual_method_internal(MethodInfo const*, MethodInfo const*)
Linker command failed with exit code 1 (use -v to see invocation)
```

#### 原因分析

查看 il2cpp 源码，这些函数受 `IL2CPP_ENABLE_WRITE_BARRIERS` 宏控制：

```cpp
#if IL2CPP_ENABLE_WRITE_BARRIERS
void Il2WriteBarrier(void** targetAddress, void* object);
void Il2WriteBarrierForType(const Il2CppType* type, void** targetAddress, void* object);
void Il2WriteBarrierForClass(Il2CppClass* klass, void** targetAddress, void* object);
#else
inline void Il2WriteBarrier(void** targetAddress, void* object) {}
inline void Il2WriteBarrierForType(const Il2CppType* type, void** targetAddress, void* object) {}
inline void Il2WriteBarrierForClass(Il2CppClass* klass, void** targetAddress, void* object) {}
#endif
```

如果宏 `IL2CPP_ENABLE_WRITE_BARRIERS` 没有被定义，那么条件编译块之后的代码将不会被编译。链接错误的原因可能是：

- **其他代码引用了这些函数**：即使宏未定义，如果代码中有其他部分引用这些函数，链接器会尝试找到它们的定义
- **IL2CPP 内部依赖**：IL2CPP 可能在内部逻辑中依赖于这些函数，即使宏未定义也期望它们以某种形式存在
- **链接器配置问题**：项目配置或链接器设置不正确

#### 解决方案

在编译脚本（3 个 `CMakeLists.txt`）中加上 `IL2CPP_ENABLE_WRITE_BARRIERS` 宏：

```cmake
add_definitions(-DIL2CPP_ENABLE_WRITE_BARRIERS)
```

---

## 最终结果

- **Unity 2022**：libil2cpp 成功编译，出包测试没问题
- **Unity 2021**：同样成功，通过出包测试
- **结论**：此方法可以稳定编译 Unity 各版本的 `libil2cpp.a`

### 关键路径

| 用途 | 路径 |
|------|------|
| il2cpp 源码位置 | `CompileLibil2cpp/LocalIl2CppData-OSXEditor/il2cpp` |
| 编译脚本位置 | `CompileLibil2cpp/iOSBuild/build_libil2cpp.sh` |
| 最终静态库位置 | `CompileLibil2cpp/iOSBuild/build/libil2cpp.a` |

---

## 参考链接

- [unity il2cpp ios 构建xcode工程编译](https://blog.csdn.net/leilonghao/article/details/121625414)
- [unity il2cpp 源码编译](https://blog.csdn.net/leilonghao/article/details/121625414)
- [Unity Global-Metadata Hiding](https://floe-ice.cn/archives/55)
- [Unity 分析global-metadata.dat 原理](https://zhuanlan.zhihu.com/p/622597028)
- [Unity保护之il2cpp](https://nszdhd1.github.io/2020/07/03/Unity%E4%BF%9D%E6%8A%A4%E4%B9%8Bil2cpp/)
- [kkjusdoit/CompileLibil2cpp](https://github.com/kkjusdoit/CompileLibil2cpp/tree/main)
