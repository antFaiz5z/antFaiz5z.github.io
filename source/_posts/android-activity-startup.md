---
title: "从零理解 Activity 启动流程"
date: 2020-08-16 07:33:04
description: "AMS → ActivityThread → Activity 完全解析"
comments: true
toc: true
categories:
  - Android
  - Framework
tags:
  - Android
  - Activity
  - Framework
  - AMS
---

# 从零理解 Activity 启动流程

> AMS → ActivityThread → Activity 完全解析

## 目录

1. [整体流程概览](#1-整体流程概览)
2. [Launcher 启动 Activity](#2-launcher-启动-activity)
3. [AMS 处理阶段](#3-ams-处理阶段)
4. [ActivityThread 执行阶段](#4-activitythread-执行阶段)
5. [Activity 生命周期回调](#5-activity-生命周期回调)
6. [关键类作用](#6-关键类作用)

---

## 1. 整体流程概览

{% mermaid flowchart LR %}
Start["startActivity()"] --> ATM["IActivityTaskManager<br/>Binder 通信"]
ATM --> AMS["AMS / ActivityTaskManagerService"]
AMS --> Starter["ActivityStarter<br/>解析 Intent / 决定启动模式"]
Starter --> AppThread["ApplicationThread"]
AppThread --> ActivityThread["ActivityThread<br/>H 发送消息"]
ActivityThread --> Launch["performLaunchActivity"]
Launch --> OnCreate["Activity.onCreate"]
{% endmermaid %}

---

## 2. Launcher 启动 Activity

### Activity.startActivity()

```kotlin
// Activity.java
public void startActivity(Intent intent) {
    startActivity(intent, null);
}

public void startActivity(Intent intent, Bundle options) {
    // 实际上是调用 instrumentation
    mInstrumentation.execStartActivity(
        this,
        mMainThread.getApplicationThread(),
        mToken,
        this,
        intent,
        -1,
        options
    );
}
```

### Instrumentation 转发

```kotlin
// Instrumentation.java
public ActivityResult execStartActivity(
    Context who, 
    IBinder contextThread,
    IBinder token, 
    Activity target,
    Intent intent,
    int requestCode,
    Bundle options
) {
    // 通过 Binder 通知系统服务
    int result = ActivityTaskManager.getService()
        .startActivity(whoThread, who.getBasePackageName(), intent,
            intent.resolveTypeIfNeeded(who.getContentResolver()),
            token, target != null ? target.mEmbeddedID : null,
            requestCode, 0, null, options);
    
    checkStartActivityResult(result, intent);
    return null;
}
```

---

## 3. AMS 处理阶段

### ActivityTaskManagerService (ATMS)

```java
// ActivityTaskManagerService.java
public final int startActivity(IApplicationThread caller,
        String callingPackage, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType,
        resultTo, resultWho, requestCode, startFlags, profilerInfo,
        bOptions, UserHandle.getCallingUserId());
}
```

### ActivityStarter 解析

```java
// ActivityStarter.java
int startActivityLocked(...){
    // 1. 解析 Intent
    // 2. 检查权限
    // 3. 决定启动模式 (standard, singleTop, singleTask, singleInstance)
    // 4. 处理 task 相关逻辑
    // 5. 检查目标 Activity 是否已在运行
    
    // 最终调用
    return startActivityUnchecked(r, sourceRecord, voiceSession,
            voiceInteractor, startFlags, true, checkedOptions, inTask,
            restrictedBgActivity, intentGrants);
}
```

### ActivityStack 任务管理

```java
// ActivityStack.java
private int startActivityUnchecked(...) {
    // 根据启动模式决定:
    // - 是否创建新 Task
    // - 是否复用已有 Activity
    // - 是否清除栈顶
    
    // 最终调用
    mRootActivityContainer.startActivityLocked(r, newTask,
            doResume, checkedOptions, inTask, restrictedBgActivity);
    
    return ActivityManager.START_SUCCESS;
}
```

---

## 4. ActivityThread 执行阶段

### ApplicationThread 接收

```java
// ActivityThread.ApplicationThread
public void scheduleTransaction(ClientTransaction transaction) {
    // 通过 H Handler 发送消息到主线程
    ActivityThread.this.sendMessage(
        ActivityThread.H.EXECUTE_TRANSACTION,
        transaction
    );
}
```

### H Handler 处理

```java
// ActivityThread.java
class H extends Handler {
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case EXECUTE_TRANSACTION:
                final ClientTransaction transaction = (ClientTransaction) msg.obj;
                mTransactionExecutor.execute(transaction);
                break;
                
            case LAUNCH_ACTIVITY:
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                handleLaunchActivity(r, null, null);
                break;
        }
    }
}
```

### handleLaunchActivity

```java
private Activity handleLaunchActivity(ActivityClientRecord r,
        Intent customIntent, ActivityInfo activityInfo, 
        Configuration overrideConfig, CompatibilityInfo compatInfo,
        String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean isForward, boolean probe, ActivityStack activityStack) {
    
    // 1. 创建 Activity
    Activity activity = performLaunchActivity(r, customIntent);
    
    // 2. 恢复状态
    if (activity != null) {
        handleResumeActivity(r.token, false, 
            r.isForward, "", r.procState, null);
    }
    
    return activity;
}
```

### performLaunchActivity 核心

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // 1. 获取 ComponentName
    ComponentName component = r.intent.getComponent();
    
    // 2. 创建 Activity 实例 (通过 ClassLoader)
    Activity activity = mInstrumentation.newActivity(
        cl, component.getClassName(), r.intent);
    
    // 3. 创建 Application (如果还没有)
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    
    // 4. 创建 Context
    Context appContext = createBaseContext(activity, r);
    
    // 5. 关联 Context
    activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window, r.configCallback);
    
    // 6. 调用 onCreate
    if (r.state != null) {
        r.state.setClassLoader(activity.getClassLoader());
    }
    mInstrumentation.callActivityOnCreate(activity, r.state);
    
    return activity;
}
```

---

## 5. Activity 生命周期回调

### onCreate 调用链

{% mermaid flowchart TB %}
LaunchActivity["performLaunchActivity"] --> CallCreate["Instrumentation.callActivityOnCreate(activity, bundle)"]
CallCreate --> PerformCreate["Activity.performCreate(bundle)"]
PerformCreate --> OnCreate2["Activity.onCreate(bundle)"]
OnCreate2 --> OnStart["Activity.onStart()"]
OnStart --> OnResume["Activity.onResume()"]
{% endmermaid %}

### Activity.attach() - 创建 Window

```java
// Activity.java
final void attach(Context context, ActivityThread thread, ...) {
    // 创建 PhoneWindow
    mWindow = new PhoneWindow(this, windowStyle, parentWindow, activityConfigCallback);
    
    // 设置 WindowManager
    mWindowManager = appContext.getSystemService(Context.WINDOW_SERVICE);
    
    // 保存组件
    mToken = token;
    mThread = thread;
}
```

### onCreate 中 setContentView 流程

{% mermaid flowchart TB %}
SetContent["Activity.setContentView(layoutResID)"] --> WindowContent["PhoneWindow.setContentView(layoutResID)"]
WindowContent --> Decor["1. 创建 DecorView"]
WindowContent --> Inflate["2. 解析 layoutResID 生成 View 树"]
WindowContent --> Frame["3. 将 View 放入 ContentFrameLayout"]
{% endmermaid %}

---

## 6. 关键类作用

| 类 | 作用 |
|---|------|
| **ActivityTaskManagerService** | 系统服务，管理 Activity 任务 |
| **ActivityStarter** | 解析 Intent，决定启动模式 |
| **ActivityStack** | 管理 Activity 栈 |
| **ApplicationThread** | App 端Binder服务端 |
| **ActivityThread** | 主线程，消息循环 |
| **H** | Handler，切换到主线程执行 |
| **Instrumentation** | 监控 Activity 生命周期 |
| **ActivityClientRecord** | Activity 客户端记录 |

---

## 面试常问问题

### Q1: Activity 启动为什么要通过 Binder？

```
答案：
1. Activity 运行在 App 进程
2. AMS 运行在 System Server 进程
3. 不同进程间通信需要 Binder
4. 这是 Android 进程间通信的基础机制
```

### Q2: Activity 启动为什么慢？

```
答案：
1. 需要与 AMS 进程通信 (Binder)
2. 需要创建 Application、Context、Window
3. 需要解析 Layout、创建 View 树
4. 需要执行生命周期方法

优化：
- 使用 ViewBinding 替代 findViewById
- 延迟初始化
- 使用 Coil/Glide 延迟加载图片
```

### Q3: onCreate 和 onResume 区别？

```
onCreate: Activity 创建时调用，只调用一次
          - 初始化 UI
          - setContentView

onResume: Activity 可见并可交互时调用
          - 每次回到 Activity 都调用
          - 刷新 UI
          - 启动动画
```

---

## 总结

```
Activity 启动关键点:

1. startActivity() 
   ↓
2. Instrumentation.execStartActivity()
   ↓
3. Binder → ATMS.startActivity()
   ↓
4. ActivityStarter 解析启动模式
   ↓
5. ApplicationThread.scheduleTransaction()
   ↓
6. H Handler 处理 (切换到主线程)
   ↓
7. performLaunchActivity() 创建 Activity
   ↓
8. callActivityOnCreate() → onCreate()
   ↓
9. handleResumeActivity() → onResume()
```

---

**相关文章**：
- [Handler/Looper/MessageQueue 源码深度解析](./android-handler-looper-messagequeue.md)
- [Android View 与 Compose 互操作避坑指南](./android-view-compose-interop.md)
