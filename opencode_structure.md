# OpenCode 请求-响应架构

## 概述

OpenCode 是一个 AI 驱动的开发工具，采用 **Effect-TS** 函数式编程框架构建。用户提交一个请求（prompt）后，系统经过 **CLI/Server → Session → Agent/Provider → LLM → Tool 执行 → Response** 的完整流水线处理，最终产出响应。

## 整体数据流

```
用户输入 (TUI/CLI/HTTP)
    │
    ▼
Server Routes ─── Session Routes (Hono HTTP)
    │
    ▼
Session.Service ─── 创建/查找 Session, 存储 Message
    │
    ▼
SessionPrompt.prompt() ─── 创建 User Message, 启动 loop
    │
    ▼
SessionPrompt.loop() ─── 主循环（runLoop）
    │
    ├── 1. 检查是否需要 Compaction
    ├── 2. 解析 Agent 信息
    ├── 3. 构建 System Prompt (SystemPrompt.Service)
    ├── 4. 解析 Tools (ToolRegistry + MCP)
    ├── 5. 调用 LLM.Service.stream()
    │       │
    │       ▼
    │   LLM.run() ─── streamText() (Vercel AI SDK)
    │       │
    │       ▼
    │   Provider 层 ─── 厂商 SDK (Anthropic/OpenAI/Google 等)
    │       │
    │       ▼
    │   LLM Response Stream (text, reasoning, tool_calls)
    │
    ├── 6. SessionProcessor 处理流事件
    │       ├── text-delta → 存储 TextPart
    │       ├── reasoning-delta → 存储 ReasoningPart
    │       ├── tool-call → 执行 Tool → 存储 ToolPart
    │       └── 错误处理
    │
    └── 7. 判断循环条件
            ├── finish="stop"/"end" → break, 返回结果
            ├── 有 tool-calls → 继续循环
            ├── 需要 compaction → 压缩后继续
            └── 超步数 → 注入 MAX_STEPS 提示后结束
```

## 核心组件/模块详解

### 1. 入口层

| 组件 | 文件 | 职责 |
|------|------|------|
| CLI 入口 | `src/index.ts` | yargs 命令分发, 数据库迁移, 日志初始化 |
| TUI 线程 | `src/cli/cmd/tui/thread.ts` | 启动 TUI worker 进程, 建立 RPC 通道 |
| TUI Worker | `src/cli/cmd/tui/worker.ts` | 运行 Server, 转发全局事件, 处理 RPC 请求 |
| Server | `src/server/server.ts` | Hono/Effect-HttpApi 双后端, 路由注册 |
| 实例路由 | `src/server/routes/instance/index.ts` | 注册所有实例级路由 (session/tool/file 等) |
| Session 路由 | `src/server/routes/instance/session.ts` | CRUD 会话, 触发 prompt |

**结构关系**: CLI/TUI → Server → Instance Routes → Session Routes → Session.Service → SessionPrompt

**对 Response 的影响**: 入口层决定了请求的进入方式（CLI/TUI/HTTP API）以及权限校验、会话管理的基础上下文。

---

### 2. Session 层

| 组件 | 文件 | 职责 |
|------|------|------|
| Session.Service | `src/session/session.ts` | 会话 CRUD, Message/Part 存储, Event 发布 |
| SessionPrompt.Service | `src/session/prompt.ts` | prompt() 入口, loop() 主循环, tool 解析/执行 |
| SessionRunState | `src/session/run-state.ts` | 运行状态管理, 并发控制, 取消 |
| SessionStatus | `src/session/status.ts` | 会话状态 (idle/busy) 管理 |
| SessionCompaction | `src/session/compaction.ts` | 上下文窗口压缩, token 溢出处理 |
| SessionProcessor | `src/session/processor.ts` | LLM 流事件处理, 增量写入 Part |
| SessionSummary | `src/session/summary.ts` | 对话摘要生成 |
| SessionRevert | `src/session/revert.ts` | 回滚支持 |
| SessionRetry | `src/session/retry.ts` | 错误重试逻辑 |
| Session/MessageV2 | `src/session/message-v2.ts` | Message 数据模型 (User/Assistant/Tool 等) |
| Session/System | `src/session/system.ts` | 系统提示词构建 |

**核心流程**: `SessionPrompt.prompt()` → 创建 User Message → `SessionPrompt.loop()` → `runLoop()` 主循环

**runLoop 循环逻辑** (src/session/prompt.ts:1400):
1. 获取当前上下文消息 (过滤已 compaction 的)
2. 检查是否有未完成的 subtask/compaction 任务
3. 确定当前 Agent
4. 创建 Assistant Message 占位
5. 解析可用 Tools (内置 + MCP + 插件)
6. 构建 System Prompt (环境 + 指令 + Skill)
7. 调用 `handle.process()` → LLM stream
8. 处理流事件 (Processor)
9. 判断: 完成 → break; 有 tool-call → 继续; 超 token → compaction → 继续

**对 Response 的影响**: Session 是请求的容器。loop 控制着模型调用的轮次(turns)，决定了是否 compaction、是否继续循环、何时终止。Processor 负责将 LLM 的流式输出写入持久化存储。

---

### 3. Agent 层

| 组件 | 文件 | 职责 |
|------|------|------|
| Agent.Service | `src/agent/agent.ts` | Agent 定义/查询, 权限规则, 默认 Agent |
| Agent 提示词 | `src/agent/prompt/*.txt` | 各 Agent (build/plan/explore/scout) 的 system prompt |
| Agent 生成 | `src/agent/generate.txt` | LLM 生成自定义 Agent 的提示词 |

**Agent 定义**: 每个 Agent 包含 name, description, mode (primary/subagent/all), permission ruleset, 可选的 model/prompt/options/steps.

**内置 Agent**: build (默认), plan (计划模式), explore (代码搜索), general (通用子代理), scout (文档/依赖研究), compaction/title/summary (隐藏).

**对 Response 的影响**: Agent 决定了:
- **权限** (可用的 tools)
- **System Prompt** (行为指导)
- **最大循环步数** (steps)
- **模型选择** (agent.model)

---

### 4. Provider (LLM) 层

| 组件 | 文件 | 职责 |
|------|------|------|
| Provider.Service | `src/provider/provider.ts` | 厂商/模型注册, 模型获取, API Key 管理 |
| Provider Transform | `src/provider/transform.ts` | 模型参数转换, providerOptions |
| Provider Schema | `src/provider/schema.ts` | ProviderID, ModelID 类型 |
| LLM.Service | `src/session/llm.ts` | 封装 streamText(), 参数组装, 工具过滤 |
| Auth | `src/auth/index.ts` | 厂商认证 (API Key / OAuth) |
| Env | `src/env/env.ts` | 环境变量映射 |

**支持的厂商**: Anthropic, OpenAI, Google, Azure, AWS Bedrock, Groq, Mistral, Perplexity, xAI, Together AI, DeepInfra, Cerebras, Alibaba Cloud, GitHub Copilot, OpenRouter, GitLab 等 20+ 家.

**LLM.stream() 流程** (src/session/llm.ts:76):
1. 获取 LanguageModel 实例 (通过 AI SDK)
2. 构建 System Prompt (Agent prompt + 环境信息 + 指令 + Skills)
3. 应用模型参数 (temperature, topP, topK, maxOutputTokens)
4. 按 Permission 过滤 Tools
5. 调用 `streamText()` (Vercel AI SDK)
6. 返回事件流 (Event Stream)

**对 Response 的影响**: Provider 直接影响:
- **回复质量/风格** (不同模型的能力差异)
- **速度/延迟** (模型大小、厂商 API 响应时间)
- **成本** (token 计价)
- **可用性** (需 API Key 和网络连通)

---

### 5. Tool 层

| 组件 | 文件 | 职责 |
|------|------|------|
| Tool.Def | `src/tool/tool.ts` | Tool 类型定义 (id, description, parameters, execute) |
| ToolRegistry | `src/tool/registry.ts` | 工具注册/发现, 权限过滤 |
| 内置 Tools | `src/tool/*.ts` | read, write, edit, bash, grep, glob, webfetch 等 |
| Shell | `src/tool/shell/shell.ts` | Shell 命令执行工具 |
| Truncate | `src/tool/truncate.ts` | Tool 输出截断 |
| Task Tool | `src/tool/task.ts` | 子代理调用工具 |
| MCP Service | `src/mcp/*.ts` | MCP 服务器管理, 工具发现 |
| Plugin Service | `src/plugin/*.ts` | 插件系统 (生命周期钩子) |

**Tools 解析** (src/session/prompt.ts:368):
- 从 ToolRegistry 解析内置工具
- 从 MCP Service 解析 MCP 工具
- 从 Plugins 获取自定义工具
- 按 Agent Permission 过滤
- 通过 ProviderTransform 转换 schema 适配模型

**Tool 执行** (src/session/processor.ts:220):
- 收到 tool-call 事件 → 读取 ToolDef → 调用 execute
- 结果写入 ToolPart → 展示在 UI
- 错误时写入 error 状态
- 支持 provider-executed (DWS Agent Platform 等)

**对 Response 的影响**: Tools 是最核心的影响因素:
- **能力边界**: 没有 edit 权限就无法修改文件
- **信息获取**: read/grep/glob 影响模型对代码的理解
- **执行能力**: bash/shell 允许模型操作运行环境
- **子代理**: Task Tool 允许模型调用其他 Agent 并行处理
- **MCP**: 可扩展外部工具集

---

### 6. 事件/总线系统

| 组件 | 文件 | 职责 |
|------|------|------|
| Bus.Service | `src/bus/index.ts` | 本地事件总线, PubSub 模式 |
| GlobalBus | `src/bus/global.ts` | 跨进程全局事件总线 |
| BusEvent | `src/bus/bus-event.ts` | 事件定义工具 |
| SyncEvent | `src/sync/index.ts` | 持久化事件系统 (Event Sourcing) |
| EventV2 | `src/v2/event.ts` | v2 事件系统 (session.next 命名空间) |
| SessionEvent | `src/v2/session-event.ts` | Session 事件定义 (prompted, step, text, tool 等) |

**事件流水线**:
- 本地 Bus: 同进程组件间通信
- GlobalBus: Worker 进程 → UI 进程 (通过 RPC)
- SyncEvent: 持久化到数据库, 支持 projector
- EventV2: 新的 session-level 事件系统 (双写到 SyncEvent)

**对 Response 的影响**: 事件系统负责:
- **状态同步**: UI 实时显示进度 (text delta, reasoning, tool results)
- **持久化**: 所有事件写入数据库
- **副作用**: projector 监听事件更新派生状态

---

### 7. 配置/状态/基础设施

| 组件 | 文件 | 职责 |
|------|------|------|
| Config.Service | `src/config/config.ts` | 配置文件加载/合并/监听 |
| Config 子模块 | `src/config/*.ts` | permission, provider, agent, mcp, lsp, plugin |
| Storage | `src/storage/*.ts` | SQLite 数据库 (Drizzle ORM) |
| Permission | `src/permission/permission.ts` | 权限系统 (allow/ask/deny) |
| Question | `src/question/question.ts` | 用户交互请求 (对话框) |
| InstanceState | `src/effect/instance-state.ts` | Effect 上下文管理 |
| Effect/Runner | `src/effect/runner.ts` | 并发执行器 |

---

## 模块依赖图

```
用户输入
  │
  ▼
┌──────────────┐
│  Server/Routes │ (Hono/Effect-HttpApi)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Session      │
│  .Service     │◄────── Bus (事件总线)
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│  SessionPrompt    │
│  .prompt() → loop │
└──────┬───────────┘
       │
       ├──────────────────────────────────┐
       ▼                                  ▼
┌──────────────┐              ┌──────────────────────┐
│  Agent        │              │  ToolRegistry + MCP   │
│  .Service     │              │  内置/自定义/MCP Tools │
└──────┬───────┘              └──────────────────────┘
       │
       ▼
┌──────────────┐
│  SystemPrompt │─── Skill + Instruction + 环境信息
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  LLM.Service  │
│  streamText() │
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│  Provider 层      │─── AI SDK (Anthropic/OpenAI/Google/...)
│  (厂商 SDK 适配)   │
└──────┬───────┘
       │
       ▼
   LLM Stream Events
       │
       ▼
┌──────────────────┐
│  SessionProcessor  │─── 解析事件, 写入 Part
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  MessageV2/Session │─── 持久化到 SQLite
└──────────────────┘
       │
       ▼
  Response (到 UI/调用者)
```

## 关键设计决策

1. **Effect-TS 函数式架构**: 整个系统基于 Effect-TS 的 `Effect<Success, Error, Requirements>` 模式, 提供强类型、可组合的异步操作管理。

2. **Event Sourcing (v2)**: 系统正在从 CRUD 向 Event Sourcing 迁移。SessionEvent 定义了完整的事件类型 (Prompted, Step.Started/Ended, Text.Delta, Tool.Called/Success/Failed 等)。

3. **Agent 权限模型**: 每个 Agent 携带 Permission Ruleset, 在 Tool 执行层面进行细粒度控制 (allow/ask/deny)。

4. **双通道通讯**: Bus (同步) + SyncEvent (持久化) + GlobalBus (跨进程), 兼顾实时性和持久性。

5. **Compaction 机制**: 当 Token 接近模型上限时, 自动触发上下文压缩 (调用 LLM 总结历史), 从而支持超长对话。

6. **Subtask/子代理**: 通过 Task Tool 支持 Agent 在运行中启动子代理进行并行处理, 结果回收后继续主流程。

7. **Vercel AI SDK 集成**: 使用 `streamText()` 作为标准 LLM 调用接口, 通过各厂商 SDK 适配不同模型, 统一处理流式输出、Tool Call、错误重试等。

## 请求生命周期总结

1. **输入**: 用户通过 TUI/CLI/HTTP 提交 prompt
2. **路由**: Server → SessionRoutes → Session.Service
3. **初始化**: 创建/获取 Session, 创建 User Message (含文本/文件/附件)
4. **主循环**:
   - 获取上下文消息 (过滤 compaction)
   - 解析 Agent (权限/Prompt/模型)
   - 解析 Tools (内置/MCP/插件/权限过滤)
   - 构建 System Prompt (Agent prompt + 环境 + Skills + 指令)
   - 调用 LLM.stream() → streamText()
   - Processor 处理流事件 (text/reasoning/tool)
   - 判断是否需要继续循环
5. **终止**: 模型结束 / 超步数 / 错误 → 返回最终 Assistant Message
6. **响应**: 写入 DB → 事件通知 → UI 更新
