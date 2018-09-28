---
title: 多线程
comments: true
toc: false
date: 2018-09-19 08:47:18
categories: C/C++
tags: C/C++
---

C++11 新标准中引入了五个头文件来支持多线程编程，它们分别是 `<atomic>`, `<thread>`, `<mutex>`, `<condition_variable>` 和 `<future>`。

<!--more-->

- `<thread>`：该头文件主要声明了 std::thread 类，另外 `std::this_thread` 命名空间也在该头文件中。

- `<mutex>`：该头文件主要声明了与互斥量(Mutex)相关的类，包括 `std::mutex_*` 一系列类，`std::lock_guard`, `std::unique_lock`, 以及其他的类型和函数。

- `<condition_variable>`：该头文件主要声明了与条件变量相关的类，包括 `std::condition_variable` 和 `std::condition_variable_any`。

- `<future>`：该头文件主要声明了 `std::promise`, `std::package_task` 两个 Provider 类，以及 `std::future` 和 `std::shared_future` 两个 Future 类，另外还有一些与之相关的类型和函数，`std::async()` 函数就声明在此头文件中。

    * `<future>` 头文件中包含了以下几个类和函数：

        - Providers 类：`std::promise`, `std::package_task`

        - Futures 类：`std::future`, `std::shared_future`

        - Providers 函数：`std::async()`

        - 其他类型：`std::future_error`, `std::future_errc`, `std::future_status`, `std::launch`

- `<atomic>`：该头文主要声明了两个类, `std::atomic` 和 `std::atomic_flag`，另外还声明了一套 C 风格的原子类型和与 C 兼容的原子操作的函数。