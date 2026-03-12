---
title: "Gradle 构建速度优化实战 30%"
date: 2022-02-24 10:36:31
description: "从 10 分钟到 7 分钟的真实优化经验"
comments: true
toc: true
categories:
  - Android
  - Build
tags:
  - Android
  - Gradle
  - Build
  - Performance
---

# Gradle 构建速度优化实战 30%

> 从 10 分钟到 7 分钟的真实优化经验

## 目录

1. [诊断工具与基线测量](#1-诊断工具与基线测量)
2. [Gradle 配置优化](#2-gradle-配置优化)
3. [依赖优化](#3-依赖优化)
4. [增量构建优化](#4-增量构建优化)
5. [并行与缓存优化](#5-并行与缓存优化)
6. [实际效果对比](#6-实际效果对比)

---

## 1. 诊断工具与基线测量

### 构建扫描 (Build Scan)

```bash
# 启用构建扫描
./gradlew build --scan

# 或在 gradle.properties 中
org.gradle.scan=true
```

### 使用 buildEnvironment 查看依赖树

```bash
# 查看项目依赖
./gradlew dependencies

# 查看特定配置依赖
./gradlew app:dependencies --configuration releaseRuntimeClasspath

# 树形输出
./gradlew app:dependencies --configuration releaseRuntimeClasspath | tree
```

### Gradle Profiler

```bash
# 安装 Gradle Profiler
./gradlew wrapper --gradle-version=8.5

# 生成性能报告
gradle-profiler --benchmark
```

---

## 2. Gradle 配置优化

### gradle.properties 关键配置

```properties
# gradle.properties

# JVM 内存配置 - 根据机器配置调整
org.gradle.jvmargs=-Xmx4096m -Dfile.encoding=UTF-8 -XX:+HeapDumpOnOutOfMemoryError

# 启用并行构建
org.gradle.parallel=true

# 启用构建缓存
org.gradle.caching=true

# 配置缓存 (Gradle 8+)
org.gradle.configuration-cache=true
org.gradle.configuration-cache.problems=warn

# 守护进程
org.gradle.daemon=true

# 按需配置
org.gradle.configureondemand=true

# 开启增量编译
kotlin.incremental=true
kotlin.incremental.useClasspathSnapshot=true

# 并行执行测试
org.gradle.workers.max=4

# 关闭不必要的任务
android.enableR8.fullMode=false
android.nonTransitiveRClass=true
```

### 建议的 JVM 内存配置

| 机器内存 | JVM 堆内存 | 配置 |
|---------|-----------|------|
| 8GB | 2-4GB | -Xmx2048m |
| 16GB | 4-8GB | -Xmx4096m |
| 32GB+ | 8-12GB | -Xmx8192m |

---

## 3. 依赖优化

### 使用 API vs Implementation

```kotlin
// build.gradle (app)

// ❌ 错误: implementation 暴露依赖给其他模块
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
}

// ✅ 正确: api 暴露，implementation 隐藏
// 如果是 app 模块的 final consumers，使用 implementation
dependencies {
    // 内部模块使用 api
    api("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    
    // 不需要暴露的使用 implementation
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
}
```

### 依赖版本管理

```kotlin
// build.gradle (project level)
ext {
    kotlin_version = '1.9.22'
    compose_compiler_version = '1.5.8'
    room_version = '2.6.1'
    hilt_version = '2.50'
}

// 使用 version catalog (推荐)
```

### 使用版本目录 (Version Catalog)

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "1.9.22"
coroutines = "1.7.3"
room = "2.6.1"
hilt = "2.50"
compose = "1.5.8"

[libraries]
kotlin-coroutines = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }

[plugins]
android-application = { id = "com.android.application", version.ref = "androidGradlePlugin" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }

[bundles]
room = ["room-runtime", "room-ktx"]
```

---

## 4. 增量构建优化

### 避免不必要的任务

```kotlin
tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile>().configureEach {
    kotlinOptions {
        // 增量编译
        incremental = true
        // 并行编译
        freeCompilerArgs += listOf("-Xsingle-module")
    }
}

// 禁用不必要的任务
tasks.matching {
    it.name.contains("Test") || it.name.contains("test")
}.configureEach {
    enabled = false
}
```

### 使用 kotlin daemon

```properties
# gradle.properties
kotlin.daemon.jvmargs=-Xmx1536m
kotlin.daemon.enabled=true
```

---

## 5. 并行与缓存优化

### 本地缓存配置

```properties
# gradle.properties

# 缓存目录 (可选自定义)
# org.gradle.cachingDirectory=/path/to/cache

# 缓存任务输出
org.gradle.caching=true

# Gradle 8+ 配置缓存
org.gradle.configuration-cache=true
```

### 清理缓存

```bash
# 清理 Gradle 缓存
./gradlew clean

# 清理 build 目录
rm -rf app/build

# 清理 Gradle 缓存
rm -rf ~/.gradle/caches

# 清理 Kotlin 编译器缓存
rm -rf .gradle
rm -rf build
```

---

## 6. 实际效果对比

### 优化前后对比

| 优化项 | 优化前 | 优化后 | 提升 |
|-------|-------|-------|------|
| 首次编译 | 10:30 | 7:15 | 31% |
| 增量编译 | 3:45 | 1:20 | 64% |
| Clean build | 10:30 | 6:50 | 35% |

### 最终配置

```properties
# gradle.properties - 最终版本
org.gradle.jvmargs=-Xmx4096m -Dfile.encoding=UTF-8 \
    -XX:+HeapDumpOnOutOfMemoryError \
    -XX:+UseParallelGC

org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
org.gradle.daemon=true
org.gradle.configureondemand=true
org.gradle.workers.max=4

kotlin.incremental=true
kotlin.daemon.jvmargs=-Xmx1536m
kotlin.daemon.enabled=true

android.useAndroidX=true
android.nonTransitiveRClass=true
android.enableR8.fullMode=false
```

---

## 常见问题排查

### 问题 1: Kotlin daemon 启动慢

```properties
# 解决: 增加 daemon 内存
kotlin.daemon.jvmargs=-Xmx2048m
```

### 问题 2: 配置阶段太慢

```kotlin
// 解决: 使用 configureOnDemand
// 在 gradle.properties 中
org.gradle.configureondemand=true
```

### 问题 3: 依赖解析慢

```kotlin
// 解决: 添加阿里云镜像 (仅限国内)
// settings.gradle
pluginManagement {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public/' }
        maven { url 'https://maven.aliyun.com/repository/google/' }
        maven { url 'https://maven.aliyun.com/repository/central/' }
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public/' }
        maven { url 'https://maven.aliyun.com/repository/google/' }
        maven { url 'https://maven.aliyun.com/repository/central/' }
    }
}
```

---

## 快速检查清单

| 检查项 | 命令 |
|--------|------|
| 开启并行 | `org.gradle.parallel=true` |
| 开启缓存 | `org.gradle.caching=true` |
| 开启配置缓存 | `org.gradle.configuration-cache=true` |
| JVM 内存足够 | `org.gradle.jvmargs=-Xmx4096m` |
| Kotlin daemon | `kotlin.daemon.enabled=true` |
| 增量编译 | `kotlin.incremental=true` |

---

## 总结

Gradle 构建优化核心：
1. **内存**: 分配足够 JVM 内存 (4GB+)
2. **并行**: 开启并行构建和并行执行
3. **缓存**: 启用 Gradle 和 Kotlin 缓存
4. **配置**: 使用配置缓存 (Gradle 8+)
5. **依赖**: 优化依赖结构，使用版本目录

---

**相关文章**：
- [Jetpack Compose 性能优化实战](./jetpack-compose-performance-optimization.md)
- [Android 内存泄漏分析与解决](./android-memory-leak-analysis.md)
