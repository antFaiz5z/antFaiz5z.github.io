---
title: "View 性能优化与卡顿分析"
date: 2025-05-20 17:52:04
description: "打造 60fps 流畅应用的实战指南"
comments: true
toc: true
categories:
  - Android
  - Performance
tags:
  - Android
  - View
  - Performance
  - Jank
---

# View 性能优化与卡顿分析

> 打造 60fps 流畅应用的实战指南

## 目录

1. [性能指标与优化目标](#1-性能指标与优化目标)
2. [卡顿原因分析](#2-卡顿原因分析)
3. [布局优化](#3-布局优化)
4. [绘制优化](#4-绘制优化)
5. [内存优化](#5-内存优化)
6. [调试工具](#6-调试工具)

---

## 1. 性能指标与优化目标

### 60fps 原则

{% mermaid flowchart TB %}
FPS["60fps = 每帧 16ms"] --> Measure["1. UI 测量（measure）"]
Measure --> Layout["2. UI 布局（layout）"]
Layout --> Draw["3. UI 绘制（draw）"]
Draw --> GPU["4. GPU 渲染"]
GPU --> Jank["超过 16ms = 卡顿"]
{% endmermaid %}

### 卡顿原因

| 原因 | 占比 | 解决方案 |
|------|------|---------|
| 过度绘制 | 40% | 移除不必要的背景 |
| 布局复杂 | 25% | 优化布局层级 |
| 主线程耗时 | 20% | 异步处理 |
| 内存问题 | 15% | 减少 GC |

---

## 2. 卡顿原因分析

### 过度绘制 (Overdraw)

```kotlin
// ❌ 问题：多层背景导致过度绘制
<LinearLayout
    android:background="@color/white">  <!-- 绘制1 -->
    
    <FrameLayout
        android:background="@drawable/bg_card">  <!-- 绘制2 -->
        
        <TextView
            android:background="@color/white" />  <!-- 绘制3 -->
            
    </FrameLayout>
</LinearLayout>

// ✅ 解决：移除不必要的背景
<LinearLayout
    android:background="@color/white">  <!-- 只绘制1 -->
    
    <FrameLayout
        android:background="@drawable/bg_card">  <!-- 只绘制2 -->
            
    </FrameLayout>
</LinearLayout>
```

### 解决过度绘制

```kotlin
// 开启过度绘制检测
// Settings > Developer Options > Debug GPU overdraw > Show overdraw areas

// 代码中检查过度绘制
ViewDebug.startHotSwapListener(object : ViewDebug.HotSwapListener {
    override fun onHotSwapComplete() {
        // 每次重绘后回调
    }
})
```

---

## 3. 布局优化

### Hierarchy Viewer 检测

{% mermaid flowchart LR %}
subgraph Before["优化前层级"]
B1["LinearLayout"] --> B2["LinearLayout"] --> B3["FrameLayout"] --> B4["TextView"]
end
subgraph After["优化后层级"]
A1["LinearLayout（直接子 View）"] --> A2["TextView"]
end
{% endmermaid %}

### 使用 ConstraintLayout 扁平化

```kotlin
// ❌ 嵌套层级深
<LinearLayout>
    <LinearLayout>
        <LinearLayout>
            <TextView />
        </LinearLayout>
    </LinearLayout>
</LinearLayout>

// ✅ 扁平化
<androidx.constraintlayout.widget.ConstraintLayout>
    <TextView
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

### include 与 merge

```xml
<!-- layout_title.xml -->
<merge xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    
    <!-- merge 减少一层布局 -->
    <TextView
        android:id="@+id/title"
        app:layout_constraintTop_toTopOf="parent" />
        
</merge>

<!-- 使用 -->
<include layout="@layout/title" />
```

---

## 4. 绘制优化

### 减少 onDraw 调用

```kotlin
// ❌ 错误：每次 invalidate 都重新创建对象
override fun onDraw(canvas: Canvas) {
    val paint = Paint()  // 每次创建！
    canvas.drawCircle(...)
}

// ✅ 正确：预先创建 Paint
class MyView(context: Context) : View(context) {
    
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.RED
    }
    
    override fun onDraw(canvas: Canvas) {
        canvas.drawCircle(..., paint)  // 复用
    }
}
```

### 使用 Canvas.save()/restore()

```kotlin
override fun onDraw(canvas: Canvas) {
    // 保存状态
    canvas.save()
    
    // 旋转画布
    canvas.rotate(45f)
    canvas.drawBitmap(bitmap, 0f, 0f, paint)
    
    // 恢复状态
    canvas.restore()
}
```

### 硬件加速

```kotlin
// 开启硬件加速
// AndroidManifest.xml
<application
    android:hardwareAccelerated="true">

// 代码中控制
view.setLayerType(View.LAYER_TYPE_HARDWARE, null)  // 硬件层
view.setLayerType(View.LAYER_TYPE_SOFTWARE, null)   // 软件层
```

---

## 5. 内存优化

### 避免内存抖动

```kotlin
// ❌ 错误：在 onDraw 中分配对象
override fun onDraw(canvas: Canvas) {
    val rect = RectF()  // 每次创建导致 GC
    canvas.drawRect(rect, paint)
}

// ✅ 正确：预先分配
class MyView : View {
    private val rect = RectF()  // 只创建一次
    
    override fun onDraw(canvas: Canvas) {
        rect.set(0f, 0f, width.toFloat(), height.toFloat())
        canvas.drawRect(rect, paint)
    }
}
```

### 减少对象创建

```kotlin
// ❌ 错误：频繁创建对象
fun process() {
    val list = ArrayList<String>()  // 每次创建
    list.add("item")
}

// ✅ 正确：复用对象
class Processor {
    private val buffer = ArrayList<String>(10)  // 复用
    
    fun process() {
        buffer.clear()
        buffer.add("item")
    }
}
```

---

## 6. 调试工具

### Systrace

```bash
# 命令行使用
python systrace.py -t 10 -o trace.html sched gfx view wm

# Android Studio 中
# Profiler > Record > Systrace
```

### Perfetto

```bash
# Android 11+ 使用 Perfetto
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace \
  <<EOF
buffers: { size: 6348608 }
data_sources: { config { name: "linux.ftrace" } }
data_sources: { config { name: "linux.process_stats" } }
data_sources: { config { name: "android.surfaceflinger.frametimeline" } }
duration_ms: 10000
EOF
```

### Layout Inspector

```
使用步骤：
1. Tools > Layout Inspector
2. 选择设备/进程
3. 查看层级、属性、渲染效果
```

### 开发者选项

| 选项 | 作用 |
|------|------|
| GPU 过度绘制 | 显示过度绘制区域 |
| GPU 渲染分析 | 显示每帧渲染时间 |
| 关闭硬件加速 | 测试软件渲染 |

---

## 性能优化 checklist

{% mermaid flowchart TB %}
Checklist["View 性能优化 checklist"] --> LayoutOpt["布局优化<br/>ConstraintLayout / 移除背景 / include + merge / 精简 padding"]
Checklist --> DrawOpt["绘制优化<br/>避免 onDraw 创建对象 / 硬件加速 / 减少 clipPath / save/restore"]
Checklist --> MemoryOpt["内存优化<br/>避免内存抖动 / 减少对象创建 / 使用对象池"]
Checklist --> Tools["工具<br/>Hierarchy Viewer / Layout Inspector / Systrace / Perfetto"]
{% endmermaid %}

---

## 面试常问

| 问题 | 答案 |
|------|------|
| 16ms 包含什么？ | measure + layout + draw + GPU 渲染 |
| 过度绘制如何检测？ | 开发者选项 GPU 过度绘制 |
| onDraw 为什么不能创建对象？ | 触发 GC 导致卡顿 |
| ConstraintLayout 优势？ | 扁平化、性能好 |

---

## 总结

{% mermaid flowchart TB %}
PerfCore["View 性能优化核心"] --> P1["减少布局层级"]
PerfCore --> P2["避免过度绘制"]
PerfCore --> P3["避免 onDraw 创建对象"]
PerfCore --> P4["使用硬件加速"]
PerfCore --> P5["使用调试工具检测"]
{% endmermaid %}

---

**相关文章**：
- [View 绘制流程深度解析](./android-view-drawing-process.md)
- [RecyclerView 原理与优化实战](./android-recyclerview-optimization.md)
