# ijkplayer 解协议模块分析

## 1. 概述

ijkplayer 的解协议模块主要负责处理各种网络协议，如 HTTP、HTTPS、RTMP 等，它是播放器与网络数据源之间的桥梁。该模块基于 FFmpeg 的协议处理能力，并进行了扩展以支持特定需求，如缓存、重连等。

## 2. 核心组件

### 2.1 IjkIOManager

`IjkIOManager` 是解协议模块的核心管理器，负责协调各种协议的打开、读取、_seek_ 和关闭操作。它维护了一个上下文映射表 `ijk_ctx_map`，用于跟踪当前活动的 URL 上下文。

关键数据结构：
- `IjkIOManagerContext`: 管理器上下文，包含协议上下文映射、应用上下文等
- `IjkURLContext`: URL 上下文，表示一个具体的协议连接

### 2.2 IjkIo 协议

`ijkio` 是 ijkplayer 自定义的协议实现，它作为 FFmpeg 协议系统的扩展点。所有通过 ijkplayer 打开的 URL 都会经过 `ijkio` 协议处理。

关键函数：
- `ijkio_open`: 打开 URL，创建并初始化协议上下文
- `ijkio_read`: 读取数据
- `ijkio_seek`: 定位到指定位置
- `ijkio_close`: 关闭连接

### 2.3 协议实现

ijkplayer 支持多种协议实现，每种协议都有对应的 `URLProtocol` 结构体和实现函数：

- `ijkio_ffio_protocol`: 基于 FFmpeg 的文件 I/O 协议
- `ijkio_androidio_protocol`: Android 平台的文件 I/O 协议
- `ijkio_cache_protocol`: 带缓存功能的协议
- `ijkio_async_protocol`: 异步 I/O 协议
- `ijkio_httphook_protocol`: 带钩子的 HTTP 协议

## 3. 调用链分析

### 3.1 打开 URL

1. `ijkmp_set_data_source` (ijkplayer.c) - 设置数据源 URL
2. `ijkmp_prepare_async_l` (ijkplayer.c) - 异步准备播放
3. `ffp_prepare_async_l` (ff_ffplay.c) - 准备播放器
4. `read_thread` (ff_ffplay.c) - 启动读线程
5. `avformat_open_input` (FFmpeg) - 打开输入格式上下文
6. `ijkio_open` (ijkio.c) - ijkplayer 协议打开函数
7. `ijkio_manager_io_open` (ijkiomanager.c) - IO 管理器打开函数
8. `ijkio_alloc_url` - 分配并初始化协议上下文
9. 具体协议的 `url_open2` 函数 - 执行实际的打开操作

### 3.2 读取数据

1. `read_thread` (ff_ffplay.c) - 读线程循环
2. `av_read_frame` (FFmpeg) - 读取数据帧
3. `ijkio_read` (ijkio.c) - ijkplayer 协议读取函数
4. `ijkio_manager_io_read` (ijkiomanager.c) - IO 管理器读取函数
5. 具体协议的 `url_read` 函数 - 执行实际的读取操作

### 3.3 定位操作

1. `stream_seek` (ff_ffplay.c) - 触发定位操作
2. `av_seek_frame` (FFmpeg) - 定位到指定帧
3. `ijkio_seek` (ijkio.c) - ijkplayer 协议定位函数
4. `ijkio_manager_io_seek` (ijkiomanager.c) - IO 管理器定位函数
5. 具体协议的 `url_seek` 函数 - 执行实际的定位操作

### 3.4 关闭连接

1. `stream_close` (ff_ffplay.c) - 关闭流
2. `avformat_close_input` (FFmpeg) - 关闭输入格式上下文
3. `ijkio_close` (ijkio.c) - ijkplayer 协议关闭函数
4. `ijkio_manager_io_close` (ijkiomanager.c) - IO 管理器关闭函数
5. 具体协议的 `url_close` 函数 - 执行实际的关闭操作

## 4. 特殊功能

### 4.1 缓存机制

ijkplayer 通过 `ijkio_cache_protocol` 实现了数据缓存功能，可以将网络数据缓存到本地文件，以提高重复播放时的加载速度和减少网络流量。

关键组件：
- `IjkCacheContext`: 缓存上下文
- `IjkCacheEntry`: 缓存条目
- `IjkCacheTreeInfo`: 缓存树信息

### 4.2 异步 I/O

通过 `ijkio_async_protocol` 实现异步 I/O 操作，可以在后台线程中执行网络请求，避免阻塞主线程。

### 4.3 钩子机制

`ijkio_httphook_protocol` 和 `ijkio_urlhook_protocol` 提供了钩子机制，可以在特定事件发生时执行自定义逻辑，如请求开始、请求结束等。

## 5. 总结

ijkplayer 的解协议模块通过扩展 FFmpeg 的协议系统，实现了对多种网络协议的支持，并提供了缓存、异步 I/O 等高级功能。该模块的核心是 `IjkIOManager`，它协调各种协议的生命周期管理，并通过 `ijkio` 协议作为统一入口点。整个模块的设计具有良好的扩展性，可以方便地添加新的协议实现。