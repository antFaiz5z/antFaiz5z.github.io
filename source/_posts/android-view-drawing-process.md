---
title: "View 绘制流程深度解析"
date: 2024-02-25 01:53:59
description: "从 measure、layout 到 draw，完全掌握 View 渲染原理"
comments: true
toc: true
categories:
  - Android
  - View
tags:
  - Android
  - View
  - Rendering
  - Draw
---

# View 绘制流程深度解析

> 从 measure、layout 到 draw，完全掌握 View 渲染原理

## 目录

1. [绘制流程概述](#1-绘制流程概述)
2. [Measure 测量阶段](#2-measure-测量阶段)
3. [Layout 布局阶段](#3-layout-布局阶段)
4. [Draw 绘制阶段](#4-draw-绘制阶段)
5. [整体流程图](#5-整体流程图)
6. [面试高频问题](#6-面试高频问题)

---

## 1. 绘制流程概述

### 三大核心方法

```
┌─────────────────────────────────────────────────────────────────────┐
│                        View 绘制三大方法                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    measure(int, int)    ──▶  确定 View 的宽高                          │
│           │                                                       │
│           ▼                                                       │
│    layout(int, int,     ──▶  确定 View 的位置                        │
│          int, int)                                                │
│           │                                                       │
│           ▼                                                       │
│    draw(Canvas)        ──▶  绘制 View 内容                          │
│                                                                      │
│    调用顺序: measure → layout → draw                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 何时触发绘制

```kotlin
// 触发重新绘制的场景
view.requestLayout()    // 触发 measure + layout + draw
view.invalidate()       // 只触发 draw
view.postInvalidate()   // 在主线程调用，触发 draw
```

---

## 2. Measure 测量阶段

### MeasureSpec 详解

```kotlin
// MeasureSpec = Mode + Size
// Mode 有三种模式:

/**
 * UNSPECIFIED (0): 父容器不限制，子 View 可以任意大小
 *                  用于 ScrollView 等容器
 * 
 * EXACTLY (1073741824): 精确模式，父容器指定了确切大小
 *                       对应 layout_width="200dp" 或 match_parent
 * 
 * AT_MOST (戵): 最大模式，子 View 不能超过指定大小
 *               对应 layout_width="wrap_content"
 */

val specMode = MeasureSpec.getMode(measureSpec)
val specSize = MeasureSpec.getSize(measureSpec)

when (specMode) {
    MeasureSpec.EXACTLY -> println("精确大小: $specSize")
    MeasureSpec.AT_MOST -> println("最大不能超过: $specSize")
    MeasureSpec.UNSPECIFIED -> println("无限制")
}
```

### onMeasure 核心逻辑

```kotlin
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    // 获取建议的宽高
    val width = getDefaultSize(suggestedMinimumWidth, widthMeasureSpec)
    val height = getDefaultSize(suggestedMinimumHeight, heightMeasureSpec)
    
    setMeasuredDimension(width, height)
}

// getDefaultSize 实现
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    
    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;  // 使用原始大小
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;  // 使用 MeasureSpec 指定的大小
            break;
    }
    return result;
}
```

### 自定义 View 的 onMeasure

```kotlin
class CustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    // 推荐尺寸
    private val defaultWidth = 200  // dp
    private val defaultHeight = 200 // dp
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val widthMode = MeasureSpec.getMode(widthMeasureSpec)
        val widthSize = MeasureSpec.getSize(widthMeasureSpec)
        val heightMode = MeasureSpec.getMode(heightMeasureSpec)
        val heightSize = MeasureSpec.getSize(heightMeasureSpec)
        
        // 计算最终尺寸
        val finalWidth = when (widthMode) {
            MeasureSpec.EXACTLY -> widthSize
            MeasureSpec.AT_MOST -> minOf(defaultWidth.dp.toPx(), widthSize)
            else -> defaultWidth.dp.toPx()
        }
        
        val finalHeight = when (heightMode) {
            MeasureSpec.EXACTLY -> heightSize
            MeasureSpec.AT_MOST -> minOf(defaultHeight.dp.toPx(), heightSize)
            else -> defaultHeight.dp.toPx()
        }
        
        setMeasuredDimension(finalWidth.toInt(), finalHeight.toInt())
    }
}
```

### ViewGroup 的 onMeasure

```kotlin
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    // 1. 测量所有子 View
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        measureChild(child, widthMeasureSpec, heightMeasureSpec)
    }
    
    // 2. 计算自己的尺寸
    // ...
    setMeasuredDimension(measuredWidth, measuredHeight)
}

// measureChild 实现
protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    // 获取子 View 的 LayoutParams
    final LayoutParams lp = child.getLayoutParams();
    
    // 计算子 View 的 MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(
        parentWidthMeasureSpec,
        mPaddingLeft + mPaddingRight,
        lp.width
    );
    
    final int childHeightMeasureSpec = getChildMeasureSpec(
        parentHeightMeasureSpec,
        mPaddingTop + mPaddingBottom,
        lp.height
    );
    
    // 测量子 View
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

---

## 3. Layout 布局阶段

### onLayout 核心逻辑

```kotlin
override fun onLayout(changed: Boolean, left: Int, top: Int, right: Int, bottom: Int) {
    // 对于 View: 无需实现
    
    // 对于 ViewGroup: 需要布局所有子 View
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        
        // 计算子 View 的位置
        val width = child.measuredWidth
        val height = child.measuredHeight
        
        // 简单的垂直布局示例
        val childLeft = left + paddingLeft
        val childTop = top + currentHeight + marginTop
        val childRight = childLeft + width
        val childBottom = childTop + height
        
        // 调用子 View 的 layout
        child.layout(childLeft, childTop, childRight, childBottom)
        
        // 记录已使用的高度
        currentHeight += height + marginTop + marginBottom
    }
}
```

### setFrame / setOpticalFrame

```kotlin
// setFrame 核心实现
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;
    
    // 检查位置是否变化
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
        
        // 保存旧值
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        
        // 如果大小变化，标记
        if (mRight - mLeft != getWidth() || mBottom - mTop != getHeight()) {
            mPrivateFlags |= PFLAG_SIZE_CHANGED;
        }
    }
    return changed;
}
```

---

## 4. Draw 绘制阶段

### draw 核心流程

```kotlin
// View.draw() 完整流程
public void draw(Canvas canvas) {
    // 1. 绘制背景
    drawBackground(canvas);
    
    // 2. 绘制内容 (如果需要)
    onDraw(canvas);
    
    // 3. 绘制子 View (ViewGroup)
    dispatchDraw(canvas);
    
    // 4. 绘制装饰 (滚动条、前景等)
    onDrawForeground(canvas);
}
```

### dispatchDraw 分发绘制

```kotlin
// ViewGroup.dispatchDraw
protected void dispatchDraw(Canvas canvas) {
    for (int i = 0; i < childrenCount; i++) {
        View child = getChildAt(i);
        
        // 检查子 View 是否可见
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
            // 绘制子 View
            drawChild(canvas, child, drawingTime);
        }
    }
}

// drawChild 核心
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    // ... 动画处理
    
    // 调用子 View 的 draw 方法
    return child.draw(canvas, this, drawingTime);
}
```

### 自定义 View 绘制示例

```kotlin
class CircleView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.RED
        style = Paint.Style.FILL
    }
    
    private var centerX = 0f
    private var centerY = 0f
    private var radius = 0f
    
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        centerX = w / 2f
        centerY = h / 2f
        radius = minOf(w, h) / 2f - 20f.dp.toPx()
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        canvas.drawCircle(centerX, centerY, radius, paint)
    }
}
```

---

## 5. 整体流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                      View 绘制完整流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    requestLayout()                                                   │
│          │                                                           │
│          ▼                                                           │
│    performTraversals() ──▶ 开始遍历                                   │
│          │                                                           │
│          ├─────────────────┐                                          │
│          ▼                 ▼                                          │
│    measure()          ┌──────────────────────────────────────┐      │
│    │                  │  1. onMeasure()                      │      │
│    │                  │  2. setMeasuredDimension()           │      │
│    │                  │  3. 如果是 ViewGroup，测量所有子View  │      │
│    ▼                  └──────────────────────────────────────┘      │
│    layout()           ┌──────────────────────────────────────┐      │
│    │                  │  1. setFrame() 设置位置               │      │
│    │                  │  2. onLayout()                        │      │
│    │                  │  3. 如果是ViewGroup，布局所有子View   │      │
│    ▼                  └──────────────────────────────────────┘      │
│    draw()            ┌──────────────────────────────────────┐      │
│                       │  1. drawBackground() 背景           │      │
│                       │  2. onDraw() 绘制内容                │      │
│                       │  3. dispatchDraw() 分发绘制子View    │      │
│                       │  4. onDrawForeground() 前景         │      │
│                       └──────────────────────────────────────┘      │
│          │                                                           │
│          ▼                                                           │
│    绘制完成                                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. 面试高频问题

### Q1: requestLayout vs invalidate 区别

| 方法 | 作用 | 触发 |
|------|------|------|
| `requestLayout()` | 重新 measure + layout + draw | 位置或大小变化 |
| `invalidate()` | 重新 draw | 内容变化 |
| `postInvalidate()` | 线程中调用 invalidate | 同上 |

### Q2: View 测量模式如何决定？

```
parentMeasureSpec 由父容器的 MeasureSpec + 子 View 的 LayoutParams 决定:

┌────────────────────────────────────────────┐
│  父 MeasureSpec    │ 子 LayoutParams │ 结果 │
├────────────────────┼─────────────────┼──────┤
│ EXACTLY            │ match_parent   │EXACTLY│
│ EXACTLY            │ wrap_content   │AT_MOST│
│ EXACTLY            │ 200dp          │EXACTLY│
│ AT_MOST            │ match_parent   │AT_MOST│
│ AT_MOST            │ wrap_content   │AT_MOST│
│ AT_MOST            │ 200dp          │EXACTLY│
│ UNSPECIFIED        │ 任意           │UNSPEC │
└────────────────────────────────────────────┘
```

### Q3: 为什么 wrap_content 不生效？

```kotlin
// 问题：wrap_content 生效但尺寸为 0
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    // ❌ 错误：使用默认实现
    super.onMeasure(widthMeasureSpec, heightMeasureSpec)
    
    // ✅ 正确：手动计算尺寸
    val desiredWidth = 200.dp.toPx()
    val desiredHeight = 200.dp.toPx()
    
    val width = resolveSize(desiredWidth, widthMeasureSpec)
    val height = resolveSize(desiredHeight, heightMeasureSpec)
    
    setMeasuredDimension(width, height)
}
```

---

## 总结

```
View 绘制核心:
─────────────────────────────────────────
1. Measure:    measure() → onMeasure() → setMeasuredDimension()
2. Layout:     layout()  → onLayout()   → setFrame()
3. Draw:       draw()    → onDraw()     → dispatchDraw()
─────────────────────────────────────────
```

---

**相关文章**：
- [View 事件分发机制完全掌握](./android-view-event-dispatch.md)
- [ViewGroup 测量与布局完全指南](./android-viewgroup-measure-layout.md)
