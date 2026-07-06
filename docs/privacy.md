# Privacy Guide

这份文档说明如何从私人 agent 项目中抽取 Tool Drawers 架构，同时避免泄露用户数据、生产配置、真实工具、私密 prompt 或关系设定。

## 总原则

不要把生产项目删一删之后直接发布。

更安全的方式是：

1. 新建一个空仓库。
2. 不带旧 git history。
3. 重新写通用类型、router、gateway 和 examples。
4. 所有示例都使用 synthetic data。
5. 发布前做 secret scan 和人工审查。

## 不要发布的内容

不要发布：

- `.env`、`.env.*`、backup files。
- API keys、tokens、cookies、session ids。
- 真实域名、Webhook URL、internal endpoint。
- 真实数据库、SQLite 文件、向量索引、日志。
- 私人聊天记录、摘要、记忆卡片、persona prompt。
- 真实联系人、账号、群组、社交媒体 handle。
- 生产路径，例如 home directory、server path、browser profile。
- 可直接控制真实设备或外部账号的工具实现。
- 含有私人昵称、仪式、关系设定、生活细节的 tool descriptions。

## 可以发布的内容

可以发布：

- 抽屉路由模式。
- 通用 Drawer / Tool 类型。
- keyword force rules 的写法。
- LLM router 的 JSON 输出约束。
- gateway register/list/call 的接口。
- synthetic examples。
- 测试用 mock tools。
- 脱敏日志格式。
- 风险分级和 confirmation gate 的示例。

## 推荐脱敏流程

### 1. 新仓库重写

使用 clean-room rewrite，而不是复制生产代码。

```bash
mkdir agent-tool-drawers
cd agent-tool-drawers
git init
```

重新写最小实现：

```txt
src/
  drawer.ts
  router.ts
  gateway.ts
examples/
  basic-agent/
docs/
  privacy.md
```

### 2. 替换所有能力域

把私人系统里的能力域换成中性名称。

```ts
const drawers = {
  memory: {
    id: "memory",
    label: "Memory",
    description: "Search and write long-term memories.",
    tools: ["search_memory", "write_memory"],
  },
  device: {
    id: "device",
    label: "Device",
    description: "Control a local companion device.",
    tools: ["speak", "set_expression"],
  },
};
```

不要保留真实工具名、真实服务名或私人世界观名词。

### 3. 替换 prompts

生产 prompt 通常包含最多私密信息。不要发布。

使用最小 router prompt：

```ts
const routerPrompt = `
You are a tool drawer router.
Select at most 2 drawers that may help answer the user's latest message.
Return JSON only: { "drawers": ["drawer_id"] }.
If no tools are needed, return { "drawers": [] }.
`;
```

### 4. 使用 synthetic examples

不要用真实用户消息做 examples。

Use:

```ts
await selectDrawers("Can you check my meetings tomorrow?");
await selectDrawers("Please remember that I prefer quiet reminders.");
await selectDrawers("Can you say that out loud?");
```

Avoid:

```ts
await selectDrawers("real private message copied from production");
```

### 5. 扫描敏感词

基础扫描：

```bash
rg -n "SECRET|TOKEN|PRIVATE|PASSWORD|API_KEY|AUTH|COOKIE|SESSION"
rg -n "localhost|127\\.0\\.0\\.1|/opt/|/home/|\\.env|\\.sqlite|\\.db"
rg -n "telegram|discord|twitter|gmail|calendar|webhook|bearer"
```

如果仓库准备公开，建议再跑 secret scanner：

```bash
gitleaks detect --source .
trufflehog filesystem .
```

这些工具不是必须，但很适合发布前检查。

### 6. 检查 git history

如果曾经把敏感文件 commit 进这个仓库，即使后来删除，也不要直接发布。

更安全的选择是新建干净仓库，把最终脱敏后的文件复制进去，再初始化 git。

```bash
git log --stat
git grep -n "SECRET\\|TOKEN\\|PASSWORD"
```

## 日志建议

不要记录完整用户消息、完整 tool arguments 或完整 tool result。

建议日志只保留：

```json
{
  "tool": "search_memory",
  "drawer": "memory",
  "ok": true,
  "elapsed_ms": 34,
  "argument_keys": ["query"],
  "result_size": 3
}
```

避免：

```json
{
  "query": "private user message",
  "result": "full private memory result"
}
```

## 高风险工具

即使 drawer 被选中，也不代表工具应该立刻执行。

建议给工具加风险等级：

```ts
type ToolRisk = "read" | "write" | "external" | "device" | "destructive";
```

高风险工具应该有 confirmation gate：

```ts
if (tool.requiresConfirmation && !confirmedByUser) {
  return {
    ok: false,
    error: "confirmation_required",
  };
}
```

## 发布前人工清单

发布前逐项确认：

- README 中没有私人项目名、域名、路径。
- examples 中没有真实聊天或真实工具。
- prompts 是通用版本，不含 persona。
- tests 使用 mock data。
- 没有 `.env`、database、logs、browser profile。
- 没有 production endpoint。
- 没有真实账号、联系人、handle。
- git history 是干净的。
- license 和 acknowledgements 写清楚。

## Acknowledgements

如果这个项目受到多个公开教程、文章或示例影响，但无法确认具体来源，可以诚实写：

```md
This project is a clean-room implementation of a common agent architecture pattern.
It was inspired by public tutorials, examples, and discussions around tool routing,
capability routing, router chains, MCP-style tools, and ReAct-style agents.

If you recognize a specific source that should be credited, please open an issue or PR.
```
