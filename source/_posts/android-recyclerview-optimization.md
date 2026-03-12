---
title: "RecyclerView 原理与优化实战"
date: 2023-03-17 23:23:19
description: "从源码到实践的深度解析 RecyclerView 架构"
comments: true
toc: true
categories:
  - Android
  - UI
tags:
  - Android
  - RecyclerView
  - Performance
  - UI
---

# RecyclerView 原理与优化实战

> 从源码到实践的深度解析

## 目录

1. [RecyclerView 架构](#1-recyclerview-架构)
2. [回收池机制](#2-回收池机制)
3. [LayoutManager 布局](#3-layoutmanager-布局)
4. [Adapter 原理](#4-adapter-原理)
5. [性能优化](#5-性能优化)
6. [常见问题解决](#6-常见问题解决)

---

## 1. RecyclerView 架构

### 四大核心组件

```
┌─────────────────────────────────────────────────────────────────────┐
│                     RecyclerView 架构                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    ┌─────────────────────────────────────────────────────────┐     │
│    │  RecyclerView                                            │     │
│    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │     │
│    │  │  Adapter    │  │  LayoutMgr  │  │  Recycler   │     │     │
│    │  │  数据绑定   │  │  布局管理   │  │  回收池     │     │     │
│    │  └─────────────┘  └─────────────┘  └─────────────┘     │     │
│    │         │                │                │              │     │
│    │         └────────────────┼────────────────┘              │     │
│    │                          ▼                               │     │
│    │              ┌─────────────────────┐                     │     │
│    │              │    ViewHolder      │                     │     │
│    │              │  (itemView + bind) │                     │     │
│    │              └─────────────────────┘                     │     │
│    └─────────────────────────────────────────────────────────┘     │
│                                                                      │
│    Adapter:    数据 → ViewHolder                                    │
│    LayoutManager: 测量 + 布局 + 滚动                                │
│    Recycler:   缓存 + 回收 ViewHolder                              │
│    ViewHolder: ItemView 的包装                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 创建 RecyclerView

```kotlin
// XML 布局
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recyclerView"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />

// 代码配置
val recyclerView = findViewById<RecyclerView>(R.id.recyclerView)

// 设置 LayoutManager (必选)
recyclerView.layoutManager = LinearLayoutManager(this)

// 设置 Adapter (必选)
recyclerView.adapter = MyAdapter(items)

// 设置 ItemDecoration (可选)
recyclerView.addItemDecoration(DividerItemDecoration(this, LinearLayoutManager.VERTICAL))

// 设置 ItemAnimator (可选)
recyclerView.itemAnimator = DefaultItemAnimator()
```

---

## 2. 回收池机制

### RecyclerView 缓存层级

```
┌─────────────────────────────────────────────────────────────────────┐
│                     RecyclerView 缓存层级                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   第1层: Attached 状态                                              │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  mAttachedScrap (ArrayList)                                 │  │
│   │  屏幕内可见的 ViewHolder，不真正移除                         │  │
│   │  用于 data binding 和 position 复用                        │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   第2层: mCacheViews (ArrayList) 默认 2 个                         │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  移出屏幕的 ViewHolder，保存 ViewType                        │  │
│   │  快速复用，不重新 bind                                      │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   第3层: RecyclerViewPool                                          │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  多个 RecyclerView 共享的缓存池                             │  │
│   │  按 ViewType 分类存储                                       │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   第4层: Adapter 创建                                              │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  所有缓存都无效，创建新的 ViewHolder                          │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   命中顺序: Attached → Cache → Pool → create                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Recycler 核心方法

```kotlin
// 获取 ViewHolder
fun getViewHolderForPosition(position: Int): ViewHolder {
    // 1. 从 AttachedScrap 获取
    val scrapHolder = getScrapViewForPosition(position, TYPE_INVALID, true)
    if (scrapHolder != null) return scrapHolder
    
    // 2. 从 CacheViews 获取
    val cachedHolder = getCachedViewForPosition(position)
    if (cachedHolder != null) return cachedHolder
    
    // 3. 从 Pool 获取
    val poolHolder = getRecycledViewPool().getRecycledView(viewType)
    if (poolHolder != null) return poolHolder
    
    // 4. 创建新的
    return createViewHolder(viewType)
}

// 回收 ViewHolder
fun recycleViewHolder(holder: ViewHolder) {
    // 检查是否需要保留在 Cache
    if (forceRecycle || holder.isRecyclable()) {
        // 加入缓存池
        recycledViewPool.putRecycledView(holder)
    }
}
```

---

## 3. LayoutManager 布局

### LinearLayoutManager 原理

```kotlin
// onLayoutChildren 实现框架
override fun onLayoutChildren(recycler: Recycler, state: RecyclerView.State) {
    // 1. 分离所有子 View
    detachAndScrapAttachedViews(recycler)
    
    // 2. 填充子 View
    fill(recycler, state, layoutState)
    
    // 3. 回收不可见的 View
    recycleByLayoutState(recycler, layoutState)
}

// fill 方法
private int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
                RecyclerView.State state) {
    // 从可滚动方向开始填充
    while ((layoutState.mLayoutDirection == LayoutState.LAYOUT_START 
            && layoutState.hasMore(state))
            || layoutState.mLayoutDirection == LayoutState.LAYOUT_END) {
        
        // 获取 ViewHolder
        View view = recycler.getViewForPosition(layoutState.mCurrentPosition);
        
        // 添加到 RecyclerView
        addView(view);
        
        // 测量
        measureChildWithMargins(view, 0, 0);
        
        // 布局
        layoutDecorated(view, left, top, right, bottom);
        
        // 更新位置
        layoutState.mCurrentPosition += layoutState.mItemDirection;
    }
}
```

### 滚动实现

```kotlin
// scrollVerticallyBy
override fun scrollVerticallyBy(dy: Int, recycler: Recycler, state: RecyclerView.State): Int {
    // 1. 处理填充
    if (dy > 0) {
        // 向上滚动，填充底部
        fill(recycler, state, layoutState)
    }
    
    // 2. 移动子 View
    offsetChildrenVertical(-dy)
    
    // 3. 回收不可见 View
    recyclerByLayoutState(recycler, state)
    
    return dy
}
```

---

## 4. Adapter 原理

### ViewHolder 绑定

```kotlin
// 标准 Adapter
class MyAdapter(private val items: List<Item>) : RecyclerView.Adapter<MyAdapter.ViewHolder>() {
    
    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val textView: TextView = itemView.findViewById(R.id.text)
    }
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_layout, parent, false)
        return ViewHolder(view)
    }
    
    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.textView.text = items[position].text
    }
    
    override fun getItemCount() = items.size
}

// DiffUtil 差量更新
class MyAdapter : RecyclerView.Adapter<ViewHolder>() {
    
    private val oldItems = mutableListOf<Item>()
    private val newItems = mutableListOf<Item>()
    
    fun submitList(newList: List<Item>) {
        val diffCallback = DiffCallback(oldItems, newList)
        val diffResult = DiffUtil.calculateDiff(diffCallback)
        
        newItems.clear()
        newItems.addAll(newList)
        
        diffResult.dispatchUpdatesTo(this)
    }
}

class DiffCallback(
    private val oldList: List<Item>,
    private val newList: List<Item>
) : DiffUtil.Callback() {
    
    override fun getOldListSize() = oldList.size
    override fun getNewListSize() = newList.size
    
    override fun areItemsTheSame(oldPos: Int, newPos: Int): Boolean {
        return oldList[oldPos].id == newList[newPos].id
    }
    
    override fun areContentsTheSame(oldPos: Int, newPos: Int): Boolean {
        return oldList[oldPos] == newList[newPos]
    }
}
```

---

## 5. 性能优化

### RecyclerView 优化清单

```kotlin
// 1. 设置固定大小
recyclerView.setHasFixedSize(true)

// 2. 使用 Stable IDs (StableId)
// Adapter
class MyAdapter : RecyclerView.Adapter<ViewHolder>() {
    override fun getItemId(position: Int): Long = items[position].id
}

// RecyclerView
recyclerView.setItemId(getItemId = true)

// 3. 预取
val layoutManager = LinearLayoutManager(this).apply {
    initialPrefetchItemCount = 4
}

// 4. 合并子项
recyclerView.setItemViewCacheSize(20)

// 5. 关闭默认 item 动画 (性能敏感场景)
recyclerView.itemAnimator = null
```

### ViewHolder 优化

```kotlin
// ❌ 错误：每次 bind 都查找 View
class BadViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    fun bind(item: Item) {
        itemView.findViewById<TextView>(R.id.text).text = item.text  // 每次 find
    }
}

// ✅ 正确：缓存 View
class GoodViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    private val textView: TextView = itemView.findViewById(R.id.text)  // 缓存
    
    fun bind(item: Item) {
        textView.text = item.text
    }
}
```

### 数据层优化

```kotlin
// ❌ 错误：每次刷新整个列表
adapter.notifyDataSetChanged()

// ✅ 正确：使用 DiffUtil
adapter.submitList(newList)

// ✅ 正确：局部刷新
notifyItemChanged(position)           // 单项变化
notifyItemRangeChanged(position, count) // 范围变化
notifyItemRemoved(position)           // 删除
notifyItemInserted(position)         // 插入
notifyItemMoved(from, to)            // 移动
```

---

## 6. 常见问题解决

### 解决嵌套 RecyclerView 滚动冲突

```kotlin
// 方法1: 设置 nested scrolling
innerRecyclerView.isNestedScrollingEnabled = false

// 方法2: 使用自定义 Behavior
recyclerView.layoutManager = object : LinearLayoutManager(context) {
    override fun canScrollVertically() = false
}
```

### ItemDecoration 分割线

```kotlin
// 简单分割线
class SimpleDividerDecoration : ItemDecoration() {
    
    private val divider: Drawable? = null
    
    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        val left = parent.paddingLeft
        val right = parent.width - parent.paddingRight
        
        for (i in 0 until parent.childCount - 1) {
            val child = parent.getChildAt(i)
            val params = child.layoutParams as RecyclerView.LayoutParams
            val top = child.bottom + params.bottomMargin
            val bottom = top + divider?.intrinsicHeight ?: 0
            
            divider?.setBounds(left, top, right, bottom)
            divider?.draw(c)
        }
    }
}
```

---

## 面试常问

| 问题 | 答案 |
|------|------|
| RecyclerView 缓存层级？ | Attached → Cache → Pool → create |
| notifyDataSetChanged vs DiffUtil？ | DiffUtil 高效，局部更新 |
| 为什么滑动卡顿？ | 绑定太重、View 过多、GC |

---

## 总结

```
RecyclerView 核心:
─────────────────────────────────────────
1. Recycler: 缓存机制，决定复用效率
2. Adapter: 数据绑定，DiffUtil 差量更新
3. LayoutManager: 布局 + 滚动
4. 优化: 预取、固定大小、DiffUtil
─────────────────────────────────────────
```

---

**相关文章**：
- [View 绘制流程深度解析](./android-view-drawing-process.md)
- [View 性能优化与卡顿分析](./android-view-performance-optimization.md)
