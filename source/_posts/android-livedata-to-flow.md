---
title: "LiveData 为什么不够好 - 迁移到 Flow 完全指南"
date: 2022-09-21 15:35:49
description: "从 LiveData 迁移到 StateFlow/SharedFlow 的完整实践"
comments: true
toc: true
categories:
  - Android
  - Kotlin
tags:
  - Android
  - LiveData
  - Flow
  - StateFlow
---

# LiveData 为什么不够好 - 迁移到 Flow 完全指南

> 从 LiveData 迁移到 StateFlow/SharedFlow 的完整实践

## 目录

1. [LiveData 的局限性](#1-livedata-的局限性)
2. [Flow 核心概念](#2-flow-核心概念)
3. [迁移策略](#3-迁移策略)
4. [ViewModel 中的迁移](#4-viewmodel-中的迁移)
5. [Compose 中的使用](#5-compose-中的使用)
6. [常见问题解决](#6-常见问题解决)

---

## 1. LiveData 的局限性

### LiveData 的问题

| 问题 | 描述 |
|------|------|
| **线程切换不可控** | `postValue` 会切换到主线程，但逻辑不透明 |
| **缺少取消机制** | 无法取消正在发射的数据 |
| **不是 Flow** | 无法与 Kotlin 协程生态集成 |
| **背压不支持** | 数据量大会导致问题 |
| **错误处理弱** | 没有 `try-catch` 机制 |

### LiveData 行为示例

```kotlin
// ❌ LiveData 的问题
class MyViewModel : ViewModel() {
    private val _data = MutableLiveData<Data>()
    val data: LiveData<Data> = _data
    
    fun loadData() {
        // 问题1: 没有取消机制
        repository.getData().observeForever { result ->
            _data.postValue(result)  // 问题2: 自动切换到主线程
        }
    }
}

// 问题3: 错误处理困难
// LiveData 没有 error 状态概念
```

---

## 2. Flow 核心概念

### StateFlow vs SharedFlow

{% mermaid flowchart LR %}
subgraph S["StateFlow"]
S1["有初始值"]
S2["始终有当前值"]
S3["适合 UI 状态"]
S4["replay = 1"]
S5["示例: val uiState = StateFlow(...)"]
S6["UI State<br/>始终保持最新"]
end
subgraph H["SharedFlow"]
H1["无初始值"]
H2["只有订阅者时才会发射"]
H3["适合事件 / 一次性消息"]
H4["replay 可配置"]
H5["示例: val events = SharedFlow(...)"]
H6["One-time Events"]
end
{% endmermaid %}

### 创建 Flow

```kotlin
// StateFlow - 有初始值
private val _uiState = MutableStateFlow(UiState())
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// SharedFlow - 无初始值，可配置 replay
private val _events = MutableSharedFlow<Event>()
val events: SharedFlow<Event> = _events.asSharedFlow()

// 带 replay 的 SharedFlow
val events = MutableSharedFlow<Event>(
    replay = 1  // 新订阅者也会收到最后一条
)

// Flow 转换
val userFlow: Flow<User> = flow {
    emit(loadFromCache())
    emit(loadFromNetwork())
}
```

---

## 3. 迁移策略

### Step 1: 将 LiveData 替换为 StateFlow

```kotlin
// ❌ Before: LiveData
class OldViewModel : ViewModel() {
    private val _user = MutableLiveData<User>()
    val user: LiveData<User> = _user
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            try {
                val result = api.getUser(userId)
                _user.value = result  // 主线程赋值
            } catch (e: Exception) {
                _user.value = null
            }
        }
    }
}

// ✅ After: StateFlow
class NewViewModel : ViewModel() {
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            try {
                val result = api.getUser(userId)
                _user.value = result
            } catch (e: Exception) {
                _user.value = null
            }
        }
    }
}
```

### Step 2: 处理 loading 状态

```kotlin
// ❌ LiveData 方式: 多个 LiveData
class OldViewModel : ViewModel() {
    val isLoading = MutableLiveData<Boolean>()
    val user = MutableLiveData<User>()
    val error = MutableLiveData<Error?>()
}

// ✅ StateFlow 方式: 单一 UI State
data class UiState(
    val isLoading: Boolean = false,
    val user: User? = null,
    val error: Error? = null
)

class NewViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true)
            try {
                val user = api.getUser(userId)
                _uiState.value = _uiState.value.copy(
                    isLoading = false,
                    user = user
                )
            } catch (e: Exception) {
                _uiState.value = _uiState.value.copy(
                    isLoading = false,
                    error = e
                )
            }
        }
    }
}
```

### Step 3: 处理事件 (一次性消息)

```kotlin
// ❌ LiveData 处理事件的问题
// 容易因为配置变更导致事件重复触发

// ✅ 使用 SharedFlow 处理事件
sealed class UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
}

class EventViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()
    
    fun onUserClick(userId: String) {
        viewModelScope.launch {
            _events.emit(UiEvent.Navigate("/user/$userId"))
        }
    }
}

// 在 Activity/Fragment 中收集
lifecycleScope.launch {
    viewModel.events.collect { event ->
        when (event) {
            is UiEvent.ShowSnackbar -> showSnackbar(event.message)
            is UiEvent.Navigate -> navigate(event.route)
        }
    }
}
```

---

## 4. ViewModel 中的迁移

### 完整示例

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    // StateFlow: UI 状态
    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
    
    // SharedFlow: 一次性事件
    private val _events = MutableSharedFlow<UserEvent>()
    val events: SharedFlow<UserEvent> = _events.asSharedFlow()
    
    // 加载用户
    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(isLoading = true)
            
            try {
                val user = repository.getUser(userId)
                _uiState.value = _uiState.value.copy(
                    isLoading = false,
                    user = user
                )
            } catch (e: Exception) {
                _uiState.value = _uiState.value.copy(
                    isLoading = false,
                    error = e.message
                )
                _events.emit(UserEvent.ShowError(e.message ?: "Unknown error"))
            }
        }
    }
    
    // 刷新
    fun refresh() {
        _uiState.value.user?.id?.let { loadUser(it) }
    }
}

// UI State
data class UserUiState(
    val isLoading: Boolean = false,
    val user: User? = null,
    val error: String? = null
)

// Events
sealed class UserEvent {
    data class ShowError(val message: String) : UserEvent()
    object NavigateBack : UserEvent()
}
```

---

## 5. Compose 中的使用

### collectAsState vs collectAsStateWithLifecycle

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    // ❌ 不推荐: 可能导致重组
    val uiState by viewModel.uiState.collectAsState()
    
    // ✅ 推荐: 感知生命周期
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // ✅✅ 最佳: 指定初始值，避免重组
    val uiState by viewModel.uiState.collectAsStateWithLifecycle(
        initialValue = UserUiState()
    )
    
    // UI 渲染
    when {
        uiState.isLoading -> LoadingView()
        uiState.error != null -> ErrorView(uiState.error)
        uiState.user != null -> UserView(uiState.user)
    }
}
```

### 依赖

```kotlin
// build.gradle
dependencies {
    // Lifecycle Runtime KTX
    implementation "androidx.lifecycle:lifecycle-runtime-ktx:2.7.0"
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0"
    implementation "androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0"
    
    // 或使用 Compose BOM
    implementation platform('androidx.compose:compose-bom:2024.02.00')
    implementation 'androidx.compose.runtime:runtime-livedata'
}
```

### 使用 LaunchedEffect 收集事件

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    
    // 收集一次性事件
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UserEvent.ShowError -> {
                    snackbarHostState.showSnackbar(event.message)
                }
                is UserEvent.NavigateBack -> {
                    // 处理导航
                }
            }
        }
    }
    
    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) {
        // Content
    }
}
```

---

## 6. 常见问题解决

### Q1: 首次发射初始值导致 UI 闪烁

```kotlin
// 问题: StateFlow 初始值触发 UI 重组
val uiState by viewModel.uiState.collectAsStateWithLifecycle()

// ✅ 解决: 设置初始值与 StateFlow 相同
val uiState by viewModel.uiState.collectAsStateWithLifecycle(
    initialValue = UserUiState()  // 与 MutableStateFlow 初始值一致
)
```

### Q2: 协程取消问题

```kotlin
// ❌ 错误: 在 viewModelScope 中启动无限循环
class BadViewModel : ViewModel() {
    private val _data = MutableStateFlow("")
    val data: StateFlow<String> = _data.asStateFlow()
    
    init {
        viewModelScope.launch {
            while (true) {
                _data.value = produceData()  // 不会自动取消
                delay(1000)
            }
        }
    }
}

// ✅ 正确: 使用 repeatOnLifecycle
class GoodViewModel : ViewModel() {
    private val _data = MutableStateFlow("")
    val data: StateFlow<String> = _data.asStateFlow()
    
    init {
        viewModelScope.launch {
            repeatOnLifecycle(State) {
                while (true) {
                    _data.value = produceData()
                    delay(1000)
                }
            }
        }
    }
}
```

### Q3: 多个数据源合并

```kotlin
// 使用 combine 合并多个 Flow
class ProfileViewModel(
    private val userRepository: UserRepository,
    private val settingsRepository: SettingsRepository
) : ViewModel() {
    
    private val userFlow = userRepository.getUser()
    private val settingsFlow = settingsRepository.getSettings()
    
    val profileState: StateFlow<ProfileState> = combine(
        userFlow,
        settingsFlow
    ) { user, settings ->
        ProfileState(
            name = user.name,
            avatar = user.avatar,
            theme = settings.theme,
            notifications = settings.notifications
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = ProfileState()
    )
}
```

---

## 迁移检查清单

| 检查项 | 状态 |
|--------|------|
| LiveData → StateFlow | ✅ |
| loading/error 状态合并到 UI State | ✅ |
| 一次性事件使用 SharedFlow | ✅ |
| 使用 collectAsStateWithLifecycle | ✅ |
| 使用 repeatOnLifecycle 收集 | ✅ |
| 移除 observe() / observeForever() | ✅ |

---

## 总结

{% mermaid flowchart LR %}
subgraph LiveData["LiveData 缺点"]
LD1["线程切换不透明"]
LD2["无取消机制"]
LD3["无背压支持"]
LD4["错误处理困难"]
LD5["非 Kotlin 一等公民"]
end
subgraph Flow["Flow 优势"]
F1["协程控制，线程安全"]
F2["协程自动取消"]
F3["背压策略可选"]
F4["try-catch 天然支持"]
F5["与 Kotlin 生态完美集成"]
end
{% endmermaid %}

---

**相关文章**：
- [Kotlin Coroutines 取消与异常处理最佳实践](./kotlin-coroutines-cancellation-and-exception-handling.md)
- [Jetpack Compose 性能优化实战](./jetpack-compose-performance-optimization.md)
