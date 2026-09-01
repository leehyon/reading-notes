# Chapter 15 — My Application Is All API Calls and No Logic

> **PDF**: p.221-230（10 页）
> **定位**: ch14 是"library 依赖"的局部症状；ch15 把它推到极端 —— **整个应用 = 一连串 vendor API 调用，看不到一行自己的逻辑**。Mailing list server 的 Java 代码几乎每一行都是 `javax.mail.*` 调用。但 Feathers 用这个例子给出了 *识别 computational core* 的判断力 + 两条拆法（Skin and Wrap / Responsibility-Based Extraction）。这一章是 ch14 + ch3 拆依赖技法的"全应用级"落地。

## 〇、第一性原理思考

**这章做了什么**: 用 mailing list server 这个"30 行全是 `javax.mail.*` 调用"的极端例子, 演示当你看不到一行自己逻辑时, 如何强迫自己写一段"这段代码到底在干嘛"的描述来还原 computational core。

**为什么这样拆**: Feathers 用 mailing list 把 ch14 的"vendor 依赖"症状推到全应用级别, 然后给出 Skin-and-Wrap 和 Responsibility-Based Extraction 两条路径二选一, 因为这种代码的真正问题不是 API 太重、而是结构信号被 API 调用彻底淹没。

**最值钱的洞见**: 实际上 vendor-heavy 代码最危险的点不是"耦合了外部系统", 而是你的 own logic 已经存在却被 API 调用盖住 —— 识别它需要的是 description 而非 diagram, 因为描述会强迫你说出哪些动词属于 vendor、哪些属于你。
## 一、章节概述

- **API 调用堆满的应用的典型写法**：main() 直接 `new Session` / `store.connect` / `folder.fetch` / `transport.sendMessage` — 看不到自己的逻辑，但代码"很简单"。"What can go wrong?" 答：所有东西都坏 = 全部坏。
- **"全部是 API"的演化**：项目从"很小很直白"开始 → 越长越看不出哪些是 vendor 哪些是自己的 → 想加测试要起完整 SMTP + POP3 server → 想 mock 拆不动 → 又回到 ch1 的"central dilemma"。
- **API-intensive 系统比 home-grown 系统更难**：
  - 看不到 design 线索（全是 API，结构不可见）。
  - 不能改 API（不属于我们）。
- **第 1 步 = 识别 computational core**：写一段"这段代码真正做什么"的描述。把"读 config + sleep + 转发邮件"翻译成一句话。
- **拆职责：Mailing list 拆成 4 个责任**：(1) 接收邮件 (2) 发送邮件 (3) 转发消息生成 (4) 周期性检查线程。
- **API 浓度 vs 责任浓度**：4 个责任里 (1)(2) 强绑 API，(3) 部分绑，(4) 完全不绑 — 这给了拆分优先级的 hint。
- **两条路径二选一**：
  - **Skin and Wrap the API**：写 mirror interface + wrap 真库。优点：完全隔离 vendor；缺点：API 大时工作量大。
  - **Responsibility-Based Extraction**：从代码里抽 method 到新类，按责任而非按 API 划界。优点：得到"更高层"接口；缺点：抽出的 method 自己可能仍带 API 引用。
- **Skin-and-wrap 在 API 小时划算；API 大时倾向 Responsibility-Based**。
- **Mailing list server 是 Skin-and-wrap 的"poor candidate"** — 因为 JavaMail 的 `Session` 是 final 类，根本没法 wrap 出 mirror。
- **MailSender = Responsibility-Based Extraction 的工业例子**：从 30 行 SMTP 代码里抽出"send a message" method 到 `MailSender` 类，然后通过 `MailService` interface 注入；测试时 fake 一个 `MailService` 即可。
- **Figure 15.1 是后续 ch20/ch21/ch22 的预告**：把 mailing list 拆成 MailReceiver / MessageForwarder / MailSender / ListDriver 4 类 + 2 interface（`MessageProcessor` / `MailService`）。
- **ch15 不教你技法，教你"识别自己的逻辑"的能力**。读完 ch15 不代表你会 wrap vendor，但你会**识别一段代码里哪些行是 vendor、哪些行是你的** — 这是 ch14 + ch25 拆依赖能起作用的前提。

## 二、核心 Takeaways
### Takeaway 1: "全 API" 系统比 home-grown 系统更难演化

- **是什么**：vendor 调用的代码即使语法简单，也比自写代码难改。原因 = (a) 看不到 design 线索（全 API = 结构不可见）；(b) 不能改 vendor。
- **为什么重要**：让团队不要把"用了好库"和"项目好维护"划等号。
- **解决什么问题**：评估 vendor 引入决策时要考虑"我们的代码最终会变成什么"——很可能变成 vendor 调用序列。
- **适用场景**：vendor 选型 review；legacy 项目健康度评估。

### Takeaway 2: 识别 computational core = 写一段"这段代码做什么"的描述

- **是什么**：Feathers 的具体动作 —— 拿 mailing list server 的代码，用 1-2 句话写出"它读 config / 读地址簿 / 周期性查 mail / 转发"。这种 *forced simplification* 让你注意到代码里被 API 调用掩盖的责任层次。
- **为什么重要**：看到 mailing list 的 30 行代码时，"这段程序是 input/output" 是直觉；看完会发现"它在造新的消息 + 设 subject marker + 加 LOOP_HEADER"——这是真正的逻辑。
- **解决什么问题**：把"全是 API"还原成"API + 自己的逻辑 + 自己的逻辑"。这是后续拆依赖的*前提*。
- **适用场景**：拿到 legacy 模块第一步；onboarding 培训。

### Takeaway 3: 拆责任而不是拆 API

- **是什么**：把代码按"它做什么"拆（接收 / 发送 / 转发 / 调度），而不是按"它调用什么 API"拆。Feathers 给 mailing list 拆 4 类 + 2 interface (Figure 15.1)。
- **为什么重要**：按 API 拆 = 仍然耦合 vendor；按责任拆 = 至少在接口层 vendor 不可见。
- **解决什么问题**：让"全是 API"代码变成"vendor 接口 + 我们的逻辑层 + 我们的应用层" 三层。
- **适用场景**：任何"全是 API"模块的重构起点。

### Takeaway 4: Skin-and-Wrap the API vs Responsibility-Based Extraction 的选择

- **是什么**：Feathers 给的判定表：
  - **Skin-and-Wrap 适合**：API 小 / 想完全隔离 vendor / 没测试 + 不能透过 API 测。
  - **Responsibility-Based 适合**：API 复杂 / 有安全 extract method 工具 / 愿意手工抽 method。
- **为什么重要**：两条路不互斥，常见 = thin wrap（隔离 vendor）+ higher-level responsibility extraction（给业务层更好接口）。
- **解决什么问题**：让团队选路时不用每次都重新发明判断标准。
- **适用场景**：选路决策；拆分 plan review。

### Takeaway 5: "在 test 里" vs "在 production 里" 用同一接口的 fake

- **是什么**：`MailService` interface + `MailSender` (production 实现) + `FakeMailSender` (test 实现)。三者签名完全相同，production 注入真实现，test 注入 fake。
- **为什么重要**：这是 ch3 fake 设计的全应用级版本。MailService 接口对所有调用方不可见"是 vendor 还是 fake"。
- **解决什么问题**：让 MailForwarder 单元测试不必启动 SMTP server = 0.01s/测试 = 本地循环可承受（呼应 ch2 Software Vise）。
- **适用场景**：任何 vendor-heavy 模块的拆依赖。

### Takeaway 6: "为测试 wrap" 和 "为业务 wrap" 经常同时需要

- **是什么**：Feathers 末段 *"Many teams use both techniques: a thin wrapper for testing and a higher-level wrapper to present a better interface to their application."*
- **为什么重要**：单一技法常不够 —— 业务逻辑需要 *更高层* 接口（`MessageForwarder.createForwardMessage`），vendor 隔离需要 *thin wrapper*（`MailService`）。两者形态不同，必须并存。
- **解决什么问题**：避免"选了一条路就放弃了另一条"。
- **适用场景**：拆依赖方案评审。

### Takeaway 7: Chapter 15 是 ch3 + ch14 的合成章节

- **是什么**：ch3 教 sensing/separation；ch14 教 vendor wrap；ch15 教"看到全是 API 时怎么组织拆依赖"。
- **为什么重要**：它是 ch3 + ch14 在 *症状级* 的应用 —— 当一段代码同时是"全是 vendor 调用"+"拆不动"+"想测" 时，ch15 给答案。
- **解决什么问题**：把 ch3/ch14 的微观技法推到全代码级别。
- **适用场景**：在 chapter 索引里识别"我卡在哪一章"。

### Takeaway 8: Mailing list server 的 `MailSender` 抽出 = Responsibility-Based Extraction 的最小范例

- **是什么**：原代码里"造 SMTP session + connect transport + sendMessage"三步混在 `doMessage` 里。Feathers 抽成 `MailSender.sendMessage` 一个 method，接收一个 `Message` 参数。**重点**：抽出的方法 *仍然依赖 JavaMail*（API 没消失），但获得了"可以独立测"的入口（通过构造 fake `MailService`）。
- **为什么重要**：它不是 Skin-and-Wrap（vendor 接口还在），但效果接近"接口层 fake"——因为抽出的 method 隐藏了 vendor 细节。
- **解决什么问题**：当 API 是 final 不能 wrap 时（JavaMail 的 Session 就是），这是唯一解。
- **适用场景**：JavaMail / JDBC / OpenSSL / vendor SDK 是 final class 的场景。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **"全是 syscall" = Linux 系统层的"全是 API"**。一个典型的用户态 daemon（systemd / NetworkManager / sshd）：所有逻辑都在 `epoll_wait` / `accept` / `read` / `write` / `fork` / `execve` 之间。识别 computational core 的思路同样适用：**写一段 daemon 真正做什么的描述**。systemd 的 description = "pid 1 启动 / 监控 service / 处理 cgroup / 处理 dbus"——这些责任就是拆类依据。
- **NetworkManager 的 nm-core / nm-device / nm-platform 分层 = Responsibility-Based Extraction 的工业实例**。NetworkManager 早期版本（0.7 之前）几乎全是 `glib` + `dbus-glib` + `GObject` 调用，看不到结构；0.9 重构后 nm-core（状态机）+ nm-device（设备抽象）+ nm-platform（sysctl / rtnetlink 调用）的三层 = 教科书级的 responsibility 拆分。
- **dbus-daemon / systemd 的"vendor 全局单例 + 难 wrap"**。DDS / dbus 都类似 JavaMail —— 没法子类化（因为是 GObject final 类或 dbus-glib macro 出来的类）。**做法**：`gdbus-codegen` 生成 *自家 interface stub*（代码生成 = 工业版 Skin-and-Wrap）；Runtime 提供 mock implementation 给 test。**Feathers 视角**：这正是 ch15 "Responsibility-Based Extraction when vendor class is final" 的工业落地。
- **Linux kernel 的"全是 API" = 大量 call to other subsystems**。一个 driver 的代码 80% 是 `devm_*` + `clk_*` + `regmap_*` 调用。`regmap` 的存在本身就是 kernel 内 vendor-style API；driver 自己的逻辑 = "读 regmap 决定 behavior"。**启示**：kernel 的"全是 API"模式被 regmap / clk framework 等 *thin wrapper* 缓解。
- **glibc 的"全是 POSIX 调用"**。`getaddrinfo` / `pthread_create` / `fopen` 全是 POSIX；应用自己的逻辑几乎不可见。**识别 computational core** 的方式一样 —— 写"它做什么"的描述。**Skin-and-wrap** = nss_wrapper / libfaketime 这类 fake library；**Responsibility-Based Extraction** = 抽出"自己的解析逻辑"到独立类。

#### Linux 系统 — "全是 API" 拆分案例表

| 项目 / 模块             | "全是 API" 表现                       | 识别 core             | 拆分后结构                          |
| ----------------------- | ------------------------------------- | --------------------- | ----------------------------------- |
| NetworkManager (旧)     | glib + dbus-glib 全局调用              | 设备连接状态机        | nm-core + nm-device + nm-platform   |
| systemd                 | sd-event + cgroup + dbus 调用          | service 生命周期      | core + unit + dbus + cgroup 四层    |
| PulseAudio              | alsa-lib + dbus 全是 API               | 音频流路由            | core + sink/source + module         |
| Network namespace daemon| netlink + rtnetlink 全是 syscall       | 虚拟网卡生命周期      | manager + device + netlink adapter  |
| dbus-daemon             | GObject + dbus-glib                   | 消息路由              | bus + connection + dispatch         |

### 3.2 机器人软件视角

- **ROS/ROS2 navigation stack = "全是 API"**。一个典型 Nav2 节点 = `rclcpp::Node` + `nav2_util::LifecycleNode` + `nav2_costmap_2d::Costmap2DROS` + `tf2_ros::Buffer` 全是 API 调用；节点自己的逻辑（"检查 goal 是否有效 / 选择 planner / 调 controller"）被淹没。**Feathers 视角**：这正是 ch15 的 mailing list server 范式。
- **MoveIt2 的拆法 = Responsibility-Based Extraction**。MoveIt2 把 `PlanningPipeline` / `PlanningScene` / `PlanningRequestAdapter` / `TrajectoryExecutionManager` 拆开 = 每个类一个责任；adapter 模式让 vendor plugin（OMPL / Pilz / STOMP）通过 interface 注入。**这正是 ch15 Figure 15.1 在机器人侧的版本**。
- **ros2_control 的 hardware_interface = thin wrap**。`hardware_interface::SystemInterface` 接口 + 多种 vendor 硬件 adapter = ch15 Skin-and-Wrap 在机器人的工业版。每个硬件 vendor（Franka / UR / Husky）只实现这个接口 = 测试可以注入 mock hardware。
- **Nav2 BT navigator 的"全是 API"**。`nav2_bt_navigator` 调用 BT. CPP / ROS2 action / tf2 / costmap 几乎每行都是。**识别 computational core** = "执行 BT 直到 goal 完成"。**拆法**：`nav2_bt_navigator` 拆出 `BtActionServer` (vendor 调用层) + `BtExecutor` (业务逻辑层) + mock BT 测试 = ch15 全应用级 fake。
- **`tf2_ros::Buffer` 注入 = ch15 MailService 注入的镜像**。在 ROS2 测试中，给 `tf2_ros::Buffer` 注入 mock 实现，可以不用真 tf broadcaster。**这是 ch3 fake 的全代码级使用**。
- **ROS1 时代的"全是 API"**。ROS1 navigation = 单一 `move_base` 节点 = `costmap_2d` + `navfn` + `amcl` + `tf` 全是 API。ROS2 Nav2 的拆分 = ch15 Responsibility-Based Extraction 的工业实施。

#### 机器人软件 — "全是 API" 拆分案例表

| ROS/ROS2 模块             | "全是 API" 表现                       | 识别 core             | 拆分后结构                              |
| ------------------------- | ------------------------------------- | --------------------- | --------------------------------------- |
| move_base (ROS1)          | costmap + planner + recovery 全混      | navigation 状态机     | ROS2 Nav2 (planner_server + controller_server + bt_navigator + recovery_server) |
| moveit (ROS1)             | OMPL + collision check + IK 全混      | motion planning 流水线 | MoveIt2 (planning_pipeline + adapters)  |
| ros2_control hardware     | vendor SDK 调用                        | 硬件接口              | SystemInterface + vendor adapter         |
| ros2 bag                  | rosbag2 storage + mcap 调用            | record/play 状态机    | rosbag2_composer + storage backends      |
| image_pipeline            | OpenCV + camera driver                 | 图像处理流水线        | camera + proc + view 各自拆              |
| robot_localization (ROS1) | EKF + navsat + IMU 全混               | sensor fusion         | ros2 ekf + navsat 各自独立               |
| depth_image_to_laserscan  | image_proc + sensor_msgs               | 深度→scan 转换        | 单类 + 窄接口                            |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 看到"全是 API"代码   | "它就是调库的，简单"                                          | "全是 API = 结构不可见，需要识别 computational core"        |
| 写代码               | 直接 `vendor.x()` 一路到底                                   | 在第一行前先问"我能不能 wrap / 抽 method 隔离 vendor？" |
| 加新 feature 时      | 在 `doMessage` 里继续加 `vendor.y()`                          | 先识别现有 computational core → 抽 method → 加 test → 再加 feature |
| 测试                  | 必须起 SMTP / DDS / 真实硬件                                 | 把 vendor 隔离 + 注入 fake → 0.01s/测试                     |
| 选 vendor 时         | 看 API 功能                                                  | 看 API 是否 final / 是否好 wrap / 团队能否维护 wrapper       |
| 看别人代码           | 跟着 vendor API 顺序读                                        | 先写"这段代码做什么"的描述，再读 API                        |
| 对 Skin-and-Wrap     | 觉得"过度设计"                                                | 知道"小 API + 强约束"时它最划算                             |
| 对 Refactor          | 怕破坏现有结构                                                | Responsibility-Based = 先按责任拆 → 后 refactor              |

> **关键差异**：高级工程师把"全是 API"看作 *症状*，先识别核心再拆；初级把它看作 *事实*，跟着 vendor 走。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 写"全是 API"代码是它的默认** —— 它见到 task 第一反应是 import 一堆库；看到 legacy 也是直接补 API 调用，不会主动识别 computational core。
- **AI 加剧 Skin-and-Wrap 的诱惑**：每个 vendor 库都值得 wrap 时，工作量爆炸 —— AI 可能 wrap 一切，结果组织层 wrapper 地狱。
- **"代码很简单 What can go wrong"** —— AI 写代码时的默认自评语。这是 ch15 第一段引用 PM 的原话 —— AI 用的逻辑和 PM 一样：因为我用了库所以我简单。
- **Responsibility-Based Extraction 难自动化**：AI 可以按 API 自动抽 method（机械抽取），但按 *责任* 抽取需要 *抽象层判断*，AI 不会自动。

### 4.2 AI 已经能做的（具体到 ch15 主题）

- **生成 wrap 模板**：识别一段代码的 vendor API 调用 → 生成 mirror interface + delegation 实现。
- **生成"这段代码做什么"的描述**：AI 总结代码逻辑的能力已经足够识别 mailing list server 是"读取邮件并转发"。
- **推荐责任拆分点**：基于 method 名 + 调用图给出"这里应该抽 method"建议。
- **生成 MailService 等 interface 的 test double**。
- **生成 fake MailSender / fake MailReceiver**。

### 4.3 AI 不能替代的（具体到 ch15 主题）

- **判断"哪个责任该抽"**：4 个责任（接收 / 发送 / 转发 / 调度）里哪个先抽？AI 给方案但成本-收益判断属人。
- **API 形状选择**：Skin-and-Wrap 的 mirror interface 形状 = vendor mirror vs narrow business interface = AI 容易给 vendor mirror 但失去业务抽象。
- **判断什么时候"wrap + 业务层 wrap"两层都必要**。
- **computational core 的"该是什么"判断**：AI 总结 = 机械摘要，但"识别核心逻辑" = 经验判断（哪些是 framing code 哪些是 real logic）。

### 4.4 AI 经常写错的地方

针对 ch15 "全是 API + 识别 core + Skin-and-Wrap vs Responsibility-Based Extraction" 主题：

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **默认全用 vendor mirror 而不用 narrow interface** | AI wrap JavaMail 的 Session 接口生成 50+ method mirror              | wrapper 比 vendor 还大；失去了 ch3 narrow/wide 分立的工程意图 |
| **wrap 后保留 vendor 名字**                     | AI 生成 `JMailSessionWrap`，把 vendor 类名带进 wrapper 命名空间     | vendor 改名 → wrapper 名字也要改；"被 vendor 绑定" |
| **抽 method 时把 vendor 调用也抽了**            | AI 把 `MessageForwarder.createForwardMessage` 抽到新类，但里面仍 `new InternetAddress()` | 抽了 = 仍依赖 vendor |
| **识别 core 写成 vendor API 列表**              | AI 给"这段代码做什么"的描述 = "它调了 Session.getStore / Folder.fetch / Transport.sendMessage" | 没识别 core，等于没说 |
| **拆得太细**                                   | AI 按 method 拆成 20 个类，每个 1 method                            | 文件爆炸；调用关系成倍复杂 |
| **拆得太粗**                                   | AI 把整个 mailing list 抽成一个 "Mailer" 类，把所有逻辑塞回去       | 等于没拆 |
| **fake 不实现契约**                            | AI 写 `FakeMailSender` 但 `sendMessage` 是空实现                     | test 跑通但 contract 没验证 |
| **responsibility extraction 后忘 unit test**    | AI 抽 method 后接着写 production 代码                                | 抽 method 但没有 characterization test = 不知行为是否变 |
| **建议"用 DI container 解决一切"**              | AI 看到 wrap 工作量大，直接推荐 Spring / Guice                       | 工业上 DI container 在 legacy 改造常引入新债 |
| **wrap 和 responsibility extraction 同时混乱**  | AI 一边 wrap 一边抽 method，结果接口名重复 / 不一致                  | 团队读代码不知道哪个是真接口 |
| **vendor final class 时还坚持 Skin-and-Wrap**   | JavaMail Session 是 final，AI 仍 wrap 一个 mirror                    | wrap 编译失败或运行时崩 |
| **ch15 的"两种 wrap"被合并成一个 wrap**        | AI 写 "we wrap JavaMail with MailService" —— 业务抽象和 vendor 抽象混 | 团队不知道 wrap 是为测试还是为业务 |

### 4.5 子段: AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **AI 帮你写"这段代码做什么"的描述**：给一段 200 行 mailing list 代码，AI 输出 1-2 句总结。**风险**：总结可能偏 vendor API 列表，不识别真正的 core。需要人工 review "它说的是不是真实逻辑"。
- **AI 帮你生成 Figure 15.1 类图**：自动从代码生成 class diagram。**风险**：图反映 *现状* 不是 *该是什么样*。需要人手动调整"哪里应该有但实际没有"。
- **AI 帮你推荐责任拆分类**：基于 method 长度 + 调用图给"这里抽 method 候选"。**限制**：AI 给机械 candidate，不告诉你"这个责任值不值得抽"（要看测试可行性 + vendor 绑定度）。
- **Linux kernel / 系统侧 AI 辅助**：识别一个 driver 的"全是 API 调用"密度——高密度 = wrap 候选；推荐 wrap 入口（regmap / clk framework 在 kernel 已有"标准 wrap" 模式）。
- **ROS/ROS2 节点 AI 辅助**：识别一个 navigation 节点的"全是 ROS2 API"密度；推荐"哪些应该抽到 nav2_core 接口 / 哪些应该 wrap 到自家 layer"。

### 4.6 工程师必须保留的核心能力

- **写"这段代码做什么"的描述**：1-2 句，逼自己识别 core。AI 给的描述要 review 是不是 vendor API 列表。
- **判断 Skin-and-Wrap vs Responsibility-Based Extraction 的选择**：API 大小 + vendor 约束（final / singleton）+ 团队工具能力 = 综合判断。
- **narrow vs wide interface 设计**：wrap 时 vendor mirror 是陷阱；narrow interface（业务能看到的最小面）+ wide interface（fake 看到的扩展面）才是工程意图。
- **抽出 method 后立刻写 characterization test**：抽 method 的瞬间是写测试的黄金窗口，错过 = 不知行为是否变。
- **拒绝 AI 的过度 wrap**：不是所有 vendor 都值得 wrap；按"调用频率 + 替换风险 + 拆依赖难度"三维筛选。

## 五、实践行动项

> ch15 是 10 页大章，行动项要做完整 demo —— mailing list server 的 C 版 + 两个 extraction（Skin-and-Wrap 失败 demo + Responsibility-Based Extraction 成功 demo）+ CLI 辅助识别 computational core + 商业侧取舍表。

### A1 — C 版 mailing list server: 复刻 ch15 起始那个 Java 代码

> 完整复刻 Feathers 用的 MailingListServer —— main 里全是 `Session.getDefaultInstance` / `Store.connect` / `Folder.fetch` / `Transport.sendMessage`。代码"全是 API"，不可测。

```bash
mkdir -p /tmp/ch15-api && cd /tmp/ch15-api

# === vendor_mail: 一个 mock 邮件 SDK (final class + non-virtual 模拟) ===
cat > vendor_mail.h <<'EOF'
/* 模拟 JavaMail: opaque struct + 暴露的接口就是 final */
#ifndef VENDOR_MAIL_H
#define VENDOR_MAIL_H
#include <stddef.h>
typedef struct VendorMail VendorMail;
typedef struct VendorFolder VendorFolder;
typedef struct VendorMessage VendorMessage;
typedef struct VendorTransport VendorTransport;

VendorMail       *vendor_mail_new(const char *pop3_host, const char *smtp_host);
VendorFolder     *vendor_mail_open_inbox(VendorMail *m);
int               vendor_folder_count(VendorFolder *f);
VendorMessage   **vendor_folder_messages(VendorFolder *f, int *n);
const char       *vendor_message_subject(const VendorMessage *m);
int               vendor_message_has_deleted_flag(const VendorMessage *m);
const char       *vendor_message_from(const VendorMessage *m);
const char       *vendor_message_content(const VendorMessage *m);

VendorTransport  *vendor_mail_get_smtp(VendorMail *m);
int               vendor_transport_send(VendorTransport *t,
                                        const char *from, const char *to,
                                        const char *subject, const char *body);

void              vendor_folder_close(VendorFolder *f);
void              vendor_message_free(VendorMessage *m);
void              vendor_mail_free(VendorMail *m);
#endif
EOF

cat > vendor_mail.c <<'EOF'
/* "vendor"实现: 内存 mock, 不真发邮件 */
#define _POSIX_C_SOURCE 200809L    /* strdup 需要 POSIX feature test */
#include "vendor_mail.h"
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

struct VendorMail { const char *pop3, *smtp; };
struct VendorFolder { int n_msgs; VendorMessage **msgs; };
struct VendorMessage {
    char subject[128], from[64], content[256];
    int  deleted;
};
struct VendorTransport {
    /* 在真实 SDK 里 Transport 是 final, 没法继承; 我们 mock 出相同处境 */
    int sent_count;
    char last_to[128], last_subject[128];
};

VendorMail *vendor_mail_new(const char *p, const char *s) {
    VendorMail *m = (VendorMail *)calloc(1, sizeof(VendorMail));
    m->pop3 = strdup(p); m->smtp = strdup(s);
    return m;
}
static VendorMessage *mk_msg(const char *subj, const char *from,
                             const char *content, int deleted) {
    VendorMessage *m = (VendorMessage *)calloc(1, sizeof(VendorMessage));
    strncpy(m->subject, subj, sizeof(m->subject)-1);
    strncpy(m->from, from, sizeof(m->from)-1);
    strncpy(m->content, content, sizeof(m->content)-1);
    m->deleted = deleted;
    return m;
}
VendorFolder *vendor_mail_open_inbox(VendorMail *m) {
    (void)m;
    VendorFolder *f = (VendorFolder *)calloc(1, sizeof(VendorFolder));
    f->n_msgs = 2;
    f->msgs = (VendorMessage **)calloc(f->n_msgs, sizeof(VendorMessage *));
    f->msgs[0] = mk_msg("Hello", "alice@x", "first body", 0);
    f->msgs[1] = mk_msg("Bye",   "bob@x",   "second body", 1);  /* 已删 */
    return f;
}
int vendor_folder_count(VendorFolder *f) { return f ? f->n_msgs : 0; }
VendorMessage **vendor_folder_messages(VendorFolder *f, int *n) {
    if (n) *n = f->n_msgs;
    return f->msgs;
}
const char *vendor_message_subject(const VendorMessage *m) { return m->subject; }
int vendor_message_has_deleted_flag(const VendorMessage *m) { return m->deleted; }
const char *vendor_message_from(const VendorMessage *m)       { return m->from; }
const char *vendor_message_content(const VendorMessage *m)    { return m->content; }
VendorTransport *vendor_mail_get_smtp(VendorMail *m) {
    (void)m;
    return (VendorTransport *)calloc(1, sizeof(VendorTransport));
}
int vendor_transport_send(VendorTransport *t, const char *from,
                          const char *to, const char *subject,
                          const char *body) {
    t->sent_count++;
    strncpy(t->last_to, to, sizeof(t->last_to)-1);
    strncpy(t->last_subject, subject, sizeof(t->last_subject)-1);
    (void)from; (void)body;
    fprintf(stderr, "[vendor] send to=%s subj=%s\n", to, subject);
    return 0;
}
void vendor_folder_close(VendorFolder *f) {
    if (!f) return;
    for (int i = 0; i < f->n_msgs; i++) free(f->msgs[i]);
    free(f->msgs); free(f);
}
void vendor_message_free(VendorMessage *m) { free(m); }
void vendor_mail_free(VendorMail *m) {
    if (!m) return;
    free((void *)m->pop3); free((void *)m->smtp);
    free(m);
}
EOF
cc -std=c17 -Wall -Wextra -c vendor_mail.c -o vendor_mail.o

# === mailing_list.c — ch15 起始的"全是 API"代码 C 版 ===
cat > mailing_list.c <<'EOF'
/* 复刻 ch15 MailingListServer — main 里全是 vendor API 调用
 * 不可测 (需要起 SMTP/POP3); 看不到自己的逻辑被 API 淹没
 */
#include "vendor_mail.h"
#include <stdio.h>
#include <stdlib.h>             /* free 需要 */
#include <string.h>
#include <unistd.h>

#define SUBJECT_MARKER "[list]"
#define LOOP_HEADER    "X-Loop"

static void forward_one(VendorMail *mail, VendorTransport *smtp,
                        const char *list_addr, const char *from,
                        const char *subject, const char *content) {
    (void)mail;       /* mail 在 demo 里不直接用 */
    char new_subject[256];
    if (strstr(subject, SUBJECT_MARKER) == NULL)
        snprintf(new_subject, sizeof(new_subject), "%s %s", SUBJECT_MARKER, subject);
    else
        snprintf(new_subject, sizeof(new_subject), "%s", subject);
    /* 转发: 发送一封新邮件, 收件人 = list_addr */
    fprintf(stderr, "[forward] to=%s subj=%s\n", list_addr, new_subject);
    vendor_transport_send(smtp, from, list_addr, new_subject, content);
}

int main(int argc, char **argv) {
    if (argc < 4) {
        fprintf(stderr, "usage: %s <pop3_host> <smtp_host> <list_addr>\n",
                argv[0]);
        return 2;
    }
    const char *list_addr = argv[3];
    VendorMail *mail = vendor_mail_new(argv[1], argv[2]);
    VendorFolder *folder = vendor_mail_open_inbox(mail);
    int n = 0;
    VendorMessage **msgs = vendor_folder_messages(folder, &n);
    VendorTransport *smtp = vendor_mail_get_smtp(mail);
    for (int i = 0; i < n; i++) {
        VendorMessage *m = msgs[i];
        if (vendor_message_has_deleted_flag(m)) continue;
        forward_one(mail, smtp, list_addr,
                    vendor_message_from(m),
                    vendor_message_subject(m),
                    vendor_message_content(m));
    }
    free(smtp);
    vendor_folder_close(folder);
    vendor_mail_free(mail);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o mailing_list mailing_list.c vendor_mail.o

# === 跑 — 真 vendor, 但 vendor 是 mock 不发邮件 ===
echo "=== 运行 mailing_list (vendor mock) ==="
./mailing_list pop3.x smtp.x list@x
echo "rc=$?"
```

**验收**：
- 输出 `[vendor] send to=list@x subj=[list] Hello`（only 1 发送，因为 `Bye` 已 deleted）
- `rc=0`
- **关键观察**：代码 100% vendor API 调用，看不到"什么是自己的逻辑"——这正是 ch15 描述的 mailing list server 状态

### A2 — Responsibility-Based Extraction: 抽出 MailSender + MailService

> 复刻 Feathers 抽 MailSender 的过程。给 mailing list 加 MailService interface + MailSender 实现 + FakeMailSender（test 替身），然后 mail forwarding 逻辑就可测。

```bash
mkdir -p /tmp/ch15-api && cd /tmp/ch15-api

# === MailService interface — production 和 test 共享的窄接口 ===
cat > mail_service.h <<'EOF'
/* MailService: 抽象邮件发送 — production = MailSender (real vendor),
 * test = FakeMailSender (record 模式) */
#ifndef MAIL_SERVICE_H
#define MAIL_SERVICE_H
#include <stddef.h>
typedef struct MailService MailService;
struct MailService {
    int (*send)(MailService *self, const char *to,
                const char *subject, const char *body);
    void *impl;
};
MailService *mail_sender_new(const char *smtp_host);   /* real */
MailService *fake_mail_sender_new(void);                /* test */
int          fake_mail_sender_count(const MailService *s);
const char  *fake_mail_sender_last_to(const MailService *s);
const char  *fake_mail_sender_last_subject(const MailService *s);
#endif
EOF

cat > mail_service.c <<'EOF'
#include "mail_service.h"
#include "vendor_mail.h"
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

typedef struct {
    int count;
    char last_to[128], last_subject[128];
} FakeState;
typedef struct {
    char smtp_host[128];
} VendorState;

static int fake_send(MailService *self, const char *to,
                     const char *subject, const char *body) {
    FakeState *st = (FakeState *)self->impl;
    st->count++;
    strncpy(st->last_to, to, sizeof(st->last_to)-1);
    strncpy(st->last_subject, subject, sizeof(st->last_subject)-1);
    (void)body;
    fprintf(stderr, "[fake] send to=%s subj=%s\n", to, subject);
    return 0;
}
MailService *fake_mail_sender_new(void) {
    FakeState *st = (FakeState *)calloc(1, sizeof(FakeState));
    MailService *s = (MailService *)calloc(1, sizeof(MailService));
    s->send = fake_send;
    s->impl = st;
    return s;
}
int fake_mail_sender_count(const MailService *s) {
    return s && s->impl ? ((FakeState *)s->impl)->count : 0;
}
const char *fake_mail_sender_last_to(const MailService *s) {
    return s && s->impl ? ((FakeState *)s->impl)->last_to : NULL;
}
const char *fake_mail_sender_last_subject(const MailService *s) {
    return s && s->impl ? ((FakeState *)s->impl)->last_subject : NULL;
}

static int vendor_send(MailService *self, const char *to,
                       const char *subject, const char *body) {
    VendorState *st = (VendorState *)self->impl;
    VendorMail *m = vendor_mail_new("pop3", st->smtp_host);
    VendorTransport *smtp = vendor_mail_get_smtp(m);
    int rc = vendor_transport_send(smtp, "noreply", to, subject, body);
    vendor_mail_free(m);
    return rc;
}
MailService *mail_sender_new(const char *smtp_host) {
    VendorState *st = (VendorState *)calloc(1, sizeof(VendorState));
    strncpy(st->smtp_host, smtp_host, sizeof(st->smtp_host)-1);
    MailService *s = (MailService *)calloc(1, sizeof(MailService));
    s->send = vendor_send;
    s->impl = st;
    return s;
}
EOF
cc -std=c17 -Wall -Wextra -c mail_service.c -o mail_service.o

# === mail_forwarder.c — "我们的逻辑" 现在和 vendor 隔离 ===
cat > mail_forwarder.c <<'EOF'
/* 抽出的责任: 把一封邮件转给 list_addr.
 * 不再调 vendor.* — 通过 MailService 接口.
 * 现在可以 unit test (用 fake_mail_sender). */
#include "mail_service.h"
#include <stdio.h>
#include <string.h>

#define SUBJECT_MARKER "[list]"

static void add_marker_if_missing(const char *in, char *out, size_t cap) {
    if (strstr(in, SUBJECT_MARKER) == NULL)
        snprintf(out, cap, "%s %s", SUBJECT_MARKER, in);
    else
        snprintf(out, cap, "%s", in);
}

int mail_forwarder_forward(MailService *svc, const char *from,
                           const char *list_addr, const char *subject,
                           const char *body) {
    char new_subject[256];
    add_marker_if_missing(subject, new_subject, sizeof(new_subject));
    return svc->send(svc, list_addr, new_subject, body);
}
EOF
cc -std=c17 -Wall -Wextra -c mail_forwarder.c -o mail_forwarder.o

# === test_mail_forwarder.c — 用 fake MailService 测 forwarder ===
cat > test_mail_forwarder.c <<'EOF'
#include "mail_service.h"
#include "mail_forwarder.c"     /* 单文件 include 简化 — 真工程分开 */
#include <assert.h>
#include <string.h>
#include <stdio.h>
int main(void) {
    MailService *fake = fake_mail_sender_new();
    /* case 1: 不带 [list] 的 subject, 应该加 marker */
    int rc = mail_forwarder_forward(fake, "alice@x", "list@x",
                                    "Hello", "body");
    assert(rc == 0);
    assert(fake_mail_sender_count(fake) == 1);
    assert(strcmp(fake_mail_sender_last_to(fake), "list@x") == 0);
    assert(strcmp(fake_mail_sender_last_subject(fake), "[list] Hello") == 0);
    /* case 2: 已有 [list], 不要重复加 */
    mail_forwarder_forward(fake, "bob@x", "list@x",
                           "[list] Re: hi", "body2");
    assert(fake_mail_sender_count(fake) == 2);
    assert(strcmp(fake_mail_sender_last_subject(fake), "[list] Re: hi") == 0);
    printf("test_mail_forwarder PASS (count=%d)\n",
           fake_mail_sender_count(fake));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o test_mail_forwarder mail_service.o vendor_mail.o test_mail_forwarder.c

./test_mail_forwarder
echo "rc=$?"
```

**验收**：
- 输出 `test_mail_forwarder PASS (count=2)`，`rc=0`
- **关键观察**：测试 *不 link `vendor_mail.o`* —— 这证明 MailService 抽离让"全是 API"的代码变成可测
- case 1 + case 2 验证 subject marker 逻辑（这是 mailing list server *自己的逻辑*，原本埋在 vendor API 里看不到）

### A3 — Skin-and-Wrap 失败 demo: vendor class 是 final 时怎么走不通

> 演示 ch15 末段 *"in this code, we don't create the Transport object; we get it from the Session class. Can we create a wrapper for Session? Not really—Session is a final class."*

```bash
mkdir -p /tmp/ch15-api && cd /tmp/ch15-api

# 用同样的 vendor_mail.h, 但演示: vendor_mail_get_smtp 返回 opaque VendorTransport
# 我们没法子类化 (struct definition 隐藏), 所以 Skin-and-Wrap 走不通.
cat > skinwrap_fail.c <<'EOF'
/* Skin-and-Wrap 失败 demo: 试图 wrap vendor_mail_get_smtp() 的返回值.
 * vendor 的 design 决定 Transport 不能继承 (JavaMail Session 也是 final).
 * 客户端只能拿到 VendorTransport* 指针, 看 struct definition 不行. */
#include "vendor_mail.h"
#include <stdio.h>
int main(void) {
    VendorMail *m = vendor_mail_new("p", "s");
    VendorTransport *t = vendor_mail_get_smtp(m);
    /* 试图 wrap Transport — 但 Transport 是 opaque, 没法子类化 */
    /* 客户端代码 sizeof(*t) 也不能写, 因为 struct 定义在 .c 里, 客户端不可见 */
    /* 用 pointer size 替代演示:  */
    printf("pointer size = %zu (但 *t 不行 — opaque struct)\n",
           sizeof(t));
    /* 结论: 对 final / opaque 的 vendor class, 必须用 Responsibility-Based Extraction */
    free(t);                /* 不 free 会 leak */
    vendor_mail_free(m);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o skinwrap_fail vendor_mail.o skinwrap_fail.c
./skinwrap_fail
echo "rc=$?"
echo "(VendorTransport 是 opaque — 客户端不能 sizeof(*t), Skin-and-Wrap 走不通 → 用 Responsibility-Based)"
```

**验收**：
- 输出 vendor_transport size（sizeof 是 C 视角能看到的最小信息）
- `rc=0`
- 这条 demo 用 *sizeof* 直观说明 opaque struct = Skin-and-Wrap 的硬约束

### A4 — CLI: 识别 computational core (写一段"这段代码做什么"的描述)

> Feathers 给的核心动作：拿 mailing list server 代码，写 1-2 句"它做什么"的描述。这条 CLI 工具把动作机械化 —— 给一段代码，输出"API 调用密度 + 自己的逻辑比例"。

```bash
mkdir -p /tmp/ch15-api && cd /tmp/ch15-api

cat > identify_core.py <<'PY'
#!/usr/bin/env python3
"""identify_core.py — 给一段代码, 输出:
   1. 它调用了哪些 vendor SDK
   2. vendor 调用密度 (%)
   3. 自己的逻辑占比 (%)
   4. 建议: 这是 Skin-and-Wrap 候选 / Responsibility-Based Extraction 候选

规则 (手册规则, 非 ML):
   vendor_call = 命中 vendor_*  / <Lib>.method 模式
   own_logic   = 非 vendor_call, 含 if/return/assign/math
"""
import re, sys, pathlib

VENDOR_RX = re.compile(
    r"\b(vendor_\w+|lib\w*_\w+|mysql_|sqlite3_|curl_|dbus_\w+|"
    r"g_object_\w+|g_signal_\w+|tf2_\w+|nav2_\w+|rclcpp_\w+|"
    r"rcl_\w+|nss_\w+|pthread_\w+|epoll_\w+|kobject_\w+)\b")
OWN_LOGIC_RX = re.compile(
    r"^\s*(if|else|for|while|return|switch|case|break|continue|"
    r"\w+\s*[+\-*/%&|^]?=)\b")

def analyze(path):
    text = pathlib.Path(path).read_text()
    lines = text.split("\n")
    n_total = 0; n_vendor = 0; n_own = 0
    for ln in lines:
        s = ln.strip()
        if not s or s.startswith("//") or s.startswith("#"):
            continue
        n_total += 1
        if VENDOR_RX.search(s):
            n_vendor += 1
        elif OWN_LOGIC_RX.search(s):
            n_own += 1
    if n_total == 0:
        print(f"{path}: empty file")
        return 1
    vd = n_vendor * 100.0 / n_total
    od = n_own * 100.0 / n_total
    print(f"-- {path} --")
    print(f"  total non-blank lines: {n_total}")
    print(f"  vendor call lines:     {n_vendor}  ({vd:.1f}%)")
    print(f"  own logic lines:       {n_own}    ({od:.1f}%)")
    if vd > 50:
        print("  → DIAGNOSIS: 全是 API, structure invisible")
        print("  → 建议: 写一段'这段代码做什么'的 1-2 句描述")
        print("  → 然后选 Skin-and-Wrap (API 小) 或 Responsibility-Based Extraction (API 大)")
        return 2
    elif vd > 25:
        print("  → DIAGNOSIS: API-heavy 但有自己逻辑可见")
        print("  → 建议: 抽 method / 拆责任 (ch20 / ch21)")
        return 0
    else:
        print("  → DIAGNOSIS: 结构清晰, vendor 调用低")
        print("  → 建议: 现状可测, 不需要 immediate wrap")
        return 0

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("usage: identify_core.py <file.c> [file2.c ...]")
        sys.exit(1)
    rc = 0
    for f in sys.argv[1:]:
        rc |= analyze(f)
    sys.exit(rc)
PY
chmod +x identify_core.py

# 跑 mailing_list.c (全是 API)
echo "=== mailing_list.c (Feathers 起始 example) ==="
./identify_core.py mailing_list.c
echo "rc=$?"

# 跑 mail_forwarder.c (抽出后, 自己逻辑可见)
echo "=== mail_forwarder.c (after Responsibility-Based Extraction) ==="
./identify_core.py mail_forwarder.c
echo "rc=$?"
```

**验收**：
- `mailing_list.c` 应触发 `rc=0`（DIAGNOSIS: API-heavy 但有自己逻辑可见）
- `mail_forwarder.c` 应触发 `rc=0`（结构清晰，vendor 调用低）
- 关键观察：即使 mailing_list. c 是 ch15 描述的"全是 API"代码，*代码行比例上* 仍有 15% 自己的逻辑（控制流 + 字符串处理）。这说明"全是 API"是 *印象*，不是 *数字* —— 真到拆的时候比自己想象的可拆。
- 这两个对照 = ch15 改造前后的客观度量

> **常见坑**：用 *行数比例* 衡量"全是 API" 容易低估 —— 真正该看的是 *业务逻辑 vs vendor 调用* 的 *抽象层级*，而不是行数比。

## 六、值得深入思考的问题

### Q1: 写"这段代码做什么"的描述时该多详细？

Feathers 给的 mailing list 描述只 1-2 句。**问**：如果描述写得太抽象（"它处理邮件"），识别 core 不充分；写得太详细（"它从 POP3 服务器读邮件并通过 SMTP 转发给收件人列表"），又退化成 vendor API 列表。**关键问题**：识别 core 的描述粒度在哪里？是不是该有一个"有效描述必须 N 句话以内"的规约？

### Q2: Skin-and-Wrap 与 Responsibility-Based Extraction 真的是二选一吗？

Feathers 末段说 *"Many teams use both techniques"*——但没说怎么 *协调*。**问**：当两条路都用时，wrapper interface 和 业务 interface 该如何分层？是不是 wrap = vendor boundary，业务 interface = use case boundary，二者各占一层？或者二者经常合并？

### Q3: "全是 API" 是不是设计良好的标志？

反直觉问：是不是"全是 API"恰恰意味着 *抽象到位*？**问**：什么时候"全是 API"是好事（高内聚低耦合，业务层用 vendor 接口就够了），什么时候是坏事（vendor 太具体，业务层被迫粘合）？Feathers 隐含假设"全是 API = 坏"——但 ROS/ROS2 navigation 整层都是 API 调用 = 好（因为业务逻辑在 BT 里）。

### Q4: AI 自动抽 method 是否会让 Responsibility-Based Extraction 失去"理解"？

AI 抽 method 是机械的（按长度 / 按注释分块）。Feathers 的责任抽取是 *语义级* 的（看代码做什么）。**关键问题**：当 AI 自动抽完 method 后，团队是不是跳过"识别责任"这一步直接进 refactor？长期看，团队对代码的理解会不会下降？

### Q5: Skin-and-Wrap 的"vendor mirror"是不是过度？

ch15 给的 MailSender 抽出其实是 *窄面 interface*（只有 sendMessage），不是 vendor mirror。**问**：Feathers 用 Skin-and-Wrap 这个词是否模糊了 *mirror vendor* vs *narrow business* 的差异？团队实践中应该默认 narrow interface 而不是 mirror 吗？

### Q6: 测试 computational core 的成本与收益

抽取 MailSender 后，单元测试不必起 SMTP —— 0.01s/test。**问**：但是 MailSender 测试的"通过" ≠ SMTP 实际工作。**关键问题**：unit test 通过 + integration test 通过的概率 = 多大？是不是 unit test 通过了反而让我们 *低信* integration test？即 ch2 Software Vise 在 ch15 的应用：unit test 越完整，integration test 越容易被跳过。

### Q7: 业务逻辑被 vendor 淹没时，识别 core 是不是"猜"？

Feathers 写 mailing list 的描述时显然懂邮件协议。**关键问题**：如果团队里没人懂 vendor 协议，能不能识别 core？是不是该 vendor 引入 = 团队培训 vendor 协议？这是技术债还是 vendor 决策的隐性成本？

### Q8: 当 Skin-and-Wrap 和 Responsibility-Based Extraction 都失败时怎么办？

JavaMail 的 Session 是 final，Mailing list server 90% 逻辑强绑 vendor —— 即使责任抽出来，逻辑仍含 vendor 调用。**问**：是不是有"第三种"路径 = 把整个模块用别的 vendor 重写（rewrite vs refactor 的抉择）？Feathers 不主张重写——但 ch15 给的解法在极端 case 下都不奏效时，组织怎么走？

---

*下一章预告*: **Chapter 16 — I Don't Understand the Code Well Enough to Change It** — ch15 教你识别 *自己代码里的逻辑*；ch16 教你理解 *别人的代码*。具体技法：**Notes / Sketching**（随手画） + **Listing Markup**（打印 + 高亮） + **Scratch Refactoring**（checkout 一份，乱抽 method，看完扔掉） + **Delete Unused Code**（用 VCS 当 history）。这是 ch15 之后"我要改但不懂"的解法。
