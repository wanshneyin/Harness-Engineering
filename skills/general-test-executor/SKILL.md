---
name: general-test-executor
description: 读取 general-test-planner 产出的自测文档中的 E2E 测试用例，调用「AI自动化测试」MCP 在真实浏览器中逐条执行，产出 E2E 测试报告（含截图与通过/失败状态）。
---

# E2E 测试执行器

## 触发条件

当用户提出以下诉求之一时启用本 skill：

- "执行E2E自测"
- "跑一下测试用例"
- "用 MCP 测试"
- "跑 E2E"
- 在 `general-test-auto` 编排流程中被调用（阶段 3B）

## 前置条件

1. `docs/self-test/test-plan.md` 已存在（由 general-test-planner 产出）
2. 开发服务已启动（默认 `http://localhost:8080`，以 `.env` 中 `port` 字段为准）
3. MCP 服务 `AI自动化测试` 可用

如果测试计划不存在，**先执行 general-test-planner skill**。

## MCP 工具清单

本 skill 使用 `AI自动化测试` MCP 服务，可用工具：

| 工具 | 用途 |
|------|------|
| `browser_navigate` | 导航到指定 URL |
| `browser_snapshot` | 获取页面无障碍快照（用于获取元素 ref） |
| `browser_take_screenshot` | 截取页面截图（用于测试报告取证） |
| `browser_click` | 点击页面元素 |
| `browser_type` | 在输入框中输入文本 |
| `browser_fill_form` | 批量填写表单 |
| `browser_select_option` | 选择下拉选项 |
| `browser_hover` | 悬停元素（触发 tooltip/下拉菜单） |
| `browser_press_key` | 按键操作 |
| `browser_wait_for` | 等待文本出现/消失或指定时间 |
| `browser_evaluate` | 在页面执行 JavaScript（检查 Vuex 状态/localStorage 等） |
| `browser_network_requests` | 检查网络请求（验证接口调用） |
| `browser_console_messages` | 检查控制台消息（捕获错误） |
| `browser_tabs` | 管理浏览器标签页 |
| `browser_navigate_back` | 后退 |
| `browser_drag` | 拖拽操作 |
| `browser_resize` | 调整浏览器窗口大小（测试响应式） |
| `browser_close` | 关闭浏览器 |

## 执行流程

### 1. 解析测试计划

读取 `docs/self-test/test-plan.md`，提取 `## 测试用例` 段落中的 E2E 用例（跳过 `## 单元测试用例` 段落），解析出：
- 用例 ID（TC-xxx 格式）、标题、优先级
- 步骤列表（操作 + 期望结果）
- 自动化提示（导航路径、交互元素、验证条件）
- 前置条件

### 2. 环境预检

```
步骤 A：确认开发服务可访问
  → CallMcpTool: browser_navigate { url: "http://localhost:8080" }
  → CallMcpTool: browser_snapshot {}
  → 验证页面加载成功

步骤 B：如需登录，先完成登录流程
  → 根据测试计划中的"前置准备"执行
```

### 3. 逐条执行用例

对每个测试用例，执行以下循环：

```
┌─ 用例开始 ─────────────────────────────────────┐
│                                                  │
│  1. 导航到测试页面                                │
│     → browser_navigate { url: test_url }         │
│     → browser_wait_for { time: 2 }               │
│                                                  │
│  2. 获取页面快照                                  │
│     → browser_snapshot {}                         │
│     → 解析快照，定位目标元素的 ref                 │
│                                                  │
│  3. 执行操作步骤                                  │
│     → 根据步骤类型调用对应 MCP 工具               │
│     → 每步操作后等待页面响应                       │
│     → 重新获取快照确认状态变化                     │
│                                                  │
│  4. 验证期望结果                                  │
│     → browser_snapshot → 检查文本/元素是否存在     │
│     → browser_network_requests → 检查接口调用     │
│     → browser_evaluate → 检查 JS 状态             │
│                                                  │
│  5. 截图取证                                      │
│     → browser_take_screenshot {                   │
│         type: "png",                              │
│         filename: "TC-xxx-result.png"             │
│       }                                           │
│                                                  │
│  6. 记录结果：PASS / FAIL / SKIP                  │
│     → FAIL 时记录实际结果与期望结果的差异          │
│     → SKIP 时记录跳过原因                         │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 4. 操作映射规则

根据自动化提示中的操作描述，映射到 MCP 工具调用：


### 参考文档（详细映射表）
```markdown
| 操作描述 | MCP 工具 | 参数 |
|---------|---------|------|
| 导航到 URL | browser_navigate | `{ url: "https://..." }` |
| 后退 | browser_navigate_back | `{}` |
| 新标签页 | browser_tabs | `{ action: "new" }` |
| 切换标签页 | browser_tabs | `{ action: "select", index: 1 }` |
| 关闭标签页 | browser_tabs | `{ action: "close" }` |

#### 元素交互（需先 snapshot 获取 ref）
| 操作描述 | MCP 工具 | 参数 |
|---------|---------|------|
| 点击 | browser_click | `{ ref: "..." }` |
| 输入 | browser_type | `{ ref: "...", text: "..." }` |
| 批量填表 | browser_fill_form | `{ fields: [{ ref, value }] }` |
| 下拉选择 | browser_select_option | `{ ref: "...", values: ["option"] }` |
| 悬停 | browser_hover | `{ ref: "..." }` |
| 拖拽 | browser_drag | `{ startRef: "...", endRef: "..." }` |
| 上传文件 | browser_file_upload | `{ paths: ["/path/to/file"] }` |

#### 等待与验证
| 操作描述 | MCP 工具 | 参数 |
|---------|---------|------|
| 等待文本出现 | browser_wait_for | `{ text: "xxx" }` |
| 等待文本消失 | browser_wait_for | `{ textGone: "xxx" }` |
| 等待时间 | browser_wait_for | `{ time: 2000 }` |
| 获取页面快照 | browser_snapshot | `{}` |
| 截图 | browser_take_screenshot | `{ filename: "xxx.png", fullPage: true }` |

#### 高级操作
| 操作描述 | MCP 工具 | 参数 |
|---------|---------|------|
| 执行 JS | browser_evaluate | `{ function: "() => {...}" }` |
| 检查网络请求 | browser_network_requests | `{ includeStatic: false }` |
| 检查控制台 | browser_console_messages | `{ level: "error" }` |
| 处理弹窗 | browser_handle_dialog | `{ accept: true, promptText: "..." }` |
| 调整窗口 | browser_resize | `{ width: 375, height: 812 }` |
| 关闭页面 | browser_close | `{}` |

### 5. 元素定位策略

由于使用的是基于 snapshot 的 ref 定位，遵循以下策略：

1. **每次操作前先取快照**：`browser_snapshot` 获取最新 DOM 状态
2. **通过文本匹配定位**：在快照 YAML 中找到包含目标文本的元素 ref
3. **通过角色匹配**：button、textbox、link 等角色 + 名称
4. **多元素歧义时**：结合上下文（父容器、位置关系）选择正确 ref
5. **动态内容等待**：操作后先 `browser_wait_for` 再取快照

### 6. 失败处理策略

- **元素未找到**：等待 2 秒后重试一次，仍未找到则标记 FAIL
- **页面加载超时**：等待最多 10 秒，超时标记 FAIL
- **断言失败**：截图记录实际状态，继续执行下一条用例
- **浏览器异常**：记录错误信息，尝试重新导航后继续

## 输出格式

输出测试报告（保存到 `docs/self-test/e2e-test-report.md`），结构如下：

```markdown
# E2E 测试报告

## 执行概览
- **执行时间**: 2024-xx-xx HH:mm
- **关联分支**: feature/xxx
- **基准 URL**: http://localhost:8080
- **用例总数**: N
- **通过**: x ✅
- **失败**: y ❌
- **跳过**: z ⏭️
- **通过率**: xx%

## 执行结果

### ✅ TC-001: [P0] 用例标题
- **状态**: PASS
- **耗时**: xs
- **截图**: [查看截图](./screenshots/TC-001-result.png)

### ❌ TC-002: [P1] 用例标题
- **状态**: FAIL
- **耗时**: xs
- **失败步骤**: 第 3 步
- **期望结果**: xxx
- **实际结果**: xxx
- **截图**: [查看截图](./screenshots/TC-002-result.png)

### ⏭️ TC-003: [P2] 用例标题
- **状态**: SKIP
- **跳过原因**: 需要登录态，当前未配置测试账号

## 失败用例分析
<!-- 对失败用例归类分析，可能的原因 -->

## 需要手动验证的场景
<!-- 自动化无法覆盖的场景 -->
```

## 约束

- **每步操作前必须取快照**，不能复用旧的 ref
- 单条用例执行超过 60 秒视为超时
- 失败不阻断，记录后继续执行下一条
- 截图文件名与用例 ID 对应，便于追溯
- 涉及敏感操作（支付/删除）的用例标记为 SKIP，不自动执行
- 发现控制台 error 时，即使业务断言通过也要在报告中标注
