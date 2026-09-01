# 知识体系总图

> 来源: *Modern C: Up and Running* (Kalin, 2022) 8 章六段式精读
> 落盘日期: 2026-08-31
> 用途: **工作中随时反查**——按"工程能力维度"重组，不是按章节复述
> 阅读量: 全文 ~224 KB 章节笔记的索引层

## 总图导览

本书 8 章内容**横向切**成 6 个工程能力维度：

| 维度 | 章节范围 | 核心问题 |
|---|---|---|
| **A. C 语言骨架与编译链** | ch1 | "C 程序从源码到运行经过哪些阶段？main 是什么角色？" |
| **B. 类型与内存表示** | ch2, ch3 | "这个值在内存里长什么样？指针和值如何转换？" |
| **C. 存储与作用域** | ch4 | "变量活多久、谁看得到？static vs extern vs volatile 怎么用？" |
| **D. I/O 与系统调用** | ch5 | "open/read/write 这套 API 怎么用？和 socket/pid/shm 是一回事吗？" |
| **E. 网络与并发** | ch6, ch7 | "TCP server 怎么搭？fork/pthread/shared memory 选哪个？" |
| **F. 工程化与跨语言** | ch8 | "怎么建库、怎么跨语言、怎么跑在浏览器？" |

每个维度下面，按"**知识卡片**"组织——每张卡片都是工作中**立即可用**的"该做什么/不该做什么"。

---

# A. C 语言骨架与编译链 (ch1)

## A1. C 程序的 4 阶段编译 (ch1.4)

```
.c  →[preprocess]→  .i  →[compile]→  .s  →[assemble]→  .o  →[link]→  binary
       cpp             cc             as            ld
```

**调试工具**：
- `-E` 看 `.i`（宏展开）——**为什么这个条件没生效？**
- `-S` 看 `.s`（汇编）——**为什么这段 C 这么慢？**
- `gcc --save-temps` 三件套全留

**反模式**：在源码层调试优化问题——必须看 `.s`，编译器把 `n++` 优化成 `add` 后源码层找不到。

## A2. main 的"三态"返回值 (ch1.3)

- `return 0` / `return EXIT_SUCCESS` —— 正常
- `return -1` / `return EXIT_FAILURE` —— 异常
- **省略 return** —— C99 起隐式 `return 0`，但**显式更稳**

**嵌入式中** `void main()` + 死循环 是惯例，**不是 C 强制**。

## A3. 命令行参数 (ch1.5)

```c
int main(int argc, char* argv[]);   // argc >= 1
// argv[0] = 程序名, argv[1..] = 用户参数
```

**模式**：
```c
if (argc < 2) { fprintf(stderr, "Usage: %s <arg>\n", argv[0]); return 1; }
```

**AI 经常漏**：`argv[i]` 直接当 `&argv[i]` 用（`argv` 已是 char* 数组，不需要 `&`）。

## A4. 变长参数 (ch1.8)

```c
double avg(int count, ...);   // count 是协议锚点
va_list ap; va_start(ap, count);
for (int i = 0; i < count; i++) sum += va_arg(ap, int);
va_end(ap);
```

**协议**：调用方必须传 count / sentinel / format string 告诉被调方"还有几个"。

**经典坑**：`va_arg(ap, int)` 实际传 `double` —— x86-64 上 double 走 XMM 寄存器，**静默读到垃圾**。

---

# B. 类型与内存表示 (ch2, ch3)

## B1. IEEE 754 三类浮点 (ch2.3)

| 类型 | 特征 | 范围 (float 32-bit) |
|---|---|---|
| **normalized** | 指数位非全 0 也非全 1 | 1.2e-38 ~ 3.4e+38 |
| **denormalized** | 指数位全 0 | 1.4e-45 ~ 1.2e-38（更小但精度低） |
| **special** | 指数位全 1 | ±0、±inf、NaN |

**5 类浮点异常** = overflow / underflow / division by zero / invalid operation / inexact

**AI 经常错**：`0.1 + 0.2 != 0.3`（最后 1 bit 不同）——必须 `fabs(a-b) < DBL_EPSILON`。

## B2. `int` 的"32-bit 假设" (ch2)

**C 标准**：`int` 至少 16 bit。**实际**：x86-64 Linux 32 bit；x86-64 Windows 32 bit；**wasm32 是 32 bit 但 `long` 是 32 bit（不是 64）**。

**可移植铁律**：
```c
#include <stdint.h>   // 关键头文件
int32_t n;            // 永远 32 bit
uint64_t u;           // 永远 64 bit
intptr_t p;           // 指针大小整数
size_t sz;            // sizeof 返回类型
```

**AI 经常错**：`sizeof(int) == 4` 写死在代码里。

## B3. 整数 wraparound 防御 (ch2)

```c
/* 错误: 直接加可能溢出 */
if (a + b > UINT_MAX) goto overflow;

/* 正确: 先消除 wrap */
if (a > UINT_MAX - b) goto overflow;
```

**`abs(INT_MIN)` 是 UB**——C 标准未定义；POSIX 安全；用 `int64_t + llabs` 显式宽度。

## B4. 字节序 + 字节对齐 (ch2, ch3)

- **Endian**: x86 little-endian / network big-endian → `<endian.h>` 的 `htonl/ntohl`
- **Alignment**: `struct { int n; char c; double d; }` sizeof = **16**（不是 13）—— 编译器 padding
- **strict aliasing**：`htonl(*(uint32_t*)buf)` 是 UB → 用 `memcpy(&n, buf, 4); n = htonl(n);`

## B5. 指针算术是"元素级" (ch3.3)

```c
int arr[10];
int* p = arr;
p++;        // 跳 4 字节, 不是 1 字节
p + 5;      // 跳 20 字节
```

**核心心智模型**：`int*` 步长 = `sizeof(int)`。`char*` 步长 = 1。

## B6. 数组 vs 指针 (ch3.4)

| 表达式 | 含义 |
|---|---|
| `arr[i]` | `*(arr + i)` —— **完全等价** |
| `i[arr]` | `*(i + arr)` —— **合法但极丑**（Kalin 演示用） |
| `&arr[i]` | `arr + i` |
| `&arr` | "整个数组"指针，步长 = `sizeof(arr)` |

## B7. `qsort` + 比较函数 = "C 模板" (ch3.7.1)

```c
int comp(const void* p1, const void* p2) {
    return n2 - n1;   /* 危险: 整数溢出 → UB! */
}
/* 正确 */
return (n1 > n2) - (n1 < n2);   /* 安全 -1/0/+1 */
```

**高级**：排序大结构 = 排序**指针数组**（移动 8B 指针 vs 8KB 结构）——8.8.1 sorting pointers to structures。

## B8. 堆管理 3 件套 (ch3.10-12)

```c
void* p = malloc(size);     // 不初始化
void* p = calloc(n, sz);     // 初始化为 0
size_t n = malloc_usable_size(p);  // 实际分配（glibc 扩展）

/* realloc 安全模式 */
void* new = realloc(p, 2*size);
if (!new) { free(p); return ERROR; }   /* 原指针仍有效! */
p = new;
```

**free 必检**：多次 free / use-after-free 是 60% C 漏洞。

---

# C. 存储与作用域 (ch4)

## C1. `static` 两副面孔 (ch4.4)

- **块内 static** = 持久化局部变量（只初始化一次）—— **单例**模式
- **块外 static** = 文件作用域 + 内部链接（**C 唯一的 "private"**）

```c
static int g_state;   /* 文件内可见, 链接器不导出 */

/* 函数内 */
void counter(void) {
    static int n = 0;   /* 跨调用保留 */
    n++;
}
```

## C2. `extern` = ODR 跨文件

```c
/* file1.c */  int g_count = 0;                  /* 定义: 不写 extern */
/* file2.c */  extern int g_count;                /* 声明: 必须 extern, 不初始化 */
```

**头文件规则**：
- 头文件只放 `extern` 声明
- 头文件**不放** `int g = 0` 定义（**每个 include 它的 .c 都有一份 = 多重定义**）
- 头文件**不放** `static` 函数（每个 .c 各自一份 = 不共享）

## C3. `volatile` 是嵌入式/多线程的"防优化器"硬约束 (ch4.6)

```c
volatile uint8_t data_ready = 0;  /* ISR 与主循环共享 */
volatile int g_signal;            /* 多线程共享 */
volatile uint32_t* const reg = (volatile uint32_t*)0x40021000;  /* MMIO */
```

**反模式 (2026)**：
- 多线程共享 `int` = **必须用 `<stdatomic.h>` 的 `atomic_int`**（volatile 不保证原子性、不保证内存序）
- 编译器会在 volatile 变量**每次访问都走内存**——滥用会让性能暴跌 10-100×

**嵌入式 ISR**：volatile 必加（ch4 活证据：删 volatile 后 main 死循环）

## C4. `register` 和 `auto` 是反模式 (ch4.3)

- `register` —— C++17 deprecated；**永远不写**——`gcc -O1` 比 `register` 强
- `auto` —— C 里是"默认的 auto"（无意义），**永远不写**——避免和 C++ `auto` 混淆

## C5. `const char*` vs `char* const` (ch4 / ch8)

| 声明 | 含义 |
|---|---|
| `const char* p` | **数据不可变**，指针可变 |
| `int* const p` | **指针不可变**，数据可变 |
| `const int* const p` | 都不可变 |

**"右往左读"**规则：从变量名开始，向左读。

**AI 经常错**：以为 `int* const p` 锁数据——其实锁的是指针。

---

# D. I/O 与系统调用 (ch5)

## D1. fd 是统一抽象 (ch5.2)

**fd 0/1/2** = stdin/stdout/stderr 预留给进程。**所有打开的资源都是 fd**：
- 文件 (ch5.2-4)
- 管道 (ch5.7)
- 套接字 (ch6.2)
- 共享内存 (ch7.4)
- 事件通知 (eventfd/signalfd/timerfd)

`dup`/`dup2`/`fdopen`/`fileno` 把 fd 和 FILE* 互转。

## D2. `open` 标志位 OR 组合 (ch5.2.1)

```c
int fd = open(path, O_RDWR | O_CREAT | O_EXCL, 0644);
                 /*  ←————  flags  ————→    ←mode→ */
```

| 常用 flag | 含义 |
|---|---|
| `O_RDONLY`/`O_WRONLY`/`O_RDWR` | 0/1/2（低 2 位专用）|
| `O_CREAT` | 不存在则创建 |
| **`O_EXCL` 必加** | 防止静默覆盖现有文件（**安全漏洞**） |
| `O_APPEND` | 追加写 |
| `O_NONBLOCK` | 非阻塞（配 `fcntl` 后改） |
| `O_CLOEXEC` | exec 时自动关闭（2026 硬规则） |
| `O_TRUNC` | 截断（写时） |

**mode 用 8 进制**：`0644` = 所有者 rw-、组 r--、其他 r--。

## D3. read 三态返回 (ch5.2, ch6.2)

```c
ssize_t n = read(fd, buf, size);
/* n > 0  = 实际读到的字节数 (可能 < size!) */
/* n == 0 = EOF (对端关闭) */
/* n == -1 = 错误, 检查 errno */
```

**网络/管道永远部分读**——必须循环：
```c
ssize_t total = 0;
while (total < n) {
    ssize_t r = read(fd, buf+total, n-total);
    if (r < 0) { if (errno == EINTR) continue; return -1; }
    if (r == 0) break;  /* EOF */
    total += r;
}
```

**AI 经常错**：`read` 一次读完假设；不处理 `EINTR`。

## D4. fopen mode 字符串 (ch5.5)

| mode | 含义 |
|---|---|
| `"r"` | 读 |
| `"w"` | 写，**截断**（静默！） |
| `"a"` | 追加 |
| `"r+"` | 读写，不截 |
| `"w+"` | 读写，截断 |
| `"b"` | 二进制（Windows 必需） |

**AI 经常错**：想追加用 `"w"`——数据丢失。

## D5. fscanf 三态返回 (ch5.5)

```c
int n;
fscanf(fp, "%i", &n);   /* 返回 1 = 成功; 0 = 输入有但转换失败; EOF = 流结束 */
```

**反模式**：
```c
while (fscanf(fp, "%i", &n) != EOF) ...   /* "abc"配%i 返回 0 != EOF = 死循环 */
```

**正确**：
```c
while (fscanf(fp, "%i", &n) == 1) ...
```

## D6. 缓冲 vs 无缓冲 (ch5.6)

- `stderr` 默认无缓冲（`_IONBF`）——crash debug 时立即输出
- `stdout` 终端 = 行缓冲（`_IOLBF`）；管道/文件 = 全缓冲（`_IOFBF`）
- `fork` 后 `printf` 双重缓冲——**`fflush(NULL)` 后再 fork**

## D7. `O_NONBLOCK` + `fcntl` 非阻塞 I/O (ch5.7)

```c
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

ssize_t r = read(fd, buf, n);
if (r < 0 && errno == EAGAIN) { /* 暂时无数据 */ }
```

**单纯 `O_NONBLOCK` read 忙等 = 浪费 CPU**——必须配 `select`/`poll`/`epoll` (ch6.3)。

## D8. 命名管道 FIFO (ch5.7.1)

```bash
mkfifo /tmp/mychannel        # 创建
writer > /tmp/mychannel &   # 写入端
reader < /tmp/mychannel      # 读出端
```

`open(path, O_RDONLY)` **阻塞到有写端**——加 `O_NONBLOCK` 才不阻塞。

---

# E. 网络与并发 (ch6, ch7)

## E1. socket = fd 延伸 (ch6.2)

**所有 socket API 跟文件/管道共享 `open/read/write/close` 模式**——ch5 的 fd 知识直接迁移。

## E2. server 3 步 / client 2 步 (ch6.2)

```c
/* server */
int srv = socket(AF_INET, SOCK_STREAM, 0);
bind(srv, (struct sockaddr*)&addr, sizeof(addr));
listen(srv, BACKLOG);
while (1) { int c = accept(srv, NULL, NULL); /* 处理 c */ close(c); }

/* client */
int s = socket(AF_INET, SOCK_STREAM, 0);
connect(s, (struct sockaddr*)&addr, sizeof(addr));
/* 用 s 收发 */
```

**`SO_REUSEADDR` 必加**（防 TIME_WAIT 阻塞重启）。

## E3. `SO_REUSEPORT` (Linux 3.9+) = 多进程 listen 同一端口

**Nginx worker** = 多进程各自 `listen(srv_fd, ...)` 同一端口，**内核做负载均衡**。

## E4. TCP 字节流 = "无消息边界"

**3 次 `write(100B)`** 可能 1 次 `read(300B)`，也可能 3 次 `read(100B)`。

**应用层必须 framing**：HTTP 用 `\r\n\r\n`、WebSocket 2B length prefix、gRPC varint、MQTT fixed header。

## E5. 8 个关键 socket flag (ch6.3)

| flag | 何时设 |
|---|---|
| `SO_REUSEADDR` | 永远 |
| `SO_KEEPALIVE` | 永远（细调 `TCP_KEEPIDLE` 等）|
| `TCP_NODELAY` | 小包交互（SSH / TLS / 控制消息） |
| `SO_LINGER (l_onoff=1, l_linger=0)` | 主动发 RST（防 TIME_WAIT 残留）|
| `SO_RCVTIMEO` | read 超时 |
| `SO_SNDTIMEO` | write 超时（**connect 用这个不靠谱**） |
| `MSG_NOSIGNAL` | 写已关 socket 不发 SIGPIPE |
| `O_NONBLOCK` | 配 `select`/`epoll` |

## E6. OpenSSL 3.0 改 API (ch6.4)

| 旧 API | 新 API (OpenSSL 3.0+) |
|---|---|
| `SSLv23_method()` | `TLS_client_method()` / `TLS_server_method()` |
| `SSL_CTX_new(method)` | 同 |
| `SSL_new(ctx)` | 同 |
| `SSL_set_fd(ssl, fd)` | 同 |
| `SSL_connect(ssl)` | 同 |
| `SSL_read` / `SSL_write` | 同 |
| `SSL_free` / `SSL_CTX_free` | 同 |

**AI 经常错**：`SSLv23_method` 在 OpenSSL 3.0 已弃用；`verify_dc` 写 stub = 中间人攻击。

## E7. fork + exec 模式 (ch7.2-3)

```c
pid_t pid = fork();
if (pid == 0) {          /* child */
    execvp("/bin/ls", args);   /* 成功后原代码不执行 */
    _exit(1);            /* exec 失败, _exit 不走 atexit */
} else {                 /* parent */
    int status;
    waitpid(pid, &status, 0);
    if (WIFEXITED(status)) printf("exit %d\n", WEXITSTATUS(status));
}
```

**exec 后写代码是死代码**（除 `if (ret == -1)` 错误处理）。

## E8. 共享内存 = 零拷贝最快 IPC (ch7.4)

```c
int fd = shm_open("/shm", O_RDWR | O_CREAT, 0644);
ftruncate(fd, SIZE);
void* mem = mmap(NULL, SIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
sem_t* sem = sem_open("/sem", O_CREAT, 0644, 0);
strcpy(mem, "data");
sem_post(sem);   /* 同步 */
munmap(mem, SIZE); close(fd); shm_unlink("/shm");
```

**必配对**：`shm_unlink` 删文件 + `munmap` 释放映射 + `close(fd)` 释放 fd。**`/dev/shm/` 默认 64MB**，Docker 容器注意。

## E9. pthread + mutex (ch7.7)

```c
static int counter = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
void update(int n) {
    pthread_mutex_lock(&lock);
    counter += n;   /* critical section */
    pthread_mutex_unlock(&lock);
}
int main(void) {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, worker, NULL);
    pthread_create(&t2, NULL, worker, NULL);
    pthread_join(t1, NULL);   /* 必 join, 否则 main 退出 → 子线程被 SIGKILL */
    pthread_join(t2, NULL);
    pthread_mutex_destroy(&lock);
    return 0;
}
```

**race condition 必查工具**：**TSan** = `gcc -fsanitize=thread`，**Helgrind** = `valgrind --tool=helgrind`。

**Miser/Spendthrift** 1000 万次 +/-1，**理论 0，实际 ±18 万到 180 万 lost update**。

## E10. deadlock 4 必要条件 + 4 防御 (ch7.8)

1. 互斥
2. 持有并等待
3. 不可剥夺
4. 循环等待

**破坏任一不死锁**：
- atomic（无锁数据结构）
- `trylock` 失败释放（避免持有并等待）
- 锁顺序排序（避免循环等待）

## E11. SIMD = 单核并行 (ch7.9)

```c
typedef double doubleV8 __attribute__((vector_size(64)));
doubleV8 a = {1,2,3,4,5,6,7,8}, b = {2,2,2,2,2,2,2,2};
doubleV8 c = a + b;  /* 1 条指令, 8 个 double 加法 */
```

**编译必须 `-march=native`**——否则 fallback 到 scalar。

---

# F. 工程化与跨语言 (ch8)

## F1. 动态库 3 名字 (ch8.7)

```
logical name: libfoo.so       ← 编译时链接 (gcc -lfoo)
    ↓ symlink
soname:      libfoo.so.1      ← 运行时 dlopen (ld.so 找)
    ↓ symlink
real name:   libfoo.so.1.0.0  ← 物理文件
```

**新装后必跑** `sudo ldconfig`（更新 `/etc/ld.so.cache`）。

**`-fPIC` 必加**——否则 `relocation R_X86_64_PC32 ... can not be used when making a shared object`。

## F2. 链接顺序 = GNU ld 是 lazy

```c
gcc main.c -lfoo -lm     /* OK: libfoo 用 libm 时, 找到后再链 libm */
gcc main.c -lm -lfoo     /* 错: libm 没被 main.c 用, 不链; libfoo 用 sqrt 找不到 */
```

**反模式**：`-lm` 放最后（依赖图从左到右）。

## F3. `signal` vs `sigaction` (ch8.6)

| | `signal` | `sigaction` |
|---|---|---|
| 行为可移植性 | System V / BSD 不一 | POSIX 一致 |
| 推荐 | ❌ legacy | ✅ 2026 默认 |

**handler 必只做 1 件事** = `volatile sig_atomic_t flag = 1;`；主循环 poll 后做实际工作。

**handler 不能调**：`printf` / `malloc` / `pthread_mutex_lock` —— **会死锁**。

## F4. WebAssembly = C 的第二个生命周期 (ch8.5)

```bash
# 编译
emcc foo.c --no-entry -s EXPORTED_FUNCTIONS='["_myfn"]' -o foo.js

# JS 调用
Module._myfn(42)
```

**限制**：
- 浏览器 wasm **没 raw socket / 文件系统**——只有 `fetch()`
- 浏览器 wasm **默认单线程**——要 `SharedArrayBuffer` + `pthread` 支持
- WASI 0.2 (2024) 标准化 = 浏览器外跑 wasm = **Deno / Wasmtime / Wasmer** 替代 Docker 容器

## F5. Python `ctypes` 调 C 库 (ch8.7.5)

```python
import ctypes
lib = ctypes.CDLL("./libfoo.so")
lib.foo.restype = None          # void 函数必设
lib.foo.argtypes = [ctypes.c_int, ctypes.c_char_p]
lib.foo(42, b"hello")
```

**强替代**：`cffi`（更 Pythonic）/ `pybind11`（C++）/ `PyO3`（Rust）。

## F6. setlocale + localeconv = i18n 标准 (ch8.4)

```c
setlocale(LC_ALL, "");  /* 读环境变量 LANG */
struct lconv* lc = localeconv();
printf("%s\n", lc->currency_symbol);  /* $ / £ / € / ¥ */
```

**生产环境**必调 `setlocale(LC_ALL, "")`——否则所有 `printf %f` / `strftime` 都按 C locale。

## F7. regex 三件套 (ch8.2)

```c
regex_t re;
regcomp(&re, "^[A-Z]{2}[1-9]{3}$", REG_EXTENDED);
if (regexec(&re, input, 0, NULL, 0) == 0) { /* match */ }
regfree(&re);
```

**POSIX regex 不支持** lookbehind/lookahead——用 PCRE2。

## F8. assert 三态 (ch8.3)

| assertion | 位置 | 检查 |
|---|---|---|
| precondition | 块前 | 块开始时必须成立 |
| invariant | 块内 | 块执行中始终成立 |
| postcondition | 块后 | 块结束时必须成立 |

`#define NDEBUG` 关闭全部 assert（**release 必加**）；C11 `_Static_assert` 编译期检查。

---

# 全书核心 Takeaways（横向 cheat sheet）

| 主题 | 一句话 |
|---|---|
| 编译 | `gcc -std=c11 -O2 -Wall -Wextra -Werror -Wpedantic -fsanitize=address,undefined foo.c -o foo` |
| 整数 | 用 `<stdint.h>` 显式宽度；**永远不** `abs(INT_MIN)`；wraparound 用减法范式 |
| 浮点 | `fabs(a-b) < FLT_EPSILON`；不用 `0.1+0.2==0.3` |
| 指针 | `void*` 配 `memcpy`；`qsort` 比较函数 `(a>b)-(a<b)`；free 后置 NULL |
| 存储 | `static` 单例 / `extern` ODR；多线程用 `<stdatomic.h>` 不用 volatile；嵌入式 ISR 必 volatile |
| I/O | `open` 必加 `O_EXCL`；read 必循环处理 `EINTR`/`EAGAIN`；`fscanf` 检查 `==期望字段数`；`SO_CLOEXEC` 必加 |
| 网络 | `SO_REUSEADDR` + `MSG_NOSIGNAL`；循环 read；OpenSSL 3.x 用 `TLS_client_method`；`SO_RCVTIMEO` 防永久阻塞 |
| 并发 | fork 后 `_exit`；pthread 必 join；mutex 只保护共享变量；TSan 编译跑测试 |
| 共享内存 | `shm_unlink` 配对；用 `sem_t` 同步；`MAP_SHARED` flag 必加 |
| SIMD | `__attribute__((vector_size))` 配 `-march=native`；现代用 `<immintrin.h>` 显式 intrinsic |
| 库 | 动态 `-fPIC` + `soname` + `ldconfig`；链接 `-lfoo -lm` 不是 `-lm -lfoo` |
| signal | `sigaction` 不是 `signal`；handler 只 set flag；不死锁的关键 |
| WebAssembly | `emcc --no-entry -s EXPORTED_FUNCTIONS='["_fn"]'`；浏览器没 socket/文件 |
| i18n | `setlocale(LC_ALL, "")` + `localeconv()`；强 i18n 用 ICU |
| Python 调 C | `ctypes.CDLL("./libfoo.so")`；`restype` / `argtypes` 必设 |

---

# AI 工具链 "100+ 错例" 横向索引

8 章笔记每章都有"AI 经常写错的地方"清单，**最危险的 15 条**（按出现频次 + 后果严重度）：

1. **switch 漏 break** → `-Wimplicit-fallthrough` 抓
2. **i++ * i++ 链** → UB，C 标准未定义
3. **free 后继续用** → ASan 抓
4. **realloc 失败丢原指针** → 临时指针模式
5. **`if (x & 1 == 0)` 优先级** → `(x & 1) == 0`（ch2 演示）
6. **混合符号比较 `int < unsigned`** → 永远 true/false
7. **多线程共享用 `volatile`** → 用 `<stdatomic.h>`
8. **`abs(INT_MIN)`** → UB；用 `int64_t` + `llabs`
9. **signal handler 用 `printf`** → 死锁
10. **read 不循环** → 网络/管道 partial read
11. **SIGPIPE 默认杀进程** → `MSG_NOSIGNAL` 或 `signal(SIGPIPE, SIG_IGN)`
12. **动态库漏 `-fPIC`** → 链接错
13. **链接顺序 `-lm -lfoo`** → undefined reference
14. **wasm 假设有 socket/文件** → 实际没有
15. **ctypes 调 C 库漏 `restype`** → 返回垃圾 int

完整清单见各章 §4.3。
