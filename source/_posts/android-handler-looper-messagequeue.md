---
title: "Handler/Looper/MessageQueue 源码深度解析"
date: 2022-06-14 18:42:08
description: "Android 主线程消息循环机制完全指南"
comments: true
toc: true
categories:
  - Android
  - Framework
tags:
  - Android
  - Handler
  - Looper
  - MessageQueue
---

# Handler/Looper/MessageQueue 源码深度解析

> Android 主线程消息循环机制完全指南

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [Looper 初始化与消息循环](#2-looper-初始化与消息循环)
3. [MessageQueue 消息队列原理](#3-messagequeue-消息队列原理)
4. [Message 消息机制](#4-message-消息机制)
5. [Handler 消息发送与处理](#5-handler-消息发送与处理)
6. [常见面试题解析](#6-常见面试题解析)

---

## 1. 整体架构概览

### 四大核心组件

{% mermaid flowchart LR %}
Handler["Handler<br/>发送和处理消息"] --> Queue["MessageQueue<br/>消息队列（单链表实现）"]
Looper["Looper<br/>不断从 MessageQueue 取消息"] --> Queue
Handler --> Message["Message<br/>消息载体"]
Looper --> Message
{% endmermaid %}

### 主线程启动流程

```java
// ActivityThread.main()
public static void main(String[] args) {
    // 1. 创建主线程 Looper
    Looper.prepareMainLooper();
    
    // 2. 创建 ActivityThread
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    
    // 3. 启动消息循环 - 阻塞方法
    Looper.loop();
    
    // 4. 消息循环退出后
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

---

## 2. Looper 初始化与消息循环

### Looper.prepareMainLooper()

```java
// Looper.java
public static void prepareMainLooper() {
    // 只能调用一次
    if (sMainLooper != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    
    // 创建主线程 Looper，并保存引用
    sMainLooper = myLooper();
}

// 实际调用
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper per thread");
    }
    // 存储 Looper 到 ThreadLocal
    sThreadLocal.set(new Looper(quitAllowed));
}
```

### Looper.loop() - 核心方法

```java
public static void loop() {
    // 获取当前线程的 Looper
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException(
            "No Looper; Looper.prepare() wasn't called on this thread.");
    }
    
    // 获取消息队列
    final MessageQueue queue = me.mQueue;
    
    // 主线程标识
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    
    // 死循环 - 这是核心！
    for (;;) {
        // 获取下一条消息，可能阻塞
        Message msg = queue.next();
        
        // 如果没有消息，loop() 会阻塞在这里
        if (msg == null) {
            // 返回 null 表示消息队列已退出
            return;
        }
        
        // ... 日志处理 ...
        
        // 分发消息到 Handler
        msg.target.dispatchMessage(msg);
        
        // 回收消息
        msg.recycleUnchecked();
    }
}
```

### 关键点：为什么不会 ANR？

{% mermaid flowchart TB %}
ANR["ANR 条件<br/>主线程阻塞 > 5 秒"] -.对比.-> Loop
Loop["Looper.loop()"] --> Step1["1. queue.next()<br/>获取消息（可阻塞）"]
Step1 --> Step2["2. msg.target.dispatchMessage()<br/>处理消息（通常很快）"]
Step2 --> Step3["3. 回到步骤 1"]
Step3 --> Step1
Step2 --> Note["关键：单条消息处理时间短<br/>耗时任务应放子线程"]
{% endmermaid %}

---

## 3. MessageQueue 消息队列原理

### MessageQueue 初始化

```java
// MessageQueue.java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    // 创建底层使用的 native 队列
    mPtr = nativeInit();
}
```

### next() - 获取下一条消息

```java
Message next() {
    for (;;) {
        if (nextPollTimeoutMs != 0) {
            Binder.flushPendingCommands();
        }
        
        // Native 方法，可能阻塞
        nativePollOnce(mPtr, nextPollTimeoutMs);
        
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            
            // 处理同步屏障（优先执行异步消息）
            if (msg != null && msg.target == null) {
                // 找到异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            
            if (msg != null) {
                // 消息还没到执行时间
                if (now < msg.when) {
                    nextPollTimeoutMs = (int) Math.min(
                        msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取到消息
                    mMessages = msg.next;
                    msg.next = null;
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有消息，阻塞
                nextPollTimeoutMs = -1;
            }
            
            // 处理退出
            if (mQuitting) {
                dispose();
                return null;
            }
        }
    }
}
```

### 同步屏障 (Sync Barrier)

```java
// 发送同步屏障
public int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
        
        // target = null 表示这是同步屏障
        msg.target = null;
        
        // 插入到队列
        // ...
    }
}

// 移除同步屏障
public void removeSyncBarrier(int token) {
    synchronized (this) {
        // 找到并移除
    }
}
```

**同步屏障的作用**：优先执行异步消息，用于 View 的 invalidate 等紧急绘制操作。

---

## 4. Message 消息机制

### Message 对象池

```java
// Message.java
public final class Message implements Parcelable {
    // 对象池
    private static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;
    private static final int MAX_POOL_SIZE = 50;
    
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0;
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
    
    void recycleUnchecked() {
        // 清理标志
        flags = 0;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        workSourceUid = 0;
        when = 0;
        target = null;
        callback = Runnable;
        data = null;
        
        // 回收到对象池
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
}
```

### Message 结构（单链表）

```java
public final class Message implements Parcelable {
    public int what;           // 消息标识
    public int arg1;          // 参数1
    public int arg2;          // 参数2
    public Object obj;        // 携带对象
    public long when;        // 执行时间
    public Handler target;    // 目标 Handler
    public Runnable callback; // 回调
    
    // 链表结构
    Message next;
    
    // 异步标志
    static final int FLAG_IN_USE = 1 << 0;
    static final int FLAG_ASYNCHRONOUS = 1 << 1;
    int flags;
}
```

{% mermaid flowchart LR %}
Head["mMessages"] --> N1["[when:100]"]
N1 --> N2["[when:200]"]
N2 --> N3["[when:300]"]
N3 --> Null["null"]
N1 --> M1["msg1<br/>(target)"]
N2 --> M2["msg2<br/>(target)"]
N3 --> M3["msg3<br/>(target)"]
{% endmermaid %}

---

## 5. Handler 消息发送与处理

### Handler 构造

```java
public Handler(Callback callback, boolean async) {
    // 检查 Looper 是否存在
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called " +
            "Looper.prepare()");
    }
    
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

### 发送消息

```java
public final boolean sendMessage(Message msg) {
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    
    // 将当前 Handler 设为消息的 target
    msg.target = this;
    
    // 插入消息队列
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    
    // 实际插入队列
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

### dispatchMessage - 消息分发

```java
public void dispatchMessage(Message msg) {
    // 1. 如果有 callback (Runnable)，优先执行
    if (msg.callback != null) {
        msg.callback.run();
        return;
    }
    
    // 2. 如果有 Handler.Callback，使用它
    if (mCallback != null) {
        if (mCallback.handleMessage(msg)) {
            return;
        }
    }
    
    // 3. 最后调用 handleMessage
    handleMessage(msg);
}
```

---

## 6. 常见面试题解析

### Q1: Handler 发送的消息如何保证按时间顺序执行？

```kotlin
// 答案：MessageQueue 使用 when (执行时间) 排序插入
// enqueueMessage 实现：

boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    
    synchronized (this) {
        if (msg.isInUse()) {
            throw new IllegalStateException();
        }
        
        // 标记正在使用
        msg.markInUse();
        msg.when = when;
        msg.next = null;
        
        // 按时间顺序插入链表
        Message p = mMessages;
        boolean needWake;
        
        if (p == null || when < p.when) {
            // 插入队首，需要唤醒
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // 找到合适位置插入
            Message prev;
            do {
                prev = p;
                p = p.next;
            } while (p != null && when >= p.when);
            
            msg.next = p;
            prev.next = msg;
            needWake = false;
        }
        
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

### Q2: 为什么主线程不会因为 loop() 死循环卡死？

```
答案：
1. loop() 不是 busy-wait，它会调用 nativePollOnce() 阻塞
2. 阻塞时线程不消耗 CPU
3. 当有消息到达时，Native 层会唤醒线程
4. 处理每条消息通常很快（<1ms），不会阻塞
```

### Q3: Handler 如何切换线程？

```kotlin
// 答案：Handler 绑定到创建时的 Looper
// 子线程创建 Handler：

class WorkerThread : Thread() {
    lateinit var handler: Handler
    
    override fun run() {
        Looper.prepare()  // 创建 Looper
        handler = Handler(Looper.myLooper()!!) {
            // 这个 callback 在子线程执行
            true
        }
        Looper.loop()  // 启动消息循环
    }
}

// 主线程发送消息到子线程 Handler
workerThreadHandler.post {
    // 这段代码在子线程执行
}
```

### Q4: MessageQueue 的 idleHandler 什么时候执行？

```java
// 当消息队列为空或只有延迟消息时执行
public void addIdleHandler(IdleHandler handler) {
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}

// 使用场景：Activity onCreate 后执行延迟初始化
Looper.myQueue().addIdleHandler {
    // 主线程空闲时执行
    performInitialization()
    false // false = 执行一次后移除
}
```

---

## 总结

```
Handler 机制核心流程：

1. 主线程创建 Looper.prepareMainLooper()
2. Looper 创建 MessageQueue
3. Looper.loop() 开启消息循环
4. Handler sendMessage() 将消息加入队列
5. MessageQueue.next() 返回待处理消息
6. Handler.dispatchMessage() 分发消息
7. 回调 handleMessage() 处理
8. 回收 Message 到对象池
9. 回到步骤 4
```

---

**相关文章**：
- [从零理解 Activity 启动流程](./android-activity-startup.md)
- [Android 内存泄漏分析与解决](./android-memory-leak-analysis.md)
