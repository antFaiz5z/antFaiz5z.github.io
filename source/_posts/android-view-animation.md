---
title: "View 动画原理与属性动画"
date: 2023-11-26 05:36:13
description: "从 Animation 到 PropertyAnimator 完全指南"
comments: true
toc: true
categories:
  - Android
  - View
tags:
  - Android
  - View
  - Animation
  - Property Animation
---

# View 动画原理与属性动画

> 从 Animation 到 PropertyAnimator 完全指南

## 目录

1. [动画类型概述](#1-动画类型概述)
2. [View Animation](#2-view-animation)
3. [属性动画 Property Animation](#3-属性动画-property-animation)
4. [Interpolator 插值器](#4-interpolator-插值器)
5. [自定义动画](#5-自定义动画)
6. [动画组合与监听](#6-动画组合与监听)

---

## 1. 动画类型概述

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Android 动画体系                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    View Animation (旧)                                              │
│    ├── Tween Animation                                              │
│    │   ├── ScaleAnimation                                           │
│    │   ├── RotateAnimation                                          │
│    │   ├── TranslateAnimation                                       │
│    │   └── AlphaAnimation                                           │
│    └── Frame Animation                                              │
│                                                                      │
│    Property Animation (新)                                          │
│    ├── ValueAnimator                                                │
│    ├── ObjectAnimator                                                │
│    └── AnimatorSet                                                  │
│                                                                      │
│    Drawable Animation                                               │
│    └── AnimationDrawable                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. View Animation

### 基本用法

```kotlin
// XML 中定义
// res/anim/scale.xml
<?xml version="1.0" encoding="utf-8"?>
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"
    android:fromXScale="1.0"
    android:toXScale="1.2"
    android:fromYScale="1.0"
    android:toYScale="1.2"
    android:pivotX="50%"
    android:pivotY="50%"
    android:fillAfter="true" />

// 代码中使用
val animation = AnimationUtils.loadAnimation(this, R.anim.scale)
view.startAnimation(animation)

// 监听动画
animation.setAnimationListener(object : Animation.AnimationListener {
    override fun onAnimationStart(animation: Animation?) {}
    override fun onAnimationEnd(animation: Animation?) {}
    override fun onAnimationRepeat(animation: Animation?) {}
})
```

### View Animation 缺点

```kotlin
// ❌ 缺点 1：只改变显示，不改变属性
view.startAnimation(TranslateAnimation(0f, 100f, 0f, 0f))
// 动画结束后，view.x 仍然是 0！

// 缺点 2：只能做简单动画
// 缺点 3：动画结束后需要手动清除
view.clearAnimation()
```

---

## 3. 属性动画 Property Animation

### ValueAnimator

```kotlin
// 基本用法
val animator = ValueAnimator.ofInt(0, 100)
animator.duration = 500
animator.addUpdateListener { animation ->
    val value = animation.animatedValue as Int
    view.layoutParams.width = value
    view.requestLayout()
}
animator.start()

// ofFloat 用法
val animator = ValueAnimator.ofFloat(0f, 1f)
animator.duration = 300
animator.addUpdateListener { animation ->
    val value = animation.animatedValue as Float
    view.alpha = value
}
animator.start()

// ofObject 用法
val animator = ValueAnimator.ofObject(
    TypeEvaluator { fraction, startValue, endValue ->
        // 自定义计算逻辑
        val color = lerp(startValue as Int, endValue as Int, fraction)
        color
    },
    Color.RED,
    Color.BLUE
)
```

### ObjectAnimator

```kotlin
// 直接改变属性
ObjectAnimator.ofFloat(view, "translationX", 0f, 100f).start()
ObjectAnimator.ofFloat(view, "translationY", 0f, 100f).start()
ObjectAnimator.ofFloat(view, "scaleX", 1f, 1.2f).start()
ObjectAnimator.ofFloat(view, "rotation", 0f, 360f).start()
ObjectAnimator.ofFloat(view, "alpha", 1f, 0f).start()

// 组合动画
ObjectAnimator.ofFloat(view, "scaleX", 1f, 1.5f, 1f).start()

// 常用属性列表
// translationX / translationY
// rotation / rotationX / rotationY
// scaleX / scaleY
// alpha
// pivotX / pivotY
// x / y
// scrollX / scrollY
```

### 自定义属性动画

```kotlin
// 1. 为 View 添加 setter
class CircleView(context: Context) : View(context) {
    
    var sweepAngle: Float = 0f
        set(value) {
            field = value
            invalidate()  // 必须调用
        }
    
    override fun onDraw(canvas: Canvas) {
        canvas.drawArc(rectF, -90f, sweepAngle, true, paint)
    }
}

// 2. 使用 ObjectAnimator
ObjectAnimator.ofInt(circleView, "sweepAngle", 0, 360).apply {
    duration = 1000
    start()
}
```

---

## 4. Interpolator 插值器

### 内置插值器

```kotlin
// 加速
AccelerateInterpolator()
AccelerateDecelerateInterpolator()

// 减速
DecelerateInterpolator()

// 弹跳
BounceInterpolator()

// 回弹
OvershootInterpolator()

// 重复
Repeatable (与 Reverse 组合)

// 线性
LinearInterpolator()
```

### 插值器对比

```
时间进度: 0    0.25   0.5   0.75   1.0
─────────────────────────────────────────
Linear:      0    0.25   0.5   0.75   1.0
Accelerate:  0    0.06   0.25  0.56   1.0
Bounce:      0    0.25   0.5   0.75   1.0
             (中间弹跳)
Overshoot:   0    0.25   0.5   0.75   1.25
             (超过终点再回弹)
```

### 自定义插值器

```kotlin
class CustomInterpolator : Interpolator {
    override fun getInterpolation(input: Float): Float {
        // input: 0-1
        // return: 自定义进度
        
        // 例如：EaseOut
        return 1 - (1 - input) * (1 - input)
    }
}
```

---

## 5. 自定义动画

### 自定义 ValueAnimator

```kotlin
class CircleRevealAnimator(private val view: View) {
    
    private var animator: ValueAnimator? = null
    
    fun start() {
        animator = ValueAnimator.ofInt(view.width, 0).apply {
            duration = 300
            interpolator = DecelerateInterpolator()
            addUpdateListener { animation ->
                val value = animation.animatedValue as Int
                view.layoutParams.width = value
                view.requestLayout()
            }
        }
        animator?.start()
    }
    
    fun cancel() {
        animator?.cancel()
    }
}
```

### 自定义 TypeEvaluator

```kotlin
// 颜色渐变
val colorAnimator = ValueAnimator.ofObject(
    ArgbEvaluator(),
    Color.RED,
    Color.BLUE,
    Color.GREEN
).apply {
    duration = 2000
    addUpdateListener { animation ->
        val color = animation.animatedValue as Int
        view.setBackgroundColor(color)
    }
}
colorAnimator.start()

// PointF 轨迹
val pathEvaluator = object : TypeEvaluator<PointF> {
    private val path = Path()
    
    override fun evaluate(fraction: Float, startValue: PointF, endValue: PointF): PointF {
        // 贝塞尔曲线
        val x = startValue.x + (endValue.x - startValue.x) * fraction
        val y = startValue.y + (endValue.y - startValue.y) * fraction
        return PointF(x, y)
    }
}
```

### 自定义 ObjectAnimator

```kotlin
// 1. 在 View 中添加属性
var progress: Float = 0f
    private set

fun setProgress(value: Float) {
    progress = value
    invalidate()
}

// 2. 创建 Animator
ObjectAnimator.ofFloat(view, "progress", 0f, 100f).apply {
    duration = 1000
    start()
}
```

---

## 6. 动画组合与监听

### AnimatorSet

```kotlin
val scaleX = ObjectAnimator.ofFloat(view, "scaleX", 1f, 1.5f).apply { duration = 300 }
val scaleY = ObjectAnimator.ofFloat(view, "scaleY", 1f, 1.5f).apply { duration = 300 }
val alpha = ObjectAnimator.ofFloat(view, "alpha", 1f, 0f).apply { duration = 300 }

// 同时播放
AnimatorSet().apply {
    playTogether(scaleX, scaleY, alpha)
    start()
}

// 顺序播放
AnimatorSet().apply {
    playSequentially(scaleX, alpha)
    start()
}

// 复杂组合
AnimatorSet().apply {
    play(scaleX).with(scaleY)
    play(alpha).after(scaleX)
    start()
}
```

### PropertyValuesHolder

```kotlin
// 同时改变多个属性
val holder1 = PropertyValuesHolder.ofFloat("scaleX", 1f, 1.5f)
val holder2 = PropertyValuesHolder.ofFloat("scaleY", 1f, 1.5f)
val holder3 = PropertyValuesHolder.ofFloat("alpha", 1f, 0.5f)

ObjectAnimator.ofPropertyValuesHolder(view, holder1, holder2, holder3).apply {
    duration = 300
    start()
}
```

### 动画监听

```kotlin
// Animator 监听
animator.addListener(object : Animator.AnimatorListener {
    override fun onAnimationStart(animation: Animator) {}
    override fun onAnimationEnd(animation: Animator) {}
    override fun onAnimationCancel(animation: Animator) {}
    override fun onAnimationRepeat(animation: Animator) {}
})

// AnimatorUpdateListener
animator.addUpdateListener { animation ->
    val value = animation.animatedValue
    // 更新 UI
}
```

---

## 面试常问

| 问题 | 答案 |
|------|------|
| View Animation vs 属性动画？ | View Animation 只改变显示，属性动画改变实际属性 |
| 动画为什么卡顿？ | 16ms 内未完成，或在主线程做耗时操作 |
| 如何停止动画？ | animator.cancel() 或 view.clearAnimation() |
| Interpolator 作用？ | 控制动画速率变化 |

---

## 总结

```
Android 动画核心:
─────────────────────────────────────────
1. View Animation: 简单但有限制
2. Property Animation: 改变实际属性，推荐
3. Interpolator: 控制速率变化
4. AnimatorSet: 组合复杂动画
─────────────────────────────────────────
```

---

**相关文章**：
- [View 绘制流程深度解析](./android-view-drawing-process.md)
- [View 性能优化与卡顿分析](./android-view-performance-optimization.md)
