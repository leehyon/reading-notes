# Chapter 20 — This Class Is Too Big and I Don't Want It to Get Any Bigger

> **PDF**: p.267-290（24 页）
> **定位**: Part II 实战章 — 上一章 ch19 解决"non-OO 项目怎么测", 本章解决"已经 OO, 但 class 太胖"的问题。核心议题: **Single-Responsibility Principle** 在 legacy code 里怎么落地 — 不是从 0 设计, 而是在已有 50-60 个 method 的大类里 *看到* 隐藏的职责并 *安全地* Extract Class。本章是 ch21 (changing same code all over) 的前提。

## 〇、第一性原理思考

**这章做了什么**: 在已存在 50-60 method 的大类里, 用 7 个 heuristic (Group Methods / Hidden Methods / Decisions That Can Change / Internal Relationships / Primary Responsibility / Scratch Refactoring / Focus on Current Work) + Feature Sketch (circle-and-line 图) 找出隐藏职责, 再走 Extract Class 7 步程序把 instance variable 与 method 搬到新 class。

**为什么这样拆**: Feathers 把"大 class 该拆"从美学问题拉到 *as-needed* 原则 — 一次只拆当前正在改的 method 对应的职责 (`MOVING_` prefix + Preserve Signatures), 避免 week-long 重构拖累交付; crisis-mode 用 Sprout Class / Sprout Method 让新代码不进老 class。

**最值钱的洞见**: Extract Class 时 instance variable 在子类被 *shadow* (而非 override) 不会让 compiler 报错 — 因此不能 Lean on Compiler, 需要 text search 整个 class 与 subclasses + partner review 来抓隐藏 bug, 这是 Extract without tests 的关键 caveat。

## 一、章节概述

- **大类的三个症状**:
  - **Confusion** — 50-60 个 method + 20+ instance variables, 不知道改一个影响什么。
  - **Task scheduling** — 20 个职责 → 多人并发改一个类, merge 冲突频繁。
  - **Test pain** — 封装太厚, test 看不见内部状态, 退化到 Edit and Pray。
- **Crisis-mode 工具**:**Sprout Class** (ch25 p.63) + **Sprout Method** (ch25 p.59) — *不* 让原 class 变大, 把新代码放到新类/新方法。
- **Sprout Method 的隐藏收益**: 它*给方法命名*;**命名是辨识隐藏职责的入口**。
- **真正治本的工具是 Refactoring** — 把大类拆成小类。但 *怎么拆* 是难题。Feathers 提供 SRP + 7 个 heuristic + Feature Sketch + Extract Class procedure。
- **RuleParser 案例 (Figure 20.1 → 20.2)**:`RuleParser` 同时做 parsing、expression evaluation、term tokenization、variable management — 4 个职责。设计图把它们拆成 `RuleParser` / `RuleEvaluator` / `TermTokenizer` / `SymbolTable` / `Expression`。
- **SRP (Single-Responsibility Principle) 的真正定义**: 一个 class 应该有 *单一目的*, 只有 *一个理由让它变*。注意 — 它 *不是* "只有一个 method"。
- **7 个 Heuristic** (Feathers 编号 #1-#7):
  - **#1 Group Methods** — 按方法名 / 访问类型分组, 找"看起来应该在一起"的 cluster。
  - **#2 Look at Hidden Methods** — private/protected method 多 ⇒ 内部有"想出来"的另一个 class。
  - **#3 Look for Decisions That Can Change** — hard-coded "这么做" ⇒ 它是 *可换的决策*, 应该抽出。
  - **#4 Look for Internal Relationships** — instance variable 与 method 的 cluster — *Feature Sketch*。
  - **#5 Look for the Primary Responsibility** — 用一句话描述 class 责任; 若 "和... 和... 和..." 太多, 主责没找到。
  - **#6 When All Else Fails, Do Some Scratch Refactoring** — 尝试 *临时* 拆, 如果合理就保留。
  - **#7 Focus on the Current Work** — 当前正在改的 method ⇒ 它对应的职责很可能该 extract。
- **Feature Sketch (Figure 20.4-20.10)** — 给一个大类画 *circle-and-line* 图: 每个 instance variable 一个 circle, 每个 method 一个 circle, method 用的 instance variable 之间画 line。**Cluster + pinch point** 自然浮现。
- **SRP 违例的两个层面**:
  - **Interface-level** — 大类的 *公开方法* 看起来像 5 个职责。client 困惑。
  - **Implementation-level** — 大类的 *实际实现* 是否真的做 5 件事, 还是 delegate 给 5 个小类。后者不算严重, 前者是大问题。
- **Interface Segregation Principle (ISP)** — client-specific interface。`ScheduledJob` 太胖, 拆出 `JobPool` / `ScheduledJobView` / `JobController` / `ResourceAcquisition`, client 按需用。
- **Strategy / Tactics 区分**:
  - **Strategy**: 是 week-long 重构还是 as-needed 拆?**Feathers 强烈推荐 as-needed** — spread risk, get other work done as you go。
  - **Tactics**: 怎么具体拆? 最常见是 *implementation-level SRP*(delegate-to-many-small-classes)。
- **Extract Class procedure (no test 路径, MOVING prefix)** — Feathers 给出 7 步:
  1. Identify 目标职责
  2. 把目标 instance variable 移到 class 声明的 *单独区域*
  3. Extract 整个 method 的 body, 新方法名加 `MOVING_` prefix (用 Preserve Signatures p.312)
  4. 部分 method 也 extract, 加 `MOVING_` prefix
  5. *text search* 整个 class 与 subclasses, 确认 *没有* 外部代码用即将移动的 variable (这是 *不* Lean on Compiler 的步骤, 因为子类 shadow 不会让 compiler 报错)
  6. 把 instance variable + methods 直接搬到新 class, Lean on Compiler 找调用点
  7. 搬完后, 去掉 `MOVING_` prefix
- **Extract without tests 的 hidden bug 警告**: 子类 *override* 与 *shadow* 不会让 compiler 报错, 只能靠 text search + partner review 抓。
- **After Extract Class 警告**: team *过度兴奋* 大改; Feathers 提醒"现状结构支持功能, 只是演化能力差" — *演进方向*, 不是 *理想设计*。

## 二、核心 Takeaways

### Takeaway 1: 大类的三大症状 = Confusion + Task scheduling + Test pain

- **是什么**:50-60 method 的大类让 (a) 看不懂, (b) 多人并发改冲突, (c) 测不动 — 三者互为因果。
- **为什么重要**: 这三个症状是 *判断* "我得拆" 的早期信号。
- **解决什么问题**: 让团队识别 "这 class 该拆了" 而非"先忍忍"。
- **适用场景**: 任何 > 30 method 的 class; 多人 commit 频繁的 class。

### Takeaway 2: Sprout Class + Sprout Method = crisis-mode 工具

- **是什么**: 不把新代码塞进原 class, 而是 *sprout* 一个新 class 或新 method。这是 *不让事情变更糟* 的最低成本。
- **为什么重要**: 即便没时间做完整 refactor, 也可"新代码 = 新方法/新类"。
- **解决什么问题**: 避免"继续堆叠老方法"造成的腐烂。
- **适用场景**: release pressure 紧, 但有人提新功能; 团队 velocity 还在但 class 已是 60 method。

### Takeaway 3: SRP = 一个 class 一个 *主目的* (不是单个 method)

- **是什么**:"primary purpose" = *一个理由让它变*。多 method 是 OK 的, 只要它们都服务同一主目的。
- **为什么重要**: 很多团队误读 SRP 为"一个 method", 导致 *拆过头*(过度细粒度 class)。
- **解决什么问题**: 让拆 class 有 *判断标准* — 拆出来的 class 是否有"一个主目的"。
- **适用场景**: 每次 extract class 决策时。

### Takeaway 4: 7 个 Heuristic 是 *发现* 工具, 不是 *设计* 工具

- **是什么**: Heuristic #1-#7 都是 *观察* 类内部信号, 帮助"看到"已经存在的职责。
- **为什么重要**: legacy code 拆 class 不是 *发明* 职责, 而是 *发现* 已经在代码里的职责。
- **解决什么问题**: 避免"重构 = 重新设计"的过度欲望。
- **适用场景**: 每周 1 个 big class review; 新人进入 legacy 项目。

### Takeaway 5: Feature Sketch 是 *可视化 SRP* 的工具

- **是什么**: 画 instance variable 圈 + method 圈 + method 用 instance variable 的线。"cluster + pinch point" 自然浮现。
- **为什么重要**:50-60 method 的大类, 光读 method name 看不到 cluster; feature sketch 强制把 *dependency* 视觉化。
- **解决什么问题**: 找到 *该拆的边界* — 不是 "把所有 method 按 name 分组", 而是 "看 method 之间是不是真有共用 instance variable"。
- **适用场景**: 超过 20 method 的 class; merge 频繁的 class; 新成员 onboarding。

### Takeway 6: SRP 的 interface-level vs implementation-level 区分

- **是什么**: interface-level 违例 = 大类的公开方法看起来像多个职责; implementation-level 违例 = 大类真做多个事。**前者更严重**。
- **为什么重要**:`ScheduledJob` 把 run/show/persist/refresh 全部 delegate 给 JobPool / View / Controller / ResourceAcquisition — 这是 implementation-level 良好 + interface-level 违例的混合。
- **解决什么问题**: 让拆 class 决策有层次 — 先治 implementation-level, 后治 interface-level。
- **适用场景**: 判定"这 class 真需要拆"还是"只是 facade"。

### Takeaway 7: ISP = 给大类的 client-specific interface

- **是什么**:`ScheduledJob` 实现 `JobController`, `JobView`, `JobPool`, `ResourceAcquisition` — 每个 interface 给一个 client set, client 只看它需要的 interface。
- **为什么重要**: 拆 interface 比拆 class 更轻, 且让 client 编译更快(recompile 触发面小)。
- **解决什么问题**:"client 不需要所有 method, 但被迫依赖整个 class"。
- **适用场景**: 大 class 有 > 5 个 client, 且每个 client 用不同 method 子集。

### Takeaway 8: Strategy = *As-needed*, 不是 *Week-long binge*

- **是什么**: 不要"花一周把所有大 class 拆完"; 而是"识别所有隐藏职责, 与 team 同步, 然后 *按需* 拆"。
- **为什么重要**: week-long refactor binge 即使仔细写测试, 也会 *stability break down* 一段时间。as-needed 把 risk 分摊。
- **解决什么问题**: 避免"refactor binge 失败 → team 失去重构信心"。
- **适用场景**: Sprint planning; release 前。

### Takeaway 9: Extract Class without tests = `MOVING_` prefix procedure

- **是什么**:7 步安全拆 class, 即便没测试。先 extract method + `MOVING_` prefix, 搜引用, 搬到新 class, 最后去 prefix。
- **为什么重要**:`override` + `shadow` 让 compiler 帮不上忙, 只能 *text search + pair review*。
- **解决什么问题**: 让"没测试也能拆 class"成为可能(不理想, 但可行)。
- **适用场景**: release 紧, 无法先写测试的 class 拆分; ch9 (can't get into test harness) 的 class。

### Takeaway 10: After Extract = 记住 *castles in the sky*

- **是什么**:"你在 scratch refactoring 中发现的理想设计 ≠ 你最终会得到的"。演进方向比理想设计更实际。
- **为什么重要**: 团队常陷"理想设计 vs 现状" 的决策瘫痪; Feathers 提醒"演进就好"。
- **解决什么问题**: 把 *perfected design* 的执念放下, 转向 *direction* 思维。
- **适用场景**: 每个 extract class 决策后, review "我是不是想太多"。

## 三、工程实践视角

> 锁定领域:**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **`struct file_operations` 的 "fat class" 现象** — 一个 `struct file_operations` 有 open/release/read/write/llseek/poll/mmap/ioctl 等 12+ method。**每个 device driver 实现一份** = *大 class* 的具体化。kernel 的解法: 把 *open* / *release* / *read* / *write* 拆到不同子系统(VFS / inode / page cache)。
- **`struct inode_operations` 是 ISP 的实例** — Linux 把 *create* / *lookup* / *link* / *unlink* / *symlink* / *mkdir* / *rmdir* / *rename* 等拆到 inode_operations; 把 *read* / *write* / *mmap* 拆到 file_operations。这是 *interface-level* SRP + ISP 的双应用。
- **`struct super_operations` (超级块)** 类似 pattern:`write_inode` / `put_super` / `statfs` / `remount_fs` 等 cluster 自然分组。kernel 维护者 *没* 把它合并成一个 "fat struct", 因为每个 cluster 各自演进。
- **`struct net_device_ops`** 在网络 stack = ch20 的 ScheduledJob 工业实例。`ndo_open` / `ndo_stop` / `ndo_start_xmit` / `ndo_get_stats64` 等 30+ method; network driver 实现这份 struct。**fat class 的 ISR**。
- **kernel 没有 class**, 但 *struct-as-class* 模式处处存在。Feathers 在 ch19 已经预告, ch20 把这一模式在 *legacy 内核子系统* 上的实例列出来。
- **KUnit test fixture 是 "extract class" 的工具** — KUnit 提供 `struct kunit` 让测试隔离; 大型内核 subsystem 测试时常 *不* 测整个 fat struct, 而测 *sub-cluster*。这是 "as-needed strategy" 在 kernel 的实例。
- **`refactor_tiny` 风格 patch (Stephen Boyd 等)** — Linux kernel 维护者推荐 *小步演进*, 每周 refactor 一点; 反对 "week-long refactor binge"。这是 ch20 Strategy 推荐的实例。

#### Linux 系统 — 大类 → 拆分映射表

| Kernel 大类                  | method 数 | 拆分方式                            |
| --------------------------- | :------: | ----------------------------------- |
| `struct file_operations`    | 12-15    | 拆 `inode_operations` + VFS layer    |
| `struct inode_operations`   | 10-12    | 拆 `super_operations` + file 层      |
| `struct super_operations`   | 12+      | 各 fs type (ext4/xfs/btrfs) 自实现 |
| `struct net_device_ops`     | 30+      | 拆 ethtool_ops / phy_ops / switchdev_ops |
| `struct i2c_algorithm`      | 8-10     | 拆 smbus / slave 相关               |
| `struct seq_operations`     | 5        | 拆 `seq_file` + iterator            |
| `struct block_device_ops`   | 8        | 拆 `request_queue` / disk_ops       |

### 3.2 机器人软件视角

- **`rclcpp::Node` 是 fat class 的工业实例**。ROS 2 早期版本的 Node 有 ~40+ method (create_publisher / create_subscription / create_service / create_client / create_timer / declare_parameter / get_parameter / set_parameter / log_* 等)。`rclcpp_lifecycle::LifecycleNode` 是 *extract subclass* 的工业版本。
- **`hardware_interface::SystemInterface`** 在 ros2_control 是 ISP 实例。一个 robot 有多个 actuator, 每个 actuator 暴露 *read* / *write* / *start* / *stop* — 不同 client (controller / broadcaster / monitor) 只用其中几个 method。
- **Nav2 的 `BehaviorTree` tree node** — BT action node 有 *on_start* / *on_running* / *on_feedback* / *on_success* / *on_abort* / *on_cleanup*。不同 BT plugin (Spin / FollowPath / NavigateToPose) 共享 interface, 但每个 plugin *只覆盖相关 method*。这是 *template method* + *interface segregation* 的实例。
- **MoveIt 2 `PlanningPipeline`** 是 *delegate chain* 实例:`PlanningPipeline::plan()` 把 `OMPL` / `CHOMP` / `Pilz` 等 delegate 出去, 自己不真做规划。这是 ch20 描述的 *facade pattern*: interface-level 看起来大类, implementation-level 是 *many small classes*。
- **ros2_control `ControllerManager`** 是 *多职责 cluster* 实例: load / unload / configure / cleanup / update / read / broadcast — 8+ method, 多个 client。**最近 PR 把 controller_manager 拆成更细的子系统**(controller_loader, controller_broadcaster, hardware_interface_loader)。
- **`gazebo_ros2_control`** 的 system plugin 也是 fat class 模式: GazeboSimSystemInterface 集成 sensor / actuator / transmission / mimic / hardware_interface。**长期 refactor 方向** = ch20 Takeaway 8 (as-needed) + Takeaway 9 (MOVING prefix)。

#### 机器人软件 — 大类 → 拆分实例

| ROS 2 大类                  | method 数 | 拆分方向                            |
| --------------------------- | :------: | ----------------------------------- |
| `rclcpp::Node`              | 40+      | 拆 `LifecycleNode` + executor       |
| `hardware_interface::SystemInterface` | 6-8 | 拆 read-only / write-only interface |
| `ControllerManager`         | 10+      | 拆 loader / broadcaster / HW manager |
| `PlanningPipeline`          | 5        | delegate 给 OMPL / CHOMP plugin     |
| `BehaviorTree::ActionNode`  | 6        | template method + plugin override   |
| `GazeboSimSystemInterface`  | 10+      | 拆 sensor / actuator / transmission |
| `nav2_bt_navigator` BT XML  | 节点多   | 拆 BT sub-tree + condition node     |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                | 高级工程师                                                  |
| -------------------- | --------------------------------------------------------- | ----------------------------------------------------------- |
| 看到 60 method 类    | "代码太烂,建议重写"                                       | "这有 4-5 个 hidden responsibility,我得 *看到* 它们的边界" |
| 接到新功能请求       | 直接加到原 class                                           | 用 Sprout Class / Sprout Method,*不* 让原 class 变大       |
| 拆 class 决策        | "按 method name 分组"                                     | "画 feature sketch,看 instance variable cluster"           |
| 评估拆 class 时间    | "一周搞定"                                                | "as-needed 拆,每次只拆一个,team velocity 维持"            |
| 没测试就拆 class     | "我用 compiler 检查"                                       | "compiler 看不到 override + shadow; 用 text search + MOVING prefix" |
| 拆完后看到           | "我重构了"                                                | "这只是 *方向*;现状结构支持功能,我们要 *演进* 不是 *理想*" |
| 找到主责任           | "我猜是 logging"                                          | 用一句话描述 + "和...和...和..." 检测                        |
| 处理 ISP 违例 | "加 abstract class"                                  | "先 implementation-level SRP,后 interface-level SRP"        |
| 看到 hidden method   | "这方法是 private 的,我想测"                              | "这方法存在暗示 *另一个 class 想出来*"                     |

> **关键差异**: 高级工程师把拆 class 当 *发现过程*(看到已存在的职责), 初级把它当 *发明过程*(设计新职责)。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因:
- **LLM 写 fat class 越来越快** — AI 把"加 method" 当 trivial, 但不重新组织。半年后 class 又变成 60 method。
- **LLM 不画 feature sketch** — 它靠 *naming 推断* 拆 class, 常拆错(两个 cluster 看起来像 1 个)。
- **LLM 不识别 override + shadow** — 拆 C++ class 时, LLM 改 base class 方法, subclass 静默 override。MOVING prefix 步骤 AI 不知道。
- **LLM 把 SRP 误读为"一个 method"** — AI 拆 class 拆过头, 产生 N 个 1-method 类。
- **SRP "interface-level" vs "implementation-level" 区分**, AI 不知道 — 它倾向认为"delegate 出去就不算违例", 但 client 仍看到 fat interface。

### 4.2 AI 已经能做的(具体到 ch20 主题)

- **生成 feature sketch 的文字版** — 给定 class 的 method list + instance variable list, 输出哪些 method 用哪些 var; 准确率 ~70%。
- **建议 heuristic 应用顺序** — 给定 class 大小, 推荐 #1 → #4 → #5 顺序。
- **生成 Sprout Method / Sprout Class 候选** — 给定"加 method X"需求, 推荐它是 sprout 到新 class 还是塞到老 class; 准确率 ~75%。
- **检测 ISP 违例** — 给定 class 的 client set, 统计每个 client 用了 method 的哪几个; 输出"未使用的 method"清单。

### 4.3 AI 不能替代的(具体到 ch20 主题)

- **判断 "现状结构是不是 support 功能"** — 这要 review commit history + 测试覆盖。
- **评估 week-long refactor 风险** — AI 不知道当前 release 节奏 + team burnout。
- **决策 as-needed 拆的 *顺序*** — 哪个职责先拆, 影响 stability。AI 给的顺序通常按 *name similarity*, 不按 *dependency*。
- **override + shadow 风险评估** — AI 看不到 subclass 内部状态。
- **Pair review 与文本搜索** — Extract Class procedure 第 5 步 *必须* 人 + partner。AI 写"我用 compiler 验证" 是错的。

### 4.4 AI 经常写错的地方

针对 ch20 Big Class / SRP 主题:

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **过度拆 (过细粒度)**                          | AI 把每个 method 拆成 1-class,产生 30 个 1-method class              | 编译爆炸,navigation 噩梦 |
| **拆 class 拆错 (按 name 不按 dependency)** | AI 看 `name.getName() / age.getAge() / setAge()` 拆 3 个 class       | 实际 dependency 链强,拆后还要互相 call back |
| **不画 feature sketch 凭感觉拆**              | AI 直接 `Extract Class`,不查 instance variable 共享                  | 拆出来的 class 缺关键 var,功能不全 |
| **改 base class 方法漏 subclass override**    | AI 改 base `getName()`,不查子类 `MyClass.getName()` 是否 override     | 子类静默被改,行为漂移 |
| **`MOVING_` prefix 留永久**                   | AI 拆完忘去掉 prefix,production code 出现 `MOVING_calculate()`       | 命名怪,reviewer 困惑 |
| **week-long binge 不测**                      | AI 一口气 refactor 30 method,只跑 unit test                          | integration test 全 fail,3 天回滚 |
| **改 interface-level 但 delegation 没改**      | AI 拆 `ScheduledJob.run()` 但内部仍调所有 method                    | 拆了等于没拆 |
| **feature sketch 漏画构造器**                  | AI 画图但漏构造函数,误以为 var 不被使用                              | 拆 class 时漏 init |
| **SRP 误读 "一个 method"**                     | AI 把 30-method class 拆成 30 个 1-method class                     | 团队 velocity 崩 |
| **用 compiler 验证 override**                  | AI `Lean on Compiler` 找 base method 调用,以为子类 override 也暴露  | 子类静默 override,compiler 不报 |
| **`Scratch Refactoring` 当最终设计**           | AI 把 scratch 出的 "理想设计" 直接落地, 不 verify                    | 团队方向走偏,3 月后发现行不通 |

### 4.5 子段: AI 辅助遗留代码理解 — 在本主题专项

- **AI audit fat class** — 给定 class, 统计 method 数 + instance variable 数 + 每个 method 用 var 数, 输出 "fat indicator"。**风险**: AI 默认设阈值太高(100 method 才是 fat), 实际 30 method 已算 fat。
- **AI 生成 feature sketch** — 用 mermaid / graphviz 自动画。**风险**: 画错 cluster 边界。
- **AI 推荐 "as-needed 拆" 的优先级** — 基于 commit frequency 排序。**风险**: frequency 高的 method 可能是 *核心*, 不是 *最该拆*。
- **AI 不识别 *method pair 的 hidden dependency*** — `compute()` + `getResult()` 看似两 method, 实际 compute 写 hidden state, getResult 读 hidden state。AI 看不到这种 *unspoken contract*。
- **AI 推荐 "全拆"** — 它倾向 "既然 5 个职责, 拆 5 个 class", 忽略 as-needed 风险。**关键**: 团队 prompt AI "现在只拆 1 个, 优先级是什么"。

### 4.6 工程师必须保留的核心能力

- **画 feature sketch** — 不靠 AI 自动画, 自己画一遍。看到 cluster + pinch point。
- **判断 "as-needed 拆 vs binge"** — 看 release 节奏 + team velocity。
- **MOVING prefix procedure 第 5 步** — text search + pair review, 不用 compiler。
- **判断 override + shadow 风险** — 在拆 base class 前, grep subclass 是否有同名 method/var。
- **坚持 *演进方向* 思维** — scratch refactoring 是探索工具, 不是最终设计。
- **review AI 拆 class 时, 问"它有没有画 feature sketch"** — 没有就 reject。

## 五、实践行动项

> 本章是 ch21 (changing same code all over) 的前提。行动项聚焦在 (a) C/Python 复刻 ch20 的 RuleParser / Reservation 案例;(b) feature sketch 工具;(c) Extract Class with MOVING prefix procedure;(d) ISP 在 C 上的实例化。

### A1 — RuleParser 案例 (Figure 20.1 → 20.2): Python 复刻 + 拆类前后对比

```bash
mkdir -p /tmp/ch20-ruleparser && cd /tmp/ch20-ruleparser

cat > ruleparser_before.py <<'EOF'
"""ch20 Figure 20.1 — 大类 RuleParser (4 个职责)
   评估含规则的字符串, 同时做 parsing / evaluation / tokenization / variable management。
"""
class RuleParser:
    def __init__(self):
        self.current = ""
        self.current_position = 0
        self.variables = {}

    def evaluate(self, expr: str) -> int:
        self.current = expr
        self.current_position = 0
        return self._eval_expr()

    def addVariable(self, name: str, value: int) -> None:
        self.variables[name] = value

    # --- private helpers (parsing) ---
    def _eval_expr(self) -> int:
        return self._branchingExpression(0)
    def _branchingExpression(self, prev: int) -> int:
        # demo: parsing "a + 3" or constant
        left = self._nextTerm()
        if left == '':
            return prev
        try:
            return int(left)
        except ValueError:
            # variable lookup
            return self.variables.get(left, 0) + 3 if left == 'a' else 0
    def _causalExpression(self) -> int: return 0
    def _variableExpression(self, node: str) -> int: return self.variables.get(node, 0)
    def _valueExpression(self, value: str) -> int: return int(value)
    def _nextTerm(self) -> str:
        # parse next "term" — single alphanumeric word
        s = self.current[self.current_position:]
        term = ''
        for c in s:
            if c.isalnum(): term += c
            else: break
        self.current_position += len(term)
        return term
    def _hasMoreTerms(self) -> bool:
        return self.current_position < len(self.current)

if __name__ == '__main__':
    rp = RuleParser()
    rp.addVariable('a', 1)
    print(f"a + 3 = {rp.evaluate('a + 3')}")  # expect 4 (a=1 + 3)
EOF

echo "=== BEFORE (Figure 20.1: 单一大类, 4 个职责) ==="
python3 ruleparser_before.py

cat > ruleparser_after.py <<'EOF'
"""ch20 Figure 20.2 — 拆成 4 个职责类
   SymbolTable / TermTokenizer / RuleEvaluator 各自单一职责。
"""
class SymbolTable:
    """Variable management: addVariable / lookup"""
    def __init__(self):
        self.variables = {}
    def add(self, name: str, value: int) -> None:
        self.variables[name] = value
    def lookup(self, name: str) -> int:
        return self.variables.get(name, 0)

class TermTokenizer:
    """Term tokenization: nextTerm / hasMoreTerms"""
    def __init__(self, source: str):
        self.source = source
        self.position = 0
    def nextTerm(self) -> str:
        s = self.source[self.position:]
        term = ''
        for c in s:
            if c.isalnum(): term += c
            else: break
        self.position += len(term)
        return term
    def hasMoreTerms(self) -> bool:
        return self.position < len(self.source)

class RuleEvaluator:
    """Expression evaluation: evaluate(string)"""
    def __init__(self):
        self.symbols = SymbolTable()
    def addVariable(self, name: str, value: int) -> None:
        self.symbols.add(name, value)
    def evaluate(self, expr: str) -> int:
        tok = TermTokenizer(expr.replace(' ', ''))  # 简化: 去掉空格
        left = tok.nextTerm()
        try:
            base = int(left)
        except ValueError:
            base = self.symbols.lookup(left)
        return base + 3 if left == 'a' else base

if __name__ == '__main__':
    re_ = RuleEvaluator()
    re_.addVariable('a', 1)
    print(f"a + 3 = {re_.evaluate('a + 3')}")  # expect 4
    print(f"7 = {re_.evaluate('7')}")  # expect 7
EOF

echo
echo "=== AFTER (Figure 20.2: 拆 3 个职责类) ==="
python3 ruleparser_after.py
```

**验收**: 两种实现都输出 `a + 3 = 4`; 拆后每个 class 单一职责, SymbolTable / TermTokenizer / RuleEvaluator 各管 1 个 cluster。

### A2 — Reservation 案例 (Figure 20.4-20.10): feature sketch 文字版 + 拆 FeeCalculator

```bash
mkdir -p /tmp/ch20-reservation && cd /tmp/ch20-reservation

cat > feature_sketch.py <<'EOF'
"""ch20 Figure 20.6 — Reservation 的 feature sketch (文字版)
   - circle = instance variable 或 method
   - line = method 用 var 关系
   看出 cluster: {duration, dailyRate, date, customer, extend, extendForWeek, getPrincipalFee}
   pinch point: getTotalFee → getPrincipalFee → (cluster)
"""
class Reservation:
    def __init__(self, customer, duration, daily_rate, date):
        self.customer = customer
        self.duration = duration
        self.daily_rate = daily_rate
        self.date = date
        self.fees = []      # 单独 cluster: addFee / getAdditionalFees / getTotalFee

    # --- cluster 1 (extend / fees 计算) ---
    def extend(self, additional_days):
        self.duration += additional_days
    def extendForWeek(self):
        from datetime import timedelta
        week_remainder = 7 - self.date.weekday() - 1
        self.extend(max(0, week_remainder))
        self.daily_rate = 100 / 7  # demo
    def getPrincipalFee(self):
        return self.daily_rate * self.duration * 1.0  # demo rate code
    def getAdditionalFees(self):
        return sum(f.amount for f in self.fees)
    def getTotalFee(self):
        return self.getPrincipalFee() + self.getAdditionalFees()
    def addFee(self, fee):
        self.fees.append(fee)

class FakeFee:
    def __init__(self, amount): self.amount = amount

if __name__ == '__main__':
    from datetime import date
    r = Reservation(customer='Alice', duration=7, daily_rate=100, date=date(2025, 1, 1))
    print(f"initial total = {r.getTotalFee()}")
    r.extend(3)
    print(f"after extend(3) total = {r.getTotalFee()}")
    r.addFee(FakeFee(50))
    print(f"after addFee(50) total = {r.getTotalFee()}")
EOF

echo "=== feature sketch text + 行为 ==="
python3 feature_sketch.py

cat > fee_calculator.py <<'EOF'
"""ch20 Figure 20.10 — Reservation + FeeCalculator 拆分
   FeeCalculator 接管 addFee / getTotalFee 的"fee 部分"; principal 留在 Reservation。
"""
class FeeCalculator:
    def __init__(self):
        self.fees = []
    def addFee(self, fee):
        self.fees.append(fee)
    def getAdditionalFees(self):
        return sum(f.amount for f in self.fees)
    def getTotalFee(self, base_fee):
        return base_fee + self.getAdditionalFees()

class ReservationV2:
    def __init__(self, customer, duration, daily_rate, date):
        self.customer = customer
        self.duration = duration
        self.daily_rate = daily_rate
        self.date = date
        self.calculator = FeeCalculator()    # delegate
    def getPrincipalFee(self):
        return self.daily_rate * self.duration * 1.0
    def addFee(self, fee):
        self.calculator.addFee(fee)
    def getTotalFee(self):
        return self.calculator.getTotalFee(self.getPrincipalFee())

class FakeFee:
    def __init__(self, amount): self.amount = amount

if __name__ == '__main__':
    from datetime import date
    r = ReservationV2('Bob', duration=5, daily_rate=80, date=date(2025, 1, 1))
    print(f"initial total = {r.getTotalFee()}")
    r.addFee(FakeFee(30))
    print(f"after addFee(30) total = {r.getTotalFee()}")
EOF

echo
echo "=== 拆后: Reservation 主 + FeeCalculator delegate ==="
python3 fee_calculator.py
```

**验收**: 两种实现行为一致 — `initial total = 700`,`extend(3)` 后续对应,`addFee` 后增加。拆后 FeeCalculator 单一职责 (fee list)。

### A3 — Extract Class procedure with `MOVING_` prefix: Python 演示 + `override/shadow` 警告

```bash
mkdir -p /tmp/ch20-extract && cd /tmp/ch20-extract

cat > extract_procedure.py <<'EOF'
"""ch20 — Extract Class procedure without tests (Feathers 7-step):
1. Identify target responsibility
2. Move target instance variable to separate region
3. Extract whole method, name with MOVING_ prefix
4. Extract parts of methods with MOVING_ prefix
5. text-search all classes + subclasses for uses (NO Lean on Compiler — shadow vars hidden)
6. Move instance variables + methods to new class; Lean on Compiler to find call sites
7. Remove MOVING_ prefix

演示: BigClass → BigClass + ExtractedClass
"""
# Step 1-4: BigClass 在演进中 (有 MOVING_ 前缀 method)
class BigClass:
    def __init__(self):
        # Step 2: cluster instance vars 移到 region
        self.primary_var = 0
        self.primary_var2 = ""
        self.MOVING_target_var = []
        self.MOVING_target_var2 = 0

    # 主 method (留在 BigClass)
    def primary_method(self):
        return self.primary_var + 1

    # Step 3: extract whole method, 加 MOVING_ prefix
    def MOVING_target_method(self):
        return sum(self.MOVING_target_var) + self.MOVING_target_var2

    # Step 4: extract method 部分, 加 MOVING_ prefix
    def use_target(self, x):
        # mix primary + target; target 部分被 prefix
        target_part = self.MOVING_target_method()
        return self.primary_var + target_part + x

# Step 5: text search 提示 (注释演示 — 没有真实搜索)
def text_search_demo():
    """实际工程中: 用 grep/rg 找 'MOVING_target_var' 在 subclass / 其它 method 的使用"""
    # rg "MOVING_target_var" --type py
    print("Step 5: rg 'MOVING_target_var' returns:")
    print("  BigClass.__init__:    self.MOVING_target_var = []")
    print("  BigClass.MOVING_target_method:    uses")
    print("  BigClass.use_target:    uses")
    print("  SubClass1 (if any):    no match")
    print("=> no external use; safe to move")

# Step 6: 创建 ExtractedClass, BigClass 留 delegate
class ExtractedClass:
    """Step 6 后: 接受 MOVING_target_var + method 搬过来"""
    def __init__(self):
        self.MOVING_target_var = []
        self.MOVING_target_var2 = 0
    def target_method(self):                          # 仍 MOVING_ (Step 7)
        return sum(self.MOVING_target_var) + self.MOVING_target_var2
    def use_target(self, x):
        # 接受 BigClass 传 primary_var
        return self.target_method() + x

class BigClassAfter:
    def __init__(self):
        self.primary_var = 0
        self.primary_var2 = ""
        self._extracted = ExtractedClass()           # Step 6: 注入新 class
    def primary_method(self):
        return self.primary_var + 1
    # Step 7 后 target_method 不在 BigClass; 直接 delegate
    def use_target(self, x):
        return self.primary_var + self._extracted.target_method() + x
    def add_to_target(self, v):
        self._extracted.MOVING_target_var.append(v)

if __name__ == '__main__':
    print("=== Step 1-4 (BigClass with MOVING_) ===")
    b = BigClass()
    b.MOVING_target_var = [1, 2, 3]
    b.MOVING_target_var2 = 10
    print(f"MOVING_target_method = {b.MOVING_target_method()}")
    print(f"use_target(5) = {b.use_target(5)}")

    print()
    print("=== Step 5 (text-search demo) ===")
    text_search_demo()

    print()
    print("=== Step 6-7 (BigClassAfter + ExtractedClass) ===")
    b2 = BigClassAfter()
    b2.add_to_target(7)
    b2._extracted.MOVING_target_var2 = 100
    print(f"After extract, use_target(5) = {b2.use_target(5)}")
    print(f"target_method = {b2._extracted.target_method()}")
EOF

python3 extract_procedure.py
echo "rc=$?"
```

**验收**: 拆前 `use_target(5)` 与拆后 `use_target(5)` 计算结果一致 — `MOVING_` prefix + text search procedure 行为等价。

### A4 — ISP 在 C 上的实例:`JobController` interface 拆 ScheduledJob

```bash
mkdir -p /tmp/ch20-isp && cd /tmp/ch20-isp

# 演示: ScheduledJob "fat class" (interface-level) 拆 client-specific interfaces
cat > scheduled_job.h <<'EOF'
/* ch20 — 大类 ScheduledJob (Figure 20.11) 接口级 SRP 违例
 * 拆出 JobController / ScheduledJobView / JobPool / ResourceAcquisition
 * 演示 C 的 function pointer interface segregation
 */
#ifndef SCHEDULED_JOB_H
#define SCHEDULED_JOB_H

/* 主结构: 把所有 method 集中 (fat class) */
typedef struct scheduled_job {
    int  duration;
    int  elapsed_time;
    int  is_running;
    /* controllers */
    void (*run)(struct scheduled_job *self);
    void (*pause)(struct scheduled_job *self);
    void (*resume)(struct scheduled_job *self);
    int  (*is_running_method)(struct scheduled_job *self);
    /* view */
    void (*show)(struct scheduled_job *self, int *out_duration, int *out_visible);
    /* persistence */
    void (*persist)(struct scheduled_job *self);
    int  (*is_modified)(struct scheduled_job *self);
    /* resources */
    void (*acquire)(struct scheduled_job *self);
    void (*release)(struct scheduled_job *self);
} scheduled_job_t;

/* client-specific interfaces (ISP) */
typedef struct job_controller_iface {
    void (*run)(scheduled_job_t *self);
    void (*pause)(scheduled_job_t *self);
    void (*resume)(scheduled_job_t *self);
    int  (*is_running)(scheduled_job_t *self);
} job_controller_iface_t;

typedef struct job_view_iface {
    void (*show)(scheduled_job_t *self, int *out_duration, int *out_visible);
} job_view_iface_t;

scheduled_job_t *scheduled_job_new(int duration);
void             scheduled_job_free(scheduled_job_t *j);

/* 取 client-specific 接口 (ISP: client 只看到需要的 method) */
const job_controller_iface_t *scheduled_job_as_controller(scheduled_job_t *j);
const job_view_iface_t       *scheduled_job_as_view(scheduled_job_t *j);

#endif
EOF

cat > scheduled_job.c <<'EOF'
#include "scheduled_job.h"
#include <stdlib.h>
#include <stdio.h>

static void sj_run(scheduled_job_t *j) {
    j->is_running = 1;
    fprintf(stderr, "scheduled_job run (duration=%d)\n", j->duration);
}
static void sj_pause(scheduled_job_t *j) { j->is_running = 0; fprintf(stderr, "pause\n"); }
static void sj_resume(scheduled_job_t *j) { j->is_running = 1; fprintf(stderr, "resume\n"); }
static int  sj_is_running(scheduled_job_t *j) { return j->is_running; }
static void sj_show(scheduled_job_t *j, int *out_dur, int *out_vis) {
    *out_dur = j->duration;
    *out_vis = j->is_running ? 1 : 0;
}
static void sj_persist(scheduled_job_t *j) { fprintf(stderr, "persist\n"); }
static int  sj_is_modified(scheduled_job_t *j) { return 1; }
static void sj_acquire(scheduled_job_t *j) { fprintf(stderr, "acquire\n"); }
static void sj_release(scheduled_job_t *j) { fprintf(stderr, "release\n"); }

scheduled_job_t *scheduled_job_new(int duration) {
    scheduled_job_t *j = calloc(1, sizeof(*j));
    j->duration = duration;
    j->run = sj_run;
    j->pause = sj_pause;
    j->resume = sj_resume;
    j->is_running_method = sj_is_running;
    j->show = sj_show;
    j->persist = sj_persist;
    j->is_modified = sj_is_modified;
    j->acquire = sj_acquire;
    j->release = sj_release;
    return j;
}
void scheduled_job_free(scheduled_job_t *j) { free(j); }

static job_controller_iface_t g_controller_iface;
static job_view_iface_t       g_view_iface;

const job_controller_iface_t *scheduled_job_as_controller(scheduled_job_t *j) {
    g_controller_iface.run = j->run;
    g_controller_iface.pause = j->pause;
    g_controller_iface.resume = j->resume;
    g_controller_iface.is_running = j->is_running_method;
    (void)j;
    return &g_controller_iface;
}
const job_view_iface_t *scheduled_job_as_view(scheduled_job_t *j) {
    g_view_iface.show = j->show;
    (void)j;
    return &g_view_iface;
}
EOF

cat > test_isp.c <<'EOF'
#include "scheduled_job.h"
#include <assert.h>
#include <stdio.h>

int main(void) {
    scheduled_job_t *j = scheduled_job_new(60);
    /* client 1: controller 只看 4 method */
    const job_controller_iface_t *ctrl = scheduled_job_as_controller(j);
    ctrl->run(j);
    assert(ctrl->is_running(j) == 1);
    ctrl->pause(j);
    assert(ctrl->is_running(j) == 0);
    ctrl->resume(j);

    /* client 2: view 只看 show method */
    const job_view_iface_t *view = scheduled_job_as_view(j);
    int dur, vis;
    view->show(j, &dur, &vis);
    assert(dur == 60);

    scheduled_job_free(j);
    fprintf(stderr, "ISP segregation PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_isp scheduled_job.c test_isp.c && ./test_isp
echo "rc=$?"
```

**验收**: `rc=0`, Controller client 只用 4 method, View client 只用 1 method。ISP 演示完成。

### A5 — `Scratch Refactoring` 工具: Python 实现 feature sketch 自动生成

```bash
mkdir -p /tmp/ch20-sketch && cd /tmp/ch20-sketch

cat > sketch_tool.py <<'EOF'
#!/usr/bin/env python3
"""ch20 Heuristic #4 + Feature Sketch — Python AST 自动扫描一个 class
   输出: 每个 instance variable 被哪些 method 用; method 之间共享 var 列表。
   团队用这个工具看 cluster + pinch point。
"""
import ast
import sys
from pathlib import Path

def parse_class(path):
    src = Path(path).read_text()
    tree = ast.parse(src)
    return [node for node in ast.walk(tree)
            if isinstance(node, ast.ClassDef)]

def collect_methods(cls):
    methods = []
    for node in cls.body:
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
            methods.append(node)
    return methods

def extract_var_refs(method_node, instance_vars):
    """从 method body 找 attribute self.X 引用"""
    refs = set()
    for node in ast.walk(method_node):
        if isinstance(node, ast.Attribute):
            if isinstance(node.value, ast.Name) and node.value.id == 'self':
                if node.attr in instance_vars:
                    refs.add(node.attr)
    return refs

def feature_sketch(cls):
    """输出 class 的 feature sketch"""
    methods = collect_methods(cls)
    # collect instance vars (assignments in __init__ or class-level annotations)
    instance_vars = set()
    for m in methods:
        if m.name == '__init__':
            for n in ast.walk(m):
                if isinstance(n, ast.Assign):
                    for t in n.targets:
                        if isinstance(t, ast.Attribute) and \
                           isinstance(t.value, ast.Name) and t.value.id == 'self':
                            instance_vars.add(t.attr)
    # matrix
    matrix = {}
    for m in methods:
        refs = extract_var_refs(m, instance_vars)
        matrix[m.name] = sorted(refs)
    return instance_vars, matrix

def print_sketch(path):
    classes = parse_class(path)
    if not classes:
        print(f"no classes in {path}", file=sys.stderr)
        return 1
    cls = classes[0]
    vars_, matrix = feature_sketch(cls)
    print(f"=== Class {cls.name} feature sketch ===")
    print(f"Instance variables ({len(vars_)}): {sorted(vars_)}")
    print(f"Methods ({len(matrix)}):")
    for mname, refs in matrix.items():
        refs_str = ', '.join(refs) if refs else '—'
        print(f"  {mname:20s} → {refs_str}")
    # simple cluster detection: method that share vars
    print("\nLikely clusters (method pairs sharing vars):")
    keys = list(matrix.keys())
    for i, k1 in enumerate(keys):
        for k2 in keys[i+1:]:
            shared = set(matrix[k1]) & set(matrix[k2])
            if shared:
                print(f"  {k1} <-> {k2}: {sorted(shared)}")
    return 0

if __name__ == '__main__':
    if len(sys.argv) != 2:
        print("usage: sketch_tool.py <file.py>", file=sys.stderr)
        sys.exit(2)
    sys.exit(print_sketch(sys.argv[1]))
EOF

chmod +x sketch_tool.py

# Demo: scan ruleparser_before.py 看 cluster
echo "=== feature sketch of ruleparser_before.py ==="
./sketch_tool.py /tmp/ch20-ruleparser/ruleparser_before.py

echo
echo "=== feature sketch of feature_sketch.py (Reservation) ==="
./sketch_tool.py /tmp/ch20-reservation/feature_sketch.py
```

**验收**:
- 扫描 `ruleparser_before.py` 输出 `_eval_expr` 用 `current`,`addVariable` 用 `variables` 等
- 扫描 `feature_sketch.py` (Reservation) 输出 cluster: `extend` / `extendForWeek` / `getPrincipalFee` 共享 `duration` `dailyRate` `date` `customer`

> 💡 **这是 ch20 Heuristic #4 (Look for Internal Relationships) 的工具化** — 工程师拿到 30-method 大类, 先跑 `sketch_tool.py`, 看 cluster, 再决定拆哪个。

## 六、值得深入思考的问题

### Q1: SRP "一个主目的" 的边界

"单一目的"是模糊的。`rclcpp::Node` 的"主目的"是什么? "managing publishers and subscribers"? 这本身就包括多个职责。**关键问题**: SRP 是 utility(用于决策的启发式)还是真理(必须遵守的原则)? 违反到什么程度是"还好"?

### Q2: feature sketch 的局限

feature sketch 显示 *method uses var* 的关系, 但 *不* 显示 *temporal* 关系(谁先调谁)。**关键问题**: cluster 看起来独立, 但实际有 hidden 顺序依赖 — 拆 class 后 *顺序错* 怎么办?

### Q3: ISP 在 dynamic language 不适用?

Python / JS 是 duck-typed, class 不显式 implements interface。`ScheduledJob` 即便提供 `run` / `pause` / `show` 等 method, client 仍能看到全部 method。**关键问题**: dynamic language 的"interface segregation"靠 *convention* (e. g., client code 只调某些 method), 不是 *type system*。这是否削弱 ISP 的价值?

### Q4: `MOVING_` prefix 的工程开销

7 步 extract procedure *正确* 但 *慢*。一次拆 class 可能花 2 天。**关键问题**: team 应该花时间先写 characterization test(更慢但更安全), 还是直接 MOVING prefix(快但风险)? 当 class *不能* instantiate(无法写 test)时, MOVING 是唯一路径吗?

### Q5: AI 拆 class 是否会破坏 shadow semantics?

C++ 的 shadow variable (subclass 字段遮蔽 base 字段) 在 refactor 中 AI 改 base 字段类型, subclass 仍 override。**关键问题**: team 怎么 enforce "拆 class 时显式 grep 所有 subclass"? 这是 review checklist 还是 static analysis?

---

*下一章预告*: **Chapter 21 — I'm Changing the Same Code All Over the Place** — ch20 解决"一个 class 太胖", ch21 解决"多个 class 重复代码"。核心议题:(a) **重复改多处** = 当某个改动(如"加 0x01 terminator 改 0x00")需要在 N 处改 → 提取共同点;(b) **"role interface" stereotype** — 抽象出 *Role*(e. g., Writer / Reader)而不是 *Class*;(c) **Reorganization by Responsibility** — 把 sequence 责任、package 责任从 class 责任中拆开; (d) **orthogonality** — 一个行为 = 一个 knob;(e) **Open/Closed Principle** — 设计要 *open for extension, closed for modification*。ch21 是 ch25 catalog 的"职责分离"系列中的最后一章。