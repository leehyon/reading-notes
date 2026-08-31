# Ch6. Networking

> 对应 PDF 物理页 205-245（印刷页 189-229）。本章 41 页，C 网络编程的入门：HTTP web client（`getaddrinfo` + `socket` + `connect` + `read/write` + `SO_RCVTIMEO`）、event-driven web server（`select` + `FD_SET` + `accept` + `bind` + `listen`，echo server）、HTTPS/OpenSSL TLS（`BIO` + `SSL_CTX` + 数字证书 + 对称/非对称加密）。**这一章直接对接机器人 ROS2 DDS、MoveIt web UI、Autoware cloud 通信**。

## §1 章节概述

1. **网络 4 层协议栈**（Kalin 表 6-1）：HTTP（应用层，web server 协议） / TCP（传输层，连接导向、可靠） / UDP（传输层，无连接、best-try） / IP（网络层，地址解析）。
2. **socket = 文件描述符的延伸**——socket 跟 file/pipe/FIFO **共享** `open/read/write/close` 模式；唯一区别是 socket 多 4 个参数版本 `send/recv` 接受 flags（`MSG_DONTWAIT` / `MSG_NOSIGNAL` / `MSG_PEEK` / `MSG_WAITALL`）。
3. **5 个核心 syscall**：`socket(domain, type, protocol)` 创建 / `bind(fd, addr, len)` 绑地址（server） / `listen(fd, backlog)` 监听（server） / `accept(fd, addr, len)` 接受（server） / `connect(fd, addr, len)` 主动连（client）。**server 3 步走 = socket + bind + listen**；**client 2 步走 = socket + connect**。
4. **`getaddrinfo` 是 DNS 解析的标准入口**——`AF_UNSPEC` 允许 IPv4/IPv6；`SOCK_STREAM` 选 TCP；返回**链表**（一个域名可能解析到多个 IP，循环试到通为止）。**配合 `freeaddrinfo` 释放**（不是 `free`）。
5. **`SO_RCVTIMEO` 是 socket 级别超时**——`setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &timeval, len)`。**`SO_SNDTIMEO` 是写超时**。**`connect` 超时**用 `SO_SNDTIMEO` 不太可靠——Linux 下**真用 `select`/`poll`**。
6. **`read`/`write` 在 socket 上是字节流**——`read` 不保证一次读满（ch5 原则）；`write` 在 PIPE_BUF 大数据时可能部分写。**Kalin ch6.2 用 `EWOULDBLOCK` 检测超时返回**——但 ch5 推荐用 `errno == EAGAIN`（两者在 Linux 同值，POSIX 不保证）。
7. **event-driven server 的核心 = `fd_set` + `select`**——`FD_ZERO` 清空 / `FD_SET(fd, &set)` 加入 / `FD_ISSET(fd, &set)` 检测 / `FD_CLR(fd, &set)` 移除。`select(nfds, &readfds, &writefds, &exceptfds, timeout)` 阻塞等事件。
8. **`select` 的限制**：`FD_SETSIZE` 默认 1024（系统限制）——C10K 问题。**`poll`** 类似但用链表（无固定上限）；**`epoll`** (Linux) / `kqueue` (BSD/macOS) 是 scalable 方案。
9. **webserver 架构 = 主循环 + accept + read/write 一次**——Kalin 实现的 echo server 是"单进程单线程"，100 backlog，每个 client 一个 request-response 后关闭。**生产级是 concurrent**（ch7 fork/pthread）。
10. **Makefile = 多文件编译的"脚本"**——`target: deps \n\t command` 格式（**必须是 TAB 缩进**）。`make` 读 `Makefile`（默认）。
11. **`curl` 是测试 web server 的标准工具**——`curl localhost:3000?msg=Hello` 模拟 GET；`curl --data "..."` 模拟 POST。`curl -v` 看 HTTP header 详细。
12. **OpenSSL = TLS + 密码学库**——`SSL_CTX` 全局上下文；`SSL` 单个连接；`BIO` = `FILE*` 的 SSL 版；`X509` = 数字证书格式。
13. **TLS 三服务**：① peer authentication（数字证书 + CA 签名 + 公私钥对） / ② confidentiality（加密 = 对称快 + 非对称解决密钥分发） / ③ reliability（消息摘要 = hash 校验）。
14. **TLS 握手流程**：客户端发 hello → 服务器回证书 → 客户端用 server 公钥加密 premaster secret → 双方推导 session key → 之后用对称加密。**HTTPS = HTTP over TLS，默认端口 443**（vs HTTP 80）。
15. **BIO API 模拟 FILE**——`BIO_new_ssl_connect` / `BIO_do_connect` / `BIO_do_handshake` / `BIO_puts` / `BIO_read` / `BIO_write` / `BIO_free` / `BIO_free_all`。**BIO 处理嵌套释放**（与 ch3.12 嵌套堆对应）。
16. **OpenSSL 编译链**：`gcc -lssl -lcrypto`——`libssl` 是 TLS 协议；`libcrypto` 是密码学原语（AES、RSA、SHA）。**macOS 自带 LibreSSL**；**Linux 通常需 `apt install libssl-dev`**。

## §2 核心 Takeaways

### T1 — socket 是文件描述符的延伸，`open/read/write/close` 模式共享
- **是什么**：`int fd = socket(AF_INET, SOCK_STREAM, 0);` 拿到 int 句柄；后续 `read` / `write` / `close` / `fcntl` / `select` 全部通用。**`dup`/`dup2`/`fdopen`/`fileno` 也通用**。
- **为什么重要**：ch5 的所有 I/O 知识**直接迁移**到 socket；ch3 内存模型（fd 泄漏、`O_CLOEXEC`）通用。
- **解决什么**：统一的 I/O 心智模型。
- **适用场景**：所有 socket 编程；**网络 + 文件 + pipe + FIFO = 一套 API**。

### T2 — `getaddrinfo` 必须 + `freeaddrinfo` 配对
- **是什么**：`int rc = getaddrinfo(host, port, &hints, &result);` 解析域名；`freeaddrinfo(result);` 释放。**`result` 是链表**（一个域名 → 多个 IP）。
- **为什么重要**：手写 `gethostbyname` 已 deprecated（**不是线程安全**）；`getaddrinfo` 是 POSIX 标准且 IPv4/IPv6 双栈。
- **解决什么**：跨平台 DNS 解析；**`AF_UNSPEC` = 同时尝试 IPv4/IPv6**。
- **适用场景**：所有需要 host → IP 转换的代码（client、server bind 时也可用 `getaddrinfo` 解析 bind 地址）。

### T3 — server `socket + bind + listen`、client `socket + connect`
- **是什么**：server 三步走用 listen；client 两步走用 connect。**`bind` 是"声明我占这个地址"**（server 必用；client 一般让 OS 选）。
- **为什么重要**：80% 的网络教程混淆 server/client 流程；背下来就懂了。
- **解决什么**：建立 TCP 双向通道。
- **适用场景**：所有 TCP server/client。

### T4 — `SO_REUSEADDR` 是 2026 年的硬规则
- **是什么**：`setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &val, sizeof(val))`（val=1）。**允许 bind 到 TIME_WAIT 的地址**——server 重启不需要等 60s。
- **为什么重要**：不设 SO_REUSEADDR 的 server 重启经常 `bind: address in use`。
- **解决什么**：快速重启、容器化（k8s pod 频繁重启）。
- **适用场景**：所有 production server。

### T5 — `select` + `fd_set` 是入门级 event-driven 模式
- **是什么**：`FD_ZERO` / `FD_SET` / `FD_ISSET` / `FD_CLR` 四宏 + `select(nfds, &readfds, &writefds, &exceptfds, timeout)` 阻塞等事件。`nfds` 是**最大 fd + 1**（不是总数）。
- **为什么重要**：理解 select 才能迁移到 `epoll`（Linux）/ `kqueue`（BSD）。
- **解决什么**：单线程服务多 client（C10K 起点）。
- **限制**：`FD_SETSIZE` 1024；`select` 每次调用都重建 fd_set（O(n)）。
- **替代**：`poll`（链表，无固定上限）；`epoll`（Linux 高效）；**`io_uring`**（Linux 5.1+，终极方案）。

### T6 — `accept` 产生"新 fd"用于 client I/O
- **是什么**：`int client_fd = accept(server_fd, &client_addr, &len);` 阻塞等 client 连。**返回的是新 fd**（与 server_fd 独立），用于 read/write。
- **为什么重要**：server_fd 只负责 accept；client_fd 负责数据传输。**关闭 client 不影响 server 持续 accept**。
- **解决什么**：长连接、并发服务。
- **适用场景**：所有 TCP server。**UDP 没有 accept**——`recvfrom` 直接收。

### T7 — `read` 字节流不是消息流；用固定长度/分隔符/分帧协议
- **是什么**：TCP 是字节流（不是消息）。**3 次 write 100B 可能在 1 次 read 收到 300B，也可能 3 次 read 各 100B**。
- **为什么重要**：HTTP 用 `\r\n\r\n` 分隔 header/body；**WebSocket 用 2-byte length prefix**；**gRPC 用 varint prefix**；**MQTT 用 fixed header**。**自定协议必加 framing**。
- **解决什么**：协议解析、消息边界。
- **适用场景**：所有 TCP 应用层协议。

### T8 — `send/recv` 4 参版本有 flags；常用 `MSG_NOSIGNAL` 防 SIGPIPE
- **是什么**：`send(fd, buf, len, flags)` 比 `write` 多 flags：`MSG_DONTWAIT`（单次非阻塞）/ `MSG_NOSIGNAL`（写已关闭 socket 不发 SIGPIPE）/ `MSG_PEEK`（看数据但不消费）/ `MSG_WAITALL`（等满 len 字节）。
- **为什么重要**：Linux 上 client 端 `write(sock)` 收到 RST 时**进程被 SIGPIPE 默认杀掉**——`MSG_NOSIGNAL` 必加。**或 `signal(SIGPIPE, SIG_IGN)`**。
- **解决什么**：优雅处理对端关闭。
- **适用场景**：所有 client；尤其 long-running service。

### T9 — OpenSSL `SSL_CTX` 是单连接 `SSL` 的工厂
- **是什么**：`SSL_CTX* ctx = SSL_CTX_new(method);` 创建全局；`SSL* ssl = SSL_new(ctx);` 单连接；`SSL_set_fd(ssl, fd);` 绑 socket；`SSL_connect(ssl)` / `SSL_accept(ssl)` 握手；`SSL_read`/`SSL_write` 加密读写；`SSL_free` / `SSL_CTX_free` 释放。
- **为什么重要**：多连接复用 1 个 ctx（节省证书加载）。
- **解决什么**：TLS 协议处理。
- **适用场景**：所有 HTTPS / TLS 服务器。
- **替代**：**mbedTLS**（嵌入式）/ **BoringSSL**（Google fork）/ **WolfSSL**（小资源）。

### T10 — X.509 证书 = 主体身份 + CA 签名 + 公钥 + 有效期
- **是什么**：证书包含 `Subject`（主体名）/ `Issuer`（CA）/ `Validity`（起止日期）/ `Subject Public Key Info`（公钥）/ `Signature`（CA 用自己私钥签的 hash）。**PEM = base64 包 + `-----BEGIN CERTIFICATE-----`**；**DER = 二进制**。
- **为什么重要**：理解证书才能配 TLS（`SSL_CTX_use_certificate_file` / `SSL_CTX_use_PrivateKey_file`）。
- **解决什么**：HTTPS 服务器身份验证。
- **适用场景**：web server、API gateway、IoT 设备身份。

### T11 — TLS 用 hybrid 加密：RSA 传 AES key + AES 加密数据
- **是什么**：TLS 握手用 RSA/EC 传对称密钥（premaster secret）；之后用 AES 加密数据。**这是 2026 年的标准**（TLS 1.3 移除了 RSA key exchange，强制 ECDHE）。
- **为什么重要**：理解为什么 TLS 慢（握手 1-2 RTT）；**TLS 1.3 = 1-RTT 握手**。
- **解决什么**：HTTPS 性能优化（TLS session resumption、TLS false start、0-RTT）。
- **适用场景**：web server、API server、IoT。

### T12 — `SO_RCVTIMEO` 是 read 超时；`SO_SNDTIMEO` 是 write 超时
- **是什么**：`setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &timeval, sizeof(timeval));`（`timeval` = {sec, usec}）。**设置后 read/write 超时返回 -1 + errno=EAGAIN/EWOULDBLOCK**。
- **为什么重要**：client 不希望永久阻塞在 read；server 也不希望 worker 线程被慢 client 占住。
- **解决什么**：超时控制。
- **适用场景**：所有需要"等不到就放弃"的场景。
- **限制**：`connect` 的 SO_SNDTIMEO 在 Linux 上**行为差异**（connect 内部分配临时端口，有锁）；**真需要 connect timeout 用 `select` + `O_NONBLOCK`**。

## §3 工程实践视角

### 3.1 Linux 系统开发视角

- **`O_NONBLOCK` 配 `epoll` 是 C10K/C100K 标配**——Nginx / Redis / Envoy / Linkerd 全部用 `epoll`（或 io_uring）。
- **`TCP_NODELAY` 关 Nagle**——`setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &val, 1)`；**小包交互场景必开**（SSH、TLS handshake、控制消息）。**大文件传输反而不开**（让 Nagle 聚包）。
- **`SO_KEEPALIVE` 是死连接检测**——默认 2 小时；**生产 server 应设更短**（`TCP_KEEPIDLE`/`TCP_KEEPINTVL`/`TCP_KEEPCNT`）。**很多 client NAT 超时 → 死 socket 累积**。
- **`SO_LINGER` 控制 close 行为**——`l_onoff=1, l_linger=0` = close 立即返回、发 RST 给对端（**不是 FIN**）；`l_linger>0` = 阻塞 close 到数据发完或超时。
- **`shutdown(fd, SHUT_WR)` = TCP 半关闭**——发完 FIN 后还能 read。**HTTP/1.0 用半关闭示意 body 结束**。
- **`getpeername` / `getsockname` 查对端/本端 IP:port**——server 日志必备。
- **`accept4` (Linux 特有) = accept + O_CLOEXEC + O_NONBLOCK 一步**——避免 race condition（accept 后没设 O_CLOEXEC 就 fork/exec 泄漏 fd）。
- **`SO_REUSEPORT` (Linux 3.9+) 多进程 listen 同一端口**——内核负载均衡。Nginx worker 多进程用这个。
- **TLS 1.3 vs TLS 1.2 性能**：TLS 1.3 1-RTT 握手、移除不安全 cipher（RC4、MD5、SHA-1）。**OpenSSL 1.1.1+ 默认 TLS 1.3**。
- **OpenSSL 弃用警告**：**OpenSSL 3.0 改 API**——`SSLv23_method` 改 `TLS_client_method` / `TLS_server_method`。**AI 写的代码经常是 OpenSSL 1.0 旧 API**。
- **BoringSSL / LibreSSL 替代**：BoringSSL = Google OpenSSL fork（Chromium 用），更小更安全。LibreSSL = OpenBSD OpenSSL fork（macOS 用）。
- **`gnutls` vs `OpenSSL`**：gnutls 早期被 OpenSSL 许可争议（OpenSSL 旧 license = SSLeay，商用问题）推动；2026 年 OpenSSL 已 Apache 2.0，差距不大。
- **`strace` 调试网络**：`strace -e trace=network -p PID` 追踪所有 socket 调用；`strace -e accept4,read,write,sendto,recvfrom ./prog`。
- **`ss` 命令查 socket**：`ss -tlnp` 查 listen；`ss -tnp` 查 established；`ss -s` 汇总。**`netstat` 已 deprecated**。
- **`tcpdump` 抓包**：`tcpdump -i any -A port 80` 看 HTTP 明文；`tcpdump -i any -w out.pcap port 443` 抓 TLS 握手；`wireshark` 解析 pcap。

### 3.2 机器人软件视角（ROS2 / 嵌入式控制）

- **ROS2 DDS 用 UDP multicast**——Fast DDS / Cyclone DDS 默认 UDP。**TCP 可选**（reliable mode）。**与 ch6 socket 模型一致**——`socket(AF_INET, SOCK_DGRAM, 0)` for UDP。
- **ROS2 discovery = multicast 239.255.0.1:7400**——`iperf3` 可测带宽，`tcpdump` 可看 QoS。
- **ROS2 client library 的 shm transport**——Cyclone DDS Iceoryx 用 `/dev/shm` 共享内存（ch7 详解），**比 UDP 跨进程快 10×**。
- **MoveIt web UI** = ROS2 → rosbridge_server → WebSocket（端口 9090）→ 浏览器 WebGL。**WebSocket = TCP 上的 framing 协议**（ch6.7 风格的"分帧"）。
- **Autoware.auto / Autoware Universe cloud 通信** = MQTT over TLS（port 8883）。MQTT 是 IBM 的 IoT 协议，**固定 2 字节 header + 变长 payload**（TCP 字节流上的"分帧"）。
- **ros2_control hardware interface 实时性** = `SO_RCVTIMEO` 配 `select` 检查 actuator feedback；超时则进入 safety mode。
- **机器人 /dev/ttyUSB0 串口通信**——`open("/dev/ttyUSB0", O_RDWR \| O_NOCTTY)` 配 `termios` 设置 baud rate。**与 socket 同 fd 模型**。
- **ROS2 跨主机通信** = `ROS_DOMAIN_ID` 环境变量配 DDS discovery 范围；**实际是 multicast group 选择**。
- **MoveIt Servo 实时控制** = `std::chrono::steady_clock` 1ms 周期 + `O_NONBLOCK` socket 收发；**用 `io_uring` 进一步降延迟**（2026 年新趋势）。
- **ROS2 调试工具**：`ros2 topic list` / `ros2 node list` / `ros2 bag record` / `ros2 service call` —— 全是 DDS over UDP/TCP。

### 3.3 初级 vs 高级工程师对照

| 习惯 | 初级 | 高级 |
|---|---|---|
| 地址解析 | `inet_aton` 写死 IP | `getaddrinfo` + `AF_UNSPEC` |
| listen 前 | `bind(sock, ...)` 然后 `listen` | 必设 `SO_REUSEADDR`；`accept4` 一次性 O_CLOEXEC |
| 阻塞 read | 永久等 | `SO_RCVTIMEO` + `select` 双保险 |
| client write 收到 RST | 被 SIGPIPE 杀掉 | `MSG_NOSIGNAL` 或 `signal(SIGPIPE, SIG_IGN)` |
| 半关闭 | 调 `close(sock)` | `shutdown(sock, SHUT_WR)` 半关闭 |
| event-driven | `select` 直接用 | 评估 `epoll` / `kqueue` / `io_uring` |
| TLS 配置 | 默认 `SSLv23_method` | 显式 `TLS_client_method` (OpenSSL 3) + cipher 列表 |
| 证书验证 | 调 `verify_dc` stub 返回 1 | 真验证 + CRL/OCSP |
| Nagle 算法 | 不管 | 小包交互 `TCP_NODELAY`；文件传输不动 |
| 死连接 | 2 小时 keepalive | `TCP_KEEPIDLE=60` + `TCP_KEEPINTVL=10` + `TCP_KEEPCNT=3` |
| 多 client | 1 client 1 thread | 1 thread + `epoll` 服务 N client |

## §4 AI 时代视角

### 4.1 这些知识还重要吗？（2026 年视角）

**极重要。** 网络是现代软件必备——LLM 生成的 C 网络代码 70% 错在**错误处理不完整**（不查 `socket` 返回值、不处理 `SIGPIPE`、不设 `SO_REUSEADDR`、不处理 partial read/write）。**AI 写 HTTPS client 经常漏证书验证**（生产安全漏洞）。所有 vllm、sglang、llama.cpp 的 HTTP server、Caddy/Nginx 替代品（Varnish、Haproxy）、API gateway（Envoy）都基于 ch6 套路。

现代工程师的 ch6 日常：
- **AI 生成的 socket 代码 review** = 查 flags 选型 + 错误处理 + 超时
- **AI 生成的 TLS 代码 review** = 查证书验证 + cipher 列表 + OpenSSL 版本兼容
- **AI 生成的 web server** = 评估 select/poll/epoll 选型

### 4.2 AI 现在能做的

- ✅ 写 HTTP client + server 完整模板
- ✅ 解释 `getaddrinfo` / `socket` / `bind` / `listen` / `accept` / `connect`
- ✅ 写 `select` + `FD_SET` echo server
- ✅ 解释 OpenSSL `BIO` API 与 `FILE*` 关系
- ✅ 解释数字证书结构与 TLS 握手

### 4.3 AI 经常写错的地方（必看）

| 错误模式 | 例子 | 后果 |
|---|---|---|
| **1. 漏 `SO_REUSEADDR`** | server 启动 `bind` 失败 | 重启要等 60s TIME_WAIT；**必加** |
| **2. 漏 `SIGPIPE` 处理** | `write(sock);` 收到 RST | 进程被 SIGPIPE 杀；用 `MSG_NOSIGNAL` 或 `signal(SIGPIPE, SIG_IGN)` |
| **3. `read` 不循环** | `read(sock, buf, n); process(buf);` | 只读到部分字节（ch5 原则）；**循环读** |
| **4. `select` 后 read 不在循环** | 以为 select 之后 read 必满 | **select 只通知"有数据"，不保证读满** |
| **5. 漏 `close(client_fd)`** | accept 后处理完忘了关 | fd 泄漏；**必 close + FD_CLR** |
| **6. `select` 不用 temp_set** | `select(&active_set)` 直接传 active | select 会修改 fd_set 破坏原集；**先 copy 到 temp_set** |
| **7. 漏 `freeaddrinfo(result)`** | getaddrinfo 后没 free | 内存泄漏（ASan 抓） |
| **8. 死等 connect** | `connect(sock);` 不设超时 | 防火墙挂包 → 永久挂起；**用 SO_SNDTIMEO 或 select** |
| **9. `bind` port < 1024 不 root** | 普通用户 bind 80 | 权限错；**用 > 1024 或 setcap** |
| **10. 漏 `-lssl -lcrypto`** | `gcc foo.c -o foo` 不链 | 链接错 `undefined reference to SSL_*` |
| **11. OpenSSL 1.0 旧 API** | `SSLv23_method()` | OpenSSL 3.0 编译警告 + 未来删除；**用 `TLS_client_method()` / `TLS_server_method()`** |
| **12. 证书验证 stub** | `verify_dc() { return 1; }` | **生产安全漏洞**：不验证证书 = 中间人攻击 |
| **13. `connect` 返回 EINPROGRESS** | 非阻塞 connect 立刻当失败 | 实际是"进行中"；**用 select 检查写就绪** |
| **14. 假设 socket write 一次写完** | `write(sock, big_buf, 1MB);` | PIPE_BUF 大时部分写；**循环写** |
| **15. `shutdown(sock, SHUT_RDWR)` vs `close`** | 不区分 | shutdown 不释放 fd；**close 才真释放** |
| **16. 漏 `O_CLOEXEC` for accept fd** | 多进程 fork 后 fd 还在 | 泄漏；**用 `accept4`** |
| **17. AI 写 GET 用 HTTP/2** | 单 `write(sock, "GET / HTTP/2")` | HTTP/2 是 binary framing；**HTTP/1.1 文本，HTTP/2 完全不同的 API** |
| **18. 假设 `gethostbyname` 是线程安全** | 多线程用同一 host | 静态缓冲 race；**用 `getaddrinfo`** |

### 4.4 工程师必须保留的核心能力

- **能在 5 秒内画出 TCP 三次握手 / 四次挥手时序图**。
- **能 30 秒内写出"循环 read + SO_RCVTIMEO + MSG_NOSIGNAL"模板**。
- **能用 `tcpdump` / `wireshark` 抓包**——debug 任何网络问题。
- **能用 `strace -e network` 跟 syscall**——`write`、`read` 在 socket 上的实际参数。
- **能用 `ss` 查 socket 状态**——`TIME_WAIT` 累积、`CLOSE_WAIT` 泄漏。
- **能跟 AI 说"用 `getaddrinfo` + `SO_REUSEADDR` + `MSG_NOSIGNAL` + `SO_RCVTIMEO` + OpenSSL 3.x 的 `TLS_client_method()`"**——ch6 词汇是 prompt 的关键。

### 4.5 wasm 工具链延伸（用户已选需要）

- **wasm 没有真网络 socket**——`socket`/`connect`/`bind` 走 emscripten 的 `wasm_socket` 后端（proxied 通过 JS）。**仅支持 WS/WSS**（W3C WebSocket API）——无 raw TCP/UDP。
- **wasm HTTPS 必须用浏览器 fetch API**——不能用 OpenSSL。**`fetch()` 在 JS 里 = C 写 wasm 调 JS 桥**。
- **AI 工具链中 wasm 的 ch6 角色**：Claude Code / Codex 沙箱跑用户 C 网络代码 = **emcc 把 socket 调用转成 WS 桥到沙箱外的代理**。AI 写 `socket(AF_INET, ...)` 在 wasm 里会失败——必须用 emscripten 的 `EMSCRIPTEN_WEBSOCKET_*` API。

## §5 实践行动项

### A1 — 编译并跑 Kalin 的 webclient（ch6.2 实战）
```bash
mkdir -p /tmp/modern-c/ch06 && cd /tmp/modern-c/ch06
cat > web_client.c <<'EOF'
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#define BuffSize 2048
extern int get_connection(const char*, const char*);
int main(void) {
    const char* host = "example.com";   /* 改成 example.com, 稳定 */
    const char* port = "80";
    const char* request = "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n";
    int sock = get_connection(host, port);
    if (sock < 0) {
        perror("connect");
        return 1;
    }
    if (write(sock, request, strlen(request)) < 0) {
        perror("write");
        return 1;
    }
    char buf[BuffSize];
    ssize_t n, total = 0;
    while ((n = read(sock, buf, sizeof(buf))) > 0) {
        total += n;
        fwrite(buf, 1, n, stdout);
    }
    fprintf(stderr, "\n%zd bytes read\n", total);
    close(sock);
    return 0;
}
EOF
cat > get_connection.c <<'EOF'
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <fcntl.h>
#include <sys/time.h>
int get_connection(const char* host, const char* port) {
    struct addrinfo hints = {0}, *result, *next;
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    int rc = getaddrinfo(host, port, &hints, &result);
    if (rc != 0) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(rc));
        return -1;
    }
    int sock = -1;
    for (next = result; next; next = next->ai_next) {
        sock = socket(next->ai_family, next->ai_socktype, next->ai_protocol);
        if (sock < 0) continue;
        if (connect(sock, next->ai_addr, next->ai_addrlen) != -1) break;
        close(sock); sock = -1;
    }
    freeaddrinfo(result);
    if (sock < 0) return -1;
    /* 2 秒超时 */
    struct timeval tv = {2, 0};
    setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
    return sock;
}
EOF
```make
# Makefile (recipe 行用真实 TAB 缩进, 复制后请把 "	$(CC)" 前的空格换回 TAB)
CFLAGS=-std=c11 -Wall -Wextra -O2
webclient: web_client.c get_connection.c
	$(CC) $(CFLAGS) -o $@ $^ -lc
```
```bash
make
./webclient | head -5
```
**验收**：看到 HTTP 响应（HTML 头 + 部分 body）；`xxx bytes read` 报告。

### A2 — echo server（ch6.3 实战 + curl 测试）
```bash
cat > echo_server.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#define PORT 3000
#define BACKLOG 100
int main(void) {
    int srv = socket(AF_INET, SOCK_STREAM, 0);
    if (srv < 0) {
        perror("socket");
        return 1;
    }
    /* 关键: SO_REUSEADDR 防 TIME_WAIT 阻塞 */
    int yes = 1;
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(PORT);
    if (bind(srv, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("bind");
        return 1;
    }
    if (listen(srv, BACKLOG) < 0) {
        perror("listen");
        return 1;
    }
    fprintf(stderr, "echo server on :%d\n", PORT);
    while (1) {
        struct sockaddr_in cli; socklen_t len = sizeof(cli);
        int client = accept(srv, (struct sockaddr*)&cli, &len);
        if (client < 0) continue;
        char ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &cli.sin_addr, ip, sizeof(ip));
        fprintf(stderr, "client %s:%d\n", ip, ntohs(cli.sin_port));
        char buf[1024];
        ssize_t n;
        /* 关键: 循环 read */
        while ((n = read(client, buf, sizeof(buf))) > 0) {
            /* 关键: 循环 write 防 partial */
            ssize_t off = 0;
            while (off < n) {
                ssize_t w = write(client, buf+off, n-off);
                if (w <= 0) break;
                off += w;
            }
        }
        close(client);
    }
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -O2 -o echo_server echo_server.c
./echo_server &
SPID=$!
sleep 1
curl -s -d "Hello, world" localhost:3000
echo "---"
kill $SPID
```
**验收**：`curl --data "Hello, world" localhost:3000` 返回 `Hello, world`；server 端 log 显示 client IP:port。

### A3 — `select` 实现多 client echo server
```bash
cat > select_server.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/select.h>
#define PORT 3001
#define BACKLOG 100
int main(void) {
    int srv = socket(AF_INET, SOCK_STREAM, 0);
    int yes = 1;
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    bind(srv, (struct sockaddr*)&addr, sizeof(addr));
    listen(srv, BACKLOG);
    fprintf(stderr, "select echo server on :%d\n", PORT);
    fd_set active, temp;
    FD_ZERO(&active);
    FD_SET(srv, &active);
    int maxfd = srv;
    while (1) {
        temp = active;  /* 关键: 每次 select 之前 copy */
        if (select(maxfd+1, &temp, NULL, NULL, NULL) < 0) {
        perror("select");
        return 1;
    }
        for (int i = 0; i <= maxfd; i++) {
            if (!FD_ISSET(i, &temp)) continue;
            if (i == srv) {  /* accept new client */
                int client = accept(srv, NULL, NULL);
                if (client >= 0) {
                    FD_SET(client, &active);
                    if (client > maxfd) maxfd = client;
                    fprintf(stderr, "new client fd=%d\n", client);
                }
            } else {  /* echo existing client */
                char buf[256];
                ssize_t n = read(i, buf, sizeof(buf));
                if (n <= 0) {  /* EOF or error */
                    close(i); FD_CLR(i, &active);
                    fprintf(stderr, "close fd=%d\n", i);
                } else {
                    write(i, buf, n);
                }
            }
        }
    }
}
EOF
gcc -std=c11 -Wall -Wextra -O2 -o select_server select_server.c
./select_server &
SPID=$!
sleep 1
# 起 2 个 client 测并发
(printf "AAA\n"; sleep 2) | nc localhost 3001 &
(printf "BBB\n"; sleep 2) | nc localhost 3001 &
sleep 1
echo "---kill---"
kill $SPID
wait
```
**验收**：两个 client 同时连；server 端 log 显示新 client 到来；`AAA` 和 `BBB` 各自被 echo 回。

### A4 — `MSG_NOSIGNAL` + `SIGPIPE` 处理（ch6 实战）
```bash
cat > nosigpipe.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <errno.h>
int main(void) {
    /* 防 SIGPIPE 默认杀进程 */
    signal(SIGPIPE, SIG_IGN);
    /* 启动 echo server (假设 A2 在跑) */
    int srv = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(3000);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
    if (connect(srv, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("connect"); return 1;
    }
    /* 写一行, server 会回 */
    const char* msg = "Hello\n";
    ssize_t w = send(srv, msg, strlen(msg), MSG_NOSIGNAL);  /* 关键: NOSIGNAL */
    printf("send returned %zd\n", w);
    /* 收 echo */
    char buf[64];
    ssize_t r = read(srv, buf, sizeof(buf));
    printf("recv: %.*s", (int)r, buf);
    close(srv);  /* server 收到 FIN 后 read 返回 0; 之后 write 收到 EPIPE */
    /* 再 write 一次 - 期望 send 返回 -1 + EPIPE, 不被杀 */
    w = send(srv, "after close\n", 12, MSG_NOSIGNAL);
    if (w < 0) printf("EPIPE (expected): errno=%d (%s)\n", errno, strerror(errno));
    return 0;
}
EOF
# 需先启动 A2 的 echo_server
./echo_server &
SPID=$!
sleep 1
gcc -std=c11 -Wall -Wextra -O2 -o nosigpipe nosigpipe.c
./nosigpipe
kill $SPID
```
**验收**：
- `send returned 6` (写成功)
- `recv: Hello\n`
- `EPIPE (expected): errno=32 (Broken pipe)` ——**不退出 139 (SIGPIPE)**，说明 MSG_NOSIGNAL 起作用

### A5 — OpenSSL HTTPS client（ch6.4 实战，需装 libssl）
```bash
which openssl || sudo apt install -y libssl-dev
cat > ssl_client.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
#include <openssl/bio.h>
#include <openssl/ssl.h>
#include <openssl/x509.h>
#include <openssl/err.h>
int main(void) {
    OpenSSL_add_all_algorithms();
    (void)ERR_load_BIO_strings();  /* OpenSSL 3.0 deprecated, 静默掉 warning */
    (void)SSL_load_error_strings();
    /* SSL_library_init() 在 OpenSSL 3.0 已自动 (OpenSSL_add_all_algorithms 隐含) */
    const SSL_METHOD* method = TLS_client_method();   /* OpenSSL 3.x */
    SSL_CTX* ctx = SSL_CTX_new(method);
    /* 关键: 启用证书验证 (AI 经常漏) */
    SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL);
    SSL_CTX_set_default_verify_paths(ctx);
    BIO* web = BIO_new_ssl_connect(ctx);
    BIO_set_conn_hostname(web, "www.example.com:443");
    SSL* ssl = NULL; BIO_get_ssl(web, &ssl);
    if (BIO_do_connect(web) <= 0) {
        fprintf(stderr, "connect fail\n");
        return 1;
    }
    if (BIO_do_handshake(web) <= 0) {
        fprintf(stderr, "handshake fail\n");
        return 1;
    }
    long vr = SSL_get_verify_result(ssl);
    printf("verify result: %ld (%s)\n", vr, vr == X509_V_OK ? "OK" : "FAIL");
    BIO_puts(web, "GET / HTTP/1.1\r\nHost: www.example.com\r\nConnection: close\r\n\r\n");
    char buf[1024];
    int n;
    while ((n = BIO_read(web, buf, sizeof(buf))) > 0) fwrite(buf, 1, n, stdout);
    BIO_free_all(web); SSL_CTX_free(ctx);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -O2 -o ssl_client ssl_client.c -lssl -lcrypto && ./ssl_client | head -3
```
**验收**：看到 `verify result: 0 (OK)`（X509_V_OK=0 表示证书有效）；输出 HTTP 响应头。如果 `verify result: ... (FAIL)` 则需要 `apt install ca-certificates`。

## §6 值得深入思考的问题

1. **C10K 问题怎么解？** `select` → `poll` → `epoll` (Linux) / `kqueue` (BSD) / `io_uring` (Linux 5.1+) / io_uring + XDP。**为什么 Nginx worker 数 = CPU 核数？** 多 worker 怎么协同？**`SO_REUSEPORT` 怎么让内核做负载均衡**？
2. **为什么 `accept` 是 race condition 源？** 多线程 accept 同一 listen fd；**Linux 3.9+ `SO_REUSEPORT` + accept 自旋** vs **每个线程独立 listen fd**。**现代 server 是单 accept + epoll 分发**（Nginx）。
3. **TLS 1.3 vs TLS 1.2 差什么？** TLS 1.3 1-RTT 握手、移除 CBC 模式、强制 ECDHE、0-RTT 数据（**有 replay 风险**）。**为什么 Cloudflare 2024 默认全 TLS 1.3**？
4. **OpenSSL 3.0 改了 API 怎么应对？** `SSLv23_method` 弃用 → `TLS_client_method` / `TLS_server_method`；`ENGINE_*` 改为 `provider`。**AI 写的 OpenSSL 代码 70% 是旧 API**——怎么 review 出来？
5. **QUIC (HTTP/3) 是不是 socket 终结者？** QUIC = UDP 上的 TLS 1.3 + 多路复用 + 0-RTT。**Chrome、Cloudflare、YouTube 默认 HTTP/3**。**Linux 6.7+ 有内核级 QUIC**。**C 怎么写 QUIC client？** `nghttp3` / `quiche` (Cloudflare) / `msquic` (Microsoft)。
6. **AI 写 HTTPS 代码 80% 漏证书验证**——你的 CI 怎么抓？clang-tidy 没这条规则，**要自己写 linter**：grep `SSL_CTX_set_verify` / `SSL_get_verify_result` 出现次数；或 **librevulture** 静态分析。
7. **wasm 的网络限制**怎么 workaround？emscripten 提供 `EMSCRIPTEN_WEBSOCKET_*` 模拟 TCP（over WS）。**但 UDP 怎么办？** WebRTC datachannel。**HTTP/3 (QUIC) 在 wasm 里基本没戏**。

---

*下一章预告*：**ch7 Concurrency and Parallelism**——59 页重量章。`fork`/`exec`/`wait`/`waitpid` 多进程 + 僵尸进程处理 + 共享内存 IPC + 文件锁 + 消息队列 + `pthread` 多线程 + race condition (Miser/Spendthrift 案例) + deadlock 演示 + **SIMD 并行**（`#pragma omp simd` / intrinsics）。**这一章是 ROS2 DDS、MoveIt 实时控制、CUDA host code 的"前置基础"**。
