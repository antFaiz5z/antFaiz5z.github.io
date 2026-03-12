---
title: "Jetpack Compose 性能优化实战"
date: 2020-09-19 01:49:03
description: "从原理到实践，打造流畅的 Compose 应用"
comments: true
toc: true
categories:
  - Android
  - Compose
tags:
  - Android
  - Jetpack Compose
  - Performance
  - UI
---

# Jetpack Compose 性能优化实战

> 从原理到实践，打造流畅的 Compose 应用

## 目录

1. [Compose 渲染原理概述](#1-compose-渲染原理概述)
2. [重组(Recomposition)优化](#2-重组recomposition优化)
3. [状态管理最佳实践](#3-状态管理最佳实践)
4. [作用域与 remember 的正确使用](#4-作用域与-remember-的正确使用)
5. [实战案例：列表性能优化](#5-实战案例列表性能优化)

---

## 1. Compose 渲染原理概述

### Composable 函数的本质

Compose 不是"命令式 UI"的替代，而是**声明式**的描述：

```kotlin
@Composable
fun MyButton(text: String, onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text(text)
    }
}
```

每次重组时：
1. Composable 函数重新执行
2. 描述新的 UI 状态
3. Compose 比较新旧"描述"（diffing）
4. 只更新实际变化的 UI

### 重组(Recomposition)触发条件

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableIntStateOf(0) }
    
    // ❌ 错误：每次重组都创建新 lambda
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
    
    // ✅ 正确：使用 lambda 引用
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

**真正触发重组的条件**：
- State 对象发生 `equals` 变化
- `@Composable` 参数变化

---

## 2. 重组(Recomposition)优化

### 问题：过度重组

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    // ❌ 问题：uiState 任何变化都导致整个屏幕重组
    Column {
        Header(uiState.user)
        Content(uiState.posts)
        Footer(uiState.timestamp)
    }
}
```

### 解决：细化状态粒度

```kotlin
// ✅ ViewModel 中拆分状态
data class UserUiState(
    val user: User = User(),
    val posts: List<Post> = emptyList(),
    val timestamp: Long = 0
)

// ❌ 还是会有问题 - 一个字段变化，整个 State 变化

// ✅ 正确做法：使用多个 StateFlow
class UserViewModel : ViewModel() {
    private val _user = MutableStateFlow(User())
    val user: StateFlow<User> = _user.asStateFlow()
    
    private val _posts = MutableStateFlow<List<Post>>(emptyList())
    val posts: StateFlow<List<Post>> = _posts.asStateFlow()
}

// ✅ 正确做法：使用 rememberSaveable + deriveStateFrom
@Composable
fun UserScreen(
    user: User,
    posts: List<Post>,
    timestamp: Long
) {
    Column {
        Header(user)         // 只有 user 变化才重组
        Content(posts)      // 只有 posts 变化才重组  
        Footer(timestamp)   // 只有 timestamp 变化才重组
    }
}
```

### 使用 DerivedStateOf 避免不必要的计算

```kotlin
@Composable
fun ArticleList(articles: List<Article>) {
    // ❌ 每次 articles 变化都重新计算
    val sortedArticles = articles.sortedByDescending { it.date }
    
    // ✅ 只有 articles 内容变化时才重新排序
    val sortedArticles by remember(articles) {
        derivedStateOf { articles.sortedByDescending { it.date } }
    }
    
    LazyColumn {
        items(sortedArticles) { article ->
            ArticleItem(article)
        }
    }
}
```

---

## 3. 状态管理最佳实践

### 避免不必要的 State 提升

```kotlin
@Composable
fun BadExample() {
    var text by remember { mutableStateOf("") }
    var list by remember { mutableStateOf(listOf<String>()) }
    
    // ❌ 状态在父组件但只在子组件使用
    ChildComponent(text = text, list = list)
}

@Composable
private fun ChildComponent(text: String, list: List<String>) {
    // 只在这里使用
    Text(text)
}
```

### 正确：状态就近管理

```kotlin
@Composable
fun GoodExample() {
    // ✅ 状态在需要的最低层级
    var text by remember { mutableStateOf("") }
    TextField(value = text, onValueChange = { text = it })
}
```

### ViewModel 状态如何暴露

```kotlin
class MyViewModel : ViewModel() {
    // ✅ 正确：暴露不可变 StateFlow
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    // ❌ 错误：暴露可变 MutableStateFlow
    val mutableState = MutableStateFlow(UiState())
}
```

---

## 4. 作用域与 remember 的正确使用

### 理解 Composable 作用域

```kotlin
@Composable
fun Parent() {
    // ❌ 错误：remember 不能跨作用域
    val data = remember { loadData() }  // 在 Parent 作用域
    
    Child()  // Child 无法访问 data
}

@Composable
private fun Child() {
    // 这里无法访问 Parent 的 remember
}
```

### 正确使用 rememberSaveable

```kotlin
@Composable
fun Screen() {
    // ✅ 跨配置变更保存状态
    var name by rememberSaveable { mutableStateOf("") }
    var age by rememberSaveable { mutableIntStateOf(0) }
    
    // ✅ 带键的 remember - 键变化时重新计算
    val cachedData by remember(key1 = userId) {
        repository.getUserData(userId)
    }
}
```

### mutableStateOf vs mutableIntStateOf

```kotlin
@Composable
fun PerformanceDemo() {
    // ❌ 使用 Any 类型，会有装箱开销
    var count by remember { mutableStateOf(0) }
    count++
    
    // ✅ 使用原始类型，避免装箱
    var count by remember { mutableIntStateOf(0) }
    count++
    
    // ✅ 推荐：使用 DoubleState, FloatState 等
    var value by remember { mutableFloatStateOf(0f) }
}
```

---

## 5. 实战案例：列表性能优化

### 原始版本：性能问题

```kotlin
@Composable
fun ArticleList(articles: List<Article>) {
    LazyColumn {
        items(articles.size) { index ->
            val article = articles[index]
            
            // ❌ 问题 1：每次都创建新对象
            ArticleCard(
                title = article.title,
                // ❌ 问题 2：lambda 每次都重新创建
                onClick = { /* handle click */ },
                // ❌ 问题 3：没有使用 key
                modifier = Modifier.fillMaxWidth()
            )
        }
    }
}
```

### 优化版本

```kotlin
@Composable
fun ArticleList(articles: List<Article>) {
    LazyColumn {
        // ✅ 使用 items 带 key
        items(
            items = articles,
            key = { it.id }  // 稳定 ID，保留重组状态
        ) { article ->
            // ✅ 使用 remember 缓存 click handler
            val onClick = remember(article.id) {
                { /* handle click */ }
            }
            
            ArticleCard(
                title = article.title,
                onClick = onClick,
                modifier = Modifier.fillMaxWidth()
            )
        }
    }
}
```

### 进阶：懒加载 + 预取

```kotlin
@Composable
fun OptimizedList(
    viewModel: ListViewModel = hiltViewModel()
) {
    val items by viewModel.items.collectAsState()
    
    LazyColumn(
        // ✅ 预加载视图之外的项
        beyondBoundsItemCount = 5
    ) {
        items(
            items = items,
            key = { it.id }
        ) { item ->
            ListItem(
                item = item,
                onClick = { viewModel.onItemClick(item.id) }
            )
        }
    }
}
```

### 使用 AsyncImage 优化图片加载

```kotlin
@Composable
fun ImageOptimized() {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .memoryCacheKey(url)
            .diskCacheKey(url)
            .build(),
        contentDescription = null,
        modifier = Modifier.fillMaxWidth(),
        // ✅ 使用 contentScale 避免重排
        contentScale = ContentScale.Crop
    )
}
```

---

## 性能检查清单

| 检查项 | 状态 |
|--------|------|
| 使用 `rememberSaveable` 保存 UI 状态 | ✅ |
| `LazyColumn` 使用 `key` 参数 | ✅ |
| 使用 `derivedStateOf` 避免重复计算 | ✅ |
| 避免在 Composable 中创建 lambda | ✅ |
| 使用原始类型 `mutableIntStateOf` | ✅ |
| 列表使用 `AsyncImage` 加载图片 | ✅ |
| `StateFlow` 暴露给 Compose | ✅ |
| 避免过度状态提升 | ✅ |

---

## 总结

Compose 性能优化的核心：

1. **减少重组**：细化状态粒度，使用 `derivedStateOf`
2. **避免重排**：为列表项提供稳定 `key`
3. **减少计算**：使用 `remember` 缓存复杂计算
4. **正确引用**：避免创建不必要的 lambda

---

**相关文章**：
- [Kotlin Coroutines 取消与异常处理最佳实践](./kotlin-coroutines-cancellation-and-exception-handling.md)
- [Android View 与 Compose 互操作避坑指南](./android-view-compose-interop.md)
