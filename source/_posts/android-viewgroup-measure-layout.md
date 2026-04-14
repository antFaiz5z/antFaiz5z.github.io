---
title: "ViewGroup 测量与布局完全指南"
date: 2025-12-22 10:57:07
description: "理解 ViewGroup 如何测量子 View 并确定位置"
comments: true
toc: true
categories:
  - Android
  - View
tags:
  - Android
  - ViewGroup
  - Layout
  - Measure
---

# ViewGroup 测量与布局完全指南

> 理解 ViewGroup 如何测量子 View 并确定位置

## 目录

1. [ViewGroup 职责](#1-viewgroup-职责)
2. [measureSpec 计算规则](#2-measurespec-计算规则)
3. [onMeasure 最佳实践](#3-onmeasure-最佳实践)
4. [onLayout 布局策略](#4-onlayout-布局策略)
5. [常见布局实现](#5-常见布局实现)

---

## 1. ViewGroup 职责

### ViewGroup vs View

{% mermaid flowchart TB %}
View["View"] --> ViewGroup["ViewGroup"]
ViewGroup --> Linear["LinearLayout"]
ViewGroup --> Relative["RelativeLayout"]
ViewGroup --> Frame["FrameLayout"]
ViewGroup --> Duty1["测量所有子 View（measureChildren）"]
ViewGroup --> Duty2["确定子 View 位置（onLayout）"]
ViewGroup --> Duty3["分发触摸事件（dispatchDraw）"]
{% endmermaid %}

### measureChildren 流程

```kotlin
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        
        // 跳过不可见的 View
        if ((child.mViewFlags & VISIBILITY_MASK) == View.GONE) {
            continue;
        }
        
        // 测量子 View
        measureChild(child, widthMeasureSpec, heightMeasureSpec);
    }
}
```

---

## 2. MeasureSpec 计算规则

### getChildMeasureSpec 核心逻辑

```kotlin
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    // spec: 父容器的 MeasureSpec
    // padding: 父容器的内边距
    // childDimension: 子 View 的 LayoutParams.width/height
    
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    
    // 可用空间 = 父容器大小 - 内边距
    int size = Math.max(0, specSize - padding);
    
    int resultSize = 0;
    int resultMode = 0;
    
    switch (specMode) {
        // 父容器是精确模式
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                // 具体数值 (如 100dp)
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // 填满父容器
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // 包裹内容，最大不能超过父容器
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
            
        // 父容器是最大模式
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // 最大不能超过父容器
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // 包裹内容，最大不能超过父容器
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
            
        // 父容器无限制
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // View 自己决定
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
    }
    
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

### 计算规则表

{% mermaid flowchart TB %}
ParentExact["父 MeasureSpec = EXACTLY"] --> E1["具体数值 → EXACTLY + 100dp"]
ParentExact --> E2["match_parent → EXACTLY + 父 size"]
ParentExact --> E3["wrap_content → AT_MOST + 父 size"]
ParentAtMost["父 MeasureSpec = AT_MOST"] --> A1["具体数值 → EXACTLY + 100dp"]
ParentAtMost --> A2["match_parent → AT_MOST + 父 size"]
ParentAtMost --> A3["wrap_content → AT_MOST + 父 size"]
ParentUn["父 MeasureSpec = UNSPECIFIED"] --> U1["具体数值 → EXACTLY + 100dp"]
ParentUn --> U2["match_parent → UNSPECIFIED + 0"]
ParentUn --> U3["wrap_content → UNSPECIFIED + 0"]
{% endmermaid %}

---

## 3. onMeasure 最佳实践

### LinearLayout 实现思路

```kotlin
class LinearLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : ViewGroup(context, attrs, defStyleAttr) {
    
    enum class Orientation { HORIZONTAL, VERTICAL }
    
    var orientation: Orientation = Orientation.VERTICAL
        set(value) {
            if (field != value) {
                field = value
                requestLayout()
            }
        }
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        // 1. 测量所有子 View
        measureChildren(widthMeasureSpec, heightMeasureSpec)
        
        // 2. 根据方向计算自己的尺寸
        val widthMode = MeasureSpec.getMode(widthMeasureSpec)
        val widthSize = MeasureSpec.getSize(widthMeasureSpec)
        val heightMode = MeasureSpec.getMode(heightMeasureSpec)
        val heightSize = MeasureSpec.getSize(heightMeasureSpec)
        
        var totalWidth = 0
        var totalHeight = 0
        
        if (orientation == Orientation.VERTICAL) {
            // 垂直布局：高度累加，宽度取最大
            for (i in 0 until childCount) {
                val child = getChildAt(i)
                if (child.visibility != View.GONE) {
                    totalHeight += child.measuredHeight
                    totalWidth = maxOf(totalWidth, child.measuredWidth)
                }
            }
            totalHeight += paddingTop + paddingBottom
        } else {
            // 水平布局：宽度累加，高度取最大
            for (i in 0 until childCount) {
                val child = getChildAt(i)
                if (child.visibility != View.GONE) {
                    totalWidth += child.measuredWidth
                    totalHeight = maxOf(totalHeight, child.measuredHeight)
                }
            }
            totalWidth += paddingLeft + paddingRight
        }
        
        // 3. 应用 MeasureSpec 规则
        val finalWidth = resolveSize(totalWidth, widthMeasureSpec)
        val finalHeight = resolveSize(totalHeight, heightMeasureSpec)
        
        setMeasuredDimension(finalWidth, finalHeight)
    }
    
    private fun resolveSize(desiredSize: Int, measureSpec: Int): Int {
        val mode = MeasureSpec.getMode(measureSpec)
        val size = MeasureSpec.getSize(measureSpec)
        
        return when (mode) {
            MeasureSpec.EXACTLY -> size
            MeasureSpec.AT_MOST -> minOf(desiredSize, size)
            else -> desiredSize
        }
    }
}
```

### 带有权重(Weight)的测量

```kotlin
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    // 1. 先测量不使用权重的子 View
    var totalWeight = 0
    var fixedDimension = 0
    
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        val params = child.layoutParams as LayoutParams
        
        if (params.weight > 0) {
            totalWeight += params.weight
        } else {
            // 固定尺寸的子 View
            if (orientation == Orientation.VERTICAL) {
                fixedDimension += child.measuredHeight
            } else {
                fixedDimension += child.measuredWidth
            }
        }
    }
    
    // 2. 计算剩余空间
    val availableSpace = when (orientation) {
        Orientation.VERTICAL -> 
            MeasureSpec.getSize(heightMeasureSpec) - paddingTop - paddingBottom - fixedDimension
        Orientation.HORIZONTAL ->
            MeasureSpec.getSize(widthMeasureSpec) - paddingLeft - paddingRight - fixedDimension
    }
    
    // 3. 重新测量带权重的子 View
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        val params = child.layoutParams as LayoutParams
        
        if (params.weight > 0) {
            val childSize = (availableSpace * params.weight / totalWeight).toInt()
            
            if (orientation == Orientation.VERTICAL) {
                val childHeightSpec = MeasureSpec.makeMeasureSpec(
                    childSize, MeasureSpec.EXACTLY
                )
                child.measure(widthMeasureSpec, childHeightSpec)
            } else {
                val childWidthSpec = MeasureSpec.makeMeasureSpec(
                    childSize, MeasureSpec.EXACTLY
                )
                child.measure(childWidthSpec, heightMeasureSpec)
            }
        }
    }
}
```

---

## 4. onLayout 布局策略

### FrameLayout onLayout

```kotlin
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        
        if (child.visibility == View.GONE) continue
        
        val width = child.measuredWidth
        val height = child.measuredHeight
        
        // 默认：所有子 View 放在左上角
        // 实际位置由 gravity 决定
        val childLeft = when (child.layoutParams as? LayoutParams)?.gravity {
            Gravity.CENTER -> (r - l - width) / 2
            Gravity.END -> r - l - width - paddingRight
            else -> paddingLeft
        }
        
        val childTop = when (child.layoutParams as? LayoutParams)?.gravity) {
            Gravity.CENTER_VERTICAL -> (b - t - height) / 2
            Gravity.BOTTOM -> b - t - height - paddingBottom
            else -> paddingTop
        }
        
        child.layout(childLeft, childTop, childLeft + width, childTop + height)
    }
}
```

### RelativeLayout onLayout

```kotlin
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    // RelativeLayout 布局更复杂：
    // 1. 第一轮：确定依赖关系，测量子 View
    // 2. 第二轮：按依赖顺序布局
    
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        // 根据规则确定位置
        // ...
        child.layout(childLeft, childTop, childLeft + width, childTop + height)
    }
}
```

---

## 5. 常见布局实现

### 垂直线性布局

```kotlin
class VerticalLinearLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : ViewGroup(context, attrs) {
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        // 测量所有子 View
        measureChildren(widthMeasureSpec, heightMeasureSpec)
        
        // 计算总高度
        var totalHeight = paddingTop + paddingBottom
        var maxWidth = paddingLeft + paddingRight
        
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            if (child.visibility != View.GONE) {
                totalHeight += child.measuredHeight
                maxWidth = maxOf(maxWidth, child.measuredWidth + paddingLeft + paddingRight)
            }
        }
        
        // 应用 AT_MOST 规则
        val width = resolveSize(maxWidth, widthMeasureSpec)
        val height = resolveSize(totalHeight, heightMeasureSpec)
        
        setMeasuredDimension(width, height)
    }
    
    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        var currentTop = paddingTop
        
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            if (child.visibility != View.GONE) {
                val childLeft = paddingLeft
                val childTop = currentTop
                val childRight = childLeft + child.measuredWidth
                val childBottom = childTop + child.measuredHeight
                
                child.layout(childLeft, childTop, childRight, childBottom)
                
                currentTop = childBottom
            }
        }
    }
}
```

### 流式布局 (FlowLayout)

```kotlin
class FlowLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : ViewGroup(context, attrs) {
    
    var horizontalSpacing = 16.dp.toPx().toInt()
    var verticalSpacing = 16.dp.toPx().toInt()
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val width = MeasureSpec.getSize(widthMeasureSpec)
        var lineHeight = 0
        var x = paddingLeft
        var y = paddingTop
        
        var maxHeight = 0
        
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            measureChild(child, widthMeasureSpec, heightMeasureSpec)
            
            if (x + child.measuredWidth > width - paddingRight) {
                // 换行
                x = paddingLeft
                y += lineHeight + verticalSpacing
                lineHeight = 0
            }
            
            x += child.measuredWidth + horizontalSpacing
            lineHeight = maxOf(lineHeight, child.measuredHeight)
            maxHeight = y + lineHeight
        }
        
        maxHeight += paddingBottom
        setMeasuredDimension(width, resolveSize(maxHeight, heightMeasureSpec))
    }
    
    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        var x = paddingLeft
        var y = paddingTop
        var lineHeight = 0
        
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            
            if (x + child.measuredWidth > r - l - paddingRight) {
                x = paddingLeft
                y += lineHeight + verticalSpacing
                lineHeight = 0
            }
            
            child.layout(x, y, x + child.measuredWidth, y + child.measuredHeight)
            
            x += child.measuredWidth + horizontalSpacing
            lineHeight = maxOf(lineHeight, child.measuredHeight)
        }
    }
}
```

---

## 面试常问

### Q1: ViewGroup 为什么需要 onMeasure 和 onLayout？

```
答：
- onMeasure: 计算自己的尺寸 + 测量所有子 View
- onLayout: 确定所有子 View 的位置
- View: 只需 draw，不需要布局
```

### Q2: measure 和 layout 为什么可能多次调用？

```
答：
- 父 View 的 onMeasure 可能被多次调用
- 第一次可能不知道子 View 的确切尺寸
- 需要多次尝试才能确定最终尺寸
- 典型的 LinearLayout 带 weight 的情况
```

### Q3: match_parent 和 wrap_content 的区别？

```
答：
- match_parent: 尝试占满父容器剩余空间
- wrap_content: 包裹内容，但不能超过父容器
- 实现区别在 getChildMeasureSpec
```

---

## 总结

{% mermaid flowchart TB %}
VGCore["ViewGroup 测量布局核心"] --> G1["measureChildren：测量所有子 View"]
VGCore --> G2["getChildMeasureSpec：计算子 View 的 MeasureSpec"]
VGCore --> G3["onLayout：根据测量结果确定子 View 位置"]
VGCore --> G4["resolveSize：应用 AT_MOST / EXACTLY 规则"]
{% endmermaid %}

---

**相关文章**：
- [View 绘制流程深度解析](./android-view-drawing-process.md)
- [自定义 View 实战从入门到精通](./android-custom-view-practice.md)
