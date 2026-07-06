# Architecture

Agent Tool Drawers 是一个小型工具路由架构。它把 agent 可用的工具按能力域分组，每轮对话先选择少量相关 drawer，再把这些 drawer 里的 tool schemas 暴露给模型。

它不是完整 agent framework，而是一个可以嵌进现有系统的 routing layer。

## 设计目标

- 减少每轮 prompt 里的无关工具。
- 让工具选择更稳定、更可解释。
- 把高风险工具和普通查询工具分开管理。
- 支持多入口：chat、voice、desktop、scheduled job、MCP wrapper。
- 让本地工具、HTTP 工具和 MCP 工具有统一调用入口。
- 保持实现足够小，方便迁移到不同 agent 项目。

## 高层流程

```txt
                  ┌────────────────────┐
                  │    user message     │
                  └─────────┬──────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │ deterministic rules │
                  └─────────┬──────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │ optional LLM router │
                  └─────────┬──────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │  selected drawers   │
                  └─────────┬──────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │ focused tool schema │
                  └─────────┬──────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │     model turn      │
                  └─────────┬──────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │   gateway call      │
                  └────────────────────┘
```

## Components

### Drawer Catalog

Drawer catalog 是系统的能力地图。它应该短、稳定、语义清楚。

```ts
export interface Drawer {
  id: string;
  label: string;
  description: string;
  tools: string[];
}

export type DrawerCatalog = Record<string, Drawer>;
```

示例：

```ts
export const drawers = {
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

### Force Router

Force router 是确定性路由。它应该先于 LLM router 执行。

适合放进 force rules 的内容：

- 明确命令：`remember this`、`send a message`、`turn on the light`
- 高风险工具：外发、删除、设备控制
- 语音入口里的短反馈：`stop`、`continue`、`louder`
- 模型经常误判的固定短语

```ts
export interface ForceRule {
  pattern: RegExp;
  drawers: string[];
}

export function forceMatchDrawers(
  message: string,
  rules: ForceRule[],
): string[] {
  const selected = new Set<string>();
  for (const rule of rules) {
    if (rule.pattern.test(message)) {
      for (const drawer of rule.drawers) selected.add(drawer);
    }
  }
  return [...selected];
}
```

### LLM Router

LLM router 负责处理没有被 force rules 覆盖的语义判断。它应该只做 routing，不做业务决策。

推荐 system prompt：

```ts
const routerSystemPrompt = `
You are a tool drawer router.
Select at most 2 drawers that may help answer the user's latest message.
Return JSON only: { "drawers": ["drawer_id"] }.
If no tools are needed, return { "drawers": [] }.
`;
```

推荐输出：

```json
{
  "drawers": ["calendar"]
}
```

实现时应当过滤非法 drawer id：

```ts
function normalizeRouterOutput(
  value: unknown,
  catalog: DrawerCatalog,
): string[] {
  const drawers = Array.isArray((value as any)?.drawers)
    ? (value as any).drawers
    : [];

  return drawers.filter((id) => (
    typeof id === "string" && Boolean(catalog[id])
  ));
}
```

### Merge Policy

Force router 和 LLM router 的结果需要合并。通常 force rules 优先。

```ts
export function mergeDrawers(
  forced: string[],
  routed: string[],
  options = { maxDrawers: 3 },
): string[] {
  return [...new Set([...forced, ...routed])]
    .slice(0, options.maxDrawers);
}
```

推荐策略：

- 如果 force rules 命中高风险 drawer，不要被 LLM router 覆盖。
- 一轮最多 2-3 个 drawer。
- 对实时入口可以禁用 LLM router，只使用 force rules。
- 对自动任务可以使用固定 drawer set，避免不必要的模型路由。

### Tool Registry

Tool registry 保存工具定义、schema、drawer 归属和执行函数。

```ts
export type ToolRisk =
  | "read"
  | "write"
  | "external"
  | "device"
  | "destructive";

export interface RegisteredTool {
  name: string;
  drawer: string;
  description: string;
  risk?: ToolRisk;
  requiresConfirmation?: boolean;
  inputSchema?: unknown;
  run: (args: unknown) => Promise<unknown> | unknown;
}
```

注册示例：

```ts
registerTool({
  name: "list_events",
  drawer: "calendar",
  description: "List upcoming calendar events.",
  risk: "read",
  inputSchema: {
    type: "object",
    properties: {
      from: { type: "string" },
      to: { type: "string" },
    },
  },
  run: async ({ from, to }) => {
    return await calendar.listEvents({ from, to });
  },
});
```

### Gateway

Gateway 是统一执行层。它把工具执行结果标准化，并处理 unknown tool、异常、耗时和日志。

```ts
export interface GatewayResult {
  ok: boolean;
  name: string;
  elapsedMs: number;
  result?: unknown;
  error?: string;
}
```

```ts
export async function callTool(
  name: string,
  args: unknown,
): Promise<GatewayResult> {
  const started = Date.now();
  const tool = registry.get(name);

  if (!tool) {
    return {
      ok: false,
      name,
      elapsedMs: 0,
      error: "unknown_tool",
    };
  }

  try {
    const result = await tool.run(args);
    return {
      ok: true,
      name,
      elapsedMs: Date.now() - started,
      result,
    };
  } catch (error) {
    return {
      ok: false,
      name,
      elapsedMs: Date.now() - started,
      error: error instanceof Error ? error.message : String(error),
    };
  }
}
```

## Entry Modes

不同入口可以使用不同 routing policy。

### Chat

适合完整流程：

```txt
force rules -> LLM router -> selected drawers -> model tools
```

### Voice

适合低延迟：

```txt
force rules -> selected drawers
```

必要时可以给 voice 入口常驻一个小工具集，例如 `device`。

### Scheduled Job

适合固定 drawer set：

```txt
scheduled trigger -> fixed drawers -> model tools
```

例如每日摘要任务可以固定打开 `memory`，不需要 router。

### MCP Wrapper

MCP wrapper 可以把 gateway 里的工具重新暴露给外部客户端。

```txt
MCP request -> gateway call -> normalized result
```

这样内部 chat 和外部 MCP 不需要维护两套工具实现。

## Risk Model

Drawer routing 不是权限系统。它只决定 tool schema 是否进入模型上下文。

真正执行工具之前，应当根据风险等级做 guard：

```ts
if (tool.requiresConfirmation && !confirmedByUser) {
  return {
    ok: false,
    error: "confirmation_required",
  };
}
```

推荐风险等级：

- `read`: 读取信息。
- `write`: 写入本地状态。
- `external`: 对外发送消息或调用外部账号。
- `device`: 控制本地设备。
- `destructive`: 删除、覆盖、付款、不可逆动作。

## Logging

日志应该帮助 debug，但不能泄露用户数据。

推荐：

```json
{
  "tool": "search_memory",
  "drawer": "memory",
  "ok": true,
  "elapsed_ms": 42,
  "argument_keys": ["query"]
}
```

避免：

```json
{
  "query": "full private user message",
  "result": "full private tool result"
}
```

## Testing Strategy

推荐测试：

- force rules 命中正确 drawer。
- LLM router 输出非法 drawer 时会被过滤。
- force + LLM 合并后保持顺序和数量限制。
- selected drawers 能返回正确 tool schemas。
- unknown tool 返回标准错误。
- tool exception 不会让 gateway 崩溃。
- high-risk tool 在未确认时不会执行。

示例：

```ts
test("force rules select memory drawer", () => {
  const selected = forceMatchDrawers("please remember this", rules);
  expect(selected).toEqual(["memory"]);
});
```

## Design Notes

保持 drawer 数量少而稳定。一个项目通常不需要二十个 drawer。

好的 drawer 应该代表「能力域」，不是页面、函数文件或数据库表。

```txt
Good: memory, calendar, messages, device
Avoid: getMemoryById, updateCalendarRow, sendButtonClicked
```

Router prompt 应该短。真正复杂的业务规则应该放在 tool guard 或 application layer，不要塞进 router。

Gateway 应该无模型依赖。这样它可以被 chat、voice、scheduled job 和 MCP wrapper 复用。
