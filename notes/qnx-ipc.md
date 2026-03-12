---
title: QNX IPC 进程间通信
tags: [QNX, IPC, 系统编程]
date: 2024-03-01
---

# QNX IPC 进程间通信

QNX Neutrino RTOS 使用**消息传递**（Message Passing）作为主要的 IPC 机制。所有进程通过内核进行同步消息传递，这是 QNX 微内核设计的核心。

## 基本概念

- **MsgSend** — 客户端发送消息并阻塞，等待服务端回复
- **MsgReceive** — 服务端接收消息并阻塞，等待客户端发来请求
- **MsgReply** — 服务端回复消息，解除客户端阻塞

所有 I/O 操作也是通过消息传递实现的，驱动程序本质上就是消息服务器。

## 代码示例

```c
// 服务端
int chid = ChannelCreate(0);

struct _msg { int type; char data[256]; } msg;
struct _reply { int status; char result[256]; } reply;

int rcvid = MsgReceive(chid, &msg, sizeof(msg), NULL);
// 处理消息...
MsgReply(rcvid, EOK, &reply, sizeof(reply));

// 客户端
int coid = ConnectAttach(0, server_pid, chid, _NTO_SIDE_CHANNEL, 0);
int err = MsgSend(coid, &msg, sizeof(msg), &reply, sizeof(reply));
```

## Pulse 信号

Pulse 是轻量级、异步的通知机制，用于不需要回复的场景：

```c
// 发送 Pulse（不阻塞）
MsgSendPulse(coid, priority, _PULSE_CODE_MINAVAIL, value);

// 服务端接收（MsgReceive 同时处理消息和 Pulse）
struct _pulse pulse;
int rcvid = MsgReceive(chid, &pulse, sizeof(pulse), NULL);
if (rcvid == 0) {
    // 这是一个 Pulse，不需要 MsgReply
}
```

## 注意事项

1. 消息传递是**同步阻塞**的，客户端等待服务端回复
2. 通道（Channel）由服务端创建，连接（Connection）由客户端建立
3. 优先级继承：服务端处理消息时会继承客户端的优先级，防止优先级反转
4. `ConnectAttach` 的 `nd` 参数为 0 时表示本地节点
