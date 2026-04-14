---
title: "Android 内存泄漏分析与解决"
date: 2023-01-18 12:15:14
description: "常见内存泄漏场景与解决方案 内存泄漏基础"
comments: true
toc: true
categories:
  - Android
  - Performance
tags:
  - Android
  - Memory Leak
  - LeakCanary
  - Performance
---

# Android 内存泄漏分析与解决

> 常见内存泄漏场景与解决方案

## 目录

1. [内存泄漏基础](#1-内存泄漏基础)
2. [常见泄漏场景](#2-常见泄漏场景)
3. [LeakCanary 使用指南](#3-leakcanary-使用指南)
4. [静态变量导致的泄漏](#4-静态变量导致的泄漏)
5. [非静态内部类与 Handler](#5-非静态内部类与-handler)
6. [资源未关闭](#6-资源未关闭)
7. [Compose 中的泄漏](#7-compose-中的泄漏)

---

## 1. 内存泄漏基础

### 什么是内存泄漏

{% mermaid flowchart TB %}
subgraph Normal["正常内存使用"]
N1["创建对象"] --> N2["使用"] --> N3["不再需要"] --> N4["GC 回收"]
end
subgraph Leak["内存泄漏"]
L1["创建对象"] --> L2["使用"] --> L3["不再需要"] --> L4["无法回收"]
L4 --> L5["原因：仍被引用"]
end
{% endmermaid %}

### 内存泄漏的影响

- **应用卡顿**: 垃圾回收频繁触发
- **OOM 崩溃**: 内存不足
- **电池消耗**: GC 占用 CPU

---

## 2. 常见泄漏场景

| 场景 | 原因 | 解决方案 |
|------|------|---------|
| 静态 View | View 持有 Activity 引用 | 使用 `WeakReference` |
| Handler | 延迟消息持有 Activity | 移除消息/使用弱引用 |
| 非静态内部类 | 隐式持有外部类引用 | 改为静态类/弱引用 |
| Thread | 线程持有 Activity | 使用 Lifecycle-aware 组件 |
| 监听器未移除 | 监听器持有对象引用 | 在 onDestroy 移除 |
| Adapter 未清理 | ViewHolder 持有 View | 使用 ViewBinding |

---

## 3. LeakCanary 使用指南

### 集成 LeakCanary

```kotlin
// build.gradle (app)
dependencies {
    // LeakCanary
    debugImplementation "com.squareup.leakcanary:leakcanary-android:2.12"
}

// 自动检测，无需其他配置
```

### LeakCanary 工作原理

{% mermaid flowchart TD %}
LifeEnd["Activity / Fragment 生命周期结束"] --> Watch["监听对象是否被回收"]
Watch --> Check["5 秒后检查引用是否仍然存在"]
Check -->|可回收| NormalLeak["正常，忽略"]
Check -->|仍存在| Dump["Dump Heap，分析引用链"]
Dump --> Report["生成泄漏报告，显示引用链"]
{% endmermaid %}

### 典型泄漏报告解读

{% mermaid flowchart TB %}
Root["GC ROOT: MainThread (id=1)"] --> Activity["this$0 (MainActivity)<br/>外部类 this"]
Activity --> HandlerLeak["mHandler (Handler)"]
HandlerLeak --> Callback["mCallback (Runnable)"]
Callback --> Inner["this$0 (anonymous class)<br/>匿名内部类持有外部引用"]
Inner --> Issue["问题：Handler 延迟消息持有 Activity 引用"]
{% endmermaid %}

---

## 4. 静态变量导致的泄漏

### 问题代码

```kotlin
class MainActivity : AppCompatActivity() {
    
    companion object {
        // ❌ 泄漏: 静态变量持有 Activity
        var instance: MainActivity? = null
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        instance = this  // 泄漏！
    }
    
    override fun onDestroy() {
        super.onDestroy()
        instance = null
    }
}
```

### 静态 View 泄漏

```kotlin
class MainActivity : AppCompatActivity() {
    
    companion object {
        // ❌ 泄漏: 静态 View 持有 Activity
        var textView: TextView? = null
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        textView = findViewById(R.id.text_view)  // 泄漏！
    }
}
```

### 解决方案

```kotlin
class MainActivity : AppCompatActivity() {
    
    // ✅ 使用弱引用
    companion object {
        var instance: WeakReference<MainActivity>? = null
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        instance = WeakReference(this)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        instance?.clear()
        instance = null
    }
}
```

---

## 5. 非静态内部类与 Handler

### Handler 泄漏

```kotlin
class MainActivity : AppCompatActivity() {
    
    // ❌ 泄漏: Handler 持有 Activity
    private val handler = Handler(Looper.getMainLooper()) {
        // 匿名内部类持有外部 Activity 引用
        println("Message processed")
        true
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 发送延迟消息
        handler.sendEmptyMessageDelayed(0, 10000)  // 10秒后执行
    }
    
    override fun onDestroy() {
        super.onDestroy()
        // ❌ 问题: Activity 已销毁，但消息还在队列中
        // handler 持有 Activity 引用，导致泄漏
    }
}
```

### 解决方案

```kotlin
class MainActivity : AppCompatActivity() {
    
    // ✅ 方案1: 使用静态 Handler
    private val handler = Handler(Looper.getMainLooper()) {
        println("Message processed")
        true
    }
    
    // ✅ 方案2: 移除所有消息
    override fun onDestroy() {
        handler.removeCallbacksAndMessages(null)
        super.onDestroy()
    }
}

// ✅ 方案3: 使用 WeakReference Handler
class SafeHandler @Nullable constructor(
    looper: Looper
) : Handler(looper) {
    
    private var mActivity: WeakReference<Activity>? = null
    
    fun setActivity(activity: Activity) {
        mActivity = WeakReference(activity)
    }
    
    override fun handleMessage(msg: Message) {
        val activity = mActivity?.get()
        if (activity != null && !activity.isFinishing) {
            // 处理消息
        }
    }
}
```

### 内部类泄漏

```kotlin
class MainActivity : AppCompatActivity() {
    
    // ❌ 泄漏: 非静态内部类
    private class AsyncTask extends AsyncTask<Void, Void, String> {
        // 隐式持有外部 Activity 引用
        override fun doInBackground(Void... voids) {
            // 后台任务
            return ""
        }
        
        override fun onPostExecute(String s) {
            // 可能访问已销毁的 Activity
        }
    }
    
    // ✅ 解决方案: 静态内部类
    private static class SafeAsyncTask extends AsyncTask<Void, Void, String> {
        private WeakReference<Activity> activityRef;
        
        SafeAsyncTask(Activity activity) {
            this.activityRef = new WeakReference<>(activity);
        }
        
        override fun onPostExecute(String s) {
            Activity activity = activityRef.get();
            if (activity != null && !activity.isFinishing) {
                // 安全访问
            }
        }
    }
}
```

---

## 6. 资源未关闭

### 常见未关闭资源

| 资源 | 后果 | 解决方案 |
|------|------|---------|
| Cursor | 内存泄漏 | `close()` |
| Stream | 内存泄漏 | `use {}` |
| Bitmap | OOM | `recycle()` |
| 注册监听器 | 泄漏 | `unregister()` |

### 正确示例

```kotlin
// ❌ 错误: 未关闭 Cursor
fun getUsers(): List<User> {
    val db = database.readableDatabase
    val cursor = db.query("users", null, null, null, null, null, null)
    
    val users = mutableListOf<User>()
    while (cursor.moveToNext()) {
        users.add(cursor.toUser())
    }
    // ❌ 未关闭 cursor
    
    return users
}

// ✅ 正确: 使用 close 或 use
fun getUsers(): List<User> {
    val db = database.readableDatabase
    val cursor = db.query("users", null, null, null, null, null, null)
    
    return cursor.use {
        val users = mutableListOf<User>()
        while (cursor.moveToNext()) {
            users.add(cursor.toUser())
        }
        users
    }
}
```

### 文件流

```kotlin
// ❌ 错误: 未关闭流
fun readFile(path: String): String {
    val fis = FileInputStream(path)
    return fis.bufferedReader().readText()
}

// ✅ 正确: 使用 use
fun readFile(path: String): String {
    return FileInputStream(path).use { fis ->
        fis.bufferedReader().readText()
    }
}
```

---

## 7. Compose 中的泄漏

### remember 生命周期问题

```kotlin
@Composable
fun Screen() {
    // ✅ 正确: remember 与 Compose 生命周期一致
    var state by remember { mutableStateOf("") }
    
    // ⚠️ 注意: 避免在 remember 中持有 Context
    val context = LocalContext.current
    val sharedPrefs = remember {
        // ⚠️ 仍可能导致泄漏，如果 Composable 生命周期异常
        context.getSharedPreferences("prefs", Context.MODE_PRIVATE)
    }
}
```

### DisposableEffect 正确清理

```kotlin
@Composable
fun SensorScreen(context: Context) {
    var rotation by remember { mutableFloatStateOf(0f) }
    
    DisposableEffect(Unit) {
        val sensorManager = context.getSystemService<SensorManager>()
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ROTATION_VECTOR)
        
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent?) {
                rotation = event?.values?.get(0) ?: 0f
            }
        }
        
        sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_UI)
        
        // ✅ 正确清理
        onDispose {
            sensorManager.unregisterListener(listener)
        }
    }
}
```

### 避免重组泄漏

```kotlin
@Composable
fun BadExample() {
    var list by remember { mutableStateOf(listOf<String>()) }
    
    // ❌ 每次重组都添加新监听器
    LaunchedEffect(Unit) {
        someFlow.collect { data ->
            list = list + data
        }
    }
}

@Composable
fun GoodExample() {
    var list by remember { mutableStateOf(listOf<String>()) }
    
    // ✅ 正确: 使用正确的作用域
    LaunchedEffect(Unit) {
        someFlow.collect { data ->
            list = list + data
        }
    }
}
```

---

## 内存分析工具

| 工具 | 用途 |
|------|------|
| LeakCanary | 自动检测 Activity/Fragment 泄漏 |
| Android Profiler | 实时内存监控 |
| MAT (Memory Analyzer Tool) | 深入分析堆内存 |
| jmap/jhat | 命令行内存分析 |

---

## 总结

{% mermaid flowchart TB %}
LeakChecklist["内存泄漏检查清单"] --> L1["静态变量使用 WeakReference"]
LeakChecklist --> L2["Handler 在 onDestroy 移除消息"]
LeakChecklist --> L3["内部类使用静态 + WeakReference"]
LeakChecklist --> L4["资源使用 use {} 自动关闭"]
LeakChecklist --> L5["监听器在 onDestroy 移除"]
LeakChecklist --> L6["使用 LeakCanary 检测"]
{% endmermaid %}

---

**相关文章**：
- [Handler/Looper/MessageQueue 源码深度解析](./android-handler-looper-messagequeue.md)
- [Jetpack Compose 性能优化实战](./jetpack-compose-performance-optimization.md)
