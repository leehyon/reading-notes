---
title: Working Effectively with Legacy Code
---

**书名**: Working Effectively with Legacy Code
**作者**: Michael C. Feathers
**出版社/年份**: Prentice Hall (Robert C. Martin Series), 2004 (本 PDF 版次 2005)
**原文路径**: `dev/books/Working_Effectively_with_Legacy_Code_2005.pdf`

## 阅读约定

每章按六段式中文模板输出:
1. 章节概述 (5~10 条要点)
2. 核心 Takeaways (是什么 / 为什么 / 解决什么 / 适用场景)
3. 工程实践视角 (锁: **Linux 系统开发 + 机器人软件**)
4. AI 时代视角 (含「AI 经常写错的地方」表 + 子段「AI 辅助遗留代码理解」)
5. 实践行动项 (`### A1 — 标题` 格式, 必须可编译运行)
6. 值得深入思考的问题

末尾追加 `*下一章预告*`。

## 章节索引

| 章节 | 标题 | PDF 页范围 | 页数 | 笔记 |
|------|------|-----------|------|------|
| 1 | Changing Software | 25-30 | 6p | ✅ [ch01-changing-software.md](ch01-changing-software.md) |
| 2 | Working with Feedback | 31-42 | 12p | ✅ [ch02-working-with-feedback.md](ch02-working-with-feedback.md) |
| 3 | Sensing and Separation | 43-50 | 8p | ✅ [ch03-sensing-and-separation.md](ch03-sensing-and-separation.md) |
| 4 | The Seam Model | 51-66 | 16p | ✅ [ch04-the-seam-model.md](ch04-the-seam-model.md) |
| 5 | Tools | 67-78 | 12p | ✅ [ch05-tools.md](ch05-tools.md) |
| 6 | I Don't Have Much Time and I Have to Change It | 79-98 | 20p | ✅ [ch06-not-much-time.md](ch06-not-much-time.md) |
| 7 | It Takes Forever to Make a Change | 99-108 | 10p | ✅ [ch07-forever-to-change.md](ch07-forever-to-change.md) |
| 8 | How Do I Add a Feature? | 109-126 | 18p | ✅ [ch08-how-do-i-add-a-feature.md](ch08-how-do-i-add-a-feature.md) |
| 9 | I Can't Get This Class into a Test Harness | 127-158 | 32p | ✅ [ch09-cant-get-class-into-harness.md](ch09-cant-get-class-into-harness.md) |
| 10 | I Can’t Run This Method | 159-172 | 14p | ✅ [ch10-cant-run-method-in-harness.md](ch10-cant-run-method-in-harness.md) |
| 11 | I Need to Make a Change. What Methods Should I Test? | 173-194 | 22p | ✅ [ch11-make-a-change-what-methods-to-test.md](ch11-make-a-change-what-methods-to-test.md) |
| 12 | I Need to Make Many Changes in One Area | 195-206 | 12p | ✅ [ch12-many-changes-in-one-area.md](ch12-many-changes-in-one-area.md) |
| 13 | I Need to Make a Change, but I Don't Know What Tests to Write | 207-218 | 12p | ✅ [ch13-dont-know-what-tests-to-write.md](ch13-dont-know-what-tests-to-write.md) |
| 14 | Dependencies on Libraries Are Killing Me | 219-220 | 2p | ✅ [ch14-dependencies-on-libraries.md](ch14-dependencies-on-libraries.md) |
| 15 | My Application Is All API Calls and No Logic | 221-230 | 10p | ✅ [ch15-application-is-all-api-calls.md](ch15-application-is-all-api-calls.md) |
| 16 | I Don't Understand the Code Well Enough to Change It | 231-236 | 6p | ✅ [ch16-i-dont-understand-the-code.md](ch16-i-dont-understand-the-code.md) |
| 17 | My Application Has No Structure | 237-248 | 12p | ✅ [ch17-my-application-has-no-structure.md](ch17-my-application-has-no-structure.md) |
| 18 | My Test Code Is in the Way | 249-252 | 4p | ✅ [ch18-test-code-in-the-way.md](ch18-test-code-in-the-way.md) |
| 19 | My Project Is Not Object-Oriented. How Do I Make Safe Changes? | 253-266 | 14p | ✅ [ch19-not-object-oriented.md](ch19-not-object-oriented.md) |
| 20 | This Class Is Too Big and I Don't Want It to Get Any Bigger | 267-290 | 24p | ✅ [ch20-class-too-big.md](ch20-class-too-big.md) |
| 21 | I'm Changing the Same Code All Over the Place | 291-310 | 20p | ✅ [ch21-changing-same-code-all-over.md](ch21-changing-same-code-all-over.md) |
| 22 | I Need to Change a Monster Method and I Can't Write Tests for It | 311-330 | 20p | ✅ [ch22-monster-method.md](ch22-monster-method.md) |
| 23 | How Do I Know That I'm Not Breaking Anything? | 331-340 | 10p | ✅ [ch23-not-breaking-anything.md](ch23-not-breaking-anything.md) |
| 24 | We Feel Overwhelmed. It Isn't Going to Get Better. | 341-346 | 6p | ✅ [ch24-overwhelmed.md](ch24-overwhelmed.md) |
| 25 | Dependency-Breaking Techniques (reference catalog) | 347-456 | 110p | ✅ [ch25-dependency-breaking-techniques.md](ch25-dependency-breaking-techniques.md) |

## 三大部分结构

- **Part I — The Mechanics of Change** (PDF p.23-75, ch1-ch5): 概念框架
  - 改变的四种理由 / 反馈的工作方式 / 感知与分离 / seam 模型 / 工具
- **Part II — Changing Software** (PDF p.77-344, ch6-ch24): 食谱 — 按"症状"分类的具体技巧
- **Part III — Dependency-Breaking Techniques** (PDF p.345-435, ch25): 参考目录 — 25 个拆分依赖的手法

