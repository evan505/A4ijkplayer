# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

A4ijkplayer 是一个基于 bilibili ijkplayer (k0.8.8) 的 Android 视频播放器库,核心特性是将原本 Android.mk 的编译方式改为 CMake,使其可以在 Android Studio 中直接调试 C/C++ 源码。

- 基于 ijkplayer k0.8.8 版本,使用 ffmpeg 4.0
- 支持 https (集成 OpenSSL)
- 包含 yuv、sdl、soundtouch 等模块
- 支持 armeabi-v7a 和 arm64-v8a 架构

## 项目架构

### 模块结构

- **A4ijkplayer/**: 核心播放器库模块 (Android Library)
  - `src/main/cpp/`: C/C++ 源码
    - `ijkmedia/`: ijkplayer 核心代码
      - `ijkj4a/`: JNI Java 绑定层
      - `ijksdl/`: SDL 封装层
      - `ijksoundtouch/`: 音频变速变调
      - `ijkplayer/`: 播放器核心逻辑
      - `ijkyuv/`: YUV 图像处理
    - `ijkprof/`: 性能分析工具
    - `otherlibs/ffmpeg/`: 预编译的 ffmpeg 库 (libijkffmpeg.so)
    - `CMakeLists.txt`: CMake 构建配置
  - `src/main/java/`: Java 接口层
    - `tv.danmaku.ijk.media.player.*`: IjkMediaPlayer Java API

- **A4ijkplayerDemo/**: 示例应用,演示如何使用 A4ijkplayer

### C/C++ 与 Java 交互

- 主要 JNI 入口: `A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/android/ijkplayer_jni.c`
- Java 侧主入口: `A4ijkplayer/src/main/java/tv/danmaku/ijk/media/player/IjkMediaPlayer.java`
- Native 库名称: `a4ijkplayer` (生成 liba4ijkplayer.so)

### 依赖关系

- ffmpeg 库已预编译,位于 `A4ijkplayer/src/main/cpp/otherlibs/ffmpeg/libs/${ANDROID_ABI}/libijkffmpeg.so`
- 如需修改 ffmpeg,必须重新编译 ffmpeg (参考原项目文档)
- 其他依赖: OpenSLES, EGL, GLESv2, jnigraphics 等系统库

## 常用开发命令

### 构建 AAR

```bash
# 构建 Debug 版本
./gradlew clean :A4ijkplayer:assembleDebug

# 构建 Release 版本
./gradlew clean :A4ijkplayer:assembleRelease
```

生成的 AAR 文件位于: `./A4ijkplayer/build/outputs/aar/`

### 构建和运行示例应用

```bash
# 构建 Demo APK
./gradlew :A4ijkplayerDemo:assembleDebug

# 安装到设备
./gradlew :A4ijkplayerDemo:installDebug

# 卸载
./gradlew :A4ijkplayerDemo:uninstallDebug
```

### 运行测试

```bash
# 运行单元测试
./gradlew :A4ijkplayer:test

# 运行 instrumentation 测试
./gradlew :A4ijkplayer:connectedAndroidTest
```

### 清理构建

```bash
# 清理所有构建产物
./gradlew clean

# 清理 CMake 构建缓存 (A4ijkplayer/build.gradle 中已配置在 clean 时自动删除 .cxx 目录)
rm -rf A4ijkplayer/.cxx
```

## 开发环境要求

- Android Studio (推荐最新稳定版)
- Android NDK 21.4.7075529 (在 A4ijkplayer/build.gradle 中指定)
- CMake 3.6.0+ (在 A4ijkplayer/build.gradle 中指定)
- Gradle 7.3.0
- JDK 8+ (sourceCompatibility/targetCompatibility 设置为 Java 8)

## 代码修改指南

### 修改 Java 代码

- 直接在 Android Studio 中修改 `A4ijkplayer/src/main/java/` 下的代码
- 修改后可直接运行 `:A4ijkplayerDemo` 查看效果

### 修改 C/C++ 代码

- ijkplayer 核心代码位于 `A4ijkplayer/src/main/cpp/ijkmedia/` 目录
- 修改后需要重新构建 AAR
- 可以在 Android Studio 中设置断点进行 native 调试

### 添加新的 C/C++ 源文件

- 需要在 `A4ijkplayer/src/main/cpp/CMakeLists.txt` 中添加对应的 file(GLOB_RECURSE ...) 或手动指定
- CMake 配置会自动递归收集各模块的 .c/.cpp/.cc 文件

### 修改 ffmpeg 源码

- 需要重新编译 ffmpeg 生成 libijkffmpeg.so
- 将新编译的 .so 文件替换到 `A4ijkplayer/src/main/cpp/otherlibs/ffmpeg/libs/${ANDROID_ABI}/` 目录
- 参考: https://github.com/alanwang4523/ijkplayer_Build4Android

## 支持的 ABI

- armeabi-v7a
- arm64-v8a

在 `A4ijkplayer/build.gradle` 的 `externalNativeBuild.cmake.abiFilters` 中配置

## 历史文档

项目的历史上下文和分析文档位于 `./ai-md` 目录,包含:
- ijkplayer 各模块分析 (协议层、解封装、解码、渲染、音视频同步等)
- 代码流程分析文档
