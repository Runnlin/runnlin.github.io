---
title: Android Binder IPC 机制
tags: [Android, Binder, IPC, 驱动]
date: 2024-02-15
---

# Android Binder IPC 机制

Binder 是 Android 系统的核心 IPC 机制，通过共享内存和驱动层实现高效的进程间通信，避免了传统 Unix IPC（如 Socket、管道）的多次数据拷贝。

## 架构层次

| 层次 | 组件 |
|------|------|
| 应用层 | AIDL、Messenger |
| 框架层 | IBinder、Binder、BinderProxy |
| Native 层 | libbinder |
| 驱动层 | /dev/binder |

## 工作原理

Binder 的核心在于**一次拷贝**（One Copy）机制：

1. 发送方将数据写入内核空间（`copy_from_user`）
2. 内核将数据映射到接收方的用户空间（`mmap`）
3. 接收方直接读取，无需第二次拷贝

相比之下，Socket 和管道需要两次拷贝（发送方→内核→接收方）。

## AIDL 示例

```java
// IMyService.aidl
package com.example;

interface IMyService {
    String getData(int id);
    void postEvent(String event);
}
```

```java
// 服务端实现
public class MyService extends Service {
    private final IMyService.Stub mBinder = new IMyService.Stub() {
        @Override
        public String getData(int id) {
            return "data_" + id;
        }
        @Override
        public void postEvent(String event) {
            Log.d("MyService", "event: " + event);
        }
    };

    @Override
    public IBinder onBind(Intent intent) { return mBinder; }
}
```

```java
// 客户端调用
IMyService service = IMyService.Stub.asInterface(binder);
String result = service.getData(42);
```

## 关键特性

- **身份验证**：Binder 驱动保证 UID/PID 的真实性，防止欺骗（`Binder.getCallingUid()`）
- **对象传递**：支持 IBinder 对象跨进程传递，实现远程对象引用
- **线程池**：每个进程自动维护 Binder 线程池（默认最大 15 个）处理并发请求
- **死亡通知**：通过 `linkToDeath` 监听对端进程死亡事件
