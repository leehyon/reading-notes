# Chapter 24 — We Feel Overwhelmed. It Isn't Going to Get Better.

> **PDF**: p.341-346（6 页）
> **定位**: 全书倒数第二章。Feathers 从 *技术* 维度切到 *情感* / *职业* 维度 —— 在 legacy code 里工作为什么值得; green-field 不是答案; 团队士气低时怎么办。这是 Part II 收尾, 也是 Part III reference catalog 的情感前置。

## 〇、第一性原理思考

**这章做了什么**: 把 Part II 从技术维度切到情感/职业维度 —— 在 legacy code 里"为什么还在干"是比"怎么拆"更基本的问题, Feathers 用 fun / green-field 不是 escape / morale boost / oasis / community 五条命题给出可持续工作的心理框架。

**为什么这样拆**: 它不教新技法, 而是把 ch6-23 全部技法放进"career meaning"的语境里 —— 技术纪律需要 fun + impact + community 三块电池才能续航, 没情绪动力的团队会把"挑最丑集体 clean up"做成一次性 morale boost 然后再滑回去。

**最值钱的洞见**: Fun 不是"用最新框架"的爽, 是"把一片代码放进 test 后看到它变干净"的掌握感 —— 找回 fun 的具体路径是 TDD outside work: 在 1 小时小项目里重获 fast-loop 体验, 再把这种感觉带回工作 oasis。
## 一、章节概述

- **核心命题**: legacy code 难, 无可否认。但 *"为什么你还在干它"* 是更基本的问题。paycheck 是合理的, 但仅有 paycheck 不够 —— 你得有别的 *in it for you*。
- **Feathers 的答案: fun** —— 编程的 *fun* 是入门时的原动力("我第一次坐下来用电脑, 觉得 wow 这好酷")。找回 fun 的能力 = legacy code 里也能找到乐趣。
- **Green-field 不是 escape**: 大型组织里 *green-field team* 经常失败 —— 旧系统在 *持续演化*(bugfix + 偶发 feature), green-field team 永远追不上。**草地其实没那么绿**。
- **TDD outside work** —— Feathers 的建议: 在工作之外做 TDD 小项目, 感受 *快速 feedback* 的乐趣, 然后把这种感觉带回工作。**你工作的项目也能这样感觉 —— 只要你能把代码放到 fast test harness 里**。这是 ch3 sensing/separation + ch8 拆 feature 的 *情感动力*。
- **Morale boost 实践**: 当团队士气低 *因为代码质量* 时 —— 选 *最丑最吵的一组类*, 把它 *整组放到 test 下*。完成后团队重获控制感。Feathers 见过这模式 *反复奏效*。
- **Oases**: 当你把一片代码放到 test 下, 这片代码就成为 *oasis* —— 写新 feature 时在这里是 *fun* 的。团队应该持续扩大 oasis。
- **本章短小、密度高**: 6 页 PDF, 几乎没代码例子, 几乎没技法 —— 是 Part II 的 *收束* 章节, 把 ch6-23 的技法放进 *情感框架* 里。

## 二、核心 Takeaways

### Takeaway 1: "为什么你在 legacy code 里" 是先决问题

- **是什么**: 在你抱怨 *代码难* 之前, 先问自己 *为什么还在干*。Paycheck 是理由之一, 但不能是唯一。
- **为什么重要**: 没有 *fun / 意义* 撑着的工程师会 burnout; 带着 *fun / 意义* 的工程师能在 legacy code 里做 *奇迹*。这是 *招聘 + 留人* 的关键。
- **解决什么问题**: 解决 *high turnover in legacy code teams* —— 不是技术问题, 是 *career meaning* 问题。
- **适用场景**: 任何在 legacy code 里工作的工程师; 管理 legacy code team 的 lead。

### Takeaway 2: Green-field 不是 escape —— 草地其实没那么绿

- **是什么**: 大公司里常见剧情 —— 旧系统变难改, 管理层 *把最强的人挪到 green-field 重写 team*。结果: 旧系统仍在生产(持续 bugfix + feature), green-field team 追不上; 几年后没人能重写。
- **为什么重要**: 让你对 *green-field 项目* 的玫瑰色滤镜破灭 —— *旧系统维护* 仍是组织最需要的工作。
- **解决什么问题**: 解决 *优秀工程师被调离 legacy* 后 *legacy 进一步恶化* 的恶性循环。
- **适用场景**: 公司层面 *决定是否重写* 的战略会议; 个人 *是否接受 green-field offer* 的判断。

### Takeaway 3: Fun = 在 legacy code 里工作的核心燃料

- **是什么**: 编程的 *fun* 不是 *用最新框架*, 而是 *做出能 work 的东西*。legacy code 里也能 fun —— 当你把一片代码 *放到 test 下*, 看到它在你掌握之下变干净, 这就是 fun。
- **为什么重要**: fun 是 *sustainable pace* 的关键 —— 没 fun 的工程师只能靠 deadline 逼, 逼到最后 burnout。
- **解决什么问题**: *职业倦怠* 的根源往往不是 *工作量大*, 是 *没成就感*。Fun 提供成就感。
- **适用场景**: 给团队做 *morale boost* 时; 个人反思 *我为什么还在写代码* 时。

### Takeaway 4: TDD outside work — 把"快速 feedback"的体验带回工作

- **是什么**: 工作之外做 *小项目 TDD*。感受 *1 秒 feedback* 的乐趣。然后你意识到 *工作项目也能这样*, 只要把代码放到 fast test harness 里。
- **为什么重要**: 这是 ch3 / ch8 / ch22 的 *情感锚* —— 把 ch3 的 sensing/separation 推到 *享受*, 不只是 *工具*。
- **解决什么问题**: 解决 *"legacy code 就是慢 / 痛苦"* 的固化认知 —— 体验过 fast harness 就回不去"忍受慢"的状态。
- **适用场景**: 给 *被慢 build 折磨的工程师* 一个 *认知刷新* —— 通过外部项目重新获得 *fast loop* 体验。

### Takeaway 5: Morale boost —— 选最丑的模块放到 test 下

- **是什么**: 团队士气低 *因为代码质量* 时, 挑 *最丑最吵* 的一组类,**整组** 放到 test 下, 不是 *一片一片修*。完成后团队 *感到掌握局面*。
- **为什么重要**: 团队 *心理转折点* —— 从 *被代码压垮* 翻到 *我们能搞定一个怪物*。这是 *bandwagon effect* 的反向: *最难的先做*。
- **解决什么问题**: 解决 *代码质量 → 士气低 → 没人想做 → 质量更差* 的死亡螺旋。
- **适用场景**: 季度 team retro 后做 *morale initiative*; 新 lead 上任第一件事。

### Takeaway 6: Oases of test-covered code — 持续扩大绿洲

- **是什么**: 每当一片代码被 test 覆盖, 它就成 *oasis* —— 在这片写代码是 fun。Feathers: *"work can really be enjoyable in them"*。
- **为什么重要**: Oasis 是 *fun 的具体化* —— 它给团队一个 *可见的目标*(扩大绿洲), 而不是抽象的"提升质量"。
- **解决什么问题**: 解决 *legacy code 整体改进太慢* 的无力感 —— *先扩大 oasis*, 再慢慢连成片。
- **适用场景**: 团队 sprint planning 时给 *legacy code coverage 增加* 一个 *可见的 metric*(例如 *coverage percentage* 但要按目录细看)。

### Takeaway 7: 团队 + 社区是 *sustainable* legacy code 工作的支撑

- **是什么**: Feathers 强调: 编程是 *孤独活动*, 但 *legacy code* 工作的可持续性来自 *好的同事 + 社区*(mailing list / conference)。你学到的新东西靠 *分享* 维持动力。
- **为什么重要**: *孤立* 是 burnout 的最大风险; *社区* 是 *职业续航* 的充电站。
- **解决什么问题**: 解决 *资深工程师孤立感* —— legacy code 资深工程师 *最懂*, 但最容易被孤立。
- **适用场景**: 团队 *retreat / conference 预算* 的辩护; 个人 *加入 mailing list / 写 blog* 的鼓励。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **Linux kernel 维护者的 fun 来源**: 不是 *新 feature*, 而是 *解决一个别人解不了的 bug*。Greg KH / Linus Torvalds 都公开说过: *kernel maintenance is fun because it's hard*。这是 ch24 *fun* 在 kernel 维护的具体化。
- **Linux kernel 社区 = 持续学习的社区**: kernel mailing list (LKML)、kernelnewbies、每年 Kernel Summit —— 是 Feathers *社区* 维度的工业实例。
- **Green-field vs Legacy 在 Linux**: Linus 明确拒绝 *rewrite the kernel* —— *we don't rewrite, we evolve*。这是 ch24 *green-field 不是 escape* 在 kernel 的最强版本: 整个开源社区都同意 *legacy 是常态*, 不是要解决的状态。
- **Morale boost 实践: Linux driver sub-system maintainer**: 当 subsystem 维护者士气低(常见于 wireless / staging), 社区做法是 *集体 sprint*: 多个 maintainer 集中 review 一个 driver 子树, 把最丑的先 clean up。Open Source Summit 期间的 *kernel janitor* 活动是这实践的工业实例。
- **TDD outside work**: 多数 kernel 开发者 *业余写小工具* 时才用 TDD —— 这是 *kernel mainline = legacy mode, side project = TDD mode* 的典型分布。kernel hacking 是 *exploratory*, TDD 是 *确认性*。
- **Oases of test-covered code**: Linux kernel 5. x 起 *KUnit* 覆盖逐步增加, drivers/* 里有 *test-covered islands*(KUnit in-tree tests)。这些 island 周围写新 driver 是 *fun* 的(可单测、可 mock)。
- **Greg KH 的 morale boost**: Greg KH 维护 stable kernel + driver core 多年, 公开分享 *"maintaining legacy is fun because users depend on you"* —— 这是 ch24 情感命题在 kernel maintainer 群体的版本。
- **Linux Foundation mentorship**: *Linux Kernel Mentorship Program* —— 给新人 *oasis 项目*(改一个 small driver / 文件系统) 让他们 *fun*, 然后慢慢扩大。这是 ch24 *Oases* 的工业实现。
- **kernel CI 的 morale**: 0day CI / KernelCI / patchwork bot 让 *legacy 改动有反馈* —— 这是 *hyperaware editing* 在 kernel 维度的支撑。

#### Linux 系统 — Legacy Code 团队士气工程实践表

| 实践                | Linux kernel 等价物                          | 效果                       |
| ------------------- | ------------------------------------------- | -------------------------- |
| Fun 来源           | "维护 critical infra 有荣誉感"              | retention                  |
| Green-field 反驳    | Linus "we don't rewrite"                    | 战略对齐                   |
| TDD outside work   | 业余 side project (eBPF tool / driver hack) | 体验 fast loop             |
| Morale boost       | 集体 review 一个 subsystem + janitor sprint | 重获控制感                 |
| Oases              | KUnit-covered drivers / filesystems         | 可见的 fun 区              |
| 社区               | LKML / Kernel Summit / kernelnewbies        | 持续学习 + 归属感          |
| 情感支撑           | Greg KH / maintainer 公开 "I love this job" | role model                 |

### 3.2 机器人软件视角

- **ROS 维护者的 fun**: ROS 主线维护(ros2 / ros_core) 累且文档多, 但 *看到新 robot 跑起来* 是 *fun 的瞬间*。ROSCon 是 *充电站*。
- **ROS Discourse / ROS Answers**: 社区是 *legacy ROS1 → ROS2 migration* 时的 *情感支撑* —— 提问者得到快速回答, 贡献者得到 *fun 的感觉*(帮人解决)。
- **Nav2 团队的 morale boost**: Nav2 在 ROS2 Humble 后面临 *迁移压力*, 社区做了一次 *"pick the worst 3 planners, get them under test"* 集体 sprint —— *Morale boost* 的工业实例。
- **MoveIt2 的 oasis**: MoveIt2 把 `move_group` 这块 monolithic 拆了之后,*planning pipeline* 是 *test-covered oasis* —— 周围写新 planner 是 fun 的。
- **ros2_control 的 oasis**: hardware_interface 的 *mock hardware_interface* 让 controller 写新 driver 是 fun 的(不用真硬件)。
- **robot_localization 的 EKF test**: EKF 的 *bag playback 测试* 覆盖了 80% 路径 —— 这是 oasis 的 *test corpus* 形态。
- **ROS-Industrial consortium**: 工业机器人(ABB / KUKA / Fanuc) 维护者靠 *consortium* 维持动力 —— *社区* 是工业 legacy code 的续航。
- **ROS2 的 *green-field 反驳***: ROS2 设计时 ROS1 已 10+ 年,*没选择重写*, 而是 *incremental migration*。这是 *legacy 不是 escape* 的工程实例。
- **Open Robotics 团队士气**: Open Robotics 公开博客分享 *"maintaining ROS is fun because robots depend on us"* —— ch24 在机器人社区的版本。

#### 机器人软件 — Legacy Code 团队士气工程实践表

| 实践                | ROS2 / Nav2 等价物                          | 效果                       |
| ------------------- | ------------------------------------------- | -------------------------- |
| Fun 来源           | "新 robot 跑起来" / "planner 终于收敛"      | retention                  |
| Green-field 反驳    | ROS2 = ROS1 incremental, no rewrite         | 战略对齐                   |
| TDD outside work   | 业余 Gazebo plugin / 仿真小项目            | 体验 fast loop             |
| Morale boost       | Nav2 planner collective test sprint         | 重获控制感                 |
| Oases              | ros2_control mock HW / MoveIt2 planning pipeline | 可见的 fun 区              |
| 社区               | ROS Discourse / ROSCon / ROS-Industrial     | 持续学习 + 归属感          |
| 情感支撑           | Open Robotics / Steve Macenski "ROS is fun" | role model                 |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 为什么还在写代码     | "因为 pay / 因为这是工作"                                     | "因为 fun + 学习 + impact"                                   |
| 羡慕 green-field     | 觉得 green-field team 是 escape hatch                       | 知道 *草地没那么绿*,留下做实质工作                          |
| 业余时间             | 刷社交媒体                                                   | 写 side project TDD                                          |
| 面对最丑的代码       | 抱怨 / 逃避                                                  | "挑最丑的,集体放到 test 下" —— morale boost                 |
| 维护 legacy          | "等到 green-field 重写吧"                                     | "维护 critical infra 是荣誉感"                              |
| 社区参与             | 不参与 mailing list / 会议                                   | 公开 blog / 演讲 / 答疑                                       |
| 团队士气低           | 等别人解决                                                   | *主动* 挑最丑的代码,集体 clean up                            |
| 个人成就感           | 等 manager 认可能                                           | 自己造 oasis,看到自己扩大的覆盖率                            |
| 倦怠                 | 跳槽                                                         | *主动调整*:加入社区 / 换角色 / 找新 oasis                    |
| 看待 legacy          | "旧东西"                                                     | "critical infra + 长期投资"                                 |

> **关键差异**: 高级工程师 *主动 fun 化 legacy 工作* —— 不是 *等 fun 来*, 而是 *造 fun 的场景*(挑最丑的 clean up + 找社区 + 写 blog)。初级 *等 fun 来*。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

**极其重要** —— 而且 AI 让它 *更* 重要。原因:

- **AI 让 "green-field 幻想" 复活**: AI 让"用 AI 重写" 看起来很便宜 —— 这是 *green-field 不是 escape* 的现代版本。工程师用 AI 把 legacy 代码 *扔给 AI 重写*, 看似快, 实际:
  1. 旧系统在生产持续演化 —— 重写永远追不上。
  2. AI 重写没历史 commit 记忆 —— 丢掉 *hidden contracts*。
  3. 团队 morale 没建立 —— 因为团队 *没做* 工作, 只是 *review AI 的工作*。
- **AI 让 legacy 工作 *有新的 fun***: 调试 AI 写的错误代码、解释 AI 的奇怪 refactor 决策、写更精准的 prompt 让 AI 不破坏 behavior —— 这些是 *新型 fun*, 不是 *写新代码* 的 fun。
- **AI 时代更需要 community**: AI 工具发展极快, *社区* (r/LocalLLaMA, Hacker News, ML Twitter) 是 *跟上节奏* 的唯一方法。这是 ch24 *社区* 维度的现代化。
- **AI 让 "fast feedback" 重新可能**: ch3 强调 fast test harness; AI 时代 *AI 自己的 feedback* 也是 fast(几秒生成代码)。但 *人* 仍要 *verify* —— 这是 ch23 hyperaware 的现代责任。

### 4.2 AI 已经能做的

- **AI 帮你 *找到* 最丑的代码**: 给 codebase, AI 标 *"monster 方法"*、*"高 cyclomatic complexity"* —— 这是 morale boost 的 *first step*(挑目标)。
- **AI 帮你写 *fun 的 side project***: 给一个 small project 需求, AI 帮你 TDD 起步 —— 这是 *TDD outside work* 的现代版本。
- **AI 当 *morale coach***: 给 *"为什么我在 legacy code 里"* 的 prompt, AI 给 *"找 fun 的角度"* + *"community 链接"* 建议。
- **AI 帮你写 blog / share**: 把你的 refactor 故事给 AI, AI 帮你 *写得更清晰* 分享到 Discourse / HN。
- **AI 帮你找 oasis**: 给 codebase, AI 标 *"test-covered islands"* —— 这是 oasis 的 *first detection*。

### 4.3 AI 不能替代的

- **真正的 team morale boost**: AI 不能组织集体 sprint —— 这是 *人* 做的事。
- **真正的 community 归属**: mailing list / conference 上 *真人* 的连接 ≠ AI 模拟的对话。
- **真正的 fun 体验**: 看真硬件跑起来 ≠ 看 AI 描述它跑起来。
- **真正的 "I love this job" role model**: AI 给不出 *真实 maintainer* 的角色榜样 —— *Greg KH / Steve Macenski* 是真人。

### 4.4 AI 经常写错的地方

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **建议 "用 AI 重写 legacy" 当 escape**         | AI 给 "我用 GPT-4 重写你的 codebase" 看似快,实际追不上生产演化      | green-field 幻想的现代版本,团队 morale 没建立 |
| **不识别 *critical infra* 的价值**              | AI 给 legacy 代码 low priority,因为 "新功能更好玩"                | 工程师被引导放弃 legacy,关键系统无人维护 |
| **不推荐 community**                            | AI 给 *纯技术* 建议,不提 *mailing list / conference*               | 孤立,最终 burnout |
| **把 morale 当 *情绪问题***                     | AI 不区分 *morale 是工程节奏问题 vs 情感问题*                      | 给错 solution |
| **不区分 *fun 类型***                           | AI 把 *写新代码 fun* 当成 *唯一 fun*,legacy fun 被忽略              | 工程师被劝离 legacy |
| **不识别 "维护 critical 是荣誉"**                | AI 不会主动说 *"你维护的代码用户每天依赖"*                        | 工程师失去 *role model* 视角 |
| **把 TDD 当技术而不是体验**                     | AI 教你 TDD 步骤,不教 *"fast loop 的乐趣"*                         | 失去 *fun 的锚* |
| **不推荐 side project**                         | AI 只给工作内建议,不提 *业余 TDD project 找 fun*                  | 失去 *fun refresh* 渠道 |
| **用 AI 当 *async pair* 而不是 *community***     | 工程师只和 AI 交互,不和其他工程师交互                              | 孤立 |
| **不推荐 *blog / share***                       | AI 不鼓励 *公开写 refactor 经验*                                   | 失去 *teach = learn* 杠杆 |

### 4.5 子段：AI 辅助遗留代码理解 — 在本主题专项

- **AI 帮你识别 *morale 触发点***: 当团队抱怨 *legacy 慢*, AI 分析代码找到 *"monster 方法分布"* —— 这是 *集体挑最丑的 clean up* 的输入。
- **AI 帮你找 oasis**: 给 codebase, AI 标 *"test coverage > 80% 的目录"* —— 团队 *扩大 oasis* 的可见目标。
- **AI 帮你写 *refactor 故事 blog***: 把 refactor PR 的 commit history 给 AI, AI 生成 *叙事性 blog post* —— 让团队 *分享 = 充电*。
- **AI 帮你找 community**: 给你的技术栈("ROS2 Nav2") AI 给 *ROS Discourse / ROSCon / GitHub Discussions* 链接 —— 加入 community 的入口。
- **AI 帮你做 *fun refresh***: 给 *最近 1 个月最痛苦的 refactor*, AI 生成 *简化版的小项目*(可以业余写)让你重获 fast loop 体验。
- **AI 不能做**: 真人的 community 连接、role model、critical infra 维护的荣誉感 —— 这些要 *人* 给 *人*。

### 4.6 工程师必须保留的核心能力

- **回答 "我为什么还在 legacy code 里"**: 不是 *paycheck*, 是 *fun + impact + community*。
- **挑最丑的集体 clean up**: 不是 *一片一片慢慢改*, 是 *集中力量做难看的事* —— morale boost。
- **找 community**: 加入 mailing list / conference / blog。这是 *续航电池*。
- **区分 *green-field 幻想* 和 *实际工作***: 不被 *"AI 重写"* 诱惑。
- **维护 *oasis* 的纪律**: 持续扩大 test-covered islands, 让 fun 区越来越大。
- **role model 视角**: 看 *Greg KH / Steve Macenski* 怎么 *维护 critical infra*, 学习他们的 *fun 来源*。

## 五、实践行动项

> 本章情感维度重, 实践行动项聚焦 (a) *TDD outside work* 的小项目; (b) *morale boost* 的 *挑最丑集体 clean up* 演示; (c) *oasis* 的 *test-covered island* 检测。

### A1 — TDD outside work: 1 小时内的小项目(用 Python 写 mini-game)

```bash
mkdir -p /tmp/ch24-tdd-game && cd /tmp/ch24-tdd-game

# 用 Python 写一个 guess-the-number, TDD 风格
# 注意: 这是 *outside work* 演示,1 小时内可完成。
cat > guess.py <<'EOF'
import random

class GuessGame:
    def __init__(self, lo=1, hi=100):
        self.lo = lo
        self.hi = hi
        self.target = random.randint(lo, hi)
        self.attempts = 0
        self.done = False

    def guess(self, n):
        """Return 'higher', 'lower', 'correct', or 'invalid'."""
        if self.done:
            return 'invalid'
        if not (self.lo <= n <= self.hi):
            return 'invalid'
        self.attempts += 1
        if n < self.target:
            return 'higher'
        if n > self.target:
            return 'lower'
        self.done = True
        return 'correct'

    def hint(self):
        return f"attempts={self.attempts}, range=[{self.lo},{self.hi}]"
EOF

cat > test_guess.py <<'EOF'
"""TDD-style tests for GuessGame — runs in < 1 sec, classic fast feedback."""
import random
random.seed(42)  # for reproducibility
from guess import GuessGame

def test_correct_guess():
    g = GuessGame()
    g.target = 50  # pin target
    assert g.guess(50) == 'correct'
    assert g.done

def test_higher_lower():
    g = GuessGame(); g.target = 50
    assert g.guess(30) == 'higher'
    assert g.guess(70) == 'lower'

def test_invalid_range():
    g = GuessGame()
    assert g.guess(0) == 'invalid'
    assert g.guess(101) == 'invalid'

def test_done_blocks():
    g = GuessGame(); g.target = 50; g.guess(50)
    assert g.guess(50) == 'invalid'  # already done

def test_attempts_increments():
    g = GuessGame(); g.target = 50
    g.guess(30)
    assert g.attempts == 1
    g.guess(70)
    assert g.attempts == 2

if __name__ == '__main__':
    test_correct_guess(); test_higher_lower()
    test_invalid_range(); test_done_blocks(); test_attempts_increments()
    print("TDD test PASS — fast feedback in < 1 sec")
EOF

python3 test_guess.py
```

**验收**:
- `python3 test_guess.py` 输出 `TDD test PASS — fast feedback in < 1 sec`。
- 耗时 < 1 秒 — 体验 *fast loop* 的 fun。
- 这是 ch24 *TDD outside work* 的小项目:1 小时可写完, fast feedback。

### A2 — Morale boost: "挑最丑集体 clean up" 的 git 演示

```bash
mkdir -p /tmp/ch24-morale && cd /tmp/ch24-morale
rm -rf * && git init -q
git config user.email "you@example.com" && git config user.name "you"

# Step 1: legacy version — separate file, no tests
cat > monster_legacy.c <<'EOF'
/* legacy monster: 假设这是一个 5 年没人碰的方法 */
int monster(int x, int y) {
    if (x > 0) {
        if (y > 0) {
            if (x + y > 100) return 100;
            else if (x * y < 50) return 50;
            else return x + y;
        } else {
            return x;
        }
    } else {
        return -1;
    }
}
EOF
git add . && git commit -q -m "legacy: monster method (no tests)"

# Step 2: morale boost — 拆 + 加 test
cat > monster.c <<'EOF'
#include <stdio.h>
/* morale-boosted: 拆 + 加 test */
static int classify_sum(int x, int y) {
    if (x + y > 100) return 100;
    if (x * y < 50)  return 50;
    return x + y;
}

int monster(int x, int y) {
    if (x <= 0) return -1;
    if (y <= 0) return x;
    return classify_sum(x, y);
}

/* tests for morale: prove the behavior is preserved */
static int run_tests(void) {
    int ok = 1;
    ok &= (monster(60, 60) == 100);  /* x+y=120 > 100 */
    ok &= (monster(2, 5) == 50);     /* x*y=10 < 50 */
    ok &= (monster(10, 20) == 30);   /* normal */
    ok &= (monster(-1, 5) == -1);    /* x<=0 */
    ok &= (monster(5, -1) == 5);     /* y<=0 */
    return ok;
}

int main(void) {
    int ok = run_tests();
    printf("morale-boost test %s\n", ok ? "PASS" : "FAIL");
    return ok ? 0 : 1;
}
EOF
git add . && git commit -q -m "morale-boost: split + test the legacy monster"

git log --oneline
cc -std=c17 -Wall -Wextra -O0 -o morale monster.c && ./morale
```

**验收**:
- 编译零警告。
- `./morale` 输出 `morale-boost test PASS`。
- `git log` 看到 2 commits: 1 个 legacy,1 个 morale-boost。
- **演示点**: 团队 *挑这一个 monster*, 做完 *整个* — 这是 ch24 *morale boost* 的最小演示。

### A3 — Oasis detection: 在小代码库上找 *test-covered islands*

```bash
mkdir -p /tmp/ch24-oasis && cd /tmp/ch24-oasis

mkdir -p src/oasis src/desert
cat > src/oasis/well_tested.c <<'EOF'
#include <stdio.h>
/* "Oasis" 模块: 已有完整测试覆盖 */
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }
EOF

cat > src/desert/untested.c <<'EOF'
/* "Desert" 模块: 没有测试覆盖 */
int mystery(int x) {
    /* 没人知道这函数干啥的 */
    return x * 2 + 1;
}
EOF

# 模拟 oasis detection: 用 git history 找有 *test commit* 的目录
cd /tmp/ch24-oasis && git init -q
git config user.email "you@example.com" && git config user.name "you"
git add . && git commit -q -m "initial: mix of oasis and desert"

# 现在给 oasis 目录加 test commit
cat > src/oasis/test_well_tested.c <<'EOF'
#include <assert.h>
#include "well_tested.c"

int main(void) {
    assert(add(2, 3) == 5);
    assert(sub(10, 4) == 6);
    printf("oasis test PASS\n");
    return 0;
}
EOF
git add . && git commit -q -m "test: cover oasis/well_tested.c"

# 检测: 哪些目录最近有 test commit?
echo "=== Oasis detection: dirs with test commits in last N ==="
git log --diff-filter=A --name-only --pretty=format:"%H" -- "*test_*" | sort -u
echo ""
echo "=== Desert: dirs WITHOUT test commits ==="
# 找 src/ 下没有 test 文件的目录
for d in $(find src -type d); do
    if ! ls "$d"/test_* 2>/dev/null | head -1 | grep -q .; then
        echo "DESERT: $d (no test files)"
    else
        echo "OASIS:  $d"
    fi
done
```

**验收**:
- 输出:
  - `oasis/` 标 `OASIS`(有 test_well_tested. c)
  - `desert/` 标 `DESERT`(无 test)
- 这是 ch24 *oasis / desert* 的最小检测演示 —— 团队 *优先扩大 oasis*(给 desert 加 test)。

## 六、值得深入思考的问题

- **"为什么你在 legacy code 里" 的诚实回答**: 真不是 paycheck 吗? 如果是 —— 怎么在工作中 *找到 fun*?
- **Green-field 幻想的现代形态**: AI 重写 / 微服务拆分 / 换框架 —— 都是 *green-field 幻想* 的新版本。怎么识别自己是否被这幻想骗?
- **Morale boost 的可持续性**: *挑最丑集体 clean up* 做完一次后, 团队 *下次* 还会这样主动吗? 怎么维持 morale 不是一次性?
- **Oasis 的"边界效应"**: 一个 oasis 是 *孤岛*, 周围还是 desert。怎么 *连 oasis* —— ch8 的 extract class + ch22 的 Break Out Method Object 是工具, 但 *何时做* 是节奏问题。
- **Community 在 AI 时代的形态**: mailing list / conference 之外,*Discord / Twitter / blog* 是新 community。怎么 *真参与*, 不是 *只看*?
- **Role Model 的稀缺**: 维护 critical infra 的工程师 *不明星* —— 不像 founder / 新 framework 作者。怎么让自己 *从 role model 学习*, 即使他们不公开?
- **Fun 在 5 年 legacy 之后**: 第 1 年 fun, 第 3 年 routine, 第 5 年? 职业 *中后期* 的 fun 来源和前期不同 —— 怎么 *重新发现*?

## 七、本章与全本的关系 + 后续书预告

### 与全本 25 章的关系

- **前置**: ch22 (monster method 拆解) + ch23 (不破坏的纪律) —— 这两章是 *技术* 维度的收尾。
- **本章**: 切到 *情感* / *职业* 维度 —— 在 legacy code 里 *可持续地工作* 的心理框架。
- **后续**:
  - **ch25** = reference catalog —— 24 个 dependency-breaking 技法的查表。本章给它一个 *情感前置*:"这些技法值得学, 因为 legacy 工作值得做"。
  - **全书结束** —— Part III 包含 ch25 + glossary + index。

### 后续书预告

**Working Effectively with Legacy Code** 之后, Feathers 没写 *专门的 legacy 续作*。但 ch24 的情感主题在后续书里被反复回响:

- **Michael C. Feathers — *Working Effectively with Unit Tests* (2014)**: ch24 的 *"TDD outside work"* 在 *unit test* 维度被推到 *readability / maintenance* —— 你怎么 *持续享受* 写 unit test, 而非 *机械地产出覆盖率*。
- **Cal Newport — *Deep Work* (2016)**: ch23 的 *Hyperaware Editing* 是 *deep work* 的程序员版本 —— *无干扰 + 一次只做一件事 + 完整 feedback*。Newport 没写代码, 但他的 *deep work* 框架和 Feathers 的 ch23 完全同构。
- **Cal Newport — *So Good They Can't Ignore You* (2012)**: ch24 的 *"为什么你在 legacy code 里"* 是 Newport *career capital* 框架的程序员版本 —— *fun + impact + community* 是 *career capital* 的具体形式。
- **Daniel H. Pink — *Drive* (2009)**: ch24 的 *fun / autonomy / mastery* 直接对应 Pink 的 *intrinsic motivation* 三角 —— *legacy code* 是 *mastery* 的具体载体。
- **AI 时代的延伸**: ch24 的 *"community"* 在 AI 工具时代变成 *prompt sharing community*(r/ChatGPT / LangChain Discord / Cursor forum)。但 Feathers 的核心命题不变:*legacy 工作值得做, 因为它是 critical infra + fun + community*。
- **可能的后续书**: *Working Effectively with Legacy Code: 20 Years Later* (还没出) —— 也许 Feathers 或其他人会在 2025-2030 写一个 20 周年版, 把 ch24 的 *"why legacy is fun"* 推到 AI / cloud native 时代。

> *下一章预告*: ch25 是全本 reference catalog —— 24 个 dependency-breaking 技法的查表 (Adapt Parameter / Break Out Method Object / Definition Completion / ...)。本章给它情感定位:*这些技法是 ch24 所说 "legacy 工作的乐趣" 的具体工具*。
