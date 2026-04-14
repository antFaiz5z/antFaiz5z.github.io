---
title: "自定义 View 实战从入门到精通"
date: 2021-11-24 06:28:11
description: "从零开始构建可复用的自定义 View"
comments: true
toc: true
categories:
  - Android
  - View
tags:
  - Android
  - Custom View
  - View
  - UI
---

# 自定义 View 实战从入门到精通

> 从零开始构建可复用的自定义 View

## 目录

1. [自定义 View 步骤](#1-自定义-view-步骤)
2. [属性定义与读取](#2-属性定义与读取)
3. [测量逻辑实现](#3-测量逻辑实现)
4. [绘制逻辑实现](#4-绘制逻辑实现)
5. [交互事件处理](#5-交互事件处理)
6. [完整案例：圆形进度条](#6-完整案例圆形进度条)

---

## 1. 自定义 View 步骤

### 五步法则

{% mermaid flowchart TB %}
Step1["1. 定义属性（attrs.xml）"] --> Step2["2. 继承 View，创建类"]
Step2 --> Step3["3. 重写 onMeasure"]
Step3 --> Step4["4. 重写 onDraw"]
Step4 --> Step5["5. 处理交互事件"]
{% endmermaid %}

---

## 2. 属性定义与读取

### 第一步：定义 attrs.xml

```xml
<!-- res/values/attrs.xml -->
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- 声明自定义属性 -->
    <declare-styleable name="CircleView">
        <!-- 颜色属性 -->
        <attr name="circleColor" format="color" />
        <!-- 尺寸属性 -->
        <attr name="circleRadius" format="dimension" />
        <!-- 布尔属性 -->
        <attr name="showBorder" format="boolean" />
        <!-- 枚举属性 -->
        <attr name="circleStyle" format="enum">
            <enum name="fill" value="0" />
            <enum name="stroke" value="1" />
        </attr>
    </declare-styleable>
</resources>
```

### 第二步：在布局中使用

```xml
<!-- layout/activity_main.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.example.CircleView
        android:layout_width="200dp"
        android:layout_height="200dp"
        app:circleColor="@color/red"
        app:circleRadius="50dp"
        app:showBorder="true"
        app:circleStyle="fill" />

</LinearLayout>
```

### 第三步：在代码中读取

```kotlin
class CircleView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    // 定义属性变量
    private var circleColor = Color.RED
    private var circleRadius = 50f.dp.toPx()
    private var showBorder = false
    private var circleStyle = Style.FILL
    
    enum class Style { FILL, STROKE }
    
    init {
        // 读取自定义属性
        attrs?.let {
            val typedArray = context.obtainStyledAttributes(
                it,
                R.styleable.CircleView
            )
            
            // 读取颜色
            circleColor = typedArray.getColor(
                R.styleable.CircleView_circleColor,
                Color.RED
            )
            
            // 读取尺寸
            circleRadius = typedArray.getDimension(
                R.styleable.CircleView_circleRadius,
                50f.dp.toPx()
            )
            
            // 读取布尔值
            showBorder = typedArray.getBoolean(
                R.styleable.CircleView_showBorder,
                false
            )
            
            // 读取枚举
            val styleOrdinal = typedArray.getInt(
                R.styleable.CircleView_circleStyle,
                0
            )
            circleStyle = Style.entries[styleOrdinal]
            
            typedArray.recycle()
        }
    }
}
```

---

## 3. 测量逻辑实现

### onMeasure 实现模板

```kotlin
class CircleView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    // 默认尺寸
    private val defaultSize = 100.dp.toPx().toInt()
    
    // 最小尺寸
    private val minSize = 50.dp.toPx().toInt()
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        // 计算宽度
        val width = getMeasureResult(defaultSize, widthMeasureSpec)
        
        // 计算高度 (对于圆形，宽度=高度)
        val height = getMeasureResult(defaultSize, heightMeasureSpec)
        
        // 取宽高的最小值确保是正方形
        val size = minOf(width, height)
        
        setMeasuredDimension(size, size)
    }
    
    private fun getMeasureResult(defaultSize: Int, measureSpec: Int): Int {
        val mode = MeasureSpec.getMode(measureSpec)
        val size = MeasureSpec.getSize(measureSpec)
        
        return when (mode) {
            MeasureSpec.EXACTLY -> size              // match_parent 或具体值
            MeasureSpec.AT_MOST -> minOf(defaultSize, size)  // wrap_content
            else -> defaultSize                      // 未指定
        }
    }
}
```

---

## 4. 绘制逻辑实现

### Canvas 常用方法

```kotlin
// 画布核心方法
canvas.drawCircle(x, y, radius, paint)        // 画圆
canvas.drawRect(left, top, right, bottom, paint)  // 画矩形
canvas.drawLine(startX, startY, endX, endY, paint)  // 画线
canvas.drawText(text, x, y, paint)           // 画文字
canvas.drawPath(path, paint)                 // 画路径
canvas.drawArc(rectF, startAngle, sweepAngle, useCenter, paint)  // 画弧
canvas.save()                                // 保存画布状态
canvas.restore()                             // 恢复画布状态
canvas.rotate(degrees)                       // 旋转
canvas.translate(dx, dy)                     // 平移
canvas.scale(sx, sy)                         // 缩放
```

### Paint 常用配置

```kotlin
private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
    // 抗锯齿
    isAntiAlias = true
    
    // 颜色
    color = Color.RED
    
    // 样式
    style = Paint.Style.FILL    // 填充
    style = Paint.Style.STROKE  // 描边
    style = Paint.Style.FILL_AND_STROKE  // 填充+描边
    
    // 描边宽度
    strokeWidth = 4f.dp.toPx()
    
    // 圆角
    strokeCap = Paint.Cap.ROUND
    strokeJoin = Paint.Join.ROUND
    
    // 文字
    textSize = 16f.dp.toPx()
    textAlign = Paint.Align.CENTER
    
    // 阴影
    setShadowLayer(4f.dp.toPx(), 2f.dp.toPx(), 2f.dp.toPx(), Color.BLACK)
}
```

---

## 5. 交互事件处理

### 触摸事件

```kotlin
class CircleView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    private var lastX = 0f
    private var lastY = 0f
    private var isDragging = false
    
    // 回调接口
    var onCircleClickListener: OnCircleClickListener? = null
    
    interface OnCircleClickListener {
        fun onClick(view: CircleView)
    }
    
    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                // 检查是否点击在圆内
                if (isPointInCircle(event.x, event.y)) {
                    lastX = event.x
                    lastY = event.y
                    isDragging = true
                    parent?.requestDisallowInterceptTouchEvent(true)
                    return true
                }
            }
            
            MotionEvent.ACTION_MOVE -> {
                if (isDragging) {
                    val dx = event.x - lastX
                    val dy = event.y - lastY
                    
                    // 移动 View
                    x += dx
                    y += dy
                    
                    lastX = event.x
                    lastY = event.y
                    return true
                }
            }
            
            MotionEvent.ACTION_UP -> {
                if (isDragging) {
                    isDragging = false
                    parent?.requestDisallowInterceptTouchEvent(false)
                    
                    // 检查是否是点击
                    if (isPointInCircle(event.x, event.y)) {
                        onCircleClickListener?.onClick(this)
                        performClick()
                    }
                    return true
                }
            }
            
            MotionEvent.ACTION_CANCEL -> {
                isDragging = false
                parent?.requestDisallowInterceptTouchEvent(false)
            }
        }
        
        return super.onTouchEvent(event)
    }
    
    private fun isPointInCircle(x: Float, y: Float): Boolean {
        val centerX = width / 2f
        val centerY = height / 2f
        val dx = x - centerX
        val dy = y - centerY
        return dx * dx + dy * dy <= circleRadius * circleRadius
    }
    
    override fun performClick(): Boolean {
        super.performClick()
        return true
    }
}
```

---

## 6. 完整案例：圆形进度条

### 完整代码

```kotlin
class CircleProgressView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    // 画笔
    private val backgroundPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 10f.dp.toPx()
        color = Color.LTGRAY
    }
    
    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 10f.dp.toPx()
        color = Color.GREEN
        strokeCap = Paint.Cap.ROUND
    }
    
    private val textPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        textAlign = Paint.Align.CENTER
        textSize = 24f.dp.toPx()
        color = Color.BLACK
    }
    
    // 进度 (0-100)
    var progress: Int = 0
        set(value) {
            field = value.coerceIn(0, 100)
            invalidate()
        }
    
    // 进度颜色
    var progressColor: Int = Color.GREEN
        set(value) {
            field = value
            progressPaint.color = value
            invalidate()
        }
    
    // 圆弧矩形
    private val rectF = RectF()
    
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        
        val padding = backgroundPaint.strokeWidth / 2
        rectF.set(
            padding,
            padding,
            w - padding,
            h - padding
        )
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        val centerX = width / 2f
        val centerY = height / 2f
        
        // 1. 画背景圆
        canvas.drawCircle(centerX, centerY, minOf(centerX, centerY) - backgroundPaint.strokeWidth, backgroundPaint)
        
        // 2. 画进度圆弧 (-90度从顶部开始)
        val sweepAngle = (progress / 100f) * 360f
        canvas.drawArc(rectF, -90f, sweepAngle, false, progressPaint)
        
        // 3. 画文字
        val text = "$progress%"
        val textY = centerY - (textPaint.descent() + textPaint.ascent()) / 2
        canvas.drawText(text, centerX, textY, textPaint)
    }
    
    // 设置进度的动画
    fun setProgressAnimated(targetProgress: Int, duration: Long = 500L) {
        val startProgress = progress
        val animator = ValueAnimator.ofInt(startProgress, targetProgress)
        animator.duration = duration
        animator.interpolator = DecelerateInterpolator()
        animator.addUpdateListener { animation ->
            progress = animation.animatedValue as Int
        }
        animator.start()
    }
}
```

### XML 布局

```xml
<com.example.CircleProgressView
    android:id="@+id/progressView"
    android:layout_width="200dp"
    android:layout_height="200dp"
    app:progressColor="@color/blue" />
```

### 使用

```kotlin
// 设置进度
progressView.progress = 75

// 动画设置进度
progressView.setProgressAnimated(75, 1000L)
```

---

## 自定义 View 检查清单

| 检查项 | 说明 |
|--------|------|
| onMeasure 正确处理 wrap_content | ✅ |
| 读取自定义属性 | ✅ |
| 处理 padding | ✅ |
| onDraw 考虑状态变化 | ✅ |
| onTouchEvent 返回值正确 | ✅ |
| 性能优化（Paint 缓存） | ✅ |

---

## 总结

{% mermaid flowchart TB %}
CustomCore["自定义 View 核心步骤"] --> C1["attrs.xml 定义属性"]
CustomCore --> C2["构造函数读取属性"]
CustomCore --> C3["onMeasure 计算尺寸"]
CustomCore --> C4["onDraw 绘制内容"]
CustomCore --> C5["onTouchEvent 处理交互"]
{% endmermaid %}

---

**相关文章**：
- [View 绘制流程深度解析](./android-view-drawing-process.md)
- [View 事件分发机制完全掌握](./android-view-event-dispatch.md)
