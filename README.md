# Agent Tool Drawers

一个轻量的 agent 工具路由架构：把工具按能力分成「抽屉」，每一轮对话只打开当前需要的抽屉，而不是把所有工具一次性暴露给模型。

它适合长期陪伴 agent、roleplay bot、桌面助手、本地自动化 agent、MCP server，以及任何工具数量开始变多、上下文开始变乱的项目。

> Status: early public release. This repository describes a clean-room architecture and example implementation. It does not include private prompts, production tools, user data, credentials, domains, or chat logs.

## 为什么需要 Tool Drawers

很多 agent 项目一开始只有几个工具，直接把全部 tool schema 塞给模型没有问题。

但当工具增长到几十个以后，常见问题会变得明显：

- 上下文变重，模型每轮都要读大量不相关工具。
- 工具选择变差，容易误调用相似工具。
- 高风险工具和普通查询工具混在一起，缺少清楚边界。
- 不同入口需要不同工具集合，比如 web chat、voice、desktop app、scheduled job。
- 本地工具、HTTP 工具、MCP 工具缺少统一调用层。

Tool Drawers 的目标是把这些能力整理成可控的结构：

```txt
user message
  -> deterministic force rules
  -> optional LLM router
  -> selected drawers
  -> focused tool schemas
  -> local gateway
  -> tool result
```

模型不需要每次背着整个工具箱说话。它只需要打开这一轮真正可能用到的抽屉。

## 核心概念

### Drawer

Drawer 是一组相关工具。它通常代表一个稳定能力域，例如 `memory`、`calendar`、`device`、`messages`。

```ts
export interface Drawer {
  id: string;
  label: string;
  description: string;
  tools: string[];
}

export const drawers: Record<string, Drawer> = {
  memory: {
    id: "memory",
    label: "Memory",
    description: "Search and write long-term companion memories.",
    tools: ["search_memory", "write_memory"],
  },
  calendar: {
    id: "calendar",
    label: "Calendar",
    description: "Read upcoming events and reminders.",
    tools: ["list_events"],
  },
  device: {
    id: "device",
    label: "Device",
    description: "Control a local companion device.",
    tools: ["speak", "set_expression", "set_light"],
  },
};
```

### Force Rules

Force Rules 是确定性的关键词规则，适合这些场景：

- 低延迟入口，例如语音控制。
- 模型容易误判的短句，例如「继续」「停下」「记一下」。
- 高风险工具，例如外发消息、设备控制、支付、删除数据。
- 用户意图非常明确，不值得再花一次 LLM router 调用。

```ts
export const forceRules = [
  {
    pattern: /remember|save this|don't forget/i,
    drawers: ["memory"],
  },
  {
    pattern: /tomorrow|schedule|calendar|meeting/i,
    drawers: ["calendar"],
  },
  {
    pattern: /speak|say it|turn your head|light/i,
    drawers: ["device"],
  },
];
```

### LLM Router

LLM Router 是可选的轻量模型判断。它在关键词没有覆盖时，根据语义选择少量 drawer。

```ts
const selectedDrawers = await selectDrawers(
  "Can you check what I have tomorrow?",
  {
    llm: true,
    maxDrawers: 2,
  }
);
```

Router 应该输出结构化 JSON，而不是自由文本：

```json
{
  "drawers": ["calendar"]
}
```

推荐约束：

- 一轮最多选择 2-3 个 drawer。
- 不确定时返回空数组。
- 不做最终行动决策，只决定「哪些能力可能需要被看见」。
- 高风险工具仍然需要确认、权限或业务层 guard。

### Gateway

Gateway 是统一执行层。聊天入口、语音入口、scheduled job、MCP wrapper 都可以通过同一个 gateway 调用工具。

```ts
registerTool({
  name: "search_memory",
  drawer: "memory",
  description: "Search long-term memory.",
  inputSchema: {
    type: "object",
    properties: {
      query: { type: "string" },
    },
    required: ["query"],
  },
  run: async ({ query }) => {
    return await memory.search(query);
  },
});

const result = await callTool("search_memory", {
  query: "birthday plan",
});
```

Gateway 的职责不是让工具变神秘，而是让工具调用有统一入口：

- register tools
- list tools by drawer
- return schemas for selected drawers
- call tools by name
- normalize success and error responses
- log execution metadata without leaking private data

## 最小流程

```ts
const forced = forceMatchDrawers(userMessage, forceRules);
const routed = await routeWithLlm(userMessage, drawers);
const selected = mergeDrawers(forced, routed, { maxDrawers: 3 });

const schemas = getToolSchemasForDrawers(selected);
const answer = await callModel({
  messages,
  tools: schemas,
});

for (const toolCall of answer.toolCalls) {
  const result = await callTool(toolCall.name, toolCall.arguments);
  await appendToolResult(toolCall.id, result);
}
```

## 适合用在什么项目

- 长期陪伴 agent
- roleplay / character agent
- 桌面助手
- companion robot / desktop pet
- MCP server
- 本地自动化工具箱
- 多渠道 agent，例如 web、voice、desktop app、scheduled job
- 需要区分普通工具和高风险工具的系统

## 不适合解决什么问题

Tool Drawers 不是完整 agent framework。它不负责：

- 记忆数据库设计
- prompt 管理
- 模型供应商封装
- 权限系统
- 用户认证
- 任务队列
- UI

它只解决一个问题：当工具变多时，如何让 agent 每轮只看见当前需要的工具集合。

## 推荐架构

```txt
src/
  drawer.ts        # Drawer and tool metadata types
  force-rules.ts   # Deterministic routing rules
  router.ts        # Optional LLM router
  gateway.ts       # Register/list/call tools
  schemas.ts       # Tool schema helpers
  logger.ts        # Redacted execution logs
examples/
  basic-agent/
  desktop-agent/
docs/
  architecture.md
  privacy.md
```

## 安全边界

Tool routing 不等于 permission system。

一个 drawer 被选中，只代表这一轮可以把相关 tool schema 给模型看。真正执行工具前，仍然应该有业务层 guard。

建议把工具分成几类：

- `read`: 读取信息，例如搜索记忆、查看日程。
- `write`: 写入本地状态，例如保存笔记、更新偏好。
- `external`: 对外发送消息、发帖、调用第三方 API。
- `device`: 控制本地设备、声音、灯光或机器人。
- `destructive`: 删除、覆盖、付款、不可逆动作。

高风险工具应该额外要求确认：

```ts
registerTool({
  name: "send_message",
  drawer: "messages",
  risk: "external",
  requiresConfirmation: true,
  run: async ({ recipient, text }) => {
    return await messages.send({ recipient, text });
  },
});
```

## Privacy

这个项目只发布架构，不发布私人系统。

如果你从自己的 agent 项目中抽取 Tool Drawers，请使用全新的仓库、全新的示例、全新的 commit history。不要直接从生产项目删改后发布。

建议检查：

```bash
rg -n "SECRET|TOKEN|PRIVATE|PASSWORD|API_KEY|AUTH|COOKIE|SESSION"
rg -n "localhost|127\\.0\\.0\\.1|/opt/|/home/|\\.env|\\.sqlite|\\.db"
rg -n "telegram|discord|twitter|gmail|calendar|domain\\.com"
```

更多建议见 [docs/privacy.md](docs/privacy.md)。

## Architecture

完整架构说明见 [docs/architecture.md](docs/architecture.md)。

## Prior Art / Acknowledgements

This project is a clean-room implementation of a common agent architecture pattern: group tools into capability drawers, route each turn to a small subset of drawers, then execute through a local gateway.

It was inspired by public tutorials, examples, and discussions around LLM tool routing, capability routing, router chains, MCP-style tools, and ReAct-style agents. No third-party code is intentionally included.

If you recognize a specific source that should be credited, please open an issue or PR.

## License

MIT
