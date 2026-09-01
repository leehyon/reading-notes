# Ch5. Input and Output

> 对应 PDF 物理页 167-204（印刷页 151-188）。本章 38 页，C I/O 的双层 API：底层 syscall `open/read/write/close`（字节级，文件描述符 0/1/2）vs 高层 `fopen/fread/fwrite/fprintf/fscanf`（多字节类型，FILE* 流）。**还覆盖重定向、`lseek` 随机访问、非阻塞 I/O（`O_NONBLOCK`+`fcntl`）、命名管道 FIFO**。**ch6 网络 socket 与本章底层 I/O 是同一套 API 范式**。

## §1 章节概述

1. **C I/O 双层 API**——底层（system-level）= `read/write` + 文件描述符（int，0/1/2 预留给 stdin/stdout/stderr）；高层（stream-based）= `fread/fwrite` + `FILE*` 流（`stdin/stdout/stderr`）。**高层是底层的 wrapper**，带缓冲。
2. **`open` syscall 签名**：`int open(const char* path, int flags, mode_t mode);` 返回 fd（≥ 0）或 -1；`flags` 是 `O_RDONLY`/`O_WRONLY`/`O_RDWR`/`O_CREAT`/`O_APPEND`/`O_NONBLOCK` 等位 OR；`mode` 是权限位 `S_IRUSR | S_IWUSR | S_IXUSR | S_IRGRP | S_IWGRP | S_IXGRP | S_IROTH | S_IWOTH | S_IXOTH`。
3. **`read/write` 返回值语义**：
   - `read(fd, buf, n)`：成功返回读取字节数；返回 0 = EOF（流结束）；返回 -1 = 错误（`errno`）
   - `write(fd, buf, n)`：成功返回写入字节数；返回 -1 = 错误。**注意**：`write` 可能在 n > PIPE_BUF 时**部分写**（需循环写）
4. **三个预定义 fd**：`0 = stdin`、`1 = stdout`、`2 = stderr`，对应宏 `STDIN_FILENO` / `STDOUT_FILENO` / `STDERR_FILENO`（来自 `<unistd.h>`）。高层对应 `stdin` / `stdout` / `stderr`（来自 `<stdio.h>`，是 `FILE*`）。
5. **重定向是 shell 机制，不是 C 机制**——`./prog < infile > outfile 2> logfile` 由 shell 把 fd 0/1/2 重指向对应文件，**C 代码完全无感**。`2>&1` 把 stderr 也重定向到 stdout。
6. **`lseek` 随机访问**：`off_t lseek(int fd, off_t offset, int whence);` `whence` = `SEEK_SET`/`SEEK_CUR`/`SEEK_END`。**只对 regular file 有意义**——pipe/socket/terminal 返回 ESPIPE 错误。
7. **高层 I/O = `FILE*` 流**——`fopen("file", "r")` 返回 `FILE*`；`fread(buf, size, nmemb, fp)` / `fwrite` / `fprintf` / `fscanf` / `fgets` / `fputs` / `feof` / `ferror` / `fflush`。**`fgetc` 返回 `int`**（不是 char）——`EOF` 是 -1，`char` 0..255 装不下。
8. **`fopen` mode 字符串**：`"r"`（读）/ `"w"`（写，截断）/ `"a"`（追加）/ `"r+"`（读写，不截断）/ `"w+"`（读写，截断）/ `"a+"`（读+追加）；加 `b` = 二进制（Windows 必需，Linux 透明）。
9. **`fscanf` 返回值三态**：
   - **正数 N** = 成功转换 N 个字段
   - **0** = 输入有但转换失败（不是 EOF）
   - **`EOF`（-1）** = 流结束（end-of-stream）
   **EOF 是"条件"不是"数据"**——读管道关闭时触发。
10. **缓冲 vs 无缓冲**：底层 `read(fd, &c, 1)` 看似"无缓冲"，但 OS 仍会从磁盘 block 读（如 4KB）→ 用户拿 1 字节。**C 标准库的"缓冲"指的是用户态的 stdio buffer**（如 8KB 用户态 buffer），不是 OS 内核。**`fgetc` 实际触发 4KB 用户态 buffer 填充**。
11. **`setvbuf` 控制 stdio 缓冲**：`setvbuf(fp, NULL, _IONBF, 0)` = 无缓冲（每次 syscall）；`_IOLBF` = 行缓冲（终端默认）；`_IOFBF` = 全缓冲（文件默认）。**`stderr` 默认无缓冲**（保证及时输出）；`stdout` 行缓冲（终端）或全缓冲（管道/文件）。
12. **非阻塞 I/O**：`O_NONBLOCK` flag 让 `read` 立即返回 -1 + `errno=EAGAIN`（"暂时无数据"）而不是阻塞。**`fcntl(fd, F_SETFL, O_NONBLOCK)` 可后改**；`select`/`poll`/`epoll` 是"先检查再读"的事件驱动模式（ch6 用）。
13. **命名管道 FIFO**：`mkfifo(path, mode)` 创建特殊文件；`open(path, O_RDONLY)` 在读端会**阻塞**直到有写端，反之亦然。**FIFO = First In First Out 队列**（字节流）。**ROS2 DDS / 日志系统** 都用 FIFO。
14. **管道 vs 套接字**：管道**单向**（一端写、一端读）；套接字**双向**。管道只能用于**有亲缘关系**的进程（父子）或**有名**（FIFO）跨进程；套接字**跨主机**也能用。
15. **FIFO 的"失败 = 正常"模式**：fifoReader 在 writer `usleep` 期间反复 read 失败（32M 次失败对 768K 次成功）——**非阻塞 I/O 的代价**。ch6 用 `select`/`epoll` 解决："先问 OS 有没有数据，没就别 read"。

## §2 核心 Takeaways

### T1 — 底层 I/O 永远是 `int fd` + 字节数；高层 I/O 是 `FILE*` + 多字节
- **是什么**：底层用文件描述符（int），高层用 `FILE*`（opaque 指针）。两者**不可混用**——`fclose(FILE*)` 不接受 `int fd`；`close(int fd)` 不接受 `FILE*`。
- **为什么重要**：网络 socket fd 用底层 API（`read`/`write`）；C 标准 I/O 走 `FILE*`。**`fdopen(fd, "r")` 把底层 fd 包装成 `FILE*`**；`fileno(fp)` 反向。
- **解决什么**：API 风格统一、性能调优（高层有缓冲，底层无）。
- **适用场景**：所有 I/O 密集型程序。

### T2 — `open` flags 是位 OR；mode 是 9 位 rwxrwxrwx
- **是什么**：`O_RDONLY` = 0、`O_WRONLY` = 1、`O_RDWR` = 2（**低 2 位**专用）。其余 flags 占用高位：`O_CREAT` (0x40)、`O_APPEND` (0x400)、`O_NONBLOCK` (0x800)、`O_TRUNC` (0x200)、`O_CLOEXEC` (0x80000)。**`mode` 用 8 进制最清晰**：`0644` = 所有者 rw-、组 r--、其他 r--。
- **为什么重要**：误用 flags 是 file I/O 头号 bug 源。**`O_CREAT` 而无 `O_EXCL`** = 静默覆盖现有文件（**安全漏洞**）。**`O_RDWR` 而不 `O_TRUNC`** 写"hello"到 "world123" 文件 = 文件变 "hello123"。
- **解决什么**：文件创建、并发安全、权限控制。
- **适用场景**：所有 `open` 调用；尤其**安全敏感**（web server 上传文件）。

### T3 — `read` 三态返回：正数/0/-1 + errno
- **是什么**：`read(fd, buf, n)` 返回 `> 0` = 实际读到字节数（**可能 < n**）；返回 `0` = EOF（对端关闭）；返回 `-1` = 错误，**`errno` 指示具体错误**：`EAGAIN`/`EWOULDBLOCK`（非阻塞但无数据）、`EINTR`（被 signal 中断）、`EBADF`（fd 无效）等。
- **为什么重要**：**`read` 不保证一次读满 `n` 字节**——网络/管道尤其常返回部分读。生产代码必须**循环读**直到读满或 EOF。
  ```c
  ssize_t total = 0;
  while (total < n) {
      ssize_t r = read(fd, buf + total, n - total);
      if (r < 0) { if (errno == EINTR) continue; return -1; }
      if (r == 0) break;  /* EOF */
      total += r;
  }
  ```
- **解决什么**：网络协议解析、文件 IO、管道通信。
- **适用场景**：所有底层 I/O；尤其 ch6 socket 编程**必须掌握**。

### T4 — `fopen` mode 字符串 + `fclose` + 缓冲策略
- **是什么**：`"r"` 读、`"w"` 写截断、`"a"` 追加；`+` 加读写；`b` 加二进制（Windows 必需）。`setvbuf` 控制缓冲：`_IONBF` (无) / `_IOLBF` (行) / `_IOFBF` (全)。
- **为什么重要**：**`"w"` 静默截断** = 误用则数据丢失；**`fclose` 前 `fflush`** 在 fork 后双缓冲里必须做；**`setvbuf(fp, NULL, _IONBF, 0)`** 关闭缓冲（调试时立即看输出）。
- **解决什么**：日志输出、配置文件、IPC 序列化。
- **适用场景**：所有高层 I/O；尤其**多进程** + `fork` 后的 stdout/stderr 同步。

### T5 — `fscanf` 返回值要分清 N/0/EOF
- **是什么**：`fscanf(fp, fmt, ...)` 返回**已转换的字段数**。**N > 0** = 成功；**0** = 输入有但转换失败（**不是 EOF！**）；**EOF (-1)** = 流结束。
- **为什么重要**：`while (fscanf(...) != EOF)` 是常见 idiom；**但 `!= EOF` 会让 `fscanf` 返回 0 时进入死循环**（如 "abc" 配 `%i` 永远转不出数字，但流不结束）。**正确**用 `while (fscanf(...) == 1)`。
- **解决什么**：解析文件、CLI 输入、协议解析。
- **适用场景**：所有 `scanf`/`fscanf` 调用。
- **陷阱**：`%i` 配 `0x` 前缀 = 16 进制；`%d` 配 `0` 前缀 = 8 进制；**`%d` 不接受 `0x`**；用 `%i` 更宽容。

### T6 — `O_NONBLOCK` + `fcntl` 是非阻塞 I/O 的双工具
- **是什么**：打开时 `O_NONBLOCK` flag；运行时 `fcntl(fd, F_SETFL, flags | O_NONBLOCK)` 后改。
- **为什么重要**：网络服务器、ROS2 DDS、数据库连接池**全用非阻塞**——单线程服务数千连接。**但单纯 `O_NONBLOCK` 的 read 忙等 = 浪费 CPU**——必须配 `select`/`poll`/`epoll`（ch6 详解）。
- **解决什么**：高并发服务器、实时响应、避免线程开销。
- **适用场景**：所有 C10K 性能优化。

### T7 — FIFO（命名管道）是**进程间最简单**的 IPC
- **是什么**：`mkfifo(path, 0666)` 创建；两端 `open` 后形成字节流通道。`shell` 可用 `cat > fifo &` 写、`cat < fifo` 读。
- **为什么重要**：比 socket 简单（无 bind/listen/accept）；比共享内存安全（自动同步）；**适合单机进程间单向流数据**（日志、监控数据、音频/视频流）。
- **解决什么**：单机进程解耦、生产者-消费者模式。
- **适用场景**：logging、metrics、跨语言 IPC（Python → C）。
- **限制**：`O_RDONLY` open 会**阻塞到有写端**；非阻塞需 `O_NONBLOCK` 配 `O_RDONLY` 才不阻塞。

### T8 — `lseek` 仅对 regular file 有效
- **是什么**：`lseek(fd, offset, SEEK_SET/CUR/END)` 移动文件指针。**pipe / socket / FIFO 不可 seek**——返回 -1 + `errno=ESPIPE`。
- **为什么重要**：网络服务器 / 管道编程用 `lseek` 编译能过但运行时必败。**Linux `pread`/`pseek`** = `read` + `lseek` 原子化版本，更安全。
- **解决什么**：数据库、文件格式解析、随机访问文件。
- **适用场景**：磁盘文件（log 索引、数据库）；不用在网络/管道。

### T9 — `fdopen` + `fileno` 是 fd 与 FILE\* 的桥
- **是什么**：`FILE* fpd = fdopen(fd, "r");` 把 fd 包装成 FILE*；`int fd = fileno(fp);` 反向。
- **为什么重要**：**网络 socket fd 想用 `fprintf`/`fgets`** = `fdopen` 包装；**`FILE*` 传给 `select`** = `fileno` 取回 fd。
- **解决什么**：高层 + 底层 API 混用。
- **适用场景**：socket + 协议解析（用 `fgets` 读 HTTP 头）、日志写到 socket。

### T10 — `stderr` 默认无缓冲、`stdout` 视情况
- **是什么**：`stderr` = `_IONBF`（每次 syscall 立即输出）；`stdout` 终端 = `_IOLBF`（行缓冲，遇 `\n` 输出），管道/文件 = `_IOFBF`（全缓冲，缓冲区满才输出）。
- **为什么重要**：`printf("..."); fflush(stdout);` 是 shell 提示的必须；`fprintf(stderr, ...)` 默认立即出（crash debug 关键）。**`fork` + `printf` 不显式 flush** = 输出可能丢（双缓冲）。
- **解决什么**：调试输出、crash dump、shell 提示。
- **适用场景**：所有交互式程序、所有 daemon。

## §3 工程实践视角

### 3.1 Linux 系统开发视角

- **`O_CLOEXEC` 是 2026 年的硬规则**——`open()` 加 `O_CLOEXEC` 让 fd 在 `exec` 时自动关闭；**防止**"父进程有 fd，子进程 exec 后这个 fd 仍开着造成泄漏"。**Linux kernel 5.11+ 默认开启 systemd 所有服务的 `O_CLOEXEC`**；**systemd unit 用 `PrivateTmp` 隔离**。
- **`pipe2(O_CLOEXEC | O_NONBLOCK)`** = `O_CLOEXEC` + `O_NONBLOCK` 一步到位；**`fcntl` 两次调用可被 signal 中断**。
- **`eventfd` / `signalfd` / `timerfd`** = Linux 专用 fd，可与 `select`/`poll`/`epoll` 整合——比 pipe 更高效。**systemd、Docker、QEMU** 全用。
- **`io_uring` (Linux 5.1+)** = 异步 I/O 终极武器——单 syscall 提交多个 read/write；`libuv`、`libev`、`tokio` 在底层用它。**比 `epoll` 更快**。
- **`sendfile` / `splice` / `tee`** = 零拷贝：文件 → socket 不经过用户空间。Nginx 静态文件、`rsync` 全用。
- **Nginx 配置里 `use epoll;`** = 替代 `select`/`poll`；处理 10K 并发连接。
- **`close` 失败 = 严重**——`close(fd)` 失败时 `errno` 可能是 `EIO`（磁盘 I/O 错误）；**`fsync` 失败时数据可能丢**。数据库 fsync 失败 = 数据完整性危机。
- **`<unistd.h>` POSIX vs `<io.h>` Windows**：`open`/`read`/`write`/`close` 在 Linux/macOS 是 POSIX；Windows 是 `_open`/`_read`/`_write`/`_close`。**跨平台**用 `#ifdef _WIN32` 包装，或用 `fopen`/`fread`（高层更可移植）。
- **glibc `stdio` 性能**：`fread` 大块（4KB+）时性能接近 `read`；小块（1B）时 `fread` 慢 5-10×。**性能敏感**走底层 + 自管理 buffer。

### 3.2 机器人软件视角（ROS2 / 嵌入式控制）

- **ROS2 DDS 中间件** = UDP multicast + 共享内存（同一主机）——**FIFO 用在 logging**。`ros2 bag record` 把 DDS 消息写到一个 `mcap` 文件（库内部用 mmap + 写）。
- **`ros2 topic echo --csv`** = 读 DDS topic 写 stdout，**重定向到文件** 是常用调试手法。
- **嵌入式 /proc 伪文件** = 读 `/proc/cpuinfo`、`/proc/meminfo`、`/proc/net/dev` ——**用 `open`+`read`+`sscanf`**，是 Linux 系统监控基础。
- **STM32 UART 重定向** = `fputc`/`fgetc` 重定向到 UART——用 `setvbuf` 设无缓冲，让 `printf` 立即输出。
- **MoveIt 调试** = `rclcpp::get_logger()` 的 `RCLCPP_INFO` 走 `fprintf(stderr, ...)` 默认无缓冲；**`RCLCPP_INFO_THROTTLE`** 限频避免刷屏。
- **ros2_control 状态机 IPC** = 共享内存 + 互斥锁（ch7 详解），**不是 FIFO**——**FIFO 字节流、共享内存结构体** 是不同设计选择。
- **机器人 SLAM map save/load** = `open(O_RDWR | O_CREAT, 0644)` + `mmap` 写 grid map，比 `fwrite` 快 10×。
- **CycloneDDS / Fast DDS 共享内存传输** = `/dev/shm/` POSIX shm（ch7 详解），是 ROS2 同机进程最常用 IPC。

### 3.3 初级 vs 高级工程师对照

| 习惯 | 初级 | 高级 |
|---|---|---|
| 打开文件 | `open(path, O_RDWR \| O_CREAT, 0666)` | 明确分 `O_RDWR` 还是 `O_WRONLY \| O_CREAT \| O_TRUNC`；`O_CLOEXEC` |
| `read` | `read(fd, buf, n); if (n != read_count) error;` | 循环读到 n 字节或 EOF；处理 `EAGAIN`/`EINTR` |
| `close` | 失败不查 | 必查；`fsync` 失败要 log |
| `fopen` | `fopen("log", "w")` | `setvbuf(fp, NULL, _IOLBF, 0);` 控制缓冲；`O_CLOEXEC` 等价（`fopen` 默认 close-on-exec） |
| `fscanf` | `while (!feof(fp)) fscanf(...)` | 检 `fscanf` 返回值（== 期望字段数） |
| 非阻塞 | `O_NONBLOCK` 裸 read 忙等 | `epoll` + 事件驱动 |
| FIFO | `mkfifo` + `cat` | 配 `O_NONBLOCK` 避免阻塞；`select` 检查 reader/writer |
| 调试输出 | `printf("DEBUG: ...")` 后用 `printf` | `fprintf(stderr, ...)`（无缓冲 + 区分日志/数据） |
| 大文件 | `fgets` 行读 | `fread` 大块 + `lseek` 跳读 |

## §4 AI 时代视角

### 4.1 这些知识还重要吗？（2026 年视角）

**极重要。** ch5 是**所有 I/O 密集型 C 程序的入门**——网络服务器、数据库、ROS2、嵌入式、医疗设备、工业控制 100% 涉及。AI 生成的 C 代码在 ch5 的错误率**与 ch3 内存管理相当**——尤其网络 socket 编程（ch6）会直接基于 ch5 套路。

现代工程师的 ch5 日常：
- **生成 I/O 代码** = 100% 靠 LLM，但**需要 review** flags 选型、错误处理
- **性能调优** = 看 `strace` 找 syscall 瓶颈；改 `O_NONBLOCK` + `epoll`
- **多语言 IPC** = Python/Rust 调 C 库（FFI）必经 ch5 范式

### 4.2 AI 现在能做的

- ✅ 写 `open`/`read`/`write`/`close` 完整循环
- ✅ 解释 fd 0/1/2 与 stdin/stdout/stderr 关系
- ✅ 写 FIFO writer/reader 双进程
- ✅ 解释 `O_NONBLOCK` + `fcntl` 用法
- ✅ 写 `fopen`/`fread`/`fprintf`/`fscanf` 标准用法

### 4.3 AI 经常写错的地方（必看）

| 错误模式 | 例子 | 后果 |
|---|---|---|
| **1. 假设 `read` 一次读满** | `read(fd, buf, n); process(buf);` | 网络/管道只读 100 字节，剩下丢了；**必须循环** |
| **2. 不处理 `EINTR`** | `read(fd, buf, n);` 在 signal 后返回 -1 | 必须 `if (errno == EINTR) continue;` |
| **3. `O_CREAT` 不加 `O_EXCL`** | `open(path, O_CREAT \| O_WRONLY, 0644)` | **安全漏洞**：静默覆盖现有文件；用 `O_CREAT \| O_EXCL` 检测存在 |
| **4. `O_WRONLY` 写到 "r" 打开的文件** | mode 字符串 vs fd flags 不匹配 | fd 0/1/2 写错方向 = 数据丢失 |
| **5. 漏 `fsync`** | 写文件后没 `fsync` | 系统崩溃 = 数据丢失；数据库必须 `fsync` 后才 ack |
| **6. 漏 `O_CLOEXEC`** | 多进程 fork 后 fd 还在 | fd 泄漏到子进程，exec 后仍未关 |
| **7. `fscanf` 返回值判错** | `while (fscanf(fp, "%i", &n) != EOF)` | "abc" 配 `%i` 永远返回 0 ≠ EOF → 死循环；用 `!= 1` |
| **8. `fgets` 不检查返回值** | `fgets(buf, n, fp); process(buf);` | EOF 时 buf 是旧内容；必须检 `NULL` |
| **9. `fprintf` 格式不匹配** | `fprintf(fp, "%s", 42);` | UB；用 `-Wformat` 抓 |
| **10. `lseek` 到 pipe** | `lseek(pipe_fd, ...)` | 编译过，运行时 `ESPIPE` 错 |
| **11. FIFO `O_RDONLY` 阻塞** | `open(fifo_path, O_RDONLY)` 无 `O_NONBLOCK` | 阻塞到有写端；服务启动失败 |
| **12. `O_NONBLOCK` 裸 read 忙等** | `while (1) { read(fd, buf, 1); if (errno == EAGAIN) usleep(1); }` | CPU 100% 浪费；用 `epoll` |
| **13. 漏 `fflush` 在 `fork` 后** | `printf("..."); fork();` | 父 + 子各自输出 1 次（双缓冲）；必须 `fflush(NULL);` |
| **14. `close` 失败不查** | `close(fd);` 完事 | 数据可能未刷盘；查 errno |
| **15. 误用 `"w"` vs `"a"`** | 想追加用 `"w"` | 静默截断文件 |
| **16. `fclose` 失败不查** | `fclose(fp);` 完事 | 缓冲数据可能丢；用 `if (fclose(fp) == EOF) error;` |

### 4.4 工程师必须保留的核心能力

- **能在 5 秒内画出 fd 0/1/2 + 重定向流向**——debug 服务的必备。
- **能 30 秒内写出"循环 read"模板**——所有底层 I/O 必备。
- **能区分 file / pipe / socket 三类的 I/O 能力**——`lseek`/`pread` 仅 file 可用；`O_NONBLOCK` 三类都能用；`sendfile` 仅 file → socket。
- **能用 `strace` 跟踪 syscall**——`strace -p PID`、`strace -e openat,read,write ./prog`。
- **能用 `lsof` 看 fd 表**——`lsof -p PID` 查进程所有打开的 fd。
- **能跟 AI 说"用 `O_CLOEXEC`、`O_NONBLOCK` 配 `epoll`、循环 read 处理 `EINTR`"**——ch5 词汇是 prompt 的关键。

### 4.5 wasm 工具链延伸（用户已选需要）

- **wasm 没有真文件系统**——`<unistd.h>` 的 `open`/`read`/`write` 在 wasm 里走 emscripten 的 `MEMFS`（内存文件系统）。**`mkfifo` 不支持**——wasm 是单进程，进程间 IPC 不需要。
- **wasm 的 stdout/stderr** = `console.log`/`console.error`（浏览器）/ Node.js 的 stdout。**`printf` 在浏览器里能工作但性能差**。
- **wasm 的 fd 概念是"虚拟的"**——emcc 提供 `STDIN_FILENO` 等宏（值仍是 0/1/2），但**实际是 console I/O**。**AI 写的网络代码在 wasm 里要特殊适配**——socket 不存在。
- **AI 工具链中 wasm 的 ch5 角色**：Claude Code / Codex 沙箱跑用户 C 代码时，**所有 I/O 走 wasm 的"虚拟 fd"层**——**没有真磁盘、没真网络**。AI 必须理解这个限制才不会写出"打开 URL"的代码。

## §5 实践行动项

### A1 — 底层 I/O：写 + 读 + 错误检查
```bash
mkdir -p /tmp/modern-c/ch05 && cd /tmp/modern-c/ch05
cat > lowlevel.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>
int main(void) {
    /* 写 */
    int fd = open("data.bin", O_CREAT | O_WRONLY | O_TRUNC, 0644);
    if (fd < 0) { perror("open"); return 1; }
    int nums[5] = {9, 7, 5, 3, 1};
    ssize_t w = write(fd, nums, sizeof(nums));
    if (w != sizeof(nums)) { perror("write"); close(fd); return 1; }
    if (close(fd) < 0) { perror("close"); return 1; }
    /* 读 */
    fd = open("data.bin", O_RDONLY);
    if (fd < 0) { perror("open read"); return 1; }
    int read_in[5];
    ssize_t r = read(fd, read_in, sizeof(read_in));
    if (r < 0) { perror("read"); close(fd); return 1; }
    printf("read %zd bytes, nums:", r);
    for (int i = 0; i < 5; i++) printf(" %d", read_in[i]);
    printf("\n");
    close(fd);
    /* 演示 O_EXCL 防止覆盖 */
    fd = open("data.bin", O_CREAT | O_EXCL | O_WRONLY, 0644);
    printf("O_EXCL: fd=%d (期望 -1, file exists)\n", fd);
    if (fd >= 0) close(fd);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -Werror -o lowlevel lowlevel.c && ./lowlevel
xxd data.bin | head -1   ## 看二进制: 09 00 00 00 07 00 00 00 ...
```
**验收**：输出 `read 20 bytes, nums: 9 7 5 3 1`；`O_EXCL` 返回 -1（文件已存在）；`xxd` 看到 20 字节二进制（int 4B × 5）。

### A2 — `fopen` mode + `setvbuf` 缓冲
```bash
cat > highlevel.c <<'EOF'
#include <stdio.h>
int main(void) {
    /* 写 + 立即 flush */
    FILE* fp = fopen("out.txt", "w");
    if (!fp) { perror("fopen"); return 1; }
    setvbuf(fp, NULL, _IOLBF, 0);   /* 行缓冲 */
    fprintf(fp, "hello, world\n");
    /* fclose 会 flush; 这里 fflush 演示 */
    fflush(fp);
    /* 读 */
    fp = fopen("out.txt", "r");
    char buf[64];
    /* fgets 返回 NULL 表示 EOF 或错误 */
    while (fgets(buf, sizeof(buf), fp)) {
        printf("got: %s", buf);
    }
    if (ferror(fp)) printf("read error\n");
    if (feof(fp))   printf("reached EOF\n");
    fclose(fp);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -o highlevel highlevel.c && ./highlevel
cat out.txt
```
**验收**：输出 `got: hello, world` + `reached EOF`（**不是** `read error`，证明 fgets 正常 EOF 不是错误）。

### A3 — `fscanf` 返回值三态 + 防死循环
```bash
cat > fscanf_demo.c <<'EOF'
#include <stdio.h>
int main(void) {
    FILE* fp = fopen("nums.txt", "w+");
    fprintf(fp, "10 20 30 abc 40\n");
    rewind(fp);
    int n, count = 0;
    /* 正确：检查 == 期望字段数 (1) */
    while (fscanf(fp, "%i", &n) == 1) {
        printf("got %d\n", n);
        count++;
    }
    printf("converted %d ints, then stream ended or failed\n", count);
    fclose(fp);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -o fscanf_demo fscanf_demo.c && ./fscanf_demo
```
**验收**：输出 10, 20, 30, 40；遇到 "abc" 后 fscanf 返回 0（**不是 EOF**），循环退出；**没有死循环**。

### A4 — 命名管道 FIFO 双进程
```bash
cat > fifo_writer.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
int main(void) {
    const char* p = "./testfifo";
    mkfifo(p, 0666);
    int fd = open(p, O_WRONLY);  /* 阻塞到 reader */
    for (int i = 1; i <= 5; i++) {
        char buf[32];
        int n = snprintf(buf, sizeof(buf), "msg %d\n", i);
        write(fd, buf, n);
        usleep(100000);
    }
    close(fd);
    unlink(p);
    return 0;
}
EOF
cat > fifo_reader.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
int main(void) {
    const char* p = "./testfifo";
    int fd = open(p, O_RDONLY);
    char buf[64];
    ssize_t n;
    while ((n = read(fd, buf, sizeof(buf)-1)) > 0) {
        buf[n] = '\0';
        printf("read: %s", buf);
    }
    close(fd);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -o fifo_writer fifo_writer.c
gcc -std=c11 -Wall -Wextra -o fifo_reader fifo_reader.c
# 终端 1:
./fifo_reader &
# 终端 2:
./fifo_writer
```
**验收**：reader 收到 5 条 "msg 1..5"；writer 关闭后 reader 收到 EOF 退出。

### A5 — `O_NONBLOCK` + `fcntl` 非阻塞 read
```bash
cat > nonblock.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
int main(void) {
    int fd = open("nonblock.c", O_RDONLY | O_NONBLOCK);
    char buf[16];
    ssize_t r;
    int success = 0, again = 0, error = 0;
    while (1) {
        r = read(fd, buf, sizeof(buf));
        if (r > 0)      success++;
        else if (r == 0) break;             /* EOF */
        else if (errno == EAGAIN) { again++; usleep(1000); }
        else            { error++; break; }
        if (success + again > 100) break;  /* 安全退出 */
    }
    printf("success=%d again=%d error=%d\n", success, again, error);
    close(fd);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -o nonblock nonblock.c
# 普通文件几乎不会 EAGAIN (内核提前预读), 改用 FIFO 管道试
mkfifo /tmp/np 2>/dev/null
# 简单 writer: 直接打开 FIFO 写端; nonblock.c 是 reader
printf 'test\n' > /tmp/np &
timeout 1 ./nonblock
wait $! 2>/dev/null
rm -f /tmp/np
```
**验收**：从管道读会看到 `success=N again=M error=0`（非阻塞 + usleep 退避）；**没有 CPU 100%**。

## §6 值得深入思考的问题

1. **`O_CLOEXEC` 是什么时候进 POSIX 的？为什么 systemd 强制开？** POSIX.1-2008 引入；之前用 `fcntl(fd, F_SETFD, FD_CLOEXEC)` 模拟。**systemd unit 的 `NoNewPrivileges=yes` + `O_CLOEXEC` 是防 docker escape 的关键**。**为什么 fork 后 fd 泄漏是 CVE 头号源？**
2. **`read` 返回 0 = EOF**——为什么管道的 EOF 是"写端关闭"，不是"读完所有数据"？`shutdown(sock, SHUT_WR)` 后 read 收到 0、write 收到 EPIPE。**TCP 半关闭** = 双向独立。
3. **`O_NONBLOCK` 的"忙等"怎么破？** `epoll` (Linux) / `kqueue` (BSD/macOS) / IOCP (Windows) / `io_uring` (Linux 5.1+)。**为什么 Node.js 用 libuv 跨平台抽象这四者？** ROS2 rclcpp 的 `wait_set` 机制是不是同一思路？
4. **FIFO 在 2026 年还有用吗？** vs Unix domain socket：FIFO 单向字节流；socket 双向。**Docker/K8s 的 `kubectl exec` 用 Unix socket**，**systemd journald 用 Unix socket**。**FIFO 主要是历史工具，新代码首选 Unix socket**。
5. **`fsync` 失败 = 数据可能丢**——为什么数据库（PostgreSQL、SQLite）必须 `fsync` 后才给 client 返回 commit？**Linux 5.15+ 的 `fsync` 错误机制变化**（之前 silently ignore，现在返回 `EIO`）——**对数据库意味着什么**？
6. **`FILE*` 缓冲 vs 内核 page cache**——为什么"flush 到磁盘"实际是"写到内核 page cache"？`fsync` 才是真写到磁盘。**Docker volume 性能问题** = page cache 透过 mount 看不到，导致双缓冲。
7. **AI 写 C I/O 代码 80% 错在 flags 选型 + 错误处理不完整**——你能写一个 PR review checklist（10 条），CI 里 grep 检查？`clang-tidy` 的 `bugprone-*` 规则里哪个最相关？

---

*下一章预告*：**ch6 Networking**——41 页中等章。`socket`/`bind`/`listen`/`accept`/`connect`/`send`/`recv` 全套 syscall；HTTP 客户端（`gethostbyname` + `connect` + `read`/`write`）；**event-driven 服务器**（`select`/`poll`/`epoll` 三选一，**ch5.7 的非阻塞 I/O 升华**）；OpenSSL TLS 套接字（`SSL_CTX_new` / `SSL_new` / `SSL_connect` / `SSL_read` / `SSL_write`）。这一章与机器人 ROS2 DDS、MoveIt web UI、autoware.auto 的 cloud 通信都直接相关。
