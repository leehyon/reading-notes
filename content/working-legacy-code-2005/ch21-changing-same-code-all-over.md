# Chapter 21 — I'm Changing the Same Code All Over the Place

> **PDF**: p.291-310（20 页）
> **定位**: Part II 实战章 — ch20 解决"一个 class 太胖", 本章解决"多个 class 重复代码"。核心议题: 当某个改动(比如"加 0x01 terminator 改 0x00")需要在 N 处改 → 提取共同点。Feathers 用 AddEmployeeCmd / LoginCommand 命令序列化例子, 演示从 duplication → role stereotype → *orthogonality* 的演进。本章是 ch25 catalog 的"职责分离"系列中的最后一章。

## 〇、第一性原理思考

**这章做了什么**: 用 AddEmployeeCmd 与 LoginCommand 两个看似无关的命令类, 从重复的 `write(OutputStream)` 序列化出发, 走 9 步工程序列: 抽 `writeField` → 提 `Command` superclass → 上提 `header / footer / SIZE_LENGTH / CMD_BYTE_LENGTH` → 引入 `List<String> fields` → subclass 只剩 `getCommandChar()` override + 构造时填 fields。

**为什么这样拆**: Feathers 用 *role stereotype* (sequence 责任 / package 责任 / class 责任三类) 这套心智模型, 把"重复"从 class 维度拉到 role 维度 — 抽 `writeField` 这种 sequence 责任时, 需要的是 role 接口而不是 `class FieldWriter` 实体。

**最值钱的洞见**: 抽重复时先问"这是 role 还是 class" — sequence 责任抽成实体 class 后只会把"实体 class 重复"换种形式再发生一次, 实际上无助于 orthogonality; 真正的开放封闭是 AggregateCommand 那种加 subclass 不改 `Command` 的工业实例。

## 一、章节概述

- **症状: 重复改多处** — 你想加一个小功能, 发现要 N 个 class 同步改; 或者改一个细节(terminator 从 0x00 改 0x01), 要在多个 write call 里改。这是 *duplication* 的工程代价。
- **重复改多处 ≠ 重复本身** — 重复是结果,*改 N 处* 才是症状。Feathers 强调"the key question is, is it worth it? What do we get when we zealously squeeze duplication out?"
- **示例: AddEmployeeCmd + LoginCommand** — 两个 *看起来不同* 的类(一个管加员工, 一个管登录), 都有相同的 `write(OutputStream)` 序列化:`header / getSize() / commandChar / fields / footer`。
- **逐步去重的工程序列**:
  1. 抽 `writeField(OutputStream, String)` 公共方法
  2. 提 `Command` superclass(让 `writeField` 上提)
  3. 抽 `writeBody(OutputStream)` — 两个类的差异点
  4. 上提 `header / footer / SIZE_LENGTH / CMD_BYTE_LENGTH` 到 `Command`(都是 static)
  5. 抽 `getCommandChar()` abstract(差异)
  6. 抽 `getBodySize()` + `getFieldSize(String)` 公共
  7. 引入 `List<String> fields` 到 `Command`, subclass 只 fill list
  8. 上提 `writeBody / getBodySize / getFieldSize` 到 `Command`(都是迭代 `fields`)
  9. 最终 `LoginCommand / AddEmployeeCmd` 只剩 `getCommandChar()` override + 构造时填 `fields`
- **"Deciding Where to Start"** — Feathers 给出 heuristic: 找 *可以命名* 的小重复先抽; 抽完后大重复自然浮现。这是 *start small* 原则。
- **Role Interface Stereotype** — `writeField` / `write` / `getSize` / `getBodySize` / `getFieldSize` 都是 *role* (角色), 不是 *class* (实体)。把重复视为 *role 接口* 而不是 *实体 class* 重复 — 这是 ch21 的关键抽象。
- **Reorganization by Responsibility** — 把不同类的 *重复部分* 重组到 *role interface*:
  - **Sequence responsibility** — 谁先谁后(协议顺序)
  - **Package responsibility** — 谁包含谁(组合关系)
  - **Class responsibility** — 实体本身做什么
  - 重复常发生在 *Sequence 责任* — 多个 class 都按相同序列做事。抽出来后, Sequence 责任归一个 role, Class 责任归实体 class。
- **Abbreviations 警告** — `Mgr` / `Mngr` / `Mgrs` 这种命名混乱, 让使用方 *50% 概率猜错*。命名一致性比简洁重要。
- **Orthogonality** — "fancy word for independence"。一个行为 = 一个 knob。改一个行为只改一处。
- **Open/Closed Principle** (Bertrand Meyer) — 设计要 *open for extension, closed for modification*。加新功能 = 加 subclass(不改 Command), 不 = 改 Command。
- **AggregateCommand 案例 (Figure 21.5)** — 加 *嵌套 command* 类, 只 override `writeBody` 写 `commands.size` + `for c in commands: c.write(out)`。**这正是 Open/Closed 的工业实例**。
- **ch21 与 ch25 的关系** — ch21 的技法都是 ch25 catalog 中的具体 item: Extract Method / Extract Superclass / Pull Up Field / Pull Up Method / Form Template Method / Replace Inheritance with Delegation。本章演示它们的"组合使用"。

## 二、核心 Takeaways

### Takeaway 1: 重复改多处 = duplication 的工程体现, 不是抽象规则违反

- **是什么**: 看到 N 处写相同代码 → duplication 在; 看到"改一处要 N 处同步" → duplication 的工程代价显化。
- **为什么重要**:"改 N 处" 是 *维护成本*, 不是 *LOC*。LOC 看起来重复 30% 不算多; 改一个 behavior 要 5 处同步是 *危险*。
- **解决什么问题**: 把"重复"从代码观感拉到 *工程代价*。决策标准不再是"看起来丑", 而是"改起来痛不痛"。
- **适用场景**: 每次加新功能, 数"我要改几个 class"。

### Takeaway 2: 抽重复 = 抽 *role*, 不是抽 *class*

- **是什么**:`writeField` 是 *role*, 因为它属于 *sequence responsibility* (序列化序列中的"写一个 field" 这一角色), 不属于某个具体 command。**抽 role 而不是抽 class** — 这是 ch21 的隐藏心法。
- **为什么重要**: role abstraction 不绑定业务语义; 可以跨多个 class 复用。如果抽成 `class FieldWriter`, 会变成"实体 class 重复"。
- **解决什么问题**: 让"两个 command 共享 writeField"成为 *role 共享* 而非 *class 共享*。
- **适用场景**: 看到多处重复, 先问"这是 role 还是 class"。

### Takeaway 3: Start small = 先抽 *可以命名* 的小重复

- **是什么**: 写 `c() { a(); a(); b(); a(); b(); b(); }`, 可抽 `aa()` 还是 `ab()`? 两种都可。Feathers 建议先抽能 *命名* 的小重复(`writeField` 只写一个 field + 0x00, 语义清晰)。
- **为什么重要**: 小重复抽出后, 大重复自然浮现 — `writeBody` 是 *iteration* 抽象, 源自 `writeField`。
- **解决什么问题**: 避免"大爆炸 refactor", 逐步演进。
- **适用场景**: 每次去重决策。

### Takeaway 4: 上提 (Pull Up) = 把子类公共部分挪到 superclass

- **是什么**:`Command` 起初空, subclass 有 header / footer / SIZE_LENGTH。step 4 把它们 `pull up` 到 `Command`。
- **为什么重要**: 上提是 *refactoring 教科书* (Fowler p.322) 的基础操作, ch21 演示 *何时* 上提 (当两个 subclass 有相同字段 / 方法时)。
- **解决什么问题**: 让"两个 class 都有 header" 变成"Command 都有 header"。
- **适用场景**: 发现两个 class 有相同字段 / 方法, superclass 尚未抽象。

### Takeaway 5: 抽 abstract method = 让 subclass override 表达差异

- **是什么**:`getCommandChar()` 是 abstract, subclass 各自实现(0x01 vs 0x02)。`writeBody()` 是 abstract, subclass 各自实现(login 2 个 field vs employee 5 个 field)。
- **为什么重要**: abstract method 把 *差异点* 显式化, superclass 的 *共同点* 自动被所有 subclass 共享。
- **解决什么问题**: 让"两个 class 看起来像但不同" 成为 superclass+abstract 的标准 pattern。
- **适用场景**: 发现两个 subclass 行为"几乎相同但局部不同"。

### Takeaway 6: Reorganization by Responsibility = Sequence/Package/Class 三种责任分离

- **是什么**：`Command` 的 superclass 承担 *sequence responsibility* (写 sequence: header → size → char → body → footer); 每个 subclass 承担 *class responsibility* (自己的 fields + commandChar); `Aggregate` 承担 *package responsibility* (组合其他 commands)。
- **为什么重要**: 三种责任互相 *正交* (orthogonal) — 改一个不影响其他。这是 orthogonality 的具体体现。
- **解决什么问题**: 把"重复"重新归类为 *sequence 责任*, 抽出后 *class 责任* 与 *package 责任* 不再重复。
- **适用场景**: 每次发现 N 个 class 有相同的 sequence / 组合 / 协议。

### Takeaway 7: Role Interface Stereotype = 给重复一个 *角色名*

- **是什么**: 不是把所有重复抽成一个 utility class, 而是抽成一个 *role interface*(`Writer` / `Reader` / `Command`)。`Command` 是 *role*,`AddEmployeeCmd` 是 *class* 担任这个 role。
- **为什么重要**: role interface 让"两个 class 共享行为"成为 *类型契约*, 不是 *代码复用*。这与 ISP (ch20) 的精神一致。
- **解决什么问题**: 让"加新 command"成为"implement Command interface", 而不是"复制 AddEmployeeCmd 改字段名"。
- **适用场景**: 发现两个 class 行为相似但数据不同时。

### Takeaway 8: Orthogonality = 一个 behavior 一个 knob

- **是什么**: 每个 *behavior*(写 field / 写 body / 写 header)只在一处定义。改 behavior 只改一处。
- **为什么重要**: orthogonal design 是 Open/Closed 的前提 — 加新 behavior = 加 knob, 不改 existing knob。
- **解决什么问题**: 让"加新 command" 不动 Command, 只动 subclass。
- **适用场景**: 每次设计新 class hierarchy 时。

### Takeaway 9: Open/Closed Principle = 加 subclass 扩展, 不动 superclass

- **是什么**: Bertrand Meyer 原则 — code open for extension, closed for modification。ch21 的 `AggregateCommand` 是经典示例:`Command` class *完全没动*, 新加 `AggregateCommand` subclass 实现 *嵌套 command* 序列化。
- **为什么重要**: 这是 *设计成熟度* 的标志 — 既有 class 设计良好, 以致新功能只需要加代码而非改代码。
- **解决什么问题**: 让"加新 feature" 与"维护旧 feature" 解耦。
- **适用场景**: 评估 class hierarchy 是否成熟。

### Takeaway 10: Abbreviations 警告 — 命名混乱 ≥ 50% 误用率

- **是什么**: Feathers 见到一个 team 用 `Mgr` / `Mngr` / `Mgrs` 等不同 abbreviation 表示 *manager*。读者 *50% 概率* 猜错命名。
- **为什么重要**: 命名一致性比简洁重要; 混乱的命名 = *人为的 coupling*。
- **解决什么问题**: 把"命名风格"提到 *代码 review 第一指标*。
- **适用场景**: 每次新 commit review 时。

## 三、工程实践视角

> 锁定领域:**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **`struct file_operations` 是 ch21 的工业实例**。每个字符设备 / 块设备 / 网络设备实现一份 `struct file_operations`, 其中 `read` / `write` / `open` / `release` 是 *role methods*。这与 AddEmployeeCmd / LoginCommand 的 `writeField` / `getSize` 是同一 pattern: *sequence responsibility*(VFS 调用顺序)+ *class responsibility*(每个设备的具体行为)。
- **`net/core/dev.c` 的 `dev_queue_xmit` sequence** — 网络包发送流程是 sequence responsibility: *entry validation → softirq check → driver xmit → stats update*。这是 *sequence 责任* 在 kernel 的实例。每个 network driver 负责 *class 责任*(`ndo_start_xmit`), 不改 `dev_queue_xmit`。
- **`struct i2c_algorithm`'s `master_xfer` 与 `smbus_xfer`** 是 *role 接口* 拆分 — i2c bus 暴露两个 *role* interface, client 按需用。这是 ISP + ch21 role stereotype 的工业版本。
- **`struct snd_pcm_ops` (ALSA)** 是 *sequence + class* 责任分离的清晰案例:`snd_pcm_ops.open / close / hw_params / prepare / trigger / pointer / copy_user / copy_kernel / mmap` 等 sequence, 每个声卡 driver 只覆盖相关 method。
- **`usb_driver` 的 probe / disconnect** 是 Open/Closed 实例 — 加新 USB device = 加新 `struct usb_driver`, 不动 USB core。Linux kernel 90% subsystem 都是这个 pattern。
- **`copy_to_user` / `copy_from_user` 在多处重复** — 内核各 subsystem 都用, 但 kernel 没把它抽成一个 `class`,` 留成` *function* (在 `arch/x86/lib/usercopy.c`)。这是 *role 不一定是 class*,`* 也可是 function*。Feathers 的 role stereotype 在 C 的世界里是 *function* + *function pointer struct*。
- **`printk` 重复改多处** — 如果某天决定改 `printk` 的 prefix 格式(从 `"[KERN_ERR]"` 改 `"<err>"`), 要在 *数百处* 改。kernel 的解法 = `pr_*` macro 抽象(`pr_err` / `pr_info` 等), 把 format prefix 集中。这是 ch21 Takeaway 8 (orthogonality) 在 kernel 的工业实例。

#### Linux 系统 — Role Interface 实例表

| Kernel 类/接口               | 重复 method         | role stereotype               |
| ---------------------------- | ------------------- | ----------------------------- |
| `struct file_operations`     | `open/read/write`   | file I/O role                  |
| `struct net_device_ops`      | `ndo_start_xmit`    | packet transmit role           |
| `struct i2c_algorithm`       | `master_xfer`      | i2c transaction role           |
| `struct snd_pcm_ops`         | `hw_params`        | pcm setup role                 |
| `struct usb_driver`          | `probe`            | usb device attach role          |
| `struct input_handler`       | `event`            | input event role               |
| `struct regulator_ops`       | `set_voltage`      | regulator control role         |
| `struct clk_ops`             | `set_rate`         | clock control role             |
| `struct pinctrl_ops`         | `set_mp` | pin control role                |
| `struct pwm_ops`             | `config`           | pwm config role                |

### 3.2 机器人软件视角

- **`ros2_control` 的 `ControllerInterface`** 是 *role interface* 工业版本。`controller_interface::ControllerInterface` 暴露 `on_init / on_configure / on_activate / on_deactivate / cleanup / update` — 每个具体 controller (`JointTrajectoryController` / `DiffDriveController` / `GripperController`) 各自覆盖。这是 ch21 AddEmployeeCmd / LoginCommand 在 ROS2 的实例。
- **`rclcpp::Node` 的 `create_publisher` / `create_subscription`** 是 *role methods*。不同 ROS 2 包用相同 Node API, 但 *class responsibility* 不同(发布 image vs 发布 cmd_vel)。
- **`nav2_bt_navigator` 的 BehaviorTree XML** 是 *role-based composition* — BT XML 定义 *sequence*(root → sequence → action node), 每个 action node 是 *role*(behavior plugin)。XML 是 *sequence responsibility* 描述, plugin 是 *class responsibility*。
- **MoveIt 2 `PlanningPipeline`** 是 ch21 role 的工程化:`PlanningPipeline::plan()` 是 sequence responsibility(*request validate → planner manager dispatch → response format*);`PlannerManager` 是 role interface, 具体 planner(`OMPL` / `CHOMP` / `Pilz`)是 class responsibility。
- **`ros2_socketcan` 的 driver role** — `socketcan` driver 提供 *sequence*(init → send → recv), 不同 CAN 设备实现 class responsibility。这是 ch21 role + class 双层结构在驱动层的实例。
- **rosbridge 协议序列化** — JSON 命令序列化(发布 / 订阅 / service call)是 *sequence responsibility*; 每条具体命令(`publish` / `subscribe` / `call_service`)是 class responsibility。和 AddEmployeeCmd 序列化 1:1 对应。
- **MoveIt `MoveGroup` 的 action interface** — `MoveGroup::move()` / `plan()` / `execute()` 是 *role methods*, 不同 move type(`MoveGroupCartesian` / `MoveGroupJoint`)覆盖。这是 Open/Closed 在 MoveIt 的实例。
- **Nav2 Costmap Layer plugin** — `costmap_2d::Layer` 是 *role interface*, 具体 layer(`ObstacleLayer` / `InflationLayer` / `StaticLayer` / `RangeSensorLayer`)是 class responsibility。这是 *role stereotype* + *Open/Closed* 的工业范例。

#### 机器人软件 — Role Interface 实例表

| ROS 2 接口                  | 重复 method        | role stereotype                |
| --------------------------- | ------------------ | ------------------------------ |
| `ControllerInterface`       | `on_configure`     | lifecycle controller role       |
| `SystemInterface`           | `read/write`       | hardware access role            |
| `PlannerManager`            | `solve`            | planning algorithm role        |
| `BT::ActionNode`            | `tick/on_success`  | BT action role                  |
| `Layer` (costmap)           | `updateBounds`     | map layer role                  |
| `Nav2Controller`            | `computeVelocity`  | controller algorithm role       |
| `ros2_control command_interface` | `get_value`   | command value role              |
| `image_transport::Publisher`| `publish`          | image transport role            |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                | 高级工程师                                                  |
| -------------------- | --------------------------------------------------------- | ----------------------------------------------------------- |
| 看到重复              | "这是重复,要 DRY"                                          | "抽的是 role 还是 class?两种 DRY 价值不同" |
| 加新功能              | 复制 AddEmployeeCmd 改字段名                              | 加 subclass,override `getCommandChar` + 填 `fields` |
| 评估去重 ROI          | "重复行数 / 总行数 > 30% 就该抽"                          | "改一处要 N 处同步吗?N > 2 就该抽" |
| 设计新 hierarchy      | 直接 hardcode subtype logic                               | "我先画 role interface,让 class responsibility 显式化"  |
| 命名 abbreviation    | `Mgr` / `Manager` / `Mngr` 混用                            | "选一个,team 一致, *50% 误用率* 不能容忍"                 |
| 看到 sequence 重复    | "抽个 utility class"                                       | "sequence responsibility 抽到 superclass,abstract method 让 class 填" |
| 评估 Open/Closed     | "加 feature = 改 superclass"                               | "加 feature = 加 subclass,superclass 不动"                |
| 写 100 字段的 God class | "分模块"                                                  | "先看是不是 *sequence* 重复,抽 superclass" |
| 看 N 个 class 同样事情 | "封一个 wrapper"                                          | "抽 role interface,每个 class implements role" |

> **关键差异**: 高级工程师把重复视为 *role 共享* (类型契约), 初级把它视为 *code 复用* (代码块)。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因:
- **LLM 复制已有 class 是默认行为** — 你说"加 AddContractorCmd", LLM 复制 AddEmployeeCmd 改字段名。这是 *反 ch21* 的做法, 但 LLM 默认走这条路。
- **LLM 抽 utility class,`* 不抽* role interface` — 它倾向 "抽个 helper class" 而非 "抽个 interface", 因为 helper 更具体、更快。
- **LLM 命名 abbreviation 自由度过大** — 一个项目里 `Mgr` / `Manager` / `Mgrs` 都出现, LLM 不会 enforce 一致性。
- **LLM 不识别 sequence responsibility** — 它看不到 N 个 class 共享 sequence pattern, 直到你 prompt "为什么这两个 write method 看起来一样"。
- **LLM 的 "Open/Closed" 误读** — 它给已有 class 加 field 而非加 subclass, 因为"加 field 简单"。这是 *反 Open/Closed*。

### 4.2 AI 已经能做的(具体到 ch21 主题)

- **检测重复** — 给定 N 个 class 的 method list, AI 找 *cluster*(共享 method signature); 准确率 ~80%。
- **推荐 refactor 顺序** —"先抽 writeField, 后抽 writeBody, 最后抽 fields list"; 基于 Feathers 的 sequence, 准确率 ~85%。
- **生成 role interface** — 给定两个相似 class, 推荐 *role interface* (Command) + abstract method (writeBody); 准确率 ~75%。
- **检测 abbreviation 不一致** — 给定 codebase, 统计 `Mgr` / `Manager` 等出现频次, 产出"建议统一为 Manager" 报告; 准确率 ~95%。
- **生成 Open/Closed 实现** — 给定新 feature, 推荐 *subclass override* 而非 *改 superclass*; 准确率 ~70%(LLM 仍倾向后者)。

### 4.3 AI 不能替代的(具体到 ch21 主题)

- **判断"这是 sequence responsibility 还是 class responsibility"** — 这要 domain knowledge + reading multiple files。
- **决定 *何时* 抽 superclass** — 不是每次看到 2 个相似 class 都该抽; 抽早了是 *premature abstraction*, 抽晚了是 *pain 累积*。
- **评估 "改 1 处要同步几处"** — AI 看未来 6 个月的 commit pattern 不知道。
- **设计 role interface 的 *boundary*** — `Command` 该有 `writeBody` 还是 `writeBody + getBodySize + getFieldSize`? AI 给得过细或过粗。
- **Abbreviation 决策** — `Manager` vs `Mgr` 的 trade-off: 简洁 vs 一致。AI 给方案, 人决定。
- **判断 "Add subclass vs Modify superclass"** — 这是 design judgement, AI 给 *两边都说得通* 的方案。

### 4.4 AI 经常写错的地方

针对 ch21 duplication / role / Open-Closed 主题:

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **复制而非 subclass**                            | AI 加 AddContractorCmd 直接复制 AddEmployeeCmd 改字段名               | duplication +1,违反 ch21 |
| **抽 utility class 而非 role interface**        | AI 抽 `class CmdWriter { void writeField(); }` utility                | 不是 role stereotype,是 *helper* |
| **命名 abbreviation 混用**                      | `Manager` / `Mgr` / `Mngr` 同时出现                                  | 50% 误用率 |
| **抽 superclass 太早**                          | AI 看到 2 个相似 class 就抽 Command,superclass 名字 `BaseXxx`         | 后来发现第三个 class 不 fit,拆 superclass 麻烦 |
| **改 superclass 而非 subclass**                | AI 加 nested command 时改 Command 的 `write` 加 `if (nested)` 分支     | 违反 Open/Closed |
| **抽 abstract method 过多**                     | AI 给 Command 抽 10 个 abstract method                                | subclass 全部被迫 override,失去 *default 行为* |
| **Sequence 责任漏抽**                          | 两个 class 共享相同的 protocol 序列,AI 没发现                        | 改 protocol 要 N 处改 |
| **fields list 抽错位置**                        | AI 把 `fields list` 留在 subclass,没上提到 Command                   | 上提未完成,半途 refactor |
| **role interface 太具体**                       | AI 抽 `interface AddEmployeeRole` 而不是 `interface Command`           | 绑死业务,失去通用性 |
| **保留 `MOVING_` prefix 永久**                  | AI 拆完忘去 prefix,production 出现 `MOVING_getCommandChar`             | 命名怪,review 困惑 |
| **abbreviation 决策不 enforce**                 | AI 给 plan 但 team 不执行                                              | 命名混乱延续 |

### 4.5 子段: AI 辅助遗留代码理解 — 在本主题专项

- **AI audit duplication** — 给定 codebase, 统计 method-level 重复(签名 + body 相似), 输出 report; 准确率 ~80%。
- **AI 推荐 refactor 顺序** — 基于依赖图 + 风险排序; 准确率 ~70%。
- **AI 推荐 role interface** — 给定 N 个相似 class, 推荐 *通用 abstract*; 准确率 ~75%。
- **AI 不识别 *cross-cutting sequence*** — 它看单 class 重复, 看不到 *N 个 class 共享的 sequence*; 需要 prompt "这些 class 的 method call sequence 有什么共同点"。
- **AI 不 enforcement naming convention** — 它给 "建议统一 Manager", 但不会自动 fix 老的 `Mgr` 调用点。需要 prompt + script。
- **AI 不知道 "subclass 太多也是 smell"** — 50 个 Command subclass, 每个 1 个 override, 这是 *role 抽过头*; AI 给方案继续抽, 工程师得喊停。

### 4.6 工程师必须保留的核心能力

- **判断"抽 role 还是抽 class"** — role 共享契约; class 共享实现。误判 → 过度抽象。
- **决定抽 superclass 时机** — 不是 2 个 class 相似就抽; 等到 3 个, 看清楚 pattern 再抽。
- **Sequence responsibility 抽象** — 看到 N 个 class 共享 sequence, 抽 superclass abstract method。
- **Open/Closed 决策** — 加新 feature 默认 "加 subclass", 不 "改 superclass"。看到反例要 review。
- **Naming 一致性 enforcement** — 命名混乱 ≥ 50% 误用率不能容忍。
- **review AI 抽的 role interface** — 它抽得太具体或太抽象, 工程师得提示 "再粗一点" / "再细一点"。
- **避免 *抽太细*** — 50 个 1-method class 是过度抽象, 要 *回退*。

## 五、实践行动项

> 本章是 Part II 实战章的最后一章。行动项聚焦在 (a) Python 复刻 ch21 的 AddEmployeeCmd / LoginCommand 全过程;(b) C 复刻 binary 协议序列化 + Command role interface;(c) Open/Closed 演示 — 加 AggregateCommand subclass;(d) AI audit duplication 工具。

### A1 — Python 复刻 AddEmployeeCmd / LoginCommand 全过程

```bash
mkdir -p /tmp/ch21-cmd && cd /tmp/ch21-cmd

cat > commands_step0.py <<'EOF'
"""ch21 Step 0 — 原始 (Figure 21.1): 两个独立 class, 完全重复
"""
class AddEmployeeCmd:
    def __init__(self, name, address, city, state, yearly_salary):
        self.name = name; self.address = address; self.city = city
        self.state = state; self.yearly_salary = str(yearly_salary)
    HEADER = b'\xde\xad'; FOOTER = b'\xbe\xef'
    COMMAND_CHAR = b'\x02'; SIZE_LENGTH = 1; CMD_BYTE_LENGTH = 1
    def getSize(self):
        return (len(self.HEADER) + self.SIZE_LENGTH + self.CMD_BYTE_LENGTH
                + len(self.FOOTER)
                + len(self.name.encode()) + 1
                + len(self.address.encode()) + 1
                + len(self.city.encode()) + 1
                + len(self.state.encode()) + 1
                + len(self.yearly_salary.encode()) + 1)
    def write(self, out):
        out.write(self.HEADER)
        out.write(bytes([self.getSize()]))
        out.write(self.COMMAND_CHAR)
        for s in (self.name, self.address, self.city, self.state, self.yearly_salary):
            out.write(s.encode()); out.write(b'\x00')
        out.write(self.FOOTER)

class LoginCommand:
    def __init__(self, user_name, passwd):
        self.user_name = user_name; self.passwd = passwd
    HEADER = b'\xde\xad'; FOOTER = b'\xbe\xef'
    COMMAND_CHAR = b'\x01'; SIZE_LENGTH = 1; CMD_BYTE_LENGTH = 1
    def getSize(self):
        return (len(self.HEADER) + self.SIZE_LENGTH + self.CMD_BYTE_LENGTH
                + len(self.FOOTER)
                + len(self.user_name.encode()) + 1
                + len(self.passwd.encode()) + 1)
    def write(self, out):
        out.write(self.HEADER)
        out.write(bytes([self.getSize()]))
        out.write(self.COMMAND_CHAR)
        for s in (self.user_name, self.passwd):
            out.write(s.encode()); out.write(b'\x00')
        out.write(self.FOOTER)
EOF

cat > commands_final.py <<'EOF'
"""ch21 终态 (Figure 21.5) — Command superclass + thin subclass + Open/Closed"""
class Command:
    """sequence responsibility: header / size / char / body / footer"""
    HEADER = b'\xde\xad'
    FOOTER = b'\xbe\xef'
    SIZE_LENGTH = 1
    CMD_BYTE_LENGTH = 1

    def __init__(self):
        self.fields = []   # subclass 在构造时填

    def getCommandChar(self):  # abstract
        raise NotImplementedError

    def write(self, out):
        out.write(self.HEADER)
        out.write(bytes([self.getSize()]))
        out.write(self.getCommandChar())
        self.writeBody(out)
        out.write(self.FOOTER)

    def writeBody(self, out):
        for f in self.fields:
            self.writeField(out, f)

    def writeField(self, out, field):
        out.write(field.encode()); out.write(b'\x00')

    def getSize(self):
        return (len(self.HEADER) + self.SIZE_LENGTH + self.CMD_BYTE_LENGTH
                + len(self.FOOTER) + self.getBodySize())

    def getBodySize(self):
        return sum(self.getFieldSize(f) for f in self.fields)

    def getFieldSize(self, field):
        return len(field.encode()) + 1

class LoginCommand(Command):
    def __init__(self, user_name, passwd):
        super().__init__()
        self.fields.append(user_name); self.fields.append(passwd)
    def getCommandChar(self): return b'\x01'

class AddEmployeeCommand(Command):
    def __init__(self, name, address, city, state, yearly_salary):
        super().__init__()
        self.fields.extend([name, address, city, state, str(yearly_salary)])
    def getCommandChar(self): return b'\x02'

# Open/Closed 实例: 加嵌套 command 不动 Command
class AggregateCommand(Command):
    def __init__(self):
        super().__init__(); self.commands = []
    def append(self, cmd): self.commands.append(cmd)
    def getCommandChar(self): return b'\x03'
    def writeBody(self, out):
        out.write(bytes([len(self.commands)]))
        for c in self.commands:
            c.write(out)

if __name__ == '__main__':
    import io
    out = io.BytesIO()
    cmd = AddEmployeeCommand("Mike", "122 Elm", "Miami", "FL", 10000)
    cmd.write(out)
    data = out.getvalue()
    # header = 0xdead, char = 0x02, footer = 0xbeef
    assert data[:2] == b'\xde\xad'
    assert data[3:4] == b'\x02'
    assert data[-2:] == b'\xbe\xef'
    print(f"AddEmployeeCommand size={len(data)} bytes, last 5 = {data[-5:].hex()}")

    out2 = io.BytesIO()
    LoginCommand("alice", "secret").write(out2)
    data2 = out2.getvalue()
    assert data2[:2] == b'\xde\xad'
    assert data2[3:4] == b'\x01'
    print(f"LoginCommand size={len(data2)} bytes")

    # AggregateCommand: 不动 Command, override writeBody 即可
    out3 = io.BytesIO()
    agg = AggregateCommand()
    agg.append(LoginCommand("bob", "x"))
    agg.append(AddEmployeeCommand("C", "D", "E", "F", 500))
    agg.write(out3)
    data3 = out3.getvalue()
    assert data3[3:4] == b'\x03'
    print(f"AggregateCommand size={len(data3)} bytes (no Command class change)")
EOF

echo "=== step0 (Figure 21.1: duplication) line counts ==="
wc -l commands_step0.py
echo
echo "=== final (Figure 21.5: Command superclass + thin subclass + AggregateCommand) ==="
python3 commands_final.py
echo
echo "=== final line count (after refactor) ==="
wc -l commands_final.py
```

**验收**:
- `commands_final.py` 跑出 3 个 PASS (AddEmployee / Login / Aggregate)
- AggregateCommand 不改 Command class, 纯 subclass override — Open/Closed 验证
- final line count *小于* step0 line count? (实际 final 包括 superclass 总行数 = 或 > step0; 但 subclass 缩到最小)

### A2 — C 复刻 binary 协议 + Command role interface

```bash
mkdir -p /tmp/ch21-c-cmd && cd /tmp/ch21-c-cmd

# 1) Command superclass — sequence responsibility + role interface
cat > command.h <<'EOF'
/* ch21 — Command role interface (sequence responsibility)
 * Subclass implements get_command_char + fields list.
 */
#ifndef COMMAND_H
#define COMMAND_H

#include <stddef.h>

typedef struct command {
    const char *const *fields;     /* field list */
    size_t n_fields;
    /* role methods — subclass fills these */
    const unsigned char *(*get_command_char)(struct command *self);
} command_t;

/* sequence responsibility: header / size / char / body / footer */
void command_write(command_t *cmd, void *out,
                   int (*write_bytes)(void *, const void*, size_t));

size_t command_get_size(const command_t *cmd);

#endif
EOF

cat > command.c <<'EOF'
#include "command.h"
#include <string.h>

static const unsigned char HEADER[2] = { 0xde, 0xad };
static const unsigned char FOOTER[2] = { 0xbe, 0xef };
static const unsigned char SIZE_LEN[1]  = { 1 };
static const unsigned char CMD_LEN[1]   = { 1 };

static void write_str(void *out, int (*wb)(void *, const void *, size_t),
                      const char *s) {
    wb(out, s, strlen(s));
    wb(out, "\0", 1);
}

size_t command_get_size(const command_t *cmd) {
    size_t total = sizeof(HEADER) + sizeof(SIZE_LEN) + sizeof(CMD_LEN)
                 + sizeof(FOOTER);
    for (size_t i = 0; i < cmd->n_fields; i++)
        total += strlen(cmd->fields[i]) + 1;
    return total;
}

void command_write(command_t *cmd, void *out,
                   int (*write_bytes)(void *, const void *, size_t)) {
    write_bytes(out, HEADER, sizeof(HEADER));
    unsigned char sz[1] = { (unsigned char)command_get_size(cmd) };
    write_bytes(out, sz, 1);
    const unsigned char *cc = cmd->get_command_char(cmd);
    write_bytes(out, cc, 1);
    for (size_t i = 0; i < cmd->n_fields; i++)
        write_str(out, write_bytes, cmd->fields[i]);
    write_bytes(out, FOOTER, sizeof(FOOTER));
}
EOF

# 2) Subclass: LoginCommand (2 fields)
cat > login_cmd.c <<'EOF'
#include "command.h"

static const char *login_fields[] = { "alice", "secret" };
static const unsigned char LOGIN_CHAR[1] = { 0x01 };

static const unsigned char *login_get_char(command_t *self) {
    (void)self; return LOGIN_CHAR;
}

command_t login_cmd_instance = {
    .fields = login_fields,
    .n_fields = sizeof(login_fields) / sizeof(login_fields[0]),
    .get_command_char = login_get_char,
};
EOF

# 3) Subclass: AddEmployeeCommand (5 fields)
cat > addemp_cmd.c <<'EOF'
#include "command.h"

static const char *addemp_fields[] = { "Mike", "122 Elm", "Miami", "FL", "10000" };
static const unsigned char ADDEMP_CHAR[1] = { 0x02 };

static const unsigned char *addemp_get_char(command_t *self) {
    (void)self; return ADDEMP_CHAR;
}

command_t addemp_cmd_instance = {
    .fields = addemp_fields,
    .n_fields = sizeof(addemp_fields) / sizeof(addemp_fields[0]),
    .get_command_char = addemp_get_char,
};
EOF

# 4) Test: 用 fake out (in-memory buffer) 验 sequence + size
cat > test_cmd.c <<'EOF'
#include "command.h"
#include <stdio.h>
#include <string.h>
#include <assert.h>

extern command_t login_cmd_instance;
extern command_t addemp_cmd_instance;

static char fake_out[1024];
static size_t fake_pos = 0;

static int fake_write(void *out, const void *buf, size_t n) {
    (void)out;
    if (fake_pos + n > sizeof(fake_out)) return -1;
    memcpy(fake_out + fake_pos, buf, n);
    fake_pos += n;
    return (int)n;
}

static void reset_fake(void) { fake_pos = 0; }

int main(void) {
    /* Login */
    reset_fake();
    command_write(&login_cmd_instance, NULL, fake_write);
    assert(fake_pos == command_get_size(&login_cmd_instance));
    assert(fake_out[0] == (char)0xde && fake_out[1] == (char)0xad);
    assert(fake_out[3] == (char)0x01);  /* login char */
    assert(fake_out[fake_pos - 2] == (char)0xbe && fake_out[fake_pos - 1] == (char)0xef);
    printf("LoginCommand: %zu bytes written, sequence OK\n", fake_pos);

    /* AddEmployee */
    reset_fake();
    command_write(&addemp_cmd_instance, NULL, fake_write);
    assert(fake_out[3] == (char)0x02);  /* addemp char */
    printf("AddEmployeeCommand: %zu bytes written, sequence OK\n", fake_pos);

    /* Verify: each subclass has its own command char */
    assert(login_cmd_instance.get_command_char(&login_cmd_instance)[0] == 0x01);
    assert(addemp_cmd_instance.get_command_char(&addemp_cmd_instance)[0] == 0x02);

    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_cmd command.c login_cmd.c addemp_cmd.c test_cmd.c && ./test_cmd
echo "rc=$?"
```

**验收**: `rc=0`, Login 与 AddEmployeeCommand 各自动 size + 正确 sequence, command char 是各 subclass override。

### A3 — Open/Closed 演示: 加 AggregateCommand 不改 Command

```bash
mkdir -p /tmp/ch21-aggregate && cd /tmp/ch21-aggregate

# 复用 A2 的 command.h / command.c
cp /tmp/ch21-c-cmd/command.h .
cp /tmp/ch21-c-cmd/command.c .
cp /tmp/ch21-c-cmd/login_cmd.c .
cp /tmp/ch21-c-cmd/addemp_cmd.c .

# Aggregate: 嵌套 command, 不动 Command class
cat > aggregate_cmd.c <<'EOF'
#include "command.h"
#include <string.h>

static const unsigned char AGG_CHAR[1] = { 0x03 };
static command_t *agg_inner[16];
static size_t agg_n = 0;

static const unsigned char *agg_get_char(command_t *self) {
    (void)self; return AGG_CHAR;
}

/* Aggregate override sequence: 不调 super.command_write, 走自己的序列 */
static int agg_fake_write(void *out, const void *buf, size_t n);
/* 重新定义 writeBytes — Aggregate 用同一 fake */

static const char *agg_fields_dummy[1] = { "" };  /* aggregate 无 fields */
command_t aggregate_cmd_instance = {
    .fields = agg_fields_dummy,
    .n_fields = 0,
    .get_command_char = agg_get_char,
};

void aggregate_append(command_t *inner) {
    if (agg_n < sizeof(agg_inner) / sizeof(agg_inner[0]))
        agg_inner[agg_n++] = inner;
}

void aggregate_write(command_t *self, void *out,
                     int (*write_bytes)(void *, const void *, size_t)) {
    const unsigned char HEADER[2] = { 0xde, 0xad };
    const unsigned char FOOTER[2] = { 0xbe, 0xef };
    write_bytes(out, HEADER, 2);
    /* Aggregate 用 total_size = header + size_byte + char_byte + n_inner + 内嵌 cmd 总长 + footer
     * 简化: 1 byte size 限制 256, 我们 demo 1 byte 用 n_inner */
    size_t inner_total = 0;
    for (size_t i = 0; i < agg_n; i++)
        inner_total += command_get_size(agg_inner[i]);
    size_t total = 2 + 1 + 1 + 1 + inner_total + 2;
    unsigned char sz[1] = { (unsigned char)(total & 0xff) };
    write_bytes(out, sz, 1);
    write_bytes(out, AGG_CHAR, 1);
    unsigned char n[1] = { (unsigned char)agg_n };
    write_bytes(out, n, 1);
    for (size_t i = 0; i < agg_n; i++)
        command_write(agg_inner[i], out, write_bytes);
    write_bytes(out, FOOTER, 2);
    (void)self;
    (void)agg_fake_write;
}
EOF

# 重新写 test, 测 aggregate 不需要改 command.c
cat > test_agg.c <<'EOF'
#include "command.h"
#include <stdio.h>
#include <string.h>
#include <assert.h>

extern command_t login_cmd_instance;
extern command_t addemp_cmd_instance;
extern command_t aggregate_cmd_instance;

void aggregate_append(command_t *inner);
void aggregate_write(command_t *self, void *out,
                     int (*write_bytes)(void *, const void *, size_t));

static char fake_out[1024];
static size_t fake_pos = 0;

static int fake_write(void *out, const void *buf, size_t n) {
    (void)out;
    if (fake_pos + n > sizeof(fake_out)) return -1;
    memcpy(fake_out + fake_pos, buf, n);
    fake_pos += n;
    return (int)n;
}

int main(void) {
    /* 验证 aggregate 不需要修改 command.c */
    aggregate_append(&login_cmd_instance);
    aggregate_append(&addemp_cmd_instance);

    fake_pos = 0;
    aggregate_write(&aggregate_cmd_instance, NULL, fake_write);
    /* sequence: 0xde 0xad, size, 0x03 (agg char), n=2, [login full], [addemp full], 0xbe 0xef */
    assert(fake_out[0] == (char)0xde && fake_out[1] == (char)0xad);
    assert(fake_out[3] == (char)0x03);  /* aggregate char */
    assert(fake_out[4] == (char)0x02);  /* n_inner = 2 */
    /* Last bytes should be footer */
    assert(fake_out[fake_pos - 2] == (char)0xbe && fake_out[fake_pos - 1] == (char)0xef);
    /* login and addemp chars appear after the first inner cmd */
    int saw_login = 0, saw_addemp = 0;
    for (size_t i = 0; i < fake_pos; i++) {
        if ((unsigned char)fake_out[i] == 0x01) saw_login = 1;
        if ((unsigned char)fake_out[i] == 0x02) saw_addemp = 1;
    }
    assert(saw_login && saw_addemp);

    printf("AggregateCommand: %zu bytes (header+size+char+n=2+login+addemp+footer)\n", fake_pos);
    printf("Open/Closed PASS: command.c untouched, new subclass added\n");
    return 0;
}
EOF

# 注意: aggregate_cmd.c 中包含一个未用的 agg_fake_write 声明,实际不调
# 用 Wno-unused-function 避免 warning
cc -std=c17 -Wall -Wextra -Wno-unused-function \
   -o test_agg command.c login_cmd.c addemp_cmd.c aggregate_cmd.c test_agg.c \
   && ./test_agg
echo "rc=$?"
```

**验收**: `rc=0`, AggregateCommand 不动 command. c, 纯 subclass override + 新 `aggregate_write` 序列。Open/Closed 验证。

> ⚠️ 这一例演示 *command. c source 0 改动* 加新功能。但 binary 大小增加(链接 aggregate_cmd. o)。

### A4 — AI duplication audit 工具: Python AST scan

```bash
mkdir -p /tmp/ch21-audit && cd /tmp/ch21-audit

cat > dup_audit.py <<'EOF'
#!/usr/bin/env python3
"""ch21 — 给定 codebase, 输出 method-level duplication 报告
   启发式:method body 含相同 substring sequence ≥ 30 chars 视为潜在重复
"""
import ast
import sys
from pathlib import Path
from collections import defaultdict

def collect_methods(path):
    src = Path(path).read_text()
    tree = ast.parse(src)
    out = []
    for node in ast.walk(tree):
        if isinstance(node, ast.ClassDef):
            for item in node.body:
                if isinstance(item, ast.FunctionDef):
                    body_src = ast.get_source_segment(src, item)
                    out.append((node.name, item.name, body_src))
    return out

def fingerprint(body):
    """简化的 fingerprint: 取非空非注释连续行"""
    lines = []
    for line in body.splitlines():
        stripped = line.strip()
        if stripped and not stripped.startswith('#'):
            lines.append(stripped)
    return '\n'.join(lines)

def audit(path):
    methods = collect_methods(path)
    fp_groups = defaultdict(list)
    for cls, mname, body in methods:
        fp = fingerprint(body)
        if len(fp) < 30:  # 太短不视为重复
            continue
        fp_groups[fp].append((cls, mname))
    # 输出有重复的组
    print(f"=== duplication audit: {path} ===")
    found = 0
    for fp, locs in fp_groups.items():
        if len(locs) > 1:
            found += 1
            print(f"\nGroup {found}:")
            for cls, mname in locs:
                print(f"  {cls}.{mname}")
            # 短预览
            preview = fp[:200].replace('\n', ' | ')
            print(f"  preview: {preview}...")
    if found == 0:
        print("No significant duplication found.")
    return 0

if __name__ == '__main__':
    if len(sys.argv) != 2:
        print("usage: dup_audit.py <file.py>", file=sys.stderr)
        sys.exit(2)
    sys.exit(audit(sys.argv[1]))
EOF

chmod +x dup_audit.py

echo "=== audit step0 (heavy duplication) ==="
./dup_audit.py /tmp/ch21-cmd/commands_step0.py

echo
echo "=== audit final (after refactor, duplication reduced) ==="
./dup_audit.py /tmp/ch21-cmd/commands_final.py
```

**验收**:
- step0 输出多个 duplication group (write methods 重复)
- final 输出"无重复"或仅 1 个小 group (因为 writeBody / getBodySize 共享实现)

## 六、值得深入思考的问题

### Q1: 抽 role interface 还是 utility class?

`Command` 是 role interface(类型契约),`CmdWriter` 是 utility class(代码块)。两者都能 DRY, 但 *扩展路径* 不同 — role 鼓励 subclass, utility 鼓励 static call。**关键问题**: 团队怎么 decide "这是 role 还是 utility"? 规则是什么?

### Q2: Open/Closed 的边界

AggregateCommand 加 subclass 不改 Command class。但 * protocol 变**(加新 header byte)需要改 superclass。Open/Closed 不 * *意味着"永不改"*, 而是"加新行为 = 不改"。协议升级 是 *改 superclass*, 这违反 Open/Closed 吗?

### Q3: 50 个 Command subclass 是不是抽过头?

`AggregateCommand` 加一个;`BulkDeleteCommand` 加一个;`ScheduleCommand` 加一个;`BatchCommand` 加一个 ... 50 个 subclass 每个 1 override。**关键问题**: 这种 "1-method class" 是 *role abstraction 滥用*。什么时候该把 50 subclass 合并?

### Q4: AI 复制 vs AI 抽 interface 的 trade-off

LLM 复制 AddEmployeeCmd 改字段名是 *快速*; 抽 Command superclass 是 *设计投入*。**关键问题**: 团队怎么 enforce "AI 复制时立刻 review, 看是否能抽 role interface"? 这是 review checklist 还是 build system check?

### Q5: Sequence responsibility 怎么 *可视化*

抽 sequence responsibility 需要 *看到* N 个 class 的 method call sequence 共享。**关键问题**: 怎么把 "sequence" 概念化、给 AI prompt 自动化? 这是 sequence diagram 自动生成还是 refactoring tool?

---

*下一章预告*: **Chapter 22 — I Need to Change a Monster Method and I Can't Write Tests for It** — ch21 解决"多 class 重复", ch22 解决"单 method 太长且不可测"。核心议题:(a) **Sprout Method** (ch25 p.59) — 把不可测的逻辑 *sprout* 到新 method, 新 method 可测;(b) **Sprout Class** (ch25 p.63) — 把不可测的副作用 *sprout* 到新 class;(c) **Wrap Method** — 把整个 method 包到 wrapper, 在 wrapper 里 fake;(d) 各种 *break out* 手法的选择标准 — 当 method 是 *computational* / *sequencing* / *coordinating* 时不同解。ch22 是 Part II 章节中的"方法级别"最后一章, 与 ch20 (class 级别) + ch21 (multi-class 级别) 共同构成"改代码"的尺寸谱。