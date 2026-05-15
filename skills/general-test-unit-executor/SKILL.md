---
name: general-test-unit-executor
description: 读取 general-test-planner 产出的自测文档中的单元测试用例，生成 Jest 测试代码并执行，产出单元测试报告。支持首次自动初始化 Jest 环境。
---

# 单元测试执行器

## 触发条件

当用户提出以下诉求之一时启用本 skill：

- "执行单测"
- "跑单元测试"
- "生成并运行单测"
- 在 `general-test-auto` 编排流程中被调用（阶段 3A）

## 前置条件

1. `docs/self-test/test-plan.md` 已存在（由 general-test-planner 产出）
2. `test-plan.md` 中包含 `## 单元测试用例` 段落

如果测试计划不存在，**先执行 general-test-planner skill**。
如果测试计划中不包含单元测试用例段落，向用户说明"本次变更无需单元测试"并跳过。

## Jest 环境初始化

每次执行前检查 Jest 环境是否就绪。如果是首次运行（项目中尚未安装测试依赖），自动完成初始化。

### 检测方式

```bash
# 检查 package.json 中是否有 test:unit script
node -e "const pkg = require('./package.json'); process.exit(pkg.scripts?.['test:unit'] ? 0 : 1)"
```

### 首次初始化步骤

如果检测到尚未初始化，按以下顺序执行：

**1. 安装依赖**

```bash
npm install --save-dev @vue/cli-plugin-unit-jest @vue/test-utils@1
```

> `@vue/test-utils@1` 对应 Vue 2；Vue 3 项目使用 `@vue/test-utils@2`。

**2. 生成 Jest 配置**

创建 `jest.config.js`：

```js
module.exports = {
  preset: '@vue/cli-plugin-unit-jest',
  testMatch: ['**/tests/unit/**/*.spec.js'],
  collectCoverageFrom: [
    'src/**/*.{js,vue}',
    '!src/main.js',
    '!src/router/**',
    '!src/locale/**',
    '!src/x7lang/**',
  ],
};
```

**3. 添加 npm script**

在 `package.json` 的 `scripts` 中添加：

```json
{
  "test:unit": "vue-cli-service test:unit",
  "test:unit:coverage": "vue-cli-service test:unit --coverage"
}
```

**4. 创建测试目录**

```bash
mkdir -p tests/unit
```

**5. 验证环境**

```bash
npx vue-cli-service test:unit --passWithNoTests
```

确认命令正常退出（exit code 0）后，初始化完成。

## 执行流程

### 1. 解析测试计划

读取 `docs/self-test/test-plan.md`，提取 `## 单元测试用例` 段落中的所有用例，解析出：
- 用例 ID、标题、优先级
- 被测文件路径、测试文件路径
- 被测函数名、测试类型
- 测试场景列表（输入/预期）
- mock 依赖和 setup 提示

### 2. 读取被测源码

对每个用例的 `target_file`：
- 使用 Read 工具读取完整源码
- 理解被测函数的签名、参数、返回值
- 识别内部依赖（import 的模块，需要 mock 的部分）
- 确认函数确实存在且与用例描述一致

### 3. 生成测试代码

根据用例规划和源码分析，为每个测试文件生成 `.spec.js` 代码。

#### 代码生成规范

**纯函数测试模板**：

```js
import { targetFunction } from '@/utils/xxx';

describe('targetFunction', () => {
  it('正常入参返回预期结果', () => {
    const result = targetFunction(input);
    expect(result).toEqual(expected);
  });

  it('空值入参返回默认值', () => {
    expect(targetFunction(null)).toBe(defaultValue);
  });

  it('异常入参抛出错误', () => {
    expect(() => targetFunction(invalidInput)).toThrow();
  });
});
```

**Vuex mutation 测试模板**：

```js
import { mutations } from '@/store/modules/xxx';

const { MUTATION_NAME } = mutations;

describe('MUTATION_NAME', () => {
  it('正确更新 state', () => {
    const state = { /* 初始 state */ };
    MUTATION_NAME(state, payload);
    expect(state.field).toBe(expectedValue);
  });
});
```

**Vuex action 测试模板**：

```js
import { actions } from '@/store/modules/xxx';

jest.mock('@/phpApi/xxx', () => ({
  apiMethod: jest.fn().mockResolvedValue({ code: 0, data: {} }),
}));

describe('actionName', () => {
  it('正确提交 mutation', async () => {
    const commit = jest.fn();
    const state = { /* 初始 state */ };
    await actions.actionName({ commit, state }, payload);
    expect(commit).toHaveBeenCalledWith('MUTATION_NAME', expectedPayload);
  });
});
```

**Vuex getter 测试模板**：

```js
import { getters } from '@/store/modules/xxx';

describe('getterName', () => {
  it('根据 state 正确计算', () => {
    const state = { /* 测试 state */ };
    const result = getters.getterName(state);
    expect(result).toEqual(expected);
  });
});
```

**Vue 组件测试模板**：

```js
import { shallowMount, createLocalVue } from '@vue/test-utils';
import Vuex from 'vuex';
import Component from '@/components/xxx.vue';

const localVue = createLocalVue();
localVue.use(Vuex);

describe('Component', () => {
  let wrapper;
  let store;

  beforeEach(() => {
    store = new Vuex.Store({ /* mock store */ });
    wrapper = shallowMount(Component, {
      localVue,
      store,
      propsData: { /* test props */ },
      mocks: {
        $t: (key) => key,       // mock i18n
        $route: { params: {} }, // mock route
      },
    });
  });

  afterEach(() => {
    wrapper.destroy();
  });

  it('渲染正确的结构', () => {
    expect(wrapper.find('.target-class').exists()).toBe(true);
  });

  it('点击触发正确的 emit', async () => {
    await wrapper.find('.btn').trigger('click');
    expect(wrapper.emitted('eventName')).toBeTruthy();
  });
});
```

**API 层测试模板**：

```js
import axios from 'axios';
import { apiMethod } from '@/phpApi/xxx';

jest.mock('axios');

describe('apiMethod', () => {
  it('发送正确的请求参数', async () => {
    axios.post.mockResolvedValue({ data: { code: 0, data: {} } });
    await apiMethod(params);
    expect(axios.post).toHaveBeenCalledWith(
      expectedUrl,
      expect.objectContaining(expectedParams),
    );
  });
});
```

#### 生成要求

- 测试文件路径遵循用例中指定的 `test_file` 字段
- 目录不存在时自动创建（`mkdir -p`）
- 每个测试文件必须能独立运行
- mock 声明必须在 import 之后、describe 之前
- 使用 `@/` 路径别名（Vue CLI 默认支持）
- 组件测试中 mock 掉 `$t`（i18n）、`$route`、`$router` 等全局注入

### 4. 执行测试

```bash
npx vue-cli-service test:unit --verbose 2>&1
```

如需 JSON 格式输出（用于程序化解析）：

```bash
npx vue-cli-service test:unit --json --outputFile=docs/self-test/unit-test-result.json 2>&1
```

#### 执行策略

- 一次性运行全部测试文件
- 如果存在失败用例，记录失败信息后继续（Jest 默认行为）
- 超时阈值：单个测试文件 30 秒（Jest 默认 5 秒/用例）

### 5. 收集结果

解析 Jest 输出（优先使用 JSON，fallback 解析终端文本），提取：
- 每个测试文件的通过/失败/跳过状态
- 失败用例的错误信息和堆栈
- 执行耗时
- 覆盖率数据（如启用）

### 6. 失败处理

- **测试文件语法错误**：记录错误，标记该用例为 FAIL，继续执行其他文件
- **import 找不到模块**：检查路径别名配置，尝试修复后重跑该文件
- **mock 不生效**：检查 mock 声明位置是否正确（必须在 import 之前或使用 `jest.mock` 声明提升）
- **组件挂载失败**：检查是否缺少必要的 mock（如 `$t`、`$store`），补充后重试一次
- **单次重试**：对首次失败的用例，尝试修复明显问题后重跑一次，仍失败则最终标记 FAIL

## 输出格式

输出测试报告（保存到 `docs/self-test/unit-test-report.md`），结构如下：

```markdown
# 单元测试报告

## 执行概览
- **执行时间**: 2024-xx-xx HH:mm
- **关联分支**: feature/xxx
- **测试文件数**: N
- **用例总数**: M
- **通过**: x
- **失败**: y
- **跳过**: z
- **通过率**: xx%
- **执行耗时**: x.xs

## 执行结果

### UT-001: [P0] 用例标题 — PASS
- **测试文件**: tests/unit/utils/xxx.spec.js
- **被测模块**: src/utils/xxx.js
- **耗时**: x.xs

### UT-002: [P1] 用例标题 — FAIL
- **测试文件**: tests/unit/store/xxx.spec.js
- **被测模块**: src/store/modules/xxx.js
- **失败场景**: "异常入参抛出错误"
- **期望结果**: throws Error
- **实际结果**: 返回 undefined（未抛出异常）
- **错误信息**: expect(received).toThrow() — Received function did not throw

### UT-003: [P2] 用例标题 — SKIP
- **跳过原因**: 被测文件依赖 native bridge，无法在 Node 环境运行

## 覆盖率摘要（如启用）
| 文件 | 语句 | 分支 | 函数 | 行 |
|------|------|------|------|------|
| src/utils/xxx.js | 95% | 80% | 100% | 95% |
| ... | ... | ... | ... | ... |

## 失败用例分析
<!-- 对失败用例归类分析，可能的原因和建议修复方式 -->

## 生成的测试文件清单
| 文件路径 | 对应用例 | 状态 |
|---------|---------|------|
| tests/unit/utils/xxx.spec.js | UT-001 | PASS |
| tests/unit/store/xxx.spec.js | UT-002 | FAIL |
| ... | ... | ... |
```

## 约束

- 生成的测试代码必须能独立运行，不依赖其他测试文件的执行顺序
- 不修改被测源文件，只生成测试文件
- mock 模块时使用 `jest.mock()` 声明提升，确保在 import 前生效
- 组件测试使用 `shallowMount` 而非 `mount`，避免子组件的副作用
- 测试文件中不包含硬编码的业务数据（如真实用户 ID、token 等）
- 生成测试前必须先 Read 被测源文件，确保测试与实际代码一致
- 首次初始化 Jest 环境时，安装完成后必须验证可正常运行
