# 第 7 章 · Characters and Strings

> 来源: *Effective C* (Seacord, 2020) — Chapter 7, pp. 119–145
> 笔记日期: 2026-08-27

---

# 一、章节概述

1. **字符编码的演化史**：从 7-bit ASCII (1968) → Extended ASCII → ISO 8859-1 → Shift-JIS → EBCDIC → Unicode (1991)。**ASCII 0x00-0x7F 是 Unicode U+0000-U+007F 的子集**。
2. **Unicode 编码空间**：U+0000 to U+10FFFF（21-bit），17 个 plane（每 plane 64K），基本多语言平面（BMP）= plane 0。**UTF-8/16/32** 是三种 Unicode 转换格式。
3. **UTF-8 是 POSIX 主流**：1~4 字节变长，**与 ASCII 兼容**（0x00-0x7F 区间字节相同）。Windows UTF-16 是变长（1~2 个 16-bit code unit）；UTF-32 是定长（4 字节），**索引 O(1)** 但空间 4 倍。
4. **BMP 之外的字符** = supplementary characters，用 **surrogate pair**（high U+D800-DBFF + low U+DC00-DFFF）编码。
5. **Source charset vs execution charset**：C 实现定义了两种——源代码用什么编码写、字符串字面量用什么编码编译。**Java 直接规定编码**；C 不规定——这是历史包袱。
6. **`char` 是 signed 还是 unsigned 是 implementation-defined**——可移植代码不能假设。**永远用 `char` 表示字符**，不用 `char` 表示小整数。
7. **`char c = 'ÿ'; if (c == EOF)` 的陷阱**：`char` 是 signed 时 `ÿ` (0xFF) sign-extend 到 `int` = -1 = EOF；**`isdigit((unsigned char)c)` 是正确写法**。
8. **`wchar_t` 是大字符集类型**：Linux 32-bit、Windows 16-bit。**16-bit 不够代表所有 Unicode**——所以 Windows `wchar_t` 无法满足 `__STDC_ISO_10646__`（除 Unicode 3.1 前）。
9. **`char16_t` / `char32_t`（C11 `<uchar.h>`）**：专门 UTF-16 / UTF-32 数据类型；`__STDC_UTF_16__` / `__STDC_UTF_32__` 环境宏标识。**VS 不定义这两个宏**。
10. **字符常量是 `int` 而不是 `char`**（K&R 历史）：`'a'` 类型是 `int`——和 C++ 不。**前缀 `L` `u` `U` 分别对应 `wchar_t` `char16_t` `char32_t`**；C2x 加 `u8`。
11. **Escape sequence 八进制陷阱**：**`'\10'` 是 backspace（8），不是字符 `'1'` + `'0'`**。`'\x8'` 是 hex 8。同一个字符三种写法：'\b' / '\10' / '\x8'。
12. **`'ab'` 多字符常量 implementation-defined**——通常实现成 int = 'a'<<8 | 'b'，但**不要用**。
13. **GCC 字符集 flag**：`fexec-charset=` / `fwide-exec-charset=` / `finput-charset=`。**Clang 只允许 UTF-8**——其他拒绝。
14. **Windows dual entry point**：`main` (narrow, ANSI code page) vs `wmain` (wide, UTF-16)。**argv encoding 由 shell 决定**——Win process 启动时是 UTF-16，shell 转 codepage 给你。
15. **Win32 A/W 后缀**：`MessageBoxA`/`MessageBoxW`、`CreateWindowExA`/`CreateWindowExW`。**显式选 A 或 W**——不要靠自动转换。
16. **C 标准库字符转换函数**：单字符 `mbtowc`/`wctomb`/`mbrtowc`/`wcrtomb`；批量 `mbstowcs`/`wcstombs`/`mbsrtowcs`/`wcsrtombs`；C11 加 `mbrtoc16`/`c16rtomb`/`mbrtoc32`/`c32rtomb`。
17. **`mbstate_t` 是 restartable 转换的状态对象**——保存中间状态以便跨调用恢复。**nonrestartable 版本**内部存状态（不线程安全）。
18. **libiconv 是 GNU 跨平台转换库**——`iconv_open` + `iconv`。Windows 有 `MultiByteToWideChar` / `WideCharToMultiByte`。
19. **C 字符串 = 数组**：`char[]`（narrow）或 `wchar_t[]`（wide）。**没有 primitive string 类型**——`size` ≠ `length`：size 是 backing array 字节数，length 是首个 null 之前的 code unit 数。
20. **字符串字面量是只读**：**修改 string literal = STR30-C 违规**——可能被合并到只读段或多 literal 共享存储。
21. **字符串字面量前缀**：`"ABC"` (char) / `L"ABC"` (wchar_t) / `u8"ABC"` (UTF-8) / `u"ABC"` (char16_t) / `U"ABC"` (char32_t)。C2x 起 `u8""` 强制 UTF-8。
22. **`const char s[4] = "abcd"` 的陷阱**：s 是 4 字节数组但 literal 是 5 字节（"abcd\0"）——**`s` 没 null 终止**！编译器不报错（设计如此）。**`const char s[] = S_INIT`（省略长度）永远安全**。
23. **`<string.h>` 函数不安全（buffer overflow 高发）**：strcpy/strcat/strncpy/strncat/strlen——**假设 caller 已校验**。
24. **C11 Annex K bounds-checking interfaces**：`strcpy_s`/`strcat_s`/`strncpy_s`/`strncat_s`/`gets_s`——**显式 size 参数**。**Microsoft 是发起者但不完全合规**（VS 用旧 API）。**Linux/macOS 默认不实现 Annex K**。
25. **`gets` 是反例代表**：C99 deprecated、C11 删除。**没有 size 参数 = 永远不要用**。**`gets_s` 是替代**——但需要 `__STDC_WANT_LIB_EXT1__`。
26. **`<string.h>` vs `<wchar.h>` 对应**：str→wcs（`strcpy`→`wcscpy`、`strlen`→`wcslen`）；mem→wmem（`memcpy`→`wmemcpy`）。**别混用**。
27. **string length 的三种"长度"**：bytes / code units / code points / **extended grapheme clusters**（用户感知字符）。**只有 bytes / code units 能用于分配存储**。
28. **`strlen` 的实现与陷阱**：`for (s=str; *s; ++s); return s-str;`——**没有 bound 检查**，传给未 null 终止的字符串 = UB；传 NULL = UB。
29. **`strncpy` 的陷阱**：**不保证 null 终止**！`strncpy(dest, src, n)` 当 `strlen(src) >= n` 时 dest 没终止符。**用 `strncpy_s` 或自己 `dest[n-1] = '\0'`**。
30. **`memcpy` vs `strcpy`**：strcpy 假设 src 是字符串、拷贝直到 null；memcpy 拷贝固定字节。**strcpy 用于字符串，memcpy 用于 raw memory**——不要互换。
31. **`strcat(strcat(strcpy(...)))` 嵌套的低效**：每次重新扫描——C2x 引入 `memccpy`（POSIX 已存在）返回末尾指针。
32. **POSIX `strdup`/`strndup`**：自动 malloc 返回新字符串——**caller 必须 free**。**Windows 用 `_strdup`**。
33. **`getenv` 返回的字符串可能被后续调用覆盖**——**`getenv` 后立刻 strdup 拷贝**是 idiom。
34. **Runtime constraints**：Annex K 函数违反约束时调用 handler（默认 abort，可设 `ignore_handler_s`）。**MSVC 用 `_set_invalid_parameter_handler` 替代**。

---

# 二、核心 Takeaways

### Takeaway 1: `char c = 'ÿ'; if (c == EOF)` 是隐藏的"字符==EOF"陷阱

- **是什么**：`char` 是 signed 时，`'ÿ'` (0xFF) sign-extend 到 `int` = -1 = EOF。**循环里检查 EOF 会永远 false**。
- **为什么重要**：K&R 时代 `getchar()` 返回 `int` 就是为了这个区分——`char` 永远装不下 EOF。
- **解决什么问题**：所有 `<ctype.h>` 调用前 cast：`isdigit((unsigned char)c)`。
- **适用场景**：所有读字符循环；所有 `<ctype.h>` 调用。

### Takeaway 2: UTF-8/32 的 O(1) 索引 vs UTF-16/UTF-8 的变长

- **是什么**：UTF-32 是定长 4 字节，第 N 个 code point 直接 `buf[N]`；UTF-8/16 变长需要扫描。
- **为什么重要**：固定长度 = 索引简单、内存大；变长 = 压缩、省空间、需要边界检测。
- **解决什么问题**：选择编码 = 平衡空间/性能/兼容性。Linux/macOS 选 UTF-8（兼容 ASCII）；Windows 历史选 UTF-16。
- **适用场景**：所有跨平台 C 代码、协议字段、序列化。

### Takeaway 3: `wchar_t` 跨平台宽度差异是隐式 bug 源

- **是什么**：Linux `wchar_t = 32-bit`、Windows `wchar_t = 16-bit`。`wcslen(L"中文")` 在 Windows = 2（surrogate 对）、在 Linux = 2（BMP code points）。
- **为什么重要**：跨平台代码用 `wchar_t` **几乎必出错**。
- **解决什么问题**：跨平台字符串用 `char` + UTF-8（POSIX 标准）；或 `char16_t`/`char32_t` 显式。
- **适用场景**：所有跨平台 C/C++ 代码。

### Takeaway 4: `const char s[4] = "abcd"` —— 数组太小导致无 null 终止

- **是什么**：literal "abcd\0" 是 5 字节，s 是 4 字节——**`s` 没 null 终止**。后续 `strlen(s)` / `strcpy(s, ...)` / `puts(s)` = 读越界 UB。
- **为什么重要**：编译器**不报错**（设计如此：允许初始化 char 数组不是 string）。
- **解决什么问题**：**省略数组长度** `const char s[] = S_INIT;` 永远安全；或用 `strncpy`/`memcpy` 显式管理。
- **适用场景**：所有用 `#define` 写字符串字面量的代码——GCC 会警告 `-Wstringop-truncation`。

### Takeaway 5: `strncpy` **不保证 null 终止**——比 strcpy 还危险

- **是什么**：`strncpy(dest, src, n)` 当 `strlen(src) >= n` 时 dest 没 null 字节。后续把它当字符串用 = UB。
- **为什么重要**：名字像 "safe strcpy" 但**完全不是**——CERT STR32-C 明确警告。
- **解决什么问题**：用 `strncpy_s`（C11 Annex K），或 `dest[n-1] = '\0'` 手动补 null，或 `snprintf(dest, n, "%s", src)`。
- **适用场景**：所有固定 buffer 拷贝场景。

### Takeaway 6: `gets` 已从 C11 删除——但遗留代码到处都是

- **是什么**：`gets(s)` 没有 size 参数，**无法防止 buffer overflow**——C99 deprecated，C11 删除。
- **为什么重要**：任何 GitHub 老仓库里几乎都能找到 `gets`——**永远要替换**。
- **解决什么问题**：用 `fgets(buf, sizeof(buf), stdin)`，**永远传 sizeof**，永远不用 gets。
- **适用场景**：所有 C 代码扫描；CVE 数据库里 `gets` 相关漏洞持续 20+ 年。

### Takeaway 7: Annex K `strcpy_s` 等安全函数——MSVC 起源但 Linux/macOS 不实现

- **是什么**：C11 Annex K 提供 `strcpy_s`/`strcat_s`/`strncpy_s`/`strncat_s`/`gets_s` 等显式带 size 参数版本。
- **为什么重要**：MSVC 是发起者（90 年代安全事件推动），但**Microsoft 不完全合规**（用旧 `_set_invalid_parameter_handler`）。**Linux/macOS 默认不实现**——`__STDC_WANT_LIB_EXT1__` 不工作。
- **解决什么问题**：跨平台安全字符串处理仍要自己写 wrapper；或依赖 POSIX `strdup`/`strndup`。
- **适用场景**：MSVC-only 项目用 `_s` 系列；跨平台项目自己写 size-aware 包装。

### Takeaway 8: string length 的四种"长度"——只有 bytes/code units 可用于分配

- **是什么**：`strlen`/`wcslen` 数 **code units**；**code points** 数需要 mbtowc 等；**extended grapheme clusters**（用户感知字符，如 emoji）数更复杂。
- **为什么重要**：`malloc(wcslen(s) + 1) * sizeof(wchar_t)` 在 Linux 是 OK；在 Windows 上 BMP 外字符需要 surrogate 对，长度计算可能错。
- **解决什么问题**：分配存储用 code units；截断字符串用 extended grapheme clusters（避免切坏 emoji）。
- **适用场景**：所有 i18n 字符串处理。

---

# 三、工程实践视角

### 嵌入式开发

- **嵌入式几乎没有 i18n 需求**——ASCII 够用。**但 escape sequence 八进制陷阱要小心**：`'\10'` 是 backspace，不是 `'1' '0'`。
- **`char` 永远是 unsigned 还是 signed 看 MCU 编译器手册**——`avr-gcc` 默认 unsigned；ARM GCC 默认 signed。**别假设**。
- **`wchar_t` 在嵌入式几乎不用**——资源不够。
- **`gets` 在 MCU 上常被误用**：很多 legacy 嵌入式代码用 `gets(uart_buf)`——**C11 删除后编译器报警告但代码还在**。
- **`strncpy` 是 fixed-buffer copy 标准**——但要手动加 `buf[n-1] = '\0'`。

### Linux 系统开发

- **Linux 默认 UTF-8**——POSIX locale 1。C 字符串字面量默认 UTF-8（执行字符集）。
- **跨平台字符处理**：**永远用 `char` + UTF-8**——不要用 `wchar_t`。libuv / libxml2 / glib 都用 UTF-8 `char*`。
- **`char16_t`/`char32_t` 在 Linux 几乎不用**——直接用 `char` UTF-8 + iconv 转。
- **`strdup`/`strndup` 是 POSIX 标准**——glibc 提供。
- **`getenv` 后立刻 `strdup`**——避免后续 `getenv` 覆盖。
- **C23 起 Annex K 被 deprecated**——标准社区承认它没成功。

### 机器人软件（ROS / ROS2）

- **ROS message 编码**：ROS1 默认 ASCII；ROS2 默认 UTF-8。`std_msgs/String` = `string data`——UTF-8。
- **ROS 跨语言桥接**：Python str ↔ C `char*` 都是 UTF-8——直接 `strdup(s)` 即可。
- **机器人 UI/语音**：可能涉及 emoji——必须 extended grapheme clusters 计数，不能用 `strlen`。
- **micro-ROS 嵌入式**：MCU 资源有限，**字符串字面量尽量 ASCII**，UTF-8 处理会增加代码大小。

### 汽车电子软件（AUTOSAR / ISO 26262）

- **AUTOSAR 编码规范禁用** `gets`、禁用 `strcpy`、禁用 `strcat`——只允许 size-bounded 版本（`strncpy`/`snprintf`），且要求 size 在编译期可验证。
- **`strncpy` 在汽车代码里常用**——但**必须配合 `buf[n-1] = '\0'`**（MISRA 规则）。
- **字符集**：车机/仪表盘可能需要本地化（中日韩）；**RTE 配置**统一 UTF-8。
- **OEM 自定义字符集**：某些 OEM 用专属 codepage——必须用 `iconv` 或 codepage-aware API。
- **AUTOSAR SWS_LogAndTrace**：所有 log API 是变参——但 format string 必须是 literal（防 format string 漏洞——ch01 已讲）。

### 初级 vs 高级工程师的关注差

| 维度 | 初级 | 高级 |
|---|---|---|
| `char` 假设 | 不知道 signed/unsigned 区别 | 知道 implementation-defined；永远 cast `(unsigned char)` |
| `gets` | 看到 `gets(s)` 就用 | **永远不**——fgets 替代 |
| `strncpy` | 当作"安全 strcpy" | 知道不保证 null，手动补 |
| 跨平台字符串 | 用 `wchar_t` | 用 `char` + UTF-8 + iconv |
| `char` 数组大小 | `const char s[4] = "abcd"` 不察觉 | `const char s[] = S_INIT` 永远安全 |
| `wchar_t` 跨平台 | 不觉差异 | 知道 Win/Linux 宽度差，避免 |
| `strdup` 后 | 忘 free | 立刻 free 或 RAII |
| escape `\10` | 觉得是 '1'+'0' | 知道是 backspace |

---

# 四、AI 时代视角

### 这个知识今天是否仍然重要？

**重要性级别：极高——i18n 是 AI 写代码最容易出错的领域**。

- **AI 经常写错**：
  - 把 `gets` 直接生成（即使 C11 已删）
  - 用 `wchar_t` 当 cross-platform string
  - 假设 `char` 总是 unsigned 或总是 signed
  - `strncpy` 后不补 null
  - `getenv` 后不用 `strdup`
  - 字符串字面量假设 ASCII

### AI 能帮助完成什么

- ✅ 把 `gets(s)` 自动改 `fgets(s, sizeof(s), stdin)`
- ✅ 把 `strncpy` 后加 `buf[n-1] = '\0'`
- ✅ 写 UTF-8 / UTF-16 转换的 wrapper
- ✅ 解释 `'\10'` vs `'\b'` vs `'\x8'` 等价
- ✅ 检查 MSVC vs GCC vs Clang 的字符集差异

### AI 无法替代什么

- ❌ **决定公司/产品的 target encoding**——涉及整个技术栈
- ❌ **跨平台字符串协议设计**——涉及 wire format 兼容性
- ❌ **AUTOSAR / DO-178C 字符集合规**——需要 cert 工具
- ❌ **extended grapheme cluster 的复杂规则**——需要 Unicode 标准查阅

### 工程师必须掌握的核心能力

1. **区分 source charset / execution charset**
2. **理解 UTF-8/16/32 的 trade-off**
3. **永远不会用 `gets`**
4. **`strncpy` 后永远补 null**（或用 `_s`）
5. **跨平台字符串用 UTF-8 `char*`**，不要 `wchar_t`

---

# 五、实践行动项

### 行动 1: 验证 `char` 在你机器上是 signed 还是 unsigned
```bash
cat > /tmp/charsign.c <<'EOF'
#include <stdio.h>
#include <limits.h>
int main(void) {
    char c = (char)0xFF;
    if (c == EOF) puts("char is signed and 0xFF == EOF (UB trap!)");
    else puts("char is unsigned (safer)");
    printf("CHAR_MIN=%d, CHAR_MAX=%d\n", CHAR_MIN, CHAR_MAX);
    return 0;
}
EOF
cc -std=c17 -o /tmp/charsign /tmp/charsign.c && /tmp/charsign
```
**预期**：在 x86 Linux GCC `char` 默认 signed — `0xFF` sign-extend = -1 = EOF（UB 比较成立但你看到陷阱）。

### 行动 2: 演示 `strncpy` 不保证 null 终止
```bash
cat > /tmp/strncpy.c <<'EOF'
#include <stdio.h>
#include <string.h>
int main(void) {
    char buf[10];
    strncpy(buf, "hello world", sizeof(buf));
    printf("buf = '%s'\n", buf);   // UB! 可能读到 buffer 外的 garbage
    printf("strlen(buf) = %zu (可能 > 10)\n", strlen(buf));
    // 安全做法：
    strncpy(buf, "hello world", sizeof(buf) - 1);
    buf[sizeof(buf) - 1] = '\0';   // 手动补 null
    printf("safe buf = '%s'\n", buf);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/strncpy /tmp/strncpy.c && /tmp/strncpy
```
**预期**：第一段 strlen 可能 > 10（读到 null 之前的所有字节）；第二段是安全的。

### 行动 3: 演示 `const char s[4] = "abcd"` 的陷阱
```bash
cat > /tmp/strconst.c <<'EOF'
#include <stdio.h>
#include <string.h>
#define S_INIT "abcd"
int main(void) {
    const char s[4] = S_INIT;
    printf("sizeof(s) = %zu\n", sizeof(s));
    // strlen(s) 是 UB! 因为 s 没 null 终止
    // printf("strlen = %zu\n", strlen(s));   // 取消注释看 UB
    // 安全做法：
    const char safe[] = S_INIT;
    printf("safe sizeof = %zu, strlen = %zu\n", sizeof(safe), strlen(safe));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -Wstringop-truncation -o /tmp/strconst /tmp/strconst.c 2>&1
/tmp/strconst
```
**预期**：GCC 警告 `-Wstringop-truncation`；safe 版本正常。

### 行动 4: 演示 escape sequence 等价
```bash
cat > /tmp/esc.c <<'EOF'
#include <stdio.h>
int main(void) {
    printf("'\\b' = %d (decimal)\n", '\b');
    printf("'\\10' = %d (octal = 8)\n", '\10');
    printf("'\\x8' = %d (hex)\n", '\x8');
    printf("Are they equal? %s\n", ('\b' == '\10' && '\10' == '\x8') ? "yes" : "no");
    // '\10' 是 octal！不是 '1'+'0'！
    printf("'\\10' as int = 0x%x\n", '\10');
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/esc /tmp/esc.c && /tmp/esc
```
**预期**：三个写法都 = 8（同一字符）。**注意**：`'\10'` 是 1 个字符，值 8（octal）。

### 行动 5: 演示 `wchar_t` 在 Linux 是 32-bit
```bash
cat > /tmp/wch.c <<'EOF'
#include <stdio.h>
#include <wchar.h>
int main(void) {
    printf("sizeof(wchar_t) = %zu bytes\n", sizeof(wchar_t));
    wchar_t w = L'中';   // BMP character
    printf("L'中' = 0x%x\n", (unsigned)w);
    printf("wcslen(L\"中文\") = %zu\n", wcslen(L"中文"));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/wch /tmp/wch.c && /tmp/wch
```
**预期**：Linux `sizeof(wchar_t) = 4`；L'中' = 0x4e2d；在 Windows 上 wchar_t 是 2 字节，行为不同。

---

# 六、值得深入思考的问题

### Q1: C 没有 primitive string——这是个**"**feature**"还是**bug**？

**支持无 primitive 的论点**：效率（直接字节访问）、ABI 稳定、灵活（用户可定义 `string_t`）。
**反对**：所有错误都是 C 字符串处理（CVE 占比巨大）。
**问题**：如果 C89 当时引入 `string_t`，今天的世界会怎样？**为什么 Java / Python 选择 primitive string 而 C 不？**

### Q2: `wchar_t` 在 Windows 16-bit / Linux 32-bit——这种差异是**设计失误**吗？

提示：C 标准允许 implementation-defined 宽度；POSIX 要求 32-bit。**问题**：C 标准为什么不强制 `wchar_t` 宽度？统一到 32-bit 会让 Windows 性能损失 50%（UTF-32 → UTF-16）——**这种"自由"是不是正确决策？**

### Q3: Annex K（C11 边界检查接口）失败的根本原因是什么？

**事实**：MS 发起、但 MS 不合规；Linux/macOS 不实现；C23 起 deprecated。
**问题**：如果当年 MS 把 `strcpy_s` 直接做进 ABI 而不是 TR 化、再开源，命运会不同吗？**Annex K 失败说明了"标准推动者必须真用它"的什么教训？**

### Q4: `'\10'` 是 backspace——**为什么 C 解析器用 octal 而不是 decimal 当 escape？**

提示：octal 3 位 max（`\777` = 511），decimal 解析会有歧义（`\123` 是 char(123) 还是字符串'1'+'2'+'3'？）。**问题**：如果当年改用 `\d{N}`（decimal）、`\o{N}`（octal）、`\x{N}`（hex）——是不是更易读？**为什么没改？**

### Q5: `gets` 在 C99 deprecated、C11 删除——**为什么 glibc 至今仍提供 `gets` 实现？**

**原因**：ABI 兼容 + 遗留链接器符号。**问题**：编译器警告 + 标准删除 = 仍可编译运行的"僵尸 API"——**这种"软删除"足够吗？**还是需要硬破坏（链接器直接拒绝）？

---

*下一章预告*: **Chapter 8 — Input/Output** —— Standard I/O Streams（`FILE*`、`stdin/stdout/stderr`）、text vs binary mode、buffering（full / line / unbuffered）、POSIX file descriptor（`open/read/write/close` vs `fopen/fread/fwrite/fclose`）、random access（`fseek`/`ftell`）、temporary files、`fprintf`/`sscanf` 的 format string、binary I/O 的 endianness 处理。这是所有"程序与外界通信"的基础。