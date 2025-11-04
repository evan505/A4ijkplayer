# ijkplayer 视频渲染流程详解 - MediaCodec + OpenGL

## 文档说明

本文档面向对 OpenGL 和 Android 渲染结构不熟悉的开发者，详细讲解 ijkplayer 如何使用 MediaCodec 解码视频，并通过 OpenGL 渲染到屏幕上。

---

## 第一部分：基础概念

在深入了解渲染流程之前，我们需要理解几个核心概念。

### 1.1 什么是 MediaCodec？

**MediaCodec** 是 Android 提供的硬件加速解码器。

**为什么需要它？**
- **软件解码**：使用 CPU 解码视频，速度慢、耗电多
- **硬件解码**：使用 GPU 或专用解码芯片，速度快、省电

**通俗理解**：
想象你要处理一大堆数学题：
- **软件解码** = 你自己用笔算（慢）
- **硬件解码** = 用计算器（快）

MediaCodec 就是这个"计算器"，专门用来解码视频。

**MediaCodec 做什么？**
```
输入：压缩的视频数据（H.264/H.265 等）
      ↓
   [MediaCodec 解码]
      ↓
输出：原始的视频帧（YUV 格式）
```

### 1.2 什么是 OpenGL？

**OpenGL ES** 是在移动设备上绘制 2D/3D 图形的标准接口。

**为什么用 OpenGL 渲染视频？**
1. **高效**：GPU 专门处理图形，比 CPU 快得多
2. **灵活**：可以添加特效（滤镜、水印、旋转等）
3. **通用**：跨平台标准

**通俗理解**：
- **CPU** = 通用工人，什么活都能干，但不专业
- **GPU** = 专业画家，只画画，但画得又快又好

OpenGL 就是指挥 GPU 这个"画家"的语言。

### 1.3 什么是 Surface？

**Surface** 是 Android 中的画布。

**Surface 的层级关系**：
```
你看到的屏幕
    ↑
SurfaceView/TextureView（显示容器）
    ↑
Surface（画布）
    ↑
OpenGL 在这里绘制
```

**通俗理解**：
- **Surface** = 一块白板
- **OpenGL** = 在白板上画画
- **SurfaceView** = 把白板挂在墙上让你看到

### 1.4 什么是纹理（Texture）？

**纹理**就是贴在 3D 物体表面的图片。

在视频渲染中：
- **解码后的视频帧** = 纹理
- **屏幕上的矩形** = 3D 物体（四边形）
- **渲染** = 把视频帧贴到矩形上

**通俗理解**：
想象你在玩拼图：
- **纹理** = 拼图上的图案
- **矩形** = 拼图的形状框架
- **OpenGL** = 把图案贴到框架上

---

## 第二部分：整体架构

### 2.1 渲染流程概览

```
┌─────────────────────────────────────────────────────────────┐
│                   ijkplayer 视频渲染流程                      │
└─────────────────────────────────────────────────────────────┘

第1步：读取压缩数据
  网络/文件 → [读取线程] → 数据包队列
                               ↓

第2步：MediaCodec 硬件解码
  数据包队列 → [MediaCodec] → 解码后的视频帧（YUV）
                                   ↓

第3步：OpenGL 纹理处理
  视频帧 → [上传到 GPU] → OpenGL 纹理
                            ↓

第4步：OpenGL 渲染
  OpenGL 纹理 → [绘制] → Surface → 屏幕
```

### 2.2 关键组件

| 组件 | 作用 | 类比 |
|------|------|------|
| **读取线程** | 从网络/文件读取数据 | 送货员 |
| **MediaCodec** | 硬件解码视频 | 解压缩工具 |
| **EGL** | 连接 OpenGL 和 Android 窗口系统 | 桥梁 |
| **OpenGL 纹理** | 存储视频帧 | 画布上的图片 |
| **OpenGL 着色器** | 处理图像效果 | 图片滤镜 |
| **Surface** | 最终显示的画布 | 屏幕 |

### 2.3 代码模块对应

| 模块 | 文件位置 | 职责 |
|------|---------|------|
| **视频解码** | `ffpipenode_android_mediacodec_vdec.c` | MediaCodec 解码封装 |
| **OpenGL 初始化** | `ijksdl_egl.c` | EGL 环境设置 |
| **纹理管理** | `ijksdl_vout_android_nativewindow.c` | Surface 和纹理管理 |
| **OpenGL 渲染** | `gles2/renderer.c` | 着色器和绘制逻辑 |
| **主控制流程** | `ff_ffplay.c` | 协调所有模块 |

---

## 第三部分：详细流程解析

### 3.1 第一步：视频数据准备

#### 3.1.1 数据从哪里来？

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c
int read_thread(void *arg) {
    VideoState *is = arg;

    // 打开视频文件/网络流
    avformat_open_input(&ic, is->filename, ...);

    // 循环读取数据包
    while (!is->abort_request) {
        AVPacket pkt;
        av_read_frame(ic, &pkt);  // 读取一个数据包

        if (pkt.stream_index == video_stream) {
            // 放入视频队列
            packet_queue_put(&is->videoq, &pkt);
        }
    }
}
```

**通俗解释**：
- `av_read_frame` = 从快递箱里拿一个包裹
- `packet_queue_put` = 把包裹放到待处理队列

#### 3.1.2 数据是什么样的？

```
原始数据包（AVPacket）：
┌──────────────────────────────────────┐
│ H.264 压缩数据                        │
│ [NALU Header][压缩的视频帧数据]      │
│                                      │
│ 大小：几 KB 到几百 KB                │
│ 格式：H.264/H.265 等编码格式          │
└──────────────────────────────────────┘
```

### 3.2 第二步：MediaCodec 硬件解码

#### 3.2.1 MediaCodec 初始化

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/android/pipeline/
// ffpipenode_android_mediacodec_vdec.c

// 1. 创建 MediaCodec 实例
jobject mediacodec = SDL_AMediaCodecJava_createByCodecName(
    env, "OMX.google.h264.decoder"  // 解码器名称
);

// 2. 配置 MediaCodec
MediaFormat *format = ...;
format.setInteger("width", 1920);
format.setInteger("height", 1080);

// 3. 设置 Surface（重要！）
SDL_AMediaCodec_configure(
    mediacodec,
    format,
    output_surface,  // OpenGL 的 Surface
    NULL,
    0
);

// 4. 启动解码器
SDL_AMediaCodec_start(mediacodec);
```

**关键点：设置 Surface**

这一步非常重要！它告诉 MediaCodec：
> "你解码出来的视频帧，直接输出到这个 Surface 上，我要用 OpenGL 处理它"

**不设置 Surface 的话**：
- MediaCodec 会把解码后的数据放在 CPU 内存
- 需要手动拷贝到 GPU（慢）

**设置 Surface 的好处**：
- MediaCodec 直接把数据放到 GPU 内存
- 零拷贝，速度快

#### 3.2.2 送数据给 MediaCodec

```c
// 循环：从队列取数据 → 送给 MediaCodec
while (解码未结束) {
    // 1. 获取一个输入缓冲区
    int input_buffer_index = SDL_AMediaCodec_dequeueInputBuffer(
        mediacodec,
        timeout_us  // 超时时间
    );

    if (input_buffer_index >= 0) {
        // 2. 获取缓冲区指针
        uint8_t *input_buffer = SDL_AMediaCodec_getInputBuffer(
            mediacodec,
            input_buffer_index
        );

        // 3. 从队列取数据包
        AVPacket pkt;
        packet_queue_get(&is->videoq, &pkt);

        // 4. 拷贝数据到缓冲区
        memcpy(input_buffer, pkt.data, pkt.size);

        // 5. 通知 MediaCodec 开始解码
        SDL_AMediaCodec_queueInputBuffer(
            mediacodec,
            input_buffer_index,
            0,              // offset
            pkt.size,       // size
            pkt.pts,        // 时间戳
            0               // flags
        );
    }
}
```

**通俗解释**：
1. `dequeueInputBuffer` = 问："有空位吗？"
2. `getInputBuffer` = 得到空位的地址
3. `memcpy` = 把数据放进去
4. `queueInputBuffer` = 说："开始处理吧！"

#### 3.2.3 MediaCodec 解码过程（内部）

```
MediaCodec 内部工作流程：
┌─────────────────────────────────────────┐
│ 输入：H.264 压缩数据                     │
│   ↓                                     │
│ [硬件解码器]                             │
│   - 熵解码                               │
│   - 反量化                               │
│   - 反变换                               │
│   - 运动补偿                             │
│   ↓                                     │
│ 输出：YUV420 原始视频帧                  │
│   - 直接写入 GPU 内存                    │
│   - 作为 OpenGL 纹理可用                 │
└─────────────────────────────────────────┘
```

**重要概念：零拷贝**

传统流程（慢）：
```
MediaCodec 解码 → CPU 内存 → 拷贝到 GPU 内存 → OpenGL
```

零拷贝流程（快）：
```
MediaCodec 解码 → 直接写入 GPU 内存 → OpenGL
```

### 3.3 第三步：OpenGL 环境准备

#### 3.3.1 什么是 EGL？

**EGL** = OpenGL 和 Android 窗口系统之间的桥梁

**通俗理解**：
- **OpenGL** = 画家
- **Surface** = 画布
- **EGL** = 把画家和画布连接起来的管家

#### 3.3.2 EGL 初始化流程

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/ijksdl_egl.c

typedef struct IJK_EGL_Opaque {
    EGLDisplay display;  // 显示设备
    EGLSurface surface;  // 渲染表面
    EGLContext context;  // OpenGL 上下文
    ANativeWindow *window;  // Android 窗口
} IJK_EGL_Opaque;

// 1. 获取 EGL 显示设备
EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
eglInitialize(display, NULL, NULL);

// 2. 选择配置（颜色格式、深度等）
EGLConfig config;
EGLint configAttribs[] = {
    EGL_RENDERABLE_TYPE, EGL_OPENGL_ES2_BIT,  // OpenGL ES 2.0
    EGL_SURFACE_TYPE, EGL_WINDOW_BIT,         // 窗口类型
    EGL_RED_SIZE, 8,      // 红色 8 位
    EGL_GREEN_SIZE, 8,    // 绿色 8 位
    EGL_BLUE_SIZE, 8,     // 蓝色 8 位
    EGL_ALPHA_SIZE, 8,    // 透明度 8 位
    EGL_NONE
};
eglChooseConfig(display, configAttribs, &config, 1, &numConfigs);

// 3. 创建 Window Surface（画布）
EGLSurface surface = eglCreateWindowSurface(
    display,
    config,
    native_window,  // 来自 SurfaceView 的窗口
    NULL
);

// 4. 创建 OpenGL 上下文
EGLint contextAttribs[] = {
    EGL_CONTEXT_CLIENT_VERSION, 2,  // OpenGL ES 2.0
    EGL_NONE
};
EGLContext context = eglCreateContext(
    display,
    config,
    EGL_NO_CONTEXT,  // 不共享上下文
    contextAttribs
);

// 5. 绑定上下文（开始使用 OpenGL）
eglMakeCurrent(display, surface, surface, context);
```

**每一步的作用**：

1. **eglGetDisplay**：连接到显示系统
   - 类比：找到画展的展厅

2. **eglChooseConfig**：选择渲染配置
   - 类比：选择画布类型（水彩纸、油画布等）

3. **eglCreateWindowSurface**：创建渲染表面
   - 类比：把画布挂到墙上

4. **eglCreateContext**：创建 OpenGL 上下文
   - 类比：准备画笔、颜料等工具

5. **eglMakeCurrent**：激活上下文
   - 类比：开始作画

#### 3.3.3 创建 Surface Texture

**SurfaceTexture** = 连接 MediaCodec 和 OpenGL 的纽带

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/android/
// ijksdl_vout_android_nativewindow.c

// 1. 生成 OpenGL 纹理 ID
GLuint texture_id;
glGenTextures(1, &texture_id);

// 2. 绑定纹理
glBindTexture(GL_TEXTURE_EXTERNAL_OES, texture_id);

// 3. 设置纹理参数
glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_EXTERNAL_OES, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);

// 4. 创建 SurfaceTexture（Java 层）
jobject surface_texture = env->NewObject(
    surfaceTextureClass,
    constructor,
    texture_id  // 关联 OpenGL 纹理
);

// 5. 从 SurfaceTexture 获取 Surface
jobject surface = env->CallObjectMethod(
    surface_texture,
    getSurfaceMethod
);

// 这个 surface 就是传给 MediaCodec 的！
```

**数据流向**：
```
MediaCodec 解码
    ↓
Surface（关联到 SurfaceTexture）
    ↓
OpenGL 纹理（texture_id）
    ↓
OpenGL 渲染管线
    ↓
屏幕
```

**关键理解**：
- **SurfaceTexture** 是一个特殊的 Surface
- MediaCodec 输出到这个 Surface
- OpenGL 从对应的纹理 ID 读取数据
- 中间无需 CPU 拷贝！

### 3.4 第四步：OpenGL 渲染

#### 3.4.1 OpenGL 渲染基础

**OpenGL 渲染 = 绘制三角形**

所有 OpenGL 图形都由三角形组成。渲染一个视频帧：
```
屏幕矩形 = 2个三角形
```

```
矩形拆分成三角形：
┌──────────┐
│         /│    三角形1：左上、左下、右上
│       /  │    三角形2：右上、左下、右下
│     /    │
│   /      │
│ /        │
└──────────┘
```

#### 3.4.2 顶点数据

**顶点**定义三角形的位置和纹理坐标：

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/gles2/renderer.c

// 顶点坐标（矩形的4个角）
GLfloat vertices[] = {
    // 位置 (x, y)      纹理坐标 (u, v)
    -1.0f,  1.0f,      0.0f, 1.0f,  // 左上
    -1.0f, -1.0f,      0.0f, 0.0f,  // 左下
     1.0f,  1.0f,      1.0f, 1.0f,  // 右上
     1.0f, -1.0f,      1.0f, 0.0f   // 右下
};
```

**坐标系统**：
```
OpenGL 坐标（-1 到 1）：
    (-1, 1) ┌────────┐ (1, 1)
            │        │
            │        │
            │        │
   (-1, -1) └────────┘ (1, -1)

纹理坐标（0 到 1）：
    (0, 1) ┌────────┐ (1, 1)
           │  图像  │
           │        │
           │        │
    (0, 0) └────────┘ (1, 0)
```

#### 3.4.3 着色器程序

**着色器** = GPU 上运行的小程序，处理每个顶点和像素

**顶点着色器**（处理位置）：
```glsl
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/gles2/vsh/mvp.vsh.c
attribute vec4 aPosition;     // 输入：顶点位置
attribute vec2 aTextureCoord; // 输入：纹理坐标
varying vec2 vTextureCoord;   // 输出：传给片段着色器

void main() {
    gl_Position = aPosition;        // 设置顶点位置
    vTextureCoord = aTextureCoord;  // 传递纹理坐标
}
```

**片段着色器**（处理颜色）：
```glsl
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/gles2/fsh/yuv420p.fsh.c
precision mediump float;

varying vec2 vTextureCoord;      // 输入：纹理坐标
uniform sampler2D uTextureY;     // Y 分量纹理
uniform sampler2D uTextureU;     // U 分量纹理
uniform sampler2D uTextureV;     // V 分量纹理

void main() {
    float y = texture2D(uTextureY, vTextureCoord).r;
    float u = texture2D(uTextureU, vTextureCoord).r - 0.5;
    float v = texture2D(uTextureV, vTextureCoord).r - 0.5;

    // YUV 转 RGB
    float r = y + 1.402 * v;
    float g = y - 0.344 * u - 0.714 * v;
    float b = y + 1.772 * u;

    gl_FragColor = vec4(r, g, b, 1.0);  // 输出颜色
}
```

**通俗理解着色器**：
- **顶点着色器** = 确定画在哪里
- **片段着色器** = 确定画什么颜色

#### 3.4.4 完整渲染流程

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/gles2/renderer.c

void IJK_GLES2_Renderer_renderOverlay(
    IJK_GLES2_Renderer *renderer,
    SDL_VoutOverlay *overlay
) {
    // 1. 清空画布
    glClear(GL_COLOR_BUFFER_BIT);

    // 2. 使用着色器程序
    glUseProgram(renderer->program);

    // 3. 更新纹理数据
    // 对于 MediaCodec：
    // - 调用 surfaceTexture.updateTexImage()
    // - 视频帧自动更新到纹理

    // 对于软件解码（YUV 数据）：
    if (overlay->format == SDL_FCC_I420) {
        // 上传 Y 分量
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, renderer->textures[0]);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE,
                     overlay->w, overlay->h, 0,
                     GL_LUMINANCE, GL_UNSIGNED_BYTE,
                     overlay->pixels[0]);  // Y 数据

        // 上传 U 分量
        glActiveTexture(GL_TEXTURE1);
        glBindTexture(GL_TEXTURE_2D, renderer->textures[1]);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE,
                     overlay->w/2, overlay->h/2, 0,
                     GL_LUMINANCE, GL_UNSIGNED_BYTE,
                     overlay->pixels[1]);  // U 数据

        // 上传 V 分量
        glActiveTexture(GL_TEXTURE2);
        glBindTexture(GL_TEXTURE_2D, renderer->textures[2]);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE,
                     overlay->w/2, overlay->h/2, 0,
                     GL_LUMINANCE, GL_UNSIGNED_BYTE,
                     overlay->pixels[2]);  // V 数据
    }

    // 4. 设置着色器变量
    glUniform1i(renderer->uniform_y, 0);  // Y 纹理 -> 纹理单元0
    glUniform1i(renderer->uniform_u, 1);  // U 纹理 -> 纹理单元1
    glUniform1i(renderer->uniform_v, 2);  // V 纹理 -> 纹理单元2

    // 5. 绑定顶点数据
    glVertexAttribPointer(
        renderer->attribute_position,  // 位置属性
        2,                            // 2个分量 (x, y)
        GL_FLOAT,                     // float 类型
        GL_FALSE,                     // 不归一化
        4 * sizeof(GLfloat),          // 步长
        vertices                      // 数据指针
    );
    glEnableVertexAttribArray(renderer->attribute_position);

    glVertexAttribPointer(
        renderer->attribute_texcoord, // 纹理坐标属性
        2,                           // 2个分量 (u, v)
        GL_FLOAT,
        GL_FALSE,
        4 * sizeof(GLfloat),
        vertices + 2                 // 偏移到纹理坐标
    );
    glEnableVertexAttribArray(renderer->attribute_texcoord);

    // 6. 绘制！
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);  // 绘制4个顶点

    // 7. 交换缓冲区（显示到屏幕）
    eglSwapBuffers(display, surface);
}
```

**绘制过程详解**：

1. **glClear**：清空画布（填充背景色）

2. **glUseProgram**：激活着色器程序
   - 类比：拿起画笔

3. **更新纹理**：
   - MediaCodec 方式：自动更新（零拷贝）
   - 软件解码：手动上传数据

4. **glUniform**：设置着色器变量
   - 告诉着色器从哪个纹理读取数据

5. **glVertexAttribPointer**：绑定顶点数据
   - 告诉 OpenGL 顶点在哪里

6. **glDrawArrays**：执行绘制
   - GPU 处理每个顶点和像素

7. **eglSwapBuffers**：显示结果
   - 类比：翻开画本的新一页

### 3.5 MediaCodec + OpenGL 的完整流程

#### 3.5.1 初始化阶段

```
第1步：创建 SurfaceView
  ↓
第2步：初始化 EGL
  - eglGetDisplay
  - eglCreateWindowSurface
  - eglCreateContext
  ↓
第3步：创建 OpenGL 纹理
  - glGenTextures
  ↓
第4步：创建 SurfaceTexture
  - new SurfaceTexture(textureId)
  ↓
第5步：初始化 MediaCodec
  - createByCodecName
  - configure(format, surface, ...)
  - start()
```

#### 3.5.2 渲染循环

```c
// 主循环（在 video_refresh 中）
while (播放中) {
    // A. 送数据给 MediaCodec
    if (有新的数据包) {
        int input_idx = dequeueInputBuffer(mediacodec);
        if (input_idx >= 0) {
            // 拷贝数据
            queueInputBuffer(mediacodec, input_idx, ...);
        }
    }

    // B. MediaCodec 解码（异步，后台进行）

    // C. 获取解码结果
    int output_idx = dequeueOutputBuffer(mediacodec);
    if (output_idx >= 0) {
        // D. 更新 SurfaceTexture
        // MediaCodec 已经把数据写入 Surface
        // 只需要通知 SurfaceTexture 更新
        surfaceTexture.updateTexImage();

        // E. OpenGL 渲染
        glClear(GL_COLOR_BUFFER_BIT);
        glUseProgram(program);
        glBindTexture(GL_TEXTURE_EXTERNAL_OES, texture_id);
        // ... 设置顶点、着色器参数
        glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

        // F. 显示到屏幕
        eglSwapBuffers(display, surface);

        // G. 释放 MediaCodec 输出缓冲区
        releaseOutputBuffer(mediacodec, output_idx, false);
    }
}
```

---

## 第四部分：关键代码分析

### 4.1 video_refresh 函数（主控制流程）

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c

static void video_refresh(void *opaque, double *remaining_time) {
    VideoState *is = opaque;

    // 1. 从帧队列获取视频帧
    Frame *vp = frame_queue_peek_readable(&is->pictq);
    if (!vp) return;

    // 2. 计算显示时间（音视频同步）
    double time = av_gettime_relative() / 1000000.0;
    if (time < vp->pts) {
        *remaining_time = vp->pts - time;
        return;  // 还没到显示时间
    }

    // 3. 显示视频帧
    video_image_display(is);  // 调用 OpenGL 渲染

    // 4. 释放帧
    frame_queue_next(&is->pictq);
}

static void video_image_display(VideoState *is) {
    Frame *vp = frame_queue_peek_last(&is->pictq);

    // 调用 SDL 显示
    SDL_VoutDisplayYUVOverlay(
        is->vout,          // 视频输出对象
        vp->bmp            // 视频帧（可能是 MediaCodec 的）
    );
}
```

### 4.2 SDL_VoutDisplayYUVOverlay（分发到渲染器）

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/ijksdl_vout.c

int SDL_VoutDisplayYUVOverlay(
    SDL_Vout *vout,
    SDL_VoutOverlay *overlay
) {
    // 调用具体实现的 display 函数
    return vout->display_overlay(vout, overlay);
}
```

### 4.3 Android Native Window 渲染

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/android/
// ijksdl_vout_android_nativewindow.c

static int func_display_overlay_l(
    SDL_Vout *vout,
    SDL_VoutOverlay *overlay
) {
    SDL_Vout_Opaque *opaque = vout->opaque;

    // 检查帧类型
    if (overlay->format == SDL_FCC__AMC) {
        // MediaCodec 硬件解码帧
        SDL_VoutOverlayAMediaCodec_releaseFrame_l(
            overlay,
            NULL,
            true  // render = true，显示到 Surface
        );
    } else {
        // 软件解码帧，使用 OpenGL 渲染
        IJK_EGL_display(
            opaque->egl,
            opaque->native_window,
            overlay
        );
    }

    return 0;
}
```

### 4.4 MediaCodec 帧释放（触发渲染）

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/android/
// ijksdl_vout_overlay_android_mediacodec.c

int SDL_VoutOverlayAMediaCodec_releaseFrame_l(
    SDL_VoutOverlay *overlay,
    SDL_AMediaCodecBufferProxy *proxy,
    bool render
) {
    // 重点：render = true 时，MediaCodec 会把帧渲染到 Surface
    SDL_AMediaCodec_releaseOutputBuffer(
        opaque->acodec,
        opaque->buffer_index,
        render  // true = 渲染到 Surface
    );

    // 这一步会：
    // 1. 通知 MediaCodec 释放输出缓冲区
    // 2. 如果 render=true，自动渲染到 Surface
    // 3. SurfaceTexture 收到新帧通知

    return 0;
}
```

### 4.5 OpenGL 渲染（软件解码）

```c
// A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/ijksdl_egl.c

int IJK_EGL_display(
    IJK_EGL *egl,
    ANativeWindow *native_window,
    SDL_VoutOverlay *overlay
) {
    IJK_EGL_Opaque *opaque = egl->opaque;

    // 1. 激活 OpenGL 上下文
    eglMakeCurrent(
        opaque->display,
        opaque->surface,
        opaque->surface,
        opaque->context
    );

    // 2. 调用 GLES2 渲染器
    IJK_GLES2_Renderer_renderOverlay(
        opaque->renderer,
        overlay
    );

    // 3. 交换缓冲区（显示）
    eglSwapBuffers(opaque->display, opaque->surface);

    return 0;
}
```

---

## 第五部分：实际例子

### 5.1 播放一个 H.264 视频的完整流程

假设我们要播放 `http://example.com/video.mp4`

#### 步骤1：初始化播放器

```java
// Java 层
IjkMediaPlayer player = new IjkMediaPlayer();
player.setDataSource("http://example.com/video.mp4");
player.setSurface(surfaceView.getHolder().getSurface());
player.prepareAsync();
```

#### 步骤2：准备阶段（Native 层）

```c
// 1. 打开网络流
avformat_open_input(&ic, "http://example.com/video.mp4", ...);

// 2. 查找流信息
avformat_find_stream_info(ic, ...);
// 发现：视频流 H.264, 1920x1080, 30fps

// 3. 初始化 MediaCodec
mediacodec = createDecoderByType("video/avc");  // H.264
configure(mediacodec, format, surface, ...);
start(mediacodec);

// 4. 初始化 OpenGL
eglCreateContext(...);
glGenTextures(1, &texture_id);
surfaceTexture = new SurfaceTexture(texture_id);
```

#### 步骤3：播放循环

```c
// 线程1：读取数据
while (播放中) {
    av_read_frame(ic, &pkt);  // 读取一个 H.264 数据包
    packet_queue_put(&videoq, &pkt);
}

// 线程2：MediaCodec 解码
while (播放中) {
    // A. 送数据
    input_idx = dequeueInputBuffer(mediacodec, timeout);
    if (input_idx >= 0) {
        buffer = getInputBuffer(mediacodec, input_idx);
        packet_queue_get(&videoq, &pkt);
        memcpy(buffer, pkt.data, pkt.size);
        queueInputBuffer(mediacodec, input_idx, ...);
    }

    // B. 取结果
    output_idx = dequeueOutputBuffer(mediacodec, &info, timeout);
    if (output_idx >= 0) {
        // 放入显示队列
        frame_queue_push(&pictq, output_idx, info.presentationTimeUs);
    }
}

// 线程3：渲染
while (播放中) {
    // 定时器触发 video_refresh
    video_refresh(is, &remaining_time);

    // video_refresh 内部：
    {
        Frame *vp = frame_queue_peek(&pictq);

        // 等待合适的显示时间
        if (当前时间 < vp->pts) {
            usleep(...);
            continue;
        }

        // 渲染
        if (MediaCodec 解码) {
            // 调用 releaseOutputBuffer(render=true)
            // MediaCodec 自动渲染到 Surface
            releaseOutputBuffer(mediacodec, vp->buffer_index, true);

            // 更新 SurfaceTexture
            surfaceTexture.updateTexImage();
        }

        // OpenGL 绘制
        glClear(...);
        glBindTexture(GL_TEXTURE_EXTERNAL_OES, texture_id);
        glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
        eglSwapBuffers(...);

        frame_queue_next(&pictq);
    }
}
```

### 5.2 数据流向图

```
完整数据流向（从网络到屏幕）：

[网络]
  ↓ HTTP
[压缩的 H.264 数据包]
  ↓ av_read_frame
[PacketQueue - videoq]
  ↓ packet_queue_get
[MediaCodec Input Buffer]
  ↓ queueInputBuffer
[MediaCodec 硬件解码器]
  ↓ dequeueOutputBuffer
[解码后的 YUV 帧 - 在 GPU 内存]
  ↓ releaseOutputBuffer(render=true)
[Surface - 关联到 SurfaceTexture]
  ↓ updateTexImage
[OpenGL Texture - GL_TEXTURE_EXTERNAL_OES]
  ↓ glDrawArrays
[EGLSurface - 渲染结果]
  ↓ eglSwapBuffers
[屏幕显示]
```

### 5.3 时间线示例

假设视频是 30fps（每帧 33.3ms）：

```
时间轴（毫秒）：
0ms    33ms   66ms   99ms   132ms  ...
│      │      │      │      │
读取   读取   读取   读取   读取   ... (读取线程)
│      │      │      │
└解码─┘└解码─┘└解码─┘└解码─┘      ... (MediaCodec)
      │      │      │      │
      └渲染─┘└渲染─┘└渲染─┘└渲染   ... (渲染线程)

详细过程：
0ms:   读取数据包1
       → MediaCodec 开始解码包1
10ms:  MediaCodec 完成解码包1
       → 帧1 进入显示队列
33ms:  video_refresh 检查时间
       → 显示帧1（releaseOutputBuffer + OpenGL）
33ms:  读取数据包2
       → MediaCodec 开始解码包2
...
```

---

## 第六部分：常见问题

### Q1: 为什么不直接用 MediaCodec 渲染，还要 OpenGL？

**答**：MediaCodec 可以直接渲染，但 OpenGL 提供了：
1. **更多控制**：可以添加滤镜、水印、旋转等效果
2. **统一接口**：软件解码和硬件解码用同一套渲染代码
3. **灵活性**：可以同时显示多个视频、画中画等

### Q2: SurfaceTexture 和普通 Texture 有什么区别？

**答**：
- **普通 Texture**：需要手动上传数据（glTexImage2D）
- **SurfaceTexture**：
  - 可以作为 MediaCodec 的输出目标
  - 数据由 MediaCodec 直接写入 GPU
  - 零拷贝，性能更好

### Q3: GL_TEXTURE_EXTERNAL_OES 是什么？

**答**：
- 这是 OpenGL ES 的扩展类型
- 专门用于外部数据源（如相机、MediaCodec）
- 纹理坐标可能需要变换矩阵

### Q4: YUV 是什么？为什么要转 RGB？

**答**：
- **YUV**：视频编码常用格式
  - Y = 亮度
  - U, V = 色度
  - 节省空间（色度可以降采样）
- **RGB**：屏幕显示格式
  - R = 红色
  - G = 绿色
  - B = 蓝色
- **转换**：在片段着色器中进行（GPU 计算）

### Q5: 如何添加滤镜效果？

**答**：修改片段着色器

例如，添加黑白效果：
```glsl
void main() {
    vec4 color = texture2D(uTexture, vTextureCoord);
    float gray = dot(color.rgb, vec3(0.299, 0.587, 0.114));
    gl_FragColor = vec4(gray, gray, gray, color.a);
}
```

---

## 第七部分：性能优化

### 7.1 零拷贝技术

**传统方式**（慢）：
```
MediaCodec → CPU 内存 → GPU 内存 → 渲染
           [拷贝1]    [拷贝2]
```

**零拷贝方式**（快）：
```
MediaCodec → GPU 内存 → 渲染
           [直接写入]
```

**实现关键**：
- MediaCodec 配置时设置 Surface
- 使用 SurfaceTexture
- 使用 GL_TEXTURE_EXTERNAL_OES

### 7.2 多缓冲技术

**双缓冲**：
```
[前台缓冲区] - 正在显示
[后台缓冲区] - 正在绘制
      ↓
  交换（eglSwapBuffers）
      ↓
[前台缓冲区] - 正在显示（新内容）
[后台缓冲区] - 正在绘制（下一帧）
```

**好处**：
- 避免画面撕裂
- 渲染和显示并行

### 7.3 异步处理

```
读取线程 ──→ 数据队列
              ↓
解码线程 ──→ MediaCodec ──→ 帧队列
                             ↓
渲染线程 ──→ OpenGL ──→ 屏幕
```

每个环节独立运行，互不阻塞。

---

## 第八部分：调试技巧

### 8.1 查看 OpenGL 错误

```c
void checkGLError(const char *op) {
    GLint error = glGetError();
    if (error != GL_NO_ERROR) {
        ALOGE("OpenGL error after %s: 0x%x", op, error);
    }
}

// 使用
glClear(GL_COLOR_BUFFER_BIT);
checkGLError("glClear");
```

### 8.2 MediaCodec 状态监控

```c
// 记录 MediaCodec 统计信息
int input_count = 0;
int output_count = 0;
int64_t total_decode_time = 0;

// 解码时
int64_t start = av_gettime();
queueInputBuffer(...);
input_count++;

dequeueOutputBuffer(...);
output_count++;
total_decode_time += (av_gettime() - start);

// 定期打印
ALOGI("MediaCodec stats: input=%d, output=%d, avg_time=%lld us",
      input_count, output_count, total_decode_time / output_count);
```

### 8.3 渲染性能分析

```c
// 计算 FPS
int frame_count = 0;
int64_t last_time = av_gettime();

void video_refresh(...) {
    frame_count++;

    int64_t now = av_gettime();
    if (now - last_time > 1000000) {  // 每秒
        float fps = frame_count * 1000000.0 / (now - last_time);
        ALOGI("Render FPS: %.2f", fps);

        frame_count = 0;
        last_time = now;
    }

    // ... 正常渲染代码
}
```

---

## 总结

### 核心流程回顾

```
[数据读取] → [MediaCodec 解码] → [OpenGL 渲染] → [屏幕显示]
     ↓              ↓                 ↓              ↓
  网络/文件      GPU 内存          SurfaceTexture   用户看到
  压缩数据      YUV 帧           OpenGL 纹理      视频画面
```

### 关键技术点

1. **MediaCodec**：硬件加速解码，快速且省电
2. **Surface/SurfaceTexture**：连接 MediaCodec 和 OpenGL
3. **OpenGL ES**：GPU 渲染，支持特效
4. **EGL**：连接 OpenGL 和 Android 窗口系统
5. **零拷贝**：数据直接在 GPU 内存流转
6. **多线程**：读取、解码、渲染并行处理

### 学习路径建议

1. **基础阶段**：
   - 学习 Android View 系统
   - 了解 SurfaceView/TextureView

2. **进阶阶段**：
   - 学习 OpenGL ES 基础
   - 学习 MediaCodec API

3. **高级阶段**：
   - 研究 ijkplayer 源码
   - 实现自定义滤镜效果

---

## 参考资源

### 官方文档
- [Android MediaCodec](https://developer.android.com/reference/android/media/MediaCodec)
- [OpenGL ES](https://www.khronos.org/opengles/)
- [EGL](https://www.khronos.org/egl/)

### ijkplayer 源码位置
- **MediaCodec 解码**：`A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/android/pipeline/ffpipenode_android_mediacodec_vdec.c`
- **OpenGL 渲染**：`A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/gles2/renderer.c`
- **EGL 管理**：`A4ijkplayer/src/main/cpp/ijkmedia/ijksdl/ijksdl_egl.c`
- **主控制流程**：`A4ijkplayer/src/main/cpp/ijkmedia/ijkplayer/ff_ffplay.c`

### 学习示例
建议按以下顺序阅读代码：
1. `ff_ffplay.c` - 理解整体流程
2. `ijksdl_egl.c` - 理解 OpenGL 初始化
3. `renderer.c` - 理解具体渲染逻辑
4. `ffpipenode_android_mediacodec_vdec.c` - 理解 MediaCodec 集成

---

**文档版本**：v1.0
**创建日期**：2025-01-04
**适用对象**：对 OpenGL 和 Android 渲染不熟悉的开发者
