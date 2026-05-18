# EvoBench: LLM Harness 工程自进化方案

---

## 目录

1. [总体架构](#1-总体架构)
2. [Phase 1: 评估基准 — AI Benchmark (Oracle)](#2-phase-1-评估-baseline—ai-benchmark-oracle)
3. [Phase 2: 组件级短板定位 — 深度诊断系统](#3-phase-2-组件级短板定位--深度诊断系统)
4. [Phase 3: 数据驱动的自进化 — 靶向优化引擎](#4-phase-3-数据驱动的自进化--靶向优化引擎)
5. [策略选型综合对比](#5-策略选型综合对比)
6. [实施路线图](#6-实施路线图)
7. [风险与应对](#7-风险与应对)

---

## 1. 总体架构

EvoBench 的核心逻辑：以 **AI Benchmark** 作为标尺（Orade），以 **Observability & Attribution** 实现组件级归因，以 **Self-Evolution Engine** 驱动自动优化，形成完整闭环。

```
┌─────────────────────────────────────────────────────────────────────┐
│                      EvoBench 总架构                                 │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Phase 1: AI Benchmark                      │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────────────┐   │   │
│  │  │  Task   │  │ Ground  │  │ Rubric  │  │   测试套件      │   │   │
│  │  │  Dataset │  │ Truth   │  │ Engine  │  │   (Sandbox)    │   │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └───────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │      Phase 2: 组件级短板定位 & 深度归因                       │   │
│  │                                                                 │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐                 │   │
│  │  │ Harness  │───►│ 全链路    │───►│  Shadow   │                 │   │
│  │  │ 分解模型  │    │ Tracking │    │ Eval     │                 │   │
│  │  └──────────┘    └──────────┘    └──────────┘                 │   │
│  │                                        │                       │   │
│  │                                        ▼                       │   │
│  │                               ┌──────────────────┐             │   │
│  │                               │  归因分析引擎     │             │   │
│  │                               │ (多重策略并联)    │             │   │
│  │                               └──────────────────┘             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │        Phase 3: 自进化引擎                                     │   │
│  │                                                                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │   │
│  │  │ Prompt/文本   │  │ 知识库/检索   │  │ Skill/MCP    │         │   │
│  │  │ 优化器        │  │ 优化器        │  │ 代码优化器   │         │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │   │
│  │  │ Agent配置    │  │ 记忆/宪法     │  │ 元学习/跨任务  │         │   │
│  │  │ 优化器        │  │ 优化器        │  │ 迁移优化器   │         │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│                    ┌────────────────────┐                            │
│                    │  验证 & 回测        │                            │
│                    │  (全量回归测试)      │                            │
│                    └────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.1 核心闭环流程

```
Epoch N Harness (当前版本)
        │
        ▼
┌─────────────────────────────────┐
│  全量 AI Benchmark 评测          │  ← 端到端评分
│  (T1~T6, 100+ tasks)            │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  全链路 Trace 收集              │  ← 每个组件 I/O + 耗时
│  + Shadow Evaluation            │  ← 中间输出质量打分
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  归因分析引擎 (多策略融合)        │  ← 定位短板组件
│  → 输出: {component: fault_prob}│
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  靶向优化器 (按组件类型分派)      │  ← 执行自动优化
│  → 输出: 优化后的 Harness v{N+1}│
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  验证回测 (全量回归)             │  ← 确认提升, 无语义退化
│  = 回到第一步                   │
└─────────────────────────────────┘
```

---

## 2. Phase 1: 评估 Baseline — AI Benchmark (Oracle)

### 2.1 定位

AI Benchmark 是 EvoBench 的"测量标尺"，详见独立文档 `AI_benchmark_design.md`。本节仅概述其在自进化方案中的角色和接口。

### 2.2 与 EvoBench 的接口

```
Benchmark 输出 (每次评测后):
{
  "overall_score": 72.5,                    // 综合得分
  "capability_scores": {                     // 五/六维能力得分
    "T1_knowledge": 68.0,
    "T2_requirement": 75.3,
    "T3_impact": 70.1,
    "T4_design": 71.8,
    "T5_implementation": 76.0,
    "T6_evolution": 65.0
  },
  "per_task_scores": [                       // 每个任务的细粒度评分
    { "task_id": "feat_0042_T2", "score": 82, "subscores": {...} },
    { "task_id": "feat_0042_T3", "score": 65, "subscores": {...} },
    ...
  ],
  "trace_data": "s3://traces/eval_20260501.jsonl",  // Trace 数据引用
  "harness_version": "v1.2.3",               // 被评测 Harness 版本
  "evaluation_timestamp": "2026-05-01T08:00:00Z"
}
```

### 2.3 作为 Oracle 的关键要求

| 要求 | 说明 | 对自进化的影响 |
|------|------|---------------|
| **稳定性** | 同一 Harness 两次评测得分偏差 < 2% | 确保优化效果可检测, 而非噪声 |
| **区分度** | 不同能力维度得分应有足够方差 | 归因分析依赖维度间的差异信号 |
| **粒度** | 每个 task 有 subscore, 而非仅总分 | 细粒度归因需要 task 级信号 |
| **可复现** | 评测环境完全容器化 | 优化前后的 A/B 比较必须公平 |
| **灵敏度** | 对 Harness 组件变化敏感 | 如果所有改动得分不变, 无法驱动进化 |

---

## 3. Phase 2: 组件级短板定位 — 深度诊断系统

**这是 EvoBench 的核心部分, 解决"AI Benchmark 只能评估端到端结果, 无法定位具体短板"的根本问题。**

### 3.1 Harness 原子化分解模型

首先需要将 Harness 工程分解为可观测、可归因的原子组件。不同的分解粒度对应不同的归因精度。

#### 3.1.1 Opencode 组件分解模型

Opencode 的架构与通用 Harness 模型有显著差异, 本节基于对 opencode 源代码的实际分析给出精确映射。

**Opencode 核心请求处理流程**:

```
CLI / Editor (入口)
    │
    ▼
┌──────────────────────────────────────────────┐
│  CLI / Server Layer                           │
│  - yargs 命令行解析/editor API entry          │
│  - SessionService: 会话创建/恢复               │
│  - createUserMessage(): 消息组装, 附件解析     │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  主 Agent 循环 (runLoop)                      │
│                                                │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │ SessionRunState │  │ MessageV2 消息管理   │   │
│  │  并发控制/状态管理│  │  - 历史裁剪/恢复     │   │
│  └──────────────┘  │  - 消息→模型消息转换   │   │
│                     └──────────────────────┘   │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Agent.Service │  │  SessionCompaction   │   │
│  │  子Agent调度   │  │  上下文窗口管理      │   │
│  │  tool: task    │  │  (溢出→摘要→替换)    │   │
│  └──────────────┘  └──────────────────────┘   │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  SessionProcessor / SessionLLM (LLM 交互层)    │
│                                                │
│  ├── System Prompt 构造:                       │
│  │    agent.prompt + environment + skills +   │
│  │    instructions → 插件 hook (transform)    │
│  │                                              │
│  ├── Tool 收集与过滤:                          │
│  │    ToolRegistry (内置) + MCP (外部) →       │
│  │    按 agent 权限过滤 → AI SDK Tool 格式     │
│  │                                              │
│  ├── AI SDK streamText(model, messages, tools) │
│  │                                              │
│  └── 事件流处理:                               │
│       text | reasoning | tool-input |          │
│       tool-call | tool-result | tool-error     │
│       step-start | step-finish                 │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  LLM Provider (packages/llm)                  │
│  Protocol→Auth→Framing→Transport→Stream       │
└──────────────────────────────────────────────┘
```

##### 通用组件 → Opencode 实际组件映射

| 通用组件 (原模型) | Opencode 实际组件 | 映射说明 |
|---|---|---|
| **Request 解析器** | `createUserMessage()` (session/prompt.ts) | 解析用户输入, 解析 `@agent` 引用为 subtask, 处理文件/MCP resource attachments |
| **记忆 Manager** | `Session` 消息历史 + `SessionCompaction` (session/compaction.ts) | 基于消息列表而非向量记忆, 通过 compaction 管理上下文窗口 |
| **知识库 Retriever** | **无独立组件** — 代之以 `read`/`grep`/`glob` 工具直接操作文件系统 | Opencode 无传统 RAG, 知识获取通过 Agent 自行决策使用文件系统工具 |
| **Skill Router** | `Agent.Service` + `ToolRegistry` (agent/ + tool/registry.ts) | Agent 决定使用哪些工具, 而非独立的路由器 |
| **宪法 (Constitution)** | `Permission.Service` + 各 Agent `permission` 配置 (permission/) | 基于 permission ruleset 做 allow/deny/ask, 非约束性 prompt 过滤 |
| **Agent Planner** | **无独立 Planner** — 由 LLM 自身通过 tool calls 在上下文中规划 | runLoop 本身是执行循环, 规划内化于 LLM 的推理链中 |
| **工具 Executor** | `Tool.execute()` + `MCP.callTool()` (tool/ + mcp/) | 内置工具 + MCP 工具 + Plugin hook |
| **LLM Call** | `SessionLLM.stream()` → AI SDK `streamText()` → `packages/llm` | 四层抽象: Protocol → Auth → Framing → Transport |
| **输出合成** | `SessionProcessor` 事件流处理 (session/processor.ts) | 将 LLM stream 事件翻译为 session message parts |
| **—** | **`Plugin` 系统** (plugin/) | Hook 系统: 可在每个阶段注入自定义逻辑 |
| **—** | **`Bus` 事件总线** (bus/) | Effect PubSub 跨组件通信 |
| **—** | **`sub-agent` (tool: task)** (tool/task.ts) | 递归启动子 Session, 通过 Bus 交换上下文 |

> **⚠️ 关键澄清: System Prompt ≠ 任务编排**
>
> 在 Opencode 中, **System Prompt 是静态输入** (identity + context + constraints), 由四部分拼接:
> 1. `agent.prompt` (或 provider default prompt) — Agent 身份/行为基础
> 2. `env` — 环境信息 (模型名、工作目录、日期等)
> 3. `instructions` — 来自 CLAUDE.md / AGENTS.md 等文件的指令
> 4. `skills` — 可用 skill 描述
>
> **System Prompt 本身不执行也不编排任务。** 它只是一个输入到 LLM 的静态文本块。
>
> **任务编排 (Orchestration) 由以下机制共同实现:**
> - **User Message**: 实际的任务描述 (来自用户输入)
> - **Message History**: 先前的对话轮次 (包括工具调用记录)
> - **Available Tools**: Agent 权限决定可用的工具集
> - **runLoop 控制流**: `prompt.ts` 中的 while 循环决定继续/停止/compaction
> - **LLM Internal Reasoning**: 规划内化于 LLM 的推理链中, 通过 tool calls 逐步执行
>
> 这意味着在归因时, "规划能力差"不能归因于 System Prompt, 而应归因于 LLM 本身或工具配置。

##### Opencode 新增的独特组件

| 组件 | 文件 | 职责 |
|------|------|------|
| **Config 系统** | config/ | 分层配置 (默认+用户+项目+远程), jsonc 解析, 变量替换 |
| **Plugin Hook 系统** | plugin/ | tool.execute.before/after, chat.params, system.transform 等 9+ hook 点 |
| **MCP 服务** | mcp/ | 管理 MCP Server 生命周期 (stdio/HTTP), 工具转换, OAuth 认证 |
| **Permission 系统** | permission/ | 基于 agent 的 allow/deny/ask 权限规则, 用户交互式授权 |
| **Compaction 系统** | session/compaction.ts | 上下文溢出检测, 消息摘要压缩, 自动窗口管理 |
| **Bus 事件总线** | bus/ | Effect PubSub, session/mcp/permission 事件跨组件分发 |
| **Runner 执行器** | effect/runner.ts | Fiber 状态机 (Idle/Running/Shell/ShellThenRun) |
| **Sub-agent (task 工具)** | tool/task.ts | 递归调用 SessionPrompt.loop, 父子 session 间上下文交换 |
| **Snapshot 快照** | tool/snapshot/ | 文件变更检测, diff 生成, 自动 revert |
| **V2 Event System (实验性)** | v2/ | 事件溯源架构, session.next.* 事件, 消息状态重放 |

#### 3.1.2 各组件接口定义 (I/O 规范) — Opencode 版

> **评价响应质量的核心逻辑**: 针对 AI Benchmark 的每一个任务类型 (T1~T6), 各组件的"好/坏"标准不同。下面给出每个组件的通用可观测指标, 以及在具体任务场景下如何通过这些指标体现优劣。

##### 基础可观测指标表

| 组件 | 输入 | 输出 | 可观测定量指标 | 可观测定性指标 |
|------|------|------|---------------|---------------|
| **CLI/入口** | 用户原始输入 + flags | `SessionPrompt.prompt()` | 解析耗时, 参数错误率, 文件解析成功率, Agent 解析准确率 | 意图保留度, 歧义处理 |
| **Session (消息历史)** | 当前会话所有消息 | 过滤后的可见消息列表 | 消息总数, 上下文 token 数, compaction 触发次数, 保护工具保留率 | 历史完整性, 摘要保真度 |
| **Compaction** | 溢出前的消息列表 | 压缩后的消息列表 | 溢出前 token 数, PRUNE_MINIMUM 触达率, 摘要 token 数, 摘要轮次 | 摘要信息损失率, 关键上下文保留度 |
| **Agent.Service** | 任务描述, 上下文 | Agent.Info (含权限/模型/温度) | Agent 匹配准确率, 模型覆盖度 | Agent 选择合理性 |
| **ToolRegistry** | Agent.Info, 模型信息 | 过滤后的 Tool.Def[] | 可用工具数, 被权限过滤掉工具数, MCP 工具数 | 工具集完备性, 多余工具比 |
| **System Prompt 合成** | agent.prompt + env + instructions + skills | 静态 system message 数组 | Prompt 总长度, 各模块占比(env/instructions/skills/agent), 插件 transform 次数, CLAUDE.md 文件数量 | Prompt 结构完整性, 指令清晰度, 各模块一致性 |
| **任务编排 (runLoop)** | user message + 消息历史 + 可用工具 + agent 配置 | LLM stream + tool call 执行序列 | 循环轮次, 工具调用序列, LLM stop/continue 决策, 信息累积曲线, 死胡同检测 | 编排合理性, 工具使用策略, 收敛速度 |
| **SessionLLM (LLM Call)** | system + 消息历史 + 工具定义 | LLM Stream (text + tool_calls) | Token 消耗 (输入/输出/思考), 首 token 延迟, 总延迟, provider 重试次数, 超时次数 | 响应格式合规性, 思考链质量, 工具调用格式正确性 |
| **Tool 执行** (内置) | Tool 参数 (Effect Schema 校验) | Tool 执行结果 | 执行成功率, 平均耗时, 超时率, 错误类型分布, 输出截断率 | 结果与任务的匹配度, 信息增益 |
| **Tool 执行** (MCP) | MCP 协议请求 | MCP 协议响应 | MCP 连接状态, 调用成功率, 认证失败次数, Server 主动关闭次数 | MCP Server 稳定性 |
| **Tool 执行** (Sub-agent) | Sub-task 描述 | Sub-agent 回复 | Sub-agent 调用次数, 深度, Parent 上下文注入 token 数, 结果带回完整性 | Sub-agent 回答质量, 上下文传递保真度 |
| **Permission 系统** | 工具请求 + agent ruleset | allow/deny/ask | 被拒绝次数, ask 用户超时次数, 用户批准/拒绝比 | 规则过度约束率, 安全漏洞遗漏率 |
| **Plugin Hook 系统** | 各 hook 点的上下文 | 可能修改后的上下文 | Hook 执行次数, 单个 hook 耗时, hook 错误率 | Hook 逻辑正确性, 副作用控制 |
| **Bus 事件总线** | 各类事件 | 事件订阅者通知 | 事件吞吐量, 订阅者错误率 | 事件处理完整性 |
| **Snapshot** | 工作目录 | 文件变更 hash | 变更检测次数, diff 生成延时, revert 成功率 | 变更检测精确度 |

##### 任务类型与指标映射

对于具体开发任务（如 T3 影响范围预测）, 各组件的好坏如何通过指标体现:

```yaml
# 任务示例: T3 影响范围预测 (资源配额查询特性)
# 期望 Agent 回答: 应当修改 resource_mgr_service + libresource_client

组件好坏判定:
  CLI/入口:
    好: 正确解析了需求描述, 无参数错误, 模式正确
    坏: 解析丢失了关键需求内容, 或选择了错误的 Agent

  System Prompt:
    好: 包含"请分析影响范围"的明确指令, 包含微内核架构约束
    坏: 缺少影响范围分析的指令框架, Agent 不知该输出什么

  Agent 行为 (规划-执行):
    好: Agent 首先 read 相关模块代码 → grep 关键符号 → 综合分析后输出
        [可观测] 工具调用序列: read(2-3个核心模块) → grep(关键API) → 回答
        工具调用顺序合理, 信息逐步积累
    坏: Agent 跳过代码阅读直接回答; 或阅读了大量无关模块
        [可观测] 工具调用序列: read(10+个无关模块) → 无 grep → 无针对性分析
        信息增益低, 探索效率低

  LLM Call:
    好: 推理链包含了"需求→现状分析→缺口→影响模块→兼容性"的完整逻辑
        [可观测] reasoning token 占比合理, tool call 参数准确
    坏: 推理链跳跃, 直接跳到结论; 或 tool call 参数使用了错误的接口名
        [可观测] reasoning token 过少(无推理)/过多(低效推理), 工具参数类型错误

  Tool 执行:
    好: read 正确读取了 resource_mgr 的核心文件, grep 找到了关键结构体
        [可观测] read 命中 ground truth 修改文件, grep 找到关键符号
    坏: read 返回"文件不存在"或"无权限", grep 返回空结果
        [可观测] tool 执行失败率高, 或虽成功但返回了无关内容

  最终输出:
    好: 正确识别 resource_mgr_service 和 lib 为主要修改对象
    坏: 错误地将修改归到 kernel_core 模块


### 3.2 全链路可观测性 (Trace)

#### 3.2.1 Trace 数据结构

每个组件的关键调用节点记录一条 Trace Entry。Opencode 的 Trace 应覆盖其核心组件的 I/O:

```json
{
  "trace_id": "eval_20260501_task_0042_T3",
  "span_id": "span_0007",
  "parent_span_id": "span_0005",
  "component": "tool_exec_read",
  "stage": "tool_execution",
  "start_time": "2026-05-01T08:00:01.123Z",
  "end_time": "2026-05-01T08:00:01.456Z",
  "duration_ms": 333,
  "status": "success",

  "input_snapshot": {
    "tool_name": "read",
    "parameters": { "path": "src/services/resource_mgr/main.c", "range": "1-100" }
  },
  "output_snapshot": {
    "tool_result": "file content (truncated to 8192 chars)",
    "result_length": 8192,
    "tool_call_error": null
  },

  "context_snapshot": {
    "messages_before": 12,
    "total_tokens_so_far": 45231,
    "compaction_triggered": false,
    "compaction_count": 0
  },

  "shadow_eval": {
    "tool_appropriateness": 0.85,
    "information_gain": 0.72,
    "tool_parameter_quality": 0.90,
    "result_utilization": 0.65
  }
}
```

#### 3.2.2 追踪实现方案对比

| 方案 | 实现方式 | 侵入性 | 精度 | 性能开销 | 选型建议 |
|------|---------|--------|------|---------|---------|
| **AOP 切面追踪** | 在组件接口处注入拦截器 | 低 | 方法级 | <5% | **首选**, 适用于统一接口的 Harness |
| **OpenTelemetry + 手动埋点** | 标准 SDK 逐点 instrument | 中 | 自定义 | <3% | 适合已有 OTEL 基础设施的团队 |
| **Proxy/Wrapper 模式** | 为每个组件创建包装类 | 中 | 组件级 | <5% | 适合 Harness 重构阶段 |
| **LLM 代理模式** | 一个 LLM 代理记录和评论所有交互 | 无侵入 | 语义级 | 高 (API 费用) | **仅作为补充**, 不适合全量启用 |
| **反向代理/中间人** | 在网络层面拦截 API 调用 | 无侵入 | API 级 | <2% | 适合 LLM Call 追踪, 不适合内部组件 |

**推荐组合**: AOP 切面追踪(主) + 反向代理(LLM Call 追踪) + 周期性 LLM 代理(深度语义分析)

### 3.3 影子评估 (Shadow Evaluation)

影子评估的核心思想: **在 Harness 运行的同时, 非侵入式地对每个中间输出进行独立质量评估**, 不阻塞主流程, 异步执行。

#### 3.3.1 各组件影子评估维度 — 量化评估指标

> **核心设计原则**: 每个评估维度应满足 (1) 可自动化评估, (2) 输出标准化数值 (0-1 或 0-100), (3) 可依据 benchmark ground truth 做对比, (4) 支持跨历史版本追踪趋势。

##### 3.3.1.1 CLI / 入口层

| 评估维度 | 评估方法 | 量化指标 | 计算公式/判定规则 |
|---------|---------|---------|----------------|
| 意图保留度 | LLM-as-Judge: 对比"原始用户输入"与"解析后的结构化任务"的语义等价性 | `IntentPreserve` (0~1) | 1: 完全保留; 0.5: 部分丢失细节; 0: 核心意图被错误理解 |
| 参数解析完整性 | 规则校验: 检查所有必填参数是否被正确提取 | `ArgCompleteness` (0~1) | 正确解析的参数数 / 总参数数 |
| Agent 选择正确性 | 对比 benchmark 预设的正确 agent 与实际选择的 agent | `AgentMatch` (0/1) | 1: 匹配; 0: 不匹配 |
| 附件/文件处理成功率 | 检查输入中的文件引用、附件是否被正确处理 | `AttachmentSuccess` (0~1) | 成功处理数 / 总引用数 |
| 中文语义准确度 | 评估中文需求中的技术术语是否被正确解析 (如"配额查询"→ quota query 而非 quota modify) | `ZhSemanticAccuracy` (0~1) | LLM-as-Judge: 术语理解正确比例 |

##### 3.3.1.2 Session / 消息历史管理

| 评估维度 | 评估方法 | 量化指标 | 计算公式/判定规则 |
|---------|---------|---------|----------------|
| 上下文完整性 | 比较当前可见消息列表是否包含所有关键历史信息 | `ContextCompleteness` (0~1) | Compaction 发生后: 关键信息保留率 |
| Compaction 保真度 | LLM-as-Judge: 比较压缩摘要与原始消息的语义等价性 | `CompactFidelity` (0~1) | 摘要是否保留了所有关键决策和产出物 |
| Token 利用率 | 当前上下文 token 数 / 模型 context window 上限 | `TokenUtilization` (0~1) | <0.3: 窗口浪费; 0.3-0.7: 合理; >0.9: 溢出风险 |
| 保护工具保留率 | compaction 后原应保护的重要工具结果是否被保留 | `ProtectedToolRetention` (0~1) | 保留的关键工具结果数 / 应有的总数 |

##### 3.3.1.3 System Prompt 合成 (静态输入)

> **重要**: System Prompt 是**静态文本输入**, 不是任务编排器。它只提供 agent 身份、环境上下文、instruction 文件和 skill 描述。**任务编排由 runLoop + LLM 推理 + 工具可用性共同驱动**。因此, 即使 System Prompt 质量完美, 也不能保证编排合理; 同样, 编排问题不应归因于 System Prompt。

| 评估维度 | 评估方法 | 量化指标 | 计算公式/判定规则 |
|---------|---------|---------|----------------|
| **结构完整性** | 规则检查: prompt 包含 agent.prompt/环境/instructions/skills 四大部分 | `StructureScore` (0~1) | 含有的必要模块数 / 4 |
| 指令清晰度 | LLM-as-Judge: 指令是否具体、无歧义、可执行 | `ClarityScore` (0~1) | 1: 清晰具体; 0.5: 有模糊表述; 0: 指令缺失或矛盾 |
| 上下文注入比例 | Skill/知识库内容占 prompt 总长度的比例 | `ContextInjectRatio` (0~1) | (skill 内容 + 知识库内容) / 总 prompt 长度 |
| 插件 transform 质量 | 检查 plugin `experimental.chat.system.transform` 是否引入了有害或矛盾的指令 | `TransformQuality` (0~1) | LLM-as-Judge: 插件增加的内容是否与系统指令一致 |
| **冗余度** | 检查 prompt 中是否存在重复指令或相互矛盾的段落 | `PromptRedundancy` (0~1) | 冗余段落数 / 总段落数; >0.3 标记为高冗余 |
| 指令与任务匹配度 | System Prompt 中的指令是否与用户请求的任务类型匹配 | `TaskAlignment` (0~1) | LLM-as-Judge: 指令内容与任务的适配性 |
| 与 agent 身份一致性 | agent.prompt 描述的 agent 角色与实际任务需要的角色是否一致 | `AgentRoleFit` (0~1) | 例如用 explore agent 做代码修改任务 → 低分 |

##### 3.3.1.4 Agent 行为分析 (核心维度)

| 评估维度 | 评估方法 | 量化指标 | 计算公式/判定规则 |
|---------|---------|---------|----------------|
| **工具选择正确性** | 对比 ground truth 中应使用的工具与 Agent 实际使用的工具 | `ToolSelectionAccuracy` (0~1) | (命中数 - 无关选择数) / 应有工具调用数 |
| **工具调用序列合理性** | 分析工具调用顺序是否逻辑递进 (read→grep→分析 而非 随机跳跃) | `SequenceCoherence` (0~1) | LLM-as-Judge + 马尔可夫转移概率: 合理转移占比 |
| **探索深度** | Agent 是否充分阅读关键代码后再做出结论 | `ExplorationDepth` (0~1) | 实际读代码文件数 / 应读的核心文件数 |
| **探索广度** | Agent 是否覆盖了所有相关领域, 还是只看了局部 | `ExplorationCoverage` (0~1) | 实际覆盖模块数 / ground truth 应覆盖模块数 |
| **信息增益率** | 每一步工具调用的信息增量 / Token 消耗 | `InfoGainRate` (0~1) | 高→高效; 低→Agent 在空转或重复无效操作 |
| **死胡同发现** | Agent 是否进入了循环/重复的无效操作 (如反复读同一个文件) | `DeadEndDetected` (0/1) | 1: 检测到死胡同; 0: 未发现 |
| **Tool Call 参数质量** | 工具参数是否准确 | `ArgPrecision` (0~1) | 正确参数数 / 总工具调用数 |
| **回退/修正次数** | Agent 是否因错误而回退或修正 | `BacktrackCount` (0~N) | 每回退一次计数+1, 高频回退→规划能力弱 |
| **调用栈深度** | sub-agent 嵌套调用的最大深度 | `MaxStackDepth` (0~N) | >3 层: 过度分解; 0: 无 sub-agent 使用 |

##### 3.3.1.5 Tool 执行层 (内置 + MCP)

| 评估维度 | 评估方法 | 量化指标 | 计算公式/判定规则 |
|---------|---------|---------|----------------|
| **执行成功率** | 自动统计 | `ToolSuccessRate` (0~1) | 成功执行数 / 总调用数 |
| **超时率** | 自动统计 | `ToolTimeoutRate` (0~1) | 超时数 / 总调用数 |
| **错误类型分布** | 自动分类 | `ErrorTypeDist` (分布向量) | PermissionError/FileNotFound/Timeout/ParseError 占比 |
| **输出截断率** | 自动检查 | `TruncationRate` (0~1) | 被截断的输出数 / 总输出数 |
| **MCP 连接稳定性** | 自动统计 | `MCPStability` (0~1) | 1 - (MCP 断连次数 / MCP 总调用次数) |
| **输出与任务的匹配度** | 工具返回结果是否与所问问题相关 | `ResultTaskAlignment` (0~1) | LLM-as-Judge: 结果对任务的直接有用性 |
| **输出质量** (代码类工具) | 对于 write/edit 工具: 检查代码规范性 | `CodeQuality` (0~1) | 静态分析: lint 通过率, 编译通过率 |
| **shell 命令安全性** | 检查 shell 命令是否包含危险性操作 | `ShellSafety` (0~1) | 规则检查: 禁止命令模式命中率 |

##### 3.3.1.6 LLM Call 层 (核心)

> **Prompt 质量评估** — 当 LLM Call 的响应质量不高时, 需要进一步归因是"Prompt 本身差"还是"模型能力不够"或是"上下文不足":

```yaml
归因决策树 — LLM Call 响应差:
  步骤1: PromptTurboCheck (Prompt 结构 + 指令清晰度)
    ├── Prompt 结构不完整 (< 0.6):
    │    → Prompt 质量问题是根因
    │    → 进入 Phase 3 Prompt 优化器
    │
    ├── Prompt 结构完整但指令模糊 (0.6~0.8):
    │    → 需要评估 ClarityScore
    │    ├── ClarityScore < 0.7: 指令清晰度问题
    │    └── ClarityScore ≥ 0.7: 可能是模型或上下文问题
    │
    └── Prompt 结构完整且指令清晰 (≥ 0.8):
         → Prompt 不是根因
         → 下一步评估 ContextUtilization 和 ModelCapability

  步骤2: ContextUtilization (上下文利用率)
    ├── 检索到的关键信息未被回答引用 (< 0.4):
    │    → 归因到 LLM 的上下文利用能力或 Token 注意力偏移
    │
    └── ContextUtilization ≥ 0.4:
         → 检查 ModelCapability (模型能力边界)

  步骤3: ModelCapability 评估
    ├── 同一 Prompt 用更强的模型重跑得分显著提升 (>15%):
    │    → 当前模型能力不足, 建议升级模型
    │
    └── 重跑后得分无显著提升:
         → 任务本身超出当前 Prompt+模型 组合的能力上限
         → 需要重新考虑任务分解或 Harness 架构
```

| 评估维度 | 评估方法 | 量化指标 | 计算公式/判定规则 |
|---------|---------|---------|----------------|
| **Prompt 结构完整性** | 规则检查: prompt 包含系统指令、任务描述、上下文、输出约束 | `PromptStructure` (0~1) | 必要模块覆盖率 |
| **指令清晰度** | LLM-as-Judge: 任务指令是否有歧义 | `InstructionClarity` (0~1) | 1: 无歧义; 0.5: 有模糊处; 0: 无法执行 |
| **上下文利用率** | 对比最终回答中引用的信息与 prompt 中提供的上下文 | `ContextUtilization` (0~1) | 引用的关键信息数 / 提供的总关键信息数 |
| **推理链质量** | LLM-as-Judge: 推理步骤是否完整, 因果关系是否正确 | `ReasoningQuality` (0~1) | 完整推理步骤数 / 应有的推理步骤数 |
| **事实准确性** | NLI/蕴含检查: 回答中的事实性断言与 ground truth 的一致性 | `FactualAccuracy` (0~1) | 正确断言数 / 总断言数 |
| **格式合规性** | 规则校验: 输出格式是否符合要求的 schema | `FormatCompliance` (0/1) | 1: 完全符合; 0: 不符合 |
| **思考效率** | reasoning token / 输出 token 的比值 | `ReasoningEfficiency` (0~1) | 0.2~0.5: 合理; <0.1: 缺乏推理; >0.7: 过度思考 |
| **Provider 稳定性** | LLM provider 返回错误的频率 | `ProviderStability` (0~1) | 1 - (失败请求数 / 总请求数) |
| **Token 成本效率** | (端到端得分) / (总 token 消耗) | `TokenCostEfficiency` (0~1) | 归一化至 0-1, 值越高性价比越好 |

##### 3.3.1.7 Permission / 权限层

| 评估维度 | 评估方法 | 量化指标 | 计算公式/判定规则 |
|---------|---------|---------|----------------|
| **过度约束率** | 是否因权限拒绝导致合理行为被阻止 | `OverConstraintRate` (0~1) | 被拒绝的合理请求数 / 总请求数 |
| **约束遗漏率** | 是否本应拒绝的工具调用被放行了 | `UnderConstraintRate` (0~1) | 放行的危险操作数 / 总操作数 |
| **Ask 超时率** | 权限询问等待用户响应但超时的比例 | `AskTimeoutRate` (0~1) | 超时的 ask 数 / 总 ask 数 |
| **规则冲突检测** | Agent permission ruleset 与全局 ruleset 是否存在冲突 | `RulesetConflict` (0/1) | 检测到冲突规则则标记 |

##### 3.3.1.8 输出合成层

| 评估维度 | 评估方法 | 量化指标 | 计算公式/判定规则 |
|---------|---------|---------|----------------|
| **答案忠实度** | 回答是否忠实基于检索到的上下文和工具结果 | `Faithfulness` (0~1) | 从上下文中可支持的断言数 / 总断言数 |
| **幻觉率** | 回答中引入的上下文中不存在的信息比例 | `HallucinationRate` (0~1) | 幻觉断言数 / 总断言数 |
| **相关性** | 回答是否直接回应了用户的问题 | `AnswerRelevance` (0~1) | LLM-as-Judge: 与问题的相关度评分 |
| **完整性** | 回答是否覆盖了用户问题中的所有子问题 | `Completeness` (0~1) | 已覆盖子问题数 / 总子问题数 |
| **code diff 质量 (T5)** | 生成的代码补丁的编译通过率和测试通过率 | `PatchQuality` (0~1) | 编译通过=0.5 + 测试通过率×0.5 |
| **设计方案质量 (T4)** | 方案与真实实现的架构一致性 | `DesignConsistency` (0~1) | 专家 rubric 评分, 归一化

#### 3.3.2 影子评估的自动化实现方案对比

| 方案 | 方法 | 优势 | 劣势 | 精度 | 成本 | 选型建议 |
|------|------|------|------|------|------|---------|
| **规则/正则/静态分析** | 硬编码规则、模式匹配 | 零成本、确定性强 | 覆盖有限, 无法处理语义 | 中(仅语法层) | 零 | **必须采用**, 基础层 |
| **LLM-as-a-Judge (单次)** | 一个 LLM 对中间输出打分 | 灵活、覆盖面广 | 成本高, 评估器本身有偏差 | 中高 | 中 | **核心方法** |
| **LLM-as-a-Judge (多数投票)** | 多个 LLM 独立评估后投票 | 偏差抵消, 稳定性高 | 成本 ×N, 延迟 ×N | 高 | 高 | **推荐关键维度采用** |
| **RAGAS 框架** | 专用评估工具链 | 专为检索评估设计, 标准化 | 仅支持检索场景 | 高(检索) | 低 | 检索评估首选 |
| **微调专用评估模型** | 训练轻量打分模型 | 低推理成本, 可定制 | 需要大量标注数据 | 依赖训练 | 高(训练) | 长期演进的策略 |
| **对比学习/NLI 模型** | 预训练蕴含/矛盾模型 | 零样本, 适合事实性检查 | 仅支持单向判断 | 中 | 零 | 事实性维度推荐 |
| **N-gram / BLEU / ROUGE** | 基于重叠的量化指标 | 确定性强, 成本为零 | 无法捕捉语义等价 | 低 | 零 | 仅作参考, 不推荐为主 |

**推荐组合**: 规则(基础检查) + RAGAS(检索评估) + LLM-as-a-Judge(多数投票, 语义维度) + NLI 模型(事实性)

#### 3.3.3 Sub-agent 与主 Agent 的上下文交互评估

Opencode 中 sub-agent 通过 `task` 工具实现: 主 Agent 调用 task 工具 → 启动子 Session → 子 Agent 独立执行 → 结果注入回主 Session 的消息历史。

这是一条**关键但容易被忽视的归因链路**。sub-agent 的错误可能被"打包"成一段文本回到主 context, 导致主 Agent 基于错误信息继续决策。

##### Sub-agent 交互的数据流

```
主 Agent 任务描述
  │
  ▼
task() 工具 → SessionPrompt.loop(子 session)
  │               │
  │               ├── 子 Agent 独立 runLoop
  │               ├── 子 Agent 有独立的消息历史、工具权限、LLM Call
  │               └── 返回: 子 Agent 的最终回复 + 部分消息摘要
  │
  ▼
主 Session 消息历史
  └── 子 Agent 结果作为 tool-result 消息的一部分注入
  └── 主 Agent 继续处理 (可能基于子结果继续决策)
```

##### Sub-agent 交互的可观测指标

| 评估维度 | 评估方法 | 量化指标 | 判定规则 |
|---------|---------|---------|---------|
| Sub-agent 任务分解质量 | 主 Agent 是否将正确的子任务分配给 sub-agent | `TaskDecompQuality` (0~1) | 对比 ground truth subtask 划分 |
| Sub-agent 执行质量 | Sub-agent 是否正确完成了分配给它的子任务 | `SubagentSuccess` (0~1) | Sub-agent 端到端评分 |
| 上下文注入保真度 | 子 Agent 的结果是否被完整、准确地带回主 Session | `ContextFidelity` (0~1) | 子 session 的关键产出 k 是否在主 session 中出现 |
| 上下文注入损失率 | 主 Session 在注入子结果后, 原有上下文是否被不当裁剪 | `ContextOverwriteRate` (0~1) | 注入后消失的关键历史信息比例 |
| Sub-agent 规划独立性 | 子 Agent 是否在没有主 Agent 不必要的干预下独立完成任务 | `AutonomyScore` (0~1) | 子 Agent 内部工具调用 vs 主 Agent 额外补充指令的比例 |
| 跨 Session 权限一致性 | 子 Agent 的权限规则集是否与主 Agent 一致 | `PermissionConsistency` (0~1) | 一致的权限规则数 / 总相关规则数 |
| Sub-agent 调用开销 | 调用 sub-agent 消耗的 token 和时间增量 | `SubagentCost` (0~1) | (子 session token + 耗时) / 总 session token + 耗时 |
| 过度分解检测 | 本可以直接完成的任务被不必要地分解为 sub-agent 调用 | `OverDecompRate` (0~1) | 不必要的 sub-agent 调用数 / 总 sub-agent 调用数 |

##### Sub-agent 相关的归因规则

```
IF E2E 得分低 AND 主 Agent 调用了 sub-agent:
  IF SubagentSuccess 低 AND ContextFidelity 低:
    → 归因: Sub-agent 本身执行失败 + 结果带回也有问题
  IF SubagentSuccess 低 AND ContextFidelity 高:
    → 归因: Sub-agent 执行失败 (子上下文被正确带回但子任务本身做错了)
  IF SubagentSuccess 高 AND E2E 得分低:
    → 归因: 主 Agent 在收到子结果后处理不当
    → 需进一步检查: TaskDecompQuality 是否子任务分配错误
  IF OverDecompRate 高:
    → 归因: Agent Planner 过度分解, sub-agent 调用增加了不必要的复杂度
```

##### Sub-agent Trace 的数据结构

```json
{
  "span_id": "span_sub_001",
  "component": "sub_agent_task",
  "parent_span_id": "span_main_agent",
  "sub_session_id": "session_sub_456",

  "input": { "task_description": "分析 resource_mgr 的现有接口", "agent": "explore" },
  "output": { "summary": "现有接口包含..., 缺少查询能力..." },

  "shadow_eval": {
    "task_decomp_quality": 0.85,
    "subagent_success": 0.72,
    "context_fidelity": 0.91,
    "cost_ratio": 0.35   // 子 session token / 总 token
  }
}
```

### 3.4 归因分析策略

这是 EvoBench 最核心的部分。以下从**不同方法论流派**的角度, 系统性列出可行的归因策略, 并给出详尽分析。

---

#### 策略 A: 分级归因矩阵 (Grading-based Attribution)

**核心思想**: 基于规则和评分矩阵, 通过 Trace + Shadow Eval 的量化指标, 按预设逻辑将端到端失分归因到组件。

**方法**:
```
IF (端到端得分 < 阈值) AND (组件 A 影子评估得分低) AND (组件 A 输出质量差)
THEN 端到端失败归因于组件 A
```

**Opencode 归因规则矩阵**:

> 以下规则按"T3 影响范围预测"任务的失败模式为例, 但规则模式适用于所有任务类型。每行代表一种可观测的失败模式组合, 自动归因到对应的 opencode 组件。

##### 规则组 A: 入口 & 解析阶段失败

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| IntentPreserve < 0.6 AND ArgCompleteness < 0.8 | CLI/入口解析问题 — createUserMessage 未正确解析需求 | 高 (0.85) |
| AgentMatch = 0 (选择了错误的 agent) AND IntentPreserve ≥ 0.8 | Agent 选择错误 — agent 配置或默认 agent 设置不当 | 高 (0.80) |
| AttachmentSuccess < 0.7 | 文件/附件处理失败 — createUserMessage 的附件解析逻辑有 bug | 中高 (0.75) |

##### 规则组 B: System Prompt 合成失败 (静态输入质量问题)

> **注意**: System Prompt 只是静态文本输入。这里的失败指"给 LLM 的输入质量差", 而非"编排/规划能力差"。编排问题应在规则组 C 中处理。

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| StructureScore < 0.6 (四模块缺失) AND ToolSelectionAccuracy ≥ 0.7 | System prompt 结构不完整 — agent.prompt 或 skill/instruction 注入部分缺失 | 高 (0.85) |
| ClarityScore < 0.6 AND InstructionClarity < 0.6 | 指令模糊 — agent.prompt 或 instruction 文件表述不清 | 高 (0.80) |
| TransformQuality < 0.5 AND StructureScore ≥ 0.8 | Plugin hook (system.transform) 引入了错误指令 | 中高 (0.75) |
| PromptRedundancy > 0.3 | Prompt 冗余度过高 — 可能被多次注入相同 skill 内容 | 中 (0.65) |
| TaskAlignment < 0.5 AND StructureScore ≥ 0.8 | 指令与任务不匹配 — 例如用 explore agent system prompt 做开发任务 | 中高 (0.75) |

##### 规则组 C: Agent 行为/工具使用失败 (任务编排/规划问题)

> **核心**: 这里的失败反映了 **runLoop 驱动下的 LLM 编排行为** — Agent 选择了哪些工具、以什么顺序、做了多少探索。这是 opencode 中"任务编排"的实际体现。

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| ToolSelectionAccuracy < 0.4 AND ExplorationCoverage < 0.3 AND ExplorationDepth < 0.3 | Agent 工具使用能力不足 — 未正确选择 read/grep 工具来探索代码 | 高 (0.90) |
| ToolSelectionAccuracy ≥ 0.7 BUT SequenceCoherence < 0.4 (工具顺序凌乱) | Agent 规划能力弱 — 虽然用了正确的工具但执行顺序不合理 | 中高 (0.75) |
| DeadEndDetected = 1 (死胡同) AND InfoGainRate < 0.2 | Agent 陷入无效循环 — 反复读相同文件或重复执行相同 shell 命令 | 高 (0.85) |
| ExplorationCoverage < 0.5 (未覆盖应分析的模块) BUT ToolSelectionAccuracy ≥ 0.6 | Agent 覆盖不足 — 遗漏了关键模块的分析, 可能知识边界问题 | 中高 (0.75) |
| BacktrackCount > 3 (频繁回退) | Agent 的前期决策质量差 — 导致频繁修正, 反映规划预判能力弱 | 中 (0.65) |
| MaxStackDepth > 3 (sub-agent 嵌套过深) | Agent 过度分解任务 — sub-agent 嵌套层级过多, 增加不必要的复杂度 | 中 (0.60) |

##### 规则组 D: Tool 执行失败

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| ToolSuccessRate < 0.5 AND MCPStability ≥ 0.9 | 内置工具执行失败 — read/write/shell 等工具实现有 bug | 高 (0.85) |
| ToolSuccessRate < 0.5 AND MCPStability < 0.6 | MCP 连接不稳定 — MCP Server 频繁断连或认证失败 | 高 (0.80) |
| ToolTimeoutRate > 0.3 | 工具超时 — 可能执行时间过长的 shell 命令或 MCP 调用 | 中高 (0.75) |
| TruncationRate > 0.5 AND ArgPrecision ≥ 0.8 | 输出截断导致信息丢失 — Tool 输出被截断, Agent 无法获取完整信息 | 中高 (0.70) |
| ShellSafety < 0.5 | shell 命令安全性问题 — 工具执行了危险/不推荐的命令模式 | 中 (0.65) |

##### 规则组 E: LLM Call 层失败

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| PromptStructure < 0.6 (Prompt 结构差) | System prompt 合成层问题 → 需检查 agent.prompt / skill 注入 | 高 (0.85) |
| PromptStructure ≥ 0.8 AND InstructionClarity < 0.6 | Agent.prompt 或 instruction 指令本身存在歧义 | 高 (0.80) |
| PromptStructure ≥ 0.8 AND InstructionClarity ≥ 0.7 AND ContextUtilization < 0.4 | LLM 未有效利用上下文 — 可能是 context window 注意力问题或模型能力不足 | 中高 (0.75) |
| ContextUtilization ≥ 0.5 AND ReasoningQuality < 0.4 | LLM 推理能力不足 — 模型无法完成需要多步推理的任务 | 中高 (0.75) |
| ReasoningEfficiency < 0.1 (无推理痕迹) | LLM 模型不产生 reasoning token (非 reasoning 模型) 或未启用 thinking | 中 (0.65) |
| FactualAccuracy < 0.5 AND ContextUtilization ≥ 0.6 | LLM 幻觉 — 模型引入了超出上下文的信息, 即使提供了足够的上下文 | 中高 (0.75) |
| ProviderStability < 0.8 (provider 频繁出错) | LLM Provider 稳定性问题 — 需要降级或切换 provider | 中 (0.60) |

##### 规则组 F: Permission/权限问题

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| OverConstraintRate > 0.3 AND AskTimeoutRate < 0.2 | Permission 规则过度约束 — 合理的工具调用被 agent permission 拒绝 | 中高 (0.75) |
| AskTimeoutRate > 0.3 (用户频繁超时) | 权限交互效率低 — 用户无法及时响应权限询问, 导致任务中断 | 中 (0.65) |

##### 规则组 G: Sub-agent 交互失败

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| SubagentSuccess < 0.5 AND ContextFidelity ≥ 0.8 | Sub-agent 执行失败 — 子任务本身出错, 但结果被正确带回 | 高 (0.85) |
| SubagentSuccess ≥ 0.7 AND ContextFidelity < 0.5 | 上下文传递失败 — 子 Agent 结果丢失或损坏, 主 Agent 无法使用 | 高 (0.80) |
| OverDecompRate > 0.4 | 过度分解 — 本可以直接执行的任务被不必要地分包给 sub-agent | 中 (0.65) |

##### 规则组 H: Compaction / 上下文管理失败

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| CompactFidelity < 0.5 (压缩摘要丢失关键信息) AND ProtectedToolRetention < 0.6 | Compaction 破坏了关键上下文 — 压缩摘要未保留重要工具执行结果 | 高 (0.85) |
| TokenUtilization > 0.9 且 任务未完成 | 上下文窗口不足 — 任务需要更多上下文但被 context window 限制 | 中高 (0.75) |
| CompactFidelity < 0.4 AND TokenUtilization < 0.7 | 不必要地触发了 compaction — 窗口未满但触发了压缩, 且摘要质量差 | 中 (0.65) |

##### 规则组 I: 端到端结果质量失败

| 观测条件 (Shadow Eval 指标) | 归因结论 | 置信度 |
|---------------------------|---------|-------|
| Faithfulness < 0.5 AND All tool 执行成功 | LLM 生成时未基于 tool 结果 — 可能是 Prompt 未约束"必须引用工具结果" | 高 (0.80) |
| HallucinationRate > 0.3 AND FactualAccuracy < 0.4 | LLM 幻觉严重 — 模型编造了不存在的信息 | 中高 (0.75) |
| Completeness < 0.5 (回答不完整) AND ToolSelectionAccuracy ≥ 0.7 | Agent 收集了足够信息但最终回答不全面 — 输出合成问题 | 中高 (0.75) |

---

##### 跨规则组判定优先级

当多个规则组同时命中时, 按"信息流上游优先"原则判定——即先流水线前端的组件优先:

```
判定优先级 (从高到低):
  1. CLI/入口解析失败 (规则组 A)  → 前面已经错了, 后面不必再看
  2. System Prompt 合成失败 (规则组 B) → Prompt 错了直接影响全部
  3. Compaction 破坏上下文 (规则组 H) → 上下文不完整导致后续所有步骤偏差
  4. Permission 过度约束 (规则组 F) → 工具无法调用的前置错误
  5. Agent 行为失败 (规则组 C) → Agent 工具使用策略问题
  6. Tool 执行失败 (规则组 D) → 工具本身出错了
  7. Sub-agent 失败 (规则组 G) → Sub-agent 级别的错误
  8. LLM Call 层失败 (规则组 E) → LLM 自身的能力或配置问题
  9. 端到端质量失败 (规则组 I) → 如果是前面都正常但最终输出差, 则是输出合成问题
```

**优缺点**:
| 维度 | 评价 |
|------|------|
| 可解释性 | ★★★★★ 规则清晰, 每步可回溯 |
| 工程实现 | ★★★★☆ 实现简单, 规则引擎即可 |
| 归因精度 | ★★★☆☆ 依赖规则库的完备性 |
| 覆盖边界案例 | ★★☆☆☆ 硬规则无法处理模糊场景 |
| 维护成本 | ★★★☆☆ 规则库需持续维护 |
| 计算开销 | ★★★★★ 极低 |

**适用场景**: 作为主要归因策略, 覆盖 80% 的常规失败模式。

---

#### 策略 B: LLM-as-a-Judge 归因 (LLM Attribution)

**核心思想**: 利用 LLM 分析全链路 Trace, 自动推断失败的根因组件。

**方法**:
```
给 LLM 完整的 Trace + 端到端结果 + Shadow Eval 分数,
Prompt: "分析此执行链路, 指出第一个出现异常的环节及原因"
```

**Prompt 模板示意**:
```
你是一个 Harness 工程诊断专家。以下是本次任务的完整执行 Trace 和影子评估结果:

[Trace JSON]
[Shadow Eval JSON]
[端到端评分: 65/100]

请分析:
1. 端到端失分的最主要根因是哪个组件?
2. 次要根因有哪些?
3. 证据是什么? (引用具体的 Trace 条目和 Shadow Eval 指标)
4. 建议的优化方向?
```

**优缺点**:
| 维度 | 评价 |
|------|------|
| 可解释性 | ★★★★☆ 输出自然语言分析, 但可能幻觉 |
| 工程实现 | ★★★★☆ 仅需一个 LLM Call |
| 归因精度 | ★★★☆☆ 依赖 LLM 诊断能力, 存在上限 |
| 覆盖边界案例 | ★★★★★ 可处理任何模糊场景 |
| 维护成本 | ★★★★★ 几乎为零 |
| 计算开销 | ★★★☆☆ 每次推理额外一次 LLM Call |

**变体 - 多数投票归因**: 调用 3-5 个不同模型/温度独立分析, 统计归因分布。
- 优势: 抵消单一 LLM 的偏差
- 成本: LLM Call × (3~5)
- 精度: 通常优于单次, 但仍受制于最佳模型能力

**变体 - Chain-of-Thought 归因**: 要求 LLM 先逐步推理再给结论, 而非直接输出诊断。
- 优势: 推理过程可审查, 减少误判

**适用场景**: 作为规则矩阵的补充, 处理规则无法覆盖的边界案例和交叉失败模式。

---

#### 策略 C: 消融实验 / 反事实归因 (Counterfactual / Ablation)

**核心思想**: 通过系统性地替换/移除/退化特定组件, 观察端到端评分的变化, 用 Δscore 衡量该组件的影响。

**方法**:

```
对 benchmark task 中的每一个 task:

1. Base Run: 正常 Harness → 得分 S_base
2. Ablation Run A: 替换组件 A 为基线版本(如无用版本) → 得分 S_A
3. Ablation Run B: 替换组件 B 为基线版本 → 得分 S_B
...
N. 计算每个组件的贡献: ΔA = S_base - S_A

组件的重要度: Δ 越大 → 该组件越关键(但非短板)
短板判定: 若 ΔA 小 但 S_A < 阈值 → 组件 A 本身质量差, 但整体不依赖 → 可能是过度设计
        若 ΔA 大 且 组件 A 当前版本弱 → 组件 A 是关键瓶颈
```

**优缺点**:
| 维度 | 评价 |
|------|------|
| 可解释性 | ★★★★★ 因果推断的黄金标准 |
| 工程实现 | ★☆☆☆☆ 需要为每个组件准备替换版本 |
| 归因精度 | ★★★★★ 最可靠的归因 |
| 覆盖边界案例 | ★★★★☆ 全面, 但受限于替换版本的质量 |
| 维护成本 | ★☆☆☆☆ 每次增添组件需维护替换版本 |
| 计算开销 | ★★☆☆☆ (N+1) × 原始成本, 组合爆炸 |

**优化 - 分层消融**: 不测试所有组件组合, 而是按层次分组测试:
- 级别 1: 文本类 (Prompt + 记忆 + 知识库 + 宪法)
- 级别 2: 逻辑类 (Planner + Router + Tool Executor)
- 级别 3: 生成类 (LLM Call + 输出合成)

先精确定位到层级, 再在该层级内做二次消融。复杂度从 O(2^N) 降至 O(log N)。

**优化 - 差分消融**: 不进行完整的替换, 而是对少量代表性 task 进行消融, 用统计推断扩展结果。

**适用场景**: 不作为常规手段, 推荐作为：
1. 周期性深度分析 (如每月运行一次全量消融)
2. 复杂失败模式的仲裁手段
3. 新组件上线的验证环节

---

#### 策略 D: Shapley Value 归因 (合作博弈论方法)

**核心思想**: 借鉴可解释 AI (XAI) 中的 Shapley Value, 将每个组件视为合作博弈中的"玩家", 计算每个玩家对最终得分的边际贡献。

**方法**:
```
对 N 个组件, 计算每个组件 i 的 Shapley Value φ(i):

φ(i) = Σ_{S ⊆ N\{i}} (|S|! (N-|S|-1)! / N!) × (v(S ∪ {i}) - v(S))

其中 v(S) 是只启用组件子集 S 时的端到端得分。
```

**优缺点**:
| 维度 | 评价 |
|------|------|
| 可解释性 | ★★★★★ 理论坚实, 有公理化保证 |
| 工程实现 | ★☆☆☆☆ 计算复杂度 O(2^N), 需要大量消融运行 |
| 归因精度 | ★★★★★ 具有唯一性和公平性 |
| 覆盖边界案例 | ★★★★★ 理论完备 |
| 维护成本 | ★☆☆☆☆ 极高 |
| 计算开销 | ★☆☆☆☆ O(2^N), 完全不可行 |

**优化 - 近似 Shapley**:
- **KernelSHAP**: 用加权线性回归近似 Shapley Value, 只需 O(N²) 次采样
- **Monte Carlo Shapley**: 随机采样组件子集, 在精度和成本间做 trade-off

**适用场景**: 理论上最优, 但实践成本过高。可考虑：
1. 极低频的深度分析 (如季度一次)
2. 仅对少数关键组件 (N ≤ 5) 进行精确计算
3. **不推荐作为常规归因手段**

---

#### 策略 E: 贝叶斯因果推断 (Bayesian Causal Inference)

**核心思想**: 构建 Harness 组件的因果图 (DAG), 用贝叶斯方法推断各组件到最终得分的因果效应。

**方法**:
```
1. 构建因果图: 组件 A → 组件 B → ... → 最终得分
2. 收集观测数据: 大量历史运行的 Trace + Shadow Eval + 得分
3. 拟合结构方程模型 (SEM) 或贝叶斯网络
4. 计算每个节点的"因果效应" (do-calculus)
5. 短板 = 因果效应显著但平均得分低的节点
```

**优缺点**:
| 维度 | 评价 |
|------|------|
| 可解释性 | ★★★★☆ 因果路径清晰, 但模型复杂 |
| 工程实现 | ★★☆☆☆ 需要大量历史数据 + 专业知识构建因果图 |
| 归因精度 | ★★★★☆ 在数据充足时非常精准 |
| 覆盖边界案例 | ★★★☆☆ 依赖因果图正确性, 遗漏边则偏差 |
| 维护成本 | ★★☆☆☆ 因果图需随 Harness 架构变化更新 |
| 计算开销 | ★★★☆☆ 需要大量采样, 但离线运行 |

**适用场景**:
- 有大量历史 Trace 数据后 (≥ 1000 次运行)
- 需要发现"隐藏瓶颈" (非直接可见的组件间交互效应)
- **中期建设目标**, 不适合 MVP 阶段

---

#### 策略 F: 过程行为分析 (Process-Oriented Behavioral Analysis)

**核心思想**: 不依赖于 end-to-end 分数差的对比, 而是深入分析 Agent 的中间决策过程, 通过"行为特征"识别短板。

**方法**:
```
对 Agent Planner 的输出进行行为模式挖掘:

1. 计划结构分析:
   - 计划是否包含"假步骤" (看似有逻辑但实际无信息增益)
   - 计划中的回溯/修正次数 → 反映规划能力
   - 工具选择与被拒绝的比例 → 反映工具选择质量

2. 探索-利用分析:
   - Agent 在搜索代码时是系统探索还是随机跳跃
   - 是否有明显的"死胡同"行为

3. 信息增益追踪:
   - 每个步骤后 Agent 的信息增益量
   - 高 token 消耗 + 低信息增益 = 无效推理
```

**针对 LLM Call 的行为分析**:

```
对 Prompt 和 Response 的深入分析:

1. Prompt 结构完整性评分:
   - 是否有系统指令、任务描述、few-shot 示例、输出格式约束
   - 各段落的清晰度和具体性

2. Context 窗口利用率:
   - 实际使用的 context 比例
   - 是否有大量无关上下文挤占窗口

3. 推理链质量:
   - 是否是真正的链式推理而非伪推理
   - 推理步骤的因果关系是否正确

4. 幻觉 / 事实漂移检测:
   - Response 中引入了多少非上下文信息
   - 与 ground truth 的矛盾点
```

**优缺点**:
| 维度 | 评价 |
|------|------|
| 可解释性 | ★★★★★ 行为层面的分析极其直观 |
| 工程实现 | ★★☆☆☆ 需要定制化分析器和 heuristic |
| 归因精度 | ★★★★☆ 能发现"隐性短板" (评分尚可但行为低效) |
| 覆盖边界案例 | ★★★★☆ 行为分析可发现统计评分无法覆盖的问题 |
| 维护成本 | ★★☆☆☆ Heuristic 需要随 Agent 行为变化更新 |
| 计算开销 | ★★★☆☆ LLM 密集型, 但可离线批量 |

**适用场景**:
- Agent Planner 和 LLM Call 的深度分析
- 发现"评分尚可但低效"的隐性短板
- **作为分级归因的补充**, 提升归因覆盖率

---

#### 策略 G: 集成归因引擎 (推荐方案)

**核心思想**: 将上述多策略并联, 输出融合的归因结果。

**架构**:

```
┌────────────────────────────────────────────────────┐
│                集成归因引擎                            │
│                                                      │
│  ┌─────────────────────────────┐                    │
│  │   第一层: 规则矩阵 (快速判定)   │  ← 处理 80% 常见案例   │
│  │   - 确定性 if-then 规则      │                     │
│  │   - 基于 Trace + Shadow Eval │                     │
│  │   - 输出: 归因结论 + 置信度    │                     │
│  └─────────────┬───────────────┘                     │
│                │                                      │
│                ▼                                      │
│  ┌─────────────────────────────┐                    │
│  │   第二层: LLM 归因 (模糊判定)  │  ← 处理 15% 边界案例   │
│  │   - 输入: Trace + 规则矩阵结果 │                     │
│  │   - 输出: 补充归因 + 置信度    │                     │
│  └─────────────┬───────────────┘                     │
│                │                                      │
│                ▼                                      │
│  ┌─────────────────────────────┐                    │
│  │   第三层: 行为分析 (隐性归因)  │  ← 处理 5% 隐性短板   │
│  │   - 过程行为模式挖掘          │                     │
│  │   - 输出: "隐性短板" 报告      │                     │
│  └─────────────┬───────────────┘                     │
│                │                                      │
│                ▼                                      │
│  ┌─────────────────────────────┐                    │
│  │   融合器: 加权投票 + 置信度校准 │                     │
│  │   - 每层按历史精度赋予权重      │                     │
│  │   - 输出最终归因: {组件, 分数, 置信度} │                │
│  └─────────────────────────────┘                     │
│                                                      │
│  定期深度分析层 (低频, 高成本)                          │
│  ┌─────────────────────────────┐                     │
│  │   消融实验 / 贝叶斯推断        │  ← 每月/每季度一次     │
│  │   - 验证和校准归因引擎          │                     │
│  │   - 发现新的失败模式            │                     │
│  │   - 更新规则矩阵                │                     │
│  └─────────────────────────────┘                     │
└──────────────────────────────────────────────────────┘
```

**融合逻辑**:

```
最终归因向量 = softmax(
    w₁ × Φ_rule(component) +
    w₂ × Φ_llm(component) +
    w₃ × Φ_behavior(component)
)

其中:
- w₁ > w₂ > w₃ (按历史精度赋值, 默认 0.6 : 0.3 : 0.1)
- 置信度 = max(最终归因向量)
- 当置信度 < 阈值(如 0.5) 时, 标记为"需要人工审查"
```

**为什么推荐集成方案**:

| 维度 | 单一策略 | 集成策略 (推荐) |
|------|---------|----------------|
| 归因覆盖率 | 60-80% | 95%+ |
| 精度 | 取决于策略选择 | 高于任一单策略 |
| 鲁棒性 | 单点失效 | 容错, 一层失败其他层补充 |
| 维护成本 | 低-中 | 中 (但回报远超成本) |
| 计算开销 | 取决于策略 | 可控 (三层可配置启用/禁用) |

### 3.5 诊断报告输出

归因引擎的输出是一个结构化的诊断报告:

```yaml
diagnosis:
  task_id: feat_0042_T3
  overall_score: 65
  primary_bottleneck:
    component: knowledge_retriever
    contribution: 0.55        # 对失分的贡献比例
    confidence: 0.87
    evidence:
      - "Retrieval Precision@3: 0.33 (threshold: 0.7)"
      - "NoiseRatio: 0.45 (threshold: 0.2)"
      - "Rule matrix: E2E低 & 规划高 & 检索低 & 利用率低"
      - "LLM diagnosis: '检索到的文档偏离了资源配额的核心概念'"
  secondary_bottlenecks:
    - component: agent_planner
      contribution: 0.25
      confidence: 0.72
      evidence: [...]
    - component: prompt_template
      contribution: 0.20
      confidence: 0.65
      evidence: [...]
  action_suggestions:
    - type: optimize_retrieval
      target: knowledge_retriever
      suggestion: "优化文档分块大小 (当前 512 → 建议 256), 增加查询扩展"
    - type: refine_prompt
      target: prompt_template
      suggestion: "在系统提示中明确要求引用检索文档的章节号"
```

---

## 4. Phase 3: 数据驱动的自进化 — 靶向优化引擎

根据 Phase 2 归因分析的结果, 靶向执行针对不同组件类型的自动优化。

### 4.0 各组件的优化目标定义 — 以微内核特性开发为中心

> **总体目标**: 提高闭源微内核操作系统的特性开发效果。具体体现为使 AI Benchmark 的 **T1~T5 五项能力得分持续提升**。
>
> 因此各组件的优化目标不是孤立的通用指标, 而是**围绕五项能力展开的、面向微内核 OS 开发场景的靶向目标**。

#### 4.0.1 五项能力对微内核开发场景的优化要义

| 能力 | 微内核场景下的"好" | 场景痛点 | 主要依赖的组件 |
|------|-------------------|---------|---------------|
| **T1 基线代码理解** | 准确区分内核态/用户态模块、理解 IPC 接口契约、识别 capability 机制、正确解析中文注释 | 代码库规模大、微内核架构不同于 Linux、中文注释可能引起语义偏差 | Tool(read/grep), LLM Call, System Prompt |
| **T2 需求理解** | 准确理解中文技术需求中的微内核术语、"配额查询"="query"非"modify"、识别隐藏的约束 | 中文需求可能有歧义、需求文档中英混排 | CLI/入口, System Prompt |
| **T3 影响分析** | 准确预测 IPC 消息变更影响的范围、识别需要修改的用户态服务、不误判为内核修改 | IPC 链路的"涟漪效应"复杂、多服务模块间依赖关系隐蔽 | Agent 编排(runLoop), Tool, Sub-agent |
| **T4 方案设计** | 设计遵循微内核最小化原则、职责层次正确(内核/服务/库)、接口设计不违反 IPC 规范 | 架构约束严格(不能把用户态逻辑放内核)、设计空间受限于闭源接口 | LLM Call, System Prompt, Agent 编排 |
| **T5 编码实现** | 代码风格与基线一致、正确使用闭源 API 和宏、编译通过率 100%、正确处理中文注释 | 闭源 API 不可搜索外部文档、需从基线代码推测用法、交叉编译环境复杂 | Tool(write/edit), LLM Call, MCP/Plugin |

#### 4.0.2 以五项能力为中心的各组件优化目标

> 每个组件的优化目标表达为: **"优化该组件 → 提升哪些 T 能力 → 在微内核开发中的具体表现"**

| 组件 | 服务于哪些 T 能力 | 优化目标 (微内核场景) | 衡量指标 | Trade-off |
|------|------------------|---------------------|---------|-----------|
| **CLI/入口** | T2 | **中文需求术语映射准确**: 将中文需求中的"服务域配额查询"精确映射到英/中文混合的技术实现语境, 而非丢失或错误术语 | `ZhSemanticAccuracy` ≥ 0.9 — 中文技术术语被正确解析的比例 | 术语精确 vs Prompt 简洁 |
| **Session/消息历史** | T1, T3, T4 | **架构决策链路保留**: 跨多轮对话保留 IPC 协议设计决策、模块职责划分依据等架构层面的上下文 | 跨轮次 `ContextCompleteness` — 多轮开发后关键架构约束是否仍在上下文中 | 保留关键架构决策 vs 窗口大小限制 |
| **Compaction** | T1, T3 | **微内核术语保真**: 压缩摘要必须保留 IPC 消息类型、能力权能(capability)类型、服务边界等微内核特有的关键信息。**压缩后不应丢失"这个 API 是内核态还是用户态"这类关键区分** | `CompactFidelity` ≥ 0.85 — 对微内核关键信息的保留比例; 分类检查: 模块归属(内核/服务/库)是否在压缩后仍可区分 | 压缩率 vs 架构信息保真度 |
| **Agent 定义/权限** | T3, T5 | **工具权限恰到好处**: (1) 能读取所有相关模块代码 (T3/T4) (2) 能编辑正确位置 (用户态服务不可误写内核, vice versa) | `OverConstraintRate` < 0.1 (不阻止必要的代码读/写) + `MisplacedEditRate` < 0.05 (不写入错误层) | 安全保护 vs 开发效率 |
| **System Prompt 合成** | T1, T3, T4 | **注入微内核架构约束**: System Prompt 必须传达以下关键约束: (1) 内核/用户态职责分离原则 (2) IPC 接口契约 (3) capability 安全模型 (4) 代码注释的中文风格指南。这些约束若无注入, Agent 可能设计出违反架构的方案 | `MicrokernelConstraintScore` (0~1) — LLM-as-Judge: System Prompt 是否包含了正确的架构约束 | 约束充分 vs Prompt 长度 |
| **任务编排 (runLoop)** | T1, T3, T4 | **合适的探索-设计-实现顺序**: Agent 在特性开发中应遵循: 读相关代码 → 理解现有接口 → 分析影响 → 设计方案 → 实现。**不应跳过读代码直接设计, 也不应过度探索而不产出** | `DevWorkflowCoverage` (0~1) — Agent 在开发过程中是否覆盖了 "理解→分析→设计→实现" 的完整工作流; `SkipDesignRate` (0~1) — 直接编码而未先设计 | 探索充分 vs 尽早产出 |
| **Tool 执行** | T1, T5 | (1) **read/grep 准确命中关键模块**: 能正确读取内核 IPC 头文件、用户态服务代码等 (2) **write/edit 符合代码风格**: 写入的代码符合该微内核的编码规范 | `CodebaseHitRate` — 读取的文件是否属于 ground truth 修改范围; `CodeStyleCompliance` — 代码风格与基线一致性 | 代码读取广度 vs 专注度 |
| **LLM Call** | T1, T2, T3, T4, T5 | **对微内核概念的理解深度和生成质量**: (1) 理解 capability、IPC、service domain 等微内核特有概念 (2) 生成的代码遵循闭源 API 的使用方式 (3) 中文注释风格与基线一致 (4) 推理链包含架构层面的考量 | `MicrokernelConceptAccuracy` (0~1) — LLM-as-Judge: 对微内核概念的理解准确度; `APIMatchRate` — 使用的 API 是否真实存在; `ChineseCommentStyleScore` — 注释风格与基线一致度 | 模型推理深度 vs 推理成本 |
| **Sub-agent** | T3, T4 | **按模块边界恰当地分解**: 微内核特性开发天然可按层分解——内核改动、用户态服务改动、客户端库改动。sub-agent 应按此边界分解任务, 各层并行分析后再汇总 | `LayerBasedDecompScore` (0~1) — sub-agent 分解是否按微内核的"内核/服务/库"分层; `CrossLayerConsistency` — 各层结果组合后是否自洽 | 分解粒度 vs 汇总一致性 |
| **MCP/Plugin** | T1, T5 | **构建与调试工具链可靠**: 提供微内核的交叉编译环境、测试运行器等基础设施工具的稳定访问 | `BuildToolStability` — 构建工具调用成功率; `TestExecSuccessRate` — 测试执行成功率 | 工具丰富性 vs 稳定性 |

#### 4.0.3 三个跨组件的元目标

```
元目标 1: T1~T5 综合得分持续提升 ↗
  每轮自进化后, 综合 score 应环比提升 ≥ 3%
  任何单 T 能力不得出现 > 5% 的回退 (负优化控制)

元目标 2: 微内核特有错误的递减 ↘
  关键错误率 (如"将用户态服务功能误放内核"、"违反 IPC 协议约定"等) 应逐轮递减
  每一轮进化应减少 ≥ 1 类高频错误

元目标 3: 中文开发环境的适配度提升 ↗
  中文注释风格与基线一致度, 中文技术需求的语义理解准确度, 应持续提升
```

#### 4.0.4 组件间的补偿效应与归因陷阱

微内核开发的特殊性导致组件间存在特殊的补偿效应:

```yaml
典型补偿效应:

  System Prompt 缺失架构约束 → LLM 设计违反微内核原则:
    → 如果 prompt 没有明确"不要把用户态逻辑放内核", 强模型可能自己"猜到"但弱模型必然犯错
    → 优化 prompt 后, 弱模型得分会显著提升 → 但这不是 LLM 变强了, 是 constraints 给了更明确的引导
    → 归因时必须区分: "prompt 缺约束" vs "模型能力不够"

  Tool 输出截断 → Agent 探索效率低:
    → 微内核 IPC 头文件往往很长 (>500行), 如果 read 工具截断输出, Agent 可能只看到前半部分, 误判接口
    → 优化 truncation 策略后, Agent 的模块理解准确率可能显著提升
    → 这很容易被误归因为"Agent 工具使用能力差"

  Compaction 丢失模块归属信息 → T3 影响分析错误:
    → 压缩摘要遗漏了"某个 API 属于用户态服务 X", Agent 后续分析时可能将影响范围误判到内核
    → 这容易被误归因为"Agent 的微内核架构理解能力不足"
```

**应对策略**: 每次优化后, 除检查目标指标外, 还需在 benchmark 的"架构错误"子维度上进行专项验证。如果架构错误减少, 证明优化方向正确; 如果仅其他指标提升而架构错误未改善, 需要怀疑是补偿效应。

#### 4.0.5 各组件优化目标与 T 能力映射索引

```
T1 (基线理解) ← 主要受: Tool(read/grep命中率) + Compaction(架构信息保真) + LLM Call(概念理解)
T2 (需求理解) ← 主要受: CLI/入口(中文术语精度) + System Prompt(微内核术语定义)
T3 (影响分析) ← 主要受: Agent编排(探索覆盖率) + Sub-agent(按层分解) + Tool(跨模块读取)
T4 (方案设计) ← 主要受: LLM Call(架构推理) + System Prompt(约束注入) + Agent编排(设计-实现顺序)
T5 (编码实现) ← 主要受: Tool(write/edit代码风格) + LLM Call(API正确性) + MCP/Plugin(构建工具)
```

> **关键洞察**: 同一组件可能服务于多个 T 能力, 但权重不同。例如 System Prompt 主要影响 T1/T4, 对 T5 影响较小。在归因和优化时, 应参考此权重分配优先级。**优先优化影响多个 T 能力且当前 Shadow Eval 得分低的组件。**

---

### 4.1 Prompt / 文本类组件优化

#### 4.1.1 优化对象

| 组件 | 优化内容 |
|------|---------|
| Request 模板 | 用户请求的结构化模板 |
| Agent 系统提示 | Agent 的初始系统指令 |
| Skill 描述文本 | Skill 的 function calling 描述 |
| 宪法规则 | 系统约束和安全规则 |
| 输出格式指令 | 期望的输出格式说明 |

#### 4.1.2 优化策略对比

##### 策略 4.1.A: 人工反馈 + LLM 改写 (半自动)

**流程**: 归因引擎定位到 Prompt 问题 → 收集失败的 prompt 示例 → LLM 分析失败模式 → LLM 生成优化版本 → 人工审查 → 部署。

| 维度 | 评价 |
|------|------|
| 效果 | ★★★★☆ 结合人类判断, 质量高 |
| 速度 | ★★☆☆☆ 依赖人工审查, 迭代慢 |
| 自动化程度 | ★★☆☆☆ 半自动 |
| 适用场景 | 初期或安全敏感场景 |

**选型建议**: 项目初期的默认方案。

##### 策略 4.1.B: DSPy 风格自动优化

**核心思想**: 将 Prompt 视为可优化的"模块"(signature), 通过编译器自动优化。

**方法**:
```
1. 将 Harness 的 Prompt 定义为 DSPy Signature:
   class TaskPlanSignature(dspy.Signature):
       """根据任务描述生成执行计划"""
       task_description = dspy.InputField()
       baseline_context = dspy.InputField()
       execution_plan = dspy.OutputField()
   
2. 定义评估指标 (来自 Benchmark T3 的评分)
3. 运行 DSPy 编译器进行 Prompt 优化:
   - 自动调整指令措辞
   - 选择最优的 few-shot 示例
   - 优化 prompt 结构
```

| 维度 | 评价 |
|------|------|
| 效果 | ★★★★☆ 系统性优化, 效果稳定 |
| 速度 | ★★★★☆ 全自动, 无人工介入 |
| 自动化程度 | ★★★★★ 全自动 |
| 适用场景 | 有明确输入/输出格式的结构化 Prompt |

**优点**:
- 业界验证效果 (DSPy 在多个 benchmark 上超越手工 Prompt)
- 自动化程度高, 可融入 EvoBench 闭环
- 与 Harness 的 Agent Planner 等结构化组件天然契合

**缺点**:
- 对非结构化、需要创造性输出的 Prompt 效果有限
- 优化后 Prompt 可能变得"怪异" (虽效果好但可读性下降)
- 需要维护优化流水线

**选型建议**: **强烈推荐作为结构化 Prompt 的主要优化手段**。

##### 策略 4.1.C: OPRO (Optimization by PROmpting)

**核心思想**: 用 meta-LLM 自动生成和改进 Prompt, 通过迭代搜索最优 Prompt。

**方法**:
```
Meta-LLM 接收:
- 当前 Prompt
- 在当前 Prompt 上的评测得分
- 失败案例分析
→ 生成改进后的 Prompt
→ 在少量验证集上测试
→ 循环直到收敛
```

| 维度 | 评价 |
|------|------|
| 效果 | ★★★★☆ 灵活, 可处理复杂目标 |
| 速度 | ★★★☆☆ 每次迭代需要 LLM Call + 验证运行 |
| 自动化程度 | ★★★★☆ 全自动, 但收敛慢 |
| 适用场景 | Agent System Prompt 等非结构化场景 |

**选型建议**: 适用于 DSPy 无法处理的非结构化 Prompt (如 Agent 个性设定、创作风格指导)。

##### 策略 4.1.D: 进化搜索 (Evolutionary Prompt Optimization)

**核心思想**: 将 Prompt 视为"基因", 通过变异 + 交叉 + 选择进行多代进化。

**方法**:
```
Population = [prompt_1, prompt_2, ..., prompt_N]
每一代:
  1. 变异: 对每个 prompt 微调措辞
  2. 交叉: 混合两个 prompt 的段落
  3. 评估: 在验证集上运行 Benchmark
  4. 选择: 保留得分最高的 Top-K
```

| 维度 | 评价 |
|------|------|
| 效果 | ★★★★★ 理论上可逼近全局最优 |
| 速度 | ★☆☆☆☆ 需要大量验证运行 (每代 K×N 次评测) |
| 自动化程度 | ★★★★☆ 全自动 |
| 适用场景 | 有充足计算资源时的终极方案 |

**选型建议**: 计算成本极高, 在项目有大量闲置计算资源时考虑, 否则不建议。

##### 4.1.E: 策略组合建议

| 场景 | 推荐方案 |
|------|---------|
| Agent Planner Prompt | DSPy 自动优化 |
| Skill 描述文本 | DSPy (结构化) |
| 宪法规则 | 人工 + LLM 改写 (安全敏感) |
| 非结构化 System Prompt | OPRO |
| 通用模板 | 人工 + LLM 改写 |

---

### 4.2 知识库 & 检索组件优化

#### 4.2.1 优化对象

| 子组件 | 优化内容 |
|--------|---------|
| 文档分块 (Chunking) | 块大小、重叠窗口、分块策略 |
| 嵌入模型 (Embedding) | 模型选择、维度、量化 |
| 索引策略 | 倒排索引、向量索引、混合索引 |
| 检索策略 | Top-K, 相似度阈值, 查询扩展 |
| 重排序 (Re-ranking) | Cross-encoder 重排, 多样性重排 |
| 知识库内容 | 文档质量、覆盖面、更新频率 |

#### 4.2.2 优化策略

##### 策略 4.2.A: 检索系统参数搜索

**方法**: 对影响检索质量的参数进行系统网格搜索:

```
搜索空间:
  - chunk_size: [128, 256, 512, 1024]
  - chunk_overlap: [0, 32, 64, 128]
  - top_k: [3, 5, 10, 20]
  - retrieval_mode: [dense, sparse, hybrid]
  - re_ranking: [true, false]
  - query_expansion: [none, simple, llm_based]

评估指标: RAGAS 框架的 Precision@k, Recall@k, NDCG@k
每次评估 = 在 AI Benchmark 子集 (T2/T3 检索密集型任务) 上运行
```

**优点**:
- 实现简单, 效果直接
- 与 EvoBench 的评测流程天然兼容

**缺点**:
- 搜索空间随参数增长指数爆炸
- 网格搜索在参数 > 5 时效率低

**改进 - 贝叶斯超参优化**: 使用 Optuna/Weights & Biases 等工具进行智能搜索。

##### 策略 4.2.B: 嵌入模型替换与微调

**方法**:
```
1. 候选模型收集: text-embedding-3-small, bge, e5, gte 等
2. 在内部代码库数据上评估各模型检索精度
3. 最优模型替换
```

**延伸 - 领域微调**: 在半年前的 OCB 代码库数据上, 用 Contrastive Learning 微调嵌入模型。

| 子策略 | 效果 | 成本 | 选型建议 |
|--------|------|------|---------|
| 模型替换 | 中等提升 (5-15%) | 低 (API 调用) | 优先尝试 |
| 领域微调 | 显著提升 (10-30%) | 高 (GPU 训练) | 长期投入 |

##### 策略 4.2.C: 查询理解与查询扩展

**核心思想**: 在将用户请求转换为检索查询时, 添加上下文以提升召回率。

**方法**:
```
原始查询: "资源配额"
→ 扩展查询: "资源配额 查询 用户态管理服务 domain 资源限制 CPU 内存"
```

**实现方案**:
| 方案 | 方法 | 效果 |
|------|------|------|
| 基于 TF-IDF 的关键词扩展 | 从领域词典提取相关词 | 通用, 可预测 |
| LLM 查询重写 | 用 LLM 将模糊查询改写为精确查询 | 灵活, 但成本高 |
| HyDE (假设文档嵌入) | 先生成假设答案, 用答案嵌入检索 | 检索精度提升显著 |

**选型建议**: HyDE 效果最佳, 但增加了一次 LLM Call。可作为检索优化的第二阶段。

##### 策略 4.2.D: 知识库内容质量提升

**核心思想**: 很多情况下, 问题不在检索机制, 而在文档内容本身。

**诊断方法**:
```
For each 检索失败案例:
  1. 检查 ground truth 相关文档是否在知识库中 → 不在 → 知识库覆盖率问题
  2. 在知识库中但检索不到 → 检索机制或文档表述问题
  3. 检索到但 LLM 未利用 → Prompt 或上下文管理问题
```

**优化手段**:
| 问题类型 | 解决方案 |
|---------|---------|
| 知识库缺少相关文档 | 自动分析未覆盖领域, 补充文档 |
| 文档表述不清晰 | 用 LLM 自动重写/摘要现有文档 |
| 文档过时 | 对比最新代码, 自动标记过时内容 |
| 文档粒度不当 | 基于检索命中模式自动调整分块 |

---

### 4.3 Skill / MCP 脚本代码优化

#### 4.3.1 优化对象

| 组件 | 优化内容 |
|------|---------|
| Skill 脚本 | 工具函数、分析脚本、转换脚本 |
| MCP 服务 | 服务接口、数据处理逻辑 |
| Harness 插件 | 自定义工具和适配器 |
| 自动化规则 | Git hooks, CI/CD 脚本 |

#### 4.3.2 优化策略

##### 策略 4.3.A: 基于执行反馈的自动化修复

**核心思想**: 利用 AgentFL 或类似框架, 通过失败的执行 Trace 自动定位并修复脚本代码。

**方法**:
```
For each 工具执行失败案例:
  1. 收集: 输入参数, 预期输出, 实际错误
  2. 定位: 分析错误栈 + Trace, 定位到具体代码行
  3. 修复: LLM 生成补丁
  4. 验证: 在隔离环境重放, 确认修复
  5. 回归: 在相关 test cases 上验证
```

**参考工具**: AgentFL, Self-Debugging, CodeR, Repilot

**实现**: 可以在 EvoBench 内部实现一个"脚本修复器"模块。

##### 策略 4.3.B: 测试驱动脚本优化

**核心思想**: 为每个 Skill/MCP 脚本自动生成测试用例, 通过测试覆盖率驱动优化。

**方法**:
```
1. 解析 Skill 脚本代码 → 生成单元测试
2. 运行测试 → 收集失败用例和覆盖率
3. 对低覆盖率/失败区域 → LLM 优化脚本
4. 迭代直到覆盖率和通过率达到阈值
```

**优势**: 将 Skill 代码质量提升转化为可量化的指标, 融入 EvoBench 闭环。

##### 策略 4.3.C: 脚本版本管理与消融

**方法**: 为每个 Skill 维护多个实现版本 (v1, v2, ...), 通过 A/B 测试选择最优版本。

**集成方式**: 在 EvoBench 中注册 Skill 版本, 每次运行记录使用的版本号和结果, 自动比较。

#### 4.3.3 策略选型建议

| 场景 | 推荐策略 |
|------|---------|
| 脚本执行出错 | 4.3.A 自动修复 |
| 脚本逻辑不完整 | 4.3.B 测试驱动 |
| 多个脚本版本 | 4.3.C A/B 测试 |
| 脚本性能问题 | 4.3.B 覆盖分析 + 手动优化 |

---

### 4.4 Agent 配置优化

#### 4.4.1 优化对象

| 参数 | 影响范围 |
|------|---------|
| LLM 模型选择 | 质量、速度、成本 |
| Temperature / Top-p | 创造性与确定性平衡 |
| Max tokens | 输出长度限制 |
| Tool choice / 工具可见性 | Agent 的工具选择行为 |
| 最大迭代步数 | 探索深度与效率 |
| 思考预算 (Thinking Budget) | 推理深度 (仅 reasoning 模型) |

#### 4.4.2 优化策略

##### 策略 4.4.A: 贝叶斯超参优化

**方法**: 在 AI Benchmark 子集上对不同配置组合进行搜索。

```
优化目标: 最大化 F1 = 2 × (得分 × 成本效率) / (得分 + 成本效率)
搜索算法: Tree-structured Parzen Estimator (TPE)
搜索空间:
  - model: {claude-sonnet, claude-opus, gpt-4o, deepseek-coder}
  - temperature: [0.0, 1.0]
  - max_tokens: [1024, 4096, 8192]
  - tool_choice: {auto, required, specific}
```

**工具推荐**: Optuna, Hyperopt, Weights & Biases Sweeps

##### 策略 4.4.B: 动态配置自适应

**核心思想**: 根据不同任务类型自动选择配置, 而非使用统一配置。

**方法**:
```
For each task:
  task_type = classify_task(task_description)
  config = config_map[task_type]
  # config_map 通过 EvoBench 的优化器自动学习
```

**示例映射**:
| 任务类型 | 推荐模型 | Temperature | Max Tokens |
|---------|---------|------------|------------|
| T1 代码知识 | Claude Opus | 0.1 | 2048 |
| T2 需求理解 | Claude Sonnet | 0.3 | 2048 |
| T3 影响预测 | Claude Opus | 0.2 | 4096 |
| T4 方案设计 | Claude Opus | 0.5 | 8192 |
| T5 编码实现 | DeepSeek Coder | 0.1 | 8192 |

---

### 4.5 记忆 & 宪法组件优化

#### 4.5.1 记忆组件优化

| 优化方向 | 方法 |
|---------|------|
| 记忆粒度 | 调整记忆片段的大小和提取精度 |
| 记忆衰减策略 | 优化时间衰减函数, 平衡近期/远期记忆 |
| 记忆融合 | 如何合并相似记忆, 去除冗余 |
| 记忆检索触发 | 什么情况下触发记忆检索, 避免过度检索 |

**自动优化方案**: 通过 EvoBench Trace 分析"记忆利用率"(最终答案中包含的记忆信息比例), 调整记忆参数。

#### 4.5.2 宪法 (Constitutional AI) 优化

| 优化方向 | 方法 |
|---------|------|
| 过度约束检测 | 分析因宪法过滤导致的"合理输出被拒绝"案例 |
| 约束遗漏检测 | 分析输出中违反安全/合规要求但未被捕获的案例 |
| 规则简化 | 合并冗余规则, 减少 token 占用 |
| 优先级排序 | 基于历史冲突频率自动调整规则的优先级 |

**自动优化方案**:
```
For each constitutional 失败案例:
  - 类型 A (过度约束): 放宽/删除该条规则
  - 类型 B (约束遗漏): 新增或强化该条规则
  - 类型 C (规则冲突): 引入优先级排序或规则分解
```

---

### 4.6 元学习 & 跨任务迁移

#### 4.6.1 失败模式知识库

**核心思想**: 将历史归因结果形成"失败模式知识库", 新任务遇到类似的失败模式时, 直接应用已知优化。

**结构**:
```yaml
failure_patterns:
  - pattern_id: fp_001
    signature: "RetrievalRecall < 0.5 且 QueryLength < 20 且 Domain=resource_mgmt"
    root_cause: "查询过短导致模糊匹配, 涉及 resource_mgmt 领域时尤其严重"
    fix: "针对 resource_mgmt 领域启用 HyDE 查询扩展"
    efficacy: 0.87  # 历史修复成功率
```

#### 4.6.2 跨任务学习

**方法**:
```
每次归因 + 优化完成 → 记录:
  - 任务特征 (领域、难度、任务类型)
  - 归因结论 (短板组件 + 失败模式)
  - 优化措施 (改变了什么, 怎么改的)
  - 优化效果 (得分提升幅度)

跨任务学习:
  - 新任务 → 特征提取 → 查找最相似的历史案例
  - 应用相似的优化策略
  - 预期效果: 优化速度提升 2-3 倍
```

#### 4.6.3 进化轨迹追踪

**核心思想**: 追踪 Harness 版本的进化过程, 识别有效的进化方向。

```yaml
evolution_history:
  - version: v1.0.0
    score: 65.2
    major_change: "初始版本"
  - version: v1.1.0
    score: 70.8
    delta: +5.6
    major_change: "优化知识库检索 (chunk_size 512→256, 启用 HyDE)"
    triggered_by: "归因分析: 检索 Precision@3 仅 0.33"
  - version: v1.2.0
    score: 73.5
    delta: +2.7
    major_change: "Agent Planner Prompt DSPy 优化"
    triggered_by: "LLM 归因: Agent 计划遗漏关键步骤"
```

---

## 5. 策略选型综合对比

### 5.1 Phase 2 诊断策略全面对比

| 策略 | 归因精度 | 实现成本 | 运行成本 | 可解释性 | 维护成本 | 推荐场景 |
|------|---------|---------|---------|---------|---------|---------|
| **A. 分级归因矩阵** | ★★★ | ★★★★ | ★★★★★ | ★★★★★ | ★★★ | 主力归因 (80% 案例) |
| **B. LLM-as-a-Judge** | ★★★ | ★★★★ | ★★★ | ★★★★ | ★★★★★ | 边界案例补充 |
| **B.v2. 多 LLM 投票** | ★★★★ | ★★★ | ★★ | ★★★★ | ★★★★ | 关键任务仲裁 |
| **C. 消融实验** | ★★★★★ | ★★ | ★★ | ★★★★ | ★★ | 定期深度分析 |
| **D. Shapley Value** | ★★★★★ | ★ | ★ | ★★★★ | ★ | 理论参考, 实践困难 |
| **E. 贝叶斯因果** | ★★★★ | ★★ | ★★★ | ★★★★ | ★★ | 数据充足后采用 |
| **F. 过程行为分析** | ★★★★ | ★★ | ★★★ | ★★★★★ | ★★ | Agent/LLM 深度分析 |
| **G. 集成方案(推荐)** | ★★★★★ | ★★★ | ★★★ | ★★★★★ | ★★★ | **最终推荐** |

### 5.2 Phase 3 自进化策略综合对比

| 策略 | 预期效果 | 实现成本 | 收敛速度 | 泛化能力 | 推荐场景 |
|------|---------|---------|---------|---------|---------|
| **4.1.A 人工+LLM 改写** | 中 | 低 | 快 | 强 | Prompt 初期优化 |
| **4.1.B DSPy 自动优化** | 高 | 中 | 中 | 中 | 结构化 Prompt |
| **4.1.C OPRO** | 高 | 中 | 慢 | 强 | 非结构化 Prompt |
| **4.1.D 进化搜索** | 很高 | 高 | 很慢 | 很强 | 有充足算力时 |
| **4.2.A 检索参数搜索** | 中-高 | 低 | 快 | 中 | 检索优化首选 |
| **4.2.B 嵌入模型替换** | 中 | 中 | 快 | 强 | 基线准备时 |
| **4.2.C 查询扩展 (HyDE)** | 高 | 中 | 快 | 强 | 检索优化推荐 |
| **4.2.D 知识库内容优化** | 高 | 中 | 中 | 强 | 持续进行 |
| **4.3.A 自动脚本修复** | 高 | 高 | 中 | 中 | Skill 质量维护 |
| **4.3.B 测试驱动优化** | 中-高 | 中 | 慢 | 中 | Skill 长期质量 |
| **4.4.A 贝叶斯配置搜索** | 中 | 低 | 快 | 强 | Agent 调优 |
| **4.4.B 动态配置自适应** | 高 | 中 | 中 | 强 | 多任务场景 |
| **4.5 记忆/宪法优化** | 中 | 中 | 中 | 中 | 安全与上下文相关 |
| **4.6 元学习跨任务** | 累积增长 | 高 | 长期 | 很强 | 长期自进化远景 |

---

## 6. 实施路线图

### Phase 0: 基础建设 (Month 1)

| 周 | 任务 | 产出 |
|----|------|------|
| W1-W2 | 完成 AI Benchmark v1 (详见 AI_benchmark_design.md) | 可评测的 Benchmark |
| W2-W3 | Harness 原子化重构: 定义组件接口, 植入 Trace | 组件化的 Harness |
| W3-W4 | 实现 AOP 切面追踪 + Trace 数据收集 | 全链路 Trace 系统 |
| W4 | 建立 Trace 存储和查询基础设施 | Trace 数据库 |

### Phase 1: 诊断能力建设 (Month 2)

| 周 | 任务 | 产出 |
|----|------|------|
| W5-W6 | 实现分级归因矩阵 (策略 A) | 基础归因引擎 |
| W6-W7 | 实现各组件影子评估 (Shadow Eval) | 影子评估模块 |
| W7-W8 | 实现 LLM-as-a-Judge 归因 (策略 B) | 补充归因模块 |
| W8 | 集成两层归因引擎, 实现报告输出 | 可运行诊断系统 v1 |

### Phase 2: 归因验证与校准 (Month 3)

| 周 | 任务 | 产出 |
|----|------|------|
| W9-W10 | 人工采样验证归因准确性, 修正规则矩阵 | 校准后的归因引擎 |
| W10-W11 | 实现过程行为分析 (策略 F) | 深度分析模块 |
| W11-W12 | 集成三层归因 (规则 + LLM + 行为) | 可运行诊断系统 v2 |
| W12 | 进行一次完整诊断评估, 产出修复建议 | 诊断报告示例 |

### Phase 3: 自进化引擎建设 (Month 4-5)

| 周 | 任务 | 产出 |
|----|------|------|
| W13-W14 | 实现 Prompt 自动优化 (DSPy + OPRO) | Prompt 优化器 |
| W14-W15 | 实现检索参数搜索 + 查询扩展 (HyDE) | 检索优化器 |
| W15-W16 | 实现自动脚本修复 (AgentFL 模式) | 代码修复器 |
| W16-W17 | 实现 Agent 配置搜索 (贝叶斯超参) | 配置优化器 |
| W17-W18 | 集成所有优化器, 实现自动调度 | 自进化引擎 v1 |
| W18-W20 | 完成第一轮全量自进化循环 (E1) | 进化报告 + 优化后的 Harness v2 |

### Phase 4: 元学习与深度自进化 (Month 6+)

| 周 | 任务 | 产出 |
|----|------|------|
| W21-W22 | 构建失败模式知识库 | 模式库 v1 |
| W22-W24 | 实现消融实验模块 (月频深度分析) | 深度分析工具 |
| W24+ | 持续运行 EvoBench 闭环, 累积进化轨迹 | 持续优化 |

---

## 7. 风险与应对

### 7.1 技术风险

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|---------|
| Benchmark Ground Truth 质量不佳导致归因偏差 | 中 | 高 | 建立严格的多专家审核流程; 每季度校准一次 |
| 归因引擎误判导致错误优化 | 中 | 中 | 集成方案通过多策略投票降低误判; 设置人工审核阈值 |
| Trace 数据量过大 | 高 | 低 | 采样策略 + 分层存储 (热/温/冷) |
| 自进化"原地踏步"(优化无改善) | 中 | 低 | 设置最小改善阈值; 若连续 2 轮无改善则触发深度分析 |
| 负优化 (优化后得分下降) | 低 | 高 | 强制回归验证; 保留前一版本回滚能力 |
| LLM-as-a-Judge 自身的偏差和幻觉 | 高 | 中 | 多数投票 + 规则验证; 定期校准 Judge 模型 |

### 7.2 工程风险

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|---------|
| 系统复杂度高, 维护困难 | 高 | 中 | 模块化设计; 清晰组件接口; 充分文档 |
| 计算资源需求大 | 高 | 中 | 分层运行 (全量 vs 快速); 规划计算预算 |
| 人工依赖 (专家审核) | 中 | 高 | 最小化人工环节; 自动化优先; 人工用于仲裁和校准 |

### 7.3 关键成功指标

- **归因准确率**: 归因引擎结论与人工专家判定的一致性 ≥ 85%
- **进化有效率**: 每轮自进化带来的正面提升率 ≥ 70%
- **综合得分提升**: 每 3 个月综合得分提升 ≥ 10%
- **归因覆盖率**: 归因引擎能定位的失败模式占比 ≥ 90%
- **负优化率**: 每轮自进化引入的退化率 ≤ 5%

---

## 附录 A: 术语表

| 术语 | 定义 |
|------|------|
| Harness | 包含 Agent、Skill、MCP、LLM 等的完整 AI 工程系统 |
| Trace | 一次请求在 Harness 各组件间的完整调用记录 |
| Span | Trace 中的一个最小单位, 代表一个组件的执行片段 |
| Shadow Evaluation | 非侵入式地对中间输出进行异步评估 |
| Attribution (归因) | 将端到端失败映射到具体组件的过程 |
| Ablation (消融) | 移除/替换特定组件以测量其影响 |
| Shapley Value | 合作博弈论中衡量各参与者贡献的方法 |
| OPRO | Optimization by PROmpting, 用 LLM 优化 Prompt |
| DSPy | 声明式编程框架, 自动优化 Prompt/LLM 调用 |
| HyDE | Hypothetical Document Embeddings, 假设文档嵌入 |
| AgentFL | Agent Fault Localization, 基于 Agent 的故障定位 |
| Chunking | 将文档分割为检索片段的过程 |
| NDCG | Normalized Discounted Cumulative Gain, 排序质量指标 |

## 附录 B: 参考文献

1. DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines (Khattab et al., 2023)
2. OPRO: Large Language Models as Optimizers (Yang et al., 2024)
3. Constitutional AI: Harmlessness from AI Feedback (Bai et al., 2022)
4. RAGAS: Automated Evaluation of Retrieval Augmented Generation (Shahul et al., 2024)
5. AgentFL: Scaling LLM-based Fault Localization to Project-Level Context (2024)
6. HyDE: Precise Zero-Shot Dense Retrieval without Relevance Labels (Gao et al., 2022)
7. Shapley Values for Feature Attribution: A Unified Approach to Explain Model Output (Lundberg & Lee, 2017)
8. Self-Debugging: Teaching LLMs to Debug Their Own Code (Chen et al., 2023)
9. SWE-Bench: Can Language Models Resolve Real-World GitHub Issues? (Jimenez et al., 2024)
10. FeatBench: A Benchmark for Feature-Level Software Engineering (2024)
11. CodeR: Issue Resolving with Multi-Agent and Task Graphs (Chen et al., 2024)
12. OpenTelemetry: Cloud Native Observability Framework (CNCF, 2023)

---

## 附录 C: EvoBench 最小可行性闭环 (MVP) 定义

**目标**: 在 3 个月内跑通第一个完整的自进化闭环。

**MVP 范围**:

| 组件 | MVP 规格 |
|------|---------|
| AI Benchmark | 20-30 个特性, 约 100 个任务 (已完成, 见 AI_benchmark_design.md) |
| Harness 分解 | 分解为 5-7 个主要组件 (简单包装, 不做大规模重构) |
| Trace | 关键组件 (Retriever, Planner, LLM Call) 植入 tracer |
| Shadow Eval | RAGAS (检索) + LLM-as-Judge (Planner + Output) |
| 归因引擎 | 分级归因矩阵 (策略 A) + 单 LLM 归因 (策略 B) |
| 自进化 | 仅实现 Prompt 优化 (4.1.B) + 检索参数搜索 (4.2.A) |
| 验证 | 全量 Benchmark 回归, 单次进化轮次 < 24 小时 |

**MVP 成功标准**:
- [ ] 归因引擎在 80% 的案例上给出正确的短板定位 (人工抽检确认)
- [ ] 完成至少一轮"评测 → 归因 → 优化 → 验证"的完整闭环
- [ ] 首轮自进化带来 ≥ 5% 的综合得分提升
- [ ] 负优化率 ≤ 10%

---

*文档版本: v1.0 | 最后更新: 2026-05-09*
