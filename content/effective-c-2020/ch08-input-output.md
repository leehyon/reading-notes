# 第 8 章 · Input/Output

> 来源: *Effective C* (Seacord, 2020) — Chapter 8, pp. 147–168
> 笔记日期: 2026-08-27

---

# 一、章节概述

1. **Stream = C 的 I/O 抽象**：FILE * 是对 socket / 键盘 / USB / 打印机 / 文件的**统一接口**。FILE 对象持有位置指示器、buffering 信息、error/eof 标志。
2. **永远不要自己分配 FILE 对象**：只能通过 stdio 函数得到 `FILE *`。
3. **C 标准 I/O 抽象度太高，缺关键能力**：没有 directory 概念、没有 file permission/locking 概念。**真实应用必须用 POSIX/Windows API**。
4. **三种 buffering 模式**：
   - **Unbuffered**：`stderr`（错误日志要立即看到）
   - **Fully buffered**：文件 I/O 攒 block 后一次性 I/O（吞吐优先）
   - **Line buffered**：遇 newline 才 flush（终端）
5. **三个预定义流**：`stdin` / `stdout` / `stderr`。启动时 `stdin/stdout` **在不指向交互设备时 fully buffered**；`stderr` **永远 unbuffered**——保证错误立即可见。
6. **Shell 重定向**：`>` 重定向 stdout，`<` 重定向 stdin，`2>` 重定向 stderr。
8. **Stream orientation**：byte-oriented vs wide-oriented。**第一次 I/O 决定 orientation**。**不要混用 narrow / wide / binary 在同一个流上**。`fwide()` 可重置。
9. **Text stream vs binary stream**：
   - text：有序字符序列组成行，每行以 newline 结尾；**Windows 是 `\r\n`，Linux 是 `\n`**，跨平台会乱
   - binary：任意字节；同实现读写一致（可能末尾补 NUL）
10. **`fopen(filename, mode)` 返回 `FILE *` 或 NULL**：mode 字符串 6 种基础（C89）+ C11 加 `"x"` 独占模式（文件已存在则失败）。
11. **mode 字符串表**：`r/w/a/rb/wb/ab/r+/w+/a+/r+b`/.../`wx/wbx/w+x/w+bx`——C11 的 `x` 是 fail-if-exists 防止 TOCTOU。
12. **永远不要 by-value 复制 `FILE *`**：`FILE my_stdout = *stdout; fputs(..., &my_stdout);` —— **UB + 通常 crash**。
13. **POSIX `open(path, oflag, ...)` 返回 file descriptor（int）**：用 `O_RDONLY` / `O_WRONLY` / `O_RDWR` 之一 + flag（`O_APPEND`/`O_TRUNC`/`O_CREAT`/`O_EXCL`）。
14. **`fileno(fp)` / `fdopen(fd)` 互转**：FILE * 和 fd 之间桥接（fdopen 后必须用 fclose 关闭）。
15. **`fclose` 会 flush + 检查错误**：可以失败（NFS 写错误）；abort() 不 flush → 数据可能丢。**所有 fclose 都检查返回值**。
16. **`close(fd)` 是 POSIX 对应**：失败时 errno = EIO。
17. **`fputc` / `putc` / `putchar` / `fputs` / `puts`** 单字符/单行写出；`putc` 常是 macro（**多次求值 side effect → 危险**，CERT FIO41-C）。
18. **`fgetc` / `getc` / `getchar` / `fgets`** 单字符/单行读入；**`fgets` 是 `gets` 的安全替代**（永远传 sizeof）。
19. **`fflush(stream)` 强制 flush output buffer**。**传 NULL = flush 所有 stream**。**last op 是 input 时调 fflush = UB**。
20. **Random access 工具**：
    - `ftell` / `fseek` 用 `long int` 表示偏移——**大文件 (>2GB) 不够**
    - `fgetpos` / `fsetpos` 用 `fpos_t`（任意大偏移 + mbstate 信息）
    - `rewind(fp)` = `fseek(fp, 0L, SEEK_SET) + clearerr(fp)`
21. **write + read 必须中间 `fflush` / `fseek` / `fsetpos` / `rewind`**——否则 UB。
22. **append mode 不可靠**：很多系统不维护 file position indicator；写时强制跳到末尾。**POSIX/Windows 有不依赖 position 的 API**——`pread`/`pwrite`。
23. **`remove()` vs POSIX `unlink()`**：`unlink` 在 POSIX 语义更清晰（只移 directory entry，文件内容当 link count = 0 才删）。`remove` 跨平台可能不一致。
24. **temporary files 的安全陷阱**：
    - `tmpfile()` (C 标准) / `tmpnam()` (C 标准，**不安全** — TOCTOU) / `mkstemp()` (POSIX，**最安全**)
    - Linux：`/tmp`、`/var/tmp`、`$XDG_RUNTIME_DIR=/run/user/$uid`
    - Windows：`%USERPROFILE%\AppData\Local\Temp`、`C:\Windows\Temp`
    - **建议**：用 `mkstemp` 或**直接用 socket / shared memory**——临时目录是 IPC 历史遗留
25. **`fscanf(stream, format, ...)` 读格式化输入**：
    - 转换说明符 `%d %*9s %*[ \t] %99[^\n]` 形式
    - `*` = assignment-suppressing（不存）
    - 整数 = 最大字段宽度
    - `[^\n]` scanset = 读到 \n 之前所有字符（行读取惯用法）
26. **`fread` / `fwrite` 二进制流**：按 `size * nmemb` 读写；**返回实际写入元素数，不是字节数**（必须检查 `== nmemb`）。
27. **二进制文件跨平台的 endianness 陷阱**：
    - big-endian：MSB 在前（网络协议 IP/TCP/UDP）
    - little-endian：LSB 在前（Intel/AMD）
    - ARM/POWER 可切换
    - **解决**：永远存一个固定 endianness 或加 header 标识

---

# 二、核心 Takeaways

### Takeaway 1: `putc` 是 macro + 多次求值 side effect = CERT FIO41-C 警告

- **是什么**：`putc(c, fp)` 通常展开为 `*fp->_ptr++ = c;` 之类——**`fp` 可能 evaluate 多次**。
- **为什么重要**：`putc(*p++, fp)` 是 UB——`*p++` 副作用多次发生。
- **解决什么问题**：永远用 `fputc`（函数，无副作用陷阱）；或确保 stream 参数是简单标识符。
- **适用场景**：所有 putc/getc 调用。

### Takeaway 2: `fgets(buf, sizeof(buf), fp)` 是 `gets` 的现代替代

- **是什么**：`fgets` 显式 size 参数 + 保留 newline（与 `gets` 不同）。
- **为什么重要**：作者反复强调——**永远不要 gets**。
- **解决什么问题**：所有 stdin 行读取场景的安全替代。
- **适用场景**：替代 gets；CLI 工具的 prompt 输入；配置文件解析。

### Takeaway 3: `write + read` 之间不 `fflush` / `fseek` = 缓冲状态错误

- **是什么**：`fputs(s, fp); fgets(buf, n, fp);` 不中间调 fseek/fflush 是 UB——可能读到旧 buffer。
- **为什么重要**：现代 stdio 实现多用 unified buffer，read/write 不能交错。
- **解决什么问题**：始终在 read/write 切换前 `fflush(fp)` 或 `fseek(fp, 0, SEEK_CUR)`。
- **适用场景**：所有同时读写的流。

### Takeaway 4: `fseek`/`ftell` 用 `long int`——大文件不够

- **是什么**：`ftell` 返回 `long int`，最大 2GB (32-bit long) 或 8EB (64-bit)。**32-bit 系统上超过 2GB 的文件 offset 截断**。
- **为什么重要**：现代数据库文件、SSD 镜像、AI 模型都是 GB~TB 级。
- **解决什么问题**：用 `fgetpos`/`fsetpos` 用 `fpos_t`（任意大偏移 + mbstate）。
- **适用场景**：所有 >2GB 文件操作。

### Takeaway 5: `append` mode + `fseek` = 不可预测

- **是什么**：`fopen("file", "a")` 后 `fseek` 不一定生效——写时强制跳末尾。
- **为什么重要**：不同实现行为不同——portable 代码应假设 append 写永远末尾追加。
- **解决什么问题**：append 写不要用 fseek；要 absolute write 用 `O_APPEND` + `pwrite(fd, buf, n, offset)`。
- **适用场景**：log 文件、append-only 数据库。

### Takeenary 6: `tmpnam()` 是已知的不安全 API——`mkstemp()` 是 POSIX 安全替代

- **是什么**：`tmpnam()` 返回临时文件名（**但不创建文件**）→ TOCTOU 窗口攻击。
- **为什么重要**：1990s 经典安全 bug。**Linux glibc 现已 deprecated `tmpnam`**。
- **解决什么问题**：用 `mkstemp(template)`——原子创建文件 + 返回 fd。
- **适用场景**：所有 temp file 创建；或者直接放弃 temp file 用 socket/shm。

### Takeaway 7: `fread` / `fwrite` 返回"元素数"——不是字节数

- **是什么**：`fread(buf, 16, 4, fp)` 想读 64 字节，返回 4 = 成功（4 元素）；返回 2 = 只读 32 字节。
- **为什么重要**：**partial write 看起来对**——但实际只写一部分就网络断了。
- **解决什么问题**：必须 `== nmemb` 才算成功；或在循环里重试。
- **适用场景**：网络协议、序列化、长 binary record。

### Takeaway 8: binary I/O 跨平台 = 必须显式处理 endianness

- **是什么**：x86 little-endian、网络协议 big-endian；ARM 可切换。`fwrite(&x, sizeof(x), 1, fp)` 在不同架构产生不同字节序。
- **为什么重要**：跨平台 binary file 不能直接读写——必须 `htonl`/`ntohl` 或固定 endianness 格式。
- **解决什么问题**：① 全用 text format（JSON、protobuf、CBOR）；② 固定 big-endian + `htonl`；③ header 标识 endianness + 转换。
- **适用场景**：网络协议、跨平台存档、AI 模型持久化。

---

# 三、工程实践视角

### 嵌入式开发

- **MCU stdio 用得很窄**：通常 UART printf 重写 `_write()` syscall；很少用 file I/O（裸机没 fs）。
- **`stdin/stdout/stderr` 在裸机不存在**——`printf` 通过 ITM/SWO/UART 实现。
- **`FILE *` 在嵌入式 = 不存在**——直接用寄存器 + DMA。
- **buffering 在嵌入式慎用**：实时控制环禁 buffer（unpredictable latency）。
- **POSIX `open`/`read`/`write` 在嵌入式 Linux（Buildroot / Yocto）上可用**——但 device node 是 `O_RDONLY` 读 `/dev/i2c-1` 等。
- **endianness 在嵌入式 = 大端**：很多 MCU 默认 big-endian（network byte order）；x86 host 必须 swap。

### Linux 系统开发

- **`fopen` + buffering 是默认**——`setvbuf(fp, NULL, _IONBF, 0)` 可关 buffer。
- **`fopen` 默认走 `open` + 用户态 buffer**——`fileno` 拿到底层 fd。
- **`open()` + `O_DIRECT`** 绕过 page cache——数据库用（性能 / 一致性）。
- **`pwrite(fd, buf, n, offset)` 是 thread-safe write at offset**——不依赖 file position（解决 append mode 问题）。
- **`pread` 同理**。**现代数据库都是 `pread`/`pwrite` + `O_DIRECT`**。
- **kernel 里 I/O**：用 `struct file_operations` / `vfs_read` / `vfs_write`，**不用 stdio**。
- **systemd / daemon 进程的 stderr = journal**——所有错误日志走 stderr（unbuffered 保证立即看到）。

### 机器人软件（ROS / ROS2）

- **ROS 节点 I/O**：`rclcpp` 用 socket / DDS；**不直接用 `fopen`**。但 ROS bag / 参数文件用 file I/O。
- **ROS2 launch files** = XML/YAML——配置文件解析可用 `fscanf`，但更推荐 `yaml-cpp` / `toml++`。
- **传感器数据 log** = binary record（点云、IMU）→ 必须考虑 endianness + magic header。
- **micro-ROS 嵌入式**：通常用 `fwrite` 把 record 写到 SD 卡；fclose 必检查。

### 汽车电子软件（AUTOSAR / ISO 26262）

- **AUTOSAR 限制使用 `fopen`/`fread`/`fwrite` 类的 high-level I/O**——推荐 AUTOSAR `Rte` / `Com` 抽象。
- **NV memory 写入（数据持久化）**：用 AUTOSAR `NvM` 模块，**不是 fopen**——它保证 wear-leveling + checksum + atomic write。
- **Log：**AUTOSAR `DCM`/`DEM` 模块——格式固定，不允许 user-defined log path。
- **`fflush(NULL)` 在汽车代码禁用**——必须显式 flush 单个 stream，避免 cascade failure 隐藏错误。
- **binary record 安全要求**：CRC32 / SHA256 校验 → endianness 必须 explicit。

### 初级 vs 高级工程师的关注差

| 维度 | 初级 | 高级 |
|---|---|---|
| `gets` | 用 `gets(buf)` | **永远不**——`fgets(buf, sizeof(buf), stdin)` |
| `putc` + side effect | `putc(*p++, fp)` | **永远 `fputc`** |
| `fseek` 在大文件 | 直接用 | `fgetpos`/`fsetpos` |
| `fopen` 失败 | 不查 NULL | 立刻 `if (!fp)` |
| `fclose` 返回值 | 忽略 | 检查 + 错误时 abort() |
| `fflush(NULL)` | 当全部 flush | **明确单 stream flush** |
| `tmpnam` | 信任返回名 | 用 `mkstemp` 或放弃 temp file |
| binary I/O endianness | 跨平台假设一致 | 显式 `htonl` 或 text format |

---

# 四、AI 时代视角

### 这个知识今天是否仍然重要？

**重要性级别：高，但内容稳定**。

- C 标准 I/O 30 年没大变化；POSIX I/O 也是稳定 ABI。
- **AI 经常写错的**：
  - 用 `gets`（即使 C11 已删）
  - `putc(*p++, fp)` side effect
  - 大文件用 `ftell`（不查 long 范围）
  - binary file 跨平台不处理 endianness
  - `tmpnam` 不知不安全

### AI 能帮助完成什么

- ✅ 自动把 `gets` 替换 `fgets`
- ✅ 给 `fread`/`fwrite` 加返回检查
- ✅ 把 `fseek`/`ftell` 改 `fgetpos`/`fsetpos`（大文件）
- ✅ 写跨平台 binary I/O 的 endianness wrapper
- ✅ 生成 `mkstemp` 模板替换 `tmpnam`

### AI 无法替代什么

- ❌ **业务上选择 file format**（binary vs text vs protobuf）
- ❌ **跨平台二进制兼容性的真实测试**——需要在两台机器实测
- ❌ **real-time 系统的 buffer 策略**——需要 latency 测量
- ❌ **AUTOSAR NV memory 的具体配置**——需要 OEM 规范

### 工程师必须掌握的核心能力

1. **buffering 三种模式** + `setvbuf` 调整
2. **正确使用 `fopen` mode 字符串**（含 C11 `x`）
3. **`fread`/`fwrite` 返回值检查 == nmemb**
4. **`fflush` 在 read/write 切换前**
5. **`fgetpos`/`fsetpos` 用于 >2GB 文件**
6. **binary file 必须显式 endianness 处理**

---

# 五、实践行动项

### 行动 1: 演示 `putc` 的 side effect 陷阱
```bash
cat > /tmp/putc_sideeffect.c <<'EOF'
#include <stdio.h>
int main(void) {
    FILE *fp = fopen("/tmp/out.txt", "w");
    char arr[] = "ABC";
    char *p = arr;
    // 危险：putc 是 macro，会多次求值 *p++
    // 安全：用 fputc（函数，只求值一次）
    fputc(*p++, fp);
    fputc(*p++, fp);
    fputc(*p++, fp);
    fclose(fp);
    printf("wrote: %s\n", "ABC (via fputc)");
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/putc /tmp/putc_sideeffect.c && /tmp/putc
cat /tmp/out.txt
```
**预期**：用 fputc 正确；改为 putc 会触发 CERT FIO41-C。

### 行动 2: 演示 `fseek`/`ftell` 在大文件上的限制
```bash
cat > /tmp/fseek_big.c <<'EOF'
#include <stdio.h>
#include <stdint.h>
#include <limits.h>
int main(void) {
    printf("sizeof(long) = %zu bytes\n", sizeof(long));
    printf("Max LONG_MAX offset = %ld\n", LONG_MAX);
    printf("If your file > %ld bytes, use fgetpos/fsetpos instead.\n", LONG_MAX);
    return 0;
}
EOF
cc -std=c17 -o /tmp/fseek_big /tmp/fseek_big.c && /tmp/fseek_big
```
**预期**：64-bit 系统上 LONG_MAX = 8EB 没问题；32-bit 上是 2GB——嵌入式 / 32-bit MCU 必须用 `fgetpos`。

### 行动 3: 演示 `tmpnam` 的不安全（用 strace 看）
```bash
cat > /tmp/tmpnam_test.c <<'EOF'
#include <stdio.h>
#include <unistd.h>
int main(void) {
    char name[L_tmpnam];
    if (tmpnam(name)) printf("tmpnam returned: %s\n", name);
    // 注意：tmpnam 只返回名字，不创建文件；这是 TOCTOU 窗口
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/tmpnam_test /tmp/tmpnam_test.c && /tmp/tmpnam_test
```
**预期**：打印一个临时文件名（**没有创建文件**）——这是攻击窗口。**安全替代**：`mkstemp(template)`。

### 行动 4: 演示 binary I/O 的 endianness
```bash
cat > /tmp/endian.c <<'EOF'
#include <stdio.h>
int main(void) {
    unsigned int x = 0x12345678;
    FILE *fp = fopen("/tmp/binary.bin", "wb");
    fwrite(&x, sizeof(x), 1, fp);
    fclose(fp);
    // 读回来，看字节顺序
    fp = fopen("/tmp/binary.bin", "rb");
    unsigned char buf[4];
    fread(buf, 1, 4, fp);
    fclose(fp);
    printf("0x12345678 stored as: %02x %02x %02x %02x\n",
           buf[0], buf[1], buf[2], buf[3]);
    // x86 是 little-endian，所以是 78 56 34 12
    // 网络协议用 big-endian，所以必须 htonl/ntohl 转换
    return 0;
}
EOF
cc -std=c17 -o /tmp/endian /tmp/endian.c && /tmp/endian
```
**预期**：打印 `78 56 34 12`（little-endian）。**ARM/POWER 可能不同**——直接说明跨平台 binary I/O 不可移植。

### 行动 5: 演示 `fgets` 替代 `gets` + check NULL
```bash
cat > /tmp/safe_input.c <<'EOF'
#include <stdio.h>
#include <string.h>
int main(void) {
    char buf[64];
    printf("Enter something (max 63 chars): ");
    if (fgets(buf, sizeof(buf), stdin) == NULL) {
        puts("EOF or read error");
        return 1;
    }
    // fgets 保留 newline；通常需要去掉
    buf[strcspn(buf, "\n")] = '\0';
    printf("You typed: '%s'\n", buf);
    return 0;
}
EOF
echo "hello" | cc -std=c17 -Wall -Wextra -o /tmp/safe /tmp/safe_input.c -include string.h && echo "hello world" | /tmp/safe
```
**预期**：打印 "You typed: 'hello world'"——`fgets` 安全且保留 buffer bound。

---

# 六、值得深入思考的问题

### Q1: `FILE *` 抽象 vs POSIX `fd` 抽象——为什么 C 同时保留两种？

**FILE\***：buffered、可移植、不暴露 OS 细节。
**fd**：unbuffered、可直接控制 syscall（pread/pwrite/fcntl/ioctl）。
**问题**：为什么不统一成一种？Java 只有 `InputStream`/`OutputStream`、Rust 只有 `File`。**C 的双层抽象是历史包袱还是设计灵活性？**

### Q2: `fseek`/`ftell` 用 `long int`——为什么 30 年不改？

POSIX 大文件扩展（LFS）已用 `off_t`（off64_t）。C 标准仍要求 `long`。**问题**：C 标准应该统一到 `int64_t` 吗？为什么一直没有？**这反映了"标准委员会 vs 编译器实现"的什么张力？**

### Q3: `tmpnam()` 自 1990s 就被认为不安全——为什么 C 标准仍保留？

POSIX 有 `mkstemp`（更安全）。**C 标准为什么不直接 deprecate `tmpnam`？** 是因为兼容老代码，还是标准委员会不想"强制"实现特定函数？

### Q4: `append` mode 的不可靠行为——为什么 C 标准不强制要求？

C 标准说 "may set position indicator beyond last written"——**问题**：append 的语义应该是什么？POSIX `O_APPEND` 是 atomic seek-to-end-then-write，**C 标准为什么没跟进？** 这种 POSIX-only 行为有没有标准化的可能？

### Q5: binary I/O 必须处理 endianness——为什么 C 标准不提供 `fwrite` 的 endianness-aware 版本？

比如 `fwrite_be` 写 big-endian、`fwrite_le` 写 little-endian。**问题**：标准化 endianness-aware I/O 会减少多少 CVE？为什么标准委员会不做？**是不是**因为网络字节序是 POSIX 范畴，不是 C 标准范畴？

---

*下一章预告*: **Chapter 9 — Preprocessor** —— file inclusion / conditional inclusion / header guard / `#define` macro（object-like / function-like）/ `#` 和 `##` 运算符 / 宏的副作用陷阱（参数多次求值）/ 可变参数宏 / `_Generic` type-generic macro / 预定义宏（`__FILE__` / `__LINE__` / `__func__`）/ include guard 与 `#pragma once`。这是 C 的"元编程"——也是 C 程序员最容易写出"看起来对、实际 UB"代码的领域。