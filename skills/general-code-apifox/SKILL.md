---
name: apifox-interface-ops
description: Query and operate APIFox API definitions via MCP. Invoke only when the user provides APIFox links and requests query/create/modify operations.
origin: user
---

# APIFox Interface Ops

通过 APIFox 链接完成接口查询/新增/修改，并按项目现有规范落地代码。

## Activate Only When

- 用户提供了 APIFox 链接，且要求 `查询`、`新增`、`修改`、`删除`、`对齐需求` 等操作。
- 支持一次输入多个 APIFox 链接。

不满足以下任一条件则不处理：

- 没有 APIFox 链接。
- 只有泛化需求，无法定位具体接口。

## APIFox Link Gate (Mandatory)

1. 提取用户输入中的 APIFox 链接（可多个）。
2. 未提取到链接时直接返回“未检测到 APIFox 链接，本次不处理”。
3. 多链接时进入批量模式，逐个处理并汇总结果。

## MCP Setup

支持 Trae、Claude、Cursor、Codex，服务名可自定义：

```json
{
  "mcpServers": {
    "<any-server-name>": {
      "url": "https://apifox.com/api/v1/mcp",
      "headers": {
        "Authorization": "Bearer <access_token>",
        "X-Apifox-Api-Version": "2025-09-01"
      }
    }
  }
}
```

Claude/Codex 常见写法：

```json
{
  "mcpServers": {
    "<any-server-name>": {
      "type": "http",
      "url": "https://api.apifox.com/mcp",
      "headers": {
        "Authorization": "Bearer <access_token>",
        "X-Apifox-Api-Version": "2025-09-01"
      }
    }
  }
}
```

配置路径：

- `trae`: `~/Library/Application Support/Trae/User/mcp.json`
- `claude`: `~/.claude.json`
- `cursor`: `~/.cursor/mcp.json`

## MCP Preflight (Mandatory)

读取接口前必须校验：

1. 存在任意符合 APIFox 结构的 MCP 服务（不要求固定名称）。
2. `url` 为 `https://apifox.com/api/v1/mcp` 或 `https://api.apifox.com/mcp`。
3. `headers.Authorization` 为 `Bearer <access_token>` 形式。
4. 存在 `headers.X-Apifox-Api-Version`（推荐 `2025-09-01`）。
5. 若客户端要求显式类型（如 Claude/Codex），校验 `type: "http"`。

预检失败则停止并给出精确修复建议。

## Core Rules

1. 先检查当前项目是否已有对应接口实现。
2. 若是重复创建风险，必须先询问用户：继续创建、覆盖、还是复用。
3. 多链接必须批量处理，并返回每条链接的状态（`已查询`/`已新增`/`已更新`/`待确认`/`失败`）。
4. 若有需求文档或明确需求，输出字段级差异：`新增字段`、`删除字段`、`字段变更`、`新增接口`。
5. 即使用户只想“查看接口信息”，也要检索当前项目并标记：`已存在`、`未定义`、`未使用`。
6. 同时检查参数使用情况，判断是否新增参数（含请求与响应），并给出影响点。
7. 无明确需求差异时，只做查询与结构化反馈，不臆造变更。

## Focus Scope (Important)

默认重点关注：

- `method`
- `url`
- 请求参数
- 响应参数

公共请求头处理规则：

- 若项目已在 fetch/request 封装层统一处理公共请求头，则忽略，不重复处理。
- 仅在两种情况下再关注公共请求头：用户明确要求关注，或该字段明显异常（缺失/冲突/高风险）。

## Project Convention Rules

落地代码前先扫描并复用现有规范：

1. 不同域名是否使用不同 fetch/request 封装。
2. 同域接口的目录、命名、错误处理、重试、序列化习惯。
3. 参数加密、签名、响应解密等安全逻辑。

禁止引入第二套请求架构。

## TypeScript Rules

TS 项目必须保证类型定义规范、可维护、可复用（请求/响应/分页/错误体）。

字段注释使用块注释，便于悬浮提示：

```ts
/**
 * 用户唯一标识
 */
userId: string
```

枚举规则：

- 明确数字枚举可使用 `enum`（或等价常量方案）。
- 通用枚举先复用后新增（如系统类型、管理员身份、布尔状态）。
- 避免重复定义同语义类型/枚举。

## Ask-Before-Action

以下情况必须先询问用户：

- 重复创建风险。
- 落点不唯一（无法确定改哪个文件）。
- 需求与 APIFox 定义冲突。
- 破坏性删除（字段/接口）。

## Output Contract

每次输出按顺序包含：

1. 链接统计（单个/批量、成功解析数）。
2. MCP 预检结果。
3. 每个链接的接口定位结果：`已存在`/`未定义`/`未使用`。
4. 参数变化与使用分析：是否新增参数、影响位置、风险说明。
5. 差异分析（新增字段/删除字段/字段变更/新增接口）。
6. 代码变更方案（含封装复用与安全逻辑对齐说明）。
7. TS 类型与枚举复用/新增决策。
8. 待用户确认项（如有）。