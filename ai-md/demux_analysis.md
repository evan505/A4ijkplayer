# ijkplayer 解封装模块分析

## 1. 概述

解封装模块是 ijkplayer 中负责解析音视频文件格式的组件，它将输入的数据流分解为独立的音频、视频和字幕数据包。该模块基于 FFmpeg 的 libavformat 库实现，并针对移动平台进行了优化。

核心实现位于：
- `A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c` - 解封装主流程
- FFmpeg 的 libavformat 库 - 底层格式解析

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

解封装器的初始化和媒体信息探测流程：

1. **`ijkmp_prepare_async`** (A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ijkplayer.c)
   - **作用**：用户层触发播放准备
   - **参数**：`IjkMediaPlayer *mp`
   - **实现细节**：
     - 检查播放器状态，必须已设置数据源
     - 通过 JNI 发送 MEDIA_PREPARING 消息
     - 异步执行，立即返回
   - **状态机**：IDLE -> PREPARING

2. **`ffp_prepare_async_l`** (A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c)
   - **作用**：FFPlayer 层的准备函数
   - **实现细节**：
     - 初始化播放器选项（格式选项、编解码器选项）
     - 设置网络超时、用户代理等参数
     - 配置硬件解码器选项
     - 创建消息队列用于与上层通信
   - **选项处理**：
     - `format_opts` - 解封装选项（如 timeout、headers）
     - `codec_opts` - 解码器选项
     - `sws_opts` - 视频缩放选项

3. **`stream_open`** (ff_ffplay.c)
   - **作用**：创建播放器状态结构并启动读线程
   - **参数**：
     - `const char *filename` - 媒体 URL
     - `AVInputFormat *iformat` - 指定输入格式（通常为 NULL，自动探测）
   - **初始化内容**：
     - 分配 `VideoState` 结构体
     - 初始化音频/视频/字幕队列
     - 创建互斥锁和条件变量
     - 初始化时钟（音频时钟、视频时钟、外部时钟）
     - 创建 `read_thread` 线程
   - **返回值**：返回 VideoState 指针，失败返回 NULL

4. **`read_thread`** (ff_ffplay.c)
   - **作用**：读取线程入口函数
   - **生命周期**：从创建到播放结束
   - **初始化阶段职责**：
     - 调用 `avformat_alloc_context` 分配 AVFormatContext
     - 设置中断回调（用于超时处理）
     - 调用 `avformat_open_input` 打开输入
     - 调用 `avformat_find_stream_info` 探测流信息
     - 选择音频/视频/字幕流
     - 打开解码器

5. **`avformat_alloc_context`** (FFmpeg libavformat)
   - **作用**：分配 AVFormatContext 结构体
   - **返回值**：返回初始化的 AVFormatContext 指针
   - **初始化内容**：
     - 设置默认值
     - 分配私有数据
     - 初始化流数组
   - **内存管理**：需要用 `avformat_free_context` 或 `avformat_close_input` 释放

6. **`avformat_open_input`** (FFmpeg libavformat)
   - **作用**：打开媒体文件/流并读取头部信息
   - **参数**：
     - `AVFormatContext **ps` - 格式上下文指针的指针
     - `const char *url` - 输入 URL
     - `AVInputFormat *fmt` - 强制使用的格式（NULL 则自动探测）
     - `AVDictionary **options` - 选项字典
   - **执行流程**：
     1. **格式探测**：
        - 如果未指定格式，读取文件头（probe size 默认 5MB）
        - 调用各个 demuxer 的 `read_probe` 函数
        - 选择匹配度最高的格式
     2. **打开协议**：调用 `avio_open2` 打开底层 I/O
     3. **读取头部**：
        - 调用 demuxer 的 `read_header` 函数
        - 解析容器格式头部（如 MP4 的 ftyp、moov box）
     4. **创建流**：为每个音频/视频/字幕轨道创建 AVStream
   - **常见格式**：
     - MP4/MOV：解析 box 结构
     - FLV：读取 FLV header 和 metadata
     - HLS：解析 m3u8 文件
     - RTMP：完成 RTMP 握手
   - **错误处理**：返回负数表示失败，需要检查具体错误码

7. **`avformat_find_stream_info`** (FFmpeg libavformat)
   - **作用**：通过读取和解码部分数据来获取详细的流信息
   - **参数**：
     - `AVFormatContext *ic` - 格式上下文
     - `AVDictionary **options` - 每个流的选项（数组）
   - **执行过程**：
     1. **读取数据包**：
        - 读取一定数量的数据包（默认最多 5 秒的数据）
        - 或读取指定数量的帧（max_analyze_duration）
     2. **尝试解码**：
        - 为每个流创建临时解码器
        - 解码部分帧以获取精确信息
     3. **提取信息**：
        - 编解码器类型、参数
        - 采样率、声道数（音频）
        - 分辨率、帧率（视频）
        - 时长、比特率
     4. **计算时基**：
        - 确定每个流的 time_base
        - 计算平均帧率
   - **性能考虑**：
     - 可能较慢（特别是网络流）
     - 可通过 `probesize` 和 `analyzeduration` 选项控制
   - **返回值**：成功返回 >=0，失败返回 <0

### 4.2 数据读取流程

解封装器的主循环，持续读取并分发数据包：

1. **`read_thread` 主循环** (A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c)
   - **循环条件**：
     - `!is->abort_request` - 未请求停止
     - 队列未满或需要读取
   - **队列管理**：
     ```c
     if (is->audioq.size + is->videoq.size + is->subtitleq.size > MAX_QUEUE_SIZE)
         SDL_Delay(10); // 等待解码消耗数据
     ```
   - **缓冲策略**：
     - **点播**：队列达到阈值暂停读取，降低内存
     - **直播**：`infinite_buffer` 模式，不限制队列
   - **Seek 处理**：
     - 检测 `seek_req` 标志
     - 调用 `av_seek_frame`
     - 刷新队列和解码器

2. **`av_read_frame`** (FFmpeg libavformat)
   - **作用**：从容器中读取一个完整的数据包
   - **参数**：
     - `AVFormatContext *s` - 格式上下文
     - `AVPacket *pkt` - 输出数据包
   - **执行流程**：
     1. **调用 demuxer**：
        - 调用具体格式的 `read_packet` 函数
        - MP4：解析 mdat box 中的数据
        - FLV：读取 FLV tag
        - TS：读取 TS packet，组装 PES
     2. **解析数据包**：
        - 确定数据包所属的流（stream_index）
        - 提取时间戳（PTS/DTS）
        - 转换时间戳到流的 time_base
     3. **设置标志**：
        - `AV_PKT_FLAG_KEY` - 关键帧标志
        - `AV_PKT_FLAG_CORRUPT` - 损坏数据标志
   - **特殊情况**：
     - **分片包**：部分格式（如 TS）可能需要多次调用才能返回完整包
     - **交错读取**：在多个流之间交替读取
   - **内存管理**：
     - AVPacket 使用引用计数
     - 需要调用 `av_packet_unref` 释放
   - **返回值**：
     - 0 - 成功读取
     - AVERROR_EOF - 文件结束
     - 其他负值 - 错误

3. **流类型判断与分发**
   ```c
   if (pkt->stream_index == is->audio_stream)
       packet_queue_put(&is->audioq, pkt);
   else if (pkt->stream_index == is->video_stream)
       packet_queue_put(&is->videoq, pkt);
   else if (pkt->stream_index == is->subtitle_stream)
       packet_queue_put(&is->subtitleq, pkt);
   ```

4. **`packet_queue_put`** (ff_ffplay.c)
   - **作用**：将数据包放入队列
   - **参数**：
     - `PacketQueue *q` - 目标队列
     - `AVPacket *pkt` - 数据包
   - **实现细节**：
     1. **创建节点**：
        ```c
        MyAVPacketList *pkt1 = av_malloc(sizeof(MyAVPacketList));
        pkt1->pkt = *pkt;
        pkt1->next = NULL;
        ```
     2. **线程同步**：
        ```c
        SDL_LockMutex(q->mutex);
        // 添加到队列尾部
        if (!q->last_pkt)
            q->first_pkt = pkt1;
        else
            q->last_pkt->next = pkt1;
        q->last_pkt = pkt1;
        q->nb_packets++;
        q->size += pkt1->pkt.size;
        SDL_CondSignal(q->cond); // 唤醒等待的解码线程
        SDL_UnlockMutex(q->mutex);
        ```
     3. **序列号管理**：
        - 每个包关联一个 serial，用于 seek 后丢弃旧包
   - **内存统计**：
     - `nb_packets` - 包数量
     - `size` - 总字节数

5. **`SDL_CondSignal`** (SDL condition variable)
   - **作用**：通知解码线程有新数据可用
   - **同步机制**：
     - 解码线程在队列为空时调用 `SDL_CondWait` 等待
     - 读线程放入数据后调用 `SDL_CondSignal` 唤醒
   - **避免丢失信号**：在持有锁的情况下发送信号

### 4.3 流结束处理

媒体文件播放到末尾的处理流程：

1. **`av_read_frame` 返回 AVERROR_EOF**
   - **检测时机**：读线程在主循环中检测返回值
   - **EOF 原因**：
     - 文件读取完毕
     - 网络流结束
     - 直播流断开
   - **代码示例**：
     ```c
     ret = av_read_frame(ic, pkt);
     if (ret == AVERROR_EOF) {
         // 处理文件结束
     }
     ```

2. **`packet_queue_put_nullpacket`** (ff_ffplay.c)
   - **作用**：向队列发送空数据包，标记流结束
   - **参数**：
     - `PacketQueue *q` - 目标队列
     - `int stream_index` - 流索引
   - **空包特征**：
     - `pkt->data = NULL`
     - `pkt->size = 0`
     - 特殊的 serial 值
   - **作用**：
     - 通知解码线程该流已结束
     - 解码线程会刷新解码器缓冲区
     - 输出所有延迟帧
   - **多流处理**：
     ```c
     if (is->audio_stream >= 0)
         packet_queue_put_nullpacket(&is->audioq, is->audio_stream);
     if (is->video_stream >= 0)
         packet_queue_put_nullpacket(&is->videoq, is->video_stream);
     ```

3. **解码器刷新**
   - **`avcodec_send_packet(NULL)`**：发送空包到解码器
   - **延迟帧输出**：某些编解码器会缓存帧（如 H.264 的 B 帧）
   - **循环调用 `avcodec_receive_frame`** 直到返回 AVERROR_EOF

4. **`ffp_notify_msg1(FFP_MSG_COMPLETED)`** (ff_ffplay.c)
   - **作用**：通知上层应用播放完成
   - **消息机制**：
     - 通过消息队列发送到 Java 层
     - Java 层回调 `OnCompletionListener`
   - **触发条件**：
     - 所有流都已解码完成
     - 最后一帧已显示
   - **状态转换**：PLAYING -> COMPLETED
   - **用户体验**：
     - UI 更新播放进度到 100%
     - 显示"播放完成"提示
     - 可能自动播放下一个（播放列表）

5. **循环播放处理**
   - **检查 `loop` 选项**：
     ```c
     if (is->loop != 0) {
         stream_seek(is, start_time, 0, 0);
         is->loop--;
     }
     ```
   - **无限循环**：`loop = -1`
   - **重新开始播放**：自动 seek 到起始位置

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