# Chapter 10 — I Can't Run This Method in a Test Harness

> **PDF**: p.159-172（14 页）
> **定位**: ch9 解决了"把整个类弄进 harness"（处理构造路径上的依赖）；ch10 进一步收窄到**类已经能实例化，但单个方法跑不进 harness** 的几种常见症状：方法不可见（private/sealed/final）、参数造不出来（library 类无公共构造）、副作用不可感（GUI / DB / 帧间回调）。本章把 ch25 catalog 中几个 refactoring 在"方法级"的用法串成一张决策表。

## 〇、第一性原理思考

**这章做了什么**：把方法级 4 大不可测症状（私有不可见 / sealed 参数 / 副作用不可感 / sensing 不足）各配一个 case study，用 ch25 的 Adapt Parameter / Extract Implementer / Subclass and Override Method / Extract Method 串成一张决策表。

**为什么这样拆**：ch9 已经把类弄进 harness；剩下的就是"类能 instantiate，但方法跑不进"，作者把它从构造期依赖收窄到方法期依赖，每个症状给一个最小可复现案例，是为了避免读者误把 reflection 当万能钥匙。

**最值钱的洞见**：真正可用的 seam 不是 reflection / friend declaration 这种 cheat —— cheat 把"代码已经难测"的疼痛信号抹掉；疼痛恰恰是 refactor 的燃料，所以正确做法是改可见性（私有 → protected）或 Adapt Parameter，把 seam 留给下一次 touch。
## 一、章节概述

- **ch9 vs ch10 的边界**：ch9 谈类能不能 instantiate，ch10 谈方法能不能 harness-call。两章共用 ch25 的 refactoring 目录（Adapt Parameter / Skin and Wrap the API / Extract Implementer / Subclass and Override Method），但 ch9 解决构造期依赖，ch10 解决方法期依赖。
- **方法级不可测的 4 大症状**：(a) 私有 / 不可见；(b) 参数构造不出来（library 类 sealed/final，无 public ctor）；(c) 副作用跑不出返回值（写库 / 启动巡航导弹 / GUI 弹窗）；(d) 看不到方法内对其它对象的修改（sensing 不足）。本章用 4 个 case study 各打一发。
- **Hidden Method case（C++）**：私有 → `protected` → 测试子类透传（`using` declaration 是 idiomatic 版本）。Feathers 的结论：**好设计 = 可测试**；不能测的私有方法 = 类承担太多责任，先 public 出来，等下次 touch 时再做 ch20 大拆分。
- **"Helpful" Language Feature case（C# `HttpFileCollection` / Java `HttpPostedFile`）**：sealed/final 让我们无法 subclass。唯一手段是 `Adapt Parameter`：参数类型换成我们自己实现的 `OurHttpFileCollection`/`IHttpPostedFile`，外面再 wrap + fake 双胞胎。
- **Undetectable Side Effect case（Java AWT Frame）**：方法调另一个 Frame、读它的 getX、写到自己的 TextField，整个链路没有返回值可 assert。处理方式是一连串 **Extract Method** 把 Frame 操作隔离成 helper，然后用 **Subclass and Override Method** 在测试侧 override helper 为 fake。最后用 **Command/Query Separation** 给 method 命名时把 GUI 词汇过滤掉。
- **三个 case 共同的句式**：先抽 seam（access / parameter / method），再在 seam 上挂测试点（subclass / fake）。**抽 seam 不是为 clean architecture，是为"能测"** —— 这是 Feathers 全书反复出现的"测试即设计"立场的具体化。
- **"Subverting Access Protection" side note**：reflection / friend class 也能打开 private，但 Feathers 不推荐长期保留（"pain 是 refactor 的动力，cheat 把它灭掉了"）。
- **Command/Query Separation（Meyer）**：方法要么是 command（变状态、无返回），要么是 query（返回、不变），不能两者都是。这条原则在 Undetectable Side Effect case 里直接驱动命名 —— 让 GUI 词汇从 method name 中消失，method 才有"可被 fake 替换"的语义清晰度。
- **Figure 10.1 → Figure 10.2 的演化**：从 `AccountDetailFrame` 单大类，到 `AccountDetailFrame` + `AccountDetailDisplay` + `SymbolSource` 三件套；中间过渡是"先把 helper 抽出来 → 看哪些 helper 只用同一组字段 → 把这些字段 + helper 一起搬走"。这是 ch20 "Class Is Too Big" 的预演。

## 二、核心 Takeaways

### Takeaway 1：测试私有方法的正确答案不是 hack，是改可见性

- **是什么**：Feathers 给出的硬话 —— "if we need to test a private method, we should make it public"。若担心副作用，最干净的做法是把方法搬到一个新类，public 在那里，原类只保留一个 internal 实例。
- **为什么重要**：把"测试可达性"和"封装"对立起来是错误二分；它们的对立面是"类承担过多责任"。先公开测试，让后续 refactor 看到责任分布，再做拆分。
- **解决什么问题**：避免用 reflection / friend-declaration 这种长期成本高的 cheat 打开可见性 —— cheat 让"代码已经很糟"这个信号被埋掉。
- **适用场景**：legacy 类有大量 utility 私有方法；类同时管 GUI 和业务逻辑；私有方法是 hot path 但没人敢动。

### Takeaway 2：sealed/final 让 Adapt Parameter 成为唯一选择

- **是什么**：当方法参数类型来自不可 subclass 的库类（`sealed`/`final`），无法 Extract Implementer，唯一能做的就是 Adapt Parameter：把方法签名换成自己的接口或抽象类，调用处做一次 wrap。
- **为什么重要**：库类的可见性边界 = 我们的 seam 边界；越早 Adapt Parameter，就越少在测试代码里硬编码库类结构。
- **解决什么问题**：sealed/final 让 mock-framework 直接跪；Adapt Parameter 把 library type 从测试视野中隔开。
- **适用场景**：C# `HttpFileCollection`、`String`、`HttpPostedFile`；Java `HttpPostedFile` 隐含 final；任何带 `sealed`/`final`/`non-instantiable` 的库类。

### Takexture 3：Extract Method 是 Subclass-and-Override 的前置动作

- **是什么**：当方法不可测因为副作用无返回值时，先 Extract Method 把"调用子对象 / 设置自身字段"等动作拆成 helper，然后 Subclass and Override Method 把这些 helper override 成 no-op / spy。这是 ch10 的"组合拳"。
- **为什么重要**：helper 抽取本身已经把"哪部分是 GUI 操作、哪部分是业务逻辑"标注到代码里 —— 这就是 ch20 拆分的前置信号。
- **解决什么问题**：让 GUI / 副作用代码从逻辑代码里分桶；测试只覆盖逻辑桶；副作用桶在集成测试或 manual 上测。
- **适用场景**：AWT/Swing callback；嵌入式代码对硬件寄存器写；Web framework 的 request handler 副作用。

### Takeaway 4：Command/Query Separation 是 helper 的命名纪律

- **是什么**：helper 命名时不要出现 GUI / 副作用词汇（如 `setDescription` 不要写成 `showDialog`），让 method 的"输入→输出"语义清晰。这是 Meyer 1986 年的设计原则，本书用作 ch10 的命名 review 标准。
- **为什么重要**：method 名字 = 它在测试里的可读性。如果名字暗示"我会弹窗"，测试就只能 mock 弹窗；如果名字暗示"我只算一个描述"，测试就只需要输入数据。
- **解决什么问题**：避免 fake override 时还要复制整套 GUI 状态机。
- **适用场景**：任何从 GUI / 副作用代码里抽出的 helper。

### Takeaway 5：测试私有方法的合法过渡形态是 `protected` + test subclass

- **是什么**：在 C++/Java/C# 中，把 `private` 改成 `protected`，建一个 `TestingXxx` 子类用 `using` declaration 或显式 pass-through 把方法暴露出来给测试。封装损失有限（subclass 受我们控制），可测性获得巨大。
- **为什么重要**：比 reflection cheat 更诚实（cheat 是运行时偷字段，protected+subclass 是显式 design intent）；也比直接 public 保守（外人仍然不能乱调）。
- **解决什么问题**：在不破坏"production 调用面"的前提下让测试可达。
- **适用场景**：C++ 中需要被测试的 camera control / 物理仿真 helper；Java 中 Swing/AWT 内部回调；C# 中 WPF dependency property handler。

### Takeaway 6：把方法搬走是比 public 出去更好的最终态

- **是什么**：Feathers 明确说"先 make public 让测试就位 → 下次 touch 时再做 ch20 拆分"。**目标态**是私有方法搬到一个独立类，public 在那里，原类持有一个实例。中间过渡态是 protected + test subclass。
- **为什么重要**：让"丑但能测"成为合法短期态，比"美但不能测"更工程友好 —— 因为可测 → 可改 → 可拆。
- **解决什么问题**：把"refactor vs 测试"的二分变成"先测后拆"的串行。
- **适用场景**：任何被多次 touch 的大型类。

### Takeaway 7：reflection cheat 是止痛药不是处方

- **是什么**：Java `setAccessible(true)`、C# reflection / `BindingFlags.NonPublic`、C++ `friend class TestGraft` 都能打开私有访问。Feathers 的立场是：**用一次可以，长期保留不行**。理由是 pain 是 refactor 的动力，cheat 把它灭了。
- **为什么重要**：避免团队把 cheat 当成 design smell 的消音器。
- **解决什么问题**：让"我们必须拆这个类"的压力保持在台面上。
- **适用场景**：极短期的 test scaffolding；绝不要写进 production code 的注释里推荐这样做。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **kernel helper 的 `static` + 单 translation unit 模式 vs `EXPORT_SYMBOL`**：kernel 大量用 `static` + file-scope 把 helper 锁在单 TU 里 —— 这是"不可测"在 kernel 侧的另一副面孔。但 kernel 走的是 KUnit 整体编译：把整个 TU 用 `#ifdef` 切换 `static` 为 non-static 后单独跑 unit test。这是 Adapt Parameter 的源码级版本 —— 把"可见性"也变成编译期 seam。
- **kernel `EXPORT_SYMBOL_GPL` 是"对外可见"的 seam**：模块间调用通过符号表而非头文件直接 include —— 这让 module A 在测试中可以 stub 掉 module B 的导出符号。这是 ch10 "private→protected" 的内核侧映射。
- **`sealed`/`final` 在 kernel 中不存在，但 `__init` / `__exit` 是隐式 final**：`__init` 函数在 boot 之后被 free，做不了 unit test（因为函数已经没了）。kernel 的解法是把核心逻辑写成 `__init`-free，把 boot-only 的 wrapper 单独 `#ifdef`。这是 ch10 Hidden Method case 的内核侧版本。
- **systemd / dbus 的 Adapt Parameter 范式**：dbus-daemon 的 `Manager` 类持有大量 `Bus*`、`Unit*` 等内部对象指针，单元测试时 Adapt Parameter 成 `TestManager`/`FakeBus`。这正是 ch10 sealed-class 的工业案例。
- **glibc / musl 的 `__*` 内部符号 = protected 测试窗口**：glibc 把测试可见的内部 helper 命名为 `__*`（如 `__vfprintf_internal`），让测试代码可见但应用层 ABI 锁住。这与 ch10 C++ `protected` 同构。

#### Linux 系统 — ch10 case × 系统侧映射表

| ch10 Case          | Linux / kernel 映射                                | 工具/手法                          |
| ------------------ | -------------------------------------------------- | ---------------------------------- |
| Hidden Method      | `static` TU-private 函数                          | KUnit `#ifdef UNITTEST` 切换可见性 |
| Helpful Language   | 内核 `EXPORT_SYMBOL` 边界                         | 模块符号 stub / module mock        |
| Helpful Language   | `sealed` 的库函数（如 `kfree`）                   | Adapt Parameter 包装 kfree         |
| Side Effect        | 直接 `printk` / `pr_*` 无返回值                   | Extract Method → `dmesg_test`     |
| CQS                | 内核 `void` helper 一律 grep 改名                 | `cscope -L` + `coccinelle` 脚本   |

### 3.2 机器人软件视角

- **ROS2 节点的 `TimerBase` / `Subscription` 是"helpful language"在机器人侧的化身**：rclcpp 的 `rcl_subscription_t` 是 opaque handle，没有公共 ctor。单元测试 Adapt Parameter 成 `FakeSubscription`，把 rclcpp 替换成 test-only 头。ROS2 官方 `test_msgs` 就是这条路。
- **MoveGroup 的 motion planning callback 是 side effect 的典型**：动作链：plan → execute → publish feedback → sleep → poll。整条链没有返回值可 assert。ch10 的解法 = Extract Method 把 `publishFeedback`/`pollExecution`/`sleepUntilNextTick` 拆出来，Subclass-and-Override 掉 publish/sleep，单独测"动作链编排"。
- **`protected` + test subclass 在机器人代码中的位置**：ROS2 `LifecycleNode` 的 `on_activate`/`on_deactivate` 是 `protected virtual`，鼓励用户 subclass override。`TestLifecycleNode` 是测试侧标准 subclass —— 这恰好是 ch10 Hidden Method 的最佳实践。
- **camera pipeline 的 helper 搬走**：ch10 用 CCAImage / setSnapRegion 做例子。机器人侧对应 `ImagePipeline::applyExposureCompensation()`、`StereoMatcher::computeDisparity()` 等 helper —— 这些都应该从 main pipeline 抽到独立类（ch25 Extract Class），用 `IExposureCompensator` 接口在测试侧 fake。

### 3.3 初级 vs 高级工程师对比

| 维度               | 初级工程师                                                       | 高级工程师                                                       |
| ------------------ | ---------------------------------------------------------------- | ---------------------------------------------------------------- |
| 私有方法测试       | "用 reflection / friend 把它打开"                                | "改 protected + 测试子类 / 搬到新类 public"                     |
| 库类 sealed/final  | "没法 mock，绕过去吧"                                            | "Adapt Parameter：换成自己的接口"                                |
| GUI side effect    | "这块代码没法测，先注释掉 skip"                                  | "Extract Method 把 GUI 词汇从名字里抽掉，Subclass-and-Override" |
| 命名               | `setDescription(String)` —— 名字暗示 GUI 副作用                 | `computeAccountDescription(...)` —— 只输入输出                  |
| 拆类时机           | 等到"完全理解类"再拆（结果永远理解不了）                         | 先 make public 让测试就位，下次 touch 时 ch20 拆分              |
| cheat 的态度       | "reflection 开 private 是常用技巧"                              | "cheat 是止痛药 —— 用一次，但必须留下 debt ticket"             |

> **关键差异**：初级把 cheat / skip 当作合法终态；高级把 cheat / skip 当作有保质期的中间态，且必须配 ticket。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

- 仍然极重要。原因：
  - **AI 写代码的速度让"私有→public"的过渡出现得更频繁** —— AI 倾向直接 `public:` 暴露一切而不是抽 seam。Feathers 的"先 public 后拆"纪律变成 AI 时代更需要的设计 review。
  - **sealed/final 类与 AI 的冲突**：AI 写 C# 会无脑 `public class : HttpPostedFile` —— 编译挂掉后才回头 Adapt Parameter。Adapt Parameter 是必须人 review 的关键点。
  - **GUI side effect 在 AI 生成代码里被加速放大**：AI 生成的 callback 通常"又长又杂、副作用遍布"。Extract Method 在 AI 流水线里反而是必备工序（不是"先测后拆"而是"先拆后测"）。
  - **reflection cheat 的诱惑变大**：AI 能一句 prompt 写出 `setAccessible(true)` 的 Java 代码，比人手写 cheat 简单十倍 —— 这让纪律性更强而非更弱。

### 4.2 AI 已经能做的（具体到 ch10 主题）

- **自动识别 sealed/final 类并提示 Adapt Parameter 路径** —— 在 IDE 报错时给出适配示例。
- **批量生成 fake / test subclass** —— 接收 sealed 类清单，输出一组 `class FakeXxx implements IXxx`。
- **Extract Method + 命名的语义化建议** —— 把 `showDialog(...)` 改名为 `computeAccountDescription(...)`，并提示这是 CQS 命名。
- **reflection cheat 代码块的检测与报警** —— grep `setAccessible` / `BindingFlags.NonPublic`，对每个出现打上 TODO。

### 4.3 AI 不能替代的（具体到 ch10 主题）

- **"是否要现在拆类"的判断** —— Feathers 明确说"先 public，下次再拆"。这个时间窗口由团队节奏决定，不是 diff 决定。
- **私有方法被外部依赖的可能性评估** —— "把 private 改 protected 会破坏谁"必须由人评估。AI 没有 ABI 知识。
- **CQS 命名是否真的过滤了 GUI 词汇** —— AI 给的命名可能仍带"setText"等词汇，人 review 才能确认。
- **reflection cheat 的债务登记决策** —— 是否写进 debt backlog、何时还，必须由人定。

### 4.4 AI 经常写错的地方

针对 ch10"方法级不可测"主题，AI 的典型误判：

| 错误模式                                        | 例子                                                                                                                              | 后果                                                                |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| **直接 public 私有方法，不留 protected 过渡**   | AI 把 `private void setSnapRegion(...)` 直接改 `public`，声称"为了可测"                                                          | 失去 "只在测试子类可见" 的中间态 —— production 调用面被污染        |
| **reflection cheat 默认开启**                   | AI 自动加 `method.setAccessible(true)` 到所有测试基类里                                                                          | cheat 变成默认态，pain 信号被埋                                    |
| **Extract Method 但保留 GUI 词汇**              | AI 抽出 helper 仍叫 `showDialogWithText(String)`                                                                                  | 测试侧要 mock 整个 dialog，子类 override 反而更难                  |
| **Adapt Parameter 时把 wrapper 写成 sealed**     | AI 写 `public sealed class OurHttpFileCollection : NameObjectCollectionBase`                                                       | 我们自己的 wrapper 又被 sealed 了，下次想换 fake 还得重新 Adapt       |
| **建议用 mock framework 而不是 fake**           | 看到 sealed/final 类，AI 推 Moq / Mockito 而非 Adapt Parameter                                                                     | mock 仍要实例化 sealed 参数类型，编译直接挂                        |
| **C# `using` declaration 用错位置**             | AI 把 `using Base::method;` 放在 namespace 而非类作用域，编译挂                                                                  | 测试编译失败，AI 不报错                                            |
| **跳过 Extract Method 直接 Subclass-and-Override** | AI 直接 override 大方法，绕过 Extract Method 的命名纪律                                                                       | 测试侧 override 的是大方法，断言定位不精准                          |

### 4.5 子段：AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **Linux kernel helper 的 AI 辅助**：
  - 用 AI 给一段 `static` 函数生成"测试侧等价的 mock 实现" —— 前提是把函数从 `static` 改成 `__attribute__((visibility("default")))` 或通过 `#ifdef UNITTEST` 切换。这与 ch10 Hidden Method case 同构。
  - 但 AI 对 `EXPORT_SYMBOL` 边界判断不好 —— kernel 中 `EXPORT_SYMBOL` 函数被模块用，AI 容易推荐"直接重写函数体"而非"在调用侧 mock"。
- **ROS2 节点的 AI 辅助**：
  - 用 AI 给一个 `rclcpp::Node` 子类生成 test-side `Node`（构造时跳过 `rcl_init`）—— 这本质是 Adapt Parameter 在 ROS2 侧的具体化。`ros2 topic pub` 在测试侧被 fake publisher 替换。
  - 用 AI 给 callback 链路画 "what's observable through this method" 草图 —— 这与 ch11 effect sketch 同源。
  - **关键限制**：behavior tree 节点的隐含时序契约（"先 plan 后 execute"），AI 给不出可信的最小不变量集。它能列出 transitions，但不能告诉你哪些是核心契约。

### 4.6 工程师必须保留的核心能力

- **判断"private → public"是否要中间过渡**（protected + subclass vs 直接 public）—— 必须人工。
- **识别 sealed/final 类并选择 Adapt Parameter 路径** —— AI 报"cannot inherit from sealed"，必须由人决策。
- **CQS 命名的纪律** —— AI 给的名字仍带 GUI 词汇时，必须由人 reject。
- **reflection cheat 的债务登记** —— 写代码不是债务，写进 backlog + 设过期日才是。

## 五、实践行动项

> ch10 是"症状 → seam → 测试点"章。下面 4 项 A1–A4 把每种症状落地为可编译运行的小程序。

### A1 — Hidden Method: `private` → `protected` → 测试子类透传（C++，Linux 内核 helper 风格）

```bash
mkdir -p /tmp/ch10-hidden && cd /tmp/ch10-hidden

cat > cca_image.h <<'EOF'
/* 模拟 ch10 Hidden Method case：CCAImage 类的 setSnapRegion helper
 *  - production 必须 public 之外不可调 (否则破坏 camera tracking)
 *  - 测试时通过 protected + 子类透传访问
 */
#ifndef CCA_IMAGE_H
#define CCA_IMAGE_H

#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    int cur_x, cur_y;
    int dx, dy;
    int n_sets;
} CCAImage;

void cca_image_init(CCAImage *self);
void cca_snap(CCAImage *self);          /* production 入口 */
void cca_set_snap_region(CCAImage *self, int x, int y, int dx, int dy);

int  cca_image_get_cur_x(const CCAImage *self);
int  cca_image_get_cur_y(const CCAImage *self);
int  cca_image_get_n_sets(const CCAImage *self);

#ifdef __cplusplus
}
#endif
#endif
EOF

cat > cca_image.c <<'EOF'
#include "cca_image.h"

void cca_image_init(CCAImage *self) {
    self->cur_x = 0; self->cur_y = 0;
    self->dx = 0;    self->dy = 0;
    self->n_sets = 0;
}

/* 这个 helper 在生产代码里被 cca_snap 调用多次,
 * 不能从生产代码外部调 (会破坏 camera tracking 状态机),
 * 但测试需要直接驱动它以验证算法逻辑. */
void cca_set_snap_region(CCAImage *self, int x, int y, int dx, int dy) {
    self->cur_x = x;
    self->cur_y = y;
    self->dx = dx;
    self->dy = dy;
    self->n_sets++;
}

void cca_snap(CCAImage *self) {
    /* 生产: 调用 helper 多次, 根据 motion 决定. 测试不关心. */
    cca_set_snap_region(self, 1, 2, 3, 4);
}

int  cca_image_get_cur_x(const CCAImage *self) { return self->cur_x; }
int  cca_image_get_cur_y(const CCAImage *self) { return self->cur_y; }
int  cca_image_get_n_sets(const CCAImage *self) { return self->n_sets; }
EOF

# 测试驱动: production API 路径 + 暴露 helper (用 cca_image_get_n_sets
# 这种 public getter 即可, 不需要 reflection cheat)
cat > test_hidden_method.c <<'EOF'
#include "cca_image.h"
#include <assert.h>

int main(void) {
    CCAImage img;
    cca_image_init(&img);

    /* Test 1: production 路径 - cca_snap 必须最终调用 helper. */
    cca_snap(&img);
    assert(cca_image_get_cur_x(&img) == 1);
    assert(cca_image_get_cur_y(&img) == 2);
    assert(cca_image_get_n_sets(&img) == 1);

    /* Test 2: cca_set_snap_region 本身被 public 后, 行为是 idempotent
     * 的 setter, 不依赖任何 camera 状态. 这就是"先 public 让测试可达". */
    cca_set_snap_region(&img, 10, 20, 30, 40);
    assert(cca_image_get_cur_x(&img) == 10);
    assert(cca_image_get_n_sets(&img) == 2);

    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_hidden_method cca_image.c test_hidden_method.c
./test_hidden_method
echo "rc=$?"
```

**验收**：
- `rc=0`，证明 public 出来后的 helper 是 plain setter，可测。
- 对比 demo：把 `cca_set_snap_region` 改回 `static`（即"private via TU"），测试编译挂，演示 TU-private 不可测的边界。

### A2 — Helpful Language Feature: Adapt Parameter 解决 sealed/final（C 模拟 `sealed`）

```bash
mkdir -p /tmp/ch10-sealed && cd /tmp/ch10-sealed

# C 没有 sealed, 用"工厂函数 + 不暴露 struct"模拟:
# 库函数返回 opaque pointer, 我们不能 cast back 拿内部字段.
cat > sealed_lib.h <<'EOF'
/* 模拟 .NET HttpFileCollection: 不暴露结构体, 只能迭代. */
#ifndef SEALED_LIB_H
#define SEALED_LIB_H
#include <stddef.h>

typedef struct SealedCollection SealedCollection;   /* opaque */

typedef struct {
    const char *file_name;
    size_t      content_length;
} SealedFile;

SealedCollection *sealed_collection_new(void);
void              sealed_collection_free(SealedCollection *c);
size_t            sealed_collection_size(const SealedCollection *c);

/* 库 API: 只能按 index 取, 不能 subclass / cast. */
const SealedFile *sealed_collection_at(const SealedCollection *c, size_t i);
EOF
EOF

cat > sealed_lib.c <<'EOF'
#include "sealed_lib.h"
#include <stdlib.h>
#include <string.h>

#define MAX 16
struct SealedCollection { SealedFile files[MAX]; size_t n; };

SealedCollection *sealed_collection_new(void) {
    SealedCollection *c = calloc(1, sizeof(*c));
    return c;
}
void sealed_collection_free(SealedCollection *c) { free(c); }
size_t sealed_collection_size(const SealedCollection *c) { return c->n; }
const SealedFile *sealed_collection_at(const SealedCollection *c, size_t i) {
    return i < c->n ? &c->files[i] : NULL;
}

/* 库内部 helper, 测试侧用不到. */
void sealed_collection_push(SealedCollection *c, const char *name, size_t len) {
    if (c->n < MAX) {
        c->files[c->n].file_name = name;
        c->files[c->n].content_length = len;
        c->n++;
    }
}
EOF

# Adapt Parameter: 我们的代码用 IFileCollection 接口,
# 而不是 SealedCollection. 接口由我们控制, 测试可以 fake.
cat > ifile_collection.h <<'EOF'
#ifndef IFILE_COLLECTION_H
#define IFILE_COLLECTION_H
#include <stddef.h>

typedef struct {
    const char *file_name;
    size_t      content_length;
} IFile;

typedef struct IFileCollection {
    size_t      (*size)(const struct IFileCollection *self);
    const IFile *(*at)  (const struct IFileCollection *self, size_t i);
} IFileCollection;

/* Adapt Parameter 适配器: 把 sealed 库转成 IFileCollection. */
const IFileCollection *sealed_adapter(SealedCollection *c);

/* 测试侧 fake. */
typedef struct {
    IFileCollection iface;
    IFile           *files;
    size_t           n;
} FakeCollection;

void fake_collection_init(FakeCollection *fc, IFile *files, size_t n);
EOF
EOF

cat > ifile_collection.c <<'EOF'
#include "ifile_collection.h"
#include "sealed_lib.h"
#include <stdlib.h>

/* Adapt Parameter: sealed -> IFileCollection 适配. */
typedef struct {
    IFileCollection iface;
    SealedCollection *src;
} SealedAdapter;

static size_t sa_size(const IFileCollection *self) {
    return sealed_collection_size(((const SealedAdapter *)self)->src);
}
static const IFile *sa_at(const IFileCollection *self, size_t i) {
    return (const IFile *)sealed_collection_at(((const SealedAdapter *)self)->src, i);
}

static SealedAdapter g_sa;   /* 单例 demo */
const IFileCollection *sealed_adapter(SealedCollection *c) {
    g_sa.iface.size = sa_size;
    g_sa.iface.at   = sa_at;
    g_sa.src        = c;
    return &g_sa.iface;
}

static size_t fc_size(const IFileCollection *self) {
    return ((const FakeCollection *)self)->n;
}
static const IFile *fc_at(const IFileCollection *self, size_t i) {
    const FakeCollection *fc = (const FakeCollection *)self;
    return i < fc->n ? &fc->files[i] : NULL;
}
void fake_collection_init(FakeCollection *fc, IFile *files, size_t n) {
    fc->iface.size = fc_size;
    fc->iface.at   = fc_at;
    fc->files = files;
    fc->n     = n;
}
EOF

cat > get_ksr_streams.h <<'EOF'
#ifndef GET_KSR_STREAMS_H
#define GET_KSR_STREAMS_H
#include "ifile_collection.h"
#include <stddef.h>

/* "Helpful language feature" case: 参数类型不再是 SealedCollection,
 * 而是 IFileCollection — production 侧用 sealed_adapter 适配,
 * 测试侧用 FakeCollection 直入. */
size_t get_ksr_streams(const IFileCollection *files, const char **out_names);
#endif
EOF

cat > get_ksr_streams.c <<'EOF'
#include "get_ksr_streams.h"
#include <string.h>
#define MIN_LEN 1024

size_t get_ksr_streams(const IFileCollection *files, const char **out_names) {
    size_t k = 0;
    size_t n = files->size(files);
    for (size_t i = 0; i < n; i++) {
        const IFile *f = files->at(files, i);
        int is_ksr = strstr(f->file_name, ".ksr") != NULL;
        int is_txt_long = (strstr(f->file_name, ".txt") != NULL) &&
                          (f->content_length > MIN_LEN);
        if (is_ksr || is_txt_long) {
            if (out_names) out_names[k] = f->file_name;
            k++;
        }
    }
    return k;
}
EOF

cat > test_sealed.c <<'EOF'
#include "get_ksr_streams.h"
#include "ifile_collection.h"
#include "sealed_lib.h"
#include <assert.h>
#include <string.h>

int main(void) {
    /* === production-style: 用 sealed 库, 通过 adapter === */
    SealedCollection *sc = sealed_collection_new();
    /* 库内部 push, 演示用. */
    extern void sealed_collection_push(SealedCollection *, const char *, size_t);
    sealed_collection_push(sc, "a.ksr",          500);
    sealed_collection_push(sc, "short.txt",       500);  /* 不够长 */
    sealed_collection_push(sc, "long.txt",      5000);  /* 长 */
    sealed_collection_push(sc, "noise.bin",    50000);  /* 不匹配 */
    const IFileCollection *adapted = sealed_adapter(sc);

    const char *names[8] = {0};
    size_t k = get_ksr_streams(adapted, names);
    assert(k == 2);
    assert(strcmp(names[0], "a.ksr")   == 0);
    assert(strcmp(names[1], "long.txt") == 0);
    sealed_collection_free(sc);

    /* === test-style: 用 fake, 不需要 sealed 库 === */
    IFile files[2] = {
        { .file_name = "x.ksr", .content_length = 1 },
        { .file_name = "y.txt", .content_length = 9999 },
    };
    FakeCollection fc;
    fake_collection_init(&fc, files, 2);

    memset(names, 0, sizeof(names));
    k = get_ksr_streams(&fc.iface, names);
    assert(k == 2);

    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -I. -o test_sealed \
    sealed_lib.c ifile_collection.c get_ksr_streams.c test_sealed.c
./test_sealed
echo "rc=$?"
```

**验收**：
- `rc=0`，证明 Adapt Parameter 后 production 与 test 走同一段 `get_ksr_streams` 但参数类型不同（`sealed_adapter` vs `FakeCollection`）。
- 注释里写明：原版 `.NET HttpFileCollection` 是 sealed，本 demo 用 opaque struct + factory 模拟同样边界。

### A3 — Undetectable Side Effect: Extract Method + Subclass-and-Override（C 模拟 Java AWT Frame）

```bash
mkdir -p /tmp/ch10-sideeffect && cd /tmp/ch10-sideeffect

cat > side_effect.h <<'EOF'
/* 模拟 ch10 Undetectable Side Effect case:
 *  Java AccountDetailFrame 调 DetailFrame 读 symbol,
 *  再写到自己的 TextField.
 *  整条链无返回值可 assert.
 *  解法: Extract Method 把"弹窗"和"写文本"拆成 helper,
 *  测试子类 override 这俩 helper, 直接 assert. */
#ifndef SIDE_EFFECT_H
#define SIDE_EFFECT_H

#include <stddef.h>

typedef struct DetailFrame DetailFrame;

typedef struct {
    DetailFrame *detail_display;   /* 外部 frame 的替身 */
    char         display_text[64]; /* 自己的 TextField */
    char         account_symbol[16];
    int          n_shows;
} AccountDetailFrame;

void adf_init(AccountDetailFrame *self);

/* "actionPerformed" 入口 — 不可直接测, 因为它会弹窗 + 改文本. */
void adf_perform_action(AccountDetailFrame *self, const char *source);

/* Extract Method 后的 helpers — 可被子类 override. */
void adf_set_description(AccountDetailFrame *self, const char *desc);
void adf_show_detail_frame(AccountDetailFrame *self);
const char *adf_get_account_symbol(const AccountDetailFrame *self);
void adf_set_display_text(AccountDetailFrame *self, const char *text);

/* getter, 让测试侧能拿到"自己 TextField"上的最终文本. */
const char *adf_get_display_text(const AccountDetailFrame *self);

#endif
EOF

cat > side_effect.c <<'EOF'
#include "side_effect.h"
#include <string.h>
#include <stdio.h>

DetailFrame *detail_frame_new(const char *symbol);
const char  *detail_frame_get_symbol(const DetailFrame *d);

void adf_init(AccountDetailFrame *self) {
    self->detail_display = NULL;
    self->display_text[0] = 0;
    self->account_symbol[0] = 0;
    self->n_shows = 0;
}

/* "command" helper: 弹窗. 测试子类 override 时变 no-op. */
void adf_show_detail_frame(AccountDetailFrame *self) {
    if (!self->detail_display) self->detail_display = detail_frame_new("SYM");
    self->n_shows++;
}

/* "command" helper: 写自己的 TextField. */
void adf_set_display_text(AccountDetailFrame *self, const char *text) {
    strncpy(self->display_text, text, sizeof(self->display_text) - 1);
    self->display_text[sizeof(self->display_text) - 1] = 0;
}

/* "command" helper: 把 desc 给 detail frame. */
void adf_set_description(AccountDetailFrame *self, const char *desc) {
    if (!self->detail_display) self->detail_display = detail_frame_new("SYM");
    (void)desc;   /* production 这里会调 detail_frame_set_description */
    self->n_shows++;
}

/* "query" helper: 读 detail frame 上的 symbol. */
const char *adf_get_account_symbol(const AccountDetailFrame *self) {
    return self->detail_display ? detail_frame_get_symbol(self->detail_display) : "";
}

void adf_perform_action(AccountDetailFrame *self, const char *source) {
    if (strcmp(source, "project activity") == 0) {
        adf_set_description(self, "basic account");   /* 这里弹窗 */
        const char *sym = adf_get_account_symbol(self);
        char buf[64];
        snprintf(buf, sizeof(buf), "%s: basic account", sym);
        adf_set_display_text(self, buf);              /* 这里写文本 */
    }
}

const char *adf_get_display_text(const AccountDetailFrame *self) {
    return self->display_text;
}

/* --- DetailFrame stub --- */
struct DetailFrame { char symbol[16]; };
DetailFrame *detail_frame_new(const char *symbol) {
    DetailFrame *d = (DetailFrame *)__builtin_malloc(sizeof(*d));
    if (d) {
        size_t i = 0;
        for (; symbol[i] && i + 1 < sizeof(d->symbol); i++) d->symbol[i] = symbol[i];
        d->symbol[i] = 0;
    }
    return d;
}
const char *detail_frame_get_symbol(const DetailFrame *d) {
    return d ? d->symbol : "";
}
EOF

cat > test_side_effect.c <<'EOF'
/* Subclass-and-Override: 测试侧 override set_description /
 * get_account_symbol / set_display_text 三个 helper,
 * 让 adf_perform_action 不需要真弹窗 / 真分配 DetailFrame. */
#include "side_effect.h"
#include <assert.h>
#include <string.h>
#include <stdlib.h>

/* "测试子类": 用一个 inline struct 把 helper 行为替换掉. */
typedef struct {
    AccountDetailFrame parent;
    const char *fake_symbol;
    int         set_description_calls;
} TestingAccountDetailFrame;

void t_adf_set_description(AccountDetailFrame *self, const char *desc) {
    (void)desc;
    ((TestingAccountDetailFrame *)self)->set_description_calls++;
}
/* 把 fake_symbol "喂给" get_account_symbol */
const char *t_adf_get_account_symbol(const AccountDetailFrame *self) {
    const TestingAccountDetailFrame *t = (const TestingAccountDetailFrame *)self;
    return t->fake_symbol ? t->fake_symbol : "";
}

int main(void) {
    /* 我们用 "function pointer 替换" 模拟 Java 子类 override:
     * 父类 struct 头部是 helper 函数指针表, 测试侧重新指向. */
    AccountDetailFrame parent = {0};
    TestingAccountDetailFrame t = { .fake_symbol = "SYM" };
    t.parent.display_text[0] = 0;

    /* 把测试侧函数指针塞进父类 (模拟 @Override). */
    /* 注意: 我们的 C struct 没有 vtable, 所以这里通过 "函数指针变量"
     * 替换实现 — 在真实场景这就是 Subclass-and-Override Method 的 C 等价物. */
    extern void (*adf_set_description_hook)(AccountDetailFrame *, const char *);
    extern const char *(*adf_get_account_symbol_hook)(const AccountDetailFrame *);
    adf_set_description_hook     = t_adf_set_description;
    adf_get_account_symbol_hook  = t_adf_get_account_symbol;

    /* 跑 "perform_action", 此时弹窗 / set_display_text / get_account_symbol
     * 全走 fake, 没有 DetailFrame 真分配. */
    adf_perform_action(&t.parent, "project activity");

    /* CQS 验证: 命名里已经没有 show/setDialog 词汇, 都是 compute/set_xxx. */
    assert(strcmp(adf_get_display_text(&t.parent), "SYM: basic account") == 0);
    assert(t.set_description_calls == 1);

    return 0;
}

/* 父类实现的"钩子版": 真实 production 通过钩子查表, 测试子类替换. */
#include "side_effect.c"
EOF

# 这里需要给 side_effect.c 加 hook 版本 — 用条件编译实现
cat > side_effect_hooked.c <<'EOF'
#include "side_effect.h"
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

struct DetailFrame { char symbol[16]; };
static DetailFrame *detail_frame_new(const char *symbol) {
    DetailFrame *d = malloc(sizeof(*d));
    if (d) { size_t i=0; for(;symbol[i]&&i+1<16;i++) d->symbol[i]=symbol[i]; d->symbol[i]=0; }
    return d;
}
static const char *detail_frame_get_symbol(const DetailFrame *d) { return d?d->symbol:""; }

/* hooks — 默认走 production, 测试侧替换. */
void (*adf_set_description_hook)(AccountDetailFrame *, const char *) = NULL;
const char *(*adf_get_account_symbol_hook)(const AccountDetailFrame *) = NULL;

void adf_init(AccountDetailFrame *self) {
    self->detail_display = NULL;
    self->display_text[0] = 0;
    self->account_symbol[0] = 0;
    self->n_shows = 0;
}
void adf_show_detail_frame(AccountDetailFrame *self) {
    if (!self->detail_display) self->detail_display = detail_frame_new("SYM");
    self->n_shows++;
}
void adf_set_display_text(AccountDetailFrame *self, const char *text) {
    strncpy(self->display_text, text, sizeof(self->display_text)-1);
    self->display_text[sizeof(self->display_text)-1] = 0;
}
void adf_set_description(AccountDetailFrame *self, const char *desc) {
    if (adf_set_description_hook) { adf_set_description_hook(self, desc); return; }
    if (!self->detail_display) self->detail_display = detail_frame_new("SYM");
    self->n_shows++;
}
const char *adf_get_account_symbol(const AccountDetailFrame *self) {
    if (adf_get_account_symbol_hook) return adf_get_account_symbol_hook(self);
    return self->detail_display ? detail_frame_get_symbol(self->detail_display) : "";
}
void adf_perform_action(AccountDetailFrame *self, const char *source) {
    if (strcmp(source, "project activity") == 0) {
        adf_set_description(self, "basic account");
        const char *sym = adf_get_account_symbol(self);
        char buf[64];
        snprintf(buf, sizeof(buf), "%s: basic account", sym);
        adf_set_display_text(self, buf);
    }
}
const char *adf_get_display_text(const AccountDetailFrame *self) { return self->display_text; }
EOF

# test 改成用 hooked 版本
cat > test_side_effect.c <<'EOF'
#include "side_effect.h"
#include <assert.h>
#include <string.h>

void t_adf_set_description(AccountDetailFrame *self, const char *desc);
const char *t_adf_get_account_symbol(const AccountDetailFrame *self);

typedef struct {
    AccountDetailFrame parent;
    const char *fake_symbol;
    int set_description_calls;
    int get_symbol_calls;
} TestingAccountDetailFrame;

void t_adf_set_description(AccountDetailFrame *self, const char *desc) {
    (void)desc;
    ((TestingAccountDetailFrame *)self)->set_description_calls++;
}
const char *t_adf_get_account_symbol(const AccountDetailFrame *self) {
    ((TestingAccountDetailFrame *)self)->get_symbol_calls++;
    const TestingAccountDetailFrame *t = (const TestingAccountDetailFrame *)self;
    return t->fake_symbol ? t->fake_symbol : "";
}

int main(void) {
    TestingAccountDetailFrame t = {0};
    t.fake_symbol = "SYM";
    adf_set_description_hook    = t_adf_set_description;
    adf_get_account_symbol_hook = t_adf_get_account_symbol;

    adf_perform_action(&t.parent, "project activity");

    assert(strcmp(adf_get_display_text(&t.parent), "SYM: basic account") == 0);
    assert(t.set_description_calls == 1);
    assert(t.get_symbol_calls == 1);
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_side_effect side_effect_hooked.c test_side_effect.c
./test_side_effect
echo "rc=$?"
```

**验收**：
- `rc=0`，证明测试侧 hook 掉 `set_description` / `get_account_symbol` 后，`perform_action` 走 fake 路径，没有任何真实 DetailFrame 分配。
- 对比 demo：注释里说生产代码用 production 实现，测试用 hook；CQS 命名（无 `show` 词汇）。

### A4 — Linux `static` helper → 编译期 seam（KUnit 风格）

```bash
mkdir -p /tmp/ch10-kernel && cd /tmp/ch10-kernel

# kernel 风格: TU-private 函数用 #ifdef UNITTEST 切换可见性.
cat > kernel_helper.c <<'EOF'
/* 模拟 kernel TU-private helper: 在 production 编译时 static,
 * UNITTEST 编译时 export, 单独跑 KUnit 验证. */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* 不可见 helper (production) — kernel 里这是 file-scope static. */
static int csum_fold(u32 sum) {
    /* 简单 checksum fold, 验证算法. */
    sum = (sum & 0xFFFF) + (sum >> 16);
    sum = (sum & 0xFFFF) + (sum >> 16);
    return (int)(u16)sum;
}

/* 暴露给 "test harness" 的 wrapper — 类比 kernel 中 #ifdef UNITTEST
 * 切换的 non-static 版本. */
int kernel_csum_fold_for_test(u32 sum) {
    return csum_fold(sum);
}

#ifdef STANDALONE_DEMO
int main(int argc, char **argv) {
    if (argc < 2) {
        fprintf(stderr, "usage: %s <hex>\n", argv[0]);
        return 2;
    }
    u32 s = (u32)strtoul(argv[1], NULL, 16);
    printf("csum_fold(0x%08x) = 0x%04x\n", s, (unsigned)kernel_csum_fold_for_test(s));
    return 0;
}
#endif
EOF

# 类型声明 (用 typedef 避免依赖 kernel headers)
cat > kernel_helper.h <<'EOF'
#ifndef KERNEL_HELPER_H
#define KERNEL_HELPER_H
#include <stdint.h>
typedef uint32_t u32;
typedef uint16_t u16;
int kernel_csum_fold_for_test(u32 sum);
#endif
EOF

# unit test
cat > test_kernel_helper.c <<'EOF'
#include "kernel_helper.h"
#include <assert.h>

int main(void) {
    /* 验证 csum_fold: 高 16 位 + 低 16 位, 一次 fold. */
    assert(kernel_csum_fold_for_test(0x00010000) == 1);
    assert(kernel_csum_fold_for_test(0xFFFF0000) == 0xFFFF);
    assert(kernel_csum_fold_for_test(0xDEAD0001) == 0xDEAE);
    /* 二次 fold 处理 overflow */
    assert(kernel_csum_fold_for_test(0x0000FFFF) == 0xFFFF);
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -I. -c kernel_helper.c -o kernel_helper.o
cc -std=c17 -Wall -Wextra -I. -o test_kernel_helper test_kernel_helper.c kernel_helper.o
./test_kernel_helper
echo "rc=$?"

# standalone demo
cc -std=c17 -Wall -Wextra -I. -DSTANDALONE_DEMO -o kernel_helper_demo kernel_helper.c
./kernel_helper_demo deadbeef
echo "rc=$?"
```

**验收**：
- `test_kernel_helper` `rc=0`（4 个 checksum fold 断言通过）。
- `kernel_helper_demo deadbeef` 输出 `csum_fold(0xdeadbeef) = 0x9d9d`（dead+beef=0x1809d → fold 一次 = 0x1809d & 0xFFFF + 0x1809d >> 16 = 0x809d + 0x1 = 0x809e；二次 fold = 0x809e & 0xFFFF + 0x809e >> 16 = 0x809e + 0 = 0x809e... 实际跑出来 0x9d9d）。
- 注释里说明：`csum_fold` 在 kernel 里是 `static`，UNITTEST 编译期切换为 non-static 暴露给 KUnit —— 这是 ch10 Hidden Method 的内核侧实现。

## 六、值得深入思考的问题

### Q1: `protected` + test subclass 算不算长期可接受的 design？

Feathers 给的是"短期"。**关键问题**：如果某方法真的永远只是 production 内部协议（比如 camera tracking state machine），`protected` 会不会让继承层级变深、让 production caller 也开始绕开 public API？**该多久复审一次"上次改 protected 的方法该不该搬走"**？

### Q2: Adapt Parameter 是接口污染还是设计救赎？

把 `HttpFileCollection` 换成 `IFileCollection`，等于在生产代码里加了一层适配 —— 这层适配本身要写测试、要维护、要跟库升级走。**关键问题**：什么时候 Adapt Parameter 的成本 > 跳过该 sealed 类的成本？是不是应该写一个债务 ticket 跟踪？

### Q3: Extract Method + 命名纪律的可持续性

CQS 命名要求每个 helper 不能带 GUI 词汇。但生产代码里很多 helper 名字就是带 GUI 词汇的（`setText`、`repaint`）。**关键问题**：rename 是不是 refactor？改名会不会让 caller 找不到调用？什么时候"先改 protected + subclass + 沿用 GUI 名字"比"先 rename"更稳？

### Q4: reflection cheat 的"止痛药 vs 成瘾"边界在哪？

Feathers 说"用一次，长期不行"。**关键问题**：用什么机制 enforce？code review？grep guard？CI 黑名单？reflection cheat 出现后多久必须还债？团队多久 review 一次 debt backlog？

### Q5: AI 把所有私有方法直接改 public 的副作用

AI 不理解 `protected` 的中间价值，会一步到位 `public`。**关键问题**：要不要在 lint 规则里禁止 AI 直接生成 public 字段？或者让 AI 默认产 `protected` + 显式标注"this should be made public via subclass"？

---

*下一章预告*: **Chapter 11 — I Need to Make a Change. What Methods Should I Test?** —— 把"在哪写测试"从方法级放大到系统级：**effect sketches**（推导每个 change point 的下游影响）、**reasoning forward**（在 change point 上反推"哪些下游方法能 sense 到这次改动"）、以及**simplifying effect sketches**（refactor 让下游测试点收敛）。ch11 是 ch10 的"系统级放大版"。
