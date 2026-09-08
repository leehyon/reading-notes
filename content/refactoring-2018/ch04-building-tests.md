# Refactoring 2/e · Chapter 4 — Building Tests

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 4。
> 章节定位:refactoring 的前置保险丝——自测试代码(self-testing code)。JUnit / PyTest / CMocka 都是同一哲学的不同实现。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的论证**:**没有测试,refactoring 是「在悬崖边闭眼跳」** — 行为不变的承诺没有 enforcement。**测试是 refactoring 的唯一判据**(与类型系统并列)。

**公理 1(自测试代码)**:每个 class 都应该有测试，跑测试 < 10 分钟,CI 必须能跑。

**公理 2(测试金字塔)**:底层是大量 unit test(快、广),上层少量 component / integration test(慢、深)。

**公理 3(测试是 learning tool)**:**写测试比读文档更快理解一个 module** — 测试即"我如何使用它"的最佳示例。

**假设 vs 结论**:
- 假设：测试是 release 前补的事
- 结论:**测试必须在写代码的同一时刻写**(Kent Beck 的 TDD 模式),**refactor 之前已经有完整测试网**

---

## 二、章节概述

1. **The Value of Self-Testing Code**:refactoring + 自测试 = 程序员可以自信地改任何地方。
2. **The JUnit Example**:JUnit 怎么用 — fixture、test method、断言。
3. **Adding More Tests**:测试不仅是 happy path,**boundary、null、异常**都要。
4. **Test Fixtures**:**fixture = 测试前置数据准备**。inline、shared object、helper function 三种风格。
5. **Putting the Tests to Work**:**refactor 中每次修改立刻跑测试**,失败就是红路标。
6. **Test Coverage**:**100% 覆盖是 myth,目标是「关键路径 + boundary」**。
7. **Where to Test First**:legacy code 测试技巧(Feathers 链接到 *Working Effectively with Legacy Code*)。

---

## 三、核心 Takeaways

### Takeaway 1 — 「测试是 refactoring 的前提」
- **是什么**:**refactoring 必须建立在「已有测试 + 跑得过」的基础上**。否则行为是否改变无法判别。
- **为什么重要**:**没有测试的 refactoring = 改完了不知道有没有 bug**。有测试 = 改完 1 跑测试 30 秒知道有没有事。
- **解决了什么问题**:legacy code 没法 refactor 的窘境 — **Feathers 的解法是「先写测试再 refactor」**,这正是 *Working Effectively with Legacy Code* 核心。
- **适用场景**:BSW 改 Cantp 流控算法，跑 CANoe 自动化 test 套件必须 green。

### Takeaway 2 — 「测试即文档」
- **是什么**:**Test 展示了 module 的「expected usage」** — 比 doc string / API doc 更可执行、更易维护。
- **为什么重要**:新人 onboarding 读 100 行 test 比读 100 行 doc 更快入门。
- **解决了什么问题**:doc 过期、test 必新(test 错了就 fail)。
- **适用场景**:BSW 配置工程师读 CanTp 的 integration test 学习 PDU 流转。

### Takeaway 3 — 「Test Fixture 三种风格」
- **是什么**:inline(每个 test 自己 new)、shared(共用一个 setUp 创建的)、helper(test factory function)。**没有银弹，看场景**。
- **为什么重要**:fixture 风格决定测试可读性 + 独立性 + 维护成本。
- **解决了什么问题**:「测试之间互串状态」的 flakiness。
- **适用场景**:CMocka 的 `setup()` / `teardown()` 选哪个，看是否 stateless。

### Takeaway 4 — 「Red-Green-Refactor cycle」
- **是什么**:**TDD 三步**:写一个 failing test → 写最少代码让它 pass → refactor。**refactor 是 cycle 的最后一步，不是独立活动**。
- **为什么重要**:它把 chapter 1 的「refactor rhythm」嵌入到日常开发,**而不是「feature 写完才回头 refactor」**。
- **解决了什么问题**:sprint 末尾的「tech debt 还款日」永远不够用——因为从来不在 add feature 时同步 refactor。
- **适用场景**:每个 PR 都包含 add test + implement + refactor,**而不是一个 sprint 全 feature 下一个 sprint 全 refactor**。

---

## 四、工程实践视角

### 如何落地
- **CI 必须跑测试**:每次 commit 触发,**失败直接 block merge**。
- **测试覆盖率可视化**:Codecov / Coverage.py,目标是「关键路径 100%」「整体 > 80%」。**不要追求 100% — 成本曲线急速上升**。
- **Test Doubles(mocks / stubs / fakes)**:Cantp 测试要用 CANoe 模拟 CAN 帧，但单元测试要用 mock CanIf。**区分 test pyramid 各层用什么 double**。

### 常见误区(初级工程师)
- **「测试覆盖率高 = 代码好」**:**覆盖率是 lagging indicator,不是 leading**。**100% 覆盖但 assertion 是 `assertTrue(true)` 没价值**。
- **「写一次代码就不改测试」**:**测试也是 refactoring 的对象**(Rename Test Helper、Extract Test Fixture 都是常见 smell)。
- **「集成测试越多越好」**:**集成测试慢 + 难维护 + flakiness 高**。金字塔要避免倒挂。

### 高级工程师更关注
- **Property-Based Testing**(Hypothesis / QuickCheck):**不写具体输入，只写 invariant** — 在 Cantp 这种「各种 PDU length 都能流控」的场景特别有效。
- **Mutation Testing**(PIT / Stryker):**通过自动改代码看测试是否还能 catch** — 比覆盖率更深的 metric。
- **Contract Test**:BSW 模块边界(CanIf → CanTp)应该有 contract test,**写测试的不是模块作者，而是 consumer**。

### 与 NeuSAR cCore V3.0 的潜在连接
- **CANoe 自动化测试** = 集成测试层,**但 Cantp 的 unit test 必须用 mock CanIf**。MCAL 硬件抽象层通常无法 unit test,**要靠 integration test + HIL**。
- **Dcm UDS service 测试**:**每个 service 应该有 happy path + NRC 异常路径 + timing violation 测试**。vector 的 vTESTstudio 提供这一套。
- **NvM 测试**:**写测试的最大挑战是「non-volatile 模拟」** — 需要 mock NvM driver 提供 read/write block,**别用真实 flash**(unit test 不可控)。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**100% 重要**。AI 能写 test,但不能保证 test 覆盖了「该覆盖的」。
- **AI 能够帮助什么**:
  - **生成 boilerplate 测试** — 给定 function signature,生成 happy path test。
  - **property-based invariant 建议** — 给定 function 描述，推断该有什么 invariant。
  - **mutation test 结果分析** — LLM 解释「为什么这个 mutation 没被 catch」并建议改进。
- **AI 无法替代什么**:
  - **「值得测什么」的判断** — 这是产品 / 业务决策,AI 不懂。
  - **「真实环境模拟」** — CANoe 仿真 vs 真车的差距只有工程师能识别。
  - **test 的最终 ownership** — 谁维护 test 决定 test 质量。
- **工程师必须掌握的核心能力**:
  - **写测试的能力**(AI 写 80%,人 review + 写 boundary case)。
  - **判断 test pyramid 是否健康的能力**(unit/component/integration 比例对不对)。
  - **flakiness 分析能力**(race condition 在测试里表现为什么，怎么 fix)。

---

## 六、实践行动项

1. **给最近一个 module 加 fixture + happy path + boundary 测试**(用本项目惯用的 framework:CMocka / GoogleTest / PyTest / Jest)。
2. **跑 mutation testing(PIT / Stryker / mutmut)** — 看 mutation score 多少,**< 70% 说明测试弱**。
3. **识别项目里「测试金字塔倒挂」的证据**(集成测试 > unit test 数量 / 耗时),**重构一个集成测试为 unit test + mock**。

---

## 七、值得深入思考的问题

1. **「100% test coverage」是不是真目标?** — 100% 行覆盖容易,**100% branch + 100% assertion quality 几乎不可能**。**真正的目标是什么?**
2. **Property-based testing 与 unit test 的边界** — 何时该用 hypothesis 写 invariant,何时该写具体 input/output?
3. **「test 是文档」** — 但 test 不解释 why,**只解释 what**。**API design intent 在哪里?** doc string 是 deprecated 吗?
4. **legacy code 没有测试怎么办?** — Feathers 的「characterization test + seam」具体怎么做?**NeuSAR cCore 既有代码覆盖如何?**

---

## 交叉引用

- 第 2 章 Principles → 测试是 refactor 的前置
- 第 1 章 First Example → 测试在 chapter 1 已「先写」,本章展开
- Feathers *Working Effectively with Legacy Code* → Legacy code 加测试的标准读物

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。
