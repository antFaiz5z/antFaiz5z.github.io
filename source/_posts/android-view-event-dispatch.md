---
title: "View 事件分发机制完全掌握"
date: 2025-02-09 12:03:51
description: "触摸事件、点击事件、拦截机制一网打尽"
comments: true
toc: true
categories:
  - Android
  - View
tags:
  - Android
  - View
  - Event Dispatch
  - Touch
---

# View 事件分发机制完全掌握

> 触摸事件、点击事件、拦截机制一网打尽

## 目录

1. [事件分发家族](#1-事件分发家族)
2. [dispatchTouchEvent 流程](#2-dispatchtouchevent-流程)
3. [onInterceptTouchEvent 拦截](#3-onintercepttouchevent-拦截)
4. [onTouchEvent 处理](#4-ontouchevent-处理)
5. [滑动冲突解决](#5-滑动冲突解决)
6. [实战案例](#6-实战案例)

---

## 1. 事件分发家族

### 三大方法

```
┌─────────────────────────────────────────────────────────────────────┐
│                       View 事件分发三大方法                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    dispatchTouchEvent(MotionEvent)                                   │
│           │                                                          │
│           ├─▶ onInterceptTouchEvent()  ← 仅 ViewGroup              │
│           │        │                                                │
│           │        ▼                                                │
│           │     true: 拦截，交给自己的 onTouchEvent                │
│           │     false: 不拦截，传递给子 View                        │
│           │                                                         │
│           └─▶ onTouchEvent(MotionEvent)                             │
│                    │                                                 │
│                    ▼                                                 │
│                 true: 消费事件                                       │
│                 false: 向上传递给父 View 的 onTouchEvent            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 事件类型

| 事件 | 含义 |
|------|------|
| ACTION_DOWN | 手指按下 |
| ACTION_MOVE | 手指移动 |
| ACTION_UP | 手指抬起 |
| ACTION_CANCEL | 事件被取消 |

---

## 2. dispatchTouchEvent 流程

### ViewGroup 分发流程

```kotlin
// ViewGroup.dispatchTouchEvent 核心逻辑
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean handled = false;
    
    if (onInterceptTouchEvent(ev)) {
        // 拦截：自己处理
        handled = onTouchEvent(ev);
    } else {
        // 不拦截：分发给子 View
        handled = child.dispatchTouchEvent(ev);
    }
    
    return handled;
}
```

### 完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                 dispatchTouchEvent 完整流程                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    DOWN 事件                                                          │
│        │                                                             │
│        ▼                                                             │
│    dispatchTouchEvent(DOWN)                                          │
│        │                                                             │
│        ├─▶ onInterceptTouchEvent(DOWN)                              │
│        │        │                                                    │
│        │        ├─▶ true ──▶ onTouchEvent(DOWN)                    │
│        │        │                              │                    │
│        │        │                              ▼                    │
│        │        │                         自己处理                    │
│        │        │                         后续事件也给自己            │
│        │        │                                                    │
│        │        └─▶ false ──▶ dispatchChildTouchEvent(DOWN)        │
│        │                                    │                        │
│        │                                    ▼                        │
│        │                              子View 处理                     │
│        │                                    │                        │
│        │                              如果子View 不处理               │
│        │                                    │                        │
│        │                                    ▼                        │
│        │                              onTouchEvent(DOWN)             │
│        │                                                             │
│        ▼                                                             │
│    如果 DOWN 被处理: 后续 MOVE/UP 继续给自己                         │
│    如果 DOWN 没处理: 后续事件不再传递                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. onInterceptTouchEvent 拦截

### 拦截场景

```kotlin
class MyViewGroup @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : ViewGroup(context, attrs) {
    
    private var lastX = 0f
    private var lastY = 0f
    
    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                lastX = ev.x
                lastY = ev.y
                // DOWN 事件默认不拦截
                return false
            }
            
            MotionEvent.ACTION_MOVE -> {
                val dx = abs(ev.x - lastX)
                val dy = abs(ev.y - lastY)
                
                // 水平滑动距离大，拦截
                if (dx > dy && dx > ViewConfiguration.get(context).scaledTouchSlop) {
                    return true  // 拦截，后续事件给自己
                }
                
                return false
            }
        }
        
        return super.onInterceptTouchEvent(ev)
    }
}
```

### 拦截机制要点

```kotlin
// 关键点 1: DOWN 事件必须返回 false
override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
    if (ev.action == MotionEvent.ACTION_DOWN) {
        return false  // 必须返回 false，否则无法传递后续事件
    }
    return true
}

// 关键点 2: 一旦拦截，后续事件直接给自己
// DOWN: 不拦截，传递给子 View
// MOVE: 拦截了
// UP: 直接给自己（不再调用 onInterceptTouchEvent）
```

---

## 4. onTouchEvent 处理

### View 处理逻辑

```kotlin
// View.onTouchEvent 核心
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            // 获取焦点（如果可点击）
            if (isFocusable() || isInTouchMode()) {
                requestFocus();
            }
            break;
            
        case MotionEvent.ACTION_MOVE:
            // 处理滑动逻辑
            break;
            
        case MotionEvent.ACTION_UP:
            // 处理点击逻辑
            if (!pointInView(x, y, touchSlop)) {
                // 触摸在 View 外，移除焦点
                clearFocus();
            }
            
            // 执行点击
            if (mPerformClick == null) {
                mPerformClick = new PerformClick();
            }
            post(mPerformClick);
            break;
    }
    
    // View 只要可点击就消费事件
    return isClickable() || isLongClickable();
}
```

### 点击与长按

```kotlin
class ClickableView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {
    
    // 设置点击监听
    fun setOnClickListener(listener: OnClickListener?) {
        // ...
    }
    
    // 设置长按监听
    fun setOnLongClickListener(listener: OnLongClickListener?) {
        // ...
    }
    
    // onTouchEvent 中处理
    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_UP -> {
                // 执行点击回调
                performClick()
            }
            MotionEvent.ACTION_DOWN -> {
                // 执行长按检测
                checkForLongClick()
            }
        }
        return true  // 可点击就返回 true
    }
    
    fun performClick(): Boolean {
        // 调用 OnClickListener.onClick
        return true
    }
}
```

---

## 5. 滑动冲突解决

### 场景：ViewPager 内部嵌套 RecyclerView

```kotlin
// ViewPager.onInterceptTouchEvent
override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
    // 如果是水平滑动，ViewPager 拦截
    // 如果是垂直滑动，不拦截，传递给 RecyclerView
    
    when (ev.action) {
        MotionEvent.ACTION_DOWN -> {
            mLastMotionX = ev.x
            mLastMotionY = ev.y
            // 不拦截 DOWN
            return false
        }
        
        MotionEvent.ACTION_MOVE -> {
            val deltaX = abs(ev.x - mLastMotionX)
            val deltaY = abs(ev.y - mLastMotionY)
            
            // 水平滑动 > 垂直滑动，拦截
            if (deltaX > deltaY) {
                return true  // 拦截
            }
            return false  // 不拦截，给子 View
        }
    }
    
    return false
}
```

### 外部拦截法

```kotlin
class ParentViewGroup : ViewGroup {
    
    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        var intercept = false
        
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                intercept = false
                // 记录初始位置
                lastX = ev.x
                lastY = ev.y
            }
            
            MotionEvent.ACTION_MOVE -> {
                val deltaX = abs(ev.x - lastX)
                val deltaY = abs(ev.y - lastY)
                
                // 水平滑动时拦截
                if (deltaX > deltaY * 2) {
                    intercept = true
                } else {
                    intercept = false
                }
            }
            
            MotionEvent.ACTION_UP -> {
                intercept = false
            }
        }
        
        return intercept
    }
}
```

### 内部拦截法

```kotlin
// 父容器不拦截所有事件
class ParentViewGroup : ViewGroup {
    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        return ev.action == MotionEvent.ACTION_MOVE
    }
}

// 子 View 处理滑动
class ChildView : View {
    
    override fun dispatchTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                parent.requestDisallowInterceptTouchEvent(true)
            }
            MotionEvent.ACTION_MOVE -> {
                if (isHorizontalScroll) {
                    parent.requestDisallowInterceptTouchEvent(true)
                }
            }
            MotionEvent.ACTION_UP -> {
                parent.requestDisallowInterceptTouchEvent(false)
            }
        }
        
        return super.dispatchTouchEvent(event)
    }
}
```

---

## 6. 实战案例

### 手势检测器 GestureDetector

```kotlin
class MyView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {
    
    // 手势检测器
    private val gestureDetector = GestureDetector(context, object : GestureDetector.SimpleOnGestureListener() {
        
        override fun onDown(e: MotionEvent): Boolean = true
        
        override fun onSingleTapUp(e: MotionEvent): Boolean {
            // 单击
            return true
        }
        
        override fun onDoubleTap(e: MotionEvent): Boolean {
            // 双击
            return true
        }
        
        override fun onScroll(e1: MotionEvent?, e2: MotionEvent, 
                            distanceX: Float, distanceY: Float): Boolean {
            // 滑动
            return true
        }
        
        override fun onFling(e1: MotionEvent?, e2: MotionEvent,
                           velocityX: Float, velocityY: Float): Boolean {
            // 快速滑动
            return true
        }
        
        override fun onLongPress(e: MotionEvent) {
            // 长按
        }
    })
    
    override fun onTouchEvent(event: MotionEvent): Boolean {
        // 委托给手势检测器
        return gestureDetector.onTouchEvent(event) || super.onTouchEvent(event)
    }
}
```

### ScaleGestureDetector 缩放

```kotlin
class ZoomableImageView : ImageView {
    
    private var scaleFactor = 1f
    
    private val scaleGestureDetector = ScaleGestureDetector(
        context,
        object : ScaleGestureDetector.SimpleOnScaleGestureListener() {
            override fun onScale(detector: ScaleGestureDetector): Boolean {
                scaleFactor *= detector.scaleFactor
                scaleFactor = max(0.5f, min(scaleFactor, 5f))
                scaleX = scaleFactor
                scaleY = scaleFactor
                return true
            }
        }
    )
    
    override fun onTouchEvent(event: MotionEvent): Boolean {
        scaleGestureDetector.onTouchEvent(event)
        return true
    }
}
```

---

## 事件分发流程总结

```
完整流程:

┌─────────────────────────────────────────────────────────────────────┐
│  Activity                                                            │
│    │                                                                  │
│    ▼ dispatchTouchEvent                                              │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ ViewGroup                                                        ││
│  │   │                                                              ││
│  │   ▼ dispatchTouchEvent                                          ││
│  │   ├─▶ onInterceptTouchEvent(DOWN) ──▶ false ──▶ 子 View        ││
│  │   │                                          │                  ││
│  │   │                                          ▼                  ││
│  │   │                                    dispatchTouchEvent      ││
│  │   │                                    onTouchEvent           ││
│  │   │                                          │                  ││
│  │   │                                    return false ──▶ 父     ││
│  │   │                                                              ││
│  │   │                                      return true ──▶ 结束   ││
│  │   │                                                              ││
│  │   ▼ onTouchEvent(DOWN) ──▶ false ──▶ Activity.onTouchEvent    ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

---

## 面试常问

| 问题 | 答案 |
|------|------|
| DOWN 事件谁先收到？ | Activity → ViewGroup → View |
| onIntercept 返回 true 会怎样？ | 后续事件不传递给子 View |
| 子 View 不处理会怎样？ | 传递给父 View 的 onTouchEvent |
| 点击和长按哪个先触发？ | 长按检测在 DOWN 时开始 |

---

**相关文章**：
- [View 绘制流程深度解析](./android-view-drawing-process.md)
- [View 滚动与滑动冲突解决](./android-view-scroll-conflict.md)
