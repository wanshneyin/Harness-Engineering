---
name: general-test-auto
description: 一键自测编排器。串联 git-analyzer → test-planner → unit-executor / e2e-executor → 汇总报告，从分析代码改动到生成自测文档，再并行执行单元测试与 E2E 测试，完成完整的分支自测流程。
---

# 一键自测编排器

## 触发条件

当用户提出以下诉求之一时启用本 skill：

- "一键自测"
- "自测这个分支"
- "帮我跑一下自测"
- "分析改动并自测"
- "自动化自测"

## 完整流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          一键自测流程                                     │
│                                                                         │
│  阶段 1          阶段 2            阶段 3                阶段 4          │
│  ┌──────────┐   ┌──────────┐   ┌───────────────────┐   ┌──────────┐   │
│  │ 分析改动  │ → │ 生成文档  │ → │ 3A 单元测试        │ ┐ │ 汇总报告  │   │
│  │          │   │          │   │    unit-executor   │ ├→│          │   │
│  │git-      │   │test-     │   ├───────────────────┤ │ │合并单测   │   │
│  │analyzer  │   │planner   │   │ 3B E2E 测试        │ ┘ │+ E2E    │   │
│  └──────────┘   └──────────┘   │    e2e-executor   │   └──────────┘   │
│       │               │        └───────────────────┘        │         │
│       ▼               ▼          │             │            ▼         │
│  change-        test-plan.md  unit-test-   e2e-test-    告知用户      │
│  analysis.md    (含两类用例)  report.md    report.md    最终结果      │
└─────────────────────────────────────────────────────────────────────────┘
```

> **关键变化**：阶段 3 从单轨道变为双轨道。单元测试（3A）通过 Jest CLI 执行，不依赖浏览器；E2E 测试（3B）通过 MCP 浏览器执行。两者可先后或并行运行，互不阻断。若本次改动不涉及可单测的文件，3A 自动跳过，流程退化为原有纯 E2E 模式。

## 执行步骤

### 前置：创建产物目录

```bash
mkdir -p docs/self-test/screenshots
```

### 阶段 1：分析改动（general-test-git-analyzer）

**读取并执行** `.cursor/skills/general-test-git-analyzer/SKILL.md`

输入：
- 当前 Git 分支的 diff（相对基准分支）
- 用户可指定基准分支，默认自动探测

产物：
- `docs/self-test/change-analysis.md`（含每个变更文件的"单测适用性"标注）

阶段完成后向用户汇报：
- 本次共改动 N 个文件
- 涉及 N 个模块
- 高风险改动 N 处
- 其中 M 个文件适合单元测试
- 询问用户："变更分析已完成，是否继续生成测试用例？"
 
要求：
- **无论用户最初是否说过“一键自测/全流程”，阶段 1 结束后都必须暂停**，等待用户明确确认继续（例如：“继续/确认/开始阶段2”）。

### 阶段 2：生成自测文档（general-test-planner）

**读取并执行** `.cursor/skills/general-test-planner/SKILL.md`

输入：
- `docs/self-test/change-analysis.md`（阶段 1 产物）

产物：
- `docs/self-test/test-plan.md`（同时包含 E2E 测试用例和单元测试用例两个段落）

阶段完成后向用户汇报：
- 生成了 N 条 E2E 用例（P0: x, P1: y, P2: z）
- 生成了 M 条单元测试用例（P0: a, P1: b, P2: c）
- 覆盖 N 个模块
- 询问用户："自测文档已生成，是否开始执行测试？"
- 如果变更文件中没有可单测的模块，则说明"本次无需单元测试，将仅执行 E2E"
 
要求：
- **阶段 2 结束后必须暂停**，等待用户明确确认继续（例如：“开始执行/确认/开始阶段3”）。

### 阶段 3A：执行单元测试（general-test-unit-executor）

**读取并执行** `.cursor/skills/general-test-unit-executor/SKILL.md`

前置检查：
1. `test-plan.md` 中存在 `## 单元测试用例` 段落（如不存在则跳过 3A，直接进入 3B）
2. Jest 环境已就绪（首次运行时 unit-executor 会自动初始化）

输入：
- `docs/self-test/test-plan.md` 中的 `## 单元测试用例` 段

产物：
- `tests/unit/` 下生成的 `.spec.js` 测试文件
- `docs/self-test/unit-test-report.md`

### 阶段 3B：执行 E2E 测试（general-test-executor）

**读取并执行** `.cursor/skills/general-test-executor/SKILL.md`

前置检查：
1. `test-plan.md` 中存在 E2E 测试用例（`## 测试用例` 段落）
2. 确认开发服务已启动
   - 如果未启动，提示用户执行 `npm run serve` 并等待
3. 确认 `AI自动化测试` MCP 可用

输入：
- `docs/self-test/test-plan.md` 中的 E2E 测试用例段

产物：
- `docs/self-test/e2e-test-report.md`
- `docs/self-test/screenshots/TC-xxx-result.png`（每条用例的截图）

> **执行顺序说明**：建议先执行 3A（单测更快、无外部依赖），再执行 3B（E2E 需要浏览器和开发服务）。如果用户指定"仅单测"或"仅 E2E"，则只执行对应轨道。

### 阶段 4：汇总报告

合并 3A 和 3B 的结果，向用户输出最终摘要：

```
## 自测完成

### 执行摘要
- 分支: feature/xxx → develop
- 变更文件: N 个

#### 单元测试
- 用例数: M 条
- 通过: a | 失败: b | 跳过: c
- 通过率: xx%

#### E2E 测试
- 用例数: N 条
- 通过: x | 失败: y | 跳过: z
- 通过率: xx%

#### 综合
- 总用例: M+N 条
- 总通过率: xx%

### 产物清单
- 变更分析: docs/self-test/change-analysis.md
- 自测文档: docs/self-test/test-plan.md
- 单测报告: docs/self-test/unit-test-report.md
- E2E 报告: docs/self-test/e2e-test-report.md
- 截图目录: docs/self-test/screenshots/
- 单测代码: tests/unit/

### 需要关注
- [列出单测失败用例及原因]
- [列出 E2E 失败用例及可能原因]
- [列出跳过的用例及原因]
- [列出需手动验证的场景]
```

## 可选参数

用户可以通过自然语言指定以下参数：

| 参数 | 默认值 | 说明 |
|------|-------|------|
| 基准分支 | 自动探测 | "对比 develop 分支自测" |
| 测试范围 | 全部改动 | "只测登录模块" |
| 优先级过滤 | 全部 | "只跑 P0 用例" |
| 基准 URL | http://localhost:8080 | "测试环境地址是 xxx" |
| 跳过阶段 | 无 | "已有自测文档，直接执行" |
| 测试类型 | 全部 | "仅单测" / "仅 E2E" / "跳过单测" |

## 断点续跑

如果流程中断（如浏览器异常/用户中断），支持从指定阶段继续：

- "从阶段 2 继续"：读取已有的 `change-analysis.md`，跳过阶段 1
- "从阶段 3 继续"：读取已有的 `test-plan.md`，跳过阶段 1 和 2
- "只跑单测"：跳过 3B，只执行阶段 3A
- "只跑 E2E"：跳过 3A，只执行阶段 3B
- "重跑失败 E2E"：读取 `e2e-test-report.md`，只执行状态为 FAIL 的用例
- "重跑失败单测"：读取 `unit-test-report.md`，只执行失败的测试文件

## 约束

- 阶段 1 → 2 → 3 必须按顺序执行，后一阶段依赖前一阶段的产物
- 阶段 3A 和 3B 之间无依赖，可独立执行
- 每个阶段开始前校验前置产物是否存在
- 阶段 3A 执行前校验 Jest 环境（首次自动初始化）
- 阶段 3B 执行前必须确认开发服务和 MCP 可用
- 所有文档产物统一输出到 `docs/self-test/` 目录
- 单测代码输出到 `tests/unit/` 目录
- `docs/self-test/` 应加入 `.gitignore`（测试产物不入库）
