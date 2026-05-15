---
name: general-test-planner
description: 基于 general-test-git-analyzer 产出的变更分析报告，生成结构化的自测文档和测试用例清单（含 E2E 用例和单元测试用例）。输出可直接被 general-test-executor 和 general-test-unit-executor 消费执行。
---

# 自测文档生成器

## 触发条件

当用户提出以下诉求之一时启用本 skill：

- "生成自测文档"
- "给出测试用例"
- "帮我写自测计划"
- 在 `general-test-auto` 编排流程中被调用（阶段 2）

## 前置条件

需要以下输入之一（优先级从高到低）：

1. `docs/self-test/change-analysis.md`（由 general-test-git-analyzer 产出）
2. 用户口述的改动说明 + 当前分支 diff

如果前置分析文档不存在，**先执行 general-test-git-analyzer skill**。

## 执行步骤

### 1. 读取变更分析

读取 `docs/self-test/change-analysis.md`，提取：
- 变更文件及其业务含义
- 影响范围与回归建议
- 风险点列表
- 每个变更文件的"单测适用性"标注

### 2. 深入理解业务逻辑

对变更分析中 **风险等级为中/高** 的文件：
- 读取文件完整源码
- 理解组件的 props、events、computed、watch
- 追踪 Vuex actions/mutations 的调用链路
- 识别接口调用的触发时机和响应处理
- 分析路由守卫和页面跳转逻辑

### 3. 生成 E2E 测试用例

每个 E2E 测试用例遵循以下结构（由 general-test-executor 消费）：

```yaml
- id: TC-001
  module: 模块名
  title: 用例标题
  priority: P0/P1/P2
  type: 功能/交互/边界/异常/回归
  precondition: 前置条件描述
  steps:
    - step: 1
      action: 具体操作描述
      expected: 期望结果
  test_url: 测试页面 URL 路径
  automation_hint: |
    给 general-test-executor 的自动化提示，描述：
    - 需要导航到的页面路径
    - 需要交互的元素特征（文本、位置）
    - 需要验证的内容（文本出现/消失、元素状态）
    - 需要检查的网络请求（如有）
```

### 4. 生成单元测试用例

根据变更分析中的"单测适用性"标注，为适合单测的文件生成用例（由 general-test-unit-executor 消费）。

#### 单测适用性判断规则

| 变更文件路径 | 单测类型 | 优先级 | 说明 |
|-------------|---------|--------|------|
| `src/utils/`, `src/global/` | 纯函数测试 | 高 | 无副作用的工具函数，最适合单测 |
| `src/store/` | Vuex 测试 | 高 | mutation/action/getter 可独立测试 |
| `src/components/` | 组件测试 | 中 | 使用 `@vue/test-utils` shallowMount |
| `src/phpApi/`, `src/javaApi/` | API mock 测试 | 中 | 验证请求参数组装和响应处理 |
| `src/router/` | 路由配置测试 | 低 | 路由守卫逻辑可单测 |
| `*.scss`, `src/locale/`, `src/x7lang/` | 不生成单测 | - | 样式/国际化文件不适合单测 |
| `src/constantConfig/`, `src/featureFlag/` | 不生成单测 | - | 纯配置文件不需要单测 |

如果变更文件中没有任何适合单测的文件，则跳过本步骤，`test-plan.md` 中不包含 `## 单元测试用例` 段落。

#### 单元测试用例结构

```yaml
- id: UT-001
  module: 模块名
  title: 用例标题
  priority: P0/P1/P2
  target_file: src/utils/xxx.js
  test_file: tests/unit/utils/xxx.spec.js
  target_function: functionName
  test_type: pure-function / vuex-mutation / vuex-action / vuex-getter / component-render / component-interaction / api-mock
  test_cases:
    - description: 正常入参返回预期结果
      input: { key: "value" }
      expected: { result: "xxx" }
    - description: 边界/异常情况
      input: null
      expected: throws Error
  mock_dependencies:
    - module: "@/phpApi/xxx"
      mock_return: { code: 0, data: {} }
  setup_hint: |
    给 general-test-unit-executor 的提示，描述：
    - 需要 mock 的模块及返回值
    - 需要初始化的 Vuex state（如测试 store）
    - 需要注入的 props/provide（如测试组件）
    - 特殊的测试环境配置
```

#### 单测用例设计要点

- **纯函数**：覆盖正常入参、边界值（空值/undefined/极端数据）、异常场景
- **Vuex mutation**：验证 state 变更是否符合预期
- **Vuex action**：mock 异步依赖，验证 commit 调用和返回值
- **Vuex getter**：给定不同 state，验证计算结果
- **组件渲染**：验证 props 不同值时的渲染输出（shallowMount + snapshot 或 DOM 断言）
- **组件交互**：trigger 事件后验证 emit 和内部状态变化
- **API 层**：mock HTTP 请求，验证参数组装和响应数据转换

### 5. 用例设计原则

#### 优先级定义
- **P0 - 冒烟**：改动的核心功能是否可用，必须测试
- **P1 - 核心流程**：主要业务流程、关键交互
- **P2 - 边界回归**：边界条件、异常处理、关联功能回归

#### 用例覆盖维度
- **功能正确性**：改动的功能按预期工作
- **交互体验**：点击/滑动/输入等交互响应正常
- **异常处理**：网络异常、空数据、超时等异常场景
- **边界条件**：极端数据、并发操作、页面刷新
- **回归验证**：与改动关联但未直接修改的功能

#### 针对本项目特点的测试关注点
- **多语言**：涉及 i18n key 变更时，至少抽查 1-2 种语言的显示
- **RTL 布局**：如有样式改动，关注阿拉伯语等 RTL 语言的布局
- **PWA 缓存**：如涉及资源变更，关注 Service Worker 缓存更新
- **移动端适配**：验证不同屏幕尺寸下的显示效果
- **Vuex 状态一致性**：跨页面状态是否正确保持/清理

### 6. 生成执行计划

将用例按执行顺序编排：

**E2E 用例编排**：
1. P0 冒烟用例优先
2. 同模块用例连续执行（减少页面跳转）
3. 相互依赖的用例按依赖顺序排列

**单元测试用例编排**：
1. 纯函数测试优先（执行最快、最稳定）
2. Vuex store 测试次之
3. 组件测试最后（需要挂载环境，相对较慢）

## 输出格式

输出一份 Markdown 文档（保存到 `docs/self-test/test-plan.md`），结构如下：

```markdown
# 自测文档

## 测试概览
- **关联分支**: feature/xxx
- **测试范围**: 一句话描述
- **E2E 用例数**: N（P0: x, P1: y, P2: z）
- **单测用例数**: M（P0: a, P1: b, P2: c）
- **预计耗时**: 约 x 分钟（单测 + E2E 自动化执行）
- **测试基准 URL**: http://localhost:8080（或实际开发服务地址）

## 前置准备
<!-- 运行测试前需要的准备工作 -->
- [ ] 开发服务已启动（E2E 需要）
- [ ] Jest 环境已就绪（单测需要，首次由 unit-executor 自动初始化）
- [ ] 已登录测试账号（如需要）
- [ ] 测试数据已准备（如需要）

## 测试用例

> 以下为 E2E 测试用例，由 general-test-executor 在浏览器中执行。

### 模块 A：xxx

#### TC-001: [P0] 用例标题
- **类型**: 功能
- **前置条件**: ...
- **步骤**:
  1. 操作 → 期望结果
  2. 操作 → 期望结果
- **自动化提示**:
  - 导航: /path/to/page
  - 操作: 点击"xxx"按钮
  - 验证: 页面出现"xxx"文本

#### TC-002: [P1] 用例标题
...

### 模块 B：xxx
...

## 单元测试用例

> 以下为单元测试用例，由 general-test-unit-executor 生成测试代码并通过 Jest 执行。
> 如果本次变更中无适合单测的文件，此段落省略。

### 模块 A：xxx

#### UT-001: [P0] 用例标题
- **被测文件**: src/utils/xxx.js
- **测试文件**: tests/unit/utils/xxx.spec.js
- **被测函数**: functionName
- **测试类型**: pure-function
- **测试场景**:
  1. 正常入参 → 预期返回值
  2. 空值入参 → 预期行为
  3. 异常入参 → 预期错误
- **mock 依赖**: 无
- **setup 提示**: 无特殊配置

#### UT-002: [P1] 用例标题
- **被测文件**: src/store/modules/xxx.js
- **测试文件**: tests/unit/store/xxx.spec.js
- **被测函数**: mutationName / actionName
- **测试类型**: vuex-mutation
- **测试场景**:
  1. 给定初始 state → mutation 后 state 符合预期
  2. ...
- **mock 依赖**: @/phpApi/xxx → { code: 0, data: {} }
- **setup 提示**: 需要初始化 Vuex store

### 模块 B：xxx
...

## 回归清单
| 功能/页面 | 回归原因 | 验证方式 |
|-----------|---------|---------|

## 已知限制
<!-- 无法通过自动化覆盖的场景，需手动验证 -->
- ...
```

## 约束

- E2E 用例步骤必须具体到可操作，禁止"检查页面正常"之类的模糊描述
- E2E `automation_hint` 必须给出足够信息让 general-test-executor 能自动执行
- 单测 `setup_hint` 必须给出足够信息让 general-test-unit-executor 能生成可运行的测试代码
- 单测用例的 `target_function` 必须是源文件中真实导出的函数/方法名
- 涉及登录态的 E2E 用例，在前置条件中明确说明
- 无法自动化的场景（如需要真实支付/短信验证码等）标注到"已知限制"
- 如果变更文件均不适合单测，`test-plan.md` 中不应出现 `## 单元测试用例` 段落
