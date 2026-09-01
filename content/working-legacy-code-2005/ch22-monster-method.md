# Chapter 22 — I Need to Change a Monster Method and I Can't Write Tests for It

> **PDF**: p.311-330（20 页）
> **定位**: ch6/ch21 已经把 *Sprout Method / Sprout Class* 当回避大方法的"快速通道"。本章反过来：不能回避时，怎么**正面强攻**大方法。把 ch8 的 Extract Method 推进到 *没有 IDE refactor 工具 + 没有测试* 的恶劣条件下，怎么一步步拆、怎么不拆错。ch23、ch25 直接接续本章技法。

## 〇、第一性原理思考

**这章做了什么**: 正面强攻 monster method —— 把 ch8 Extract Method 推到没 IDE 工具 + 没测试的恶劣条件下, 用形态分类、sensing variable、break-out-method-object 一寸寸拆 1000 行怪物。

**为什么这样拆**: 它把"怎么拆"从技法问题重写成"感知 + 形态判断"问题 —— bulleted 走 find sequences、snarled 走 skeletonize, 让 50 行测试不到的孤岛靠 sensing 字段被 pin 住再动刀。

**最值钱的洞见**: 实际上 monster 不是要"一次性想清楚再拆", 而是先把 Extract What You Know 的小零计数 (0-count) 抽到当前类、再 Be Prepared to Redo —— insight 不是拆前有, 是拆错几次才长出来。
## 一、章节概述

- **Monster method 的两种基本形态**：*Bulleted*（缩进浅，像项目符号 — 几个 block 平行排列）和 *Snarled*（一个巨大的缩进嵌套 — 几个 block 套在一起）。现实代码常是 mix。Feathers 给两个 Reservation 案例做对比。
- **Bulleted 的弱点是 "块之间临时变量串联"**：A 块声明的 temp 在 B 块里被用 — `extract` 后 B 块找不到 A 块的临时变量。copy/paste 行不通。Bulleted 的优势：缩进浅，导航不迷路；这是拆的第一步"信心的基础"。
- **Snarled 的弱点是 "导航即迷失"**：嵌套 4–5 层之后你开始 *vertigo*。Snarl 内部通常又嵌着 bulleted 块 — 这是隐藏测试死角，因为测试 case pin 不到这种深缩进里的行为。
- **工具支持的两种路径**：*有 IDE refactor 工具*（extract method 安全 — IDE 自己 verify 引用图）vs *没有 IDE refactor 工具*（必须自己用 test 兜底）。Feathers 强调"有工具就只用工具、把所有非工具编辑推迟到测试就位后"——这是 *Hyperaware Editing* (ch23) 的工具化形态。
- **手动拆的"先小再大"原则**：从 0–3 行 0-count 抽起 — 完全不传参数 / 不返回值；再扩到 coupling count ≤ 2 的小方法；最后才动大块。这是 *Extract What You Know* 的核心。
- **Sensing Variable**：在没有 IDE refactor 工具时，往类里加一个 instance boolean/int 字段 + 在待拆分支里赋值 —— test 通过这个字段 pin 行为。拆完后再删除。这是"在 production 代码里临时加 instrumentation" 的合法形态。
- **Gleaning Dependencies**：先把 *关键行为* 写测试保护，再 *未保护* 地拆 *次要行为*。前提是次要行为如果错，会 *立刻显现*（用户立刻看到 display 错；但 addEntry 错可能要几周才暴露）。这给"次要行为可以裸拆"开了许可证。
- **Break Out Method Object**：当 monster 里的临时变量本该 *互相耦合* 但又 *只在本方法内* 用时，把整个方法体搬进一个**新类**，原方法变成构造 + `run()`。临时变量变成新类的实例变量 — 它们自然 *有 sensing 价值*，因为测试可以在 run() 之后读取。这是 Ward Cunningham 的"发明抽象"路线。
- **Strategy 选择**：Skeletonize（拆 control + body 成两个方法）vs Find Sequences（把 condition+body 一起拆成一个方法）。Bulleted 偏 Find Sequences；Snarled 偏 Skeletonize — Feathers 自己承认两套建议冲突，他在工作时两套都来回用。
- **Extract to Current Class First**：拆出方法先落在**当前类**，即使名字显得怪（"recalculateOrder" 出现 order 字 — 暗示该挪到 Order 类）。理由：拆错了可以无成本撤回；若一开始就跨类挪，错一次要回滚两个文件。这是 *Be Prepared to Redo Extractions* 的安全网。
- **Extract Small Pieces First + Be Prepared to Redo**：先小再大；拆过几版之后常常会发现原来想抽"那一大块"其实该抽"另一小段"。这是 design insight 的渐进揭示 — Feathers 把它当作拆方法的核心反馈循环。
- **本章是 ch25 reference catalog 的"用法入门"**：每个技法（Sensing Variable / Gleaning / Break Out Method Object / Skeletonize / Find Sequences / Extract to Current Class）都在 ch25 里再展开一次。读 ch22 之后读 ch25 是配套。

## 二、核心 Takeaways

### Takeaway 1: Monster method 形态分类先于技法选择

- **是什么**: 在动手拆之前，先用"项目符号感"（bulleted）vs "嵌套眩晕感"（snarled）分类。形态不同，*默认策略*不同。
- **为什么重要**: Bulleted 块间临时变量串联是 *extract 立刻报错* 的主因；Snarled 的导航迷失让人 *还没开始就想放弃*。形态分类让你选 *Find Sequences* 还是 *Skeletonize*。
- **解决什么问题**: 把"我面对的是 1000 行怪物"变成"我面对的是一个 bulleted 段，几个 snarled 块 — bulleted 段先按 sequences 拆，snarled 段先 skeletonize"。
- **适用场景**: 任何超过 50 行的方法、刚接手一个 PR 的 monster；以及评估 refactor 优先级时。

### Takeaway 2: Sensing Variable 是"在生产代码里临时开 instrumentation"的合法手段

- **是什么**: 添加一个 `public boolean nodeAdded = false` 这样的 instance field，在 monster 内的关键分支里赋值；test 读它确认分支被走过；拆完后删掉。
- **为什么重要**: 这是 *没 IDE 工具 + 没测试* 的双恶劣条件下，唯一能 pin "某段条件是否被走过" 的廉价方式。它本质是"为了拆，把方法暂时变成 self-instrumented"。
- **解决什么问题**: 让 manual Extract Method 从"完全无验证"变成"每个抽出去的方法至少有 1 个 boolean sensing 兜底"。
- **适用场景**: 任何没有 IDE refactor 工具的代码（C / 嵌入式 / 旧 Java）；也用作 ch9 / ch10 "构造得了但看不到效果" 的 sensing 补充（ch3 的延伸）。

### Takeaway 3: Gleaning Dependencies — "关键行为先测，次要行为可裸拆"

- **是什么**: 写测试保护 monster 中的 *关键路径*（业务核心 — 比如 add entry）。剩余 *次要路径*（显示、日志、临时通知）若错立刻显现，因此可以裸拆。Feathers 自己说"感觉像偷懒 — 但行为的优先级本就不一样"。
- **为什么重要**: 这是"测试覆盖率不是均匀的" — 关键代码 100% 覆盖，次要代码靠可见性兜底。这是 realistic 工程节奏，不是 *coverage theater*。
- **解决什么问题**: 当 monster 方法测试全部行为代价太高（要造 30 个 fixture），用 Gleaning 把测试聚焦到关键 5 个 case，次要路径裸拆。
- **适用场景**: 嵌入式 / Linux kernel — 关键路径（数据 race、内存回收）必须测；次要路径（log、统计）错一次 dmesg 即可见，可裸拆。

### Takeaway 4: Break Out Method Object — 把临时变量升格为 sensing 通道

- **是什么**: 把 monster 方法体搬到新类 `MethodRunner`，临时变量变成 instance variables；原方法变成 `new MethodRunner(...).run()`。新类的 fields 天然能被 test 读到。
- **为什么重要**: 当你意识到 monster 里的"临时变量"其实就是 *跨多个子步骤共享的状态*，把它们 lift 成 fields 后，sensing 是免费的。
- **解决什么问题**: 解决"局部变量没法做 sensing — 又不想把它污染成 instance variable"的两难。把"instance variable 污染"局限在一个新类里。
- **适用场景**: C++ 复杂算法（图像处理、协议解析）、ROS 节点复杂 callback、Linux driver probe 流程。feathers 在 ch25 (330) 详述。

### Takeaway 5: Skeletonize vs Find Sequences — 两条互斥但都对的策略

- **是什么**: 同一段 `if (cond) { body; }` — 你可以拆成 `if (cond_predicate(x)) do_body(x);`（skeletonize — 留 control structure），也可以拆成 `recalc(x)`（find sequences — 把 cond+body 一起吞掉）。Bulleted 偏 sequences；snarled 偏 skeletonize。
- **为什么重要**: 拆 monster 不是"找唯一正确答案" — 是"找一个能让你继续往下拆的角度"。两条策略在不同上下文各自赢。
- **解决什么问题**: 让团队不再纠结"为什么拆出来这么怪" — 怪只是当前策略的中间态，下一轮会自然重整。
- **适用场景**: review PR 时看到"为啥方法叫 recalculateOrder，里面没 return" — 知道这是 Extract to Current Class First 的中间态。

### Takeaway 6: Extract to Current Class First + Be Prepared to Redo

- **是什么**: 即便一个 chunk 看起来明显属于另一类（比如 `recalculateOrder` 该在 `Order`），第一版也先抽到当前类 — 命名可以怪，但撤回便宜。重拆 / 跨类搬在测试就位后再做。
- **为什么重要**: 跨类 move 比同内 extract 难 10 倍的撤回成本。"先放当前类" = 把决策推迟到 insight 更清楚时。
- **解决什么问题**: 解决"我想清楚了再拆" vs "我想错了就回不去"的矛盾 — 用 *临时丑名字* 把这个矛盾消掉。
- **适用场景**: 大型重构 / 架构迁移期、ROS 节点拆分到 component container、kernel 拆 module。

### Takeaway 7: 有 IDE refactor 工具 — 用工具独占；不要混手工编辑

- **是什么**: 当 IDE 有 extract method 等安全 refactor 时，**只用工具完成结构变更**，连"调整顺序""格式化""重命名"都推迟到测试就位后做。理由：工具保证安全，手工编辑没保护 — 二者混在一起时，无法事后分辨"哪行是工具的（安全）哪行是我的（手工）"。
- **为什么重要**: 这是 hyperaware editing 的工具化体现。一旦混了手工，diff 里每行都得人工 verify — refactor 工具的 leverage 归零。
- **解决什么问题**: 在 C++ / Java 重型 codebase 里用 IDE 拆 1000 行 monster 时, 保持"refactor commit"和"feature commit"互相独立。
- **适用场景**: IntelliJ IDEA / Eclipse 重构 Java；CLion / Visual Assist 重构 C++。Clangd 没 extract method — 这时本节建议反向适用（手工+test）。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **Linux 内核里的 monster method**：典型是 `ieee80211_tx_status` 这类长达 200+ 行的 tx status 处理函数，里面嵌套 switch on `info->flags`，每 case 又有 10+ 行 bit 检查。形态偏 snarled。
- **bulleted 形态在内核少见**：因为内核开发者倾向于"小函数"哲学。但 `__do_softirq` / `do_task_stat` 这类 *dump* 函数常是 bulleted — 一连串 printk 块。Bulleted 块间没有临时变量串联 — `extract` 干净。
- **没有 IDE refactor 工具** —— `clangd` 没有安全 extract method；coccinelle 是 *semantic patch* 但不是 IDE interactive。这把 Linux 内核代码逼到 *Takeaway 7* 的反向：手工 + test 兜底。
- **Sensing Variable 的内核等价物**：trace_printk / ftrace —— 但这两个是 *out-of-band*，不修改生产代码、不通过 test 断言。**真在内核里做 in-band sensing**，惯用法是 `DECLARE_TRACE` + `trace_<event>_enabled()`，但这仍属 instrumentation 不是 refactor helper。本章 sensing variable 在内核的工程替代是：**新增 debugfs 字段**（`debugfs_create_bool("node_added", 0444, ...)`），test / 调试脚本读它。
- **Gleaning Dependencies 的内核应用**：在 `inode_init_owner` 这类"既有 security 关键路径又有 inode 模式赋值"的方法里 — LSM hook 是关键路径（必须测），inode->i_mode 赋值是次要（错一次 ls -l 即可见）—— 关键路径有 KUnit test，次要路径可裸 refactor。
- **Break Out Method Object 的内核实例**：`tcp_sendmsg` 拆 `tcp_sendmsg_locked` + `tcp_sendmsg_fastopen` 是"把临时的 lock state 升格成 instance variable"的内核版本。但严格说 Break Out Method Object 在 C 里要做成 `struct tcp_send_state { ... }` 临时对象，比 C++/Java 啰嗦得多。
- **Skeletonize 在内核**：典型是把 `if (cond) { body; }` 拆成 `if (should_handle(...)) handle(...);` — 但内核社区偏好 *inline* 反对过度函数化 — Skeletonize 在内核常被 reviewer 反对。Find Sequences 在内核更受欢迎（"这段逻辑打包成一个 named function"）。
- **Extract to Current Class First 的内核应用**：C 没有 class，"先放当前文件 + 怪名字" 对应 `static int foo_handle_x(...)` + 后续把它挪到 `foo-handle.c`。撤回成本仍低（C 里 `git revert` 一行）。
- **Be Prepared to Redo 的内核成本**：一次 `static` 内函数的重命名 = `git mv` + sed + 一轮 build。**Linux 维护者偏好 "一次性想清楚再发 patch"**——这和 Feathers 的 "先小再大" 哲学相反。但 *out-of-tree refactor*（机器人 / 公司内部 fork）按 Feathers 来反而更快。

#### Linux 系统 — Monster Method 拆解策略表

| 怪物类型            | 例子 (Linux kernel)                      | 推荐首步                  | sensing 通道                |
| ------------------- | ---------------------------------------- | ------------------------- | --------------------------- |
| Bulleted dump 函数  | `do_task_stat`, `seq_print` 家族         | Find Sequences            | tracepoint / `printk` 计数 |
| Snarled tx handler  | `ieee80211_tx_status`                    | Skeletonize               | debugfs `node_added`        |
| 复杂状态机          | `tcp_fastretrans_alert`                  | Break Out Method Object   | struct field + KUnit        |
| 嵌套 lock 操作      | `mmap_region` / `do_munmap`              | Skeletonize + sensing var | lockdep `lock_acquired`     |
| 业务规则密集        | `nf_conntrack_tcp_packet`                | Gleaning Dependencies     | nf_log 计数 + conntrack hash |
| 算法 + 协议混合     | `xfs_iflush_done` / `ext4_writepages`    | Break Out Method Object   | tracepoint writeback        |

### 3.2 机器人软件视角

- **ROS 节点 callback 是经典 monster**：导航节点的 `executeCb`（Nav2 BT navigator）长达 200+ 行，混了 path 接收、controller 调用、BT 状态恢复、recovery 行为。形态偏 snarled。
- **ROS1 vs ROS2 拆 monster 的差异**：ROS1 是 *global state*（param server / topic）—— sensing 要把 `NodeHandle` 抽 interface（ch25 Adapt Parameter 的应用）。ROS2 是 *composition* + *DI* —— sensing 便宜得多。
- **`rclcpp::Node` 的 Break Out Method Object**：把 `executeCb` 拆成 `class NavController { run() {...}; state_; ... }` + 节点持有 `NavController controller_; controller_.run();`。这等价于 Feathers 的方法对象。
- **Nav2 / MoveIt2 的真实例子**：Nav2 在 ROS2 化时把 `move_base` 的 monolithic callback 拆成 `PlannerServer` + `ControllerServer` + `BtNavigatorServer` + `RecoveryServer` —— 形态对应 *Extract to Current Class First → Move to New Class*。当时 `SmacPlanner` 等新规划器能 plug-in 就是因为 `PlannerServer` 已经存在 seam。
- **sensing var 在机器人测试的版本**：`MockClock` (ch3 提过) + `RecordingPublisher` + `RecordingTfBuffer`。在 Nav2 的 PR 历史里，BT node 测试几乎全靠这些 fake。
- **Gleaning 在机器人**：控制器算法的核心（PID + 力矩映射）必须测；ROS message 的 packing/unpacking (sensor_msgs) 错一次 rviz 即可见 —— 可裸 refactor。
- **Skeletonzie vs Find Sequences 在 ROS2 行为树**：BT XML 把"条件 + 动作"分节点 —— 这是 *Skeletonize* 的声明式版本。ROS2 BehaviorTree. CPP 把节点当 `if (cond) { action; }` 的拆解形式 — 每个 ConditionNode + ActionNode 就是拆好的小方法。Nav2 大量 BT node 是这一哲学的实例。
- **Be Prepared to Redo 在 Nav2**：从 ROS1 move_base 移植到 Nav2 时多次"先拆 server 后合 server"（recovery server 多次合并又拆分）—— 是 Feathers "redo extractions" 的工程案例。
- **`ros2_control` 的硬件接口**：fake 的 hardware_interface:: SystemInterface 把 monster 的 controller loop 拆成 update() + write() 两段 —— `Skeletonize` 的工业实例。

#### 机器人软件 — Monster Method 拆解策略表

| 怪物类型            | 例子 (ROS2 / Nav2)                       | 推荐首步                  | sensing 通道                |
| ------------------- | ---------------------------------------- | ------------------------- | --------------------------- |
| BT executeCb        | `nav2_bt_navigator::ExecuteBt`           | Skeletonize (BT XML)      | MockClock + BT recorder     |
| 控制器主循环        | `ros2_control::ControllerManager::update` | Find Sequences           | fake hardware_interface     |
| Perception pipeline | `image_proc::RectifyNode::imageCb`       | Gleaning (核心 rectify vs resize) | RecordingPublisher |
| Planner 综合        | `nav2_planner::PlannerServer::computePath` | Break Out Method Object  | costmap snapshot diff       |
| Multi-sensor fusion | `robot_localization::Ekf::odomCb`        | Skeletonize               | bag playback + state topic  |
| Lifecycle 状态机    | `nav2_lifecycle_manager`                 | Skeletonize               | Lifecycle topic subscriber  |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 看到 monster        | "重构它" — 立刻开 IDE extract                               | 先分类 bulleted / snarled；选 skeletonize 还是 find sequences |
| 拆的粒度             | 一次抽 30 行的大方法                                         | 一次抽 2-3 行 0-count 的微方法；命名可怪                     |
| 没有 IDE 工具        | "没法做"                                                    | 用 sensing variable + extract what you know + gleaning      |
| 临时变量处理         | 全 lift 到 instance variable（污染）                         | Break Out Method Object — 把临时变量局限在一个新类里        |
| 跨类 move            | 一开始就挪到正确位置                                         | Extract to Current Class First — 先放当前类，insight 更清楚再挪 |
| 拆错一次             | "我前面都白做了" — 放弃                                      | "Be Prepared to Redo" — 撤回重来；拆错是 insight 的成本     |
| 工具可用时           | 工具 + 手工混用                                              | 工具独占一轮；手工编辑等测试就位后                           |
| sensing 字段         | "这字段污染设计"                                             | "这是 refactor 的临时脚手架，拆完删"                          |
| Gleaning             | 觉得不严格 / 偷懒                                            | 关键行为 vs 次要行为分开对待是 realistic 工程节奏             |

> **关键差异**: 高级工程师 *先小再大，先当前类后跨类，先关键后次要* —— 这三条都反直觉，但都从"拆错的撤回成本"反推出来。初级倾向于"想清楚再动手"，但 insight 在动手后才清楚。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然重要。AI 在 monster method 上 **写得最差** 的恰好是"sensing variable 暴露成 public"和"工具/手工混用"这两条。原因：

- AI 写 Java 时默认把所有 field 写 `public` —— sensing var 的"暂时 public" 它不会主动改回 `private`，reviewer 容易漏。
- AI 看到 IDE 有 extract method，会"顺便"加注释 / 改格式 —— 破坏 *Takeaway 7* 的工具独占原则。
- AI 倾向于"想清楚了再抽大方法"——反 *Be Prepared to Redo*。
- AI 给 C 代码做 Break Out Method Object 时，写出来的 struct 字段不分组（*all instance variable*），不是"局限在新类"。

### 4.2 AI 已经能做的

- **识别 bulleted / snarled 形态**：准确率 70-80%。给一段 monster，AI 标"这是 bulleted，3 段串联"。
- **建议 sensing variable 位置**：AI 能识别"该字段 pin 哪个分支被走过"。准确率 60-70%。
- **生成 skeletonized 重构版本**：把 `if (cond) { body; }` 拆成 `if (cond_predicate()) do_body();` —— 准确率 80%（前提是 cond + body 在同一函数）。
- **找出 coupling count 为 0 的安全 extract 候选**：扫描 monster 找不传参不返回的 chunk —— 准确率 90%。
- **建议 Break Out Method Object 的 struct layout**：基于临时变量列表给出 fields + `run()` 方法签名 —— 准确率 70%。

### 4.3 AI 不能替代的

- **判断 Gleaning Dependencies 的边界**：哪些是关键行为 vs 次要行为 —— 需要 domain knowledge + 业务优先级。AI 给"哪些会失败立刻可见"判不准（这是产品/用户视角）。
- **决定 skeletonize 还是 find sequences**：取决于你"接下来想往哪个方向拆"，AI 看不到长期 plan。
- **跨类 move 的"正确"位置**：AI 给"挪到 Order 类"，但 Order 类本身要不要拆 / 挪到哪是 architecture judgement。
- **debugfs field 在内核的命名空间、权限 (0444 vs 0644)**：AI 不知道团队 debugfs 命名约定。

### 4.4 AI 经常写错的地方

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **sensing var 写成 public 永久字段**            | AI 给 `public boolean nodeAdded = false;` 拆完留 production 里       | API 表面污染，调用方误用 |
| **混用 IDE 工具和手工编辑**                      | AI 用 IntelliJ 抽完一段又手动调换两行顺序                            | diff 失去"工具=安全"的保证; 重做时无法 revert 半截 |
| **"想清楚"再抽大方法**                          | AI 直接抽 30 行块；不先 0-count 起步                                 | 拆错回滚成本高; insight 没机会出现 |
| **临时变量 lift 成 instance 而不是新类**        | AI 把 monster 的所有 local vars 提为 instance fields                 | 污染生产类的 API 表面, 不局限在新类 |
| **跨类 move 跳过 Extract to Current Class**    | AI 抽 `recalculateOrder` 直接挪到 `Order.cpp`                       | 错一次要 revert 两个文件 + 改 build 依赖 |
| **Gleaning 误判关键 vs 次要**                   | AI 把 display 错当关键行为（"看起来显眼"），把 add 当次要            | 测试保护错位; 真正的业务 bug 没覆盖 |
| **sensing var 忘记拆完后删**                    | AI 写出 `nodeAdded = true;` 字段但 refactor 后忘了清理               | dead field 永久留存, 文档里写"为什么有这字段"的困惑 |
| **Skeletonize 把 cond 拆得过细**                | AI 拆出 `isOrderReady()` + `hasLimit()` + `needsRecalc()` 三层       | 过度函数化; 阅读链路过长; review 时难懂 |
| **C 里 Break Out Method Object struct 字段扁平** | struct `TcpSendState` 30 个字段平铺, 不分组                          | 后续扩展时 struct layout 改动 = 全字段重排 |
| **debugfs sensing var 命名随意**                | AI 创建 `node_added` 但不放在子系统专属目录                         | debugfs 全局变脏; 用户找不到; 维护成本涨 |

### 4.5 子段：AI 辅助遗留代码理解 — 在本主题专项

- **识别 monster 的"段落边界"**：给 1000 行方法，AI 自动识别 N 个块 —— 比人眼快。可作为拆之前的 *first scan*。
- **生成 sensing variable 的 stub**：基于"哪段条件是关键"，AI 给 `nodeAdded` / `cookieSeen` / `pathComputed` 等候选 —— 准确率 60%，人 verify 一遍够。
- **生成 Skeletonize 版本**：`if (cond) { body; }` → `if (predicate()) action();` —— 准确率高，但需要人 review 命名。
- **识别 Gleaning Dependencies 的关键行为**：AI 看代码结构（函数命名、调用频率）给"哪些看起来是关键路径"，但业务关键性（*哪个错会让用户亏钱*）必须人来标。
- **生成 Break Out Method Object 的 struct 定义**：临时变量列表 → struct fields —— AI 干这活比人快 10 倍。
- **维护 sensing variable 的"删除清单"**：refactor 结束后 AI 扫一遍"哪些临时字段没被引用了"——避免 dead field 留存。
- **AI 风险**: 它会在 extract 后"顺手优化" —— 改循环、改命名、调格式 —— 破坏 *Takeaway 7*。人必须 push back 守住"refactor commit = 0 行为改动"。

### 4.6 工程师必须保留的核心能力

- **形态分类**：bulleted vs snarled 是拆 monster 的 first decision，AI 不能替。
- **sensing variable 的"临时"约束**：知道它是 refactor 脚手架，不是 design。Review 时看到 `public boolean` 在 production 立刻警觉。
- **Gleaning 的关键 vs 次要判断**：domain knowledge + 业务优先级 — 不可外包。
- **0-count extract 的耐心**：知道从 0 行小方法抽起比抽 30 行更安全 —— AI 倾向于"做大事"，人必须 *耐心做小事*。
- **Be Prepared to Redo 的心态**：拆错是 insight 的成本不是浪费。AI 看到 redo 会"自我怀疑"重写一遍 —— 人要敢 redo。
- **工具独占的纪律**：有 IDE refactor 工具时只用工具，手工等测试就位 —— 这是 review 时一眼能看出的纪律点。

## 五、实践行动项

> 本章聚焦"manual + 没测试"条件下拆 monster。行动项用 C 复刻 Feathers 的 Reservation:: extend / DOMBuilder 案例；跑通 compile + 简单执行。

### A1 — 复刻 Feathers 的 bulleted + snarled Reservation 方法 (C 版)

```bash
mkdir -p /tmp/ch22-monster && cd /tmp/ch22-monster

cat > reservation.c <<'EOF'
/* Reservation::extend 的 bulleted + snarled C 复刻 */
#include <stdio.h>
#include <string.h>

typedef enum { LUXURY=0, SUV=1, VAN=2, REGULAR=3 } VehicleType;
typedef enum { GIG=0, NON_GIG=1 } Location;
typedef enum { NOT_AVAIL_LUXURY, NOT_AVAIL_SUV, NOT_AVAIL_VAN, AVAILABLE } Status;
typedef enum { INITIAL=0, HELD=1 } State;
typedef enum { VIP_DIAMOND=0, VIP_NORMAL=1 } VipStatus;

/* ---- monster: bulleted 形态 (浅缩进, 多 block 平行) ---- */
int reservation_extend_bulleted(
    VehicleType type, Location loc, int customer_id,
    long starting_date, int additional_days,
    int vip_status, int last_cookie, int days_held)
{
    (void)type; (void)loc; (void)starting_date; (void)last_cookie; (void)days_held;
    int ident_cookie = -1;
    Status status = (Status)(customer_id % 4);  /* fake 状态 */

    /* 块 A: 升级路径 */
    switch (status) {
        case NOT_AVAIL_LUXURY:
            ident_cookie = 1000 + additional_days;
            break;
        case NOT_AVAIL_SUV: {
            int the_days = additional_days + additional_days;
            ident_cookie = 2000 + the_days;
            break;
        }
        case NOT_AVAIL_VAN:
            ident_cookie = 3000 + additional_days;
            break;
        case AVAILABLE:
        default:
            ident_cookie = 9999;
            break;
    }

    /* 块 B: waitlist 检查 */
    int need_waitlist = (ident_cookie != -1);
    (void)need_waitlist;

    /* 块 C: 客户等级标记 */
    int upgrade_query = (vip_status == VIP_DIAMOND);

    /* 块 D: 最终 extend 调用 */
    int final_cookie = last_cookie;
    if (!upgrade_query) {
        final_cookie = ident_cookie;
    } else {
        final_cookie = ident_cookie + 1;
    }
    return final_cookie;
}

/* ---- monster: snarled 形态 (深缩进) ---- */
int reservation_extend_snarled(
    VehicleType type, Location loc, int customer_id,
    long starting_date, int additional_days, int state_code)
{
    (void)type; (void)starting_date; (void)additional_days;
    int hold_cookie = -1;
    State state = (state_code > 0) ? HELD : INITIAL;

    if (customer_id > 0) {
        Status status = (Status)(customer_id % 4);
        if (status != AVAILABLE) {
            switch (status) {
                case NOT_AVAIL_LUXURY:
                    hold_cookie = 1000;
                    if (loc == GIG && customer_id == 45) {
                        /* 嵌套 4 层 */
                        int code = customer_id % 3;
                        if (code == 1) {
                            int total = 2000;
                            if (state == INITIAL || state == HELD) {
                                total += 100;
                                if (loc == GIG && additional_days > 2) {
                                    if (state == HELD) total += 30;
                                }
                            }
                            hold_cookie = total;
                        }
                    }
                    break;
                case NOT_AVAIL_SUV:
                    hold_cookie = 2000;
                    break;
                case NOT_AVAIL_VAN:
                    hold_cookie = 3000;
                    break;
                default: break;
            }
        } else {
            hold_cookie = 9999;
        }
    }
    return hold_cookie;
}

int main(void) {
    int r1 = reservation_extend_bulleted(REGULAR, GIG, 7, 20260101L, 3,
                                         VIP_DIAMOND, 500, 5);
    int r2 = reservation_extend_snarled(LUXURY, GIG, 6, 20260101L, 4, 1);
    printf("bulleted -> %d, snarled -> %d\n", r1, r2);
    return (r1 == 10000 && r2 == 3000) ? 0 : 1;
}
EOF

cc -std=c17 -Wall -Wextra -O0 -o reservation reservation.c && ./reservation
```

**验收**：
- 编译零警告（`-Wall -Wextra`，所有 unused param / local var 用 `(void)x;` 显式标记）。
- `./reservation` 输出 `bulleted -> 10000, snarled -> 3000`，exit 0。
- 读代码可见两种形态（缩进浅 vs 深 4 层）。
- 改 `customer_id` 看不同 case: `customer_id=6 → status=NOT_AVAIL_VAN → 3000`；`customer_id=10 → status=NOT_AVAIL_LUXURY → 进 4 层嵌套`。
- 改 `vip_status` 为 `VIP_NORMAL` 时 r1 应当=9999（不 +1）。

### A2 — Sensing Variable: DOMBuilder 的 nodeAdded 复刻 + 测试

```bash
mkdir -p /tmp/ch22-sensing && cd /tmp/ch22-sensing

cat > dom_builder.c <<'EOF'
/* DOMBuilder.processNode + sensing var 的 C 复刻 */
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

typedef enum { TF_A=0, TF_G=1, TF_H=2, TF_GLOT=3, TF_OTHER=4 } NodeType;

typedef struct {
    NodeType type;
    int is_child;
} XDOMNNode;

typedef struct {
    int node_added;          /* <-- sensing var, refactor 完删 */
    int para_count;          /* 业务字段 */
} DOMBuilder;

static int is_basic_child(XDOMNNode *node) {
    return node->type == TF_G || node->type == TF_H
        || (node->type == TF_GLOT && node->is_child);
}

void dom_process_node(DOMBuilder *b, XDOMNNode *nodes, int n) {
    b->node_added = 0;
    b->para_count = 0;
    for (int i = 0; i < n; i++) {
        if (is_basic_child(&nodes[i])) {
            b->para_count++;
            b->node_added = 1;
        }
    }
}

/* tests */
static int test_add_on_basic(void) {
    XDOMNNode ns[] = { {.type=TF_G, .is_child=0} };
    DOMBuilder b; dom_process_node(&b, ns, 1);
    return b.node_added == 1 && b.para_count == 1;
}
static int test_no_add_on_non_basic(void) {
    XDOMNNode ns[] = { {.type=TF_A, .is_child=0} };
    DOMBuilder b; dom_process_node(&b, ns, 1);
    return b.node_added == 0 && b.para_count == 0;
}
static int test_glot_child(void) {
    XDOMNNode ns[] = { {.type=TF_GLOT, .is_child=1} };
    DOMBuilder b; dom_process_node(&b, ns, 1);
    return b.node_added == 1 && b.para_count == 1;
}
static int test_glot_not_child(void) {
    XDOMNNode ns[] = { {.type=TF_GLOT, .is_child=0} };
    DOMBuilder b; dom_process_node(&b, ns, 1);
    return b.node_added == 0 && b.para_count == 0;
}

int main(void) {
    int ok = 1;
    ok &= test_add_on_basic();
    ok &= test_no_add_on_non_basic();
    ok &= test_glot_child();
    ok &= test_glot_not_child();
    printf("dom_sensing_test %s\n", ok ? "PASS" : "FAIL");
    return ok ? 0 : 1;
}
EOF

cc -std=c17 -Wall -Wextra -O0 -o dom_builder dom_builder.c && ./dom_builder
```

**验收**：
- 编译零警告。
- 输出 `dom_sensing_test PASS`。
- 改一个测试用例（例如 `test_add_on_basic` 让 ns[0] type=TF_A），应该 FAIL —— 验证 sensing var 真在保护。

### A3 — Break Out Method Object: 把 monster 的临时变量升格成 struct field

```bash
mkdir -p /tmp/ch22-method-obj && cd /tmp/ch22-method-obj

cat > order_calc.c <<'EOF'
/* Break Out Method Object: 把 monster 拆成 OrderCalculator.run() */
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

typedef struct {
    int daily_target;
    double interest_rate;       /* 0.05 = 5% */
    int compensation_percent;   /* 0-100 */
    int total_compensation;
    int marginal_count;
    /* 这些临时变量原是 monster 内部 local, 现升格成 struct field */
} OrderCalculator;

static int marginal_rate(void) { return 3; }

static void recalculate_orders(OrderCalculator *c, int n_orders) {
    c->total_compensation = 0;
    c->marginal_count = 0;
    for (int i = 0; i < n_orders; i++) {
        if (marginal_rate() > 2) {
            /* 计算补偿 */
            int base = c->daily_target * 10;
            int bonus = (base * c->compensation_percent) / 100;
            double with_interest = (double)bonus * (1.0 + c->interest_rate);
            c->total_compensation += (int)with_interest;
            c->marginal_count++;
        }
    }
}

OrderCalculator *order_calc_new(int dt, double ir, int cp) {
    OrderCalculator *c = calloc(1, sizeof(*c));
    c->daily_target = dt;
    c->interest_rate = ir;
    c->compensation_percent = cp;
    return c;
}
void order_calc_run(OrderCalculator *c, int n_orders) {
    recalculate_orders(c, n_orders);
}
void order_calc_free(OrderCalculator *c) { free(c); }

int main(void) {
    OrderCalculator *c = order_calc_new(100, 0.05, 10);
    order_calc_run(c, 5);

    /* sensing 通过 struct field: 不需要 return value */
    int pass = (c->marginal_count == 5) && (c->total_compensation > 0);
    printf("marginal_count=%d total_comp=%d -> %s\n",
           c->marginal_count, c->total_compensation,
           pass ? "PASS" : "FAIL");
    order_calc_free(c);
    return pass ? 0 : 1;
}
EOF

cc -std=c17 -Wall -Wextra -O0 -o order_calc order_calc.c && ./order_calc
```

**验收**：
- 编译零警告。
- 输出 PASS。
- `marginal_count` 和 `total_compensation` 是 struct field —— *test 通过 struct field sensing*，不通过 return value。这证明 "临时变量升格 = free sensing"。

## 六、值得深入思考的问题

- **"monster method"的判定阈值是多少？** Feathers 没给数字。50 行算不算 monster？300 行？1000 行？行业经验值（SonarQube 默认 50，Uncle Bob 给 20）的依据是什么？
- **sensing variable 留在 production 是 design debt 吗？** Feathers 说"拆完删掉"，但 *ch9 / ch10 / ch11* 里 sensing var 经常留下来作为 *test hook*。何时该删？何时该留作 *test-only 字段*？
- **Gleaning Dependencies 的边界**：业务关键 vs 次要的划分由谁来定？*Product Owner*？*Tech Lead*？*QA*？不同人定的边界不一样，跨团队怎么 align？
- **Break Out Method Object 适用规模**：临时变量少于 5 个时拆 class 是不是过度工程化？阈值在哪？
- **有 IDE 但拒绝用 + 手工拆**：什么场景下合理？可能是 IDE 抽出的方法命名太差，需要人调整。但 *什么阈值* 下值得放弃 IDE 的安全保证？
- **跨类 move 一次走对 vs 两次重拆**：经验数据上 *Extract to Current Class First → Move* 的成功率 vs *一次抽对* 的成功率哪个高？需要团队度量。

## 七、本章与全本的关系 + 后续书预告

### 与全本 25 章的关系

- **前置**: ch8 *How Do I Add a Feature?* — 在已有测试的基础上添加 feature；ch21 *I'm Changing the Same Code All Over the Place* — duplicate code 是 monster 的成因之一。
- **本章**: 把 ch8 的 Extract Method 推到 *没 IDE + 没测试* 的恶劣条件；引出 *Sensing Variable / Gleaning / Break Out Method Object / Skeletonize / Find Sequences* 等手法。
- **后续**:
  - **ch23** = 本章"在恶劣条件下别出错"的 *心理纪律* 版 —— hyperaware editing / single-goal editing / preserve signatures / lean on the compiler / pair programming。
  - **ch24** = 本章之后的 *情感管理* —— 在 monster / 没测试 / 团队压力下, 工程师心理过载怎么办。
  - **ch25** = 本章提到所有手法的 *reference catalog* —— 25 个 dependency-breaking technique 的查表。

### 后续书预告

**Working Effectively with Legacy Code** 之后, Feathers 没有专门再写 *legacy code* 的续作。行业里接续的两本书是:

- **Michael C. Feathers — *Working Effectively with Unit Tests* (2014)**: 把 ch2 的 TDD / characterization test 推到 unit test 的 *readability* 和 *maintenance* 维度。
- **Working Effectively with Legacy Code** 的精神在 *Michael Nygard — Release It!* (2018 第 2 版) 第 7-9 章（*Stability Patterns*）和 *Sam Newman — Monolith to Microservices* (2019) 第 5 章（*Migrating Routes*）中延伸。
- **AI 时代的延伸**: *Titus Winters — Software Engineering at Google* (2020) 第 16 章（*Large-Scale Changes*）+ *Hyrum Wright — Hyrum's Law* 的工程化 —— ch22 / ch23 的 hyperaware + 单步编辑原则, 在 *Hyrum's Law* 视角下就是"任何观察到的行为都是契约"的反面 —— refactor 越是 hyperaware, 越是把 implicit 契约固化下来, 反而增大未来的 refactor 阻力。AI 工具 (Cursor / Aider / Claude Code) 的"大块重写"和 Feathers 的"0-count extract"是哲学对立 —— 后续 *AI + Legacy* 系列书 (还没出现, 可能 2027-2028) 会尝试调和这对立。

> *下一章预告*: ch23 探讨 *hyperaware editing* —— 在没有完整 feedback loop 时, 怎么靠 *心理纪律* (single-goal, preserve signatures) + *compiler 协作* (lean on the compiler) + *pair programming* 来保证不 break 既有行为。
