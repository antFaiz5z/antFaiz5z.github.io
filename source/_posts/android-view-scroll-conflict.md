---
title: "View 滚动与滑动冲突解决"
date: 2025-09-26 02:00:16
description: "嵌套滚动场景的完整解决方案 滑动冲突场景"
comments: true
toc: true
categories:
  - Android
  - View
tags:
  - Android
  - View
  - Nested Scrolling
  - Touch
---

# View 滚动与滑动冲突解决

> 嵌套滚动场景的完整解决方案

## 目录

1. [滑动冲突场景](#1-滑动冲突场景)
2. [View 的滑动方式](#2-view-的滑动方式)
3. [外部拦截法](#3-外部拦截法)
4. [内部拦截法](#4-内部拦截法)
5. [NestedScrolling 机制](#5-nestedscrolling-机制)
6. [实战：CoordinatorLayout 原理](#6-实战coordinatorlayout-原理)

---

## 1. 滑动冲突场景

### 典型冲突场景

{% mermaid flowchart TB %}
subgraph Scene1["场景 1: ViewPager + RecyclerView"]
VP["ViewPager<br/>左右滑动"] --> RV1["RecyclerView<br/>上下滑动"]
RV1 --> Conflict1["冲突：左右滑动和上下滑动如何区分"]
end
subgraph Scene2["场景 2: ScrollView + RecyclerView"]
SV["ScrollView<br/>上下滚动"] --> RV2["RecyclerView<br/>列表项滚动"]
RV2 --> Conflict2["冲突：到底该谁滚动？"]
end
{% endmermaid %}

---

## 2. View 的滑动方式

### 方式对比

| 方式 | 原理 | 优缺点 |
|------|------|--------|
| scrollTo/scrollBy | 移动 View 内容 | 不改变 View 位置，性能好 |
| 动画 (Translation) | 改变 View 属性 | GPU 加速，性能好 |
| LayoutParams | 改变 LayoutParams | 性能差，需要 requestLayout |
| requestDisallowIntercept | 父容器不拦截 | 解决滑动冲突 |

### scrollTo / scrollBy 实现

```kotlin
// scrollBy: 相对滚动
view.scrollBy(100, 0)  // 向右滚动 100px

// scrollTo: 绝对滚动
view.scrollTo(100, 0)  // 滚动到 (100, 0)

// scroll 移动的是 View 的内容，不是 View 本身
// 正值表示内容向左/上移动（看起来 View 像是向右/下移动）

// 自定义可滚动 View
class ScrollView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : ViewGroup(context, attrs) {
    
    private var lastY = 0f
    private var scrollY = 0
    
    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                lastY = event.y
            }
            MotionEvent.ACTION_MOVE -> {
                val dy = (lastY - event.y).toInt()
                // 相对滚动
                scrollBy(0, dy)
                lastY = event.y
            }
        }
        return true
    }
    
    override fun scrollBy(x: Int, y: Int) {
        scrollTo(scrollX + x, scrollY + y)
    }
    
    override fun scrollTo(x: Int, y: Int) {
        // 边界检查
        val maxY = computeVerticalScrollRange() - height
        val newY = y.coerceIn(0, maxY)
        
        if (newY != scrollY) {
            scrollY = newY
            // 移动内容
            scrollTo(scrollX, scrollY)
        }
    }
}
```

### 动画方式滚动

```kotlin
// 使用 Scroller
val scroller = Scroller(context)

fun smoothScrollTo(destX: Int, destY: Int) {
    val deltaX = destX - scrollX
    val deltaY = destY - scrollY
    scroller.startScroll(scrollX, scrollY, deltaX, deltaY, 500)
    invalidate()
}

override fun computeScroll() {
    if (scroller.computeScrollOffset()) {
        scrollTo(scroller.currX, scroller.currY)
        postInvalidate()
    }
}

// 使用属性动画
ObjectAnimator.ofInt(scrollableView, "scrollY", targetY).apply {
    duration = 300
    start()
}
```

---

## 3. 外部拦截法

### 原理

{% mermaid flowchart LR %}
Down["DOWN"] --> Record["记录初始位置"]
Record --> NoIntercept["不拦截"]
Move["MOVE"] --> Judge["判断滑动方向"]
Judge --> Decide["拦截 / 不拦截"]
Up["UP"] --> End["不拦截"]
Key["关键"] --> Intercept["onInterceptTouchEvent"]
{% endmermaid %}

### 实现

```kotlin
class HorizontalViewPager : ViewGroup {
    
    private var lastX = 0f
    private var lastY = 0f
    private var isIntercept = false
    
    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                lastX = ev.x
                lastY = ev.y
                isIntercept = false  // DOWN 不拦截
            }
            
            MotionEvent.ACTION_MOVE -> {
                val dx = abs(ev.x - lastX)
                val dy = abs(ev.y - lastY)
                
                // 水平滑动超过垂直滑动，拦截
                if (dx > dy && dx > ViewConfiguration.get(context).scaledTouchSlop) {
                    isIntercept = true
                    lastX = ev.x
                }
            }
        }
        
        return isIntercept
    }
    
    override fun onTouchEvent(event: MotionEvent): Boolean {
        // 处理自己的滑动逻辑
        return true
    }
}
```

---

## 4. 内部拦截法

### 原理

{% mermaid flowchart LR %}
Parent["父容器"] --> Child["子 View 处理事件"]
Child --> Dispatch["dispatchTouchEvent"]
Down2["DOWN"] --> DReq["requestDisallowIntercept(false)"]
Move2["MOVE"] --> Boundary["滑动到边界"]
Boundary --> MReq["requestDisallowIntercept(true)"]
Up2["UP"] --> UReq["requestDisallowIntercept(false)"]
{% endmermaid %}

### 父容器实现

```kotlin
class ParentViewGroup : ViewGroup {
    
    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        // 外部拦截法：不拦截 DOWN，其他事件由子 View 决定
        return ev.action != MotionEvent.ACTION_DOWN
    }
}
```

### 子 View 实现

```kotlin
class ChildRecyclerView : RecyclerView {
    
    override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                // 不允许父容器拦截
                parent.requestDisallowInterceptTouchEvent(true)
            }
            
            MotionEvent.ACTION_MOVE -> {
                // 检测是否滚动到边界
                if (!canScrollVertically(1) || !canScrollVertically(-1)) {
                    // 滚动到边界，让父容器处理
                    parent.requestDisallowInterceptTouchEvent(false)
                }
            }
            
            MotionEvent.ACTION_UP -> {
                parent.requestDisallowInterceptTouchEvent(false)
            }
        }
        
        return super.dispatchTouchEvent(ev)
    }
}
```

---

## 5. NestedScrolling 机制

### 核心接口

```kotlin
// 父容器实现 NestedScrollingParent
class NestedParentView : ViewGroup, NestedScrollingParent {
    
    override fun onStartNestedScroll(child: View, target: View, nestedScrollAxes: Int): Boolean {
        // 是否接收嵌套滚动
        return true
    }
    
    override fun onNestedPreScroll(target: View, dx: Int, dy: Int, consumed: IntArray) {
        // 在子 View 滚动前处理
        // consumed[0] = dx 表示已消费 x 方向滚动
        // consumed[1] = dy 表示已消费 y 方向滚动
    }
    
    override fun onNestedScroll(target: View, dxConsumed: Int, dyConsumed: Int, 
                               dxUnconsumed: Int, dyUnconsumed: Int) {
        // 子 View 滚动后，处理剩余滚动
    }
    
    override fun onStopNestedScroll(target: View) {
        // 滚动结束
    }
}

// 子 View 实现 NestedScrollingChild
class NestedChildView : View, NestedScrollingChild {
    
    private val helper = NestedScrollingChildHelper(this)
    
    override fun onTouchEvent(e: MotionEvent): Boolean {
        when (e.action) {
            MotionEvent.ACTION_DOWN -> {
                // 启动嵌套滚动
                helper.startNestedScroll(View.SCROLL_AXIS_VERTICAL)
            }
            
            MotionEvent.ACTION_MOVE -> {
                // 分发嵌套滚动
                helper.dispatchNestedScroll(0, dyConsumed, 0, dyUnconsumed, null)
            }
            
            MotionEvent.ACTION_UP -> {
                // 停止嵌套滚动
                helper.stopNestedScroll()
            }
        }
        return super.onTouchEvent(e)
    }
}
```

### RecyclerView 嵌套滚动

```kotlin
// RecyclerView 已实现 NestedScrollingChild
// ScrollView 已实现 NestedScrollingParent
// 直接组合使用即可

// 外部 ScrollView
<ScrollView
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.recyclerview.widget.RecyclerView
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</ScrollView>

// 解决: 设置 RecyclerView 不处理嵌套滚动
recyclerView.layoutManager = LinearLayoutManager() {
    // 或者使用 nested scrolling
}
```

---

## 6. 实战：CoordinatorLayout 原理

### Behavior 机制

```kotlin
// 自定义 Behavior
class CustomBehavior : CoordinatorLayout.Behavior<View> {
    
    override fun layoutDependsOn(parent: CoordinatorLayout, child: View, dependency: View): Boolean {
        // 判断 child 是否依赖 dependency
        return dependency.id == R.id.dependency
    }
    
    override fun onDependentViewChanged(parent: CoordinatorLayout, child: View, dependency: View): Boolean {
        // 当 dependency 变化时，更新 child
        child.y = dependency.y + dependency.height
        return true
    }
    
    override fun onLayoutChild(parent: CoordinatorLayout, child: View, layoutDirection: Int): Boolean {
        // 自定义布局
        parent.onLayoutChild(child, layoutDirection)
        // ...
        return true
    }
}

// 使用 Behavior
<androidx.coordinatorlayout.widget.CoordinatorLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <View
        android:id="@+id/dependency"
        android:layout_width="match_parent"
        android:layout_height="100dp" />

    <View
        android:layout_width="match_parent"
        android:layout_height="100dp"
        app:layout_behavior=".CustomBehavior" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

---

## 滑动冲突解决总结

{% mermaid flowchart TB %}
Strategy["滑动冲突解决策略"] --> Outer["外部拦截法（推荐）<br/>父容器在 onInterceptTouchEvent 判断方向"]
Strategy --> Inner["内部拦截法<br/>子 View 控制父容器是否拦截"]
Strategy --> Nested["NestedScrolling（推荐）<br/>官方嵌套滚动方案，自动处理冲突"]
{% endmermaid %}

---

## 面试常问

| 问题 | 答案 |
|------|------|
| scrollBy 和 scrollTo 区别？ | scrollBy 相对滚动，scrollTo 绝对滚动 |
| 滑动冲突如何解决？ | 外部拦截法/内部拦截法/NestedScrolling |
| CoordinatorLayout 原理？ | Behavior 机制，监听依赖 View 变化 |

---

**相关文章**：
- [View 事件分发机制完全掌握](./android-view-event-dispatch.md)
- [RecyclerView 原理与优化实战](./android-recyclerview-optimization.md)
