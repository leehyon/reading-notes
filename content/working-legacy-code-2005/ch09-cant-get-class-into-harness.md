# Chapter 9 — I Can't Get This Class into a Test Harness

> **PDF**: p.127-158（32 页）
> **定位**: 本章是 *症状-技法目录* 的代表章。ch8 TDD 的"第 0 步 — get code under test" 在这里展开为 *7 个真实案例 + 7+ 个 dependency-breaking technique*。每个 case 按 "症状描述 → 技法选择 → 完整 C/Java 代码 → trade-off" 展开。这是 Part II 中 *"症状最长"的一章* — ch9 是 ch25 catalogue 的具体样本。

## 〇、第一性原理思考

**这章做了什么**: 用 7 个真实案例(Irritating Parameter / Hidden Dependency / Construction Blob / Irritating Global Dependency / Horrible Include / Onion Parameter / Aliased Parameter)逐一演示 ch25 catalogue 里 7+ 个 dependency-breaking technique 在 C++ 与 Java 上的具体落法。

**为什么这样拆**: 把"测不了"按 4 大原因分类(对象创建难 / harness build 难 / 构造副作用 / 构造 sensing), 每类挂对应的 ch25 技法; ch9 是 ch25 操作手册的 *带血带肉* 样本。

**最值钱的洞见**: 同一技法在不同语言不可用性不同 — Extract and Override Factory Method 在 C++ 不可用(构造时 virtual 不 dispatch 到 derived), 必须用 Supersede Instance Variable 替代; Pass Null 在 Java/C# 是 0 改动利器, 在 C/C++ 是 UB 源。

Singletons 的"only one instance"和"isolated testable"本质 tension — Feathers 立场通常拆 singleton 性质,1 instance 的安全靠 build-time 检查 + 团队规约实现, 而不是运行时强制。

## 一、章节概述

- **核心论断**：*I can't get this class into a test harness* 是 legacy 改动的最常见阻塞。Feathers 把它拆成 **4 大原因**：(1) 对象创建难；(2) test harness build 难；(3) 构造函数副作用；(4) 构造中要做 sensing。
- **解构武器库**：ch9 给出 *7 个 case*，每个 case 用 *特定技法* 解决。技法来自 ch25 catalogue — 包括 **Pass Null** / **Extract Interface** / **Extract Implementer** / **Subclass and Override Method** / **Parameterize Constructor** / **Extract and Override Factory Method** / **Supersede Instance Variable** / **Introduce Static Setter** / **Parameterize Method**。
- **Case 1 — Irritating Parameter**：`CreditValidator(RGHConnection, CreditMaster, validatorID)`。`RGHConnection` 构造时真连服务器。**解**：Extract Interface on `RGHConnection` + 测试里用 `FakeConnection`。或者 *Pass Null*（Java/C#）如果不需要用那个参数。
- **Case 2 — Hidden Dependency**：`mailing_list_dispatcher` 在构造函数里 `new mail_service`。构造函数 *悄悄* 创建对象。**解**：**Parameterize Constructor** — 把 `mail_service` 从构造参数传入。C++ 里加 `initialize()` 方法 *Preserve Signatures*，旧构造器仍可工作。
- **Case 3 — Construction Blob**：`WatercolorPane` 构造里 new 5+ 个对象，依赖网密集。**解**：**Supersede Instance Variable** — 加 `supersedeCursor(FocusWidget*)` setter，测试构造完再换。或者 **Extract and Override Factory Method**（Java/C#，**C++ 不能用**）。
- **Case 4 — Irritating Global Dependency**：`PermitRepository.getInstance()` 是 singleton，测试间共享状态。**解**：**Introduce Static Setter** — 加 `setTestingInstance(repo)` 让测试 swap。或者 **Subclass and Override Method** 把 behavior 隔离。
- **Case 5 — Horrible Include Dependencies**（C++ 专属）：`Scheduler` 类需要 `#include` 一堆 . h。**解**：*写替代 definition* — 测试文件加 `void SchedulerDisplay::displayEntry(const string&) {}` 让 SchedulerDisplay 的方法 *空*。**缺点**：必须单独 build test 二进制。
- **Case 6 — Onion Parameter**：`SchedulingTask(Scheduler, MeetingResolver)` → `SchedulingTaskPane(SchedulingTask)`。参数链深。**解**：Extract Interface on *immediate* 依赖（`SchedulingTask`）；如果 `SchedulingTask extends SerialTask` 且所有 public 在父类，Java 可以在 `SchedulingTask` 上提一个 interface 包含 `SerialTask` 的方法。
- **Case 7 — Aliased Parameter**：`IndustrialFacility(OriginationPermit permit)`，`Permit basePermit` 字段存 permit 但不能存 `IOriginationPermit`（Java 接口不能继承 class）。**解**：**Subclass and Override Method** — 测试用 `AlwaysValidPermit extends FakeOriginationPermit` 直接 override `validate()` 设 flag。
- **Pass Null 的语言限制**：在 Java/C# 中 *非常实用*（运行时异常捕捉）。在 C/C++ 中 *基本不可用* — 运行时 UB。
- **C++ 的特殊问题**：constructor 中 virtual dispatch 不会 resolve 到 derived class（标准规范）。所以 Extract and Override Factory Method 在 C++ 不可用，必须 Supersede Instance Variable 替代。
- **Singletons 的本质困难**："only one instance" 与 "isol testable" 是 tension。**两条路径**：(a) 拆 singleton 性质 — 公开 constructor + 团队规约；(b) `setTestingInstance` + 测试纪律。**Feathers 立场**：拆 singleton 性质通常合理，1 instance 的"安全"通过 build-time 检查 + 团队规约实现。
- **Ch25 catalogue 引用**：ch9 用到的所有技法都出自 ch25。ch9 是 *应用场景*，ch25 是 *操作手册*。

## 二、核心 Takeaways

### Takeaway 1: 不可测 = 4 大原因分类

- **是什么**：4 大不可测原因 — (1) 对象构造难；(2) test harness build 难；(3) 构造副作用；(4) 构造中 sensing。
- **为什么重要**：分类让你 *针对每种原因* 选最合适的技法。不是"一招打天下"。
- **解决什么问题**：让"为什么测不了" 的提问变成可回答 — 每种原因对应一类技法。
- **适用场景**：评估每个不可测类时，先分类再选 ch25 技法。

### Takeaway 2: Pass Null 是 Java/C# 的 *利器*，C/C++ 的 *陷阱*

- **是什么**：构造时传 `null` 占位，让 *真正* 用到该参数的代码路径立即抛 NPE；测试只跑 *不碰那个参数* 的路径。
- **为什么重要**：在 Java/C# 是最便宜的拆法 — 0 改动。C/C++ 是 UB 源。
- **解决什么问题**：测试只需要 *SUT 不碰那个参数* 的场景。
- **适用场景**：Irritating Parameter 的最轻量解。

### Takeaway 3: Extract Interface vs Extract Implementer 的选择

- **是什么**：Extract Interface 抽新 interface 类；Extract Implementer 把原类变 implementer，保留 inherit 路径。Java 用 `implements`，C++ 用纯虚基类。
- **为什么重要**：Extract Interface 让 *所有 caller* 重写（破坏面大）；Extract Implementer 让 *调用方不感知*。
- **解决什么问题**：C++ 项目优先 Extract Implementer；Java 项目按 caller 数量决定。

### Takeaway 4: Parameterize Constructor = Hidden Dependency 的标准解

- **是什么**：把构造里 `new Foo()` 改成构造参数 `Foo*`。测试传 fake；生产传真实。
- **为什么重要**：相比 *全局 swap*（Introduce Static Setter），Parameterize 让 *每个 instance 独立*。
- **解决什么问题**：Hidden Dependency 中"对象在构造里悄悄创建"。
- **适用场景**：任何"构造里 new X" 的类。C++ 还可以 *preserve 旧构造器签名* 通过 `initialize()` 方法。

### Takeaway 5: Supersede Instance Variable = Construction Blob 的兜底

- **是什么**：加 setter `setField(NewValue)` 替换对象。测试构造完再换。
- **为什么重要**：C++ 项目里 Extract and Override Factory Method *不能* 在 constructor 里用 — virtual dispatch 不工作。Supersede 是 *唯一选择*。
- **解决什么问题**：Construction Blob + C++。
- **适用场景**：C++ 项目里构造里有复杂对象图。

### Takeaway 6: Introduce Static Setter = singleton 的标准拆法

- **是什么**：给 singleton 加 `setTestingInstance(NewValue)`；测试 setUp 时 swap；测试 tearDown 时 reset。
- **为什么重要**：保持 singleton 的"只一个"语义（在生产代码中），但允许 *测试 swap*。
- **解决什么问题**：singleton 的 *全局共享状态* 让 test 间互相污染。
- **适用场景**：Java/C# 的 singleton legacy。

### Takeaway 7: Subclass and Override Method = Aliased Parameter 的标准解

- **是什么**：测试里建一个 `FakeXxx extends Xxx` override *SUT 真正用到的 method*。构造时传 `new FakeXxx()`。
- **为什么重要**：不需要抽 interface — override 具体方法够用。
- **解决什么问题**：Aliased Parameter（参数类型是父类，但需要子类化才能 fake）。
- **适用场景**：验证逻辑藏在 override 方法里时（`validate()` 是经典）。

### Takeaway 8: C++ 的 Horrible Include 是 *无解之解*

- **是什么**：测试文件加 `void SchedulerDisplay::displayEntry(const string&) {}` — 把被依赖类的方法 *空实现*。
- **为什么重要**：C++ 的 transitive include 让 *抽出接口* 也得改很多文件。**替代 definition** 是 *测试文件内* 局部解。
- **解决什么问题**：C++ 大类 + 严重 header 依赖。
- **适用场景**：Feathers 自己说 *"huge class with severe dependency problems"*。**不是日常技术**。

### Takeaway 9: 单 instance "安全" 不依赖 singleton 关键字

- **是什么**：singleton 是 Java/C++ 的 *强约束* — 编译/运行时报错。但旧 C/汇编没这约束仍能写出安全系统。
- **为什么重要**：singleton 的"安全"价值被高估。**真正"安全"靠团队规约** — 加 build-time grep + runtime alarm 比 singleton 关键字更可控。
- **解决什么问题**：让"该不该拆 singleton" 的判断 *基于实际需求*，不是 *基于习惯*。
- **适用场景**：每次写 singleton 前问 *为什么*。

### Takeaway 10: 第 0 步是 *多技法组合*

- **是什么**：把 class 放进 test harness 往往不是 *一个* 技法，而是 *2-3 个技法的组合*。例如：`Extract Interface` + `Parameterize Constructor` + `Pass Null`。
- **为什么重要**：单技法不够时，*技法组合* 才是工程现实。
- **解决什么问题**：避免"找不到完美单法就放弃" 的瘫痪。
- **适用场景**：每个不可测类。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **Hidden Dependency = 内核的 late binding 陷阱**。`vfs_read()` 内部 `new inode_operations` 在调用时 — 用户态 helper 看不到，但 kernel module 测时必须 mock。**Parameterize Constructor 思路**：把 `struct file_operations` 作参数传入（=kernel 的 indirect call）。
- **Construction Blob = `super_block` 的初始化**。`mount_bdev()` 内部 new 6+ 个对象（super、inode、dentry、file、...）。**Supersede 思路**：kernel 的 `set_sb()` / `set_dentry()` 系列 setters；`set_blocksize()` 等 *late binding* helper。
- **Irritating Global = kernel globals**。`current` (current task) / `jiffies` / `printk` log buffer 都是 *隐式全局依赖*。**Introduce Static Setter 思路**：kernel 的 `cmocka` 用 `mock()` 替换 `current`；`kunit` 用 fake time API。
- **Horrible Include = kernel `linux/list.h` 依赖爆炸**。任何 . c 改 `list.h` = 重编 1000+ . o。**解**：用 `container_of` + forward declarations 而不是 include list. h。
- **Subclass and Override Method 在 kernel = `xxx_override`**。`i2c_client` 注册 driver 时用 `i2c_driver_override()` 替换默认 behavior — 等价于 runtime override。

#### Linux — Dependency Breaking 映射

| ch9 case          | Linux 对应                                  | 工程动作                         |
| ----------------- | ----------------------------------------- | ------------------------------ |
| Irritating Parameter | `__init` 函数接收 hardware config 指针        | platform_data 参数化             |
| Hidden Dependency | `module_init` 内部 create device           | late probe + dependency injection |
| Construction Blob | `super_block` 初始化                       | `set_sb()` / 内部 setter 拆分    |
| Irritating Global | `current` / `jiffies` / `printk` 全局     | kunit mock + fake clock         |
| Horrible Include  | `linux/list.h` transitive                  | forward decl + container_of     |
| Onion Parameter   | `struct i2c_algorithm` 链式依赖             | 抽 `i2c_algorithm` interface     |

### 3.2 机器人软件视角

- **Irritating Parameter = ROS DDS transport**。`rclcpp::Node` 构造时需要 `RMWContext`，但 RMW 在测试时不该真启动 DDS。**Extract Interface**：用 `rclcpp::Node::make_shared(fakes)` 替换。
- **Hidden Dependency = `lifecycle::LifecycleNode` 构造**。`on_configure` 内部 connect 到 DDS + 创建 publisher。**Parameterize Constructor**：把 publisher 创建移到 `on_activate`。
- **Construction Blob = `nav2_planner` 构造**。构造里 new costmap、tf listener、BT factory、logger。**Supersede Instance Variable**：每个 component 是 setter 替换。
- **Irritating Global = `tf2_ros::Buffer` 全局**。所有 node 共享同一个 tf buffer。**Introduce Static Setter**：testing 用 `setTestingBuffer(fake_buffer)`。
- **Onion Parameter = `nav2_msgs/BehaviorTree` 参数链**。Goal → NavigateToPose → ComputePath → FollowPath 链路深。**Subclass and Override Method**：override `ComputePath` 节点跳过真 planner。
- **Aliased Parameter = `rosbag2` 消息封装**。`SerializedMessage` 继承 `MessageBase`，但 storage 层需要 interface。**Extract Interface on message base**。

#### 机器人 — Dependency Breaking 映射

| ch9 case          | ROS/ROS2 对应                                  | 工程动作                         |
| ----------------- | --------------------------------------------- | ------------------------------ |
| Irritating Parameter | `rclcpp::Node` 接收 `RMWContext`              | 抽 `RMW` interface + fake        |
| Hidden Dependency | `lifecycle::LifecycleNode` 在构造 connect DDS | 移到 `on_activate`              |
| Construction Blob | `nav2_planner` 构造                          | 各 component 单独 setter         |
| Irritating Global | `tf2_ros::Buffer` 全局                       | testing 用 fake buffer swap     |
| Onion Parameter   | BT 节点链                                      | override 叶子 BT 节点              |
| Aliased Parameter | `rosbag2` 消息封装                             | 抽 `MessageBase` interface       |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                    |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 看到不可测类         | "重写吧"                                                    | "4 大原因分类 + ch25 技法组合"                              |
| 选 Pass Null         | 不区分语言（C/C++ 也用）                                      | Java/C# 用；C/C++ 改用 Extract Interface                    |
| 选 Extract Interface | 默认                                                        | C++ 优先 Extract Implementer；Java 看 caller 数 |
| Hidden Dependency    | "改不动"                                                    | Parameterize Constructor + Preserve Signatures              |
| Construction Blob   | 整树重写                                                    | Supersede Instance Variable（C++）或 Extract and Override Factory Method（Java） |
| singleton 冲突      | "singleton 不能测"                                            | "Introduce Static Setter + 团队规约 + runtime alarm"        |
| Onion Parameter     | 4 层构造都 fake                                               | 只 fake 最 immediate 依赖                                    |

> **关键差异**：高级工程师把每个不可测类 *当作 ch25 技法组合题*；初级把它当 *单技法失败题*。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 写代码 = 让"加依赖"更便宜** — 但 ch9 是 *拆依赖* 目录，AI 帮不上"该不该拆" 的判断。
- **AI 不知道 C++ 的 virtual-in-constructor 限制** — 经常推荐 Extract and Override Factory Method 在 C++ 项目。
- **AI 倾向 Pass Null 默认** — 在 C/C++ 项目这是 UB 源。
- **AI 给 singleton 测试方案时常 *忘了 reset*** — 测试间共享状态，污染后续 test。

### 4.2 AI 已经能做的（具体到 ch9 主题）

- **识别 4 大原因**：从构造函数体 + 字段类型 + static 调用，自动分类。准确率 70-80%。
- **推荐 ch25 技法组合**：基于 4 大原因分类 + 语言，AI 推荐 2-3 个技法组合。准确率 60%。
- **自动生成 Fake 类**：基于 SUT 调用的方法集合，AI 生成 Fake 子类。准确率 80%。
- **检测 Pass Null 在 C/C++ 中的风险**：基于代码上下文，提醒 unsafe。准确率 90%。

### 4.3 AI 不能替代的（具体到 ch9 主题）

- **判断"singleton 该不该拆"**：拆 singleton 是设计决策，AI 给的是 *plausible*，不是 *correct*。
- **判断 *技法组合* 的顺序**：先 Pass Null 还是先 Extract Interface？顺序影响 ROI。
- **判断 Horrible Include 何时该做**：这是 *huge class + severe dependency* 的最后手段，AI 不知道阈值。
- **判断 Onion Parameter 的 *fake 哪一层***：fake leaf node 还是 fake root？影响 test 速度。

### 4.4 AI 经常写错的地方

针对 ch9 7 大 case 主题：

| 错误模式                                          | 例子                                                                                                  | 后果 |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ---- |
| **C++ 推荐 Extract and Override Factory Method** | AI 给 C++ 项目建议 override factory method — 但 C++ constructor 中 virtual 不 dispatch 到 derived   | 测试不生效；SUT 仍调 base class 实现 |
| **Pass Null 在 C/C++ 中用**                       | AI 给 C 项目传 NULL 给 `RGHConnection*` — 但 `connection->connect()` 是 UB                          | 运行时崩溃；测试结果不可信                                  |
| **Introduce Static Setter 但忘 reset**            | AI 给 singleton 加 `setTestingInstance` 但忘了 `resetForTesting()` — 测试间互相污染                  | 测试 5 通过但顺序变 6 失败                                  |
| **Extract Interface 时漏方法**                    | AI 抽 `IRGHConnection` 但漏 `connect()` / `disconnect()` — 测试构造时缺方法报错                  | 测试无法编译                                                |
| **Parameterize Constructor 改了所有 caller**       | AI 改构造签名后忘了 propagate 到 10 个调用点                                                          | 编译失败；diff 爆炸                                          |
| **C++ 推荐 Subclass and Override Method 但忘 destructor**| AI override 一个方法但父类析构不 virtual — 测试用 `delete p` 时 UB                              | 测试运行时 UB；看似通过实际错                                |
| **Onion Parameter 选错层**                        | AI 给多层 chain 只 mock 最 deep 那层，但 SUT 实际用 middle 层                                      | 测试空跑；什么都验不到                                       |
| **Aliased Parameter 推荐 Extract Interface**      | AI 给 `OriginationPermit` 抽 interface，但 `Permit basePermit` 不能存 interface                      | compile 失败；建议背离 case 7 的标准解 |
| **fake 比 SUT 还复杂**                            | AI 给 fake 类 50 个方法，但 SUT 只用 3 个                                                            | 维护成本暴增；fake 反而成为 risk                              |

### 4.5 子段：AI 辅助遗留代码理解 — 在本主题专项

- **AI 帮你"自动检测 4 大原因"**：扫构造函数体 + 字段 + static call。准确率高。
- **AI 帮你"推荐技法组合"**：基于 4 大原因 + 语言，AI 给 2-3 技法。**风险**：不知道 C++ 限制。
- **AI 帮你"生成 Fake 类"**：基于 SUT 的方法调用集合，自动生成 fake。
- **AI 不会替你 *review 拆依赖决策* **：拆 singleton / 拆构造签名涉及团队规约。

### 4.6 工程师必须保留的核心能力

- **判断 4 大原因分类**：每个不可测类第一步。
- **C++ vs Java 的技法差异**：virtual dispatch、forward declaration、include guard。
- **Pass Null 的语言限制**：C/C++ 不用。
- **技法组合的顺序**：先 separation 还是先 sensing？
- **singleton 拆与不拆的设计判断**：业务需求 vs 测试需求。

## 五、实践行动项

> ch9 是 7 case + 多技法的目录章。下面 4 个 demo 各对应 1 个 case，演示核心技法 + 测试。

### A1 — Irritating Parameter: Extract Interface + Fake

```bash
mkdir -p /tmp/ch09-fake && cd /tmp/ch09-fake

# 接口
cat > connection.h <<'EOF'
#ifndef CONNECTION_H
#define CONNECTION_H
typedef struct Connection {
    int  (*connect)(struct Connection *self);
    int  (*disconnect)(struct Connection *self);
    int  (*report_for)(struct Connection *self, int id);
} Connection;
#endif
EOF

# 真实: 模拟硬件连接, 连接耗时
cat > real_conn.c <<'EOF'
#include "connection.h"
#include <stdio.h>
#include <unistd.h>
static int real_connect(Connection *c) {
    (void)c; fprintf(stderr, "[real] connecting...\n");
    /* 真实硬件耗时, 测试不该真跑 */
    return 0;
}
static int real_disconnect(Connection *c) {
    (void)c; fprintf(stderr, "[real] disconnecting\n"); return 0;
}
static int real_report_for(Connection *c, int id) {
    (void)c; return id * 100;  /* 模拟服务器返回 */
}
Connection *real_connection_new(void) {
    static Connection c = { real_connect, real_disconnect, real_report_for };
    return &c;
}
EOF

# Fake: 立刻返回
cat > fake_conn.c <<'EOF'
#include "connection.h"
static int fake_connect(Connection *c)    { (void)c; return 0; }
static int fake_disconnect(Connection *c) { (void)c; return 0; }
static int fake_report_for(Connection *c, int id) {
    (void)c; return 42;  /* 测试时给固定值 */
}
Connection *fake_connection_new(void) {
    static Connection c = { fake_connect, fake_disconnect, fake_report_for };
    return &c;
}
EOF

# SUT: CreditValidator-like class
cat > validator.h <<'EOF'
#ifndef VALIDATOR_H
#define VALIDATOR_H
#include "connection.h"
typedef struct Validator {
    Connection *conn;
    int success, total;
} Validator;
void validator_init(Validator *v, Connection *c);
int  validator_validate(Validator *v, int customer_id);
#endif
EOF

cat > validator.c <<'EOF'
#include "validator.h"
void validator_init(Validator *v, Connection *c) {
    v->conn = c; v->success = 0; v->total = 0;
}
int validator_validate(Validator *v, int customer_id) {
    v->total++;
    int r = v->conn->report_for(v->conn, customer_id);
    if (r > 0) v->success++;
    return r;
}
EOF

cat > test_validator.c <<'EOF'
#include "validator.h"
#include <assert.h>
#include <stdio.h>
int main(void) {
    Connection *fake = fake_connection_new();
    Validator v; validator_init(&v, fake);

    /* fake 总返回 42 > 0 → success */
    validator_validate(&v, 1);
    validator_validate(&v, 2);
    validator_validate(&v, 3);

    assert(v.total == 3);
    assert(v.success == 3);
    fprintf(stderr, "test_validator PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_validator validator.c fake_conn.c test_validator.c
./test_validator
echo "rc=$?"
```

**验收**：`test_validator PASS` + `rc=0`。证明 Extract Interface 让 fake 替换真 connection — 测试 *不连真硬件*。

### A2 — Hidden Dependency: Parameterize Constructor

```bash
mkdir -p /tmp/ch09-param && cd /tmp/ch09-param

# service 接口
cat > service.h <<'EOF'
#ifndef SERVICE_H
#define SERVICE_H
typedef struct Service {
    int (*get_status)(struct Service *s);
    void (*connect)(struct Service *s);
} Service;
#endif
EOF

# RealService: 模拟 hidden dep, 构造时 new
cat > service.c <<'EOF'
#include "service.h"
static int rs_get_status(Service *s) { (void)s; return 1; }
static void rs_connect(Service *s)    { (void)s; }
Service *service_new(void) {
    static Service s = { rs_get_status, rs_connect };
    return &s;
}
EOF

# FakeService: 测试用
cat > fake_service.c <<'EOF'
#include "service.h"
static int fs_get_status(Service *s) { (void)s; return 1; }
static void fs_connect(Service *s)    { (void)s; }
Service *fake_service_new(void) {
    static Service s = { fs_get_status, fs_connect };
    return &s;
}
EOF

# Dispatcher: hidden dep 改为参数 (Parameterize Constructor)
cat > dispatcher.h <<'EOF'
#ifndef DISPATCHER_H
#define DISPATCHER_H
#include "service.h"
typedef struct Dispatcher {
    Service *svc;
    int connected;
} Dispatcher;
void dispatcher_init(Dispatcher *d, Service *s);
void dispatcher_init_default(Dispatcher *d);  /* 生产用 */
#endif
EOF

cat > dispatcher.c <<'EOF'
#include "dispatcher.h"
void dispatcher_init(Dispatcher *d, Service *s) {
    d->svc = s; d->connected = 0;
    /* Hidden Dependency 现在是参数, 不在构造里悄悄 new */
    s->connect(s);
    d->connected = (s->get_status(s) == 1);
}
void dispatcher_init_default(Dispatcher *d) {
    /* Preserve 老构造签名: 默认用 real service */
    dispatcher_init(d, service_new());
}
EOF

cat > test_param.c <<'EOF'
#include "dispatcher.h"
#include <assert.h>
#include <stdio.h>
int main(void) {
    /* 通过 fake 参数注入: 不依赖真 service */
    Dispatcher d;
    dispatcher_init(&d, fake_service_new());
    assert(d.connected == 1);
    fprintf(stderr, "test_param PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_param dispatcher.c fake_service.c test_param.c
./test_param
echo "rc=$?"
```

**验收**：`test_param PASS` + `rc=0`。证明 Parameterize Constructor 让 fake service 注入，老构造签名 (`init_default`) 仍工作。

### A3 — Construction Blob: Supersede Instance Variable

```bash
mkdir -p /tmp/ch09-blob && cd /tmp/ch09-blob

# 模拟 cursor widget
cat > cursor.h <<'EOF'
#ifndef CURSOR_H
#define CURSOR_H
typedef struct Cursor {
    int x, y;
} Cursor;
static inline void cursor_init(Cursor *c, int x, int y) { c->x = x; c->y = y; }
#endif
EOF

# CursorWidget: 老 API
cat > cursor_widget.h <<'EOF'
#ifndef CURSOR_WIDGET_H
#define CURSOR_WIDGET_H
#include "cursor.h"
typedef struct CursorWidget {
    Cursor *inner;  /* 构造里 new 的隐藏对象 */
    int count;
} CursorWidget;
void  cw_init(CursorWidget *w, int x, int y);
int   cw_get_count(const CursorWidget *w);
/* Supersede Instance Variable: setter 替换 */
void  cw_supersede_cursor(CursorWidget *w, Cursor *new_cursor);
#endif
EOF

cat > cursor_widget.c <<'EOF'
#include "cursor_widget.h"
#include <stdlib.h>
void cw_init(CursorWidget *w, int x, int y) {
    w->inner = malloc(sizeof(Cursor));
    cursor_init(w->inner, x, y);
    w->count = 0;
}
int cw_get_count(const CursorWidget *w) { return w->count; }
void cw_supersede_cursor(CursorWidget *w, Cursor *new_cursor) {
    free(w->inner);          /* C++ 注意: 也 delete 旧对象 */
    w->inner = new_cursor;
}
EOF

# 测试 fake cursor
cat > test_blob.c <<'EOF'
#include "cursor_widget.h"
#include <stdio.h>
#include <assert.h>
int main(void) {
    CursorWidget w; cw_init(&w, 0, 0);
    /* Supersede: 替换成测试用的 fake cursor */
    static Cursor fake;
    cursor_init(&fake, 100, 200);
    cw_supersede_cursor(&w, &fake);

    assert(w.inner->x == 100);
    assert(w.inner->y == 200);
    fprintf(stderr, "test_blob PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_blob cursor_widget.c test_blob.c
./test_blob
echo "rc=$?"
```

**验收**：`test_blob PASS` + `rc=0`。证明 Construction Blob 通过 Supersede Instance Variable 替换 inner 对象。

### A4 — Irritating Global Dependency: Introduce Static Setter

```bash
mkdir -p /tmp/ch09-global && cd /tmp/ch09-global

# Singleton-ish repo
cat > repo.h <<'EOF'
#ifndef REPO_H
#define REPO_H
typedef struct Repo {
    int data[8];
    int n;
    int (*find)(struct Repo *r, int key);
} Repo;
void repo_init(Repo *r);
int  repo_find(Repo *r, int key);
void repo_add(Repo *r, int v);
/* Introduce Static Setter: 测试用 global swap */
void repo_set_test_instance(Repo *r);
Repo *repo_get_test_instance(void);
#endif
EOF

cat > repo.c <<'EOF'
#include "repo.h"
static Repo *g_test_repo = NULL;
static int default_find(Repo *r, int key) {
    for (int i = 0; i < r->n; i++)
        if (r->data[i] == key) return 1;
    return 0;
}
void repo_init(Repo *r) {
    r->n = 0; r->find = default_find;
}
int repo_find(Repo *r, int key) { return r->find(r, key); }
void repo_add(Repo *r, int v) { if (r->n < 8) r->data[r->n++] = v; }
void repo_set_test_instance(Repo *r) { g_test_repo = r; }
Repo *repo_get_test_instance(void) { return g_test_repo; }
EOF

# SUT 模拟 PermitRepository.getInstance() 的 caller
cat > caller.h <<'EOF'
#ifndef CALLER_H
#define CALLER_H
#include "repo.h"
typedef struct Caller {
    Repo *ref;  // 测试 set 后, SUT 用这个
    int found;
} Caller;
void caller_init(Caller *c);
int  caller_query(Caller *c, int key);
#endif
EOF

cat > caller.c <<'EOF'
#include "caller.h"
void caller_init(Caller *c) { c->ref = NULL; c->found = 0; }
int caller_query(Caller *c, int key) {
    Repo *r = c->ref ? c->ref : repo_get_test_instance();
    if (!r) return -1;
    return repo_find(r, key);
}
EOF

cat > test_global.c <<'EOF'
#include "repo.h"
#include "caller.h"
#include <assert.h>
#include <stdio.h>
int main(void) {
    /* 每个测试开始 swap */
    Repo test_repo; repo_init(&test_repo);
    repo_add(&test_repo, 42);
    repo_add(&test_repo, 100);
    repo_set_test_instance(&test_repo);

    Caller c; caller_init(&c);
    assert(caller_query(&c, 42)  == 1);
    assert(caller_query(&c, 100) == 1);
    assert(caller_query(&c, 7)   == 0);

    fprintf(stderr, "test_global PASS\n");
    /* 测试结束 reset */
    repo_set_test_instance(NULL);
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_global repo.c caller.c test_global.c
./test_global
echo "rc=$?"
```

**验收**：`test_global PASS` + `rc=0`。证明 Introduce Static Setter 让 singleton 类在测试中可 swap，tearDown 中 reset 防止 test 间污染。

## 六、值得深入思考的问题

### Q1: 7 个 case 之间有没有"统一" 框架？

Feathers 给 7 case 各不同技法。**关键问题**：是否存在 *统一的拆依赖算法*？还是必须 case-by-case？这是 ch25 catalogue 能否成为 *模式语言* 的根本问题。

### Q2: C++ 的"virtual-in-constructor"限制怎么让 Extract and Override Factory Method 完全失效？

C++ 标准规定 constructor 中 virtual call 不 dispatch 到 derived class — 但 Java/C# 没有这限制。**关键问题**：C++ 项目里 Constructor Blob 该用什么替代？Supersede Instance Variable 是唯一选择吗？是否有更优雅的方案？

### Q3: Pass Null 在 production 代码里该不该用？

Ch9 明确说 *test only*。但很多团队把 Pass Null 当 *default* — "测不动就 null 一下"。**关键问题**：如何强制 Pass Null 只在测试路径出现？靠 review？靠 runtime 报警？

### Q4: singleton 的"安全" vs "可测" 怎么权衡？

Feathers 倾向 *团队规约 + runtime alarm* 替代 singleton 关键字。**关键问题**：singleton 的价值被高估了 — 但"团队规约" 的 *执法成本* 高吗？什么场景下 singleton 真不可拆？

### Q5: Aliased Parameter 的 Subclass and Override Method 在 C++ 里有什么陷阱？

C++ override concrete method 时，*析构* 不 virtual 会导致 delete 父类指针 = UB。**关键问题**：C++ 的 Subclass and Override Method 怎么 *安全*？是不是要先保证 *父类有 virtual destructor*？

### Q6: Horrible Include 是 *兜底* — 但何时不该用它？

Ch9 自承 *"reserve for huge class with severe dependency problems"*。**关键问题**：huge class 该用 Horrible Include，还是该先花时间做 Extract Class？什么时候 Horrible Include 是 *回避* 而不是 *解构*？

### Q7: AI 推荐技法组合时如何防止 *语言错误*？

AI 不知道 C++ 限制。**关键问题**：是否应该把 C++/Java/Python 的 *可用技法矩阵* 注入 AI 的 system prompt？这是 *工具配置* vs *教育* 的权衡。

### Q8: 第 0 步的 *ROI 怎么量化*？

每个不可测类都要 ch9 拆依赖 — 时间成本可能 1-10 天。**关键问题**：第 0 步的 *价值怎么算*？对后续 feature 加的速度 / bug 修复时间 / 测试覆盖的提升？

---

*下一章预告*: **Chapter 10 — I Can't Run This Method in a Test Harness** —— ch9 是 *类级别* 的拆依赖。ch10 深入到 *方法级别* —— 当 *整个类* 能进 test harness 但 *某方法* 不能跑（依赖 hard-to-construct 参数 / 副作用 / 私有状态）时怎么办。ch10 教 *Parameterize Method* / *Extract Method* / *依赖隐藏字段* / *Constructor 注入* 等更精细的技法。ch9 + ch10 一起 = TDD 第 0 步的完整武器库。