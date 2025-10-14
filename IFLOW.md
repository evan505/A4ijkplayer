# A4ijkplayer 项目概述

## 项目简介

A4ijkplayer 是一个基于 [ijkplayer](https://github.com/bilibili/ijkplayer) k0.8.8 版本的 Android 播放器库。该项目的主要目的是将 ijkplayer 的编译方式从 Android.mk 迁移到 CMake，以便于在 Android Studio 中进行开发、调试和维护。

- **核心功能**: 播放网络视频（支持 https）、本地视频文件。
- **主要技术栈**: 
  - C/C++ (ijkplayer 核心逻辑，包括 ffmpeg, sdl, soundtouch 等模块)
  - Java/Kotlin (Android 应用层接口)
  - CMake (构建系统)
  - Gradle (Android 构建工具)
- **架构**: 采用 Android Library 模块 (`A4ijkplayer`) 提供播放器功能，通过 JNI 调用底层 C/C++ 代码。
- **特性**:
  - 支持 https 网络视频播放（集成了 OpenSSL）。
  - 可以在 Android Studio 中直接查看、跳转和调试 ijkplayer 的 C/C++ 源码。
  - 提供了示例应用 (`A4ijkplayerDemo`) 用于演示如何集成和使用。

## 项目结构

```
A4ijkplayer/
├── A4ijkplayer/               # 核心播放器库模块
│   ├── build.gradle           # 库模块的构建配置
│   └── src/
│       └── main/
│           ├── cpp/           # C/C++ 源码 (ijkplayer 及其依赖)
│           │   ├── CMakeLists.txt  # CMake 构建脚本
│           │   ├── ijkmedia/       # ijkplayer 核心源码
│           │   ├── ijkprof/        # 性能分析相关代码
│           │   └── otherlibs/      # 第三方库 (如预编译的 ffmpeg)
│           └── java/          # Java 接口代码
├── A4ijkplayerDemo/           # 示例应用程序模块
│   ├── build.gradle           # 应用模块的构建配置
│   └── src/
│       └── main/
│           ├── AndroidManifest.xml
│           ├── java/          # 示例应用的 Java 代码
│           └── res/           # 资源文件
├── build.gradle               # 顶层构建脚本
├── settings.gradle            # 项目设置，包含所有模块
├── gradle.properties          # Gradle 属性配置
├── gradlew & gradlew.bat      # Gradle Wrapper 脚本
└── README.md                  # 项目说明文档
```

## 构建与运行

### 环境要求

- Android Studio (推荐最新稳定版)
- Android NDK (版本 21.4.7075529，已在 `A4ijkplayer/build.gradle` 中指定)
- CMake (版本 3.6.0，已在 `A4ijkplayer/build.gradle` 中指定)

### 构建 AAR

要构建 A4ijkplayer 库的 AAR 文件，请在项目根目录下执行以下命令：

```bash
# 构建 Debug 版本
./gradlew clean :A4ijkplayer:assembleDebug

# 构建 Release 版本
./gradlew clean :A4ijkplayer:assembleRelease
```

生成的 AAR 文件将位于 `./A4ijkplayer/build/outputs/aar/` 目录下。

### 运行示例应用

要运行示例应用 `A4ijkplayerDemo`，请先确保已构建 A4ijkplayer 库，然后执行：

```bash
./gradlew :A4ijkplayerDemo:assembleDebug
```

然后可以在 Android Studio 中直接运行 `A4ijkplayerDemo` 模块，或者将生成的 APK 安装到设备上。

## 开发约定

- **C/C++ 代码**: ijkplayer 核心代码位于 `A4ijkplayer/src/main/cpp/ijkmedia/` 目录下。修改这些代码后需要重新构建 AAR。
- **Java 代码**: Android 接口代码位于 `A4ijkplayer/src/main/java/` 目录下。可以在 Android Studio 中直接修改、编译和调试。
- **第三方库**: 预编译的 ffmpeg 库位于 `A4ijkplayer/src/main/cpp/otherlibs/ffmpeg/` 目录下。如果需要更新 ffmpeg 版本，则需要替换此目录下的文件。
- **CMake 配置**: CMake 构建脚本位于 `A4ijkplayer/src/main/cpp/CMakeLists.txt`。如果添加了新的 C/C++ 源文件，需要在此文件中进行相应的配置。


## 项目文档目录

历史上下文信息位于 `./ai-md` 目录
