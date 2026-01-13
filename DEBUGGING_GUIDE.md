# OpenCode 核心模块调试指南

> 版本: v1.1.6-debug
> 更新时间: 2026-01-13
> 注意: 本指南专注于核心模块的调试，不包含 desktop app 相关内容

## 目录

- [1. 项目架构概览](#1-项目架构概览)
- [2. 核心模块介绍](#2-核心模块介绍)
- [3. 调试工具与命令](#3-调试工具与命令)
- [4. 日志系统详解](#4-日志系统详解)
- [5. 调试技巧与最佳实践](#5-调试技巧与最佳实践)
- [6. 常见调试场景](#6-常见调试场景)
- [7. 性能分析与优化](#7-性能分析与优化)

---

## 1. 项目架构概览

### 1.1 整体架构

OpenCode 是一个基于 Bun 的 monorepo 项目，采用分层架构设计：

```
┌─────────────────────────────────────────────────┐
│              用户交互层                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ CLI(TUI) │  │   Web    │  │  Console │      │
│  │ opencode │  │   app    │  │  平台    │      │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘      │
└────────┼─────────────┼─────────────┼────────────┘
         │             │             │
         ▼             ▼             ▼
┌─────────────────────────────────────────────────┐
│         服务层 (Hono + Bun)                     │
│  ┌──────────────────────────────────────────┐  │
│  │  HTTP/WebSocket Server                   │  │
│  │  - REST API (100+ 路由)                  │  │
│  │  - SSE 事件流                             │  │
│  │  - WebSocket (PTY)                        │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│              核心业务逻辑层                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Session  │  │  Agent   │  │  Plugin  │      │
│  │ Manager  │  │  System  │  │  System  │      │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘      │
│        │             │             │            │
│  ┌─────┴─────────────┴─────────────┴─────┐     │
│  │        Tool Registry & Execution       │     │
│  │  (Bash, Edit, Read, Write, Grep, 等)  │     │
│  └──────────────────┬─────────────────────┘     │
└─────────────────────┼───────────────────────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
    ┌────────┐  ┌─────────┐  ┌─────────┐
    │Provider│  │   MCP   │  │ Storage │
    │  APIs  │  │ Servers │  │  Layer  │
    └────────┘  └─────────┘  └─────────┘
```

### 1.2 技术栈

- **运行时**: Bun (JavaScript/TypeScript)
- **Web 框架**: Hono (轻量级 HTTP 服务器)
- **前端框架**: SolidJS (响应式 UI)
- **AI SDK**: Vercel AI SDK + 多提供商集成
- **构建工具**: Turbo (monorepo 构建)
- **Schema 验证**: Zod
- **MCP 协议**: Model Context Protocol SDK

---

## 2. 核心模块介绍

### 2.1 主包：opencode (`packages/opencode`)

**职责**: 核心 CLI 应用和服务器实现

**关键目录结构**:
```
src/
├── agent/          # Agent 系统 (build, plan, general, explore, compaction)
├── cli/            # CLI 命令和 TUI 界面
├── server/         # Hono HTTP 服务器 + WebSocket
├── session/        # 会话管理、消息处理、压缩
├── tool/           # 工具注册表 (Bash, Edit, Read, Write, Grep, Glob, 等)
├── plugin/         # 插件系统
├── mcp/            # Model Context Protocol 集成
├── provider/       # AI 提供商集成
├── project/        # 项目和工作树管理
├── storage/        # 数据持久化层
├── pty/            # 伪终端管理
├── lsp/            # Language Server Protocol
├── permission/     # 权限系统
├── bus/            # 事件总线
├── config/         # 配置管理
└── util/           # 工具函数
```

**入口点**:
- **CLI**: `src/index.ts` (Yargs 命令解析器)
- **Binary**: `bin/opencode` (平台特定二进制分发器)
- **Server**: `src/server/server.ts` (Hono 应用)

**可用命令**:
```bash
opencode run      # 执行 AI 命令
opencode serve    # 启动 headless 服务器
opencode auth     # 认证管理
opencode agent    # Agent 操作
opencode mcp      # MCP 服务器管理
opencode github   # GitHub 集成
opencode debug    # 调试命令集
opencode stats    # 统计信息
opencode export   # 导出数据
opencode import   # 导入数据
```

### 2.2 SDK (`packages/sdk/js`)

**职责**: 提供客户端和服务端 SDK，用于编程式访问 OpenCode

**关键文件**:
- `src/index.ts` - 主导出 (客户端 + 服务端)
- `src/client.ts` - 客户端 SDK (`createOpencodeClient`)
- `src/server.ts` - 服务端 SDK (`createOpencodeServer`, `createOpencodeTui`)
- `gen/` - 生成的类型和 OpenAPI 客户端

**导出**:
- `.` - 完整 SDK
- `./client` - 仅客户端
- `./server` - 仅服务端
- `./v2` - V2 API 端点

**使用场景**: 允许外部工具生成 OpenCode 服务器并进行编程式交互

### 2.3 Plugin 系统 (`packages/plugin`)

**职责**: 插件接口和钩子系统，用于扩展 OpenCode 功能

**关键文件**:
- `src/index.ts` - 插件类型和钩子定义
- `src/tool.ts` - 工具定义接口
- `src/shell.ts` - BunShell 包装器

**钩子系统**:
```typescript
event               # 监听系统事件
config              # 响应配置变更
tool                # 注册自定义工具
auth                # 自定义认证方法
chat.message        # 拦截消息
chat.params         # 修改 LLM 参数
permission.ask      # 权限处理
tool.execute.before # 工具执行前钩子
tool.execute.after  # 工具执行后钩子
```

### 2.4 Function (`packages/function`)

**职责**: Cloudflare Workers/Durable Objects，用于无服务器会话共享和同步

**关键组件**:
- `src/api.ts` - SyncServer Durable Object (实时会话同步)
- GitHub App 集成
- 通过 WebSocket 实现会话共享
- R2 bucket 存储共享会话

### 2.5 Util (`packages/util`)

**职责**: 共享工具函数库

**模块**:
- `binary.ts` - 二进制工具
- `encode.ts` - 编码助手
- `error.ts` - 错误处理
- `identifier.ts` - ID 生成
- `path.ts` - 路径工具
- `retry.ts` - 重试逻辑

### 2.6 App (`packages/app`)

**职责**: SolidJS 前端 UI

**技术栈**:
- SolidJS + Vite + TailwindCSS
- Kobalte (UI 组件库)
- Ghostty 终端 (Web 终端模拟器)
- Marked + Shiki (Markdown 渲染和语法高亮)

**关键目录**:
- `src/app.tsx` - 主应用组件
- `components/` - UI 组件
- `pages/` - 应用页面
- `context/` - 上下文提供者
- `hooks/` - 自定义钩子

### 2.7 UI (`packages/ui`)

**职责**: 可复用 UI 组件库

**导出**:
- `./*` - 组件 (tsx 文件)
- `./pierre` - Pierre diff 库
- `./hooks` - 共享钩子
- `./context` - 上下文提供者
- `./styles` - CSS 样式
- `./theme` - 主题系统

### 2.8 Console (`packages/console`)

**职责**: 后端 SaaS 平台 (多包结构)

**子包**:
- **console/core** - 核心后端逻辑
  - Drizzle ORM (PlanetScale/Postgres)
  - 用户管理、计费 (Stripe)、工作区、模型
- **console/app** - SolidJS Web 应用
- **console/function** - 无服务器函数
- **console/mail** - 邮件模板和发送
- **console/resource** - 共享资源

### 2.9 Script (`packages/script`)

**职责**: 构建时工具

**功能**:
- 版本计算 (基于 git 分支、npm registry)
- 频道管理 (latest vs preview)
- Bun 版本强制

### 2.10 Web (`packages/web`)

**职责**: Astro 文档和营销网站

**技术栈**: Astro + @astrojs/solid-js + @astrojs/starlight

---

## 3. 调试工具与命令

### 3.1 debug 命令集

OpenCode 提供了一套专门的调试命令，位于 `opencode debug` 子命令下：

```bash
# 查看所有调试命令
opencode debug --help

# 调试配置
opencode debug config

# 调试 LSP (Language Server Protocol)
opencode debug lsp

# 调试 Ripgrep 搜索
opencode debug ripgrep

# 调试文件操作
opencode debug file

# 调试 scrap (临时文件)
opencode debug scrap

# 调试 skill (技能系统)
opencode debug skill

# 调试 snapshot (快照)
opencode debug snapshot

# 调试 agent (Agent 系统)
opencode debug agent

# 查看路径信息
opencode debug paths

# 等待命令 (用于长时间运行调试)
opencode debug wait
```

**调试命令源码位置**: `packages/opencode/src/cli/cmd/debug/`

### 3.2 开发模式运行

```bash
# 在 opencode 包目录下运行开发模式
cd packages/opencode
bun run dev

# 从根目录运行
bun run dev
```

**配置**: 开发脚本位于 `package.json` 的 `scripts.dev`

### 3.3 启动调试服务器

```bash
# 启动 headless 服务器 (默认端口 4096)
opencode serve

# 带日志输出
opencode serve --print

# 指定日志级别
opencode serve --log-level DEBUG
```

### 3.4 TypeScript 类型检查

```bash
# 运行类型检查
bun run typecheck

# 在根目录运行所有包的类型检查
bun turbo typecheck
```

---

## 4. 日志系统详解

### 4.1 日志架构

OpenCode 使用自定义的日志系统，位于 `packages/opencode/src/util/log.ts`。

**特性**:
- 结构化日志
- 多级别日志 (DEBUG, INFO, WARN, ERROR)
- 自动日志文件轮转
- 服务标签系统
- 性能计时器

### 4.2 日志级别

```typescript
export const Level = z.enum(["DEBUG", "INFO", "WARN", "ERROR"])

// 级别优先级
const levelPriority: Record<Level, number> = {
  DEBUG: 0,
  INFO: 1,
  WARN: 2,
  ERROR: 3,
}
```

**设置日志级别**:

```bash
# 方法 1: 通过命令行参数
opencode serve --log-level DEBUG

# 方法 2: 通过环境变量 (需要查看代码确认)
LOG_LEVEL=DEBUG opencode run "your command"
```

### 4.3 日志位置

日志文件存储在全局日志目录中：

```bash
# 查看日志路径
opencode debug paths

# 典型路径 (Linux/macOS)
~/.local/share/opencode/logs/

# 日志文件命名格式
# 开发模式: dev.log
# 生产模式: 2026-01-13T120530.log (ISO 8601 格式)
```

**日志清理策略**:
- 自动保留最近 10 个日志文件
- 启动时自动清理旧日志

### 4.4 使用日志系统

**在代码中创建 Logger**:

```typescript
import { Log } from "@/util/log"

// 创建带服务标签的 logger
const log = Log.create({ service: "my-service" })

// 使用 logger
log.info("操作开始", { userId: "123", action: "create" })
log.error("操作失败", { error: err, context: "database" })
log.debug("调试信息", { data: complexObject })
log.warn("警告", { reason: "deprecated API" })

// 性能计时
const timer = log.time("expensive-operation")
// ... 执行操作 ...
timer.stop()

// 或者使用 dispose (自动停止)
{
  using timer = log.time("auto-stop-operation")
  // ... 执行操作 ...
} // timer 自动停止
```

**Logger 方法**:

| 方法 | 描述 |
|------|------|
| `debug(message, extra?)` | 调试级别日志 |
| `info(message, extra?)` | 信息级别日志 |
| `warn(message, extra?)` | 警告级别日志 |
| `error(message, extra?)` | 错误级别日志 |
| `tag(key, value)` | 添加标签到 logger |
| `clone()` | 克隆 logger |
| `time(message, extra?)` | 创建计时器 |

### 4.5 日志格式

```
[日志级别] [ISO时间戳] +[距上次日志毫秒数] [标签=值] [消息]

示例:
INFO  2026-01-13T12:05:30 +125ms service=server method=GET path=/session 请求开始
ERROR 2026-01-13T12:05:35 +5234ms service=database error=Connection timeout 操作失败
```

### 4.6 查看日志

```bash
# 实时查看日志
tail -f ~/.local/share/opencode/logs/dev.log

# 查看最近的日志
tail -100 ~/.local/share/opencode/logs/dev.log

# 搜索特定服务的日志
grep "service=server" ~/.local/share/opencode/logs/dev.log

# 搜索错误日志
grep "ERROR" ~/.local/share/opencode/logs/dev.log
```

---

## 5. 调试技巧与最佳实践

### 5.1 使用 Bun 的内置调试工具

**运行测试**:
```bash
cd packages/opencode
bun test

# 运行特定测试
bun test src/session/session.test.ts

# 带覆盖率
bun test --coverage
```

**使用 Bun 的调试器**:
```bash
# 以调试模式运行
bun --inspect src/index.ts run "your command"

# 然后在 Chrome 中打开 chrome://inspect
```

### 5.2 调试服务器 API

**使用 curl 测试 API**:

```bash
# 获取健康检查
curl http://localhost:4096/global/health

# 列出会话
curl http://localhost:4096/session

# 获取特定会话
curl http://localhost:4096/session/<session-id>

# 发送消息
curl -X POST http://localhost:4096/session/<session-id>/message \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello"}'
```

**查看 API 规范**:

服务器实现了 OpenAPI 规范，可以通过以下方式查看：
```typescript
// 在 server.ts 中
import { generateSpecs } from "hono-openapi"
```

### 5.3 调试 Session 和 Message

**Session 架构**:

```
Session
  ├── messages[]        # 消息列表
  ├── compaction       # 压缩状态
  ├── summary          # 摘要
  ├── status           # 状态
  └── prompt           # 提示词
```

**调试 Session**:

```bash
# 查看会话统计
opencode stats

# 导出会话
opencode export <session-id>

# 导入会话
opencode import <file>

# 查看会话详情
opencode session <session-id>
```

**在代码中调试**:

```typescript
import { Session } from "@/session"
import { Storage } from "@/storage/storage"

// 读取会话
const sessionData = await Storage.read(["session", projectID, sessionID])

// 调试消息
const messages = await Session.Message.list(projectID, sessionID)
messages.forEach(msg => {
  console.log("Message:", msg.role, msg.content)
})
```

### 5.4 调试 Agent 系统

**Agent 类型**:
- `build` - 构建验证
- `plan` - 任务规划
- `general` - 通用任务
- `explore` - 代码探索
- `compaction` - 消息压缩

**调试 Agent**:

```bash
# 查看 Agent 配置
opencode debug agent

# 列出可用 Agent
opencode agent list

# 运行特定 Agent
opencode run --agent explore "分析项目结构"
```

**在代码中调试**:

```typescript
import { Agent } from "@/agent/agent"

// 获取 Agent 定义
const agent = Agent.get("explore")
console.log("Agent config:", agent)

// 调试 Agent 提示词
console.log("System prompt:", agent.systemPrompt)
console.log("Permissions:", agent.permissions)
```

### 5.5 调试 Tool 执行

**Tool Registry**:

所有工具都在 `src/tool/registry.ts` 中注册。

**调试 Tool**:

```typescript
import { ToolRegistry } from "@/tool/registry"

// 列出所有工具
const tools = ToolRegistry.list()
console.log("Available tools:", tools.map(t => t.name))

// 获取特定工具
const bashTool = ToolRegistry.get("Bash")
console.log("Bash tool schema:", bashTool.parameters)
```

**添加日志到 Tool 执行**:

在 `server.ts` 中，工具执行通过钩子系统处理：
```typescript
// tool.execute.before hook
// tool.execute.after hook
```

### 5.6 调试 Plugin 系统

**加载 Plugin**:

```bash
# 查看已加载的插件
opencode mcp list

# 添加插件
opencode mcp add <name> <command>

# 移除插件
opencode mcp remove <name>
```

**调试 Plugin 钩子**:

```typescript
// 在插件代码中
export default plugin({
  name: "my-plugin",
  hooks: {
    "tool.execute.before": async (context) => {
      console.log("Tool about to execute:", context.tool)
      return context
    },
    "tool.execute.after": async (context) => {
      console.log("Tool executed:", context.result)
      return context
    }
  }
})
```

### 5.7 调试权限系统

**Permission 架构**:

```typescript
// PermissionNext 规则
type Rule = {
  type: "allow" | "deny" | "ask"
  pattern?: string  // Glob 模式
  tool?: string     // 工具名称
}
```

**调试权限**:

```bash
# 查看当前权限配置
opencode debug config

# 在代码中
import { PermissionNext } from "@/permission/next"

// 检查权限
const canExecute = await PermissionNext.check({
  tool: "Bash",
  parameters: { command: "ls" }
})
console.log("Permission:", canExecute)
```

### 5.8 调试 Storage 层

**Storage 架构**:

```typescript
// 存储键结构
type StorageKey =
  | ["session", projectID, sessionID]
  | ["message", projectID, sessionID, messageID]
  | ["config", projectID]
  | ["snapshot", projectID, snapshotID]
```

**调试 Storage**:

```typescript
import { Storage } from "@/storage/storage"

// 读取数据
const data = await Storage.read(["session", projectID, sessionID])
console.log("Session data:", data)

// 列出所有会话
const sessions = await Storage.list(["session", projectID])
console.log("Sessions:", sessions)

// 写入数据
await Storage.write(["session", projectID, sessionID], newData)
```

### 5.9 调试事件总线

**Bus 系统**:

```typescript
import { Bus } from "@/bus"
import { GlobalBus } from "@/bus/global"

// 订阅事件
Bus.on(Session.Event.MessageCreated, (event) => {
  console.log("Message created:", event.data)
})

// 发布事件
Bus.emit(Session.Event.MessageCreated, {
  sessionID,
  messageID,
  content: "..."
})

// 全局事件
GlobalBus.on(Server.Event.Connected, () => {
  console.log("Server connected")
})
```

### 5.10 性能分析

**使用 Log.time**:

```typescript
import { Log } from "@/util/log"
const log = Log.create({ service: "my-service" })

// 方法 1: 手动停止
const timer = log.time("database-query")
const result = await database.query()
timer.stop()

// 方法 2: 自动停止
{
  using timer = log.time("auto-timer")
  await expensiveOperation()
} // 自动调用 timer.stop()
```

**查看性能日志**:

```bash
grep "duration=" ~/.local/share/opencode/logs/dev.log
```

---

## 6. 常见调试场景

### 6.1 调试 AI 提供商问题

**场景**: AI 请求失败或返回错误

**调试步骤**:

1. **检查认证**:
```bash
opencode auth list
opencode auth add <provider>
```

2. **查看提供商配置**:
```bash
opencode debug config
```

3. **查看日志中的提供商错误**:
```bash
grep "service=provider" ~/.local/share/opencode/logs/dev.log
grep "ERROR.*provider" ~/.local/share/opencode/logs/dev.log
```

4. **在代码中添加调试**:
```typescript
// packages/opencode/src/provider/provider.ts
import { Log } from "@/util/log"
const log = Log.create({ service: "provider" })

log.debug("Provider request", {
  provider: providerName,
  model: modelName,
  messages: messages.length
})
```

### 6.2 调试会话状态问题

**场景**: 会话状态不一致或丢失

**调试步骤**:

1. **检查 Storage**:
```typescript
import { Storage } from "@/storage/storage"

const sessionData = await Storage.read(["session", projectID, sessionID])
console.log("Session state:", JSON.stringify(sessionData, null, 2))
```

2. **检查消息列表**:
```bash
opencode export <session-id> > session.json
cat session.json | jq '.messages'
```

3. **查看压缩状态**:
```typescript
import { SessionCompaction } from "@/session/compaction"

const compaction = await SessionCompaction.get(projectID, sessionID)
console.log("Compaction:", compaction)
```

4. **检查事件流**:
```bash
# 监听 SSE 事件
curl -N http://localhost:4096/global/event
```

### 6.3 调试工具执行问题

**场景**: 工具执行失败或行为异常

**调试步骤**:

1. **检查工具注册**:
```typescript
import { ToolRegistry } from "@/tool/registry"

const tool = ToolRegistry.get("Bash")
console.log("Tool schema:", tool.parameters)
console.log("Tool function:", tool.execute)
```

2. **添加工具执行日志**:
```typescript
// 在工具实现中添加日志
import { Log } from "@/util/log"
const log = Log.create({ service: "tool", tool: "Bash" })

log.info("Tool executing", {
  command: parameters.command,
  cwd: process.cwd()
})
```

3. **检查权限**:
```typescript
import { PermissionNext } from "@/permission/next"

const permission = await PermissionNext.check({
  tool: "Bash",
  parameters: { command: "ls" }
})
console.log("Permission result:", permission)
```

4. **查看执行历史**:
```bash
grep "tool=" ~/.local/share/opencode/logs/dev.log | tail -50
```

### 6.4 调试 MCP 服务器问题

**场景**: MCP 服务器连接失败或工具不可用

**调试步骤**:

1. **列出 MCP 服务器**:
```bash
opencode mcp list
```

2. **测试 MCP 连接**:
```typescript
import { MCP } from "@/mcp"

const servers = await MCP.list(projectID)
for (const server of servers) {
  console.log("MCP Server:", server.name)
  const tools = await MCP.tools(projectID, server.name)
  console.log("Available tools:", tools.map(t => t.name))
}
```

3. **查看 MCP 日志**:
```bash
grep "service=mcp" ~/.local/share/opencode/logs/dev.log
```

4. **手动启动 MCP 服务器**:
```bash
# 查看 MCP 配置
opencode debug config | grep mcp

# 手动运行 MCP 服务器命令进行测试
<mcp-command>
```

### 6.5 调试消息压缩问题

**场景**: 消息压缩失败或丢失信息

**调试步骤**:

1. **检查压缩配置**:
```typescript
import { SessionCompaction } from "@/session/compaction"

const compaction = await SessionCompaction.get(projectID, sessionID)
console.log("Last compaction:", compaction.lastCompactedAt)
console.log("Message count:", compaction.messageCount)
```

2. **手动触发压缩**:
```typescript
await SessionCompaction.compact(projectID, sessionID)
```

3. **查看压缩日志**:
```bash
grep "service=compaction" ~/.local/share/opencode/logs/dev.log
```

4. **调试 compaction agent**:
```bash
opencode debug agent
```

### 6.6 调试文件操作问题

**场景**: 文件读写失败或路径问题

**调试步骤**:

1. **检查项目根目录**:
```typescript
import { Instance } from "@/project/instance"

const project = Instance.state().project
console.log("Project root:", project.root)
console.log("Current working directory:", process.cwd())
```

2. **调试文件路径**:
```bash
opencode debug paths
```

3. **检查文件权限**:
```bash
ls -la <file-path>
stat <file-path>
```

4. **查看文件操作日志**:
```bash
grep "service=file" ~/.local/share/opencode/logs/dev.log
```

### 6.7 调试 WebSocket/PTY 问题

**场景**: 终端连接失败或输入输出异常

**调试步骤**:

1. **检查 PTY 状态**:
```typescript
import { Pty } from "@/pty"

const pty = await Pty.get(ptyID)
console.log("PTY status:", pty.status)
console.log("PTY command:", pty.command)
```

2. **查看 WebSocket 连接**:
```bash
# 使用 wscat 测试
npm install -g wscat
wscat -c ws://localhost:4096/pty/<pty-id>
```

3. **检查 PTY 日志**:
```bash
grep "service=pty" ~/.local/share/opencode/logs/dev.log
```

### 6.8 调试构建和类型错误

**场景**: TypeScript 类型错误或构建失败

**调试步骤**:

1. **运行类型检查**:
```bash
cd packages/opencode
bun run typecheck
```

2. **检查依赖**:
```bash
bun install
```

3. **清理并重新构建**:
```bash
bun run clean
bun run build
```

4. **查看构建日志**:
```bash
bun run build 2>&1 | tee build.log
```

---

## 7. 性能分析与优化

### 7.1 识别性能瓶颈

**使用日志计时器**:

```typescript
import { Log } from "@/util/log"
const log = Log.create({ service: "performance" })

// 关键操作计时
{
  using timer = log.time("message-processing")
  await processMessage(message)
}

{
  using timer = log.time("tool-execution")
  await executeTool(tool, parameters)
}

{
  using timer = log.time("ai-request")
  const response = await provider.generateText(prompt)
}
```

**分析日志性能数据**:

```bash
# 查找慢操作
grep "duration=" ~/.local/share/opencode/logs/dev.log | \
  awk '{for(i=1;i<=NF;i++){if($i~/duration=/){print $i}}}' | \
  sort -t= -k2 -n | \
  tail -20

# 统计各操作平均耗时
grep "duration=" ~/.local/share/opencode/logs/dev.log | \
  awk '{
    for(i=1;i<=NF;i++){
      if($i~/operation=/) op=$i;
      if($i~/duration=/) dur=$i;
    }
    gsub(/.*=/,"",op); gsub(/.*=/,"",dur);
    sum[op]+=dur; cnt[op]++;
  }
  END {
    for(op in sum) printf "%s: avg=%.2fms count=%d\n", op, sum[op]/cnt[op], cnt[op]
  }'
```

### 7.2 优化建议

**1. 消息压缩优化**

- 定期运行消息压缩
- 避免长会话无压缩运行
- 监控压缩效果

**2. 缓存策略**

```typescript
// 使用 lazy 缓存
import { lazy } from "@/util/lazy"

const expensiveData = lazy(() => {
  // 仅在首次访问时计算
  return computeExpensiveData()
})
```

**3. 批量操作**

```typescript
// 避免循环中的单个操作
// ❌ 不好
for (const file of files) {
  await processFile(file)
}

// ✅ 更好
await Promise.all(files.map(processFile))
```

**4. 工具执行优化**

- 使用并行工具调用
- 避免重复的文件读取
- 缓存频繁访问的数据

### 7.3 内存分析

**使用 Bun 的内存分析**:

```bash
# 运行时内存快照
bun --heap-snapshot src/index.ts

# 分析堆快照
# 在 Chrome DevTools 中加载 .heapsnapshot 文件
```

**监控内存使用**:

```typescript
import { Log } from "@/util/log"
const log = Log.create({ service: "memory" })

// 定期记录内存使用
setInterval(() => {
  const usage = process.memoryUsage()
  log.info("Memory usage", {
    rss: (usage.rss / 1024 / 1024).toFixed(2) + "MB",
    heapUsed: (usage.heapUsed / 1024 / 1024).toFixed(2) + "MB",
    heapTotal: (usage.heapTotal / 1024 / 1024).toFixed(2) + "MB"
  })
}, 60000) // 每分钟
```

### 7.4 网络请求优化

**监控 AI 提供商请求**:

```typescript
import { Log } from "@/util/log"
const log = Log.create({ service: "provider" })

// 记录请求详情
log.info("AI request", {
  provider: providerName,
  model: modelName,
  tokens: inputTokens,
  streaming: isStreaming
})

// 记录响应详情
log.info("AI response", {
  provider: providerName,
  duration: elapsedMs,
  tokensUsed: totalTokens,
  cost: estimatedCost
})
```

---

## 总结

本指南涵盖了 OpenCode 核心模块的调试方法，包括：

1. **架构理解**: 清晰的模块分层和职责划分
2. **调试工具**: 完整的 debug 命令集和日志系统
3. **实用技巧**: 针对不同场景的调试方法
4. **性能优化**: 识别和解决性能瓶颈

**关键调试资源**:

- 日志文件: `~/.local/share/opencode/logs/`
- Debug 命令: `opencode debug <subcommand>`
- 源码位置: `packages/opencode/src/`
- API 端点: `http://localhost:4096`

**下一步**:

1. 熟悉日志系统和 debug 命令
2. 在关键模块添加调试日志
3. 使用性能计时器识别瓶颈
4. 参考本指南解决具体问题

**贡献**:

如果发现新的调试技巧或改进建议，欢迎更新本文档！

---

**文档维护**:
- 作者: OpenCode Team
- 版本: v1.1.6-debug
- 最后更新: 2026-01-13
