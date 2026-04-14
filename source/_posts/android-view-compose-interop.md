---
title: "Android View 与 Compose 互操作避坑指南"
date: 2021-10-06 15:22:21
description: "混合开发中常见的陷阱与解决方案 Compose in View (传统 View 中使用 Compose)"
comments: true
toc: true
categories:
  - Android
  - Compose
tags:
  - Android
  - Jetpack Compose
  - View
  - Interop
---

# Android View 与 Compose 互操作避坑指南

> 混合开发中常见的陷阱与解决方案

## 目录

1. [Compose in View (传统 View 中使用 Compose)](#1-compose-in-view-传统-view-中使用-compose)
2. [View in Compose (Compose 中使用传统 View)](#2-view-in-compose-compose-中使用传统-view)
3. [常见崩溃与解决方案](#3-常见崩溃与解决方案)
4. [生命周期同步问题](#4-生命周期同步问题)
5. [性能最佳实践](#5-性能最佳实践)

---

## 1. Compose in View (传统 View 中使用 Compose)

### 基本用法

```kotlin
// 在 XML 布局中
<androidx.compose.ui.platform.ComposeView
    android:id="@+id/compose_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />

// 在 Activity/Fragment 中
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        findViewById<ComposeView>(R.id.compose_view).apply {
            setContent {
                MyComposable()
            }
        }
    }
}
```

### 在 Fragment 中使用

```kotlin
class MyFragment : Fragment() {
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            setContent {
                MyComposable()
            }
        }
    }
}
```

---

## 2. View in Compose (Compose 中使用传统 View)

### AndroidView 基础

```kotlin
@Composable
fun WebViewScreen() {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                webViewClient = WebViewClient()
                loadUrl("https://example.com")
            }
        },
        update = { webView ->
            // View 更新时调用
            webView.loadUrl("https://new-url.com")
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

### 常见场景：MapView

```kotlin
@Composable
fun MapViewScreen() {
    val mapView = rememberMapViewWithLifecycle()
    
    AndroidView(
        factory = { mapView },
        modifier = Modifier.fillMaxSize()
    )
}

@Composable
fun rememberMapViewWithLifecycle(): MapView {
    val context = LocalContext.current
    val mapView = remember {
        MapView(context).apply {
            onCreate(null)
        }
    }
    
    // ✅ 正确的生命周期管理
    DisposableEffect(Unit) {
        val lifecycle = LocalLifecycleOwner.current.lifecycle
        lifecycle.addObserver(MapLifecycleObserver(mapView))
        
        onDispose {
            lifecycle.removeObserver(MapLifecycleObserver(mapView))
        }
    }
    
    return mapView
}

class MapLifecycleObserver(
    private val mapView: MapView
) : LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun onResume() = mapView.onResume()
    
    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun onPause() = mapView.onPause()
}
```

---

## 3. 常见崩溃与解决方案

### 崩溃 1：Can't create handler inside thread that has not called Looper.prepare()

```kotlin
// ❌ 错误：在非主线程更新 Compose
thread {
    someState.value = "new value"  // 崩溃！
}

// ✅ 正确：切换到主线程
thread {
    Handler(Looper.getMainLooper()).post {
        someState.value = "new value"
    }
}

// 或者使用 coroutines
viewModelScope.launch(Dispatchers.Main) {
    someState.value = "new value"
}
```

### 崩溃 2：Composing on non-main thread

```kotlin
// ❌ 错误：后台线程调用 Composable
thread {
    setContent {  // 崩溃！
        Text("Hello")
    }
}

// ✅ 正确：主线程执行
runOnUiThread {
    composeView.setContent {
        Text("Hello")
    }
}
```

### 崩溃 3：View is not attached to a window manager

```kotlin
@Composable
fun BadExample(view: View) {
    // ❌ 错误：在重组中调用 view 的方法
    AndroidView(
        factory = { view },
        update = { v ->
            // 危险：View 可能已 detached
            v.visibility = View.VISIBLE
        }
    )
}

// ✅ 正确：检查 view.isAttachedToWindow
AndroidView(
    factory = { view },
    update = { v ->
        if (v.isAttachedToWindow) {
            v.visibility = View.VISIBLE
        }
    }
)
```

---

## 4. 生命周期同步问题

### 问题：Compose 生命周期与 View 不一致

{% mermaid flowchart TB %}
Lifecycle["Activity / Fragment 生命周期"] --> OnCreate["onCreate"]
OnCreate --> OnStart["onStart"]
OnStart --> OnResume["onResume"]
OnResume --> OnPause["onPause"]
OnCreate --> ComposeCreated["Compose Created"]
OnStart --> ComposeStarted["Started"]
OnResume --> ComposeResumed["Resumed"]
OnPause --> ComposePaused["Paused"]
ComposePaused --> Risk["问题：Composition 可能在 onDestroy 后继续执行"]
{% endmermaid %}

### 解决：使用 rememberUpdatedState

```kotlin
@Composable
fun DataObserver(data: SomeData) {
    // ✅ 始终获取最新数据，不触发重组
    val latestData by rememberUpdatedState(data)
    
    LaunchedEffect(data) {
        // 使用 latestData
    }
}
```

### 解决：使用 DisposableEffect 管理资源

```kotlin
@Composable
fun SensorView(context: Context) {
    val sensorManager = context.getSystemService<SensorManager>()
    var rotation by remember { mutableFloatStateOf(0f) }
    
    DisposableEffect(Unit) {
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ROTATION_VECTOR)
        
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent?) {
                rotation = event?.values?.get(0) ?: 0f
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }
        
        sensorManager.registerListener(
            listener, 
            sensor, 
            SensorManager.SENSOR_DELAY_UI
        )
        
        // ✅ 正确清理
        onDispose {
            sensorManager.unregisterListener(listener)
        }
    }
    
    Text("Rotation: $rotation")
}
```

---

## 5. 性能最佳实践

### 避免重组时重建 View

```kotlin
@Composable
fun OptimizedAndroidView() {
    val view = remember {
        SomeExpensiveView(context).apply {
            // 一次性配置
            isFocusable = true
        }
    }
    
    AndroidView(
        factory = { view },  // 只创建一次
        update = { v ->
            // 更新逻辑
        }
    )
}
```

### 使用 InteropViewModel 桥接状态

```kotlin
// 桥接：ViewModel 暴露 Compose 可用的 State
class InteropViewModel : ViewModel() {
    private val _text = MutableStateFlow("Hello")
    val text: StateFlow<String> = _text.asStateFlow()
    
    fun updateText(newText: String) {
        _text.value = newText
    }
}

// Compose 端
@Composable
fun Screen(viewModel: InteropViewModel = hiltViewModel()) {
    val text by viewModel.text.collectAsState()
    
    AndroidView(
        factory = { context ->
            TextView(context).apply {
                // 使用 Flow 收集
                viewModel.lifecycleScope.launch {
                    viewModel.text.collect { value ->
                        text = value
                    }
                }
            }
        }
    )
}
```

---

## 互操作场景对照表

| 场景 | 方案 |
|------|------|
| XML 中嵌入 Compose | `ComposeView` |
| Fragment 中嵌入 Compose | `ComposeView` |
| Compose 中使用 View | `AndroidView` |
| Compose 中使用 WebView | `AndroidView` |
| Compose 中使用 MapView | `AndroidView` + `DisposableEffect` |
| View 中使用 Compose | `ComposeView.setContent` |
| 需要共享状态 | ViewModel + StateFlow |

---

## 常见错误检查清单

| 错误 | 解决方案 |
|------|---------|
| 在后台线程调用 Composable | 切换到主线程 |
| View 已 detach 后更新 | 检查 `isAttachedToWindow` |
| 内存泄漏 | 使用 `DisposableEffect` 清理 |
| 重组导致 View 重建 | 使用 `remember` 缓存 |
| 生命周期不同步 | 使用 `LifecycleObserver` |

---

**相关文章**：
- [Jetpack Compose 性能优化实战](./jetpack-compose-performance-optimization.md)
- [从零理解 Activity 启动流程](./android-activity-startup.md)
