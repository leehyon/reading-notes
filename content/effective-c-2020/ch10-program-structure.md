# 第 10 章 · Program Structure

> 来源: *Effective C* (Seacord, 2020) — Chapter 10, pp. 185–198
> 笔记日期: 2026-08-27

---

# 一、章节概述

1. **Componentization = 把大程序拆成可复用的小组件**：跨"接口边界"通信。原则：低耦合、高内聚。
2. **Cohesion（内聚）**：组件内元素的相关度。`<string.h>` 是高内聚（strcpy/strlen/strcat 全相关）；混 strlen + tan + thread 是低内聚。
3. **Coupling（耦合）**：组件间依赖度。**松耦合** = 头文件可单独 include；**紧耦合** = 必须按特定顺序 include。
4. **Code reuse 是基本工程伦理**：相同逻辑实现多次 = bug 源、size 膨胀、维护成本。**永远用 `strlen` 不自己写 for 循环**。
5. **设计接口在 general vs specific 之间 trade-off**：太具体 = 改需求时难复用；太泛 = 当前用例繁琐。
6. **Data abstraction = 公开接口与实现分离**：公开接口放 header；实现细节藏 .c 或 internal header。**改实现不改 header → 调用方代码不破**。
7. **Header 应 self-contained**：自己需要的 header 自己 include，不要让用户先 include。**leak implementation detail 是反模式**。
8. **同一组件多源**：network.h + `network.c` (Linux) + `network_win32.c` (Windows)。**platform-specific 实现用同名 symbol 不同 .c**。
10. **Opaque type（不透明类型）** = forward-declared incomplete struct：`typedef struct collection_type collection_type;` → 用户只见指针不见实现。**真正的封装**。
11. **Opaque type 范式**：
    - **外部 header**：`typedef struct foo foo;` + 函数声明（接收 `foo *`）
    - **内部 header**：完整定义 `struct foo { ... };`
    - 实现的 .c include 两个；用户只 include 外部
12. **Opaque type 的好处**：换底层数据结构（数组→链表→红黑树）不影响调用方代码。
13. **Executable 分类**：
    - **executable**（`a.out`、`foo.exe`）——可直接运行
    - **library**（`lib*.a` / `lib*.so`）——需链接
    - **driver**（kernel driver）
    - **firmware image**（裸机 ROM 烧录）
14. **Library 分类**：
    - **static library（archive）**：链接时**全量并入可执行文件**；优化好；可执行独立（无需运行时依赖）
    - **dynamic library（shared object）**：运行时按需加载；多进程共享；可独立升级但有版本兼容风险
15. **Static vs dynamic trade-off**：static 启动快、部署简单；dynamic 升级灵活、节省内存。**现代 OS 一般 dynamic 优势大于劣势**。
16. **Static library 文件约定**：Linux/macOS `lib<name>.a`，Windows `<name>.lib`。**`ar rcs` 创建 archive**。
17. **Dynamic library 文件约定**：Linux `lib<name>.so`，macOS `lib<name>.dylib`，Windows `<name>.dll`。**`-fPIC` 编译**。
18. **Linkage 三种**：
    - **external linkage**：跨 translation unit 同一 entity（默认函数/全局变量）
    - **internal linkage**：仅本 translation unit 可见（`static` 函数/全局变量）
    - **no linkage**：参数、块作用域变量、enum 常量
19. **`static` 在 file scope vs block scope 含义不同**：
    - **file scope static** = internal linkage（每个 TU 独立实体）
    - **block scope static** = static storage duration（程序启动到结束）+ no linkage
    - **常见面试题**："`static int i;` 在函数内 vs" "在文件 scope"
20. **Linkage 冲突 = UB**：CERT DCL36-C。同一标识符在多个 TU 声明不同 linkage = UB。
21. **`extern` 的含义**：声明而非定义；**"这个标识符在别处定义"**。函数声明默认带 extern 效果（implicitly）。
22. **链接规则**：
    - **public API 用 external linkage**（header 里声明，无 static）
    - **implementation 用 internal linkage**（.c 里用 static 私有）
23. **Program = Library + Driver**：一个静态库 + 一个 main 程序。
24. **Static library 构建流程**：
    ```bash
    cc -c isprime.c -o isprime.o        # 编译为 object
    ar rcs libPrimalityUtilities.a isprime.o   # 打 archive
    cc driver.o -L. -lPrimalityUtilities -o primetest   # 链接
    ```
25. **`ar rcs`**：`r`=replace、`c`=create archive、`s`=write index（`ranlib` 等价）。
26. **`-L<dir>` + `-l<name>`**：链接器搜索路径 + 库名（省略 `lib` 前缀和 `.a` 后缀，自动补）。
27. **`-c` flag**：只编译不链接，生成 object file。
28. **`-o` flag**：指定输出文件名（object、可执行、archive 都用）。

---

# 二、核心 Takeaways

### Takeaway 1: cohesion + coupling 是 C 项目结构的两大轴

- **是什么**：内聚 = 组件内元素相关度；耦合 = 组件间依赖度。**好设计 = 高内聚 + 低耦合**。
- **为什么重要**：低内聚头文件让用户 include 一堆不需要的东西；高耦合头文件强制 include 顺序。
- **解决什么问题**：好的 module 边界 = 可替换实现 + 可独立测试 + 可独立维护。
- **适用场景**：所有项目结构设计。

### Takeaway 2: opaque type 是 C 的"封装"——forward decl + 不全定义

- **是什么**：`typedef struct foo foo;` 只声明类型名 + struct 标签，不暴露成员。用户只能拿 `foo *`。
- **为什么重要**：用户代码**完全依赖不到** struct 内部成员 → 换实现零成本。
- **解决什么问题**：真正实现 ADT (AbstractDType)；用户被迫走 API 函数。
- **适用场景**：所有 library API（`FILE *` 就是经典 opaque type）。

### Takeaway 3: header 应 self-contained——需要的 header 自己 include

- **是什么**：你的 header 用了 `size_t` 就 `#include <stddef.h>`，不要假设用户先 include 了。
- **为什么重要**：user 一旦 include 顺序变了 = 编译错。**IWYU（include-what-you-use）原则**。
- **解决什么问题**：可预测的依赖；用户不需要记 include 顺序。
- **适用场景**：所有 public header。

### Takeaway 4: `static` 在 file scope 给 internal linkage，在 block scope 给 static storage duration

- **是什么**：
    ```c
    static int g;          // file scope: internal linkage (各 TU 独立 g)
    void f(void) {
        static int n = 0;  // block scope: static storage duration, no linkage
        ++n;
    }
    ```
- **为什么重要**：面试高频题；混淆会导致"我以为是全局变量但每个 .c 都有自己的"。
- **解决什么问题**：精确控制符号可见性 + 变量生命周期。
- **适用场景**：模块内 helper 函数（`static`）；模块级计数器（`static` 局部）。

### Takeaway 5: static library 启动快 + 可执行独立；dynamic 升级灵活

- **是什么**：static = 链接时全量并入 binary；dynamic = 运行时按需 load。
- **为什么重要**：autosar/汽车 ECU 用 static（可预测启动时间 + 无运行时依赖）；手机 app 用 dynamic（节省空间 + 后台升级）。
- **解决什么问题**：选型 = 部署模式 + 升级频率 + 安全策略。
- **适用场景**：lib 选择；Makefile 写法。

### Takeaway 6: public API 用 external linkage，implementation 用 `static`

- **是什么**：header 函数声明无 static = external；.c 里 helper 用 `static` = internal。
- **为什么重要**：**限制全局 namespace 污染**；防止 user 调用你的 helper；编译时优化更好（可见性更窄）。
- **解决什么问题**：好工程卫生；防止符号冲突。
- **适用场景**：所有 .c 文件。

### Takeaway 7: `ar rcs` 是构建 archive 的标准命令

- **是什么**：`r`=replace old、`c`=create archive、`s`=write object index。
- **为什么重要**：是 Linux/macOS 创建 static library 的标准工具。
- **解决什么问题**：无需 IDE，用 shell 就能组装项目。
- **适用场景**：所有 C 项目构建脚本；CI；嵌入式 Makefile。

### Takeaway 8: `cc -L<dir> -l<name>` 是链接外部库的标准写法

- **是什么**：`-L` 加搜索路径；`-l<name>` 加库名（自动加 `lib` 前缀和 `.a/.so` 后缀）。
- **为什么重要**：跨平台一致；`make` / `cmake` 都用这模式。
- **解决什么问题**：不需要写完整路径。
- **适用场景**：所有依赖外部库的构建命令。

---

# 三、工程实践视角

### 嵌入式开发

- **MCU BSP 几乎全部 static linking**：可执行 image 直接烧到 ROM，无 OS 加载 dynamic lib。
- **Opaque type 在嵌入式 = HAL 抽象层**：`typedef struct uart_dev uart_dev_t;` —— 用户只见 handle 不知寄存器布局。
- **Embedded linker script**：决定每个符号放哪个 section；`KEEP` 指令保留 entry 函数。
- **`-ffunction-sections -fdataf-sections + -Wl,--gc-sections`**：剔除未用代码（嵌入式 flash 紧张时）。
- **MISRA 要求 internal linkage 显式**——所有模块私有函数必须 `static`。
- **cross-compilation**：`CC=arm-none-eabi-gcc`，链接脚本由 BSP 提供，library path 用 sysroot。

### Linux 系统开发

- **pkg-config 是核心工具**：`pkg-config --libs libcurl` 自动给 `-lcurl -L/usr/lib`。
- **CMake 用 `add_library` + `target_link_libraries`**：构建 static/dynamic 都支持。
- **动态库版本管理**：soname（`libfoo.so.1`）+ realname（`libfoo.so.1.0.0`）+ devlink（`libfoo.so`）。
- **ABI 兼容性**：加字段 = break ABI → **必须改 major version**。
- **Linux kernel 全是 static**：没有 .so，单体内核 + modules（`.ko`）按需加载。

### 机器人软件（ROS / ROS2）

- **ROS package = 独立 CMake 项目**：每个 package 是独立组件，可被其他 package 通过 `find_package` 依赖。
- **ROS msg/srv 自动生成代码**：用 .msg → 生成 .h + .cpp（C 也有 .h），用户 include 即可。
- **micro-ROS 嵌入式**：纯 static linking，因为 MCU 没 dynamic loader。
- **ROS2 rclcpp** 大量用 opaque type：`rcl_node_t`、`rcl_publisher_t` 都 forward-declared。

### 汽车电子软件（AUTOSAR / ISO 26262）

- **AUTOSAR BSW 模块 = static library**：MCAL / CDD / COM / RTE 等都是 lib。
- **OEM 集成 = link BSW libs + 应用 object** = ECU firmware。
- **Opaque type 在 AUTOSAR 标配**：`Std_ReturnType`、`Dem_EventIdType` 等全是 handle。
- **MISRA + ISO 26262 要求 internal linkage 显式**：helper 函数必须 `static`。
- **Static linking 在 AUTOSAR**：deterministic 启动时间 + cert 流程简单。
- **避免 dynamic library**：动态加载 + cache miss + 启动时间不确定 = 不适合安全关键。

### 初级 vs 高级工程师的关注差

| 维度 | 初级 | 高级 |
|---|---|---|
| Header 内容 | 自己需要的靠用户 include | self-contained，IWYU 原则 |
| `static` 含义 | 不知道 file scope vs block scope | 精确区分 linkage vs storage |
| Opaque type | 不熟 | collection / handle / FD / FILE * 都是 |
| Library 选择 | 全部 static | static vs dynamic 按需选择 |
| 链接顺序 | `-L. -lfoo` 经常漏 | `pkg-config` / cmake 自动化 |
| Linkage 冲突 | 不知道 | UB，linker 静默选其一 |
| Module 边界 | 模糊 | 每个 .c 内部 helper 都 static |
| Build artifact | 一锅煮 | library + driver 分开构建 |

---

# 四、AI 时代视角

### 这个知识今天是否仍然重要？

**重要性级别：高——项目结构是长期维护的关键**。

- 现代 C 项目几乎所有都自动 CMake/Makefile 化，但**理解模块边界、linkage、opaque type**仍是不可替代的工程判断力。
- **AI 经常写错的**：
  - Header 不 self-contained（依赖用户先 include 别的）
  - Opaque type 不完整（既没 forward decl 也没 internal struct）
  - Library 选择错（dynamic 用在不该用的地方）
  - Linkage 冲突（不同 TU 同一标识符不同声明）

### AI 能帮助完成什么

- ✅ 生成 CMakeLists.txt / Makefile
- ✅ 自动添加 `static` 到模块私有 helper
- ✅ 把 header 改成 self-contained（IWYU）
- ✅ 生成 opaque type 的双 header 结构
- ✅ 解释 `ar rcs` / `-L -l` 的工作方式

### AI 无法替代什么

- ❌ **决定模块边界**——涉及长期演化、团队协作
- ❌ **选 static vs dynamic**——涉及部署、安全、升级策略
- ❌ **设计 opaque type API**——需要用例驱动
- ❌ **Linkage 冲突排查**——需要看实际 link map

### 工程师必须掌握的核心能力

1. **cohesion + coupling 思维**
2. **opaque type 双 header 模式**
3. **三种 linkage 区别**
4. **`static` 多义性区分**
5. **`ar rcs` 创建 archive**
6. **`-L -l` 链接语法**

---

# 五、实践行动项

### 行动 1: 演示 opaque type 双 header 模式
```bash
mkdir -p /tmp/opaque_demo && cd /tmp/opaque_demo
cat > collection.h <<'EOF'
#ifndef COLLECTION_H
#define COLLECTION_H
#include <stddef.h>
typedef struct collection_type collection_type;
extern int create_collection(collection_type **out);
extern void destroy_collection(collection_type *col);
extern int add_to_collection(collection_type *col, int value);
extern size_t count_collection(const collection_type *col);
#endif
EOF
cat > collection_internal.h <<'EOF'
#ifndef COLLECTION_INTERNAL_H
#define COLLECTION_INTERNAL_H
#include "collection.h"
struct collection_type { size_t count; int *data; size_t cap; };
#endif
EOF
cat > collection.c <<'EOF'
#include "collection_internal.h"
#include <stdlib.h>
int create_collection(collection_type **out) {
    collection_type *c = calloc(1, sizeof(*c));
    if (!c) return -1;
    *out = c; return 0;
}
void destroy_collection(collection_type *col) { free(col->data); free(col); }
int add_to_collection(collection_type *col, int value) {
    if (col->count >= col->cap) {
        size_t newcap = col->cap ? col->cap * 2 : 4;
        int *p = realloc(col->data, newcap * sizeof(int));
        if (!p) return -1;
        col->data = p; col->cap = newcap;
    }
    col->data[col->count++] = value;
    return 0;
}
size_t count_collection(const collection_type *col) { return col->count; }
EOF
cat > main.c <<'EOF'
#include <stdio.h>
#include "collection.h"   // 注意：只 include 外部 header
int main(void) {
    collection_type *c;
    create_collection(&c);
    add_to_collection(c, 10);
    add_to_collection(c, 20);
    printf("count = %zu\n", count_collection(c));
    destroy_collection(c);
    return 0;
}
EOF
cc -std=c17 -Wall -I. main.c collection.c -o /tmp/opaque_demo/app && /tmp/opaque_demo/app
```
**预期**：编译成功；main.c **没有** access `c->count`（opaque），所有访问走 API。

### 行动 2: 演示 `static` 在 file scope vs block scope 的差异
```bash
cat > /tmp/static_diff.c <<'EOF'
#include <stdio.h>
static int g = 100;        // file scope: internal linkage, each TU has own g
void f(void) {
    static int counter = 0;  // block scope: static storage, no linkage
    ++counter;
    printf("counter = %d\n", counter);
    printf("g = %d\n", g);
}
int main(void) {
    f(); f(); f();
    return 0;
}
EOF
cc -std=c17 -o /tmp/static_diff /tmp/static_diff.c && /tmp/static_diff
```
**预期**：counter 从 1 → 2 → 3（static storage 跨调用保留）；g 是 file scope static。

### 行动 3: 演示 build static library 完整流程
```bash
mkdir -p /tmp/staticlib_demo/bin
cat > /tmp/staticlib_demo/mathutil.c <<'EOF'
#include "mathutil.h"
int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }
EOF
cat > /tmp/staticlib_demo/mathutil.h <<'EOF'
#ifndef MATHUTIL_H
#define MATHUTIL_H
int add(int a, int b);
int mul(int a, int b);
#endif
EOF
cat > /tmp/staticlib_demo/main.c <<'EOF'
#include <stdio.h>
#include "mathutil.h"
int main(void) { printf("3+5=%d, 3*5=%d\n", add(3,5), mul(3,5)); return 0; }
EOF
cd /tmp/staticlib_demo
cc -std=c17 -c mathutil.c -o bin/mathutil.o
ar rcs bin/libmathutil.a bin/mathutil.o
cc -std=c17 main.c -Lbin -lmathutil -o bin/app
./bin/app
```
**预期**：打印 `3+5=8, 3*5=15`。**完整自包含 build 流程**。

### 行动 4: 演示 link map 查看 linkage
```bash
cat > /tmp/linksrc.c <<'EOF'
extern int public_func(void);
static int private_helper(void) { return 42; }
int public_func(void) { return private_helper(); }
EOF
cc -std=c17 -c /tmp/linksrc.c -o /tmp/linksrc.o
nm /tmp/linksrc.o | grep -E "public_func|private_helper"
```
**预期**：`public_func` 是 global（T）；`private_helper` 是 local（t）——`static` 给 internal linkage。

### 行动 5: 用 IWYU 原则重写 header（self-contained）
```bash
cat > /tmp/bad_header.h <<'EOF'
// 不 self-contained: 依赖用户先 include <stddef.h>
size_t my_strlen(const char *s);
EOF
cat > /tmp/good_header.h <<'EOF'
// self-contained: 显式 include 依赖
#ifndef GOOD_HEADER_H
#define GOOD_HEADER_H
#include <stddef.h>
size_t my_strlen(const char *s);
#endif
EOF
echo "bad 头需要用户预先 include，good 头自带 — 工程规范必查"
```
**目的**：理解 IWYU 原则——header 永远自包含。

---

# 六、值得深入思考的问题

### Q1: opaque type 看起来很好——为什么 C 标准不直接引入"private 关键字"？

**观点 A**：C 的 opaque type + forward decl + IWYU 已足够；引入新关键字是过度。
**观点 B**：Java/C++ 的 `private` 简化了"封装"语义；C 的两个 header 是 workaround。
**问题**：C 标准应不应该引入 `private`/`public` 关键字来简化封装？**这对 ABI 有什么影响？**

### Q2: static library 已"几乎足够"——为什么现代 OS 还保留 dynamic linking？

**提示**：浏览器、IDE、游戏都用 dynamic 加载插件。
**问题**：dynamic linking 的真正价值是"运行时扩展性"还是"节省磁盘/内存"？**这影响了你今天该如何选型？**

### Q3: `static` 在 file scope 给 linkage、在 block scope 给 storage——同一关键字双重含义是 bug 吗？

**问题**：标准委员会是否考虑过用 `internal` / `persistent` 区分？**为什么没有？** 这是 C 历史包袱还是实用主义？

### Q4: MISRA 强制所有 internal helper 用 `static`——这与"static 函数 inline"有什么 trade-off？

**问题**：`static inline` 在 header 里 vs `static` 在 .c 里——哪个更利于 MISRA 合规 + linker + cache？

### Q5: 在 container/robot/汽车这种多团队项目里，opaque type + 自包含 header + static linkage 三件套是否足够？

**问题**：当 50 个团队各自有自己的"opaque type"时，是否会出现 ABI 兼容性灾难？**如何设计跨团队的"接口标准"？**

---

*下一章预告*: **Chapter 11 — Debugging, Testing, and Analysis** —— `assert` / `static_assert`（C11） / runtime assertion、compiler flags（`-Wall -Wextra -Werror -Wpedantic -fsanitize`）、debugging（gdb / lldb）、unit testing（Unity / Check / cmocka）、static analysis（clang-tidy / cppcheck）、dynamic analysis（Valgrind / ASan / UBSan / MSan / TSan）。这是把前面 10 章知识变成"可信工程产出"的最后一公里。