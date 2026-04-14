---
title: "SurfaceView 与 TextureView 完全解析"
date: 2023-11-18 02:24:57
description: "视频播放与相机预览的最佳选择 View 家族对比"
comments: true
toc: true
categories:
  - Android
  - Graphics
tags:
  - Android
  - SurfaceView
  - TextureView
  - Graphics
---

# SurfaceView 与 TextureView 完全解析

> 视频播放与相机预览的最佳选择

## 目录

1. [View 家族对比](#1-view-家族对比)
2. [SurfaceView 原理与使用](#2-surfaceview-原理与使用)
3. [TextureView 原理与使用](#3-textureview-原理与使用)
4. [性能对比](#4-性能对比)
5. [选择指南](#5-选择指南)

---

## 1. View 家族对比

### View 渲染模式

{% mermaid flowchart TB %}
subgraph V["普通 View (View)"]
V1["渲染方式: 软件渲染 / 硬件渲染"]
V2["绘制: 主线程 Canvas.draw()"]
V3["层级: 在 View 层级中"]
V4["16ms 限制: 必须在 16ms 内完成"]
end
subgraph S["SurfaceView"]
S1["渲染方式: GPU 合成"]
S2["绘制: 子线程 SurfaceCanvas"]
S3["层级: 独立窗口，高于 View 层级"]
S4["特点: 可以在子线程绘制"]
end
subgraph T["TextureView"]
T1["渲染方式: 硬件加速"]
T2["绘制: 子线程 / SurfaceTexture"]
T3["层级: 在 View 层级中"]
T4["特点: 支持旋转、缩放、动画"]
end
{% endmermaid %}

---

## 2. SurfaceView 原理与使用

### SurfaceView 原理

{% mermaid flowchart LR %}
Create["SurfaceView 创建"] --> C1["1. 创建独立 Surface（双缓冲）"]
Create --> C2["2. 创建独立 Canvas"]
Create --> C3["3. 在子线程中渲染"]
DrawThread["绘制线程"] --> Back["Back Buffer<br/>(后缓冲区)"]
Back --> Front["Front Buffer<br/>(前缓冲区)"]
Front --> Screen["显示设备 / 屏幕"]
Pros["优势"] --> P1["子线程渲染，不阻塞主线程"]
Pros --> P2["独立 Surface，双缓冲无闪烁"]
Pros --> P3["适合视频播放、相机预览、游戏"]
{% endmermaid %}

### SurfaceView 使用

```kotlin
class SurfaceViewDemo @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : SurfaceView(context, attrs), SurfaceHolder.Callback {
    
    private var drawThread: DrawThread? = null
    
    init {
        // 设置回调
        holder.addCallback(this)
        isFocusable = true
    }
    
    override fun surfaceCreated(holder: SurfaceHolder) {
        // Surface 创建完成，启动绘制线程
        drawThread = DrawThread(holder).apply {
            start()
        }
    }
    
    override fun surfaceChanged(holder: SurfaceHolder, format: Int, width: Int, height: Int) {
        // 尺寸变化
    }
    
    override fun surfaceDestroyed(holder: SurfaceHolder) {
        // 停止绘制线程
        drawThread?.running = false
        drawThread?.join()
    }
    
    // 绘制线程
    inner class DrawThread(private val surfaceHolder: SurfaceHolder) : Thread() {
        var running = true
        
        override fun run() {
            while (running) {
                var canvas: Canvas? = null
                try {
                    // 获取 Canvas (锁定 Surface)
                    canvas = surfaceHolder.lockCanvas()
                    
                    synchronized(surfaceHolder) {
                        // 绘制
                        drawFrame(canvas)
                    }
                } finally {
                    // 解锁并提交
                    canvas?.let {
                        surfaceHolder.unlockCanvasAndPost(it)
                    }
                }
            }
        }
        
        private fun drawFrame(canvas: Canvas) {
            canvas.drawColor(Color.BLACK)
            // 绘制内容
        }
    }
}
```

### 相机预览示例

```kotlin
class CameraPreview(context: Context) : SurfaceView(context), SurfaceHolder.Callback {
    
    private var camera: Camera? = null
    
    init {
        holder.addCallback(this)
    }
    
    override fun surfaceCreated(holder: SurfaceHolder) {
        camera = Camera.open().apply {
            setPreviewDisplay(holder)
            startPreview()
        }
    }
    
    override fun surfaceDestroyed(holder: SurfaceHolder) {
        camera?.apply {
            stopPreview()
            release()
        }
        camera = null
    }
}
```

---

## 3. TextureView 原理与使用

### TextureView 原理

{% mermaid flowchart LR %}
Texture["SurfaceTexture"] --> Buffer["Surface Buffer"]
Buffer --> GPU["GPU 合成"]
GPU --> ViewTree["TextureView<br/>(位于 View 树中)"]
Feature["TextureView 特性"] --> F1["需要硬件加速"]
Feature --> F2["参与普通 View 层级"]
Feature --> F3["支持旋转、缩放、透明度"]
Adv["优势"] --> A1["支持动画和变换"]
Adv --> A2["可以在主线程更新"]
Adv --> A3["占用内存少"]
Dis["劣势"] --> D1["需要硬件加速"]
Dis --> D2["不支持同时多个使用"]
{% endmermaid %}

### TextureView 使用

```kotlin
class TextureViewDemo @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : TextureView(context, attrs), SurfaceTextureListener {
    
    init {
        surfaceTextureListener = this
    }
    
    override fun onSurfaceTextureAvailable(surface: SurfaceTexture, width: Int, height: Int) {
        // SurfaceTexture 可用，启动绘制
        startPreview()
    }
    
    override fun onSurfaceTextureSizeChanged(surface: SurfaceTexture, width: Int, height: Int) {
        // 尺寸变化
    }
    
    override fun onSurfaceTextureDestroyed(surface: SurfaceTexture): Boolean {
        return true
    }
    
    override fun onSurfaceTextureUpdated(surface: SurfaceTexture) {
        // SurfaceTexture 更新
    }
    
    private fun startPreview() {
        // 使用 SurfaceTexture 创建 MediaPlayer 或 Camera
    }
    
    // 应用变换
    fun setRotation(degrees: Float) {
        rotation = degrees
    }
    
    fun setScale(scaleX: Float, scaleY: Float) {
        scaleX = scaleX
        scaleY = scaleY
    }
}
```

### 视频播放示例

```kotlin
class VideoPlayerView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : TextureView(context, attrs), TextureView.SurfaceTextureListener {
    
    private var mediaPlayer: MediaPlayer? = null
    private var surfaceTexture: SurfaceTexture? = null
    
    init {
        surfaceTextureListener = this
    }
    
    fun setVideoPath(path: String) {
        try {
            mediaPlayer?.release()
            mediaPlayer = MediaPlayer().apply {
                setDataSource(path)
                setSurface(Surface(surfaceTexture))
                setOnPreparedListener { mp ->
                    mp.start()
                }
                prepareAsync()
            }
        } catch (e: IOException) {
            e.printStackTrace()
        }
    }
    
    override fun onSurfaceTextureAvailable(surface: SurfaceTexture, width: Int, height: Int) {
        surfaceTexture = surface
    }
    
    override fun onSurfaceTextureDestroyed(surface: SurfaceTexture): Boolean {
        mediaPlayer?.release()
        return true
    }
}
```

---

## 4. 性能对比

### 性能对比表

| 指标 | SurfaceView | TextureView |
|------|-------------|-------------|
| 渲染线程 | 子线程 | 子线程 |
| 双缓冲 | ✅ 原生 | ✅ SurfaceTexture |
| 硬件加速 | ✅ GPU 合成 | ✅ GPU 合成 |
| 变换支持 | ❌ 不支持 | ✅ 旋转/缩放 |
| 动画支持 | ❌ 效果差 | ✅ 流畅 |
| 内存占用 | 低 | 中 |
| 耗电 | 低 | 中 |
| 截图困难 | ✅ 困难 | ✅ 简单 |

### 渲染性能测试

```
SurfaceView:
- 帧率: 稳定 60fps
- CPU: 较低 (子线程渲染)
- 延迟: 最低

TextureView:
- 帧率: 稳定 60fps
- CPU: 中等 (需要 GPU 合成)
- 延迟: 略高

普通 View:
- 帧率: 可能掉帧
- CPU: 高 (主线程渲染)
- 延迟: 高
```

---

## 5. 选择指南

### 决策表

| 场景 | 推荐 | 原因 |
|------|------|------|
| 视频播放 | SurfaceView | 低延迟、子线程渲染 |
| 相机预览 | SurfaceView | 实时性要求高 |
| 视频通话 | TextureView | 需要变换 |
| 游戏背景 | SurfaceView | 高帧率需求 |
| 需要旋转/缩放动画 | TextureView | 支持变换 |
| 需要截图 | TextureView | 简单实现 |

### 代码示例

```kotlin
// 视频播放 - SurfaceView
class VideoSurfaceView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : SurfaceView(context, attrs), MediaPlayer.OnPreparedListener {
    
    private var mediaPlayer: MediaPlayer? = null
    
    fun play(url: String) {
        mediaPlayer = MediaPlayer().apply {
            setDataSource(url)
            setDisplay(holder)
            setOnPreparedListener(this@VideoSurfaceView)
            prepareAsync()
        }
    }
    
    override fun onPrepared(mp: MediaPlayer?) {
        mp?.start()
    }
}

// 视频播放 - TextureView
class VideoTextureView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : TextureView(context, attrs), TextureView.SurfaceTextureListener {
    
    private var mediaPlayer: MediaPlayer? = null
    
    fun play(url: String) {
        // 类似实现，但可以使用变换
        setRotation(90f)  // 旋转
        setScaleX(1.5f)  // 缩放
    }
}
```

---

## 面试常问

| 问题 | 答案 |
|------|------|
| SurfaceView 为什么快？ | 子线程渲染，不阻塞主线程 |
| TextureView 有什么优势？ | 支持变换、动画 |
| 两者区别？ | SurfaceView 独立窗口，TextureView 在 View 树中 |

---

## 总结

{% mermaid flowchart TB %}
Choose["选择原则"] --> C1["高性能 / 低延迟 → SurfaceView"]
Choose --> C2["需要变换 / 动画 → TextureView"]
Choose --> C3["相机 / 视频播放 → SurfaceView"]
Choose --> C4["视频通话 → TextureView"]
{% endmermaid %}

---

**相关文章**：
- [View 绘制流程深度解析](./android-view-drawing-process.md)
- [View 性能优化与卡顿分析](./android-view-performance-optimization.md)
