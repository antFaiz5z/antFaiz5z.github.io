---
title: "Kotlin Coroutines 取消与异常处理最佳实践"
date: 2022-08-05 08:01:46
description: "本文深入探讨 Kotlin Coroutines 的取消机制与异常处理，帮助你写出健壮的非阻塞代码。"
comments: true
toc: true
categories:
  - Android
  - Kotlin
tags:
  - Kotlin
  - Coroutines
  - CancellationException
  - Exception Handling
---

# Kotlin Coroutines 取消与异常处理最佳实践

> 本文深入探讨 Kotlin Coroutines 的取消机制与异常处理，帮助你写出健壮的非阻塞代码。

## 目录

1. [协程取消的基本原理](#1-协程取消的基本原理)
2. [ CancellationException 的真相](#2-cancellationexception-的真相)
3. [SupervisorJob vs Job：异常传播的差异](#3-supervisorjob-vs-job异常传播的差异)
4. [try-catch 与 CoroutineExceptionHandler](#4-try-catch-与-coroutineexceptionhandler)
5. [最佳实践总结](#5-最佳实践总结)

---

## 1. 协程取消的基本原理

### 取消的触发机制

协程取消并非强制终止，而是**协作式**的。当调用 `job.cancel()` 时：

```kotlin
fun main() = runBlocking {
    val job = launch {
        repeat(1000) { i ->
            println("Working $i ...")
            delay(100)  // 重要：检查取消状态
        }
    }
    
    delay(250)
    println("Canceling job...")
    job.cancel()
    job.join()
    println("Job canceled!")
}
```

**关键点**：协程必须在**Suspension Point**（延迟点）检查取消状态。

### 没有检查取消会怎样？

```kotlin
// ❌ 错误：永远不会响应取消
launch {
    repeat(1000) { i ->
        // CPU 密集型任务，没有 suspension point
        computeHeavyStuff()
    }
}

// ✅ 正确：定期检查取消
launch {
    repeat(1000) { i ->
        computeHeavyStuff()
        yield()  // 让出协程，检查是否被取消
    }
}
```

---

## 2. CancellationException 的真相

### 它是特殊的异常

```kotlin
launch {
    try {
        delay(1000)
    } catch (e: CancellationException) {
        println("协程被取消: $e")
        // 必须重新抛出或处理
        throw e
    }
}
```

**重要特性**：
- `CancellationException` 是**正常的**控制流，不是错误
- 它**不会**触发 `CoroutineExceptionHandler`
- 它**会**传播到父协程（除非使用 `SupervisorJob`）

### 区分取消和真正异常

```kotlin
launch {
    try {
        // 模拟网络错误
        throw IOException("Network error")
    } catch (e: IOException) {
        // 真正的异常，需要处理
        println("Handle error: $e")
    }
}
```

---

## 3. SupervisorJob vs Job：异常传播的差异

### 普通 Job 的异常传播

```
┌─────────────────────────────────────────────────────────────┐
│                      普通 Job                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   子协程 A (失败)  ──▶  CancellationException ──▶ 父协程     │
│                              │                              │
│                              ▼                              │
│                        全部取消                             │
│                                                             │
│   子协程 B ──▶ 也被取消（即使 B 没问题）                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### SupervisorJob 的异常传播

```
┌─────────────────────────────────────────────────────────────┐
│                   SupervisorJob                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   子协程 A (失败)  ──▶  CancellationException ──▶ 父协程     │
│                              │                              │
│                              ▼                              │
│                      只取消 A 自己                           │
│                                                             │
│   子协程 B ──▶ 继续运行（不受影响）                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 代码示例

```kotlin
// ❌ 普通 Job：一个失败，全部失败
val parentJob = CoroutineScope(Dispatchers.Main).launch {
    launch {
        delay(100)
        throw RuntimeException("A failed")
    }
    launch {
        delay(200)
        println("B never runs!")  // 永远不会执行
    }
}

// ✅ SupervisorJob：一个失败，不影响其他
val supervisor = CoroutineScope(Dispatchers.Main + SupervisorJob())
supervisor.launch {
    launch {
        delay(100)
        throw RuntimeException("A failed")
    }
    launch {
        delay(200)
        println("B still runs!")  // 会正常执行
    }
}
```

---

## 4. try-catch 与 CoroutineExceptionHandler

### CoroutineExceptionHandler 的作用

```kotlin
val handler = CoroutineExceptionHandler { context, exception ->
    println("Caught: $exception")
}

CoroutineScope(Dispatchers.Main + handler).launch {
    throw RuntimeException("Error!")
}
```

**注意**：以下情况**不会**触发 Handler：
1. 协程被取消 (`CancellationException`)
2. `launch` 在 `coroutineScope` 内（而非 `CoroutineScope`）

### 正确组合：Handler + SupervisorJob

```kotlin
val scope = CoroutineScope(
    Dispatchers.Main + 
    SupervisorJob() + 
    CoroutineExceptionHandler { _, throwable ->
        println("全局异常: $throwable")
    }
)

// 各个子任务互不影响
scope.launch { taskA() }
scope.launch { taskB() }
scope.launch { taskC() }
```

### 结构化并发的重要性

```kotlin
// ❌ 错误：取消外部作用域
suspend fun badExample() {
    val scope = CoroutineScope(Dispatchers.IO)
    try {
        scope.launch { task1() }
    } catch (e: Exception) {
        // task1 仍在运行！
    }
}

// ✅ 正确：使用 coroutineScope
suspend fun goodExample() = coroutineScope {
    try {
        launch { task1() }
    } catch (e: Exception) {
        // task1 会被正确取消
    }
}
```

---

## 5. 最佳实践总结

### 1. 使用 SupervisorJob 处理多任务

```kotlin
class UserRepository(
    private val api: ApiService
) {
    private val scope = CoroutineScope(
        Dispatchers.IO + 
        SupervisorJob() + 
        CoroutineExceptionHandler { _, _ -> }
    )
    
    fun fetchUserAndPosts() {
        scope.launch { fetchUser() }      // 失败不影响
        scope.launch { fetchPosts() }     // 失败不影响
    }
}
```

### 2. 在 suspend 函数中正确处理异常

```kotlin
suspend fun <T> safeCall(block: suspend () -> T): Result<T> {
    return try {
        Result.success(block())
    } catch (e: CancellationException) {
        throw e  // 重新抛出，不包装
    } catch (e: Exception) {
        Result.failure(e)
    }
}

// 使用
val result = safeCall { api.getUser() }
result.onSuccess { user -> /* ... */ }
result.onFailure { error -> /* ... */ }
```

### 3. ViewModel 中正确使用 viewModelScope

```kotlin
class MyViewModel(
    private val repository: MyRepository
) : ViewModel() {
    
    // viewModelScope 已经是 SupervisorJob
    // 子协程失败不会取消整个 ViewModel
    
    fun loadData() {
        viewModelScope.launch {
            val user = repository.getUser()
            _user.value = user
        }
    }
}
```

### 4. 取消时清理资源

```kotlin
launch {
    try {
        withContext(Dispatchers.IO) {
            file.use { /* 操作文件 */ }
        }
    } catch (e: CancellationException) {
        // 清理工作
        file.close()
        throw e
    }
}

// 或者使用 finally
launch {
    val file = openFile()
    try {
        // 操作文件
    } finally {
        file.close()  // 始终执行
    }
}
```

### 5. 常见错误对比

| 错误写法 | 正确写法 |
|---------|---------|
| `launch { /* 无取消检查 */ }` | `launch { /* 定期 yield() */ }` |
| `CoroutineScope(Dispatchers.Main).launch` | `viewModelScope.launch` 或 `lifecycleScope.launch` |
| `try-catch` 包裹 `launch` | 使用 `CoroutineExceptionHandler` |
| `Job()` | `SupervisorJob()` 当需要独立失败 |

---

## 参考资料

- [Kotlin Coroutines 官方文档](https://kotlinlang.org/docs/coroutines-guide.html)
- [协程异常处理深度解析](https://elizarov.medium.com/structured-concurrency-722de7848a07)

---

**相关文章**：
- [LiveData 为什么不够好 - 迁移到 Flow 完全指南](./android-livedata-to-flow.md)
- [从零理解 Activity 启动流程](./android-activity-startup.md)
