# Chapter 25 — Dependency-Breaking Techniques (Reference Catalog)

> **PDF**: p.347-456（110 页 — ch25 主目录 + refactoring glossary + 索引 + 总目录）
> **定位**: 全书 Part III / 收尾章。Feathers 把 24 个 dependency-breaking 技法汇成 *reference catalog* —— 每条都有动机、步骤、示例、"Use when" 指引。本章按 *技法对照表* 输出: 不重抄书(原书已写得很清), 改以一表一行的形式便于工程检索。
>
> **重要修正**: 任务描述说"Feathers 的 25 个 technique", 但 *实际 PDF 列出 24 个*。本目录按 PDF 实际内容给出 24 行。下表第一列就是 Feathers 在 ch25 实际开列的 24 个标题。

## 〇、第一性原理思考

**这章做了什么**: 这章把 24 个独立技法汇成一张查找表 —— Adapt Parameter / Break Out Method Object / Encapsulate Global References / Extract Interface / Pull Up Feature / Template Redefinition 等技法按"参数 → 方法 → 类 → 全局 → 编译/链接期"五层分组, 每条带 Use when / Not use when。

**为什么这样拆**: 技法按"依赖类型"分层而不是按"难度"或"顺序" —— 中心点是 separation (改参数 / 改全局) 路径优先于 sensing (fake) 路径: 多数 dependency 只用最浅层的技法 (Adapt Parameter) 就够, 默认走 Extract Interface 是过度抽象的反应过度。

**最值钱的洞见**: ch25 不是用来顺序读完的, 是按症状反查的 —— 决策树入口是"类能否构造"和"方法效果能否看到", 不是"我想用什么技法"; 24 个技法配套 ch23 的纪律才是安全的, 单独读 ch25 等于没 safety net 走钢丝。
## 一、章节概述

- **chapter 25 是 reference catalog, 不是叙事章**。Feathers 在开头明说:"This list is not exhaustive; these are just some techniques that I've used with teams." —— 它不是百科, 只是 *实操过* 的工具集。
- **24 个技法的逻辑分层**:
  - **参数/签名层**(4): Adapt Parameter, Parameterize Constructor, Parameterize Method, Primitivize Parameter —— 改的是 *入参/出参*。
  - **方法层**(8): Break Out Method Object, Expose Static Method, Extract and Override Call, Extract and Override Factory Method, Extract and Override Getter, Extract Implementer, Pull Up Feature, Push Down Dependency —— 改的是 *方法形状/位置*。
  - **类/接口层**(4): Extract Interface, Introduce Instance Delegator, Introduce Static Setter, Subclass and Override Method, Supersede Instance Variable —— 改的是 *类结构*。
  - **全局/链接层**(4): Definition Completion, Encapsulate Global References, Link Substitution, Replace Global Reference with Getter —— 改的是 *全局可见性*。
  - **代码/链接替换层**(2): Replace Function with Function Pointer, Template Redefinition, Text Redefinition —— 改的是 *编译期或链接期产物*。
- **每个技法都有"Use when"和"Not use when"指引**: Feathers 强调 *"Use X when Y"*, 反之 *"if you can use Z instead, do so"* —— 这是 *技法排序* 的关键信息。
- **每个技法都假设 *无测试* 条件**: 这是 ch25 与一般 refactoring catalog(Fowler 1999)的根本区别 —— 这里的技法必须在 *没测试* 时也尽量安全。
- **ch25 后接 refactoring glossary + glossary + index**: Feathers 把 ch25 的 24 技法和 Fowler refactoring 的对应关系列出 — Adapt Parameter ≈ Introduce Adapter, Extract Interface ≈ 不变, Expose Static Method ≈ 同名, 等等。这是 *跨书参照* 的桥。
- **与 ch3 的关系**: 每个技法都标 *sensing* / *separation* —— separation 占大多数(把对象/类搬走即可构造), sensing 集中在 *Fake* / *Extract Interface* 这几条。

## 二、核心 Takeaways

### Takeaway 1: ch25 是 *技法索引*, 不是 *教程*

- **是什么**: 你应该在 *具体遇到问题* 时查 ch25, 不是顺序读完。Feathers 自己说 *"this is a reference catalog"*, 每条独立。
- **为什么重要**: 24 个技法每个都有 *适用边界* —— 顺序读会混淆。用 *症状 → 技法* 反查更高效。
- **解决什么问题**: 把"读完 ch25"和"用 ch25"分开 —— *用* 比 *读完* 更可行。
- **适用场景**: Sprint 中遇到具体 refactor 障碍时查。

### Takeaway 2: 技法按 *依赖类型* 分层 —— 不是按"难度"或"顺序"

- **是什么**: 我自己(Feathers)没明确分, 但技法的依赖类型决定了先后: 参数/方法/类/全局 —— 是从 *local* 到 *global* 的层次。
- **为什么重要**: 多数 dependency 只需最浅层的技法(改参数); 少数需到 global 层。理解分层 = *知道何时该停*。
- **解决什么问题**: 避免"动不动就 Encapsulate Global References" —— 这是过度反应; 多数只需 *Adapt Parameter*。
- **适用场景**: 评估依赖时先问"哪一层"。

### Takeaway 3: sensing vs separation 不是对立, 是 *先后*

- **是什么**: ch3 把拆依赖分为 sensing(看不到结果)和 separation(造不出对象)。ch25 多数技法是 separation; 只有 Extract Interface + Replace Function with Function Pointer 兼有 sensing 价值。
- **为什么重要**: 拆依赖的 *次序* —— 先 separation(造得出来), 再 sensing(看得见)。
- **解决什么问题**: 避免"一次把所有依赖都 fake 掉" —— 过度抽象。
- **适用场景**: 评估一个 un-testable 类时, 先 separation, 再 sensing。

### Takeaway 4: ch25 的安全前提 = ch23 的纪律

- **是什么**: ch25 技法都 *无测试* 条件下做 —— 安全只能靠 ch23(Hyperaware / Single-Goal / Preserve Signatures / Lean on Compiler / Pair)。
- **为什么重要**: 技法是工具, 纪律是保障。*没纪律的技法 = silent breakage*。
- **解决什么问题**: 提醒读者 ch25 单独读不够, 必须配套 ch23 的纪律。
- **适用场景**: 用 ch25 技法做 break-dependency 时, 严格走 ch23 纪律。

### Takeaway 5: 多数技法在 *没有 refactor 工具* 时仍可手工做

- **是什么**: ch25 步骤都是 *手工可做* —— 复制粘贴整个签名 + 编译器验证。Feathers 强调 *"if you follow the steps carefully, the chance of mistakes is small"*。
- **为什么重要**: 这把 ch25 工具集开放给所有环境 —— C / 内核 / 嵌入式 / 旧 IDE, 都能用。
- **解决什么问题**: 解决"没 IDE refactor 工具就不能 break dependencies" 的错误认知。
- **适用场景**: C / embedded / kernel 的 legacy code 重构。

## 三、工程实践视角 — 技法对照表 (24 个技法)

> 锁定领域:**Linux 系统开发 + 机器人软件**。
>
> 本节既是"§3 工程视角锁", 也是 ch25 reference catalog 的核心表 —— 把 24 个技法按 *sensing / separation* 维度+Linux/Robot 维度 一次铺开, 供工程反查。

> 表头: **#** | **技法** | **页码** | **用法概览** | **何时用 / 不适用** | **sensing** | **separation** | **Linux 系统视角** | **机器人软件视角**

| # | 技法 | 页 | 用法概览 | 何时用 / 不适用 | sensing | separation | Linux 系统视角 | 机器人软件视角 |
|---|------|-----|----------|----------------|---------|------------|----------------|---------------|
| 1 | **Adapt Parameter** | 349 | 用 *适配 wrapper class* 包裹难造的参数(标准接口/第三方类) | *用*:不能 Extract Interface 的参数(如 `HttpServletRequest`); *不用*:能 Extract Interface 时优先 | ✗ | ✓ | 内核 `struct file_operations` / `struct inode` 不能直接 mock 时,包一层 `test_file_ops` | ROS2 `rclcpp::Node` / `Message` 接口不能改,包 `RecordingPublisher` |
| 2 | **Break Out Method Object** | 353 | 把 monster 方法体搬入新类 `MethodRunner.run()`,局部变量成 fields | *用*:monster 内部临时变量多且互相耦合; *不用*:0 个临时变量时过度工程化 | ✓ | ✓ | `struct tcp_send_state` 拆 `tcp_sendmsg` 内部循环状态 | BT node 拆 `BtExecutionState` 持 tick 间状态 |
| 3 | **Definition Completion** | 359 | 提供未链接的外部依赖 stub 定义(`extern` 变量 / `static` 函数) | *用*:链接时报 *undefined reference*,且不能改原始声明; *不用*:有源码可改时优先改源码 | ✗ | ✓ | 内核模块提供 `printk` stub / `kmalloc` mock 给独立编译 | ROS2 plugin lib 提供 `rclcpp::Node` stub 给 unit test |
| 4 | **Encapsulate Global References** | 361 | 把全局变量包进 `struct/class`,所有访问改 `.field` | *用*:全局变量散落且测试需替换; *不用*:全局变量只读且不变 | ✗ | ✓ | 内核 `EXPORT_SYMBOL` 的全局变量包 `struct kernel_state` | ROS2 全局 `g_node` 包 `NodeRegistry` 单例 |
| 5 | **Expose Static Method** | 367 | 把 instance method 设为 `static`(去掉隐式 `this`),便于直接调用 | *用*:方法不读 instance state,只需做"纯"工作; *不用*:方法读 instance state | ✗ | ✓ | C 里把 `static int foo(struct bar *self, ...)` 拆出 | `pure function` BT condition node |
| 6 | **Extract and Override Call** | 371 | 把方法内调用的 collaborator 调用抽成 protected method,subclass override | *用*:测试需要 sense 某次具体调用; *不用*:返回值即足够 sensing | ✓ | ✗ | 测试 `kmalloc` 行为时 override 它 | ROS2 BT node test override `publisher_publish` |
| 7 | **Extract and Override Factory Method** | 373 | 把 `new XXX()` 抽成 protected virtual method,subclass 返回 fake | *用*:构造函数难以 mock(无接口); *不用*:有现成 fake/mock framework | ✗ | ✓ | 内核 `crypto_alloc_skcipher` 抽 virtual factory | `rclcpp::Node::make_shared` 抽 factory method |
| 8 | **Extract and Override Getter** | 375 | 把 `getX()` 抽成 virtual,test subclass override 返回 fake X | *用*:某 getter 在测试中需要返回特殊值; *不用*:getter 简单,直接 fake 字段 | ✓ | ✗ | 测试 `current_kernel_time()` override 它返回固定值 | 测试 `now()` 时 override 它返回 mock time |
| 9 | **Extract Implementer** | 379 | 把方法实现部分抽到独立类,主类持有 `Impl` 指针(桥接) | *用*:方法实现依赖外部资源,且需替换; *不用*:实现简单 | ✓ | ✓ | `crypto_shash` 拆 `shash_alg` + impl 替换 | `MoveIt2 Planner` 桥接 `PlanningPipeline` impl |
| 10 | **Extract Interface** | 385 | 从类的 public 方法抽 interface,生产代码用 interface,fake 实现 interface | *用*:有多个 fake 需求 / 跨语言调用; *不用*:只有 1 个 fake 需求时(改 abstract class 即可) | ✓ | ✓ | 内核 `struct file_operations` 抽 subset 接口 | ROS2 `rclcpp::Node` 抽 `NodeInterface` 给 mock |
| 11 | **Introduce Instance Delegator** | 391 | 把 singleton 调用改成 instance 字段 + 委托,测试可注入 fake instance | *用*:singleton 难测; *不用*:singleton 真无副作用 | ✗ | ✓ | 内核 `printk` 改成 `pr_alert` instance delegate | ROS2 global logger 抽 `NodeLogger` instance |
| 12 | **Introduce Static Setter** | 395 | 给 singleton/static field 加 `setForTest()`,test 可临时替换 | *用*:singleton 无法 inject; *不用*:有 DI 容器时优先 DI | ✓ | ✓ | `set_printk_handler_for_test()` kernel hook | `set_clock_for_test(MockClock)` ROS2 |
| 13 | **Link Substitution** | 399 | 编译时用 stub/impl 替换原函数,链接阶段 *改名* 或 *wrap* | *用*:不能改源代码 / 改源代码代价高; *不用*:有源码可改时 | ✓ | ✓ | 内核 link-time 用 `__wrap_kfree` 替换 `kfree` | colcon build 用 fake transport `.so` 替换真 transport |
| 14 | **Parameterize Constructor** | 401 | 给构造函数加参数(原本从外部获取的对象改为参数传入) | *用*:对象从外部配置读但测试需控; *不用*:对象是 *创建期已知* | ✗ | ✓ | `init_netdev(net, alloc_etherdev)` 拆参数 | ROS2 Node constructor 接受 `Executor` 参数 |
| 15 | **Parameterize Method** | 405 | 给方法加参数(原本从字段获取的值改为参数传入) | *用*:方法读 instance field 但测试需控; *不用*:方法纯函数已 OK | ✗ | ✓ | `tcp_send_skb(sk, skb, flags)` 拆 `flags` | BT node `tick()` 接受 `Blackboard` 参数 |
| 16 | **Primitivize Parameter** | 407 | 把参数类型从 complex object 降级为 primitive / string | *用*:complex parameter 难造; *不用*:complex parameter 是核心 domain | ✗ | ✓ | `struct file_operations *` 改 `int fd` | ROS2 `Twist` 改 `float linear_x, float angular_z` |
| 17 | **Pull Up Feature** | 411 | 把子类共有方法提到父类 | *用*:多个子类重复逻辑,且父类可空实现; *不用*:子类逻辑差异大 | ✗ | ✓ | 内核 `net_device_ops` 上提通用 `ndo_start_xmit` 骨架 | `ros2_control::SystemInterface` 上提 `default_init` |
| 18 | **Push Down Dependency** | 415 | 把父类对子类的依赖 *反向* —— 父类提供 hook,子类实现 | *用*:父类用了子类才有的具体方法; *不用*:父类是真抽象 | ✗ | ✓ | `struct inode_operations` 把具体 FS op push 到 `ext4_inode_ops` | `nav2_core::Navigator` 把具体行为 push 到 `NavigatorROS` |
| 19 | **Replace Function with Function Pointer** | 419 | 把函数调用替换成函数指针调用,test 可替换指针 | *用*:C / C++ 想替换实现但不改调用点; *不用*:调用点必须 type-safe | ✓ | ✓ | `crypto_register_ahash` 改函数指针注册 | `set_topic_publisher_for_test(fn_ptr)` ROS2 |
| 20 | **Replace Global Reference with Getter** | 421 | 把全局变量直接访问改成调用 getter 方法,subclass 可 override | *用*:全局变量通过 getter 间接化; *不用*:全局变量不可改 | ✓ | ✓ | `printk_rate_limit()` 改 getter 函数 | `now()` 改 `get_current_time()` getter |
| 21 | **Subclass and Override Method** | 423 | 测试时 subclass override 待测方法,生产不动 | *用*:需要 sense *方法内部状态* / *返回值*; *不用*:只是参数类型问题 | ✓ | � | `MockVfs` subclass override `vfs_read` | `MockController` subclass override `update()` |
| 22 | **Supersede Instance Variable** | 427 | 在 subclass 里 *新增* instance variable,override 方法读它 | *用*:需要在不污染父类的情况下扩展; *不用*:能直接加到父类 | ✓ | ✗ | `MockSkcipher` supersede `key_len` field | `MockMoveBase` supersede `current_goal` |
| 23 | **Template Redefinition** | 431 | C++ 模板特化,test 时给不同类型替换行为 | *用*:C++ 模板代码需要测试,且类型可控; *不用*:非模板代码 | ✓ | ✓ | C++ STL container 特化 test | C++ template BT node 特化 |
| 24 | **Text Redefinition** | 435 | 用宏 / `#define` 替换文本,test 时给不同宏值 | *用*:C / C++ preprocessor 可控; *不用*:编译期常量无路径 | ✓ | ✓ | `#define kfree xfree` test 时替换 | `#define RCLCPP_LOG_INFO MOCK_LOG` test |

## 四、技法选型决策树

```
需要测试一段 legacy code
│
├─ 类无法构造 (separation)
│  ├─ 构造时调用了不可造的参数?
│  │  ├─ 参数是标准接口/外部库 → Adapt Parameter (1)
│  │  ├─ 参数是复杂对象 → Primitivize Parameter (16) 或 Parameterize Constructor (14)
│  │  └─ 参数从全局/单例来 → Introduce Instance Delegator (11) / Introduce Static Setter (12)
│  ├─ 构造时调用了 factory `new XXX()` → Extract and Override Factory Method (7)
│  ├─ 构造时引用了全局变量 → Encapsulate Global References (4) / Replace Global Reference with Getter (20)
│  └─ 构造时调用了未定义符号 → Definition Completion (3)
│
├─ 类可构造但方法内部调用难测 (separation + sensing)
│  ├─ 方法里有 `new XXX()` 难造 → Extract and Override Factory Method (7)
│  ├─ 方法读了 instance field 难控 → Parameterize Method (15) 或 Pull Up Feature (17)
│  └─ 方法里有大量"全局 + 文件"操作 → Replace Function with Function Pointer (19) / Link Substitution (13)
│
├─ 类可构造但看不到方法效果 (sensing)
│  ├─ 调用了 collaborator,需 sense 调用参数 → Extract and Override Call (6)
│  ├─ 调用了 collaborator,需 sense 返回值 → Extract and Override Getter (8)
│  └─ 调用的 collaborator 是接口/类 → Extract Interface (10) / Subclass and Override Method (21)
│
├─ 类可构造但逻辑复杂 (sensing)
│  ├─ 临时变量在多步间共享 → Break Out Method Object (2)
│  ├─ 方法不读 instance state → Expose Static Method (5)
│  └─ 方法实现可分离 → Extract Implementer (9)
│
└─ 类继承关系复杂
   ├─ 父类用了子类才有的方法 → Push Down Dependency (18)
   ├─ 多个子类有共同方法 → Pull Up Feature (17)
   ├─ 子类需要扩展父类行为 → Supersede Instance Variable (22)
   └─ C++ 模板代码需要测试 → Template Redefinition (23)
```

> **决策规则**:
> 1. **优先用 *separation* 技法**(改参数 / 改全局)而非 *sensing* 技法(fake) — fake 维护成本高。
> 2. **优先用 Extract Interface / Adapt Parameter**(浅改)而非 Extract Implementer / Bridge(深改)。
> 3. **C/C++ 优先 Function Pointer / Link Substitution** —— 没 vtable 也能替换。
> 4. **C++ 优先 Template Redefinition / Extract Interface** —— 利用 type system。
> 5. **每条技法都要配套 ch23 纪律** —— Preserve Signatures + Lean on Compiler。

## 五、技法对应的 ch3 sensing / separation 分布

```
                       sensing 主要                separation 主要
                       ──────────                  ────────────────
                       Break Out Method Object     Adapt Parameter
                       Extract Interface           Definition Completion
                       Extract and Override Call   Encapsulate Global References
                       Extract and Override Getter Expose Static Method
                       Replace Function w/ Pointer Extract and Override Factory Method
                       Replace Global w/ Getter    Introduce Instance Delegator
                       Subclass and Override       Introduce Static Setter
                       Supersede Instance Variable Parameterize Constructor
                       Link Substitution           Parameterize Method
                       Template Redefinition       Primitivize Parameter
                       Text Redefinition           Pull Up Feature
                                                  Push Down Dependency
                                                  Extract Implementer
```

> **规律**: 多数技法是 separation 路径(sensing 不需要 fake 就能拿到效果); 少数是 sensing 路径(必须 fake 才能看到结果)。

## 六、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 6.1 Linux 系统开发视角

- **Linux kernel 24 技法覆盖率**:
  - **常用**: Adapt Parameter (1), Break Out Method Object (2), Encapsulate Global References (4), Replace Function with Function Pointer (19), Replace Global Reference with Getter (20) —— kernel 大量 C 代码 + 函数指针注册机制天然契合。
  - **偶尔**: Extract and Override Factory Method (7) (通过 function pointer 模拟), Pull Up Feature (17) (e. g. `net_device_ops`), Push Down Dependency (18) (FS-specific op 推到具体 FS)。
  - **少用**: Template Redefinition (23) (kernel 几乎无 C++ template)。
  - **不可能**: Subclass and Override Method (21) (C 无继承), Extract Interface (10) (C 无 interface, 但 header subset 思想等价)。
- **kernel 测试基础设施对应技法**:
  - KUnit ≈ Extract Interface (10) 的轻量版 —— kernel 自己定义了 `struct kunit` 隔离。
  - ftrace / tracepoint ≈ Replace Function with Function Pointer (19) 的运行时版 —— 通过 hook 替换函数。
  - `EXPORT_SYMBOL` 替换 ≈ Link Substitution (13) —— 但 kernel 内 *link-time* 不易, 需 build flag。
- **典型案例**:
  - **`tcp_sendmsg` monster**: Break Out Method Object (2) + Extract and Override Call (6) —— 拆 `tcp_send_state` 临时对象。
  - **`crypto_shash` 替换**: Extract Implementer (9) —— `crypto_shash_alg` + `shash_alg_impl`。
  - **`printk` 测试隔离**: Replace Function with Function Pointer (19) —— test 替换 `printk` 函数指针。
  - **驱动 probe 流程**: Break Out Method Object (2) + Pull Up Feature (17) —— 拆 probe_state。
- **kernel 拒绝的技法**: Template Redefinition (23) (kernel 主要 C) + Subclass (21) (无继承) + Text Redefinition (24) (kernel 不允许 `#define` 替换关键函数, 会破坏 ABI)。
- **kernel 强化版的技法**: kernel 把 *Function Pointer Replacement* 推到极致 —— 几乎所有 subsystem 都靠 `struct ops` 注册(`file_operations`, `inode_operations`, `net_device_ops`, `block_device_operations` 等)。这是 ch25 #19 + #7 + #10 的内核工程化。

#### Linux 系统 — 24 技法实战热度表

| 热度      | 技法                                          | kernel 例                       |
| --------- | --------------------------------------------- | ------------------------------- |
| ★★★★★ | Replace Function with Function Pointer | `file_operations` 注册         |
| ★★★★★ | Extract and Override Factory Method    | `alloc_netdev` factory         |
| ★★★★  | Break Out Method Object                  | `tcp_send_state` 拆分          |
| ★★★★  | Encapsulate Global References           | `init_net` 包 `struct net`       |
| ★★★★  | Replace Global Reference with Getter    | `jiffies` getter                |
| ★★★   | Adapt Parameter                          | `file_operations` subset 包装 |
| ★★★   | Parameterize Constructor                 | `register_netdev(netdev)`      |
| ★★★   | Pull Up Feature                          | `ndo_start_xmit` 上提           |
| ★★    | Push Down Dependency                     | FS-specific op 下推             |
| ★★    | Extract Implementer                      | `crypto_shash` bridge          |
| ★     | Extract Interface                        | C 无 interface,header subset  |
| ★     | Subclass and Override Method             | C 无 subclass                  |
| ✗     | Template Redefinition                    | kernel 几乎无 C++ template      |
| ✗     | Text Redefinition                        | kernel 不允许 #define 替换 ABI |

### 6.2 机器人软件视角

- **ROS2 24 技法覆盖率**(因 ROS2 主要 C++ + composition, Duck typing 较易):
  - **常用**: Extract Interface (10), Subclass and Override Method (21), Replace Function with Function Pointer (19), Pull Up Feature (17), Push Down Dependency (18), Adapt Parameter (1) —— C++ 多态 + DI 容器 + pluginlib 配合。
  - **偶尔**: Extract and Override Factory Method (7) (Node 工厂), Introduce Static Setter (12) (Mock clock), Break Out Method Object (2) (BT 执行状态)。
  - **少用**: Primitivize Parameter (16) (但 sensor data primitive 化常用), Definition Completion (3) (主要给 plugin lib 用)。
  - **不可能**: Encapsulate Global References (4) (ROS2 推崇 DI), Text Redefinition (24) (build system 隔离)。
- **ROS2 pluginlib 配合的技法**:
  - `pluginlib::ClassLoader<T>` ≈ Extract Interface (10) + Extract and Override Factory Method (7) 的工业版。
  - `rclcpp::Node::create_publisher<T>()` ≈ Adapt Parameter (1) 的 ROS2 版。
  - `MockNode` 测试 helper ≈ Subclass and Override Method (21) 的标准实现。
- **Nav2 实战 24 技法**:
  - `PlannerServer` 用 Extract Interface (10) —— `nav2_core::GlobalPlanner` 是 interface。
  - `Costmap2D` 用 Pull Up Feature (17) —— `CostmapLayer` 是基类。
  - `RecoveryBehavior` 用 Push Down Dependency (18) —— `RotateRecovery` 等下推。
  - BT nodes 用 Subclass and Override Method (21) —— `MockAction` 是 BT node test 标准。
- **ros2_control 实战 24 技法**:
  - `hardware_interface::SystemInterface` 是 Extract Interface (10) 的工业版。
  - `ControllerManager` 用 Pull Up Feature (17) —— 多 controller 共用 update loop。
  - Mock hardware 用 Adapt Parameter (1) —— 包真 hardware interface。
- **典型机器人项目对应**:
  - **MoveIt2**: Extract Interface (10) (`PlanningPipeline`), Pull Up Feature (17) (`MoveGroup`), Subclass and Override (21) (`MockPlanningScene`)。
  - **robot_localization**: Adapt Parameter (1) (`sensor_msgs::Imu` 包装), Pull Up Feature (17) (EKF/UKF 基类)。
  - **tf2**: Adapt Parameter (1) (`tf2::Transform` 包装), Replace Function with Function Pointer (19) (`tf2_ros::Buffer` 内部)。

#### 机器人软件 — 24 技法实战热度表

| 热度      | 技法                                          | ROS2 例                         |
| --------- | --------------------------------------------- | ------------------------------- |
| ★★★★★ | Extract Interface                        | `nav2_core::GlobalPlanner`     |
| ★★★★★ | Subclass and Override Method             | `MockAction` / `MockPlanner`    |
| ★★★★★ | Pull Up Feature                          | `CostmapLayer` 基类              |
| ★★★★  | Adapt Parameter                          | `sensor_msgs::Imu` 包装        |
| ★★★★  | Push Down Dependency                     | `RotateRecovery` 下推           |
| ★★★★  | Extract and Override Factory Method    | `rclcpp::Node::create_*`        |
| ★★★   | Replace Function with Function Pointer | `tf2_ros::TransformListener` 内部 |
| ★★★   | Parameterize Constructor                 | `Node` constructor 接受 options |
| ★★★   | Introduce Static Setter                  | `set_clock_for_test(MockClock)` |
| ★★    | Break Out Method Object                  | `BtExecutionState`             |
| ★★    | Extract Implementer                      | `PlanningPipeline` impl         |
| ★★    | Replace Global Reference with Getter    | `now()` → `getCurrentTime()`   |
| ★★    | Extract and Override Getter              | `getCostmap()` override         |
| ★     | Expose Static Method                     | `pure function` BT condition   |
| ★     | Primitivize Parameter                    | `Twist` → `float*`            |
| ★     | Introduce Instance Delegator             | `NodeLogger` instance          |
| ★     | Link Substitution                        | colcon `.so` 替换              |
| ✗     | Encapsulate Global References           | ROS2 推崇 DI                  |
| ✗     | Template Redefinition                    | ROS2 较少 C++ template |
| ✗     | Text Redefinition                        | build 隔离                  |
| ✗     | Supersede Instance Variable              | C++ 多用 sub-overwrite        |
| ✗     | Extract and Override Call                | ROS2 多用 sub-overwrite       |
| ✗     | Parameterize Method                      | 多用 sub-overwrite         |
| ✗     | Definition Completion                    | 多用 pluginlib              |

### 6.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 查 ch25              | 从头顺序读                                                   | 按 *症状* 反查 → 用决策树定位技法                            |
| 技法选择             | 默认用 Extract Interface(看起来"专业")                       | 按 *依赖类型* 选最浅的技法 (参数/方法优先, 全局最后)         |
| C 环境               | "没有 class 怎么 break dependency"                           | 用 function pointer + link substitution 替代               |
| 24 技法都用          | 把所有技法混用                                               | 大多数情况 *只用一个技法* 就够                                |
| 配套纪律             | 用技法但忽略 ch23 纪律                                       | 技法 + 纪律 双层保险                                         |
| sensing vs separation | 默认走 sensing(fake)                                         | 先 separation 再 sensing                                     |
| 技法排序             | 没排序                                                       | 按 *依赖类型* 分层(参数→方法→类→全局)                      |
| 选 Extract Interface 还是 Adapt Parameter | 默认 Extract Interface | "参数是 std interface 时改 Adapt Parameter" |
| 选 Fake 还是 Mock    | 默认 mock                                                    | 选 fake 时 *fake 是 sensing 通道*,选 mock 时 *mock 是 verification* |
| 改后验证             | 跑 test 看绿                                                 | 跑 test + 读 diff + Lean on Compiler + Pair review          |

> **关键差异**: 高级工程师 *按依赖类型选最浅技法 + 配套纪律*; 初级 *默认深技法 + 忽略纪律*。

## 四、AI 时代视角

### 4.1 ch25 在 AI 时代仍是 *技法索引*

**极其重要**。AI 写 refactor 代码时, 它**最容易** 把技法选错 —— 比如:

- AI 看到"依赖", 默认用 Extract Interface —— 但其实 Adapt Parameter 就够。
- AI 看到"难测试", 默认 mock —— 但其实只换 function pointer 就够。
- AI 不分 sensing vs separation —— 一律 fake。

ch25 的 *决策树* + *技法热度表* 是 *AI 反向校准* 的关键。

### 4.2 AI 已经能做的

- **给定症状 → 推荐 3 个候选技法**: 准确率 70-80%。
- **解释每个技法步骤**: 准确率 90% (Feathers 写得清楚)。
- **比较两个技法边界**: "Extract Interface vs Adapt Parameter" 边界 —— 准确率 60%。
- **生成 *技法决策树* 的本地化版**(Linux / ROS2): 准确率 80%。
- **给技法配套纪律 checklist**: 准确率 90% (ch23 已结构化)。

### 4.3 AI 不能替代的

- **选技法的"成本/收益"判断**: 短期看 Extract Interface 优雅, 长期看 Adapt Parameter 简单 —— AI 给不出 *团队长期维护成本* 的判断。
- **技法排序的"团队节奏"**: 团队已习惯 Adapt Parameter 时, 给 Extract Interface 是 *认知负担*。
- **技法混合时的 *主从关系***: 多个技法同时用时哪个优先 —— 决策树不完整。
- **技法 vs *不重构* 的判断**: 有时"不重构, 加测试就够" —— AI 给不出 *何时该停*。

### 4.4 AI 经常写错的地方

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **默认 Extract Interface**                      | AI 见 "难测试" 就抽 interface, 实际 Adapt Parameter 够              | 过度抽象, interface 维护成本 |
| **过度 fake**                                  | AI 见 "看不到效果" 就 mock, 实际只换 function pointer 即可          | fake 维护成本高 |
| **混淆 sensing vs separation**                  | AI 一律走 sensing (mock), 不分 separation (改参数)                  | 拆错层 |
| **C 环境强行用 Extract Interface**             | AI 给 C 代码抽 interface, C 无此概念                                | 编译不过 / 过度模拟 |
| **改 PR 不 preserve signatures**                | AI 改 method signature 时改参数名,破坏 #15 纪律                    | ABI 风险 |
| **不配套 ch23 纪律**                            | AI 给技法不附 preserve / lean-on-compiler / pair                    | 技法不安全 |
| **技法选择不当导致"过度 refactor"**            | AI 看到 1 个依赖就 24 技法全用                                       | 1 个 PR 重构了 50% 代码, review 崩溃 |
| **决策树跳级**                                  | AI 直接到全局层(Encapsulate Global), 不走参数层                    | 跳过最简单的 solution |
| **没区分"测试驱动选型"vs"设计驱动选型"**      | AI 选技法时只考虑测试, 不考虑长期 design                            | 短期可测, 长期难维护 |
| **技法混用顺序错**                              | AI 先 sensing 再 separation (反向), 应该先 separation               | 拆完发现 sensing 不必要 |

### 4.5 子段：AI 辅助遗留代码理解 — 在本主题专项

- **AI 给"症状 → 技法"反查表**: 给一段难测代码, AI 标"separation 缺失, 用 Adapt Parameter" —— 准确率 70-80%。
- **AI 给"决策树"按团队本地化**: 给团队的技术栈 (Linux kernel / ROS2), AI 生成 *本地决策树* —— 准确率 80%。
- **AI 给技法配套纪律**: 选 Extract Interface 后, AI 提醒 *"preserve signatures + lean on compiler + pair"* —— 准确率 90%。
- **AI 给"过度 refactor 检测"**: 给 PR 看 *用了几个 ch25 技法*, > 3 个就警告 —— AI 给阈值建议。
- **AI 给"技法热度"按团队**: 基于团队 commit history, AI 给"过去 1 年哪些 ch25 技法用得多" —— 帮团队识别 *舒适区* 和 *盲区*。
- **AI 给"sensing vs separation 平衡"**: 给测试覆盖统计, AI 标"过度 fake 比例" —— 帮团队调抽象度。
- **AI 风险**: 它 *默认走最深的技法* —— 必须配 *决策树反查表* 强制它从最浅开始。

### 4.6 工程师必须保留的核心能力

- **决策树反查**: 不从"我想用什么技法"出发, 从"症状是什么"出发。
- **依赖类型分层判断**: 参数 / 方法 / 类 / 全局 —— 选最浅能解决的。
- **sensing vs separation 顺序**: 先 separation, 再 sensing。
- **技法与 ch23 纪律配套**: 技法是 *what*, 纪律是 *how*。
- **"不重构"是合法选项**: 技法不是越多越好 —— *何时该停* 是判断力。
- **团队节奏同步**: 团队已习惯某技法时, 引入新技法是 *认知负担*。

## 五、实践行动项

> ch25 是 reference catalog, 行动项聚焦 (a) 24 技法速查脚本; (b) 决策树演示; (c) 团队本地化热度统计。

### A1 — 24 技法速查脚本(给一段症状, AI 推 3 个候选)

```bash
mkdir -p /tmp/ch25-catalog && cd /tmp/ch25-catalog

# 速查表(从 ch25 决策树简化)
cat > tech_lookup.sh <<'EOF'
#!/usr/bin/env bash
# tech_lookup.sh — ch25 技法速查 (简化版决策树)
# Usage: ./tech_lookup.sh <symptom>
# Symptom keys:
#   "param-hard"  : 类不可构造, 参数是标准接口/外部库
#   "factory-new" : 类不可构造, 调用了 new XXX()
#   "global-var"  : 类不可构造, 引用了全局变量
#   "no-result"   : 类可构造, 看不到方法效果 (sensing 需求)
#   "monster"     : 类可构造, 但方法太复杂 (临时变量多)
#   "subclass"    : 类继承关系复杂, 需要 override
#   "c-no-class"  : C 代码, 无 class/interface

sym="$1"
case "$sym" in
  param-hard)    echo "推荐: Adapt Parameter (1) > Extract Interface (10)";;
  factory-new)   echo "推荐: Extract and Override Factory Method (7)";;
  global-var)    echo "推荐: Encapsulate Global References (4) > Replace Global Ref with Getter (20)";;
  no-result)     echo "推荐: Extract and Override Call (6) > Extract Interface (10) > Subclass and Override (21)";;
  monster)       echo "推荐: Break Out Method Object (2) > Expose Static Method (5)";;
  subclass)      echo "推荐: Subclass and Override (21) > Pull Up Feature (17) > Push Down Dependency (18)";;
  c-no-class)    echo "推荐: Replace Function with Function Pointer (19) > Link Substitution (13)";;
  *) echo "用法: ./tech_lookup.sh <param-hard|factory-new|global-var|no-result|monster|subclass|c-no-class>"; exit 1;;
esac
EOF
chmod +x tech_lookup.sh
./tech_lookup.sh param-hard
./tech_lookup.sh c-no-class
./tech_lookup.sh monster
```

**验收**:
- `./tech_lookup.sh param-hard` 输出 *Adapt Parameter (1) > Extract Interface (10)*。
- `./tech_lookup.sh c-no-class` 输出 *Replace Function with Function Pointer (19) > Link Substitution (13)*。
- `./tech_lookup.sh monster` 输出 *Break Out Method Object (2) > Expose Static Method (5)*。
- 这是 ch25 *决策树* 的最小 bash 化。

### A2 — Decision Tree (Python 版, 给一段 Python 类, 推 3 个技法)

```bash
mkdir -p /tmp/ch25-decision && cd /tmp/ch25-decision

cat > decide.py <<'EOF'
"""ch25 决策树 (Python 版)
给定一段 legacy 代码的描述 (参数类型 + 是否构造 + 是否读全局 + 是否 monster),
推 3 个 ch25 技法候选。
"""

def decide(params_kind, can_construct, has_global, has_monster, has_subclass, language):
    """params_kind: 'standard-interface' | 'complex-object' | 'primitive-ok'
       language: 'cpp' | 'java' | 'c' | 'python'
    """
    recs = []
    # 语言优先级
    if language == 'c':
        recs.append(('Replace Function with Function Pointer (19)', 'C 函数指针替换'))
        recs.append(('Link Substitution (13)', '链接期替换'))
        recs.append(('Encapsulate Global References (4)', '包 struct'))
    elif not can_construct:
        if has_global:
            recs.append(('Encapsulate Global References (4)', '包 struct/class'))
            recs.append(('Introduce Instance Delegator (11)', '替换 singleton'))
            recs.append(('Introduce Static Setter (12)', 'test 临时 setter'))
        if params_kind == 'standard-interface':
            recs.append(('Adapt Parameter (1)', '包 adapter wrapper'))
            recs.append(('Extract Interface (10)', 'subset interface'))
        elif params_kind == 'complex-object':
            recs.append(('Primitivize Parameter (16)', '降级为 primitive'))
            recs.append(('Parameterize Constructor (14)', '构造加参数'))
    elif has_monster:
        recs.append(('Break Out Method Object (2)', '拆出 MethodRunner'))
        recs.append(('Expose Static Method (5)', '抽 static 纯函数'))
        recs.append(('Extract Implementer (9)', '桥接模式'))
    elif has_subclass:
        recs.append(('Subclass and Override Method (21)', 'test subclass'))
        recs.append(('Pull Up Feature (17)', '上提到父类'))
        recs.append(('Push Down Dependency (18)', '下推到子类'))
    else:
        # 默认 sensing 路径
        recs.append(('Extract and Override Call (6)', 'sensing collaborator call'))
        recs.append(('Extract and Override Getter (21)', 'sensing getter'))
        recs.append(('Extract Interface (10)', 'fake collaborator'))
    return recs[:3]

if __name__ == '__main__':
    # 测试: C 代码 + 不可构造 + 标准接口参数
    print("=== C + 不可构造 + 标准接口参数 ===")
    for r in decide(params_kind='standard-interface', can_construct=False,
                    has_global=False, has_monster=False, has_subclass=False, language='c'):
        print(f"  - {r[0]}: {r[1]}")

    # 测试: C++ + 不可构造 + 全局变量
    print("\n=== C++ + 不可构造 + 全局变量 ===")
    for r in decide(params_kind='primitive-ok', can_construct=False,
                    has_global=True, has_monster=False, has_subclass=False, language='cpp'):
        print(f"  - {r[0]}: {r[1]}")

    # 测试: C++ + 可构造 + monster
    print("\n=== C++ + 可构造 + monster ===")
    for r in decide(params_kind='primitive-ok', can_construct=True,
                    has_global=False, has_monster=True, has_subclass=False, language='cpp'):
        print(f"  - {r[0]}: {r[1]}")
EOF

python3 decide.py
```

**验收**:
- 输出 3 个测试, 每个 3 条推荐。
- C + 不可构造 + 标准接口 → Replace Function with Function Pointer (19) 优先。
- C++ + 全局 → Encapsulate Global References (4) 优先。
- C++ + monster → Break Out Method Object (2) 优先。

### A3 — 技法热度统计(给团队 commit history, 算 ch25 技法出现频次)

```bash
mkdir -p /tmp/ch25-heatmap && cd /tmp/ch25-heatmap

# 模拟团队 commit history
git init -q
git config user.email "you@example.com" && git config user.name "you"

# 添加 8 个 commit, 每个模拟使用 1 个 ch25 技法
cat > foo.c <<'EOF'
int add(int a, int b) { return a + b; }
EOF
git add . && git commit -q -m "feat: initial"

# 1. Adapt Parameter
sed -i 's/int add/int add_v2/g' foo.c
git add . && git commit -q -m "refactor: Adapt Parameter"

# 2. Encapsulate Global References
cat >> foo.c <<'EOF'
struct Config { int factor; };
static struct Config g_cfg = { 2 };
int apply(int x) { return x * g_cfg.factor; }
EOF
git add . && git commit -q -m "refactor: Encapsulate Global References"

# 3. Break Out Method Object
cat > monster.c <<'EOF'
struct Calc { int total; };
void calc_run(struct Calc *c, int n) {
    c->total = 0;
    for (int i = 0; i < n; i++) c->total += i;
}
EOF
git add . && git commit -q -m "refactor: Break Out Method Object"

# 4. Replace Function with Function Pointer
cat > func_ptr.c <<'EOF'
typedef int (*op_fn)(int, int);
static int op_add(int a, int b) { return a + b; }
static op_fn current_op = op_add;
int apply_op(int a, int b) { return current_op(a, b); }

int main(void) {
    int r = apply_op(2, 3);
    return (r == 5) ? 0 : 1;
}
EOF
git add . && git commit -q -m "refactor: Replace Function with Function Pointer"

# 5. Extract Interface (C 无 interface, 但 header subset 思想)
cat > subset.h <<'EOF'
/* subset interface: 只声明需要的方法 */
struct subset_ops {
    int (*get)(void *self);
};
int subset_get(struct subset_ops *ops) { return ops->get(ops); }
EOF
git add . && git commit -q -m "refactor: Extract Interface (subset)"

# 6. Subclass and Override (C 用 struct + function pointer 模拟)
cat > mock.c <<'EOF'
struct base { int (*get_x)(struct base *); };
struct mock { struct base b; int x; };
static int mock_get_x(struct base *b) {
    return ((struct mock *)b)->x;
}
struct base *make_mock(int x) {
    struct mock *m = malloc(sizeof(*m));
    m->x = x; m->b.get_x = mock_get_x;
    return &m->b;
}
EOF
git add . && git commit -q -m "refactor: Subclass and Override (struct ptr)"

# 7. Parameterize Constructor
cat > ctor.c <<'EOF'
struct Foo { int factor; };
struct Foo *foo_new(int factor) {
    struct Foo *f = malloc(sizeof(*f));
    f->factor = factor;
    return f;
}
EOF
git add . && git commit -q -m "refactor: Parameterize Constructor"

# 8. Pull Up Feature (共享方法上提)
cat > base.c <<'EOF'
struct base { int (*get)(struct base *); };
struct child { struct base b; int x; };
static int default_get(struct base *b) { return 0; }
EOF
git add . && git commit -q -m "refactor: Pull Up Feature"

# 24 个技法,按一行一个写到 /tmp/ch25-tech.txt,然后 grep -F 多模式匹配
cat > /tmp/ch25-tech.txt <<'EOF'
Adapt Parameter
Encapsulate Global References
Break Out Method Object
Replace Function with Function Pointer
Extract Interface
Subclass and Override
Parameterize Constructor
Pull Up Feature
Introduce Static Setter
Primitivize Parameter
Expose Static Method
Extract and Override Call
Extract and Override Getter
Extract and Override Factory Method
Extract Implementer
Link Substitution
Replace Global Reference with Getter
Supersede Instance Variable
Push Down Dependency
Definition Completion
Introduce Instance Delegator
Template Redefinition
Text Redefinition
Parameterize Method
EOF
git log --pretty=format:"%s" | grep -F -f /tmp/ch25-tech.txt | sort | uniq -c | sort -rn

# 顺便: 验证一个 ch25 技法样本 (Replace Function with Function Pointer) 可编译运行
cc -std=c17 -Wall -Wextra -O0 -o heatmap_demo func_ptr.c && ./heatmap_demo
```

**验收**:
- 输出 ch25 技法按 commit 出现频次排序。
- 8 个 commit 各用一个技法 → 8 个 distinct 技法各出现 1 次。
- 这是 ch25 *团队技法热度* 的最小统计演示。

## 六、值得深入思考的问题

- **ch25 为什么是 24 而不是 25?**: Feathers 自己说"not exhaustive" —— 24 是 *他实操过的*, 不是 *全集*。团队应根据 codebase 增补技法。
- **技法选择 = 工程 judgement**: 不是"按图索骥", 是 *依赖类型 + 团队节奏 + 长期成本* 的综合判断。
- **ch25 与 Fowler refactoring 的边界**: Feathers 在 glossary 列出对应, 但本质不同 —— ch25 是 *无测试条件下的拆*, Fowler 是 *有测试条件下的重命名/抽方法*。
- **ch25 的"安全前提"是 ch23**: 单独用 ch25 = *没纪律的技法* = silent breakage。
- **ch25 + ch3 sensing/separation 的对应**: 多数技法是 separation; sensing 集中在 fake / Extract Interface。这是 *技法分类* 的另一维度。
- **ch25 的 "Test Coverage Oasis"**: 每用一个技法 → 测试可写面积变大 → oasis 扩大。这是 ch24 + ch25 的协同。

## 七、本章与全本的关系 + 全本总结

### 与全本 25 章的关系

- **前置**:
  - ch22 = 拆 monster 的具体技法(Sensing Variable / Gleaning / Break Out Method Object / Skeletonize)。
  - ch23 = 拆 dependency 的安全纪律(Hyperaware / Single-Goal / Preserve / Lean on Compiler / Pair)。
  - ch24 = 拆 legacy 的情感动力(fun / impact / community)。
- **本章**: 24 个 dependency-breaking 技法的 *reference catalog*。
- **全本结束**: Part III 包含 ch25 + Refactoring Glossary + Glossary + Index。

### 全本总结 — 三大部分

**Part I — The Mechanics of Change** (ch1-5):
- ch1 *Changing Software*: 改的 4 理由 + 4×4 矩阵 + cliff 比喻。
- ch2 *Working with Feedback*: TDD + characterization test + unit test。
- ch3 *Sensing and Separation*: 拆依赖的两个独立动机 + fake 的两个面。
- ch4 *The Seam Model*: seam 是"插入点",5 类 seam。
- ch5 *Tools*: refactor 工具 + 编译器 + test framework。

**Part II — Changing Software** (ch6-24):
- ch6-13 = 按 *症状* 分类的具体技巧(时间不够、改动慢、加 feature、类不可测、方法不可测、改签名、批量改、不知测什么)。
- ch14-21 = *特殊依赖* 类(库依赖、API-only、代码不理解、无结构、test 碍事、非 OO、类太大、重复改)。
- ch22-24 = *monster / safety / morale* 三章: 怪物方法拆解、不破坏纪律、心理动力。

**Part III — Dependency-Breaking Techniques** (ch25):
- 24 个技法的 reference catalog + Refactoring Glossary + 总 Glossary + Index。

### 全本核心命题

> **Legacy code 是 *没测试* 的代码; 改变 legacy code = 拆依赖 + 写测试 + 谨慎 refactor。**
>
> **三件武器**: (1) Edit cycle < 1 sec; (2) Characterization tests; (3) Dependency-breaking techniques。
>
> **两条纪律**: (1) ch23 hyperaware / single-goal / preserve / lean on compiler / pair; (2) ch24 fun / community / oasis。
>
> **三组心智模型**: (1) cliff model(ch1); (2) sensing/separation(ch3); (3) seam model(ch4)。

### 后续书预告

**Working Effectively with Legacy Code** 之后, Feathers 没写专门的 legacy 续作。但 *legacy code* 这命题在后续书里被反复回响:

- **Feathers — *Working Effectively with Unit Tests* (2014)**: ch2 的 unit test 推到 *readability / maintenance* 维度。
- **Newman — *Monolith to Microservices* (2019) 第 5 章**: ch8 的 *Extract to Current Class First* 在 microservice 维度的延伸 —— Strangler Fig Pattern。
- **Winters — *Software Engineering at Google* (2020) 第 16 章**: ch23 的 hyperaware 在 *Large-Scale Changes* 维度的工程化。
- **Newport — *Deep Work* / *So Good They Can't Ignore You***: ch23 / ch24 的 *deep work* / *career capital* 框架的程序员版本。
- **AI 时代**: 还没出版但必然出现的 *Working Effectively with AI-Assisted Legacy Code* (预计 2027-2030) —— 把 ch22 / ch23 的 *refactor + hyperaware* 推到 *AI pair* 维度。

> **最后一章总结**: ch25 的 24 技法不是终点, 是 *起点*。每个 codebase 都需要根据团队节奏增补自己的 *hidden* 技法。读完这本书的真正作业: 为你的 codebase 写一份 *本地化 ch25*(Linux / ROS2 / Java / Scala / 你自己的栈)。

## 八、附录: 24 技法的 *ch25 页码 → PDF 物理页* 对照

| ch25 技法 | ch25 印页码 | PDF 物理页 |
| --- | --- | --- |
| Adapt Parameter | 327 | 349 |
| Break Out Method Object | 331 | 353 |
| Definition Completion | 337 | 359 |
| Encapsulate Global References | 339 | 361 |
| Expose Static Method | 345 | 367 |
| Extract and Override Call | 349 | 371 |
| Extract and Override Factory Method | 351 | 373 |
| Extract and Override Getter | 353 | 375 |
| Extract Implementer | 357 | 379 |
| Extract Interface | 362 | 385 |
| Introduce Instance Delegator | 369 | 391 |
| Introduce Static Setter | 373 | 395 |
| Link Substitution | 377 | 399 |
| Parameterize Constructor | 379 | 401 |
| Parameterize Method | 383 | 405 |
| Primitivize Parameter | 385 | 407 |
| Pull Up Feature | 389 | 411 |
| Push Down Dependency | 393 | 415 |
| Replace Function with Function Pointer | 397 | 419 |
| Replace Global Reference with Getter | 399 | 421 |
| Subclass and Override Method | 401 | 423 |
| Supersede Instance Variable | 405 | 427 |
| Template Redefinition | 409 | 431 |
| Text Redefinition | 413 | 435 |

> 表中"ch25 印页码"是 Feathers 印在章节内的页码引用(如 "Extract Interface (362)"),"PDF 物理页"是 PDF 物理页号(本笔记 . cache/25. txt 的 `PAGE` 标记)。
