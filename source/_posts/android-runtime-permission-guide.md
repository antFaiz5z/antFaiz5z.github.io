---
title: "Android 13+ 运行时权限适配完全指南"
date: 2020-08-27 22:32:36
description: "完整覆盖 Android 13 (API 33) 到 Android 15 的权限变化"
comments: true
toc: true
categories:
  - Android
  - Permissions
tags:
  - Android
  - Permissions
  - Android 13
  - Android 15
---

# Android 13+ 运行时权限适配完全指南

> 完整覆盖 Android 13 (API 33) 到 Android 15 的权限变化

## 目录

1. [权限模型概述](#1-权限模型概述)
2. [普通权限与危险权限](#2-普通权限与危险权限)
3. [Android 13 重大变化：细化权限](#3-android-13-重大变化细化权限)
4. [权限请求最佳实践](#4-权限请求最佳实践)
5. [权限拒绝处理与引导](#5-权限拒绝处理与引导)
6. [代码实现模板](#6-代码实现模板)

---

## 1. 权限模型概述

### Android 权限分类

{% mermaid flowchart LR %}
subgraph Normal["普通权限 (Normal)"]
N1["网络"]
N2["振动"]
N3["蓝牙"]
N4["WiFi"]
N5["无需授权，系统自动授予"]
end
subgraph Dangerous["危险权限 (Dangerous)"]
D1["位置（精确 / 模糊）"]
D2["相机"]
D3["麦克风"]
D4["存储（媒体 / 文件）"]
D5["通讯录 / 日历 / 身体传感器 / 短信 / 电话"]
end
subgraph Special["特殊权限 (Special)"]
S1["SYSTEM_ALERT_WINDOW"]
S2["WRITE_SETTINGS"]
S3["REQUEST_IGNORE_BATTERY_OPTIMIZATIONS"]
end
{% endmermaid %}

---

## 2. 普通权限与危险权限

### 普通权限示例

```kotlin
// 无需任何声明，自动授予
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.VIBRATE" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

### 危险权限组

```kotlin
// 危险权限需要用户授权
// 权限组示例：

LOCATION:     ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION
CAMERA:       CAMERA
MICROPHONE:  RECORD_AUDIO
CONTACTS:     READ_CONTACTS, WRITE_CONTACTS, GET_ACCOUNTS
CALENDAR:     READ_CALENDAR, WRITE_CALENDAR
STORAGE:      READ_EXTERNAL_STORAGE, WRITE_EXTERNAL_STORAGE
PHONE:        READ_PHONE_STATE, CALL_PHONE, READ_CALL_LOG
SMS:          SEND_SMS, READ_SMS, RECEIVE_MMS
```

---

## 3. Android 13 重大变化：细化权限

### Android 13 主要变化

| 版本 | 变化 |
|------|------|
| API 33 | 细分存储权限、媒体权限、通知权限 |
| API 34 | 更严格的日历/提醒权限 |
| API 35 | 前景服务权限细化 |

### 1) 媒体权限细化 (API 33+)

```kotlin
// ❌ Android 12 及以前
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

// ✅ Android 13+ 按类型请求
// 读取图片/视频
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />

// 读取音频
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

// 读取视频
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
```

### 2) 通知权限 (Android 13+)

```kotlin
// Android 13+ 必须请求通知权限
fun requestNotificationPermission() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        val permission = Manifest.permission.POST_NOTIFICATIONS
        if (checkSelfPermission(permission) != PackageManager.PERMISSION_GRANTED) {
            requestPermissions(arrayOf(permission), REQUEST_NOTIFICATION)
        }
    }
}
```

### 3) 精确位置 (Android 12+)

```kotlin
// Android 12+ 可以只请求模糊位置
// 但精确位置权限仍然需要

// 模糊位置
Manifest.permission.ACCESS_COARSE_LOCATION

// 精确位置
Manifest.permission.ACCESS_FINE_LOCATION

// 建议：同时请求两个，系统会显示最精确的那个
requestPermissions(
    arrayOf(
        Manifest.permission.ACCESS_FINE_LOCATION,
        Manifest.permission.ACCESS_COARSE_LOCATION
    ),
    REQUEST_LOCATION
)
```

---

## 4. 权限请求最佳实践

### 权限检查封装

```kotlin
object PermissionHelper {
    
    // 检查权限
    fun hasPermission(context: Context, permission: String): Boolean {
        return ContextCompat.checkSelfPermission(
            context, permission
        ) == PackageManager.PERMISSION_GRANTED
    }
    
    // 检查多个权限
    fun hasPermissions(context: Context, permissions: Array<String>): Boolean {
        return permissions.all {
            hasPermission(context, it)
        }
    }
    
    // 检查是否应该显示权限说明
    fun shouldShowRationale(activity: Activity, permission: String): Boolean {
        return activity.shouldShowRequestPermissionRationale(permission)
    }
    
    // Android 13+ 通知权限特殊处理
    fun hasNotificationPermission(context: Context): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            ContextCompat.checkSelfPermission(
                context,
                Manifest.permission.POST_NOTIFICATIONS
            ) == PackageManager.PERMISSION_GRANTED
        } else {
            true  // Android 13 以下不需要
        }
    }
}
```

### 权限请求模板 (Activity)

```kotlin
class PermissionActivity : AppCompatActivity() {
    
    companion object {
        const val REQUEST_CODE = 1001
        
        // 权限集合
        val LOCATION_PERMISSIONS = arrayOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION
        )
        
        val CAMERA_PERMISSIONS = arrayOf(
            Manifest.permission.CAMERA
        )
        
        val MEDIA_PERMISSIONS = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            arrayOf(
                Manifest.permission.READ_MEDIA_IMAGES,
                Manifest.permission.READ_MEDIA_VIDEO,
                Manifest.permission.READ_MEDIA_AUDIO
            )
        } else {
            arrayOf(Manifest.permission.READ_EXTERNAL_STORAGE)
        }
    }
    
    // 需要请求的权限
    private var pendingPermissions: Array<String>? = null
    private var onGranted: (() -> Unit)? = null
    private var onDenied: ((List<String>) -> Unit)? = null
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 读取传入的权限参数
        pendingPermissions = intent.getStringArrayExtra("permissions")
        
        if (pendingPermissions.isNullOrEmpty()) {
            finish()
            return
        }
        
        // 检查是否已授权
        if (PermissionHelper.hasPermissions(this, pendingPermissions!!)) {
            onGranted?.invoke()
            finish()
            return
        }
        
        // 请求权限
        requestPermissions(pendingPermissions!!, REQUEST_CODE)
    }
    
    @Deprecated("Deprecated in Java")
    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray
    ) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        
        if (requestCode != REQUEST_CODE) return
        
        val deniedList = mutableListOf<String>()
        
        grantResults.forEachIndexed { index, result ->
            if (result != PackageManager.PERMISSION_GRANTED) {
                deniedList.add(permissions[index])
            }
        }
        
        if (deniedList.isEmpty()) {
            // 全部授权
            onGranted?.invoke()
        } else {
            // 部分/全部拒绝
            onDenied?.invoke(deniedList)
        }
        
        finish()
    }
    
    // Builder 模式简化调用
    class Builder(private val activity: Activity) {
        private val permissions = mutableListOf<String>()
        private var onGranted: (() -> Unit)? = null
        private var onDenied: ((List<String>) -> Unit)? = null
        private var onAllDenied: (() -> Unit)? = null
        
        fun addPermissions(vararg permissions: String) = apply {
            this.permissions.addAll(permissions)
        }
        
        fun onGranted(action: () -> Unit) = apply {
            this.onGranted = action
        }
        
        fun onDenied(action: (List<String>) -> Unit) = apply {
            this.onDenied = action
        }
        
        fun onAllDenied(action: () -> Unit) = apply {
            this.onAllDenied = action
        }
        
        fun request() {
            if (permissions.isEmpty()) {
                onGranted?.invoke()
                return
            }
            
            val intent = Intent(activity, PermissionActivity::class.java).apply {
                putExtra("permissions", permissions.toTypedArray())
            }
            
            activity.startActivityForResult(intent, REQUEST_CODE)
            
            // 注意：实际实现需要通过 LiveData/Callback 传递结果
            // 这里简化展示
        }
    }
}
```

---

## 5. 权限拒绝处理与引导

### 引导用户打开设置

```kotlin
fun showPermissionSettings(activity: Activity) {
    AlertDialog.Builder(activity)
        .setTitle("权限需要")
        .setMessage("应用需要相关权限才能正常运行。请在设置中开启权限。")
        .setPositiveButton("去设置") { _, _ ->
            val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                data = Uri.fromParts("package", activity.packageName, null)
            }
            activity.startActivity(intent)
        }
        .setNegativeButton("取消", null)
        .show()
}
```

### 优雅降级策略

```kotlin
@Composable
fun LocationScreen() {
    var hasLocationPermission by remember {
        mutableStateOf(
            PermissionHelper.hasPermission(
                LocalContext.current,
                Manifest.permission.ACCESS_FINE_LOCATION
            )
        )
    }
    
    when {
        hasLocationPermission -> {
            LocationContent()
        }
        shouldShowRequestPermissionRationale(Manifest.permission.ACCESS_FINE_LOCATION) -> {
            PermissionRationaleContent(
                onRequest = { /* 再次请求 */ }
            )
        }
        else -> {
            PermissionDeniedContent(
                onOpenSettings = { /* 打开设置 */ }
            )
        }
    }
}
```

---

## 6. 代码实现模板

### Activity Result API (推荐)

```kotlin
// 1. 定义权限 contract
class ManageCameraPermission {
    companion object {
        val CONTRACT = object : ActivityResultContract<Void?, Boolean>() {
            override fun createIntent(context: Context, input: Void?): Intent {
                return Intent(MediaStore.ACTION_REQUEST_CAMERA)
            }
            
            override fun parseResult(resultCode: Int, intent: Intent?): Boolean {
                return resultCode == Activity.RESULT_OK
            }
        }
    }
}

// 2. 使用 registerForActivityResult
class MyActivity : AppCompatActivity() {
    
    private val cameraPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            openCamera()
        } else {
            showCameraPermissionDenied()
        }
    }
    
    private val locationPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        val granted = permissions.entries.filter { it.value }
        val denied = permissions.entries.filterNot { it.value }
        
        when {
            granted.isNotEmpty() -> useLocation(granted.keys)
            denied.isNotEmpty() -> showLocationDenied(denied.keys)
        }
    }
    
    fun requestCamera() {
        cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
    }
    
    fun requestLocation() {
        locationPermissionLauncher.launch(
            arrayOf(
                Manifest.permission.ACCESS_FINE_LOCATION,
                Manifest.permission.ACCESS_COARSE_LOCATION
            )
        )
    }
}
```

### Compose 权限处理

```kotlin
@Composable
fun rememberPermissionState(
    permission: String
): PermissionState {
    val context = LocalContext.current
    
    return rememberPermissionState(
        contract = ActivityResultContracts.RequestPermission(),
        onPermissionResult = { /* handle result */ }
    )
}

// 或者使用 accompanist
@Composable
fun LocationPermissionRequest() {
    val permissionState = rememberMultiplePermissionsState(
        permissions = listOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION
        )
    )
    
    Button(
        onClick = { permissionState.launchMultiplePermissionRequest() }
    ) {
        Text("Grant Permission")
    }
}
```

---

## 权限清单速查表

| 权限 | API 级别 | 声明方式 | 说明 |
|------|---------|---------|------|
| POST_NOTIFICATIONS | 33 | runtime | Android 13+ 通知 |
| READ_MEDIA_IMAGES | 33 | runtime | 读取图片 |
| READ_MEDIA_VIDEO | 33 | runtime | 读取视频 |
| READ_MEDIA_AUDIO | 33 | runtime | 读取音频 |
| READ_EXTERNAL_STORAGE | <33 | runtime | 读取存储 |
| CAMERA | all | runtime | 相机 |
| RECORD_AUDIO | all | runtime | 麦克风 |
| ACCESS_FINE_LOCATION | all | runtime | 精确位置 |
| ACCESS_COARSE_LOCATION | all | runtime | 模糊位置 |
| INTERNET | all | manifest | 网络 |

---

## 总结

1. **Android 13+ 变化**：细粒度媒体权限、通知权限
2. **最佳实践**：使用 Activity Result API
3. **优雅降级**：提供替代功能或引导用户设置
4. **测试**：在真实设备上测试各权限场景

---

**相关文章**：
- [Jetpack Compose 性能优化实战](./jetpack-compose-performance-optimization.md)
- [Gradle 构建速度优化实战 30%](./android-gradle-build-optimization.md)
