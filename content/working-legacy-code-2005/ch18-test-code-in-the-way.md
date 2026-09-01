# Chapter 18 — My Test Code Is in the Way

> **PDF**: p.249-252（4 页）
> **定位**: Part II 工具纪律章 — 把 ch17 的"项目无结构"反衬到"test 文件夹怎么摆"。4 页短章, 看似朴素, 实际承载两个隐藏命题: (a) **命名规约**让 source/test 在视觉上是平级而不是隔离墙; (b) **test code 是否进部署** = 一个 deployment size 的工程选择, 不是审美选择。本章是 ch19 (non-OO test) 的工具前置。

## 〇、第一性原理思考

**这章做了什么**: 用 `*Test` / `Fake*` / `Testing*` 三套 prefix 把 test 与 prod class 在 IDE 字母序里视觉平级对齐, 再用 deployment size (server-side / commercial) 决定 test code 该不该随 prod 一起打进 artifact。

**为什么这样拆**: Feathers 把"测试代码该不该隔离"从审美辩论拉到 deployment size 这个客观变量, 因为长期 navigation tax 才是 team 偷偷放弃写测试的真正原因 — 目录来回切等于每次 alt-tab。

**最值钱的洞见**: Test code 是否进 deployment artifact 实际上不是洁癖问题, 而是"server-side 我们控制的运行时 vs commercial 别人机器上的字节数"这个工程边界 — 决策错位的典型就是"为洁癖分离"半年后 navigation 痛到 team 不写测试。

## 一、章节概述

- **Test code 阻碍 production 的"工程意义"** — 不只是文件数膨胀, 而是当 test 与 prod 混居时, navigation cost 失控。Feathers 的核心建议是 *ergonomics-driven* 命名规约, 而非一刀切的隔离。
- **Class naming convention 是第一道闸**: 被测类 `DBEngine` ↔ `DBEngineTest`(或 `TestDBEngine`)。Feathers 个人偏好 **Test 后缀**, 原因是 IDE 的字母序把同对 class 排在一起。
- **三种测试类的命名空间分布**:(1) `*Test` — 被测单元的 mirror test;(2) `Fake*` — collaborator 替身, 通常是别处 package 的子类;(3) `Testing*` — Subclass-and-Override-Method 技法 (ch25 p.401) 产生的 testing subclass。三者在同一目录里靠前缀字母序自然分组。
- **Test location 决策的真正变量是 deployment size**: server-side / 我们控制的目标 = 同一目录部署; commercial / 别人机器 = 必须剔出。决策错位的典型是"为了洁癖而把 prod/test 隔离", 后来 navigation 疼痛, 大家不写测试。
- **Java package 跨物理目录是反例**: 同 `com.orderprocessing.dailyorders` package 可同时位于 `source/...` 和 `test/...` 两棵树下。IDE 通常把它们视为同一视图, 所以"location ≠ package"。
- **目录树的 navigation tax 是一种真实的工程成本**: 来回切目录等于每次 alt-tab; 一旦"找 test 文件 5 秒"成为日常摩擦, team 会偷偷放弃写 test。
- **Feathers 给的 fallback 方案**: 同目录 + build script 剥出 test class 用于 deployment。前提是 *好命名规约* 已立。
- **本章没有给出"通用最佳"** — 它显式说"I'm not dogmatic about this arrangement"。关键判断指标: **ergonomics** (它是不是让你少花时间找文件?)。这个判断本身是工程师的能力。
- **与 ch19 的衔接**: 本章给出"同一目录、IDE 排"前提, ch19 会讲 non-OO 项目里这些"测试夹克"(`Fake*` / `Testing*`)在 C 这种没有 package 的语言里怎么落地。ch20 / ch21 进一步推论: 命名规约是 class 拆分后 navigation 的前提。

## 二、核心 Takeaways

### Takeaway 1: Class Naming Convention = 视觉版 source tree

- **是什么**: 用 prefix / suffix 让 test class 与 production class 在 IDE 字母序视图里平级相邻。`DBEngine` ↔ `DBEngineTest`。
- **为什么重要**: 在没有 IDE filter 的时代, grep + alphabetical sort 是第一生产力。今天 IDE 的"Camel case filter"内化了部分, 但跨 IDE / 跨 editor (vim / IDE / `rg`) 时 *扁平前缀字母序* 仍是统一体验。
- **解决什么问题**: 让"看 source 同时看 test"不需要打开第二视图。
- **适用场景**: 任何规模 > 5 个 class 的项目; 团队成员需要 nightly review / pair 时。

### Takeaway 2: Test 后缀 > Test 前缀

- **是什么**:`FooTest` 而不是 `TestFoo`。把 `Test` 当 suffix。
- **为什么重要**: 字母序里 `FooTest` 与 `Foo` 的距离 = 4 字符;`TestFoo` 与 `Foo` 的距离 = 整个字母序。这是 Feathers 的 ergonomic 论证。
- **解决什么问题**: 让"打开 IDE 输入 Foo"看到 `Foo`,`FooTest`,`FakeFoo` (如有) 连排。
- **适用场景**: IDE 用字母序;`rg` + 文件名匹配; code review 时一对一对看。

### Takeaway 3: 三个前缀构成"测试基础设施"的命名空间

- **是什么**:`*Test`(mirror)、`Fake*`(collaborator 替身)、`Testing*`(Subclass and Override Method 专用)。三者在同一目录下靠 prefix 自然聚类。
- **为什么重要**: 生产代码的 prefix 规则通常另有约定, 如业务领域前缀 (订单 `Ord*`、支付 `Pay*`)。test 类的 prefix 与生产代码 prefix *分离* = 在 source tree 里视觉独立。
- **解决什么问题**:`Fake*` 通常是 *别处类的子类*; 字母序把它们排到 fake 区, 远离业务前缀区, 降低"把 fake 当生产类"的误用风险。
- **适用场景**: 任何带 mock / spy / fake 的项目; mock framework 生成的 helper class。

### Takeaway 4: Test Location = 由 deployment size 决定, 不是审美

- **是什么**: server-side / 我们控制的目标 → 同目录同 deployment; commercial / 别人机器 → 必须把 test code 剔出。Feathers 反对"为洁癖分离"。
- **为什么重要**: 当 team 因为"看不下去"分离 test,6 个月后 navigation 痛 → 大家不写 test → 测试债累积。这是负反馈。
- **解决什么问题**: 把"test 该不该同目录"的辩论锚定到 *deployment 时它真的占字节吗* 这个客观变量。
- **适用场景**: CI pipeline 决策 / 部署 artifact 决策 / 客户授权产品的 release engineering。

### Takeaway 5: Java package ≠ 物理目录 — 是反例

- **是什么**: 同一 package 的 class 可跨 `source/...` 和 `test/...` 两棵物理子树; IDE 通常把它们当同视图。
- **为什么重要**: 这告诉你"location 永远是 secondary",`package` 才是 primary 维度。C / Python 没有 package 等价物, 所以本章论点 *部分失效* — 见 ch19。
- **解决什么问题**: 防止"因为目录不同所以需要 import"这种逻辑错。
- **适用场景**: 评估跨语言项目的 test 布局; 决定一个 mixed-language 项目 (Java back + C front) 的 test location 怎么混。

### Takeaway 6: Build Script 剔出 vs 目录分离 — fallback 选择

- **是什么**: 如果一定要剔出 test (deployment size 紧), 推荐用 build script (Makefile / CMake / Gradle) *在产物里* 删 test class,**不**靠 *源码目录分离*。前提是命名规约已立。
- **为什么重要**: 源码分离 → 失去同 IDE 视图 → navigation cost; build script 分离 → 源码还在, artifact 不在 → navigation 保留 + 部署小。
- **解决什么问题**: 既满足 deployment 小, 又保留 IDE ergonomics。
- **适用场景**: embedded firmware 部署; mobile app store 发布; 资源紧张的 edge runtime。

### Takeaway 7: Ergonomics 是判断指标, 不是"我喜欢哪种"

- **是什么**: Feathers 反复说 "ergonomics is important" — 真正的衡量问题是 *它让你找文件花多久?*。
- **为什么重要**: 如果命名规约让查找时间增加 2 秒/次 × 50 次/天 = 100 秒/天/人 = 团队一年 8 工时。这种 cost 不可见但真实。
- **解决什么问题**: 把"规约好不好"的辩论从美学拉到可量化。
- **适用场景**: 每个新项目第一周 review 命名规约; 季度回顾里 track "我花了多久找文件"。

## 三、工程实践视角

> 锁定领域:**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **deployment size 紧 = 嵌入式 firmware**。Linux 内核本身把 test 编为 `*-test.ko`, 这些 *不是* production kernel image 的一部分。`make modules_install` 默认剔 test module。这是 ch18 规则在 kernel 里的实例:**源码相邻、产物分离**。
- **内核目录布局按 *subsystem* 而不是 test/prod 隔离**。`drivers/net/wireless/...` 里 production 和 selftest (KUnit) 共存。这种 layout 等价于 ch18 推荐的"ergonomic"风格, 因为 network wireless 维护者 review patch 时 *生产代码和 test 在同一 patch 里*。
- **Kselftest 是 `Testing/` 风格**。Linux 自带 `tools/testing/selftests/` 目录, 所有 kernel self-test 集中在此 — 这是 ch18 Takeaway 3 的 *`Testing*` 命名空间* 的工业版本。
- **`obj-y` vs `obj-$(CONFIG_FOO_TEST)`** 的 Makefile 模式: 同一源码目录里, test 代码默认 *不* 编入 vmlinux / uImage;`CONFIG_FOO_TEST=y` 时才编。这是 ch18 Takeaway 6 (build script 分离) 的真实例子。
- **KUnit 与 kselftest 的差别**体现 ch18 Takeaway 4: KUnit = build into kernel, 适合 always-on 回归 (内存 / 调度); kselftest = userspace harness, 适合功能验证。前者 deployment size 紧, 后者松。
- **systemd 仓库 layout 走另一条路**:`src/` + `src/test/` 两棵子树, test 用 meson 单独 `meson test` 跑。这是 *目录分离* 派, 因为 systemd 是 daemon (binary 紧、不需要 customer 端 deployment)。Linux daemon 团队常选这条路。

#### Linux 系统 — Test Location 决策表

| 项目类型        | production 同目录? | 决策依据                      | 业界代表                       |
| --------------- | :----------------: | ----------------------------- | ------------------------------ |
| Kernel module   | ✅ 源同,产物分     | CONFIG_FOO_TEST gating         | `drivers/net/wireless/`        |
| Kernel self-test| ❌ 独立目录         | userspace harness             | `tools/testing/selftests/`     |
| System daemon      | ❌ 独立目录         | deployment 是 binary only     | systemd / dbus                 |
| Embedded daemon | ✅ 源同,产物分     | flash size 紧张               | busybox, buildroot             |
| Cross-compile   | ✅ 源同,产物分     | cross toolchain 编译长        | Yocto / OpenWrt                |

### 3.2 机器人软件视角

- **ROS2 包的 layout 是 ch18 Takeaway 5 的强例**。`ros2 pkg create my_pkg` 生成 `my_pkg/`(src)+`my_pkg/test/`(pytest)+ `my_pkg/launch/`(xml)。ament_cmake 把 test 编为 *与 production 同 lib, 但 default 不跑*。这是 ch18 Takeaway 6 的 ROS2 实例。
- **`colcon test` 是 build-script 剔出**。ROS2 默认 `colcon build` 编 test source, 但 *不* 跑;`colcon test` 触发隔离环境跑。**navigation cost 不变**(IDE 看到 test/),**deployment 不带 test**(setup. py exclude)。这是 ch18 推荐方案的行业标准实现。
- **ROS2 的 `ament_lint`** 系列 = ch18 Takeaway 7 (ergonomics) 的工具化。`ament_copyright`, `ament_pep257`, `ament_flake8` 等等。一键 lint 让"目录规约好不好"变成 *运行命令 1 步* 验证, 不再靠 review 讨论。
- **ros2_control hardware_interface 的 mock** = ch18 Takeaway 3 的 `Fake*` 范例。`mock_components/` 目录专门放 `Fake*Motor` / `Fake*Sensor`, 与 `hardware_interface/` 分开; 但同 package 同 deployment。
- **Nav2 behavior tree 测试 = ch18 Takeaway 3 的 `Testing*` 范例**。Nav2 写 `Behavior` 的 testing subclass 用于 MockClock 测试, 文件名前缀 `testing_*`。字母序把 `testing_*` 集中, 与 production `behavior_*` 分离。
- **micro-ROS / embedded robot** = ch18 Takeaway 4 强例。micro-ROS 客户端 lib 通常部署在 MCU (256KB Flash), test code 必须 *不* 编进 firmware;`MICRO_ROS_TESTS=0` 是 build script 默认。这是 *deployment size 决定 layout* 的极端形态。

#### 机器人软件 — Test Code Layout 模式

| 模式          | 生产代码 | 测试代码   | 构建工具剔除        | 适用                  |
| ------------- | -------- | ---------- | ------------------- | --------------------- |
| ament_cmake   | `src/`   | `test/`    | ament 自动           | ROS2 标准             |
| colcon pkg    | 同 pkg   | 同 pkg     | `BUILD_TESTING`     | ROS2 workspace        |
| micro-ROS     | `src/`   | `test/`    | `-DBUILD_TESTING=OFF` | MCU 部署             |
| ros2_control  | `hw_if/` | `mock_hw/` | ament 同             | hardware mock         |
| ROS1 (legacy) | `src/`   | `test/`    | catkin 自动          | 旧 ROS 项目          |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                | 高级工程师                                                  |
| -------------------- | --------------------------------------------------------- | ----------------------------------------------------------- |
| 新成员 onboarding    | "test 文件在哪?" → 等一周才搞清                          | 第一天就能 `grep -rn 'FooTest'` 定位;命名规约让他零猜     |
| 命名规约 vs IDE       | 觉得 IDE filter 万能,命名无所谓                          | 命名规约先立, IDE filter 是 fallback                       |
| Test 目录决策 | "我喜欢 src/test 分开" (洁癖驱动) | "我们的 deployment artifact 是什么? size 紧吗?" 决定布局 |
| 看到 `Fake*`        | "这是 production 类的变体吧"                              | "这是 test infra,字母序把它和 production 分开"             |
| 看到 `Testing*`     | "这是测试专用 subclass? 不污染 production"                | "这是 ch25 Subclass-and-Override-Method 的产物,有意为之"  |
| 决策原则            | 选目录布局凭直觉                                          | 由 *deployment size* 决定 + *navigation cost* 验证         |

> **关键差异**: 高级工程师把 test layout 当 *工程决策*(deployment size + ergonomics 量化), 初级把它当 *审美决定*(我喜欢 / 不喜欢)。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然重要。原因:
- **AI 自动生成 test class**(LLM-Codegen 现在写 unit test 一气呵成)。**AI 不强制命名规约** — 它可能写 `TestFoo`,`Foo_Test`,`FooTests` 三种风格混居一个项目。**没有规约的工程 → 不可读**。
- **AI 写的 test helper class**(Fake / Spy / Stub)经常 *in-place 命名* — `MockDB`, `FakeDBEngine`, `StubDB`, `MockDBEngine2`。这种命名是 *没有规约的命名*, 长期导致 test infra 不可维护。
- **AI 不知道 deployment size** — 它默认 source tree = deployment tree, 从不问"flash size 是多少"。嵌入式 / edge runtime 项目里这是 *致命假设*。
- **AI 重命名破坏 IDE ergonomics** — AI 大改 source tree 时常把 `*Test` 全部重命名为 `*Tests` (复数), 字母序变, IDE 视图变。这是 ch18 Takeaway 2 的退化。

### 4.2 AI 已经能做的(具体到 ch18 主题)

- **批量 audit 现有 test class 命名** — 给定 source tree, 统计命名模式, 产出 "Test prefix / Test suffix / mixed" 报告; 精度高。
- **建议命名规约** — 基于语言习惯(Java 偏 `Test` suffix; Python 偏 `test_` prefix; C 偏 `test_` 或 `_test` suffix); AI 给定项目推荐规约, 准确率 ~85%。
- **自动 git mv** — 把 `TestFoo` 重命名为 `FooTest` 批量 + 自动更新 import。**风险**: AI 不更新 *所有引用*, 需要 review。
- **检测 Fake*/Testing* 的混淆** — 给定代码, AI 标出 `Fake*` 是不是真在 test 文件里被 import, 而 production 文件 import 是错的。

### 4.3 AI 不能替代的(具体到 ch18 主题)

- **判断 deployment size 是不是"紧"** — 这是 release engineering 决策, 涉及客户合同、license 法务、硬件 spec。AI 看不到合同。
- **评估 navigation cost 是否真实存在** — AI 看不到 IDE 用户实际体验; 它只能给"理论上 IDE filter 应该很快"。
- **权衡洁癖 vs 实用性** — 当 team 有人坚持 "test 必须分目录" 时, 这是政治决策, 不是工程决策, AI 不能 *mediation*。
- **决定 build script 剔出的复杂度** — 比如 Yocto 的 `IMAGE_INSTALL_remove` 配置 vs `CONFIG_FOO_TEST=n` — 哪个 deployment pipeline 支持, AI 不知道 build pipeline 内部状态。

### 4.4 AI 经常写错的地方

针对 ch18 Test Layout/Naming 主题:

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **命名混用 prefix/suffix**                      | AI 同时写 `TestFoo`,`FooTest`,`FooTests` 三种                       | IDE 字母序乱, review 视觉一致性丢 |
| **把 `Fake*` 写成 production helper**           | AI 让生产代码 `import { FakeDBEngine }`,因为它"看起来像 helper"      | test infra 污染 production;deployment 体积膨胀 |
| **把 `Testing*` 当普通 subclass**              | AI 改 ch25 Subclass-and-Override 时直接命名为 `MyClass`             | 失去"testing subclass"语义,其他 reader 困惑 |
| **目录分离与命名分离混用**                      | AI 拆 `test/` 目录时仍写 `TestFoo` 前缀                              | 双重隔离,失去字母序 ergonomics |
| **不知道 deployment size**                     | AI 给嵌入式项目建议 "全部 source 同目录,代码更整洁"                 | firmware 体积膨胀,客户验收 fail |
| **git mv 不全更新引用**                        | AI 重命名 test class 时漏改 reflection / dynamic import              | CI 失败,排查 1 天 |
| **`Fake*` 不是子类却用了 subclass 名**         | AI 把 `FakeDBEngine` 写成 `extends DBEngine` 但 method 签名不覆盖     | 编译失败 |
| **build script 不剔除 test class**             | AI 写 CMake 但 `BUILD_TESTING=ON` 是默认                              | firmware 部署后体积翻倍 |
| **`*Test` 名称撞业务名**                       | 业务已有 `User`,AI 写 `UserTest` 也好;若业务有 `UserTest`(集成测试),AI 写新的 `UserTest` 撞名 | 重命名冲突 |
| **不区分 mirror test vs characterization test** | AI 把 ch2 的 characterization test 也叫 `FooTest`                   | 失去"characterization = 探索性的"语义,review 时误以为是 unit test |

### 4.5 子段: AI 辅助遗留代码理解 — 在本主题专项

- **AI audit 一棵 source tree 找命名违反**:`rg "class.*Test.*extends|class.*Test.*implements"` 然后报告 "38% 用 prefix,62% 用 suffix",给规约。**风险**:AI 误判 — Java 社区混 prefix/suffix 是常见,AI 可能过度建议统一。
- **AI 推荐 "把 test 移到独立 subtree" 时强提示 deployment size** — 这是 ch18 Takeaway 4 的 AI 化。**风险**: AI 默认建议"分目录更整洁", 忽略 deployment cost。
- **AI 写 `Fake*` 类时不会自动加 docstring** — 这是 *test infra 的文档债*; legacy project 里 Fake 类无注释, 后人接手时根本不知道它是 mock 还是 production 替代。
- **AI 不建议 "build script 剔出" 模式** — 它默认 source = deployment tree, 从不提 build system 一刀。嵌入式团队要 *主动 prompt* 让 AI 考虑。
- **AI 不区分 fake subclass 语义** — `FakeDB` 是 *方法覆盖* 假替身, 还是 *独立类* 假替身, AI 给的 stub 一样。review 时必须人工。

### 4.6 工程师必须保留的核心能力

- **决定 test location 由 deployment size 驱动**, 不是审美。
- **审计 AI 生成的命名**, 统一 prefix/suffix 风格。
- **保证 `Fake*` / `Testing*` 只被 test 文件 import**, production 文件不引用。
- **写 build script 剔出 test class**(`BUILD_TESTING=OFF`, `CONFIG_FOO_TEST=n`)以满足 deployment size 约束。
- **判断 "AI 给的命名规约是不是过严"** — 比如 Python 社区已经有 `test_` prefix 传统, AI 强行改成 `Test` suffix 反倒错。
- **按 *navigation cost* 验证规约** — 用 `time rg "Foo"` 测查找时间; 规约改进后, 这一时间应下降。

## 五、实践行动项

> 本章是 ch19 (non-OO test) 的工具前置。行动项聚焦在 (a) 用 C / Python 复现 ch18 的命名规约;(b) 演示 deployment size 决策点;(c) build script 剔出 test 的最小工程。

### A1 — C 项目里的 test layout 命名空间演示

```bash
mkdir -p /tmp/ch18-test-layout/src/{module_a,module_b,test_helpers} && cd /tmp/ch18-test-layout

# 演示: production 模块 + 测试模块共存,前缀字母序天然分组
cat > src/module_a/db.h <<'EOF'
#ifndef DB_H
#define DB_H
#include <stddef.h>
typedef struct { int id; const char *name; } db_record_t;
int db_find(const char *id, db_record_t **out);
#endif
EOF
cat > src/module_a/db.c <<'EOF'
#include "db.h"
#include <stdlib.h>
#include <string.h>
static db_record_t fake_table[3] = {
    {1, "Alice"}, {2, "Bob"}, {3, "Carol"}
};
int db_find(const char *id, db_record_t **out) {
    int target = atoi(id);
    for (int i = 0; i < 3; i++) {
        if (fake_table[i].id == target) {
            *out = &fake_table[i];
            return 1;
        }
    }
    *out = NULL;
    return 0;
}
EOF

# Test class (mirror test): DBTest
cat > src/module_a/DBTest.c <<'EOF'
#include "db.h"
#include <assert.h>
#include <string.h>
int main(void) {
    db_record_t *rec = NULL;
    assert(db_find("2", &rec) == 1);
    assert(rec && strcmp(rec->name, "Bob") == 0);
    assert(db_find("99", &rec) == 0);
    return 0;
}
EOF

# Fake collaborator: FakeLogger
cat > src/test_helpers/FakeLogger.h <<'EOF'
#ifndef FAKE_LOGGER_H
#define FAKE_LOGGER_H
#include <stddef.h>
typedef struct { int warn_count; int error_count; } FakeLogger;
void FakeLogger_init(FakeLogger *l);
void FakeLogger_warn(FakeLogger *l, const char *msg);
void FakeLogger_error(FakeLogger *l, const char *msg);
#endif
EOF
cat > src/test_helpers/FakeLogger.c <<'EOF'
#include "FakeLogger.h"
#include <stdio.h>
void FakeLogger_init(FakeLogger *l) { l->warn_count = l->error_count = 0; }
void FakeLogger_warn(FakeLogger *l, const char *msg) {
    l->warn_count++;
    fprintf(stderr, "[FAKE WARN] %s\n", msg);
}
void FakeLogger_error(FakeLogger *l, const char *msg) {
    l->error_count++;
    fprintf(stderr, "[FAKE ERROR] %s\n", msg);
}
EOF

# Testing subclass — 与 db.c 二选一 (production 用 db.c, testing 用 TestingScanner.c)
# 不能两个都编 (linker 会报 multiple definition); 由 Makefile / build script 选
cat > src/module_a/TestingScanner.c <<'EOF'
#include "db.h"
#include <stdlib.h>
#include <string.h>
/* Testing build 时, 用 in-memory fake table 替换真 db_find */
typedef struct { int id; const char *name; } _fake_rec_t;
static _fake_rec_t _fake[2] = {{2, "Bob"}, {1, "Alice"}};
int db_find(const char *id, db_record_t **out) {
    int t = atoi(id);
    static db_record_t r[2];
    /* 持久 buffer: static r[] 引用,不能 stack-local */
    static char name_bufs[2][64];
    for (int i = 0; i < 2; i++) {
        if (_fake[i].id == t) {
            strncpy(name_bufs[i], _fake[i].name, sizeof(name_bufs[i])-1);
            name_bufs[i][sizeof(name_bufs[i])-1] = 0;
            r[i].id = _fake[i].id;
            r[i].name = name_bufs[i];
            *out = &r[i];
            return 1;
        }
    }
    *out = NULL;
    return 0;
}
EOF

# Build & verify — production build 只链 db.c; testing build 只链 TestingScanner.c
echo "=== 字母序查看 (生产 / 测试 / Fake / Testing 自然聚类) ==="
ls -1 src/module_a/ src/test_helpers/

echo
echo "=== production build (链 db.c, 不链 TestingScanner.c) ==="
cc -std=c17 -Wall -Wextra -Isrc/module_a -Isrc/test_helpers \
   -o /tmp/test_db src/module_a/db.c src/module_a/DBTest.c
/tmp/test_db && echo "DBTest PASS"

echo
echo "=== testing build (链 TestingScanner.c, 不链 db.c) ==="
cc -std=c17 -Wall -Wextra -Isrc/module_a -Isrc/test_helpers \
   -o /tmp/test_scanner_test src/module_a/TestingScanner.c src/module_a/DBTest.c
/tmp/test_scanner_test && echo "TestingScanner+DBTest PASS"
```

**验收**:
- 目录字母序:`DBTest.c` (mirror), `TestingScanner.c` (subclass), `FakeLogger.{c,h}` (test infra) 自然分组
- production build 只链 `db.c`; testing build 只链 `TestingScanner.c`(ch18 Takeaway 6 的 build script 分离 — 二选一避免 `multiple definition`)
- 两个 binary 都跑出 "PASS"

### A2 — Python 工程: 用 `pytest` + conftest 演示 test location 决策

```bash
mkdir -p /tmp/ch18-py/{src,test} && cd /tmp/ch18-py

cat > src/calculator.py <<'EOF'
class Calculator:
    def add(self, a, b): return a + b
    def sub(self, a, b): return a - b
EOF

# 命名规约:Python 习惯 test_*.py, 但 *Test* class 后缀
cat > test/calculator_test.py <<'EOF'
import sys, os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'src'))
from calculator import Calculator

class CalculatorTest:
    def setup_method(self): self.calc = Calculator()
    def test_add(self): assert self.calc.add(2, 3) == 5
    def test_sub(self): assert self.calc.sub(10, 4) == 6

class FakeCalculatorTest:    # Fake* 命名规约示范
    def setup_method(self):
        self.fake = Calculator()  # 实际 fake / spy 替身
    def test_stub_add(self): assert self.fake.add(0, 1) == 1
EOF

# pytest discovery 检查字母序分组
echo "=== test files alphabetical order ==="
ls -1 test/

echo
echo "=== pytest run ==="
PYTHONPATH=src python3 -m pytest test/ -v 2>&1 | head -25
```

**验收**:
- `test_calculator.py` 一文件包含 `CalculatorTest`(mirror)+ `FakeCalculatorTest`(fake 命名规约示范), 字母序清晰
- pytest 输出 PASS

### A3 — Build Script 剔出 test class: 用 Makefile + `make test` gating

```bash
mkdir -p /tmp/ch18-makefile/{src,test} && cd /tmp/ch18-makefile

cat > src/driver.h <<'EOF'
#ifndef DRIVER_H
#define DRIVER_H
int driver_run(const char *input);
#endif
EOF
cat > src/driver.c <<'EOF'
#include "driver.h"
int driver_run(const char *input) {
    return input[0] == 'A' ? 1 : 0;
}
EOF
cat > test/DriverTest.c <<'EOF'
#include "driver.h"
#include <assert.h>
int main(void) {
    assert(driver_run("A") == 1);
    assert(driver_run("B") == 0);
    return 0;
}
EOF

# Makefile: BUILD_TESTING 控制是否编 test (默认 OFF,模拟 ch18 Takeaway 6)

```make
CC      ?= cc
CFLAGS  ?= -std=c17 -Wall -Wextra
BUILD_TESTING ?= 0

driver. o: src/driver. c src/driver. h
	$(CC) $(CFLAGS) -c -o $@ src/driver. c

# production artifact: 仅 driver. o (deployment size 安全)
libdriver. a: driver. o
	ar rcs $@ driver. o

# test artifact: 仅在 BUILD_TESTING=1 时编
ifeq ($(BUILD_TESTING),1)
DriverTest: test/DriverTest. c libdriver. a
	$(CC) $(CFLAGS) -Isrc -o $@ test/DriverTest. c libdriver. a
endif

. PHONY: all test clean
all: libdriver. a
test:
ifeq ($(BUILD_TESTING),1)
	./DriverTest
else
	@echo "BUILD_TESTING=0: test not built (deployment size safe)"
	@echo "run 'make BUILD_TESTING=1 test' to enable"
endif
clean:
	rm -f *. o *. a DriverTest
```

echo "=== 默认 build (BUILD_TESTING=0): production-only ==="
make all 2>&1 | tail -5
ls -la libdriver.a DriverTest 2>/dev/null
test -f libdriver.a && ! test -f DriverTest && \
    echo "✅ libdriver.a built, DriverTest NOT built (deployment safe)"

echo
echo "=== make test (BUILD_TESTING=0): test 不存在, message 而已 ==="
make test 2>&1 | tail -3

echo
echo "=== make clean + rebuild with BUILD_TESTING=1 ==="
make clean
make BUILD_TESTING=1 DriverTest 2>&1 | tail -5
test -x DriverTest && ./DriverTest && echo "✅ DriverTest PASS"

echo
echo "=== 部署 size 校验: production-only build 的 artifact ==="
make clean && make all
ls -la libdriver.a
file libdriver.a
```

**验收**:
- 默认 `make all` 只产 `libdriver.a`(deployment size 安全; 无 `DriverTest` binary)
- `make BUILD_TESTING=1 DriverTest` 显式开启才编 test,`./DriverTest` 跑出 PASS
- 这就是 ch18 Takeaway 6 (build script 剔出) 的工业 Makefile 实例, 与 Linux kernel `CONFIG_FOO_TEST`、ROS2 `BUILD_TESTING`、Yocto `IMAGE_INSTALL_remove` 同思想

## 六、值得深入思考的问题

### Q1: 命名规约 vs IDE filter — 哪个更重要?

Feathers 倾向命名规约(因为跨 IDE / 跨编辑器)。现代 IDE 的 CamelCase filter 已经把字母序的需求削弱。**关键问题**: 当 team 统一 IDE (如只准用 CLion / VSCode) 时, 命名规约是否还必要? 或者 IDE filter 足够?

### Q2: `Fake*` 与 `Mock*` 命名冲突

业界有些团队用 `Mock*`(Mockito 风格), 有些用 `Fake*`(Feathers 风格)。**关键问题**: 团队规约应该用哪个? 两个 prefix 在字母序里都聚集, 但语义意图不同 — Mock = 主动 verify; Fake = 被动 record。混用会让 reader 困惑。

### Q3: "Test code 不进 deployment" 的边界

嵌入式 firmware / mobile app / SaaS / system daemon 的"deployment"边界各不同。**关键问题**: 怎么定义 "deployment size 紧"? 是用 artifact 字节数、客户合同、还是 flash spec? AI 不能给合同答案。

### Q4: `Testing*` subclass 的隐藏风险

`Testing*` 用 Subclass-and-Override-Method 技法在 production 类中做 override — 它生产运行时是不是真的剔出?**关键问题**: subclass 的 vtable 是否在 production binary 里? 编译器通常不会自动剔 *未被引用* 的 method (除非 `--gc-sections` + 引用图分析)。这导致 `Testing*` 实际 *污染* production binary, 违背 ch18 Takeaway 6 的初衷。

### Q5: AI 写 test helper 时命名自由度过大

LLM 倾向生成 *自创命名*(因为它没受过团队规约训练), 导致同一项目 50 个 `Fake*` 类命名规则不统一。**关键问题**: 团队应该用 *命名规约 spec 文件*(如 `CONTRIBUTING.md` 里写明规则)+ AI prompt engineering, 还是用 *lint 规则*(如 `pylint` 自定义 check)事后强制?

---

*下一章预告*: **Chapter 19 — My Project Is Not Object-Oriented. How Do I Make Safe Changes?** — 把 ch18 的"test layout"推进到 non-OO 项目: C / COBOL / FORTRAN 没有 package / class / interface, ch18 的"IDE 字母序"假设失效。本章给出 (a) link seam + fake library(Feathers 在 ch4 预告),(b) preprocessing seam `#define ksr_notify(...)`(C 独有的 seam),(c) 把 `ksr_notify` 封装成 OO class 的 incremental 路径 — 当语言有 OO 扩展(C++ / VB / COBOL OO)时, 可以 *piece by piece* 升级。这是 ch25 的具体应用。