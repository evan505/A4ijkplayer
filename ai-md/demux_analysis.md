# ijkplayer 解封装模块分析

## 1. 概述

解封装模块是 ijkplayer 中负责解析音视频文件格式的组件，它将输入的数据流分解为独立的音频、视频和字幕数据包。该模块基于 FFmpeg 的 libavformat 库实现，并针对移动平台进行了优化。

## 2. 核心功能

### 2.1 格式探测与打开

解封装过程始于 `avformat_open_input` 函数调用，该函数会根据文件头或 URL 协议自动探测媒体格式，并创建相应的 `AVFormatContext` 上下文。

关键代码路径：
1. `ffp_prepare_async_l` (ff_ffplay.c) - 异步准备播放器
2. `read_thread` (ff_ffplay.c) - 读线程主函数
3. `avformat_alloc_context` - 分配格式上下文
4. `avformat_open_input` - 打开并初始化输入格式上下文

### 2.2 流信息查找

在打开输入文件后，需要通过 `avformat_find_stream_info` 函数获取详细的流信息，包括编解码器参数、时基等。

关键代码路径：
1. `read_thread` (ff_ffplay.c) - 读线程主函数
2. `avformat_find_stream_info` - 查找流信息
3. `setup_find_stream_info_opts` - 设置查找选项

### 2.3 流选择

ijkplayer 支持选择特定的音频、视频和字幕流进行播放。通过 `av_find_best_stream` 函数可以找到最佳的流。

关键代码路径：
1. `read_thread` (ff_ffplay.c) - 读线程主函数
2. `av_find_best_stream` - 查找最佳流
3. `stream_component_open` - 打开流组件

### 2.4 数据包读取

解封装的核心是循环读取数据包，通过 `av_read_frame` 函数从输入文件中提取音视频数据包。

关键代码路径：
1. `read_thread` (ff_ffplay.c) - 读线程主函数
2. `av_read_frame` - 读取数据包
3. `packet_queue_put` - 将数据包放入队列

## 3. 数据结构

### 3.1 AVFormatContext

这是 FFmpeg 中最重要的解封装数据结构，包含了输入文件的所有信息：
- `iformat`: 输入格式
- `pb`: 输入协议上下文
- `streams`: 流数组
- `nb_streams`: 流数量
- `metadata`: 元数据

### 3.2 AVStream

表示一个媒体流，包含：
- `codecpar`: 编解码器参数
- `time_base`: 时基
- `start_time`: 起始时间戳
- `duration`: 持续时间

### 3.3 PacketQueue

ijkplayer 自定义的数据包队列，用于在解封装和解码之间缓冲数据：
- `first_pkt`, `last_pkt`: 队列头尾指针
- `nb_packets`: 数据包数量
- `size`: 队列总大小
- `mutex`, `cond`: 同步原语

## 4. 调用链分析

### 4.1 初始化流程

1. `ijkmp_prepare_async` (ijkplayer.c) - 触发异步准备
2. `ffp_prepare_async_l` (ff_ffplay.c) - 准备播放器
3. `stream_open` (ff_ffplay.c) - 打开流
4. `read_thread` (ff_ffplay.c) - 启动读线程
5. `avformat_alloc_context` - 分配格式上下文
6. `avformat_open_input` - 打开输入
7. `avformat_find_stream_info` - 查找流信息

### 4.2 数据读取流程

1. `read_thread` (ff_ffplay.c) - 读线程循环
2. `av_read_frame` - 读取数据包
3. `packet_queue_put` - 放入音频队列/视频队列/字幕队列
4. `SDL_CondSignal` - 通知解码线程

### 4.3 流结束处理

1. `av_read_frame` 返回 AVERROR_EOF
2. `packet_queue_put_nullpacket` - 发送空包标记流结束
3. `ffp_notify_msg1(FFP_MSG_COMPLETED)` - 通知播放完成

## 5. 特性与优化

### 5.1 实时流处理

对于实时流（如 RTMP、HLS），ijkplayer 会进行特殊处理：
- 调整 `max_frame_duration` 参数
- 启用 `infinite_buffer` 模式
- 处理 RTSP 协议的暂停/播放

### 5.2 缓冲控制

通过 `FFDemuxCacheControl` 结构体控制解封装缓冲：
- `min_frames`: 最小帧数
- `max_buffer_size`: 最大缓冲区大小
- `high_water_mark_in_bytes`: 高水位标记

### 5.3 元数据处理

通过 `ijkmeta_set_avformat_context_l` 函数提取和设置媒体元数据，供上层应用使用。

## 6. 错误处理

解封装模块具有完善的错误处理机制：
- 网络错误检测与重连
- 文件结束处理
- 错误状态通知
- 资源清理

## 7. 总结

ijkplayer 的解封装模块基于 FFmpeg 的 libavformat 库，实现了完整的媒体文件格式解析功能。该模块具有良好的扩展性，支持多种媒体格式，并针对移动平台进行了优化，包括缓冲控制、实时流处理等特性。通过清晰的调用链和数据结构设计，解封装模块能够高效地为后续的解码和播放流程提供数据支持。