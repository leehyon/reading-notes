# Ch7. Concurrency and Parallelism

> 对应 PDF 物理页 246-304（印刷页 231-289）。本章 59 页（全书第二重头），C 并发全景：fork 多进程 + pipe + exec + 共享内存 + 文件锁 + 消息队列 + pthread 多线程 + mutex + race condition + deadlock + SIMD 指令级并行。**ch6 网络的服务器（多 client 必走并发）+ 嵌入式 MCU 实时控制（裸机无 OS）**都依赖本章。

## §1 章节概述

1. **并发 vs 并行**：并发 = 同时处理多任务（单核也成立，preemptive 调度）；并行 = 真正同时跑（多核或多指令）。**SIMD 是指令级并行**——1 个指令作用于多个数据。
2. **fork 进程 vs pthread 线程**：
   - **进程**：独立地址空间；context switch 5-15ms（要换 page table）；适合 CPU 密集+隔离。
   - **线程**：共享地址空间；context switch ns 级（仅寄存器）；适合 I/O 密集+共享数据。
   - Linux 实际上把线程当"轻量级进程"——threads = tasks from scheduler POV。
3. **`fork()` 三态返回**：父进程收到子进程 pid；子进程收到 0；失败 -1。**`if (pid == 0)` 分支子进程；`else` 分支父进程**——经典 fork-if。
4. **僵尸进程（zombie）**：子进程已退出但父进程未 `wait()` 收尸——PID 仍在 process table。**防御 3 法**：`signal(SIGCHLD, SIG_IGN)` 自动 reaper / 父 `waitpid(-1, &st, 0)` / 子用 `_exit` (不走 atexit 清理)。
5. **`exec` 族 = 替换进程镜像**：当前进程 pid 不变，但 text/data/heap/stack 全换成新程序。**exec 成功后**原代码**不再执行**——所以"exec 后 free(args)" 这种代码是死代码（Kalin 7-3 printf 注释明确）。6 个变体（`execv`/`execvp`/`execve`/`execl`/`execle`/`execlp`）按参数形态分。
6. **POSIX 共享内存** = `shm_open` + `ftruncate` + `mmap(MAP_SHARED)` + `sem_open/sem_wait/sem_post`。Linux 上**共享内存文件在 `/dev/shm/`**。`sem_post` 增 1、`sem_wait` 等到 >0 减 1。
7. **POSIX vs System V 共享内存**：POSIX 用 `shm_open` + 文件名；System V 用 `shmget(key, ...)` + numeric key。**不要混用**。
8. **文件锁** = `fcntl(fd, F_SETLK/F_GETLK, &flock)` 配合 `struct flock { l_type, l_whence, l_start, l_len, l_pid }`。`F_WRLCK` 排他、`F_RDLCK` 共享、`F_UNLCK` 解锁。`F_SETLK` 不阻塞、`F_SETLKW`（带 W = wait）阻塞。
9. **消息队列 (System V)**: `msgget(key, flags)` 拿 qid；`msgsnd(qid, &msg, size, IPC_NOWAIT)` 发；`msgrcv(qid, &msg, size, type, flags)` 按 type 收；`msgctl(qid, IPC_RMID, NULL)` 删。**比 FIFO 强**：按 type 优先级收，**不强制 FIFO**。`ftok(path, proj_id)` 把路径转成 key。
10. **`pthread_create(&tid, NULL, fn, arg)` 4 参**：tid / attr (默认 NULL) / start fn / arg to fn。`pthread_join(tid, &ret)` 阻塞等线程结束。`pthread_exit(NULL)` main 退出但线程继续。
11. **critical section = 必须原子执行的代码段**。`account += n` 看似一行，实际是"读 + 加 + 写"3 步。**2 线程并发** 1000 万次 +/-1 后 account 几乎不可能为 0（**真实值**：Kalinn 跑出 203692 和 -1800416）。
12. **`pthread_mutex_t` 4 函数**：`pthread_mutex_init` / `pthread_mutex_lock` / `pthread_mutex_unlock` / `pthread_mutex_destroy`。`PTHREAD_MUTEX_INITIALIZER` 静态初始化。**只保护"账户"这类共享资源**，不保护"线程局部变量"（栈已隔离）。
13. **deadlock 4 必要条件**（Coffman）：① 互斥 ② 持有并等待 ③ 不剥夺 ④ 循环等待。**`pthread_mutex_trylock` + 锁顺序排序** 是常见预防。
14. **SIMD 指令级并行**：Intel SSE2 128-bit / AVX 256-bit / AVX-512 512-bit 寄存器；8/16 元素并行算。**GCC 扩展 `__attribute__((vector_size(N)))`** 让你直接写 `doubleV8 a = b + c;` 编译成 YMM 指令。
15. **Flynn 4 象限**：SISD（单核）/ SIMD（GPU 向量）/ MISD（罕见，航天容错）/ **MIMD**（多核 + 多线程 = 主流）。本章 7.7 pthread 是 MIMD；7.9 SIMD 是 SIMD。

## §2 核心 Takeaways

### T1 — `fork()` 复制整个进程，COW 优化让它"轻"
- **是什么**：`fork()` 创建子进程，父子各一份独立的虚拟地址空间。**Linux 用 Copy-On-Write**：子进程"读时共享"父的物理页，只在"写时"才真正复制。
- **为什么重要**：COW 让 `fork()` 几乎瞬时（μs 级）；但**子进程一旦写内存就触发 page fault**。**`fork()` 之后立刻 `exec()`** 是传统 Unix server 模式（Nginx master-worker fork）——避免子进程复制父的大内存。
- **解决什么**：快速多进程 + 隔离 + 共享初始状态。
- **适用场景**：daemon 化（`fork` 两次让 init 接管）、Nginx worker pool、pre-fork web server（apache mpm_prefork）。

### T2 — 僵尸进程 = 父没 `wait()` 的孤儿
- **是什么**：子进程退出，PID 还留在 process table（Z 状态）。**kernel 不能立刻删 PID**——等父来确认。父一直不 wait → PID 表满 → 不能再 fork。
- **解决什么**：进程泄漏。
- **防御**：
  1. `signal(SIGCHLD, SIG_IGN)` 一劳永逸（Linux 特有，**POSIX 不保证**）
  2. `waitpid(-1, &st, WNOHANG)` 在 event loop 里非阻塞 reaper
  3. **子用 `_exit()`** 而非 `exit()` —— 跳过 atexit + stdio flush，加速父的 SIGCHLD
- **适用场景**：所有 long-running daemon。

### T3 — `exec` 替换进程镜像，**原代码不再运行**
- **是什么**：`execv(path, argv)` 成功后，当前进程的 text/data/heap/stack **整个换掉**，PID 不变。**从 exec 调用后**的代码（包括返回值检查 `if (-1 == ret)`）在成功时**永远不执行**。
- **为什么重要**：**exec 是原子操作**——如果失败才返回 -1，所以 `execv` 之后写 `free` / `printf` 是**死代码**。
- **解决什么**：父进程 fork 出一个"运行新程序"的子进程 = Unix 经典 fork-exec 模式（shell 启动命令的原理）。
- **6 个变体**：`execl`/`execv`/`execle`/`execve`/`execlp`/`execvp`——`l` = list 参数、`v` = vector 参数；`e` = 自带 env；`p` = PATH 搜索。`execve` 是**唯一 syscall**，其他都是 libc wrapper。

### T4 — 共享内存是最快的 IPC
- **是什么**：多进程**映射同一段物理内存**（POSIX `/dev/shm/foo` 文件 + `mmap(MAP_SHARED)`）。`memcpy` 一次即 IPC 完成。
- **为什么重要**：**比 pipe/socket 快 100×**——零拷贝。**Redis、PostgreSQL、systemd 内部**都大量用。
- **代价**：**没有同步**——必须自己加 `sem_t` 或 `pthread_mutex`（注意 pthread mutex 是**进程内**，跨进程要 `sem_t`）。
- **API 4 件套**：`shm_open` 创建 + `ftruncate` 设大小 + `mmap` 映射 + `shm_unlink` 清理（**不 unlink 会泄漏到 /dev/shm**）。

### T5 — 文件锁 vs `flock` (BSD) vs `fcntl` (POSIX)
- **是什么**：**`fcntl` 是 POSIX**（Linux 默认），`flock` 是 BSD（macOS 默认）——**不要混用**。`fcntl` 用 `struct flock { l_type, l_whence, l_start, l_len, l_pid }` 5 字段，**支持字节区间锁**。
- **`F_SETLK` vs `F_SETLKW`**：前者立即返回（成功或 EAGAIN），后者阻塞等锁。
- **场景**：多进程**写同一文件**（如数据库 WAL、log）。**比 mutex 慢**——跨进程要走 syscall。

### T6 — 消息队列 = 带 type 标签的 FIFO
- **是什么**：System V `msgget` 拿 qid；`msgsnd` / `msgrcv` 按 `long type` 过滤。**type > 0 收 ≥ type 的最小一条**；**type == 0 收最早一条**；**type < 0 收 ≤ |type| 的最小一条**。
- **为什么重要**：比 pipe 灵活（按 type 优先级），比 socket 简单（不需要 bind/listen）。**但** System V IPC 在容器化里**已不流行**（`/dev/shm` 默认 64MB 上限，docker 共享 namespace 容易冲突）。
- **现代替代**：Redis Streams / RabbitMQ / ZeroMQ。但 ch7 是教学，IPC 概念不过时。

### T7 — `pthread_create` 4 参 + `pthread_join` 必配对
- **是什么**：`pthread_create(&tid, NULL, fn, arg)` 启动；`pthread_join(tid, &ret)` 阻塞等。**main 线程必须 join 全部子线程**——否则 main 退出 → 整个进程退出 → 未 join 的线程被 kill。
- **为什么重要**：**`pthread_exit` 不让进程结束**——但 POSIX 不保证。**Linux 行为：main 调 `pthread_exit(NULL)` 才让子线程继续**（ch7.7 listing 7-12）。
- **新手坑**：忘记 join → 子线程"幽灵运行"被 SIGKILL；忘记 init mutex → 静态 `PTHREAD_MUTEX_INITIALIZER` 是 default attrs，**recursive / errorcheck 必须 init**。

### T8 — `pthread_mutex` = 临界区入口
- **是什么**：`pthread_mutex_lock(&m)` 阻塞到获锁；`unlock` 释放。**只保护共享变量**，不保护栈/堆局部变量。
- **性能**：**无竞争时** ~25ns (现代 x86) **比 atomic 快**（atomic 用 LOCK prefix）。
- **类型**：
  - `PTHREAD_MUTEX_NORMAL`（默认）— 死锁后行为未定义（**永远不解锁**）
  - `PTHREAD_MUTEX_ERRORCHECK` — 死锁返回 EDEADLK
  - `PTHREAD_MUTEX_RECURSIVE` — 同一线程可重入（**慢**，用 `pthread_rwlock` 替代）
- **不要用在 ISR** — pthread mutex 不是 ISR-safe，嵌入式用 `stdatomic` + `atomic_flag`。

### T9 — race condition 的根源 = read-modify-write 非原子
- **是什么**：`account += n` 在 CPU 层面是 3 步（load / add / store）。**多线程** 抢占这 3 步 → lost update。
- **Miser/Spendthrift** 演示：1000 万次 +1 + 1000 万次 -1，**理论结果 0**，实测 20 万或 -180 万（差 18 万到 180 万次 lost update）。
- **修复**：mutex 包 critical section（**ch7.7 listing 7-15**）→ 1000 万次 = 0。

### T10 — deadlock 4 必要条件 + 4 防御
- **是什么**：
  1. 互斥访问
  2. 持有并等待（持有一锁等另一锁）
  3. 不可剥夺
  4. 循环等待
- **破坏任一即不死锁**：
  1. lock-free 数据结构（atomic）
  2. 一次性取所有锁
  3. `pthread_mutex_trylock` 失败就释放已有
  4. 锁顺序排序（全局 `lock A before B` 约定）

### T11 — SIMD 让单核也能并行
- **是什么**：CPU 的 SSE2/AVX/AVX-512 寄存器（128/256/512 bit），1 条指令处理多个数据元素。**8 个 double 加法 = 1 条指令**（AVX-512）。
- **GCC 扩展**：`typedef double doubleV8 __attribute__((vector_size(64)));` —— 直接写 `doubleV8 c = a + b;` 编译成 YMM 指令。
- **限制**：所有元素必须同类型；不能跨界寄存器宽度。
- **应用**：图像处理（RGB 像素）/ SIMD 物理仿真 / 神经网路 forward pass / hash 加速。
- **现代工具**：自动向量化 `-O3 -ftree-vectorize`；intrinsic 函数 `<immintrin.h>` 显式控制；CUDA / OpenCL / SYCL 做 GPU SIMD。

### T12 — `__attribute__((vector_size))` 是 GCC 扩展
- **是什么**：GCC/Clang 支持的 type attribute，让 C 结构体被当 SIMD 寄存器。
- **限制**：**非标准 C**——MSVC 不支持；**C++ 中可移植写法是 `__m256d` 等 `<immintrin.h>` 类型**。
- **替代**：
  - **C11 `<stdalign.h>` + union**——最可移植
  - **`<immintrin.h>` intrinsic**——直接对应 AVX 指令
  - **`#pragma omp simd`**——OpenMP SIMD 指令

## §3 工程实践视角

### 3.1 Linux 系统开发视角

- **`fork+exec` 模式 = 容器化基础**——Docker 容器本质是 `unshare(CLONE_NEWNS) + pivot_root`；runc 内部也用 `clone3`。**K8s kubelet 创建 pod 也是 fork+exec pattern**。
- **Nginx master-worker** = `fork()` 出 worker 进程，共享 listen fd（`SO_REUSEPORT`）。worker 死掉 master 立刻 `fork` 新的。**vs Apache prefork = pre-fork + per-connection fork**（慢但简单）。**vs Apache worker/event = pthread per connection**（更快但需 mutex）。
- **systemd 是 fork-exec 之王**——service 文件每一行都是 `ExecStart=` 触发 `fork+exec`。`systemd-run` / `systemd-cgtop` 都用 cgroup + namespace。
- **Linux `clone3(2)` (since 3.19)** = `fork`/`pthread_create`/`unshare` 的统一接口；用 flags 决定共享什么（CLONE_VM / CLONE_FILES / CLONE_FS 等）。**Docker 容器内部 100% 走 clone3**。
- **POSIX `mqd_t` (POSIX 消息队列, `<mqueue.h>`) vs System V `msgget`**——POSIX 版的优点是 `/dev/mqueue/` 文件系统可视化（`mq_overview` 工具）、关闭 fd 自动清理。**新代码优先 POSIX**。
- **`eventfd` vs `pipe` (ch5)** —— Linux 专用：单 `uint64` 计数器；`write(2)` 增 1；`read` 一次性读完。**比 pipe 快、占 fd 少**——systemd / Docker / QEMU 内部用。
- **`io_uring` 替代 `epoll+threadpool`**——Linux 5.1+ 单 syscall 提交多个 I/O；Nginx 实验支持。
- **`/proc/sys/kernel/pid_max`** 默认 32768 = 单进程最多 fork 这么多次；**容器里要调大**。
- **`prctl(PR_SET_CHILD_SUBREAPER, 1)`**——让进程收养所有孤儿（init 角色）；Docker container 用。

### 3.2 机器人软件视角（ROS2 / 嵌入式控制）

- **ROS2 DDS 用共享内存** = Cyclone DDS Iceoryx / Fast DDS Shared Memory Transport；**`/dev/shm/`** 下的 POSIX shm。**比 UDP multicast 快 10-100×**。配合 ch7.4 的 `shm_open/mmap/sem_*` API（虽然 ROS2 用 iceoryx 包装过）。
- **ROS2 rclcpp executor** = 多线程任务调度 = pthread pool（`rclcpp::executors::MultiThreadedExecutor`）。`callback_group` 决定 callback 在哪个线程跑——**线程安全靠 `std::mutex`/`std::atomic` + callback 内部串行**。
- **ros2_control hardware interface** 实时循环 = `pthread_create(SCHED_FIFO, ...)` 配 Linux RT-PREEMPT；priority 99。**mutex 用 `PTHREAD_PRIO_INHERIT` 防 priority inversion**。
- **MoveIt IK 求解器多线程** = `std::thread::hardware_concurrency()` 检测 CPU 核数 → OpenMP `#pragma omp parallel for` 并行算 8 个候选 IK → SIMD 加速（`#pragma omp simd`）。
- **CUDA host code** = `fork` 出 worker 进程 × `pthread_create` 出 worker 线程 × `cudaLaunchKernel` 出 GPU thread block。**3 层并行**。
- **Webots / Gazebo 仿真器** = master 进程 + N 个 robot 进程（每 robot 一进程） + 共享内存同步状态。**Deadlock 经典**——仿真循环里 mutex 顺序要全局一致。
- **RTOS 嵌入式（无 OS）**：pthread 不可用；用 **`xTaskCreate` (FreeRTOS) / `tx_thread_create` (ThreadX)**——同样 mutex/semaphore API。
- **机器人 IPC 选型**：
  - 单机多进程 = `shm_open + sem_t` 或 **Unix domain socket**
  - 跨机 = UDP multicast (DDS) 或 TCP (rosbridge)
  - 实时 = RT-PREEMPT pthread + `SCHED_FIFO`
- **AI 训练并行** = PyTorch DataLoader 用 `torch.multiprocessing`（本质 fork）；`DistributedDataParallel` 用 NCCL over GPU-direct RDMA + Gloo over TCP。

### 3.3 初级 vs 高级工程师对照

| 习惯 | 初级 | 高级 |
|---|---|---|
| 多进程启动 | `system("prog &")` | `fork+execvp`+`waitpid` 收尸 |
| 僵尸进程 | 不管 / 用 `atexit` 兜底 | `signal(SIGCHLD, SIG_IGN)` 或 event loop `waitpid(-1, &st, WNOHANG)` |
| 共享内存 | 不用 | `shm_open + ftruncate + mmap(MAP_SHARED) + sem_t` |
| 线程 | `pthread_create` 后不 join | 必 `pthread_join`；main 用 `pthread_exit(NULL)` 让子线程继续 |
| mutex | 一锁到底 | `PTHREAD_MUTEX_ERRORCHECK` 防死锁自检；`pthread_mutex_trylock` 防饥饿 |
| race condition | 加 mutex 全锁 | 优先 `atomic` (C11) / `std::atomic` (C++)，锁最小粒度 |
| deadlock 防御 | 不知道 | 锁顺序文档化 + `lockdep` 工具（kernel 自带） |
| SIMD | 不用 | `__attribute__((vector_size))` 或 `<immintrin.h>` 直接写 AVX |
| 消息队列 | 用 pipe | 按 type 用 `msgget` / `mq_open`，IPC_NOWAIT 防阻塞 |
| fork 后 fd | 全部继承 → 资源浪费 | `FD_CLOEXEC` + 关键 fd 显式 close |

## §4 AI 时代视角

### 4.1 这些知识还重要吗？（2026 年视角）

**极重要。** 任何高并发服务（数据库/消息队列/web server/AI 推理/机器人控制）= ch3 + ch4 + ch5 + ch6 + **ch7** 的综合。**AI 写的 C 并发代码** 90% 错在**"看似能跑，实际有 race/deadlock"**——必须靠工具（TSan、Helgrind）抓。

现代工程师的 ch7 日常：
- **AI 写 multithreaded C** = 必跑 **ThreadSanitizer (TSan)** 编译运行才能信
- **AI 写 fork/exec** = 必看 fd 泄漏表
- **AI 写共享内存** = 必查 `shm_unlink` 配对
- **AI 写 SIMD** = 必看 GCC 文档 + CPU feature flag (`-march=native`)

### 4.2 AI 现在能做的

- ✅ 写 `fork+exec` 完整 demo（含 `waitpid` 收尸）
- ✅ 解释 race condition 机制（read-modify-write 非原子）
- ✅ 写 pthread + mutex 标准模板
- ✅ 推荐 mutex vs atomic vs spinlock 的选择
- ✅ 解释 SIMD 寄存器和 GCC `vector_size` attribute

### 4.3 AI 经常写错的地方（必看）

| 错误模式 | 例子 | 后果 |
|---|---|---|
| **1. `pthread_create` 后忘 `pthread_join`** | `pthread_create(&t, NULL, fn, NULL);` 完事 | main 退 → 整个进程死 → 子线程"幽灵运行"被 SIGKILL |
| **2. mutex 忘 unlock** | `pthread_mutex_lock(&m); /* 忘 unlock */` | 死锁，**所有等此锁的线程永久卡** |
| **3. mutex 在 fork 后没重置** | `fork()` 之后子进程**继承**父的 mutex 状态 | 子进程 unlock 一个它没 lock 的 mutex → 未定义行为（Linux 通常 silently ignore） |
| **4. race condition 用 mutex 但 critical section 不完整** | `account += n` 不在 mutex 里 | 数据竞争 lost update |
| **5. signal handler 里调 pthread 函数** | `void handler(int sig) { pthread_mutex_lock(&m); }` | signal handler 只能用 **async-signal-safe** 函数（`write`、`_exit`、`sem_post`）；其他 UB |
| **6. `signal(SIGCHLD, SIG_IGN)` 当万灵药** | Windows 跨平台代码 | POSIX 不保证；Windows 无 SIGCHLD；用 `waitpid` 才对 |
| **7. `fork()` 在多线程程序** | 4 线程 + `fork()` | 只有调用 fork 的线程存在子进程里；**mutex 状态混乱**——用 `pthread_atfork` 处理 |
| **8. `exec` 后写死代码** | `execv("/bin/ls", args); printf("失败\n");` | exec 成功时 printf 永远不执行；**先 printf 再 exec**，或检查 ret |
| **9. `shm_open` 后不 `shm_unlink`** | `/dev/shm/foo` 一直残留 | 下次启动看到"已存在"；重启 OOM |
| **10. `mmap` 后忘 `munmap`** | 共享内存段常驻 | 资源泄漏 |
| **11. `sem_wait` 死锁（多 semaphore）** | `sem_wait(&a); sem_wait(&b);` 反向另一线程 | 经典 deadlock；用锁顺序或 `sem_timedwait` |
| **12. deadlock demo 改成"修复"但忘删 `usleep`** | ch7.8 listing 7-16 注释 `/* fix is in! */` | 修复后 usleep 让本来不 deadlock 的情况反而 deadlock |
| **13. 假设 `pthread_cancel` 立即终止** | `pthread_cancel(t);` 后立即 free(t 的资源) | cancel 是 deferred cancellation point；用 `pthread_join` + cleanup_push |
| **14. SIMD 用 `__attribute__` 但不开 `-march=native`** | `typedef doubleV8 ...` 但编译 `-O2` | 编译器 fallback 到 scalar 循环，**无 SIMD** |
| **15. `mmap(NULL, size, ...)` 但 `size` 是 0** | size 计算溢出得 0 | 失败（EINVAL）；用 `calloc(s, n)` 算 size |
| **16. `fork()` 在多线程后立刻 `_exit()` 而非 `exit()`** | `exit()` 调 atexit 处理器 | atexit 在子进程里跑错（lock 状态混乱）；用 `_exit` |
| **17. 假设 `pipe2(O_CLOEXEC)` 自动避免 fork 泄漏** | pipe 用 `O_CLOEXEC` 但其他 fd 没设 | 其他 fd 仍泄漏到子进程；**全局 `-fvisibility=hidden + O_CLOEXEC`** |

### 4.4 工程师必须保留的核心能力

- **能 5 秒内画出 fork 后父子进程的地址空间图**。
- **能 30 秒内写出"safe fork 模板"**（`fork_atfork` 注册 + 子进程 `_exit` + 父 `waitpid`）。
- **能用 ThreadSanitizer (TSan) 跑**——`-fsanitize=thread` 编译，**所有 race condition 都报**。
- **能用 Helgrind / DRD 跑**——`valgrind --tool=helgrind` 抓 race + deadlock。
- **能区分 mutex / spinlock / atomic / rwlock**——按场景选。
- **能跟 AI 说"用 `pthread_atfork` + TSan 编译 + lockdep 验证"**——ch7 词汇是 prompt 的关键。

### 4.5 wasm 工具链延伸（用户已选需要）

- **wasm 几乎不支持 pthread**——emscripten 提供 `pthread` stub（编译过但实际是 worker）；`fork` 完全没；`mmap` 用 `MEMFS` 内存模拟；**`sem_*` 是 counter 模拟**。
- **wasm 的并发** = **Web Workers** + SharedArrayBuffer（COOP/COEP 头开启）+ Atomics。**Wasm threads 提案**让 pthread 直接编译成 wasm threads——**2026 年已部分支持**（Chrome/Firefox/Safari）。
- **AI 工具链中 wasm 的 ch7 角色**：Claude Code / Codex 沙箱跑用户 C 代码，**多线程靠 WASM threads 提案 + SharedArrayBuffer**；`pthread_mutex` → wasm atomic ops；`fork` 不支持（沙箱单进程）；`mmap(MAP_SHARED)` → SharedArrayBuffer。

## §5 实践行动项

### A1 — fork + waitpid 收尸（ch7.2/7.3 实战）
```bash
mkdir -p /tmp/modern-c/ch07 && cd /tmp/modern-c/ch07
cat > fork_wait.c <<'EOF'
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
int main(void) {
    pid_t pid = fork();
    if (pid == -1) {
        perror("fork");
        return 1;
    }
    if (pid == 0) {
        /* child */
        printf("child: pid=%d, ppid=%d\n", getpid(), getppid());
        _exit(0);   /* 关键: _exit 而非 exit, 跳过 atexit */
    }
    /* parent */
    int status;
    pid_t r = waitpid(pid, &status, 0);
    if (r == -1) {
        perror("waitpid");
        return 1;
    }
    if (WIFEXITED(status)) {
        printf("parent: child %d exited with %d\n", r, WEXITSTATUS(status));
    }
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -O2 -o fork_wait fork_wait.c && ./fork_wait
```
**验收**：看到 child 输出 + parent 收到 status 0；**`ps -ef | grep fork_wait` 退出后无残留**。

### A2 — POSIX 共享内存 + semaphore（ch7.4 实战，需 `-lrt -lpthread`）
```bash
cat > shmem.h <<'EOF'
#define BackingFile "/shmem_demo"
#define SemaphoreName "/shmem_demo_sem"
#define AccessPerms 0644
#define ByteSize 512
#define MemContents "Hello from shared memory!\n"
EOF

cat > shm_writer.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <semaphore.h>
#include <string.h>
#include "shmem.h"
void report_and_exit(const char* msg) {
    perror(msg);
    exit(-1);
}
int main(void) {
    int fd = shm_open(BackingFile, O_RDWR | O_CREAT, AccessPerms);
    if (fd < 0) report_and_exit("shm_open");
    ftruncate(fd, ByteSize);
    caddr_t memptr = mmap(NULL, ByteSize, PROT_READ | PROT_WRITE,
                           MAP_SHARED, fd, 0);
    if (memptr == (caddr_t) -1) report_and_exit("mmap");
    sem_t* semptr = sem_open(SemaphoreName, O_CREAT, AccessPerms, 0);
    if (semptr == (void*) -1) report_and_exit("sem_open");
    strcpy(memptr, MemContents);
    sem_post(semptr);  /* 让 reader 进 */
    sleep(5);          /* 给 reader 5s */
    munmap(memptr, ByteSize);
    close(fd);
    sem_close(semptr);
    shm_unlink(BackingFile);   /* 关键: 不删就泄漏 */
    sem_unlink(SemaphoreName);
    return 0;
}
EOF

cat > shm_reader.c <<'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <semaphore.h>
#include "shmem.h"
static void report_and_exit(const char* msg) { perror(msg); exit(-1); }
int main(void) {
    int fd = shm_open(BackingFile, O_RDWR, AccessPerms);
    if (fd < 0) report_and_exit("shm_open");
    caddr_t memptr = mmap(NULL, ByteSize, PROT_READ | PROT_WRITE,
                           MAP_SHARED, fd, 0);
    if (memptr == (caddr_t) -1) report_and_exit("mmap");
    sem_t* semptr = sem_open(SemaphoreName, 0);
    if (semptr == (void*) -1) report_and_exit("sem_open");
    sem_wait(semptr);   /* 等 writer */
    printf("reader got: %s", memptr);
    sem_post(semptr);
    munmap(memptr, ByteSize);
    close(fd);
    sem_close(semptr);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -O2 -o shm_writer shm_writer.c -lrt -lpthread
gcc -std=c11 -Wall -Wextra -O2 -o shm_reader shm_reader.c -lrt -lpthread
./shm_writer &
WPID=$!
sleep 1
./shm_reader
wait $WPID
```
**验收**：reader 输出 `Hello from shared memory!`；writer 5s 后退出；`ls /dev/shm/` 无残留 `/shmem_demo`。

### A3 — pthread + mutex（ch7.7.2 Miser/Spendthrift 修复）
```bash
cat > thread_counter.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#define N 10000000
static int counter = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
static void* worker(void* arg) {
    for (int i = 0; i < N; i++) {
        pthread_mutex_lock(&lock);
        counter++;
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}
int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, worker, NULL);
    pthread_create(&t2, NULL, worker, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("expected %d, got %d\n", 2 * N, counter);
    pthread_mutex_destroy(&lock);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -O2 -o thread_counter thread_counter.c -lpthread && ./thread_counter
echo "--- 不加 mutex (race condition) ---"
sed 's/pthread_mutex_lock(&lock);//; s/pthread_mutex_unlock(&lock);//' thread_counter.c > race.c
gcc -std=c11 -O2 -o race race.c -lpthread && ./race
```
**验收**：带 mutex 输出 `expected 20000000, got 20000000`；不带 mutex 输出其他值（race condition lost update）。

### A4 — 用 TSan 抓 race condition
```bash
# TSan 编译
gcc -std=c11 -fsanitize=thread -g -O1 -o thread_counter_tsan thread_counter.c -lpthread
./thread_counter_tsan
# 期望: 警告 "data race" + stack trace
```
**验收**：TSan 报 `WARNING: ThreadSanitizer: data race` 并指出 `counter++` 位置（如果先跑无 mutex 版本）。

### A5 — SIMD 验证 (`__attribute__((vector_size))`)
```bash
cat > simd.c <<'EOF'
#include <stdio.h>
#define Length 8
typedef double doubleV8 __attribute__((vector_size(Length * sizeof(double))));
int main(void) {
    doubleV8 v1 = {1.1, 2.2, 3.3, 4.4, 5.5, 6.6, 7.7, 8.8};
    doubleV8 v2 = {2.0, 2.0, 2.0, 2.0, 2.0, 2.0, 2.0, 2.0};
    doubleV8 sum = v1 + v2;     /* 一条 SIMD 指令, 8 个 double 加法 */
    for (int i = 0; i < Length; i++) printf("%f ", sum[i]);
    putchar('\n');
    return 0;
}
EOF
gcc -std=c11 -O2 -march=native -o simd simd.c
./simd
echo "--- 看汇编 (验证 SIMD) ---"
gcc -std=c11 -O2 -march=native -S simd.c -o -
grep -E "vadd|ymm|zmm" simd.s 2>/dev/null | head -3
```
**验收**：输出 `3.100000 4.200000 ...` 8 个值；汇编有 `vaddpd ymm0, ymm1, ymm2`（AVX）或 `vaddpd zmm0, zmm1, zmm2`（AVX-512）。

## §6 值得深入思考的问题

1. **`fork()` 后多线程的 mutex 状态**怎么处理？Linux `pthread_atfork(prepare, parent, child)` 注册 3 回调——prepare 在 fork 前 lock 所有 mutex，parent 在 fork 后立即 unlock，child 在 fork 后**只 unlock 不重置状态**。**Docker 容器内部用 `clone3` 替代 fork**就是为了避免这个复杂性。**AI 写跨平台代码怎么处理？**
2. **deadlock 4 必要条件**怎么在 C 代码层强制检查？**Linux kernel 自带 `lockdep`**——编译时打开，所有锁顺序都被记录，违反时报 warning。**userspace** 用 `liblockdep` 或 **`pthread_mutex` 加 errorcheck 属性**。**Rust 没有这个问题**因为 `std::sync::Mutex` 自带 poison 机制。
3. **SIMD 编译的实际门槛** = CPU 至少 SSE4.2（2010 后 x86 全支持）。但 **`-march=native` 编译的 binary 不能跨 CPU 跑**——老 CPU 没 AVX-512 指令就 SIGILL。**正确做法**：默认 `-msse2`（1999 起的最低），用 `__attribute__((target("avx2")))` 局部函数选择性加速；运行期 `getauxval(AT_HWCAP)` 选实现。**FFmpeg / x264 / OpenCV** 都有这层 dispatch。
4. **POSIX vs System V IPC 在容器化的命运**？**System V 用 `shmget(key, ...)` + numeric key**——容器之间**共享 namespace 容易冲突**；**POSIX 用 `shm_open(name, ...)` + 文件名**——同样问题。**Docker 默认** `--ipc=private`（独立 IPC namespace）。**K8s 默认每个 pod 独立 IPC**。**生产微服务**几乎不用共享内存——改用 Redis / message broker。
5. **AI 写多线程 C 代码怎么测？** = **ThreadSanitizer (TSan) 是答案**。`gcc -fsanitize=thread` 编译，跑测试。**Helgrind** 是另一个选择但慢 10×。**DRD (Data Race Detector)** 是 Helgrind 的轻量版。**Mutex 顺序问题** Linux kernel 有 `lockdep`；userspace 用 `liblockdep`。
6. **wasm threads 提案 2026 状态**？Chrome / Firefox / Safari 2023 起支持。`pthreads` 编译成 wasm + SharedArrayBuffer。**emcc `-pthread` 选项自动启用**。**WASI 0.2 (2024) 标准化**。**Claude Code / Codex 沙箱**已经支持 wasm pthread——**意味着 ch7 的 `pthread_mutex` 在 wasm 里能跑**。
7. **`fork()` vs `clone3(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND)` 的差异**？**`clone3` 可以只共享部分资源**——Docker 容器就是这种用法（`CLONE_NEWNS`/`CLONE_NEWPID`/`CLONE_NEWNET`）。**Kubernetes runc 也是**。**`fork()` 是 `clone3(CLONE_CHILD_SETTID | SIGCHLD)` 的简化版**。

---

*下一章预告*：**ch8 Miscellaneous Topics**——54 页收尾章。`regex.h` 正则表达式（`regcomp`/`regexec`） / `<assert.h>` 断言（`assert` + `NDEBUG` + `static_assert` C11） / locale 与 i18n（`setlocale` + Unicode） / **WebAssembly**（emcc 工具链把 C 编成 wasm；`EMSCRIPTEN_KEEPALIVE`） / `<signal.h>` 信号（`SIGUSR1`/`SIGCHLD` + 自定义 handler） / 动态库构建（`ar rcs` + `nm` + `LD_LIBRARY_PATH` + C 和 Python 互调 `ctypes`）。**这一章是"工具箱"——把 ch1-ch7 串起来**。
