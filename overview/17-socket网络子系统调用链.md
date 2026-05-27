# socket/connect/bind 网络子系统调用链

> 基于 glibc 2.43.9000，分析网络 socket API 的实现架构  
> 分析工具：源码阅读 + clangd 定义跳转

---

## 概述

glibc 的网络 socket API 是系统调用的**薄封装层**，核心价值在于：

1. **取消点支持**：阻塞 socket 操作作为 POSIX 线程取消点（SYSCALL_CANCEL）
2. **双路径兼容**：独立 syscall（现代内核）vs `socketcall` 多路复用（旧内核）
3. **时间戳转换**：32位/64位时间格式透明转换（`setsockopt`/`getsockopt`/`recvmmsg`）
4. **地址族抽象**：`__SOCKADDR_ARG` 联合体支持多种地址族

---

## 1. 系统调用分类

### 1.1 非阻塞 API（INLINE_SYSCALL_CALL，非取消点）

| API | syscall | 源文件 |
|-----|---------|--------|
| `socket()` | `socket` | `sysdeps/unix/sysv/linux/socket.c` |
| `bind()` | `bind` | `sysdeps/unix/sysv/linux/bind.c` |
| `listen()` | `listen` | `sysdeps/unix/sysv/linux/listen.c` |
| `shutdown()` | `shutdown` | `sysdeps/unix/sysv/linux/shutdown.c` |
| `getsockname()` | `getsockname` | `sysdeps/unix/sysv/linux/getsockname.c` |
| `getpeername()` | `getpeername` | `sysdeps/unix/sysv/linux/getpeername.c` |
| `socketpair()` | `socketpair` | `sysdeps/unix/sysv/linux/socketpair.c` |
| `getsockopt()` | `getsockopt` | `sysdeps/unix/sysv/linux/getsockopt.c` |
| `setsockopt()` | `setsockopt` | `sysdeps/unix/sysv/linux/setsockopt.c` |

### 1.2 阻塞 API（SYSCALL_CANCEL，取消点）

| API | syscall | 源文件 |
|-----|---------|--------|
| `connect()` | `connect` | `sysdeps/unix/sysv/linux/connect.c` |
| `accept()` | `accept` / `accept4` | `sysdeps/unix/sysv/linux/accept.c` |
| `accept4()` | `accept4` | `sysdeps/unix/sysv/linux/accept4.c` |
| `send()` | `send` / `sendto` | `sysdeps/unix/sysv/linux/send.c` |
| `recv()` | `recv` / `recvfrom` | `sysdeps/unix/sysv/linux/recv.c` |
| `sendto()` | `sendto` | `sysdeps/unix/sysv/linux/sendto.c` |
| `recvfrom()` | `recvfrom` | `sysdeps/unix/sysv/linux/recvfrom.c` |
| `sendmsg()` | `sendmsg` | `sysdeps/unix/sysv/linux/sendmsg.c` |
| `recvmsg()` | `recvmsg` | `sysdeps/unix/sysv/linux/recvmsg.c` |
| `sendmmsg()` | `sendmmsg` | `sysdeps/unix/sysv/linux/sendmmsg.c` |
| `recvmmsg()` | `recvmmsg_time64` | `sysdeps/unix/sysv/linux/recvmmsg.c` |

---

## 2. 核心调用链

### 2.1 socket() — 创建套接字

```
socket(domain, type, protocol)
│
└── __socket(fd, type, domain)  [sysdeps/unix/sysv/linux/socket.c:22]
    │
    ├── [现代内核] __ASSUME_SOCKET_SYSCALL
    │   └── INLINE_SYSCALL_CALL(socket, fd, type, domain)
    │       └── syscall(__NR_socket, ...)
    │
    └── [旧内核回退]
        └── SOCKETCALL(socket, fd, type, domain)
            └── INLINE_SYSCALL(socketcall, 2, SOCKOP_socket, args[])
                └── syscall(__NR_socketcall, 1, args)
```

**设计说明**：
- `type` 参数可包含 `SOCK_CLOEXEC | SOCK_NONBLOCK` 标志（Linux 2.6.27+）
- 不是取消点（创建操作不阻塞）

### 2.2 bind() — 绑定地址

```
bind(fd, addr, len)
│
└── __bind(fd, addr.__sockaddr__, len)  [sysdeps/unix/sysv/linux/bind.c:22]
    │
    ├── [现代内核] INLINE_SYSCALL_CALL(bind, fd, addr, len)
    └── [旧内核] SOCKETCALL(bind, fd, addr, len, 0, 0, 0)
```

**设计说明**：
- `__CONST_SOCKADDR_ARG` 是 `const struct sockaddr *` 的类型安全联合体
- 不是取消点

### 2.3 listen() — 监听连接

```
listen(fd, backlog)
│
└── __listen(fd, backlog)  [sysdeps/unix/sysv/linux/listen.c]
    │
    ├── [现代内核] INLINE_SYSCALL_CALL(listen, fd, backlog)
    └── [旧内核] SOCKETCALL(listen, fd, backlog)
```

### 2.4 connect() — 建立连接（取消点）

```
connect(fd, addr, len)
│
└── __libc_connect(fd, addr.__sockaddr__, len)  [sysdeps/unix/sysv/linux/connect.c:22]
    │
    ├── [现代内核] __ASSUME_CONNECT_SYSCALL
    │   └── SYSCALL_CANCEL(connect, fd, addr, len)
    │       └── __syscall_cancel(fd, addr, len, 0, 0, 0, __NR_connect)
    │           ├── [未设置取消标志] → 直接 syscall
    │           └── [已设置取消标志] → 取消点检查 + syscall
    │
    └── [旧内核]
        └── SOCKETCALL_CANCEL(connect, fd, addr, len)
            └── SYSCALL_CANCEL(socketcall, SOCKOP_connect, args[])
```

**取消点机制**（详见 signal/07 和 pthread/12）：

```
__syscall_cancel(arg1..arg6, nr)    [sysdeps/unix/sysv/linux/syscall_cancel.c]
│
├── 检查 pd->cancelhandling 的 CANCELED_BITMASK
│   └── [已取消 + CANCELTYPE_BITMASK=async] → __syscall_do_cancel()
│
├── 标记 cancelpc 区间开始
├── syscall(nr, arg1..arg6)          // 实际系统调用
├── 标记 cancelpc 区间结束
│
└── [信号中断返回时]
    └── sigcancel_handler 检查 PC 是否在 cancelpc 区间
        └── [在区间内 + CANCELED] → 执行取消
```

### 2.5 accept() / accept4() — 接受连接（取消点）

```
accept(fd, addr, len)
│
└── __libc_accept(fd, addr.__sockaddr__, len)  [sysdeps/unix/sysv/linux/accept.c]
    │
    ├── [有 accept syscall] SYSCALL_CANCEL(accept, ...)
    ├── [有 accept4] SYSCALL_CANCEL(accept4, fd, addr, len, 0)
    └── [旧内核] SOCKETCALL_CANCEL(accept, ...)

accept4(fd, addr, len, flags)
│
└── SYSCALL_CANCEL(accept4, fd, addr, len, flags)
    // flags: SOCK_CLOEXEC | SOCK_NONBLOCK
```

### 2.6 send() / sendto() — 发送数据（取消点）

```
send(fd, buf, len, flags)
│
└── __libc_send(fd, buf, len, flags)  [sysdeps/unix/sysv/linux/send.c]
    │
    ├── [有 send syscall] SYSCALL_CANCEL(send, fd, buf, len, flags)
    ├── [无 send 有 sendto] SYSCALL_CANCEL(sendto, fd, buf, len, flags, NULL, 0)
    └── [旧内核] SOCKETCALL_CANCEL(send, fd, buf, len, flags)

sendto(fd, buf, len, flags, addr, addrlen)
│
└── __libc_sendto(...)  [sysdeps/unix/sysv/linux/sendto.c]
    └── SYSCALL_CANCEL(sendto, fd, buf, len, flags, addr, addrlen)

sendmsg(fd, msg, flags)
│
└── __libc_sendmsg(fd, msg, flags)  [sysdeps/unix/sysv/linux/sendmsg.c]
    └── SYSCALL_CANCEL(sendmsg, fd, msg, flags)

sendmmsg(fd, vmessages, vlen, flags)
│
└── __sendmmsg(...)  [sysdeps/unix/sysv/linux/sendmmsg.c]
    └── SYSCALL_CANCEL(sendmmsg, fd, vmessages, vlen, flags)
```

### 2.7 recv() / recvfrom() — 接收数据（取消点）

```
recv(fd, buf, len, flags)
│
└── __libc_recv(fd, buf, len, flags)  [sysdeps/unix/sysv/linux/recv.c]
    │
    ├── [有 recv syscall] SYSCALL_CANCEL(recv, fd, buf, len, flags)
    ├── [无 recv 有 recvfrom] SYSCALL_CANCEL(recvfrom, fd, buf, len, flags, NULL, NULL)
    └── [旧内核] SOCKETCALL_CANCEL(recv, fd, buf, len, flags)

recvfrom(fd, buf, len, flags, addr, addrlen)
│
└── __libc_recvfrom(...)  [sysdeps/unix/sysv/linux/recvfrom.c]
    └── SYSCALL_CANCEL(recvfrom, fd, buf, len, flags, addr, addrlen)

recvmsg(fd, msg, flags)
│
└── __libc_recvmsg(fd, msg, flags)  [sysdeps/unix/sysv/linux/recvmsg.c]
    └── SYSCALL_CANCEL(recvmsg, fd, msg, flags)
```

### 2.8 recvmmsg() — 批量接收（取消点 + 时间转换）

```
recvmmsg(fd, vmessages, vlen, flags, timeout)
│
└── __recvmmsg64(fd, vmessages, vlen, flags, timeout)
    │
    ├── [__TIMESIZE != 64] 保存各消息的 msg_controllen（用于时间戳转换）
    │
    ├── recvmmsg_syscall(fd, vmessages, vlen, flags, timeout)
    │   ├── SYSCALL_CANCEL(recvmmsg_time64, ...)       // 64位时间 syscall
    │   └── [ENOSYS 回退]
    │       ├── valid_timespec64_to_timespec(timeout)   // 转为 32位
    │       ├── SYSCALL_CANCEL(recvmmsg, ...)           // 32位时间 syscall
    │       └── 回写剩余 timeout
    │
    └── [__TIMESIZE != 64]
        └── __convert_scm_timestamps()                  // 转换 cmsg 中的时间戳
```

---

## 3. 双路径架构：独立 syscall vs socketcall 多路复用

### 3.1 历史背景

Linux 最初只有一个 `socketcall(2)` 系统调用，通过 `call` 参数区分操作：

```c
// 旧方式：所有 socket 操作走同一个 syscall number
syscall(__NR_socketcall, SOCKOP_socket,  args[]);  // call=1
syscall(__NR_socketcall, SOCKOP_bind,    args[]);  // call=2
syscall(__NR_socketcall, SOCKOP_connect, args[]);  // call=3
// ... 共 20+ 种操作
```

现代内核（x86_64、AArch64）为每个操作提供独立 syscall number：
```c
syscall(__NR_socket,  domain, type, protocol);
syscall(__NR_bind,    fd, addr, len);
syscall(__NR_connect, fd, addr, len);
```

### 3.2 编译时选择

```c
// sysdeps/unix/sysv/linux/socket.c
int __socket(int fd, int type, int domain)
{
#ifdef __ASSUME_SOCKET_SYSCALL      // 64位架构始终定义
  return INLINE_SYSCALL_CALL(socket, fd, type, domain);
#else                                // 32位旧内核可能未定义
  return SOCKETCALL(socket, fd, type, domain);
#endif
}
```

### 3.3 SOCKETCALL 宏展开

```
SOCKETCALL(socket, fd, type, domain)
→ __SOCKETCALL3(SOCKOP_socket, fd, type, domain)
→ INLINE_SYSCALL(socketcall, 2, SOCKOP_socket,
     ((long int [3]){(long int)fd, (long int)type, (long int)domain}))
→ syscall(__NR_socketcall, 1, args_array)
```

### 3.4 SOCKETCALL_CANCEL（阻塞操作）

```
SOCKETCALL_CANCEL(connect, fd, addr, len)
→ __SOCKETCALL_CANCEL3(SOCKOP_connect, fd, addr, len)
→ SYSCALL_CANCEL(socketcall, SOCKOP_connect,
     ((long int [3]){fd, addr, len}))
→ __syscall_cancel(..., __NR_socketcall)  // 取消点包装
```

---

## 4. setsockopt / getsockopt 时间戳兼容层

### 4.1 问题

旧内核（`!__ASSUME_TIME64_SYSCALLS`）不支持 64位时间的 socket 选项。

### 4.2 setsockopt 调用链

```
__setsockopt(fd, level, optname, optval, len)
│
├── setsockopt_syscall(fd, level, optname, optval, len)
│   ├── [现代] INLINE_SYSCALL_CALL(setsockopt, ...)
│   └── [旧内核] SOCKETCALL(setsockopt, ...)
│
└── [r == -1 && errno == ENOPROTOOPT]  // 64位时间选项不支持
    └── setsockopt32(fd, level, optname, optval, len)
        ├── SO_RCVTIMEO_NEW → SO_RCVTIMEO_OLD
        │   └── timeval64 → timeval32 转换后重试
        ├── SO_SNDTIMEO_NEW → SO_SNDTIMEO_OLD
        │   └── timeval64 → timeval32 转换后重试
        ├── SO_TIMESTAMP_NEW → SO_TIMESTAMP_OLD
        │   └── 直接重试（int 类型无需转换）
        └── SO_TIMESTAMPNS_NEW → SO_TIMESTAMPNS_OLD
            └── 直接重试
```

### 4.3 getsockopt 调用链

```
__getsockopt(fd, level, optname, optval, len)
│
├── getsockopt_syscall(fd, level, optname, optval, len)
│
└── [r == -1 && errno == ENOPROTOOPT]
    └── getsockopt32(fd, level, optname, optval, len)
        ├── SO_RCVTIMEO/SNDTIMEO:
        │   ├── getsockopt_syscall(..., OLD_optname, &tv32)
        │   └── timeval32 → timeval64 转换回写
        └── SO_TIMESTAMP/TIMESTAMPNS:
            └── 直接用旧选项号重试
```

---

## 5. SYSCALL_CANCEL 取消点机制

### 5.1 宏展开路径

```
SYSCALL_CANCEL(connect, fd, addr, len)
→ __SYSCALL_CANCEL_CALL(connect, fd, addr, len)
→ __SYSCALL_CANCEL_DISP(__SYSCALL_CANCEL, connect, fd, addr, len)
→ __SYSCALL_CANCEL3(connect, fd, addr, len)
→ __syscall_cancel(__SSC(fd), __SSC(addr), __SSC(len), 0, 0, 0, __NR_connect)
```

### 5.2 __syscall_cancel 执行流

```
__syscall_cancel(arg1, arg2, arg3, arg4, arg5, arg6, nr)
│
├── cancelhandling = pd->cancelhandling (atomic load)
├── [CANCELED_BITMASK 已设 + CANCELTYPE=async]
│   └── __syscall_do_cancel()  → 不返回
│
├── [标记: __syscall_cancel_arch_start]
│   └── 进入不可中断 syscall 前的"取消安全区"
│
├── raw_syscall(nr, arg1..arg6)        // 实际内核调用
│
├── [标记: __syscall_cancel_arch_end]
│   └── 离开取消安全区
│
└── return result
```

### 5.3 与 SIGCANCEL 的交互

当线程处于 `connect()`/`recv()` 等阻塞 syscall 中时：
1. 另一线程调用 `pthread_cancel()`
2. 向目标线程发送 `SIGCANCEL` 信号
3. 内核中断阻塞 syscall，返回 `-EINTR`
4. `sigcancel_handler` 检查 PC 是否在 `__syscall_cancel_arch_start` 和 `_end` 之间
5. 如果在区间内且 `CANCELED` 标志已设 → 执行取消（`__syscall_do_cancel()`）

---

## 6. 地址结构抽象

### 6.1 __SOCKADDR_ARG 联合体

```c
// socket/sys/socket.h
typedef union {
  struct sockaddr *__sockaddr__;
  struct sockaddr_at *__sockaddr_at__;
  struct sockaddr_ax25 *__sockaddr_ax25__;
  struct sockaddr_dl *__sockaddr_dl__;
  struct sockaddr_eon *__sockaddr_eon__;
  struct sockaddr_in *__sockaddr_in__;
  struct sockaddr_in6 *__sockaddr_in6__;
  struct sockaddr_inarp *__sockaddr_inarp__;
  struct sockaddr_ipx *__sockaddr_ipx__;
  struct sockaddr_iso *__sockaddr_iso__;
  struct sockaddr_ns *__sockaddr_ns__;
  struct sockaddr_un *__sockaddr_un__;
  struct sockaddr_x25 *__sockaddr_x25__;
} __SOCKADDR_ARG __attribute__ ((__transparent_union__));
```

- 用户可传入任何地址结构指针，编译器自动通过联合体转换
- 实际传给 syscall 的始终是 `addr.__sockaddr__`（通用指针）

---

## 7. 典型使用场景的完整调用链

### 7.1 TCP 服务器

```
1. socket(AF_INET, SOCK_STREAM, 0)
   → INLINE_SYSCALL(socket, 3, 2, 1, 0)
   → fd

2. bind(fd, &addr, sizeof(addr))
   → INLINE_SYSCALL(bind, 3, fd, &addr, 16)

3. listen(fd, backlog)
   → INLINE_SYSCALL(listen, 2, fd, backlog)

4. accept(fd, &client_addr, &len)           // ← 取消点
   → __syscall_cancel(fd, &client_addr, &len, 0, 0, 0, __NR_accept4)
   → 阻塞直到连接到达
   → client_fd

5. recv(client_fd, buf, size, 0)            // ← 取消点
   → __syscall_cancel(client_fd, buf, size, 0, 0, 0, __NR_recvfrom)

6. send(client_fd, response, len, 0)        // ← 取消点
   → __syscall_cancel(client_fd, response, len, 0, 0, 0, __NR_sendto)

7. shutdown(client_fd, SHUT_RDWR)
   → INLINE_SYSCALL(shutdown, 2, client_fd, 2)
```

### 7.2 UDP 客户端

```
1. socket(AF_INET, SOCK_DGRAM, 0)
   → INLINE_SYSCALL(socket, 3, 2, 2, 0)

2. sendto(fd, data, len, 0, &server, sizeof(server))    // ← 取消点
   → __syscall_cancel(fd, data, len, 0, &server, sizeof(server), __NR_sendto)

3. recvfrom(fd, buf, size, 0, &from, &fromlen)          // ← 取消点
   → __syscall_cancel(fd, buf, size, 0, &from, &fromlen, __NR_recvfrom)
```

---

## 8. 辅助工具函数

### 8.1 地址转换

| 函数 | 位置 | 功能 |
|------|------|------|
| `inet_pton()` | `resolv/inet_pton.c` | 文本→二进制地址 (IPv4/IPv6) |
| `inet_ntop()` | `resolv/inet_ntop.c` | 二进制→文本地址 |
| `inet_addr()` | `inet/inet_addr.c` | IPv4 文本→in_addr_t（过时） |
| `inet_ntoa()` | `inet/inet_ntoa.c` | in_addr→文本（过时，非线程安全） |

### 8.2 字节序

| 函数/宏 | 实现 | 功能 |
|---------|------|------|
| `htons()` / `ntohs()` | 内联 `__bswap_16` | 16位主机→网络序 |
| `htonl()` / `ntohl()` | 内联 `__bswap_32` | 32位主机→网络序 |

---

## 9. 源文件布局

```
socket/                              — 通用 stub 和头文件
├── socket.c                         — stub: __socket
├── bind.c                           — stub: __bind
├── connect.c                        — stub: __connect
├── listen.c                         — stub: __listen
├── accept4.c                        — stub: accept4
├── send.c / recv.c                  — stub
├── sendto.c / recvfrom.c            — stub
├── sendmsg.c / recvmsg.c            — stub
├── sendmmsg.c / recvmmsg.c          — stub
├── getsockopt.c / setsockopt.c      — stub
├── shutdown.c                       — stub
├── socketpair.c                     — stub
├── getsockname.c / getpeername.c    — stub
└── sys/socket.h                     — API 声明 + __SOCKADDR_ARG

sysdeps/unix/sysv/linux/             — Linux 实际实现
├── socket.c                         — INLINE_SYSCALL_CALL
├── bind.c                           — INLINE_SYSCALL_CALL
├── connect.c                        — SYSCALL_CANCEL
├── listen.c                         — INLINE_SYSCALL_CALL
├── accept.c                         — SYSCALL_CANCEL
├── accept4.c                        — SYSCALL_CANCEL
├── send.c / recv.c                  — SYSCALL_CANCEL
├── sendto.c / recvfrom.c            — SYSCALL_CANCEL
├── sendmsg.c / recvmsg.c           — SYSCALL_CANCEL
├── sendmmsg.c / recvmmsg.c         — SYSCALL_CANCEL + 时间转换
├── getsockopt.c / setsockopt.c     — INLINE_SYSCALL + 时间回退
├── shutdown.c                       — INLINE_SYSCALL_CALL
├── socketpair.c                     — INLINE_SYSCALL_CALL
└── socketcall.h                     — SOCKETCALL/SOCKETCALL_CANCEL 宏

sysdeps/unix/sysdep.h                — SYSCALL_CANCEL 核心定义
├── __syscall_cancel()               — 取消点检查 + syscall 执行
├── __SYSCALL_CANCELn 宏族           — 参数分派
└── __SYSCALL_CANCEL_CALL            — 入口宏

inet/                                — 地址操作
├── inet_addr.c                      — inet_addr (过时)
├── inet_ntoa.c                      — inet_ntoa (过时)
└── inet_net.c                       — 网络地址计算

resolv/                              — 名称解析
├── inet_pton.c                      — inet_pton (主入口)
├── inet_pton_length.c               — inet_pton4/inet_pton6 实现
└── inet_ntop.c                      — inet_ntop
```

---

## 10. 设计总结

| 设计决策 | 原因 |
|----------|------|
| 薄封装而非厚抽象 | socket API 语义由内核定义，glibc 无需额外逻辑 |
| SYSCALL_CANCEL vs INLINE_SYSCALL | POSIX 要求阻塞操作为取消点，非阻塞操作不是 |
| 双路径 socketcall 兼容 | 支持 Linux 2.x~3.x 旧内核（32位 x86 等） |
| 时间转换层 | Y2038 兼容：在不支持 64位时间的旧内核上自动降级 |
| `__SOCKADDR_ARG` 联合体 | 类型安全：避免强制转换错误，编译时检查 |
| accept → accept4(flags=0) | 部分架构无独立 accept syscall，用 accept4 模拟 |
| send → sendto(NULL,0) | 部分架构无独立 send syscall，用 sendto 模拟 |
