# Chapter 14 — Dependencies on Libraries Are Killing Me

> **PDF**: p.219-220（2 页）
> **定位**: 整本书最短一章。它把"过度依赖第三方库"列为一个独立症状——并非因为它罕见，而因为它常常被误读为"我们买了库=降低了成本"。Feathers 给的反话：**每一次直接调用 = 一次失去的 seam**。这一章是 ch15 的前置（API 调用塞满整个应用 = 库依赖的极端表现），也是 ch25 catalog 中 Skin and Wrap / Encapsulate Global Reference 的对应章节。

## 〇、第一性原理思考

**这章做了什么**: 把"裸调 vendor 库"识别为一种独立症状, 并演示一旦直接 `new VendorLib.Foo()` 或依赖 vendor 单例, 你就在每一个调用点丢了一个 seam, 后续测试和替换都变成不可达。

**为什么这样拆**: Feathers 用 once dilemma 和 restricted override dilemma 这两个 corner case 把"vendor 依赖"问题钉死在 seam 模型上, 逼你承认 wrap 不是优化、而是没别的路可走。

**最值钱的洞见**: 实际上 wrap 的成本不是"多写一层接口", 而是失去"以后再换"这个选项本身 —— 不写 wrapper 不会让你"以后真的不换", 而是让你"以后根本换不动"。
## 一、章节概述

- **代码复用 ≠ 降低总成本**：第三方库能省前期开发时间，但如果全代码"裸用"（promiscuous use），后期改不动 — vendor 一涨版税/一断供，整个应用立刻塌方。Feathers 反复用过类似例子：vendor 抬价 → 应用不赚钱 → 换不掉 → 重写。
- **每次硬编码库类调用 = 失去一个 seam**。"Every hard-coded use of a library class is a place where you could have had a seam." 这是 ch4 seam 模型的直接应用：库类是 final/sealed/method-non-virtual 时，seam 缺失。**对策：写一个 thin wrapper** —— 让自己代码对 wrapper 编程，wrapper 委托给真库，测试用 fake 替身。
- **平台大战时代的反讽**：2004 写书时 Java/. NET 都在"广撒网"地造库 — 把生态做大对客户有利，但对"被绑在某个生态"的项目而言反而脆弱。**Feathers 暗线**：平台锁定 = 库锁定的放大版。
- **两类 dilemma：once dilemma + restricted override dilemma**：
  - **once dilemma**（单例困境）：库假设某类只有一个实例 → `Introduce Static Setter` 等多数拆依赖技法用不上（ch25 p.372）。**唯一出路：wrap 单例本身**。
  - **restricted override dilemma**（覆盖受限困境）：C++/Java 允许声明 `final` / `sealed` / non-virtual → 没法在子类里 override 替身 → 没法做 sensing / separation。**解法**：coding convention（假装 public method 非 virtual，在 test 里 override）有时和 language feature 等效。
- **"约束过严的设计"是错的**：library designer 用 language feature 把生产环境封死，**忘了代码也要在 test 里跑**。Feathers 主张：在 production 用规约代替强制，保留 test override 的能力。
- **避免在代码里"撒"直接库调用**：自证预言的规避 ("we'll never change") 是 legacy 的起点。Wrap 越早越便宜；裸用越久越贵。
- **chapter 14 是 ch25 拆依赖技法的语料库**：Skin and Wrap the API（ch25 catalog 后还会再提）/ Encapsulate Global Reference / Introduce Static Setter — 全部在本章"为什么需要"那一侧列了原始动机。

## 二、核心 Takeaways

### Takeaway 1: 每一个裸用库类的地方 = 失去一个 seam

- **是什么**：调用 `new VendorLib.Foo()` / `VendorLib.Static.bar()` / 直接传 `VendorLib.Baz` 做参数 — 这些 *每一处* 都把"换实现"的位置锁死在 vendor 的 API 上。原本可以换成 fake 的，现在不行。
- **为什么重要**：seam 数 ≈ 演化自由度。Feathers 的 seam 模型（ch4）在这里兑现 — 不留 seam = 测不了 = 改不动 = legacy。
- **解决什么问题**：让团队在写第一行库调用前，先停下来 5 秒：*"我能不能让它通过我的 wrapper 走？"*
- **适用场景**：所有 vendor 库、平台 SDK（JavaMail / . NET System.* / Win32 / POSIX socket）、公司内部 cross-team library。

### Takeaway 2: Vendor 单例困境 — Wrap 是唯一出路

- **是什么**：很多库（Logger / Config / Runtime / DB connection pool）假设全程序只有一个实例。`Introduce Static Setter`（ch25）能拆，但拆了影响 production 行为；其它多数拆依赖技法对单例不奏效。**只剩一条路**：把单例自己也包一层。
- **为什么重要**：单例是"singleton disguised as class" — 它把"全局状态"伪装成 OO 接口，**所有拆 seam 工具都失效**。
- **解决什么问题**：当 ch9/ch10 的 Extract Interface / Parameterize Constructor 都无法用时（因为拿不到 setter），wrapper 是后手。
- **适用场景**：log4j / slf4j / Java `Runtime` / . NET `Trace` / ROS2 `rclcpp::init` 全局 — 这些都是行业实际碰到过的单例难题。

### Takeaway 3: 受限覆盖困境 — Coding convention 可替代 language feature

- **是什么**：Java `final` / C# `sealed` / C++ non-virtual method 限制子类 override。Feathers 指出：**"假装"public method 在 production 里是 non-virtual（团队规约），实际保留 virtual，在 test 里选择性 override** —— 两条路得到等效结果。
- **为什么重要**：它给"测试需要继承" vs "production 想限制继承"一个共存解 — 不必动用语言强制。
- **解决什么问题**：拒绝"为了测试就把 production 改成可继承"的设计妥协，也拒绝"为了 design 干净就把 test 路径封死"。
- **适用场景**：自己写 framework 时；library 提供方告知"内部使用请勿继承"时；长期维护的 team library。

### Takeaway 4: Wrap 越早越便宜 — "我们以后不换" 是自证预言

- **是什么**：Feathers 警告 *"Avoid littering direct calls to library classes in your code. You might think that you'll never change them, but that can become a self-fulfilling prophecy."* 这条直击 PM 的常见判断"我们定了 vendor 就别折腾"。**事实上**：不写 wrapper → 改不动 → 真"换不掉" → 自证预言成真。
- **为什么重要**：把 wrap 当 default hygiene，不是"等项目大了再说"。
- **解决什么问题**：把 wrapper 的成本计入"库引入决策"里 — vendor 选择 = 同时选了"为它写 wrapper"。
- **适用场景**：技术选型评审；vendor 锁定风险评估；架构 review。

### Takeaway 5: Ch14 给的是动机，ch25 给的是技法 — 配合使用

- **是什么**：ch14 不教具体怎么 wrap — 它只告诉读者"必须 wrap"。ch25 的 `Skin and Wrap the API` / `Encapsulate Global Reference` / `Introduce Static Setter` 给具体步骤。
- **为什么重要**：症状章（part II）和技法章（part III）的对应 — ch14 是症状，ch25 是药方。
- **解决什么问题**：避免"知道该 wrap 但不知道从哪行开始"的瘫痪。
- **适用场景**：读完 ch14 觉得"这正是我们的症状" → 直接翻 ch25 对应小节看代码示例。

### Takeaway 6: Library designer 的"约束洁癖"应让位 testability

- **是什么**：Feathers 直言："Library designers who use language features to enforce design constraints are often making a mistake. They forget that good code runs in production and test environments." 用 `final` / `sealed` 表达"我不想让你继承"看似合理，**结果** = 用户的 test harness 拿不到替身。
- **为什么重要**：它把"design constraint"和"test constraint"看作*两种约束* — 前者靠 convention / javadoc / convention；后者靠 seam。
- **解决什么问题**：library 作者在"加 final"前先答一次：*"如果用户想 fake 我，他们能吗？"*
- **适用场景**：内部 framework 设计 review；公共 SDK 的开放性评估。

### Takeaway 7: 库之战 — "广撒网" 战略对客户的隐性伤害

- **是什么**：2004 年 Java/. NET 都在扩展平台让用户"离不开" — 这种生态绑架 = 库锁定的平台级版本。Feathers 把 ch14 放在平台战背景下写，给读者一个 *meta 视角*。
- **为什么重要**：单看 ch14 它只讲"代码层 wrap"；放回背景，它讲"组织层 wrap" = 多 vendor / 多语言抽象。
- **解决什么问题**：让团队警惕"我们已被平台绑死"，从架构层留出 vendor-agnostic 中间层（典型：Hexagonal Architecture / Ports & Adapters 的"port"）。
- **适用场景**：技术战略 1-3 年规划；vendor 决策会议。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **glibc 是 final-class 的现实版本**。`FILE *` / `FILE*` 的接口不让你继承；`FILE` 的 method 都通过函数指针表（vtable-equivalent）但**不是 C++ vtable**。结果：想 fake `fopen` / `fread`，没有 Java 式的"继承 + override"路径，**只剩 LD_PRELOAD**（见 ch3 A4）+ **接口层 wrap**两条。glibc 内部其实做了 internal symbol abstraction（`__fxprintf` 等）作为 wrap 切入口 — 这就是 ch14 的"vendor 在自己内部也用了 wrapper"。
- **systemd / dbus 的"vendor 内部 wrapper"**：dbus 的 `DBusConnection` 在内部用 `_dbus_connection_*` private symbol 层与上层 `dbus_connection_*` 隔开。**为什么**：上层是稳定的 API；下层可以改。**这正好对应 Feathers 的 thin wrapper 思路 — 不是为 testability 而为 API 稳定性**，但产物是同一类构造：interface 稳定 + impl 可换。
- **Linux kernel 的 module API** = 强制抽象层。`module_init` / `module_exit` / `module_param` 都在 module 上方留接口 — 内核 *本身* 在做 ch14 的事情（把硬件细节 wrap 成 module API）。**启示**：即使是不能改的代码（kernel），也通过接口层允许 module 替换 — 给测试 / driver 替换提供位置。
- **POSIX socket / 文件描述符 = once dilemma 的标准实例**。一个进程里 "the network namespace" 是单例 — 你不能 setUp/tearDown 切换 namespace 除非 unshare/clone — 这正是 `Introduce Static Setter` 失效的场景。**工业对策**：test 用 `unshare -n` / network namespace mock + LD_PRELOAD `socket()` fake。**启示**：ch14 的"唯一出路 wrap" 在 Linux 下转译为"唯一出路 LD_PRELOAD / namespace isolation"。
- **vendor 锁定风险 — 历史上的真事**：MySQL → MariaDB 分叉时，许多 patch 没及时迁移 = 库锁定的工程代价。Linux 系统侧的教训：永远要有一个"vendor-agnostic 中间层"（典型：`libdbus` 抽象），不要让应用代码里全是 `mysql_*` 调用。

#### Linux 系统 — 库依赖 wrap 取舍表

| 库类型                  | seam 难度     | 推荐拆法                          | 工业实例                          |
| ----------------------- | ------------- | --------------------------------- | --------------------------------- |
| glibc 标准 C 库         | 极难（final） | LD_PRELOAD + thin wrapper         | `faketime` / `fakeclock` 项目     |
| POSIX socket / 文件 I/O | 难（系统调用）| LD_PRELOAD / namespace 隔离        | `unshare -n` + `nss_wrapper`      |
| 系统日志（syslog）      | 中            | 引入 `Logger` interface           | log4c / spdlog / glog             |
| 时间 `clock_gettime`    | 中            | function-pointer 注入 / LD_PRELOAD | `libfaketime` / ch3 A4             |
| DB 库（libpq / sqlite3）| 难            | Repository pattern + Adapter       | SOCI / ODBC                       |
| IPC（dbus / shared mem）| 中            | interface 抽象 + fake impl        | GTest's dbus mock                 |

### 3.2 机器人软件视角

- **ROS/ROS2 的 `rclcpp::init` = 全局单例困境**。`rclcpp::init` 一调，进程进入 DDS 中间件；多个 `rclcpp::Node` 实例可以建，但 context 全局唯一。**这正是 ch14 once dilemma**。**做法**：`ros2_control` 的 `hardware_interface::SystemInterface` 把硬件 wrap 成接口；测试时给一个 mock hardware 不必真 rclcpp:: init。
- **ros2_control 的硬件抽象 = ch14 wrap 的工业最佳实践**。每个真实硬件（Husky / Spot / Franka / UR）通过 hardware interface 暴露给上层 controller；上层的 controller 不 import 任何 hardware SDK。**这等于 Feathers 的 thin wrapper** — 只不过动机是"硬件抽象"而非"测试"，产物形态一致。
- **`ros2 bag` / `image_transport` = API 占用整个应用的范例**。一个 navigation 节点 80% 代码是 `ros2 bag` 调用 / `image_transport::Publisher` 调用 — ch15 就要讲这种"全是 API"的场景怎么拆。**ch14 给前置**：先 wrap 这些 API，ch15 再用 wrapping 的结果做 extract class。
- **MoveIt2 / Nav2 plugin 系统 = "把库当 seam 来用"**。两者都强制业务代码通过 pluginlib 加载 planner / controller — 而不是 `new NavFnPlanner()`。**pluginlib 本身就是 ch14 wrap 思想的工业级实现**：vendor plugin 被包成 `Planner` 接口。
- **ROS1 的 `tf` 是 once dilemma 的实际痛点**。`tf::TransformListener` 一个进程一个实例；unit test 想 swap 出去 = 几乎要重写 `tf` 自己。**ROS2 的 `tf2_ros::Buffer` 改成可注入** = 正是 ch14 的"wrap 唯一出路"落地。
- **机器人 software supply chain** = vendor 锁定的工程风险放大版。一个工业机器人项目可能含：相机 SDK（vendor A）+ 机械臂 SDK（vendor B）+ AGV SDK（vendor C）+ 自家算法层。**任何裸用 vendor SDK 的代码 = vendor 涨价 / 跑路 = 整个项目塌方**。**架构铁律**：所有 vendor SDK 调用必须经自家 wrapper。

#### 机器人软件 — 库依赖 wrap 启示表

| 组件                  | 现状 seam 难度 | ch14 wrap 形式                  | 工业实例                        |
| --------------------- | -------------- | ------------------------------- | ------------------------------- |
| ROS2 DDS / rclcpp     | 极难（全局）   | `rclcpp::Node` wrapper + mock init | `rclcpp::Node::make_shared(fakes)` |
| 相机 SDK（如 RealSense）| 难            | `CameraInterface` + Adapter     | `realsense2_camera` 包装        |
| 机械臂 SDK            | 难             | `RobotInterface` + state struct | `moveit2` / `ros2_control`      |
| 导航 / SLAM           | 中             | Pluginlib                       | Nav2 plugin system               |
| MoveIt! planner       | 中             | `planning_interface::Planner`   | MoveIt's plugin interface         |
| 传感器驱动（IMU/LiDAR）| 中            | `sensor_msgs::msg::Imu` 替换 fake | `ros2 bag` play + mock         |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 看到 vendor 库       | "它能跑就行"                                                  | "我先写 wrapper — 即使 vendor 不变，wrapper 让我 unit test 有了 fake" |
| 写库调用             | 直接 `vendor.Foo()` 满代码                                   | 先 import 自己 wrapper，调 wrapper                          |
| 库升级时             | 改所有调用                                                   | 改 wrapper 一处即可                                          |
| 听到"以后不换 vendor"| 同意                                                          | 警觉：自证预言                                               |
| 看到 `final` 类      | 觉得"安全"                                                   | 警觉：testability 损失                                       |
| 拆依赖技术           | 只用 mock framework                                          | mock + LD_PRELOAD + wrapper + namespace 全套                |
| 写公共库             | "为了安全全部 final"                                          | "internal symbol 留开口，public API 稳定"                   |
| 处理 vendor 单例     | 接受单例                                                      | wrap 单例或换 namespace                                      |

> **关键差异**：高级工程师把"用库"看作 *绑定事件*（commitment），必须 wrap 才能保持解绑能力；初级把"用库"看作 *省事*。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **LLM 倾向"裸调"**：AI 写 Java/Python 时不会主动写 wrapper —— 它看到 vendor lib 就直接 import → 全代码 = 裸用 vendor 的形态（这正是 ch14 的症状）。
- **vendor 数量爆炸**：AI 加剧"调啥有啥"的诱惑（HTTP client / ORM / 消息队列 / ML SDK）→ wrap 工作量成倍上升。
- **AI 写 final/sealed 没自觉**：AI 推荐的 design 经常"为了安全把 class 设 final"，正击 ch14 restricted override dilemma 的反面。
- **ch14 描述的"自证预言"在 AI 时代更显著**：AI 用一个库好用 → 它继续用 → 代码全是这个库 → 团队不敢换 → 真换不掉。

### 4.2 AI 已经能做的（具体到 ch14 主题）

- **识别裸用**：给一段代码，AI 可找出所有 `vendor_lib.*` 直接调用点 + 推荐 wrap 位置（extract function 级别）。
- **生成 thin wrapper 初稿**：识别所有 vendor class → 生成 mirror interface + delegation 实现。准确率 60-80%。
- **指出 `final` / `sealed` / `non-virtual`**：静态扫描 + 提示"这处可能影响 testability"。
- **测 wrap 前/后测试覆盖率变化**：引入 wrapper 前后 fake 测试覆盖密度对比。

### 4.3 AI 不能替代的（具体到 ch14 主题）

- **判断"是否值得 wrap"**：每个库调用点 wrap 都有成本（额外文件 / 调用栈 / 心智）。AI 会过度 wrap 或 wrap 不到位 — 这是 judgement。
- **wrapper interface 形状**：AI 给的 wrapper 经常 1:1 镜像 vendor API，丧失了"窄面 vs 宽面"的设计意图。
- **vendor 锁定的战略评估**：库的 exit cost = 商业风险 + 培训成本 + 数据迁移。AI 看不到组织层。
- **`final` 是否该加** 的真实判断：AI 默认全加 final，需要人 reasoning "test path 需要 override 怎么办"。

### 4.4 AI 经常写错的地方

针对 ch14"裸用库 = 失去 seam"主题，AI 典型误判：

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **AI 默认全 final**                            | AI 生成 `public final class OrderProcessor`；使用者想 fake 写不出   | testability 永久损失，team 不得不 fork library |
| **wrap 太厚**                                  | AI 把 vendor SDK 包成 6 层 abstract factory + DI container           | 文件膨胀 10×；读代码找不到 vendor API 在哪调用 |
| **wrap 1:1 镜像 vendor**                       | AI wrapper 方法签名 = vendor 方法签名，不做 narrow interface         | 失去了 ch3 Takeaway 6 "窄面 vs 宽面" 的工程意图 |
| **wrap 后没了 logging / retry**                | AI 写 wrapper 只 `return vendor.method()`，省了 vendor 自身的 retry / cache | production 行为悄悄变了 |
| **vendor 单例被 wrap 但 wrap 还是单例**        | AI 把 `Runtime.getRuntime()` 改成 `RuntimeWrapper.get()` 但内部仍调 getRuntime | 拆了等于没拆 |
| **wrap 后忘了写 fake**                         | AI 写完 wrapper 后接着写 production 调用，没 fake                    | testable interface 存在但没测试，仍是 legacy |
| **直接 `new vendor.Foo()` 而不是 wrap**        | AI 看到 wrap 文件存在，但仍直接在 production code 里 `new`           | wrap 是装饰品，没人真用 |
| **建议"直接用 vendor 的 interface"**            | vendor 类没有 interface，AI 给"加个 interface 字段" — 但 vendor 没 interface | 行不通：vendor 类是 final，根本不能 assign 给 arbitrary interface |
| **vendor 出新版后 wrapper 静默 broken**         | vendor 把 method 改名，AI 没改 wrapper                               | 编译能过，调用 `vendor.method()` 抛 `NoSuchMethodError` |
| **wrap 太轻，无 error mapping**                | AI wrapper 只透传 vendor exception，不做 exception translation      | test fake 时必须重写异常类型；production 测试两套异常模型 |

### 4.5 子段: AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **Linux kernel / 系统 daemon 的库依赖 AI 辅助**：
  - AI 给一段 syscall wrapper 标注"哪些是 vendor 直接调用（glibc / libstdc++）哪些是自家 wrap" — 一眼看出 seam 缺失密度。
  - 但 AI 对 *glibc 不可替换* 的现实不会主动标注 — 它会建议"加 LD_PRELOAD 替身"，但它不知道 glibc 的 symbol versioning 在某些版本拦不住 fake。
- **ROS/ROS2 节点的库依赖 AI 辅助**：
  - AI 给一个 launch 文件标注"哪些 node 直接 import 了 vendor SDK（RealSense / Spot / UR）" — 这些 node 都是 wrap 候选。
  - AI 给 MoveIt2 / Nav2 配置推荐 wrap plugin，但实际 Nav2 已有 `nav2_core::Planner` plugin interface — AI 推荐时常和已有 pluginlib 重叠。
- **AI 不会自动做组织层 wrap**：例如识别"我们的订单系统 = vendor SDK + 自家算法 1:1 调用" —— AI 看不到组织层的 vendor 锁定风险。

### 4.6 工程师必须保留的核心能力

- **判断"哪些 vendor 调用值得 wrap"**：不是 100%，按"调用频率 + 替换风险 + 拆依赖难度"三维评估。
- **设计 wrapper interface 形状**：窄面（生产可见）vs 宽面（fake 可见）的两端分立。
- **vendor 单例困境的对策选择**：wrap / namespace isolation / LD_PRELOAD / 重写 — 哪个划算。
- **coding convention 替代 `final`**：团队规约 vs 强制语言特性的成本收益。
- **拒绝 AI 的过度 wrap / 不足 wrap 双向建议**：review 每一处 vendor 调用。
- **vendor 锁定的组织层评估**：合同 / 培训 / 数据迁移 = AI 不可见的隐性成本。

## 五、实践行动项

> 本章最短，只有 2 页。4 个行动全部聚焦在 **vendor 库依赖的具体 wrap 模式** 上：thin wrapper / once dilemma wrap / final class 通过 convention 解套 / 用 grep 自测裸用密度。

### A1 — 给一个"全是 vendor 调用"的 mock 程序写 thin wrapper，然后测 fake

> **完整复刻 ch14 中心示例** — vendor `vendor_math` 有 final class `VendorCalc`，`add()` 是 non-virtual，不能继承。Feathers 给的解法 = wrap 出 `Calc` interface + `VendorCalcAdapter` 委派 + `FakeCalc` 测试替身。

```bash
mkdir -p /tmp/ch14-lib && cd /tmp/ch14-lib

# === vendor_math — 一个 final-class, non-virtual 的 mock 库 ===
cat > vendor_math.h <<'EOF'
/* 模拟 vendor SDK: final class, non-virtual method */
#ifndef VENDOR_MATH_H
#define VENDOR_MATH_H
#include <stddef.h>
typedef struct VendorCalc VendorCalc;     /* opaque, "final" */
VendorCalc *vendor_calc_new(int seed);
void        vendor_calc_free(VendorCalc *);
int         vendor_calc_add(VendorCalc *self, int a, int b);  /* non-virtual */
#endif
EOF

cat > vendor_math.c <<'EOF'
#include "vendor_math.h"
#include <stdlib.h>
struct VendorCalc { int seed; };
VendorCalc *vendor_calc_new(int seed) {
    VendorCalc *v = (VendorCalc *)malloc(sizeof(VendorCalc));
    v->seed = seed;
    return v;
}
void vendor_calc_free(VendorCalc *v) { free(v); }
int vendor_calc_add(VendorCalc *self, int a, int b) {
    /* vendor 的"特性": seed 为奇数时 +1, 偶数时正常 */
    if (self->seed & 1) return a + b + 1;
    return a + b;
}
EOF
cc -std=c17 -Wall -Wextra -c vendor_math.c -o vendor_math.o

# === 我们的 wrap 层 ===
cat > calc.h <<'EOF'
/* thin wrapper: 暴露 Calc interface, hide VendorCalc (final) */
#ifndef CALC_H
#define CALC_H
typedef struct Calc Calc;
struct Calc {
    int (*add)(Calc *self, int a, int b);
    void *impl;       /* vendor 或 fake — SUT 看不到 */
};
Calc *vendor_calc_adapter_new(int seed);
Calc *fake_calc_new(void);
int   fake_calc_calls(const Calc *c);
#endif
EOF

cat > calc.c <<'EOF'
#include "calc.h"
#include "vendor_math.h"
#include <stdlib.h>
#include <string.h>

typedef struct { int n; } FakeState;
typedef struct { VendorCalc *vc; } VendorState;

static int fake_add(Calc *self, int a, int b) {
    FakeState *st = (FakeState *)self->impl;
    st->n++;
    return a + b;          /* fake 永远准 */
}
Calc *fake_calc_new(void) {
    static FakeState s = {0};
    Calc *c = (Calc *)malloc(sizeof(Calc));
    c->add = fake_add;
    c->impl = &s;
    return c;
}
int fake_calc_calls(const Calc *c) {
    return c && c->impl ? ((FakeState *)c->impl)->n : 0;
}

static int vendor_add(Calc *self, int a, int b) {
    return vendor_calc_add(((VendorState *)self->impl)->vc, a, b);
}
Calc *vendor_calc_adapter_new(int seed) {
    VendorState *st = (VendorState *)malloc(sizeof(VendorState));
    st->vc = vendor_calc_new(seed);
    Calc *c = (Calc *)malloc(sizeof(Calc));
    c->add = vendor_add;
    c->impl = st;
    return c;
}
EOF
cc -std=c17 -Wall -Wextra -c calc.c -o calc.o

# === SUT: 只看 Calc interface ===
cat > order.c <<'EOF'
#include "calc.h"
int order_total(Calc *c, int price, int qty) {
    return c->add(c, price, qty);   /* 看不到 vendor / fake 的差异 */
}
EOF
cc -std=c17 -Wall -Wextra -c order.c -o order.o

# === 测试: 用 fake, 不必 link vendor_math ===
cat > test_order.c <<'EOF'
#include "calc.h"
#include <assert.h>
#include <stdio.h>
extern int order_total(Calc *, int, int);
int main(void) {
    Calc *f = fake_calc_new();
    assert(order_total(f, 10, 3) == 13);
    assert(order_total(f, 5,  2) ==  7);
    assert(fake_calc_calls(f) == 2);    /* fake 记录 */
    printf("test_order PASS (fake count=%d)\n", fake_calc_calls(f));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o test_order order.o calc.o vendor_math.o test_order.c

# === 跑 ===
./test_order
echo "rc=$?"
echo "--- 检查: test_order.c 不 include vendor_math.h ---"
if grep -q 'vendor_math.h' test_order.c; then
    echo "FAIL: test_order.c 直接 include 了 vendor 头 — wrap 没起到隔离作用"
    exit 1
else
    echo "OK: test_order.c 通过 Calc interface 调用, 不直接 include vendor"
fi
echo "--- 检查: vendor_math.o 被链接了 (因为 calc.o 引用了 vendor 符号) ---"
echo "    这是预期的: 隔离点是 *源代码 include 层*, 不是 *链接层*."
echo "    真工程常用 -fvisibility=hidden + shared lib 边界做 link-time 隔离."
```

**验收**：
- `./test_order` 输出 `test_order PASS (fake count=2)`，`rc=0`
- 关键观察：`test_order.c` *不 include `vendor_math.h`* —— wrap 把 vendor 隔离在 calc. c 里；test 只看 Calc interface
- 这才是 ch14 wrap 的工程本意：**隔离点是源代码 include 层**，不是 link 层（除非用 shared library + symbol visibility）

> **掉过的坑**（重要！）：
> 1. **隔离在 include 层，不在 link 层**。`calc.c` 引用了 vendor 符号 → 必须 link `vendor_math.o` → test binary 仍含 vendor symbol。要做到 link-time 隔离必须用 `-fvisibility=hidden` + shared library boundary。本 demo 用 *include 隔离* 演示 wrap 的工程意图。
> 2. **fake 的 state 用 `static`** 在 demo 里没问题，但**真工程不要 static 全局** —— 多个 fake instance 会共享 state。本 A1 是 demo 妥协。

### A2 — Vendor 单例 wrap（once dilemma）：绕开 `vendor_global_get()` 不能 setter

> **完整复刻 ch14 第二大 dilemma**：vendor 把单例锁死在 `vendor_global_get()` 里，没有 setter，没法 `Introduce Static Setter`。Feathers 解法 = **wrap 单例本身**。

```bash
mkdir -p /tmp/ch14-lib && cd /tmp/ch14-lib

# === vendor: 全局单例, 没法 set, 只能 get ===
cat > vendor_global.h <<'EOF'
#ifndef VENDOR_GLOBAL_H
#define VENDOR_GLOBAL_H
typedef struct { int level; } VendorConfig;
const VendorConfig *vendor_global_get(void);   /* 没法 setter */
void vendor_global_set_level(int level);       /* 但有这一个 setter */
#endif
EOF

cat > vendor_global.c <<'EOF'
#include "vendor_global.h"
static VendorConfig g_cfg = { .level = 1 };
const VendorConfig *vendor_global_get(void) { return &g_cfg; }
void vendor_global_set_level(int level) { g_cfg.level = level; }
EOF
cc -std=c17 -Wall -Wextra -c vendor_global.c -o vendor_global.o

# === wrap: 提供 ConfigProvider interface + 用 thread-local 替身 ===
cat > config_provider.h <<'EOF'
#ifndef CONFIG_PROVIDER_H
#define CONFIG_PROVIDER_H
typedef struct ConfigProvider ConfigProvider;
struct ConfigProvider {
    int (*level)(ConfigProvider *self);
    void *impl;
};
ConfigProvider *vendor_config_adapter_new(void);
ConfigProvider *fake_config_new(int level);   /* test 侧 */
void            fake_config_set_level(ConfigProvider *c, int level);
#endif
EOF

cat > config_provider.c <<'EOF'
#include "config_provider.h"
#include "vendor_global.h"
#include <stdlib.h>

typedef struct { int level; } FakeState;

static int fake_level(ConfigProvider *self) {
    return ((FakeState *)self->impl)->level;
}
ConfigProvider *fake_config_new(int level) {
    FakeState *st = (FakeState *)malloc(sizeof(FakeState));
    st->level = level;
    ConfigProvider *c = (ConfigProvider *)malloc(sizeof(ConfigProvider));
    c->level = fake_level;
    c->impl = st;
    return c;
}
void fake_config_set_level(ConfigProvider *c, int level) {
    ((FakeState *)c->impl)->level = level;
}

static int vendor_level(ConfigProvider *self) {
    (void)self;
    return vendor_global_get()->level;
}
ConfigProvider *vendor_config_adapter_new(void) {
    ConfigProvider *c = (ConfigProvider *)malloc(sizeof(ConfigProvider));
    c->level = vendor_level;
    c->impl = NULL;
    return c;
}
EOF
cc -std=c17 -Wall -Wextra -c config_provider.c -o config_provider.o

# === 测试: 不依赖 vendor_global 的全局 state ===
cat > test_config.c <<'EOF'
#include "config_provider.h"
#include "vendor_global.h"     /* 这个 include 是必要的, 因为我们要直接读 vendor 全局验证 */
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>             /* free 需要 */
int main(void) {
    /* 用 fake — 不会污染 vendor 全局 */
    ConfigProvider *f = fake_config_new(7);
    assert(f->level(f) == 7);
    fake_config_set_level(f, 99);
    assert(f->level(f) == 99);
    /* 关键: vendor 全局仍是 1, 没被 fake 污染 */
    printf("vendor_global level = %d (expect 1)\n",
           vendor_global_get()->level);
    assert(vendor_global_get()->level == 1);
    printf("test_config PASS — fake isolated from vendor global\n");
    free(f);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o test_config config_provider.o vendor_global.o test_config.c

./test_config
echo "rc=$?"
```

**验收**：
- 输出 `vendor_global level = 1 (expect 1)` + `test_config PASS — fake isolated from vendor global`，`rc=0`
- 这证明 wrap 把 fake 和 vendor 隔离开；**测试运行不依赖 vendor 全局状态**

### A3 — 统计自己仓库的 vendor 裸用密度

> 实战 ch14 第一守则：*"Every hard-coded use of a library class is a place where you could have had a seam."* —— 写个 grep 脚本数一下。

```bash
mkdir -p /tmp/ch14-lib && cd /tmp/ch14-lib

cat > vendor_density.sh <<'EOF'
#!/bin/bash
# vendor_density.sh — 统计 vendor 库裸用密度.
# 用法: ./vendor_density.sh <src_dir> <vendor_namespace_regex>
# 例:   ./vendor_density.sh ./src 'vendor_'
# 输出: 总行数, vendor 命中数, 密度 (%)

set -e
DIR="${1:-.}"
PATTERN="${2:-vendor_}"
TOTAL=$(grep -rE '[[:alnum:]_]' "$DIR" --include='*.c' --include='*.h' \
        --include='*.py' --include='*.java' --include='*.cpp' 2>/dev/null \
        | wc -l)
# 用 \b 作为 word boundary, vendor_ 后面跟 [a-zA-Z0-9_] 即可, 不需要 ^
# 注意: \b 是 GNU grep -E 支持, 多数 Linux 发行版可用; macOS 用 -E 同样支持
HITS=$(grep -rE "\\b${PATTERN}[a-zA-Z0-9_]" "$DIR" --include='*.c' --include='*.h' \
       --include='*.py' --include='*.java' --include='*.cpp' 2>/dev/null \
       | wc -l)
if [[ "$TOTAL" -eq 0 ]]; then
    echo "no source files found in $DIR"
    exit 1
fi
DENSITY=$(awk "BEGIN {printf \"%.1f\", $HITS*100.0/$TOTAL}")
echo "vendor_density: $HITS / $TOTAL = $DENSITY%"
if awk "BEGIN {exit !($DENSITY > 5.0)}"; then
    echo "WARN: vendor density > 5%, wrap candidates abundant"
    exit 2
fi
echo "OK: density <= 5%"
EOF
chmod +x vendor_density.sh

# 用本会话前面生成的 vendor_math.* / vendor_global.* 试一下
mkdir -p src
cp vendor_math.h vendor_math.c vendor_global.h vendor_global.c src/

# 跑 (vendor_ 前缀)
./vendor_density.sh ./src 'vendor_'
echo "rc=$?"
```

**验收**：
- 输出 `vendor_density: X / Y = Z%`（高密度，因为 src 里基本全是 vendor）
- `rc=2` 触发 WARN（"vendor density > 5%, wrap candidates abundant"）
- 这脚本是 ch14 "Every hard-coded use = lost seam" 的工程度量版
- **常见坑**：如果 pattern 用 `^vendor_` 而非 `vendor_`，grep 只匹配行首，整个目录 0 命中 → density=0 假阴性。必须用 `\b` word boundary 或裸 `vendor_`。

### A4 — Restricted override dilemma: 用 convention + 注释替代 `final`

> **复刻 ch14 第三大 dilemma**：vendor 类有 `final` 修饰；我们想 override 又不能改 vendor。Feathers 解法：coding convention = "SUT 视为 final，但 test 里偷偷 override"。

```bash
mkdir -p /tmp/ch14-lib && cd /tmp/ch14-lib

# === vendor: final class 等价: 不暴露 vtable ===
cat > vendor_final.h <<'EOF'
/* vendor: 类不可继承 — 用 opaque struct + 暴露函数指针不暴露 struct 定义 */
#ifndef VENDOR_FINAL_H
#define VENDOR_FINAL_H
typedef struct VendorFinal VendorFinal;
VendorFinal *vendor_final_new(void);
int          vendor_final_compute(VendorFinal *self, int x);
#endif
EOF

cat > vendor_final.c <<'EOF'
#include "vendor_final.h"
#include <stdlib.h>
struct VendorFinal { int secret; };
VendorFinal *vendor_final_new(void) {
    VendorFinal *v = (VendorFinal *)malloc(sizeof(VendorFinal));
    v->secret = 0;
    return v;
}
int vendor_final_compute(VendorFinal *self, int x) {
    /* vendor 自家逻辑: x * 2 */
    return x * 2 + self->secret;
}
EOF

# === 用规约 + function pointer 表做 "fake override" ===
#     关键约定: production 代码不写 VendorFinal*; 通过接口层间接用.
cat > final_wrapper.h <<'EOF'
#ifndef FINAL_WRAPPER_H
#define FINAL_WRAPPER_H
/* Production 接口: 看不到 VendorFinal (final) */
typedef struct FinalWrapper FinalWrapper;
struct FinalWrapper {
    int (*compute)(FinalWrapper *self, int x);
    void *impl;
};
FinalWrapper *vendor_final_adapter_new(void);
FinalWrapper *fake_final_new(void);    /* test 侧 */
#endif
EOF

cat > final_wrapper.c <<'EOF'
#include "final_wrapper.h"
#include "vendor_final.h"
#include <stdlib.h>

typedef struct { int override; } FakeState;

static int fake_compute(FinalWrapper *self, int x) {
    FakeState *st = (FakeState *)self->impl;
    /* "假装"override: 即使 vendor final, fake_compute 替代了 vendor 逻辑 */
    return x * 100 + st->override;
}
FinalWrapper *fake_final_new(void) {
    FakeState *st = (FakeState *)malloc(sizeof(FakeState));
    st->override = 1;
    FinalWrapper *f = (FinalWrapper *)malloc(sizeof(FinalWrapper));
    f->compute = fake_compute;
    f->impl = st;
    return f;
}

static int vendor_compute(FinalWrapper *self, int x) {
    VendorFinal *vf = (VendorFinal *)self->impl;
    return vendor_final_compute(vf, x);
}
FinalWrapper *vendor_final_adapter_new(void) {
    FinalWrapper *f = (FinalWrapper *)malloc(sizeof(FinalWrapper));
    f->compute = vendor_compute;
    f->impl = vendor_final_new();
    return f;
}
EOF

# === 测试 ===
cat > test_final.c <<'EOF'
#include "final_wrapper.h"
#include <assert.h>
#include <stdio.h>
int main(void) {
    /* 用 fake "override" 掉 vendor final 类 — 因为我们 wrap 在中间层, vendor 看不出 */
    FinalWrapper *f = fake_final_new();
    /* fake: 5 * 100 + 1 = 501; vendor: 5 * 2 = 10 */
    assert(f->compute(f, 5) == 501);
    printf("test_final PASS (fake returned %d, vendor would return 10)\n",
           f->compute(f, 5));
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_final vendor_final.c final_wrapper.c test_final.c
./test_final
echo "rc=$?"
```

**验收**：
- `./test_final` 输出 `test_final PASS (fake returned 501, vendor would return 10)`，`rc=0`
- 关键观察：测试 *不调用 vendor_final_compute*（fake_compute 完全替代）— 这就是 ch14 "restricted override dilemma" 的工程解

> **A1-A4 共用工作目录** `/tmp/ch14-lib`；每个 A 单独 `.c` 文件，依次跑通。

## 六、值得深入思考的问题

### Q1: Wrap 自己的库还是 wrap vendor 的库？

Feathers 暗示是 vendor。但**自家内部 team library 也是 vendor**（跨团队使用）—— wrap 它意味着"团队 B 用了团队 A 的库 = 自动 wrap = 测试套件自动 fake" 的工业级落地。**关键问题**：wrap 是从"vendor 给我们用"开始扩展，还是从"我们的公共库被外部用"开始强制？哪种政策让组织保持 seam 密度？

### Q2: Wrapper 是不是另一种 vendor 锁定？

我们 wrap 了 vendor API 后，业务代码依赖 wrapper 接口。**问**：wrapper 接口本身是不是新的 vendor 锁定？历史上很多公司 wrap 了 Oracle JDBC，然后 Oracle 走了，wrapper 一层代码变成负担。**关键问题**：wrapper 接口的设计应该多"vendor-neutral"（抽象到谁都能实现）？过度抽象 vs 适度抽象的边界在哪？

### Q3: Coding convention 替代 `final` 是不是只是"半步安全"？

如果团队约定"production 不 override"但 compiler 不强制，新人无意中 override 了 production class — vendor 的 design intent 悄悄被破坏。**关键问题**：convention 替代 language feature 是不是给"testability 优先"让路太大？code review 该有什么样的 checklist 防范这种破坏？

### Q4: AI 自动 wrap 之后我们失去了对 wrapper 形状的判断

AI 给的 wrapper 通常 1:1 镜像 vendor API；这恰恰丢失了 ch3 Takeaway 6 "narrow vs wide interface" 的设计意图。**关键问题**：是不是该给 AI 一个"wrapper interface 应该比 vendor 窄"的硬规约？还是有更好的工具引导？

### Q5: Once dilemma 的 wrap 真的隔离了 vendor 单例吗？

我们在 A2 演示了 wrap 让测试不污染 vendor 全局，但 vendor_global_set_level 仍然可以调。**关键问题**：测试时万一有人忘了 fake、直接调 vendor setter，测试还能隔离吗？工业上需不需要"测试期间 vendor global 写保护"的更强机制？

### Q6: 当 vendor 是我们公司自己时 wrap 谁？

很多大公司内部有 cross-team library（"基础设施 SDK"）。**关键问题**：基础设施团队该不该强制下游 wrap？强制 wrap 是不是对"基础设施下沉"的反向抑制？这种 wrap 决策在公司内的治理角色是什么？

### Q7: Skin-and-wrap vs responsibility-based extraction —— 何时用哪个？

这是 ch15 的预告，但 ch14 已经埋伏笔：vendor 是"小 API + 强约束"（final / singleton）时倾向 skin-and-wrap；vendor 是"大 API + 弱约束"（继承层级深）时倾向 responsibility-based extraction。**关键问题**：什么信号决定该走哪条路？这是 ch15 核心议题。

---

*下一章预告*: **Chapter 15 — My Application Is All API Calls and No Logic** — ch14 讲的"vendor 调用必须 wrap"是局部症状；ch15 把它推到极端 —— **整个应用 = 一连串 vendor API 调用**。这是 ch14 once dilemma 的"全代码级别"版本。Mailing list server 例子显示：识别 computational core + Skin-and-Wrap vs Responsibility-Based Extraction 两条路。**ch15 给的不是技法，是 *识别哪些是 API、哪些是自己的逻辑* 的判断力**。
