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

```
┌─────────────────────────────────────────────────────────────────────┐
│                        View 渲染模式                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   普通 View (View)                                                  │
│   ├── 渲染方式: 软件渲染 / 硬件渲染                                  │
│   ├── 绘制: 主线程 Canvas.draw()                                    │
│   ├── 层级: 在 View 层级中                                          │
│   └── 16ms 限制: 必须在 16ms内完成                                  │
│                                                                      │
│   SurfaceView                                                       │
│   ├── 渲染方式: GPU 合成                                            │
│   ├── 绘制: 子线程 SurfaceCanvas                                    │
│   ├── 层级: 独立窗口，地位 View 层级                                │
│   └── 特点: 可以在子线程绘制                                         │
│                                                                      │
│   TextureView                                                        │
│   ├── 渲染方式: 硬件加速                                            │
│   ├── 绘制: 子线程 TextureRegistry                                  │
│   ├── 层级: 在 View 层级中                                          │
│   └── 特点: 支持变换(旋转/缩放)、动画                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. SurfaceView 原理与使用

### SurfaceView 原理

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SurfaceView 原理                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    SurfaceView 创建:                                                │
│    1. 创建独立 Surface (双缓冲)                                     │
│    2. 创建独立 Canvas                                                │
│    3. 渲染到 Surface 的子线程中                                     │
│                                                                      │
│    双缓冲机制:                                                      │
│    ┌───────────────┐    ┌───────────────┐                          │
│    │  Back Buffer  │◀───│  绘制线程     │                          │
│    │  (后缓冲区)    │    │  (我们的线程)  │                          │
│    └───────┬───────┘    └───────────────┘                          │
│            │                                                         │
│            ▼                                                         │
│    ┌───────────────┐    ┌───────────────┐                          │
│    │ Front Buffer  │───▶│   显示设备    │                          │
│    │  (前缓冲区)    │    │   (屏幕)      │                          │
│    └───────────────┘    └───────────────┘                          │
│                                                                      │
│    SurfaceView 优势:                                               │
│    - 子线程渲染，不阻塞主线程                                        │
│    - 独立 Surface，双缓冲无闪烁                                      │
│    - 适合视频播放、相机预览、游戏                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

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

```
┌─────────────────────────────────────────────────────────────────────┐
│                     TextureView 原理                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    TextureView:                                                     │
│    - 需要硬件加速                                                   │
│    - 作为普通 View 参与 View 层级                                   │
│    - 使用 SurfaceTexture 作为缓冲区                                │
│    - 支持变换(旋转、缩放、透明度)                                   │
│                                                                      │
│    渲染流程:                                                        │
│    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐             │
│    │ Surface     │──▶│ Surface    │──▶│  GPU        │             │
│    │ Texture    │   │ (Buffer)   │   │  合成       │             │
│    └─────────────┘   └─────────────┘   └──────┬──────┘             │
│                                                 │                    │
│                                                 ▼                    │
│                                          ┌─────────────┐            │
│                                          │  TextureView│            │
│                                          │  (View 树)  │            │
│                                          └─────────────┘            │
│                                                                      │
│    TextureView 优势:                                               │
│    - 支持动画和变换                                                 │
│    - 可以在主线程更新                                              │
│    - 占用内存少                                                    │
│                                                                      │
│    TextureView 劣势:                                               │
│    - 需要硬件加速                                                  │
│    - 不支持同时多个使用                                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

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

```
选择原则:
─────────────────────────────────────────
高性能/低延迟 → SurfaceView
需要变换/动画 → TextureView
相机/视频播放 → SurfaceView
视频通话 → TextureView
─────────────────────────────────────────
```

---

**相关文章**：
- [View 绘制流程深度解析](./android-view-drawing-process.md)
- [View 性能优化与卡顿分析](./android-view-performance-optimization.md)
