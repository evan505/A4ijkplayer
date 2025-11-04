# ijkplayer 解协议模块分析

## 1. 概述

ijkplayer 的解协议模块主要负责处理各种网络协议，如 HTTP、HTTPS、RTMP 等，它是播放器与网络数据源之间的桥梁。该模块基于 FFmpeg 的协议处理能力，并进行了扩展以支持特定需求，如缓存、重连等。

核心实现位于：
- `A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c` - 读取线程和协议初始化
- FFmpeg 的 libavformat 库 - 底层协议支持

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

播放器打开网络或本地媒体资源的完整流程：

1. **`ijkmp_set_data_source`** (A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ijkplayer.c)
   - **作用**：设置播放器的数据源 URL
   - **参数**：`IjkMediaPlayer *mp`, `const char *url`
   - **实现细节**：保存 URL 字符串到播放器上下文中，支持 http/https/rtmp/file 等协议
   - **返回值**：成功返回 0，失败返回负数错误码

2. **`ijkmp_prepare_async_l`** (ijkplayer.c)
   - **作用**：异步准备播放，避免阻塞主线程
   - **实现细节**：获取播放器互斥锁，调用 FFPlayer 层的准备函数
   - **线程安全**：通过 `pthread_mutex_lock` 保证线程安全
   - **状态转换**：播放器状态从 IDLE 转换到 PREPARING

3. **`ffp_prepare_async_l`** (A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c)
   - **作用**：FFmpeg 播放器层的准备入口
   - **实现细节**：
     - 检查播放器状态，确保未初始化
     - 解析 URL 中的特殊参数（如 timeout、headers 等）
     - 配置 AVFormatContext 的选项字典
     - 调用 `stream_open` 创建读线程
   - **错误处理**：失败时清理资源并返回错误码

4. **`read_thread`** (ff_ffplay.c)
   - **作用**：播放器的读取线程主函数，负责从网络/文件读取数据
   - **生命周期**：从 `stream_open` 启动，直到播放结束或出错
   - **关键职责**：
     - 打开输入流（调用 `avformat_open_input`）
     - 查找流信息
     - 循环读取数据包并分发到音频/视频/字幕队列
     - 处理 seek 操作
   - **线程模型**：独立线程运行，通过条件变量与解码线程同步

5. **`avformat_open_input`** (FFmpeg libavformat)
   - **作用**：FFmpeg 打开输入流的核心函数
   - **参数**：`AVFormatContext **ps`, `const char *url`, `AVInputFormat *fmt`, `AVDictionary **options`
   - **实现细节**：
     - 探测文件格式（基于文件头或扩展名）
     - 打开底层协议（通过 avio_open2）
     - 读取文件头，初始化 demuxer
     - 分配流结构体
   - **协议处理**：根据 URL schema（http/https/rtmp/file）调用相应的协议处理器
   - **返回值**：成功返回 0，失败返回 AVERROR 错误码

6. **协议层打开函数** (FFmpeg protocol layer)
   - **HTTP/HTTPS**：`http_open` - 建立 TCP 连接，发送 HTTP GET 请求
   - **RTMP**：`rtmp_open` - RTMP 握手，建立流媒体连接
   - **File**：`file_open` - 打开本地文件，返回文件描述符
   - **实现细节**：
     - 设置超时参数
     - 处理重定向（对于 HTTP）
     - 验证服务器响应
     - 配置缓冲区大小

### 3.2 读取数据

数据包的循环读取和分发流程：

1. **`read_thread` 主循环** (A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c)
   - **作用**：持续从输入源读取数据包
   - **循环条件**：
     - 播放器未停止（`!is->abort_request`）
     - 缓冲区未满或需要继续读取
   - **缓冲区管理**：
     - 检查音频/视频队列大小
     - 达到高水位时暂停读取，降低内存占用
     - 无限缓冲模式下不检查队列大小
   - **性能优化**：
     - 批量读取数据包
     - 队列满时 sleep 10ms，避免 CPU 空转

2. **`av_read_frame`** (FFmpeg libavformat)
   - **作用**：从 AVFormatContext 读取一个完整的数据包
   - **参数**：`AVFormatContext *s`, `AVPacket *pkt`
   - **实现细节**：
     - 调用 demuxer 的 `read_packet` 函数
     - 从底层协议读取原始数据
     - 解析容器格式，提取完整的音频/视频帧
     - 填充 AVPacket 结构（data、size、pts、dts、stream_index）
   - **内存管理**：自动分配 AVPacket 内存，调用者需要释放
   - **返回值**：
     - 成功返回 0
     - 文件结束返回 AVERROR_EOF
     - 错误返回其他 AVERROR 代码

3. **协议层读取函数** (FFmpeg protocol layer)
   - **HTTP 读取流程**：
     - `http_read` - 从 socket 读取 HTTP body 数据
     - 检查 Content-Length，避免读取越界
     - 处理分块传输编码（chunked）
     - 自动重连机制（连接断开时）
   - **缓冲机制**：
     - 使用内部缓冲区（默认 32KB）
     - 减少系统调用次数
     - 提高读取效率
   - **数据流向**：`Socket -> Protocol Buffer -> AVIOContext -> Demuxer -> AVPacket`

4. **数据包分发**
   - **`packet_queue_put`** (ff_ffplay.c)
     - 根据 `stream_index` 将包放入对应队列
     - 音频包 -> `audioq`
     - 视频包 -> `videoq`
     - 字幕包 -> `subtitleq`
   - **队列同步**：
     - 使用互斥锁保护队列操作
     - 条件变量通知解码线程
   - **内存控制**：
     - 记录队列总大小
     - 超过限制时阻塞读取线程

### 3.3 定位操作（Seek）

视频播放位置跳转的完整流程：

1. **`stream_seek`** (A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c)
   - **作用**：触发播放位置跳转
   - **参数**：
     - `VideoState *is` - 播放器状态
     - `int64_t pos` - 目标位置（微秒）
     - `int64_t rel` - 相对偏移
     - `int seek_by_bytes` - 是否按字节定位
   - **实现细节**：
     - 计算目标时间戳（转换为 AVStream 的 time_base）
     - 设置 seek 标志和目标位置
     - 通知读取线程执行 seek
     - 清空已缓存的音视频队列
   - **线程同步**：通过 `seek_req` 标志和条件变量同步

2. **`av_seek_frame`** (FFmpeg libavformat)
   - **作用**：在媒体文件中定位到指定时间点
   - **参数**：
     - `AVFormatContext *s` - 格式上下文
     - `int stream_index` - 流索引（-1 表示默认流）
     - `int64_t timestamp` - 目标时间戳
     - `int flags` - 定位标志（AVSEEK_FLAG_BACKWARD/FORWARD）
   - **定位策略**：
     - **基于时间**：根据 PTS 定位到关键帧
     - **基于字节**：直接定位到文件偏移量（实时流不支持）
     - **向后定位**：找到小于目标时间的最近关键帧
     - **向前定位**：找到大于目标时间的最近关键帧
   - **索引使用**：
     - 优先使用文件索引（如 MP4 的 moov box）
     - 无索引时进行二分查找
   - **返回值**：成功返回 0，失败返回负数

3. **协议层定位函数**
   - **`avio_seek`** (FFmpeg AVIOContext)
     - 底层协议的 seek 接口
     - 检查协议是否支持 seek（`seekable` 标志）
   - **HTTP 协议 Seek**：
     - 发送 Range 请求头：`Range: bytes=start-end`
     - 服务器返回 206 Partial Content
     - 重建 TCP 连接到新位置
     - 不支持 seek 的直播流返回错误
   - **File 协议 Seek**：
     - 调用 `lseek` 系统调用
     - 直接移动文件指针
     - 速度最快
   - **RTMP 协议 Seek**：
     - 发送 RTMP seek 命令
     - 等待服务器响应
     - 清空网络缓冲区

4. **Seek 后的恢复流程**
   - 刷新解码器状态（`avcodec_flush_buffers`）
   - 清空已解码但未显示的帧
   - 重置音视频时钟
   - 恢复正常播放状态

### 3.4 关闭连接

播放结束或用户停止时的资源清理流程：

1. **`stream_close`** (A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c)
   - **作用**：关闭流并释放所有资源
   - **清理顺序**（严格按此顺序避免死锁）：
     1. 设置 `abort_request` 标志，通知所有线程退出
     2. 等待读取线程结束（`SDL_WaitThread`）
     3. 关闭音频解码器和输出设备
     4. 关闭视频解码器和渲染器
     5. 清空数据包队列
     6. 释放 AVFormatContext
     7. 销毁互斥锁和条件变量
   - **线程安全**：
     - 使用条件变量唤醒所有等待的线程
     - 确保所有线程退出后再释放资源
   - **内存管理**：
     - 释放所有 AVPacket 和 AVFrame
     - 释放队列中的数据
     - 避免内存泄漏

2. **`avformat_close_input`** (FFmpeg libavformat)
   - **作用**：关闭 AVFormatContext 并释放相关资源
   - **参数**：`AVFormatContext **s` - 格式上下文指针的指针
   - **清理内容**：
     - 关闭所有流（AVStream）
     - 释放编解码器上下文
     - 释放元数据字典
     - 关闭底层 IO 上下文（AVIOContext）
     - 将指针设置为 NULL
   - **注意事项**：调用后原指针被置空，不可再次使用

3. **协议层关闭函数**
   - **`avio_close`** / `avio_closep`**
     - 刷新写缓冲区（如果有）
     - 调用协议层的 close 函数
     - 释放 AVIOContext 内存
   - **HTTP 协议关闭**：
     - 发送完整的数据包（避免 TCP RST）
     - 关闭 socket 连接（`close(fd)`）
     - 释放 HTTP 上下文内存
   - **File 协议关闭**：
     - 调用 `close` 系统调用
     - 刷新文件缓冲区
   - **RTMP 协议关闭**：
     - 发送 RTMP closeStream 命令
     - 关闭 RTMP 连接
     - 释放 RTMP 会话状态

4. **资源泄漏检查**
   - ijkplayer 在 DEBUG 模式下会检查：
     - AVPacket 引用计数
     - AVFrame 引用计数
     - 内存分配统计
   - 确保所有资源正确释放

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