# 闭源微内核操作系统 AI 代码理解、设计与实现 Benchmark  
## —— 完整方案设计文档

---

## 目录

1. [Benchmark 定位与总体目标](#1-benchmark-定位与总体目标)
2. [评估能力矩阵](#2-评估能力矩阵)
3. [构建基本原则](#3-构建基本原则)
4. [任务类型体系](#4-任务类型体系)
5. [数据来源与提取流程](#5-数据来源与提取流程)
6. [数据组织与目录结构](#6-数据组织与目录结构)
7. [任务实例详细 Schema](#7-任务实例详细-schema)
8. [评分体系](#8-评分体系)
9. [保密与使用方案](#9-保密与使用方案)
10. [最小可行版本 MVP 规划](#10-最小可行版本-mvp-规划)
11. [实施路线图](#11-实施路线图)
12. [最终定义](#12-最终定义)

---

## 1. Benchmark 定位与总体目标

### 1.1 定位

本 benchmark 是一个**基于闭源微内核操作系统真实历史新特性构造的多维度 AI 能力评测基准**。其核心设定为：

- **时间轴划分**：以半年前稳定代码版本为基线，近半年内真实开发的新特性为测试素材；
- **信息不对称设计**：AI 只能看到基线代码库（旧版本）和新特性需求描述，**完全不能访问**新特性的真实实现代码、diff、commit、测试用例或评审记录；
- **评测对比依据**：以该新特性后来的真实落地代码、修改文件、接口变更、设计文档以及领域专家的标准答案作为 ground truth；
- **能力链覆盖**：从"理解现有系统"到"解读新需求"到"预测影响范围"到"设计方案"到"编码实现"形成完整闭环。

### 1.4 中文语言环境适配总则

本闭源操作系统的开发者团队包含中文开发环境背景，代码库中存在以下中文语言特征，benchmark 必须兼容：

1. **代码注释中文化**：代码中的注释、TODO、FIXME、HACK 等可能使用中文撰写；
2. **中文设计文档**：需求文档、设计方案、接口说明存在中文版本或中英混合版本；
3. **中文术语嵌入**：部分模块名、配置项、日志输出、错误提示可能包含中文拼音或中文缩写的命名风格；
4. **中文提交信息**：commit message、code review 记录可能使用中文；
5. **中英混合编码习惯**：部分代码中存在中文命名变量（汉语拼音或直接中文标识符）的情况。

> **本 benchmark 要求 AI 输出必须为中文（简体中文），以真实反映目标中文开发者的使用场景。同时评估 AI 对代码库中中文内容的准确理解能力。**

所有任务类型的输入（需求描述、上下文文档）默认使用中文编写，AI 的回答/方案/代码注释也以中文为默认语言。

### 1.2 本质定义

> **这是一个评测 AI 在真实、大型、闭源系统软件工程场景中综合认知与实现能力的 benchmark，而不仅是代码生成或填空测试。**

### 1.3 与通用代码 Benchmark 的关键区别

| 维度 | 通用代码 Benchmark | 本 Benchmark |
|------|-------------------|-------------|
| 代码可见性 | 开源、可训练的代码 | 闭源、模型无法预训练 |
| 任务形式 | 函数补全、bug修复、简单功能实现 | 系统级需求分析、影响预判、跨模块设计、受控实现 |
| 评测依据 | 测试用例通过率 | 真实实现 + 专家 oracle + 自动测试 |
| 架构复杂性 | 单文件或少量模块 | 微内核多组件协作系统 |
| 数据污染风险 | 高（训练数据可能包含测试集） | 极低（闭源代码天然隔离） |

---

## 2. 评估能力矩阵

本 benchmark 系统性评估 AI 在复杂系统软件工程中的**五大核心能力**。

### 2.1 T1 - Baseline 代码库知识理解能力

衡量 AI 仅凭阅读代码，能否准确掌握现有系统的架构、接口和运行机制。

| 能力子项 | 评估说明 | 微内核特殊要求 |
|---------|---------|---------------|
| **模块定位** | 判断某功能属于微内核核心、用户态服务、驱动服务、客户端库还是工具层 | 不能将所有能力默认归入内核 |
| **接口理解** | 理解系统调用、IPC 接口、服务 API、内部函数签名和契约 | 区分同步/异步 IPC、消息格式、错误码约定 |
| **数据结构理解** | 识别关键对象、句柄、权限令牌、资源描述符、引用计数字段 | 理解跨地址空间传递的数据结构要求 |
| **调用链追踪** | 能完整追踪一个典型请求从用户态入口经 IPC 到内核处理再返回的全链路 | 具备跨越多个用户态服务和内核态的端到端视角 |
| **职责边界** | 准确区分内核、用户态服务、客户端库、驱动服务之间的分工和约束 | 掌握微内核最小化原则，识别越界设计 |
| **中文注释理解** | 准确理解代码中中文注释、中文 TODO、中文文档注释的实际含义 | 中文注释可能涉及微内核特有术语的中文表达，需准确映射到对应英文概念 |

### 2.2 T2 - 新特性需求理解能力

衡量 AI 面对半结构化技术需求描述时，能否准确把握目标、边界和约束。

| 能力子项 | 评估说明 |
|---------|---------|
| **需求目标概括** | 用技术语言精准描述该特性最终要达成的系统行为和能力。默认要求使用中文输出 |
| **功能边界识别** | 明确区分必须支持的行为、可选扩展和明确不属此需求的内容 |
| **基线缺口分析** | 对照 baseline 代码，指出旧版本在机制、接口或数据层面缺少什么 |
| **约束识别** | 捕捉兼容性承诺、权限模型、服务隔离边界、配置开关要求等限制 |
| **非目标识别** | 避免错误地将需求扩大为不必要的大范围系统改造 |
| **中文需求语义理解** | 准确理解中文描述的技术需求，正确处理中英混排术语（如"通过 IPC 消息完成资源配额查询"），避免将非标准中文表达误解为技术概念 |

### 2.3 T3 - 影响范围预测能力（核心能力之一）

衡量 AI 在不接触真实实现的情况下，能否准确预见一个系统变更的"涟漪效应"。

| 能力子项 | 评估说明 |
|---------|---------|
| **模块命中** | 预测该特性会触及哪些内核模块、用户态服务或库组件 |
| **文件/组件预测** | 给出大致需要修改的目录、逻辑组件或文件类型 |
| **接口变化预测** | 判断是否需要新增或修改系统调用、IPC 消息类型、服务 API、客户端库接口 |
| **控制流变化** | 分析请求发起、内核处理、服务协调、响应返回的路径将如何变化 |
| **数据流变化** | 指出新增状态、字段、资源计数、配置数据的产生、传递和持久化路径 |
| **兼容性影响** | 判断旧版本客户端、旧配置文件、旧行为约定是否会受到破坏 |

### 2.4 T4 - 方案设计推理能力

衡量 AI 能否在现有架构约束下，提出技术可行、职责合理、接口恰当的解决方案。

| 能力子项 | 评估说明 |
|---------|---------|
| **架构合理性** | 方案是否符合微内核最小化原则和该项目已有的架构风格 |
| **职责划分** | 计算逻辑、服务逻辑是否放置在正确的层次（内核 vs 用户态服务 vs 客户端库） |
| **接口设计** | 提出的 IPC 协议、API 签名、配置项是否必要且不过度 |
| **数据/控制流设计** | 能否清晰描述一次请求从发起到完成的完整生命周期 |
| **兼容性考虑** | 是否主动识别并保留了需要向后兼容的旧接口、旧配置、旧行为 |
| **风险识别** | 能否指出关键技术风险、竞争条件、资源泄漏可能、安全边界问题 |
| **测试建议** | 能否提出关键测试场景、边界条件和验证方法（不要求实现） |

### 2.5 T5 - 编码实现能力

衡量 AI 在给定详细设计方案的前提下，能否将设计准确转化为可编译、可运行、正确实现的系统代码。

| 能力子项 | 评估说明 |
|---------|---------|
| **方案到代码翻译** | 能否将设计文档中的模块划分、接口定义、数据流等准确转化为代码结构 |
| **API/头文件正确使用** | 是否正确引用基线中已存在的头文件、宏、接口约定，不引入不存在或错误的依赖 |
| **编码规范遵循** | 是否符合该代码库的命名约定、错误处理模式、内存管理策略、锁规则等惯用法 |
| **编译通过能力** | 在指定工具链和构建配置下，能否无错误、无关键警告地完整编译 |
| **功能正确性** | 生成的补丁应用后，能否通过针对该特性设计的独立测试套件 |
| **回归影响控制** | 是否保持存量功能不被破坏（作为辅助评价指标） |
| **多语言注释一致性** | 新增代码的中文/英文注释风格是否与目标模块现有注释习惯保持一致；新写注释是否准确反映了代码逻辑，不会因语言切换引入语义偏差 |

---

## 3. 构建基本原则

本 benchmark 的设计和运营严格遵循以下原则：

### 3.1 有效性 (Validity)
每道题目必须直接对应上述五项能力的至少一项，评分权重要与该能力在真实工程中的重要性匹配。题目设计需真实反映一线系统工程师的认知过程。

### 3.2 可靠性 (Reliability)
- 固定 baseline 代码版本快照；
- 统一需求描述格式和详细程度；
- 容器化沙盒评测环境（固定工具链版本、依赖库、系统镜像）；
- 同一模型多次评测的结果应具有高度一致性。

### 3.3 防数据泄露与公平性 (Contamination & Fairness)
- **根本优势**：代码库完全闭源，模型训练阶段绝对无法接触；
- 新特性真实实现、diff、评审记录、测试源码**永不对外暴露**；
- 若需第三方模型评测，采用"受控虚拟机 + 封闭 API"模式，仅返回评分，不返回任何源码或 oracle 数据；
- 所有需求描述、设计方案文档必须经过脱敏审查，确保不泄露实现线索。

### 3.4 任务独立性 (Independence)
- 选取的新特性之间在代码、接口、IPC 协议、配置项层面严格独立；
- 每个任务可以单独理解、单独完成、单独评分；
- 避免一个任务的失败导致连环失败，确保评分的纯净性。

### 3.5 可量化评估 (Quantitative & Automated)
- **T1~T4**：采用专家 checklist + 结构化命中统计 + LLM 辅助语义复核；
- **T5**：采用自动编译 + 自动化测试 + 规则性代码规范检查；
- 所有指标均转化为 0~100 的标准分。

### 3.6 充分且一致的上下文 (Context Adequacy)
每个任务提供相同的基线环境：
- 完整代码库快照；
- 架构概览文档（脱敏后）；
- 核心术语表和缩写说明；
- 构建系统和工具链说明（T5 任务）；
- 不给信息不对称导致的随机性空间。

### 3.7 挑战性与区分度 (Difficulty & Discrimination)
- 任务难度分三级：基础 (basic)、中等 (medium)、高级 (advanced)；
- 覆盖从"单一模块接口理解"到"跨内核-用户态 IPC 新特性设计"；
- 能够有效区分不同代际、不同规模的 AI 模型。

### 3.8 可维护与可扩展性 (Maintainability)
- 支持按季度或半年度更新 baseline 和新增特性；
- 模块化的数据组织，便于按领域、难度、任务类型灵活组合评测；
- 评分器设计为可插拔架构。

### 3.9 中文语言兼容性 (Chinese Language Compatibility)
- 所有任务类型的输入（需求描述、设计文档、架构概览）默认使用**中文（简体）**编写；
- AI 的回复/输出默认要求使用**中文**，以真实反映中文开发者的使用体验；
- 代码库中的中文注释、中文命名、中文提交信息等均需纳入理解评估范围；
- benchmark 数据（需求文档、oracle 标注等）的中文表达须经过至少两名中文母语专家审核，确保语义准确无歧义；
- 评分时不要求 AI 使用文言文或特定中文风格，简体中文的技术性表达即可。

---

## 4. 任务类型体系

### 4.1 六类任务概览

```
┌─────────────────────────────────────────────────────────────┐
│                       AI Benchmark                          │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           输入：Baseline Codebase (半年前)            │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                                │
│         ┌──────────────────┼──────────────────┐            │
│         │                  │                  │            │
│    ┌────▼────┐       ┌────▼────┐       ┌────▼────┐        │
│    │   T1    │       │   T2    │       │   T3    │        │
│    │ 代码库  │       │ 需求    │       │ 影响    │        │
│    │ 知识理  │       │ 理解    │       │ 范围    │        │
│    │ 解      │       │         │       │ 预测    │        │
│    └────┬────┘       └────┬────┘       └────┬────┘        │
│         │                  │                  │            │
│         │           ┌──────▼──────┐           │            │
│         │           │     T4      │           │            │
│         └───────────►  方案设计   ◄───────────┘            │
│                     │   预测      │                        │
│                     └──────┬──────┘                        │
│                            │                                │
│                     ┌──────▼──────┐     ┌───────────────┐  │
│                     │     T5      │     │    T6         │  │
│                     │  编码实现   │     │  自进化能力    │  │
│                     │ (给定设计)  │     │  评估         │  │
│                     └─────────────┘     └───────┬───────┘  │
│                                                  │          │
│  评测依据：真实实现代码 + 真实修改路径 + 专家 Oracle      │
└─────────────────────────────────────────────────────────────┘
```

**T6 定位**：T6 不属于核心"理解-设计-实现"能力链路，而是**元能力 (meta-capability)** 评估——衡量 AI 在迭代反馈、知识迁移和自主进化方面的潜力，与本 benchmark 辅助 AI Agent 自进化的战略目标直接对应。

### 4.2 任务类型详细定义

#### T1 - 代码库知识理解任务

| 属性 | 说明 |
|------|------|
| **目标** | 评测 AI 对 baseline 代码库本身的理解深度 |
| **输入** | baseline 代码库（含中文注释/文档）+ 针对现有代码的技术问题（中文） |
| **输出** | 相关模块、关键接口、调用链路径、核心数据结构、职责划分说明（中文） |
| **评测依据** | baseline 代码 + 专家编写的标准答案 |
| **示例问题** | "在当前 baseline 中，一个资源对象的创建、引用传递、所有权转移和释放的完整生命周期是怎样的？涉及哪些模块和 IPC 消息？" |
| **输出格式** | 结构化文本（中文），按子问题逐项回答 |

#### T2 - 需求理解任务

| 属性 | 说明 |
|------|------|
| **目标** | 评测 AI 对新特性目标和边界的把握能力 |
| **输入** | baseline 代码库 + 新特性需求描述文档（中文） |
| **输出** | 需求目标概括、功能边界声明、基线缺口分析、可能涉及模块、不应改变的现有行为列表（中文） |
| **评测依据** | 真实新特性实现 + 真实设计文档 + 专家标注的关键评分点 |
| **关键考察** | 能否准确区分"这个特性必须做什么"和"这个特性不需要做什么"；能否准确理解中文技术需求中的专业术语表达 |

#### T3 - 影响范围预测任务

| 属性 | 说明 |
|------|------|
| **目标** | 评测 AI 对系统变更涟漪效应的预见能力 |
| **输入** | baseline 代码库 + 新特性需求描述文档 |
| **输出** | 可能涉及的模块、可能修改的文件/组件类型、可能的接口/IPC 变更、控制流变化、数据流变化、兼容性影响评估 |
| **评测依据** | 真实修改文件列表、真实新增/修改符号、真实接口变更文档、专家 oracle |
| **关键考察** | 命中真实修改的关键模块和接口，同时不过度扩散到无关模块 |

#### T4 - 方案设计预测任务

| 属性 | 说明 |
|------|------|
| **目标** | 评测 AI 能否提出技术上可行、架构上合理的设计方案 |
| **输入** | baseline 代码库 + 新特性需求描述文档 |
| **输出** | 完整设计方案：需求理解、现状分析、涉及模块、推荐方案、接口/IPC/配置变更、数据结构变化、控制流/数据流、兼容性考虑、风险、测试建议 |
| **评测依据** | 真实实现方案、真实代码变更、真实接口设计、专家评分 rubric |
| **关键考察** | 方案与原架构一致性、职责层次正确性、接口设计恰当性 |

#### T5 - 编码实现任务

| 属性 | 说明 |
|------|------|
| **目标** | 评测 AI 将详细设计文档转化为可编译、正确运行代码的能力 |
| **输入** | baseline 代码库 + 由专家编写的完整技术设计方案文档（中文，不包含具体代码实现细节，但包含接口定义、数据结构和算法描述） |
| **输出** | 一个 git diff 格式的补丁文件，包含所有需要的代码修改。新增注释语言风格应与目标模块保持一致（中文模块保持中文注释，英文模块保持英文注释） |
| **评测依据** | 自动化编译脚本、自动化测试套件（黑盒/灰盒）、回归测试、代码规范检查 |
| **关键考察** | 编译成功率、功能测试通过率、代码规范合规性、回归影响程度、中文注释与代码逻辑的一致性 |

#### T6 - 自进化能力评估任务

T6 是一组元能力评估任务，衡量 AI 在面对系统工程问题时的**迭代改进、知识迁移和自主进化能力**。

##### T6.1 迭代改进（Iterative Improvement）

| 属性 | 说明 |
|------|------|
| **目标** | 评测 AI 在收到专家反馈后能否有效改进已有方案 |
| **输入** | baseline 代码库 + 新特性需求 + AI 之前生成的 v1 方案 + 模拟 Code Review 的专家反馈 |
| **输出** | 改进后的 v2/v3 方案 + 改进说明 |
| **评测依据** | v1->v2 改进幅度、是否准确响应了反馈中的关键问题、是否引入了新错误 |
| **关键考察** | 接受反馈的能力、改进的质量、迭代收敛速度 |
| **难度** | Medium |

##### T6.2 知识迁移（Knowledge Transfer）

| 属性 | 说明 |
|------|------|
| **目标** | 评测 AI 能否从先前任务中学习并应用到后续相关任务 |
| **输入** | 连续 2~3 个递增难度的相关特性任务 + 各任务的真实 oracle 对前一任务质量的反馈 |
| **输出** | 每个任务的方案 + 跨任务总结（学到了什么、下次如何改进） |
| **评测依据** | 后续任务相比首个任务的质量提升趋势、错误重复率下降、经验总结的准确性 |
| **关键考察** | 技能迁移效率、错误模式自识别能力 |
| **难度** | Medium-Hard |

##### T6.3 自主进化（Self-Evolution）— 长期目标

| 属性 | 说明 |
|------|------|
| **目标** | 评测 AI 在开放式场景中自主发现问题、设计方案、实现并自验证的全流程能力 |
| **输入** | baseline 代码库 + 开放式改进目标（如"提升系统的可观测性"） |
| **输出** | AI 自行识别问题 → 设计方案 → 实现 → 自验证的完整报告 |
| **评测依据** | 问题发现质量、方案完整性、实现正确性、自我验证的充分性 |
| **关键考察** | 自主性、系统性思维、完整进化链条 |
| **难度** | Advanced |

### 4.3 任务之间的逻辑关系

```
同一 Feature
   │
   ├── T2 需求理解 ──── 独立评测
   │
   ├── T3 影响范围预测 ─ 独立评测
   │
   ├── T4 方案设计预测 ─ 独立评测
   │
   └── T5 编码实现 ──── 使用专家编写的标准设计方案作为输入
                       (不依赖 AI 在 T4 中的输出)

跨 Feature（T6 自进化能力）
   Feature_A T2/T3/T4 v1 ──反馈──> Feature_A v2 ──反馈──> v3 ... (T6.1 迭代改进)
   Feature_1 ──> Feature_2 (相关递增) ──> Feature_3      (T6.2 知识迁移)
   基线 + 开放目标 ──> 自主探索 ──> 方案 ──> 实现         (T6.3 自主进化)
```

**说明**：
- T5 使用由专家基于真实实现反推编写的标准设计方案，而非 AI 在 T4 中生成的方案。
- T6 是元能力评估，不依赖单一 Feature 的 ground truth，而是评测 AI 在**迭代和迁移过程中的行为质量**。
- T6 的评分 oracle 包括：专家对改进幅度的判定、迭代间的质量差异分析、知识迁移效率的量化指标。

### 4.5 不纳入当前阶段的任务（明确边界）

| 任务类型 | 排除原因 |
|---------|---------|
| 给定 diff 的解释/分析任务 | 会暴露真实实现，违背核心设定 |
| 性能调优任务 | 评测噪声大，难以自动化且一致性差 |
| 安全攻防/漏洞挖掘 | 保密风险高，评分主观性强 |
| 长期历史演进分析 | 超出"近半年"数据窗口，任务构造复杂 |
| 多特性交叉依赖协同实现 | 违反独立性原则，当前阶段暂不纳入 |

**说明**：原"AI 自行编译/调试/迭代修复"已部分纳入 T6 自进化能力评估。T6 采用受控迭代模式（专家提供模拟反馈），而非完全开放的自编译自调试，在可控范围内衡量 AI 的迭代改进能力。

---

## 5. 数据来源与提取流程

### 5.1 整体流程概览

```
Step 1     Step 2       Step 3       Step 4       Step 5
确定       收集近半年    筛选独立      构造AI        构造
Baseline   候选新特性   可用的新特性   可见输入      Private Oracle
───►       ───►        ───►         ───►         ───►

  Step 6       Step 7       Step 8       Step 9
  专家标注     脱敏与       生成任务      形成最终
  评分点       质量检查     实例          Benchmark 数据集
  ───►        ───►        ───►         ───►
```

### 5.2 详细步骤

#### Step 1：确定 Baseline

**操作清单**：
1. 选取半年前最近一个正式发布的稳定标签（tag），记为 `BASELINE_TAG`；
2. 导出该标签的完整源码快照到 `baseline/` 目录；
3. 确保包含完整的构建系统（Makefile、Kconfig 或等价配置）；
4. 提取并脱敏架构文档、核心接口列表、IPC 协议定义、主要数据结构说明；
5. 验证该版本处于可构建、可测试的完整状态。

**选版约束**：
- 必须是稳定发布点或稳定分支截点；
- 不能处于大规模重构的中间状态；
- 后续近半年的新特性可以清晰映射到该版本作为基线；
- 最好有对应的正式发布说明文档。

#### Step 2：收集近半年候选新特性

**数据来源**：
1. **版本控制系统**：`BASELINE_TAG..HEAD` 之间的分支、合并提交、PR/MR 记录；
2. **需求管理系统**：目标时间范围内标记为"新功能"的需求条目；
3. **发布说明**：近半年各小版本的 release note 中列出的新功能；
4. **设计文档库**：新增的设计文档或 RFC 文档；
5. **测试用例库**：新增的功能测试文件。

**收集方法**：
1. 遍历所有合并到主分支的非 bugfix 合并请求；
2. 提取每个合并请求的：标题、描述、修改文件列表、新增/修改的符号；
3. 建立候选特性清单，初步标记每个特性的类型、涉及模块、变更规模。

#### Step 3：筛选独立可用的新特性

**保留标准（全部满足）**：
- [ ] 是新增功能或新增接口能力；
- [ ] 有真实且完整的实现代码（已合并到主线）；
- [ ] 可以形成清晰、独立的需求描述；
- [ ] 有相关的测试用例（或者可以提取/构造测试）；
- [ ] 不属于高安全敏感或高度保密模块；

**剔除标准（满足任意一条即剔除）**：
- [ ] 纯 bugfix（行为修正，非新功能）；
- [ ] 纯代码重构（结构改变，功能不变）；
- [ ] 纯格式化/注释/文档修正；
- [ ] 大规模目录或文件重命名/迁移；
- [ ] 依赖链复杂（依赖基线版本之后新增的其他特性）；
- [ ] 需求描述严重缺失或无法脱敏；
- [ ] 涉及安全防御机制的核心实现；
- [ ] 无法形成清晰评分点。

**独立性验证（每个保留特性必须通过）**：

| 验证维度 | 验证方法 |
|---------|---------|
| 需求独立性 | 该特性的需求能否在不引用其他新特性的前提下完整描述？ |
| 代码依赖独立性 | 该特性的修改文件/符号集与其他候选特性的修改文件/符号集是否存在重叠？使用符号依赖分析工具交叉检查。 |
| IPC/接口独立性 | 该特性是否依赖其他新特性新增的 IPC 消息类型或 API 接口？ |
| 服务依赖独立性 | 该特性是否依赖其他新增用户态服务或驱动服务？ |
| 配置独立性 | 该特性是否依赖其他新增的配置项或构建选项？ |
| 功能可验证独立性 | 将该特性单独 cherry-pick 到 baseline 上，能否编译通过且功能测试通过？（可能需要微调以剔除偶发耦合，如公共头文件冲突） |

#### Step 4：构造 AI 可见输入

**需求描述文档 (`requirement_clean.md`) 编写规范**：

1. **目标描述**：
   - 用简洁的中文描述该特性要实现什么能力；
   - 说明该特性解决什么实际问题或满足什么使用需求；
   - 允许中英混排的技术术语，如"IPC 消息"、"capability（能力权能）"。

2. **功能范围**：
   - 列出核心需要支持的行为（用 `MUST`/`SHOULD` 表示）；
   - 明确排除的行为边界（用 `NOT IN SCOPE` 表示）；
   - 各项功能描述使用中文撰写。

3. **约束条件**：
   - 兼容性要求；
   - 权限模型约束；
   - 服务隔离要求；
   - 性能关键路径限制（如适用）。

4. **编写禁忌**：
   - ❌ 不要直接指定具体函数名、变量名、结构体字段名；
   - ❌ 不要透露具体的 IPC 消息类型编号或接口编号；
   - ❌ 不要给出实现算法细节；
   - ❌ 不要引用真实 commit hash 或内部系统信息。

5. **中文表达规范**：
   - 使用简体中文，避免生僻字和文言文表达；
   - 技术术语首次出现时可标注英文原文，如"能力权能（capability）"；
   - 保持术语在全文中的一致性，同一概念不要混用中英文不同表述；
   - 避免有歧义的中文表达，必要时补充英文原文消除歧义。`

**示例（资源配额查询特性）**：
```markdown
# 需求：用户态管理服务资源配额查询

## 目标
当前系统支持对服务域进行资源配额分配，但缺少运行时查询配额使用情况的能力。
本特性需提供一种机制，允许用户态管理工具查询指定服务域的当前资源配额和使用量。

## 功能范围
- MUST：支持查询指定服务域的 CPU、内存、IO 资源的配额上限和当前使用量
- MUST：返回结果包含资源类型、配额上限、当前使用值
- SHOULD：支持批量查询多个服务域
- NOT IN SCOPE：不改变资源分配的流程和行为
- NOT IN SCOPE：不新增资源控制或限流机制

## 约束
- 需要遵循现有的权限检查模型
- 查询接口需通过已有的管理通道提供
- 不能破坏现有服务域的资源配置格式
```

**中文版本示例（实际使用的格式）**：
```markdown
# 需求：用户态管理服务资源配额查询

## 目标
当前系统支持对服务域（service domain）进行资源配额分配，但缺少运行时查询配额使用情况的能力。
本特性需提供一种机制，允许用户态管理工具查询指定服务域的当前资源配额和使用量。

## 功能范围
- MUST：支持查询指定服务域的 CPU、内存、IO 资源的配额上限（limit）和当前使用量（usage）
- MUST：返回结果包含资源类型、配额上限、当前使用值
- SHOULD：支持批量查询多个服务域
- NOT IN SCOPE：不改变资源分配的流程和行为
- NOT IN SCOPE：不新增资源控制或限流机制

## 约束
- 需遵循现有的权限检查模型（capability-based）
- 查询接口需通过已有的管理 IPC 通道提供
- 不能破坏现有服务域的资源配置格式
- 接口定义需与现有中英混排代码风格一致

## 术语说明
- 服务域（service domain）：资源隔离的基本单位，等同于 capability 体系中的 protected process
- 配额 (quota）：资源上限配置值
- 使用量（usage）：当前实时消耗值
```

**T5 设计方案文档 (`design_for_implement.md`) 编写规范**：

由专家基于真实实现反推编写，要求如下：
1. 描述"做什么功能"和"为什么需要"；
2. 指明实现层次：在内核核心、用户态服务、客户端库还是驱动服务；
3. 定义接口契约：IPC 消息结构、API 参数和返回值（可用抽象类型描述）；
4. 描述数据流和控制流（可用文字+流程图描述）；
5. 给出错误处理策略；
6. 指明兼容性约束；
7. **严禁**：直接给出可复制的代码片段、精确的变量名、文件路径。

#### Step 5：构造 Private Oracle

为每个特性建立私有评测数据集，详见第 6 节数据组织。

#### Step 6：专家标注评分点

**操作要求**：
- 由 2~3 名对该系统熟悉的资深工程师独立标注；
- 对每个任务类型（T2/T3/T4/T5）分别编写评分 rubric；
- 标识"必须命中"的关键点（关键词、模块名、接口类型）；
- 标识"明显错误"的扣分项（如将用户态服务逻辑错误放入内核）；
- 交叉审核并达成一致。

#### Step 7：脱敏与质量检查

**脱敏检查清单**：
- [ ] 移除所有内部 IP 地址、主机名、路径；
- [ ] 移除开发者姓名、邮箱、工号；
- [ ] 移除项目代号、客户名、产品名（用通用代号替代）；
- [ ] 移除安全密钥、证书、认证令牌信息；
- [ ] 检查需求描述中是否透露了具体实现线索；
- [ ] 确认中文表达的语义准确性，避免词语歧义或表意不清；
- [ ] 确认中文文档中嵌入的系统内部缩写、术语已添加必要的中文注解；

**质量检查清单**：
- [ ] 每个需求文档是否清晰完整、无歧义？
- [ ] 每个 private oracle 是否准确对应真实实现？
- [ ] 评分 rubric 是否具有可操作性？
- [ ] T5 测试套件是否能在仅应用参考补丁时全部通过？
- [ ] T5 测试套件是否在不应用任何补丁时预期失败？

#### Step 8：生成标准化任务实例

按照第 7 节的 schema 将每个任务序列化为 JSON/JSONL 格式。

#### Step 9：形成 Benchmark 数据集

按以下结构组织最终数据集：

```
benchmark_root/
├── baseline/
│   ├── src/                     # 完整基线代码
│   ├── build_system/            # 构建系统文件
│   └── docs/                    # 脱敏后的架构文档
├── config/
│   ├── glossary.md              # 术语表
│   ├── arch_overview.md         # 架构概览
│   └── domain_tags.yaml         # 标签定义
├── tasks/
│   ├── t1_knowledge/            # T1 任务集
│   │   ├── t1_001.json
│   │   └── ...
│   ├── t2_requirement/          # T2 任务集
│   │   ├── feat_0001_T2.json
│   │   └── ...
│   ├── t3_impact/               # T3 任务集
│   │   ├── feat_0001_T3.json
│   │   └── ...
│   ├── t4_design/               # T4 任务集
│   │   ├── feat_0001_T4.json
│   │   └── ...
│   └── t5_implementation/       # T5 任务集
│       ├── feat_0001_T5.json
│       └── ...
├── features_data/               # 以特性为单位组织的数据
│   ├── feat_0001/
│   │   ├── input/
│   │   │   ├── requirement_clean.md
│   │   │   └── design_for_implement.md  # 仅 T5
│   │   ├── private_oracle/
│   │   │   ├── actual_patch.diff
│   │   │   ├── actual_changed_files.txt
│   │   │   ├── actual_changed_symbols.json
│   │   │   ├── actual_interface_changes.md
│   │   │   ├── actual_design_summary.md
│   │   │   ├── tests.patch
│   │   │   ├── build.sh
│   │   │   ├── regression_tests.patch
│   │   │   └── scoring_rubric.yaml
│   │   └── metadata.json
│   └── ...
├── evaluation/
│   ├── harness/                 # 评测执行引擎
│   │   ├── runner.py
│   │   ├── sandbox/
│   │   └── scorer.py
│   └── rubric_engine/           # 评分规则引擎
└── results/
    └── templates/               # 结果报告模板
```

---

## 6. 数据组织与目录结构

### 6.1 单个 Feature 的完整数据组织

```
features_data/
└── feat_0042/
    ├── input/                             # AI 可见
    │   ├── requirement_clean.md           # 需求描述（T2/T3/T4 使用）
    │   └── design_for_implement.md        # 专家编写的标准设计（T5 使用）
    │
    ├── private_oracle/                    # 绝密，仅供评测使用
    │   ├── actual_patch.diff              # 真实实现 diff
    │   ├── actual_changed_files.txt       # 真实修改文件列表（相对于仓库根）
    │   ├── actual_changed_symbols.json    # 真实新增/修改的符号
    │   ├── actual_interface_changes.md    # 真实接口变化说明
    │   ├── actual_design_summary.md       # 真实设计摘要
    │   ├── tests.patch                    # 测试补丁（黑盒/灰盒化处理后）
    │   ├── build.sh                       # 编译脚本（不对外暴露）
    │   ├── regression_tests.patch         # 存量回归测试补丁（可选）
    │   └── scoring_rubric.yaml            # 统一评分规则
    │
    └── metadata.json                      # 特性元信息
```

### 6.2 metadata.json 完整字段定义

```json
{
  "feature_id": "feat_0042",
  "title": "Add resource quota query for management service",
  "description_short": "Provide a mechanism for user-mode management tools to query resource quota usage",
  
  "baseline_commit": "abc123def456",
  "feature_commits": [
    "def456789abc",
    "ghi789012def"
  ],
  
  "creation_date": "2025-01-15",
  "merge_date": "2025-02-20",
  
  "domain_tags": ["user_service", "ipc", "resource_mgmt"],
  "architecture_layer": "user_service",
  
  "difficulty": "medium",
  "estimated_complexity": {
    "files_touched": 5,
    "new_symbols": 12,
    "modified_symbols": 3,
    "new_ipc_messages": 2
  },
  
  "independence_verified": true,
  "independence_notes": "No dependency on other half-year features. Shares only common headers.",
  
  "task_types_available": [
    "T2_requirement_understanding",
    "T3_impact_prediction", 
    "T4_design_prediction",
    "T5_code_implementation"
  ],
  
  "test_suite_info": {
    "total_test_cases": 8,
    "test_type": "functional",
    "estimated_runtime_seconds": 120,
    "requires_hardware": false
  },
  
  "sensitivity": "internal",
  "can_be_external_sample": false,
  "reviewers": ["expert_001", "expert_002"],
  "last_reviewed": "2025-06-01",
  
  "language": {
    "input_language": "zh-CN",
    "expected_output_language": "zh-CN",
    "code_comments_languages": ["zh-CN", "en"],
    "glossary_available": true
  }
}
```

### 6.3 领域标签体系 (domain_tags)

| 标签 | 含义 | 典型场景 |
|------|------|---------|
| `kernel_core` | 微内核核心 | 进程管理、IPC 引擎、中断分发、地址空间管理 |
| `ipc` | IPC 协议或消息接口 | 消息格式定义、端点管理、通道协议 |
| `user_service` | 用户态系统服务 | 文件系统服务、网络栈、资源管理服务 |
| `driver_service` | 驱动服务 | 设备驱动、总线适配、中断处理代理 |
| `client_lib` | 客户端库 | 用户态 API 封装、语言绑定、辅助工具库 |
| `resource_mgmt` | 资源管理 | 内存配额、CPU 分配、IO 带宽控制 |
| `memory` | 内存相关 | 页表管理、共享内存、缓存策略 |
| `scheduling` | 调度 | 线程调度、优先级继承、时间片管理 |
| `security` | 权限/隔离/访问控制 | 能力模型、权限检查、审计日志 |
| `config_boot` | 配置/启动/初始化 | 启动参数、配置文件解析、服务发现 |
| `tool` | 工具 | 调试工具、性能分析、管理控制台 |
| `test` | 测试相关 | 测试框架、测试桩、模拟环境 |

### 6.4 actual_changed_symbols.json 结构

```json
{
  "new_symbols": [
    {
      "name": "service_quota_query",
      "type": "function",
      "location": "src/services/resource_mgr/",
      "signature": "int service_quota_query(domain_id_t did, quota_info_t* out)",
      "visibility": "public_api"
    }
  ],
  "modified_symbols": [
    {
      "name": "handle_ipc_resource_request",
      "type": "function",
      "location": "src/services/resource_mgr/",
      "change_type": "added_branch",
      "description": "Added new IPC message type handling for quota queries"
    }
  ],
  "new_ipc_messages": [
    {
      "msg_type": "RES_QUOTA_QUERY",
      "msg_type_id": "0x3010",
      "direction": "client_to_service"
    },
    {
      "msg_type": "RES_QUOTA_RESPONSE",
      "msg_type_id": "0x3011",
      "direction": "service_to_client"
    }
  ],
  "new_config_items": [
    "resource_mgr.enable_quota_query"
  ]
}
```

### 6.5 scoring_rubric.yaml 结构

```yaml
feature_id: feat_0042
rubrics:
  
  T2_requirement_understanding:
    total_score: 100
    items:
      - dimension: requirement_goal
        max_score: 25
        key_points:
          - "Query resource quota usage at runtime"
          - "Return quota limit and current usage"
          - "Service domain as the query target"
        must_mention:
          - "resource quota"
          - "query"
        negative_points:
          - "Mentioning modification of allocation logic": -10
          
      - dimension: functional_boundary
        max_score: 20
        key_points:
          - "Query is read-only"
          - "Does not include resource allocation change"
          - "Does not include new cgroup or limit mechanisms"
        must_mention:
          - "read-only" or equivalent
          
      - dimension: baseline_gap
        max_score: 25
        key_points:
          - "No existing query interface"
          - "Resource data exists but is not exposed"
          - "Need IPC message extension"
          
      - dimension: likely_modules
        max_score: 20
        key_points:
          - "Resource management service (user-space)"
          - "IPC message definitions"
        penalty_for:
          - "Putting logic in kernel core": -10
          
      - dimension: unchanged_behaviors
        max_score: 10
        key_points:
          - "Allocation behavior unchanged"
          - "Existing clients unaffected"
          - "Boot/config compatibility maintained"

  T3_impact_prediction:
    total_score: 100
    items:
      - dimension: module_hit
        max_score: 25
        actual_modules: ["resource_mgr_service", "libresource_client"]
        scoring:
          exact_match: 25
          partial_match: 15
          wrong_major_module: -10
          
      - dimension: interface_ipc_hit
        max_score: 25
        actual_changes: ["new_ipc_msg_type", "extended_client_lib_api"]
        scoring:
          hit_all_major: 25
          hit_some: 15
          missed_all: 0
          
      - dimension: control_data_flow
        max_score: 20
        key_correct_paths:
          - "Client -> IPC request -> resource service -> query data -> IPC response -> Client"
        penalty_for:
          - "Going through kernel core unnecessarily": -10
          
      - dimension: compatibility
        max_score: 15
        must_mention:
          - "Old clients work unchanged"
          
      - dimension: over_prediction_control
        max_score: 15
        penalty_per_irrelevant_module: -5

  T4_design_prediction:
    total_score: 100
    items:
      - dimension: requirement_understanding
        max_score: 15
        
      - dimension: baseline_analysis
        max_score: 20
        
      - dimension: module_responsibility
        max_score: 20
        ideal_placement: "user-space resource service"
        penalty:
          kernel_core: -15
          
      - dimension: feasibility
        max_score: 20
        
      - dimension: interface_design        max_score: 15
        
      - dimension: risks_and_tests
        max_score: 10

  T5_code_implementation:
    total_score: 100
    items:
      - dimension: compilation_success
        max_score: 30
        scoring: "30 if first-compile success, 0 if failure"
        
      - dimension: feature_test_pass
        max_score: 50
        total_tests: 8
        scoring: "(passed_tests / total_tests) * 50"
        
      - dimension: regression_test_pass
        max_score: 10
        total_regression_tests: 45
        scoring: "(passed / total) * 10"
        
      - dimension: code_compliance
        max_score: 10
        checks:
          - "No use of forbidden APIs": 3
          - "Naming convention follows baseline": 3
          - "Error handling follows standard pattern": 2
          - "No memory leak pattern detected": 2
```

---

## 7. 任务实例详细 Schema

### 7.1 T1 - 代码库知识理解任务

```json
{
  "task_id": "t1_knowledge_001",
  "task_type": "baseline_knowledge",
  "feature_id": null,
  "difficulty": "basic",
  "domain_tags": ["kernel_core", "ipc"],
  "language": "zh-CN",
  
  "visible_input": {
    "baseline_commit": "abc123",
    "question": "在当前的 baseline 代码中，请完整追踪一个 capability（能力权能）的生命周期：它是如何被创建的、如何通过 IPC 在进程间传递、如何被撤销的？涉及哪些内核对象和数据结构？",
    "context_hints": [
      "请重点关注 capability 管理子系统",
      "考虑 IPC 消息传递中的 capability 转移机制"
    ],
    "output_format": [
      "involved_modules",
      "key_kernel_objects_and_structures",
      "creation_path_step_by_step",
      "transfer_path_step_by_step",
      "revocation_path_step_by_step",
      "resource_cleanup_and_release"
    ]
  },
  
  "private_oracle": {
    "expert_answer_summary": "private_oracle/expert_answer_t1_001.md",
    "scoring_rubric": {
      "involved_modules": {
        "must_include": ["capability_manager", "ipc_engine", "process_manager"],
        "max_score": 20
      },
      "key_structures": {
        "must_include": ["cap_t", "cap_object", "ipc_message"],
        "max_score": 20
      },
      "creation_path": {
        "must_include_steps": ["syscall_cap_create", "capability_stored_in_kernel", "handle_returned_to_process"],
        "max_score": 20
      },
      "transfer_path": {
        "must_include_steps": ["cap_serialize_to_ipc", "receiver_validation", "cap_deserialize", "reference_counting"],
        "max_score": 20
      },
      "revocation_path": {
        "must_include": ["cascade_revocation", "notification_to_holders"],
        "max_score": 10
      },
      "cleanup": {
        "must_include": ["reference_count_zero", "kernel_object_free"],
        "max_score": 10
      }
    }
  }
}
```

### 7.2 T2 - 需求理解任务

```json
{
  "task_id": "feat_0042_T2",
  "task_type": "requirement_understanding",
  "feature_id": "feat_0042",
  "difficulty": "medium",
  "domain_tags": ["user_service", "ipc", "resource_mgmt"],
  "language": "zh-CN",
  "depends_on": [],
  
  "visible_input": {
    "baseline_commit": "abc123",
    "requirement_doc": "features_data/feat_0042/input/requirement_clean.md",
    "optional_docs": [
      "config/arch_overview.md",
      "config/glossary.md"
    ]
  },
  
  "expected_output": {
    "format": "structured_markdown",
    "language": "zh-CN",
    "sections": [
      {
        "name": "requirement_goal",
        "description": "Summarize the technical goal of this feature in 2-3 sentences",
        "required": true
      },
      {
        "name": "functional_boundary",
        "description": "List what this feature MUST do and what it explicitly does NOT do",
        "required": true
      },
      {
        "name": "baseline_gap_analysis",
        "description": "Identify what is currently missing in the baseline that this feature requires",
        "required": true
      },
      {
        "name": "likely_affected_modules",
        "description": "List the modules likely involved, with brief justification",
        "required": true
      },
      {
        "name": "unchanged_behaviors",
        "description": "List existing behaviors that should be preserved unchanged",
        "required": true
      }
    ]
  },
  
  "private_oracle": {
    "rubric_ref": "features_data/feat_0042/private_oracle/scoring_rubric.yaml#T2_requirement_understanding",
    "expert_key_points": [
      "Must identify resource management service as the primary affected module",
      "Must recognize the need for a new query interface",
      "Must explicitly state that resource allocation behavior is unchanged",
      "Should identify that this is a read-only operation"
    ],
    "common_errors": [
      "Putting implementation logic in kernel core",
      "Changing the resource allocation flow",
      "Missing the need for a client library update",
      "Overcomplicating with new configuration mechanisms"
    ],
    "language_notes": {
      "input_language": "zh-CN",
      "expected_output_language": "zh-CN",
      "technical_term_handling": "中文需求中的英文技术术语（如 IPC、capability、quota）应保持原样，不应强行翻译"
    }
  }
}
```

### 7.3 T3 - 影响范围预测任务

```json
{
  "task_id": "feat_0042_T3",
  "task_type": "impact_prediction",
  "feature_id": "feat_0042",
  "difficulty": "medium",
  "domain_tags": ["user_service", "ipc", "resource_mgmt"],
  "language": "zh-CN",
  "depends_on": [],
  
  "visible_input": {
    "baseline_commit": "abc123",
    "requirement_doc": "features_data/feat_0042/input/requirement_clean.md",
    "optional_docs": [
      "config/arch_overview.md",
      "config/glossary.md"
    ]
  },
  
  "expected_output": {
    "format": "structured_markdown",
    "sections": [
      {
        "name": "affected_modules",
        "description": "List modules that will likely need modification, with reasoning for each",
        "required": true
      },
      {
        "name": "likely_files_or_components",
        "description": "Predict which directories, components, or file types will be touched",
        "required": true
      },
      {
        "name": "interface_changes",
        "description": "List any likely changes to system calls, IPC messages, service APIs, or client library interfaces",
        "required": true
      },
      {
        "name": "control_flow_changes",
        "description": "Describe how the request processing path will change, from entry to completion",
        "required": true
      },
      {
        "name": "data_flow_changes",
        "description": "Describe what new data is generated, transmitted, stored, or displayed",
        "required": true
      },
      {
        "name": "compatibility_concerns",
        "description": "Identify any backward compatibility risks or concerns",
        "required": true
      }
    ]
  },
  
  "private_oracle": {
    "actual_changed_files": "features_data/feat_0042/private_oracle/actual_changed_files.txt",
    "actual_changed_symbols": "features_data/feat_0042/private_oracle/actual_changed_symbols.json",
    "actual_interface_changes": "features_data/feat_0042/private_oracle/actual_interface_changes.md",
    "rubric_ref": "features_data/feat_0042/private_oracle/scoring_rubric.yaml#T3_impact_prediction",
    "hit_criteria": {
      "module_level": "Match by architectural layer and service name (exact not required)",
      "file_level": "Match by directory and component type",
      "interface_level": "Match by interface type (add new IPC, extend existing API, etc.)"
    }
  }
}
```

### 7.4 T4 - 方案设计预测任务

```json
{
  "task_id": "feat_0042_T4",
  "task_type": "design_prediction",
  "feature_id": "feat_0042",
  "difficulty": "medium",
  "domain_tags": ["user_service", "ipc", "resource_mgmt"],
  "language": "zh-CN",
  "depends_on": [],
  
  "visible_input": {
    "baseline_commit": "abc123",
    "requirement_doc": "features_data/feat_0042/input/requirement_clean.md",
    "optional_docs": [
      "config/arch_overview.md",
      "config/glossary.md"
    ]
  },
  
  "expected_output": {
    "format": "design_document",
    "sections": [
      {
        "name": "requirement_understanding",
        "description": "Brief restatement of the problem and goal",
        "max_words": 200
      },
      {
        "name": "baseline_analysis",
        "description": "Analysis of current system state: what exists, what's missing",
        "max_words": 300
      },
      {
        "name": "affected_modules",
        "description": "Modules impacted and their roles in the solution"
      },
      {
        "name": "proposed_design",
        "description": "Core design: where logic goes, how components interact",
        "max_words": 500
      },
      {
        "name": "interface_ipc_changes",
        "description": "Specific interface, IPC message, or configuration changes proposed"
      },
      {
        "name": "data_structure_changes",
        "description": "New or modified data structures and their purposes"
      },
      {
        "name": "control_and_data_flow",
        "description": "End-to-end request lifecycle under the new design"
      },
      {
        "name": "compatibility_considerations",
        "description": "How the design preserves backward compatibility"
      },
      {
        "name": "risks",
        "description": "Technical risks and potential issues"
      },
      {
        "name": "test_suggestions",
        "description": "Key test scenarios, edge cases, and validation approaches"
      }
    ]
  },
  
  "private_oracle": {
    "actual_design_summary": "features_data/feat_0042/private_oracle/actual_design_summary.md",
    "actual_patch": "features_data/feat_0042/private_oracle/actual_patch.diff",
    "rubric_ref": "features_data/feat_0042/private_oracle/scoring_rubric.yaml#T4_design_prediction",
    "architectural_constraints": [
      "Microkernel isolation must be maintained",
      "Resource service is user-space, not kernel",
      "Use existing IPC framework, don't invent new transport"
    ]
  }
}
```

### 7.5 T5 - 编码实现任务

```json
{
  "task_id": "feat_0042_T5",
  "task_type": "code_implementation",
  "feature_id": "feat_0042",
  "difficulty": "medium",
  "domain_tags": ["user_service", "ipc", "resource_mgmt"],
  "language": "zh-CN",
  "depends_on": [],
  
  "visible_input": {
    "baseline_commit": "abc123",
    "design_doc": "features_data/feat_0042/input/design_for_implement.md",
    "optional_docs": [
      "config/arch_overview.md",
      "config/glossary.md",
      "baseline/docs/api_reference.md"
    ],
    "build_environment": {
      "toolchain": "gcc-12.3.0",
      "build_system": "make",
      "target_arch": "x86_64",
      "build_command": "make all",
      "test_command": "make test FEATURE=feat_0042"
    }
  },
  
  "expected_output": {
    "format": "git_diff_patch",
    "instructions": "Generate a unified diff patch (git format-patch style) containing all code changes needed. Only include files under src/ and include/. Follow the existing code conventions in baseline.",
    "constraints": [
      "Use only APIs and headers available in the baseline",
      "Follow the naming conventions observed in nearby files",
      "Add appropriate error handling consistent with existing patterns",
      "Do NOT modify any existing public API signatures"
    ]
  },
  
  "private_oracle": {
    "test_patch": "features_data/feat_0042/private_oracle/tests.patch",
    "build_script": "features_data/feat_0042/private_oracle/build.sh",
    "regression_tests_patch": "features_data/feat_0042/private_oracle/regression_tests.patch",
    "reference_patch": "features_data/feat_0042/private_oracle/actual_patch.diff",
    "rubric_ref": "features_data/feat_0042/private_oracle/scoring_rubric.yaml#T5_code_implementation",
    "sandbox_config": {
      "image": "benchmark-sandbox:latest",
      "timeout_seconds": 600,
      "memory_limit_mb": 4096,
      "cpu_limit": 4,
      "network": "none"
    }
  }
}
```

---

## 8. 评分体系

### 8.1 评分哲学

1. **真实实现是参考答案，但不是唯一正确答案**：
   - 若 AI 提出了与真实实现等价或合理不同的方案，应获得相应分数；
   - 评分注重"是否可行、是否合理、是否遵循架构约束"，而非逐字符匹配。

2. **分层评分**：
   - **T1~T4**：以专家 checklist 为主，自动化命中统计为辅，LLM 语义复核作为第三道保险；
   - **T5**：以自动化编译/测试结果为主，代码规范检查为辅。

3. **错误严重性区分**：
   - **严重架构错误**（如将用户态服务功能放入微内核）→ 大幅扣分；
   - **遗漏关键接口变化** → 明显扣分；
   - **过度预测/扩散到无关模块** → 按比例扣分。

### 8.2 T1 评分细则

| 维度 | 满分 | 评分方式 |
|------|------|---------|
| 模块识别正确性 | 15 | 专家核对：必须命中核心模块 |
| 关键数据结构识别 | 15 | 专家核对：必须命中核心结构体 |
| 创建路径描述 | 15 | 专家核对步骤完整性 |
| 传递/转移路径描述 | 15 | 专家核对步骤完整性 |
| 释放/回收路径描述 | 10 | 专家核对步骤完整性 |
| 微内核边界意识 | 10 | 是否正确区分内核态/用户态职责 |
| **中文注释理解** | 10 | 对代码中中文注释、中文 TODO、中文文档引用的理解准确度，是否能从中文注释中准确提取技术信息 |
| **中文表达质量** | 10 | 中文输出的技术表达是否准确、术语使用是否一致、是否存在因语言理解偏差导致的技术错误 |

### 8.3 T2 评分细则

| 维度 | 满分 | 评分方式 |
|------|------|---------|
| 需求目标概括 | 20 | 命中核心目标关键词，无重大遗漏 |
| 功能边界识别 | 15 | 正确区分 MUST/SHOULD/NOT IN SCOPE |
| 基线缺口分析 | 20 | 准确指出缺失的机制或接口 |
| 可能涉及模块 | 15 | 命中1~2个真实核心模块 |
| 不变行为识别 | 10 | 明确指出不应改变的关键行为 |
| **中文需求语义理解** | 10 | 是否准确理解中文需求中的专业术语和中英混排描述，无语义误解。如将中文"配额查询"正确映射到 "quota query" 而非错误理解为配额修改 |
| **中文表达质量** | 10 | 技术表达是否有歧义、术语使用是否准确一致、是否合理使用中英混排的技术表达 |

### 8.4 T3 评分细则

| 维度 | 满分 | 评分方式 |
|------|------|---------|
| 模块命中 | 25 | 与真实修改模块对比，按命中比例计分 |
| 接口/IPC/API 命中 | 25 | 与真实新增/修改接口对比 |
| 控制流/数据流合理 | 20 | 方向正确即可，不要求精确路径 |
| 兼容性影响判断 | 15 | 识别出关键兼容性风险 |
| 过度扩散控制 | 15 | 每列举一个无关模块扣2分，扣完为止 |

**命中判定规则**：
- 模块名称不完全一致但职责边界正确 → 可得分（需专家判定）；
- 列举了正确模块但也列举了大量无关模块 → 过度扩散扣分；
- 未命中任何真实模块 → 该项零分。

### 8.5 T4 评分细则

| 维度 | 满分 | 评分方式 |
|------|------|---------|
| 需求理解准确性 | 15 | 设计方案所基于的问题理解是否正确 |
| 基线现状分析 | 15 | 对当前局限和目标位置的分析到位程度 |
| 模块职责划分 | 15 | 逻辑放置层次正确性（用户态/内核/库） |
| 方案可行性 | 15 | 在现有架构约束下是否可落地实施 |
| 接口/IPC/兼容性设计 | 15 | 接口设计恰当性、兼容性处理 |
| 风险与测试建议 | 10 | 风险识别具体程度、测试场景覆盖度 |
| **中文设计文档能力** | 10 | 设计方案的中文表述是否清晰、技术逻辑是否通顺、图示描述是否准确。能否用中文写出结构化的技术设计方案 |
| **术语一致性** | 5 | 方案中使用的中文技术术语是否与基线代码库中的命名习惯一致，是否出现同一个概念在方案中混用多个不同术语的情况 |

**特殊计分规则**：
- 将用户态服务职责错误放入微内核核心 → 模块职责划分项扣 15 分；
- 提出完全不同的但架构合理的方案 → 可经人工评审给分；
- 泛泛而谈、未结合 baseline 具体代码 → 各维度均低分。

### 8.6 T5 评分细则

| 维度 | 满分 | 评分方式 |
|------|------|---------|
| 编译成功 | 25 | 一次编译成功 25 分；编译失败 0 分（终止后续评测） |
| 功能测试通过 | 45 | 通过率 × 45，如 6/8 通过 = 33.75 分 |
| 回归测试通过 | 10 | 通过率 × 10 |
| 代码规范合规 | 10 | 静态检查得分 |
| **中文注释质量** | 10 | 新增代码中的中文注释是否与目标模块现有的注释语言风格一致；注释是否准确反映代码逻辑，无因中英文切换导致的语义偏差或误导性注释。如果目标模块原为纯英文注释，新增部分应保持一致 |

**编译失败处理**：
- 若编译失败，功能测试和回归测试直接记 0 分；
- 代码规范合规得分仍可通过静态分析给出；
- 最终总分 = 编译(0) + 功能(0) + 回归(0) + 合规(基于代码静态分析)。

**功能测试构造要求**：
- 严格黑盒化：仅通过公开 API/IPC 接口测试，不依赖内部实现细节；
- 每个特性至少 5 个测试用例；
- 覆盖正常路径、边界条件和错误路径；
- 在仅应用参考补丁时确保 100% 通过；
- 在不应用任何实现补丁时至少部分失败（验证测试有效性）。

### 8.7 T6 评分细则

T6 评分采用**相对改进评估**而非绝对正确性评估，核心衡量 AI 的迭代成长和知识迁移能力。

| 维度 | 满分 | 评分方式 |
|------|------|---------|
| **迭代改进质量** | 30 | 从 v1 到 v2 的效果提升幅度。专家评估 v2 相比 v1 在正确性、完整性、合理性方面的改进百分比。如果 v2 劣于 v1 则计 0 分 |
| **反馈响应准确度** | 25 | 是否正确识别并响应了反馈中的关键改进点。每遗漏一个重要反馈点扣 5 分 |
| **错误重复率下降** | 15 | v2 中是否重复了 v1 中已被指出的错误。每重复一个扣 5 分 |
| **知识迁移效率** | 20 | 在 T6.2 中，后续任务相比首个任务的质量提升百分比、完成速度提升、错误率下降等综合指标 |
| **自我反思质量** | 10 | 跨任务总结是否准确识别了自身改进模式和经验教训，反思是否具有可操作性 |

**T6.3 自主进化单独评分**：

| 维度 | 满分 | 评分方式 |
|------|------|---------|
| 问题发现质量 | 25 | 在开放式场景中识别的问题是否真实且重要 |
| 方案完整性 | 25 | 设计是否覆盖了发现问题的所有方面 |
| 实现正确性 | 25 | 代码实现是否可工作、符合基线架构风格 |
| 自验证充分性 | 25 | AI 是否主动验证了自己的方案，验证方法的合理性和覆盖度 |

**T6 特殊计分规则**：
- 如果 v2 比 v1 更差（引入新错误、偏离需求），则迭代改进质量项计 0 分；
- 如果 AI 未经反馈就自主发现问题并改进，可给予额外加分（每项 +5 分，上限 +15）；
- 如果 AI 在迭代中创建了可复用的辅助工具/脚本，给予知识迁移效率加分（+5 分）。

### 8.9 汇总评分体系

**单任务维度得分**：每个任务独立评分 0~100。

**能力维度聚合**：
```
能力 T2 总分 = avg(所有 T2 任务得分)
能力 T3 总分 = avg(所有 T3 任务得分)
...
能力 T6 总分 = avg(所有 T6 任务得分)
```

**综合得分**（两种模式）：

```
标准模式（仅 T1-T5）：
综合得分 = w1×T1_avg + w2×T2_avg + w3×T3_avg + w4×T4_avg + w5×T5_avg

自进化模式（含 T6）：
综合得分 = w1×T1_avg + w2×T2_avg + w3×T3_avg + w4×T4_avg + w5×T5_avg + w6×T6_avg
```

**建议默认权重**（可根据实际需求调整）：
- w1 (知识理解) = 0.10
- w2 (需求理解) = 0.15
- w3 (影响预测) = 0.20
- w4 (方案设计) = 0.20
- w5 (编码实现) = 0.20
- w6 (自进化能力) = 0.15

**难度加权**：可提供 easy/medium/hard 分组统计，观察模型在不同难度下的表现差异。

**语言适配附加维度**：对总分可附加"中文语言适配系数"（0.9~1.1），由专家评估模型在整个评测过程中对中国开发者使用场景的适配程度，包括中文理解准确性、术语一致性、注释风格协调性等综合表现。该系数不改变单项得分，仅用于综合报告的附加说明。

### 8.10 评分执行流程

```
AI 提交答案
    │
    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ T1/T2/T3/T4  │    │     T5       │    │     T6       │
│ 评分流程      │    │  评分流程    │    │  评分流程    │
└──────────────┘    └──────────────┘    └──────────────┘
    │                      │                  │
    ▼                      ▼                  ▼
专家 checklist         沙箱编译测试       迭代对比分析
    │                      │                  │
    ▼                      ▼                  ▼
自动化命中统计        自动化测试套件       改进幅度量化
    │                      │                  │
    ▼                      ▼                  ▼
LLM 语义复核           代码规范检查         知识迁移评估
    │                      │                  │
    ▼                      ▼                  ▼
人工抽检验证           人工抽检验证         专家终审
    │                      │                  │
    └──────────────────────┴──────────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   统一汇总    │
                     │    报告      │
                     └──────────────┘
```

---

## 9. 保密与使用方案

### 9.1 保密等级定义

| 数据类别 | 保密等级 | 说明 |
|---------|---------|------|
| Baseline 源码 | 绝密 | 闭源代码库，严格内部使用 |
| 新特性真实实现 diff | 绝密 | 包含核心实现细节 |
| Private Oracle 数据 | 绝密 | 评分依据，绝不能泄露 |
| 测试套件源码 | 机密 | T5 评测依赖，泄露即失效 |
| 需求描述文档 | 内部 | 信息密度高，需控制传播 |
| T5 设计方案文档 | 内部 | 由专家编写，含架构信息 |
| 架构概览/术语表 | 内部 | 脱敏后可选择性对外 |
| 评分结果报告 | 可对外 | 仅含聚合分数和分析结论 |

### 9.2 内部评测模式（推荐）

```
┌───────────────────────────────────────────────┐
│              安全内网环境                       │
│                                                 │
│  ┌───────────┐      ┌──────────────┐           │
│  │  AI 模型   │─────►│  答案收集器   │           │
│  │  (推理)    │      └──────┬───────┘           │
│  └───────────┘             │                    │
│        ▲                   ▼                    │
│        │           ┌──────────────┐            │
│  ┌─────┴──────┐    │  评测执行器   │            │
│  │ Baseline   │    │  (沙箱内)     │            │
│  │ 只读挂载   │    └──────┬───────┘            │
│  └────────────┘           │                    │
│                           ▼                    │
│                    ┌──────────────┐            │
│                    │  评分引擎     │            │
│                    │  (访问Oracle) │            │
│                    └──────┬───────┘            │
│                           │                    │
│                           ▼                    │
│                    ┌──────────────┐            │
│                    │  结果数据库   │            │
│                    └──────────────┘            │
│                                                 │
│  ═══════════════ 防火墙 ═══════════════        │
│              外部网络（不可访问源码）             │
└───────────────────────────────────────────────┘
```

### 9.3 对外封闭评测模式（若需第三方模型评测）

```
第三方模型提供商
       │
       │ HTTPS (仅传输需求文本和答案)
       ▼
┌─────────────────────────────────┐
│      评测服务平台（我方托管）      │
│                                   │
│  ┌──────────────────────────┐   │
│  │  需求/任务分发 API        │   │
│  │  (只返回需求文本，不返回源码) │   │
│  └──────────┬───────────────┘   │
│             │                    │
│  ┌──────────▼───────────────┐   │
│  │  答案接收 API              │   │
│  │  (接收模型输出的文本/补丁)  │   │
│  └──────────┬───────────────┘   │
│             │                    │
│  ┌──────────▼───────────────┐   │
│  │  受控沙箱评测环境          │   │
│  │  - 编译测试                │   │
│  │  - Oracle 对比             │   │
│  │  - 完全隔离外网             │   │
│  └──────────┬───────────────┘   │
│             │                    │
│  ┌──────────▼───────────────┐   │
│  │  结果返回 API              │   │
│  │  (只返回评分数字和分析)     │   │
│  └──────────────────────────┘   │
│                                   │
│  所有源码、Oracle 永不离开此环境   │
└─────────────────────────────────┘
```

**对外返回数据格式**：
```json
{
  "model_id": "model_xyz",
  "benchmark_version": "1.0",
  "evaluation_date": "2026-06-01",
  "overall_score": 72.5,
  "capability_scores": {
    "T1_baseline_knowledge": {"score": 68.0, "tasks_count": 10},
    "T2_requirement_understanding": {"score": 75.3, "tasks_count": 25},
    "T3_impact_prediction": {"score": 70.1, "tasks_count": 25},
    "T4_design_prediction": {"score": 71.8, "tasks_count": 25},
    "T5_code_implementation": {"score": 76.0, "tasks_count": 25},
    "T6_self_evolution": {"score": 65.0, "tasks_count": 10}
  },
  "domain_breakdown": {
    "kernel_core": {"avg_score": 65.0, "tasks_count": 15},
    "ipc": {"avg_score": 78.0, "tasks_count": 20},
    "user_service": {"avg_score": 74.0, "tasks_count": 30},
    "resource_mgmt": {"avg_score": 72.0, "tasks_count": 12}
  },
  "difficulty_breakdown": {
    "basic": {"avg_score": 85.0, "tasks_count": 30},
    "medium": {"avg_score": 70.0, "tasks_count": 50},
    "advanced": {"avg_score": 55.0, "tasks_count": 20}
  }
}
```

---

## 10. 最小可行版本 MVP 规划

### 10.1 MVP 目标

在第一阶段（建议 3~4 个月）构建一个高质量、小规模、内部使用的 benchmark 版本，验证整体设计和评分体系的有效性。

### 10.2 MVP 范围

| 项目 | 规格 |
|------|------|
| **Baseline** | 半年前一个稳定发布版本，附带完整构建环境和架构文档 |
| **特性数量** | 20~30 个经过严格筛选的独立新特性 |
| **任务总数** | 100~140 个任务 |
| **任务分布** | T1: 10 题 / T2: 20~30 题 / T3: 20~30 题 / T4: 20~30 题 / T5: 20~30 题 / T6: 10 题（T6.1 迭代改进为主） |
| **难度分布** | basic 35% / medium 40% / advanced 25% |
| **评分方式** | 专家 checklist + 自动化命中统计 + 自动化编译测试 |
| **使用范围** | 严格内部评测，不外发 |
| **评测对象** | 2~3 个代表性 AI 模型（包括通用大模型和代码专项模型） |

### 10.3 MVP 不包含的内容

- ❌ T6.3 自主进化完全评估（仅包含 T6.1 迭代改进的试点任务）
- ❌ 对外封闭评测平台（二期建设）
- ❌ 大规模自动化 LLM 辅助评分（二期引入）
- ❌ 多语言/多架构支持（聚焦单一主语言和架构）
- ❌ 持续集成自动更新（二期建设）
- ❌ 多特性联合依赖任务（保持独立性原则）

### 10.4 MVP 成功标准

- [ ] 所有任务经过 2 名以上专家审核，评分 rubric 达成一致；
- [ ] T5 任务：参考补丁在沙箱中编译通过且测试 100% 通过；
- [ ] T5 任务：空补丁在沙箱中至少部分测试失败；
- [ ] 至少完成 2 个主流 AI 模型的评测，评分结果具有区分度；
- [ ] 评测流程端到端跑通，单任务平均评测时间 < 10 分钟；
- [ ] 内部评审确认 benchmark 结果与实际开发中使用 AI 的体感一致。

---

## 11. 实施路线图

### Phase 1：基础建设（第 1-2 月）

| 周次 | 任务 | 产出 |
|------|------|------|
| W1-W2 | 确定 baseline 版本，导出完整快照 | baseline 代码库就绪 |
| W2-W3 | 建立特性收集和筛选流水线 | 候选特性清单 |
| W3-W4 | 独立特性筛选和验证 | 20~30 个确认独立特性 |
| W4-W5 | 开发数据提取和脱敏工具 | 自动化脚本 |
| W5-W6 | 搭建沙箱评测环境（Docker） | 可复现的编译测试环境 |
| W6-W8 | 编写需求描述和 T5 设计方案模板 | 文档模板和编写规范 |

### Phase 2：数据构造（第 2-3 月）

| 周次 | 任务 | 产出 |
|------|------|------|
| W7-W8 | 为前 10 个特性编写完整数据 | 10 个完整 feature 数据包 |
| W8-W9 | 专家标注评分 rubrics | 10 个评分规则文件 |
| W9-W10 | T5 测试套件提取和黑盒化处理 | 10 套可用测试 |
| W10-W11 | 编写 T1 代码库知识理解问题 + T6.1 迭代改进任务构造（选取 5 个特性构造"v1 + 专家反馈"数据对） | 10 个 T1 任务 + 5 个 T6.1 试点任务 |
| W11-W12 | 质量检查和 pilot 测试 | 质量报告和修复 |
| W12-W13 | 完成剩余 10~20 个特性数据 | 完整数据集 |

### Phase 3：评测验证（第 3-4 月）

| 周次 | 任务 | 产出 |
|------|------|------|
| W13-W14 | 开发评测执行器和评分引擎 | 自动化评测工具 |
| W14-W15 | 对模型 A 进行完整评测 | 评测报告 v1 |
| W15-W16 | 对模型 B 进行完整评测 | 评测报告 v2 |
| W16-W17 | 分析评分一致性和区分度 | 质量分析报告 |
| W17-W18 | 根据结果微调 rubrics 和任务 | 优化后数据集 |
| W18 | 内部评审和最终报告 | MVP 正式版本 |

### Phase 4：迭代优化（第 4 月后）

- 根据内部评测反馈优化任务设计和评分体系；
- 增加新 baseline 版本和新特性，形成半年度更新机制；
- 开发对外封闭评测平台（若需要）；
- 引入 LLM 辅助评分以提高效率；
- 扩展到更多领域标签和难度级别；
- 完善 T6.1 迭代改进任务覆盖，试点 T6.2 知识迁移评估。

### Phase 5：自进化深度评估（第 5 月后）

- 全面铺开 T6.2 知识迁移评估任务，构造 10+ 组递增难度特性序列；
- 试点 T6.3 自主进化评估，选择 3~5 个开放式改进目标；
- 将迭代改进评分流程自动化（Agent-As-Judge 模式）；
- 引入进化轨迹可视化报告（v1→v2→v3 的变化趋势）；
- 建立自进化能力基线数据，追踪不同模型在迭代场景下的表现差异。

---

## 12. 最终定义

> **闭源微内核操作系统 AI 代码理解、设计与实现 Benchmark**  
>  
> 这是一个基于真实闭源微内核操作系统历史新特性构造的多维度 AI 能力评测基准。  
>  
> **核心设定**：  
> - 以半年前稳定版本作为 AI 唯一可见的代码上下文  
> - 以近半年真实落地的新特性需求作为题目输入  
> - AI 完全不能访问新特性的真实实现、diff、测试或评审记录  
> - 评测以真实实现代码、真实修改路径、真实接口变化和领域专家标注作为 ground truth  
>  
> **能力链覆盖**：  
> - **T1**：对现有系统架构、接口和机制的深度理解  
> - **T2**：对新特性技术需求的目标、边界和约束的准确把握  
> - **T3**：在不接触真实实现的情况下对变更影响范围的预见能力  
> - **T4**：在微内核架构约束下提出合理、可行的设计方案  
> - **T5**：在给定详细设计时产出可编译、通过测试的代码实现  
>  
> **核心价值**：  
> 它不是在简化环境下的孤立编程测试，而是在**真实、大型、闭源、微内核操作系统的完整工程上下文中**，系统性评估 AI 从"理解旧系统"到"读懂新需求"到"预判影响范围"到"设计方案"到"编写可靠代码"到"自我进化迭代"的全栈能力，从而为 AI 在复杂系统软件开发中的实际应用能力提供高保真的量化度量。
>
> **中文环境适配价值**：  
> 本 benchmark 完整支持中文开发环境场景——需求描述、设计文档、代码注释均以中文为默认语言，评测 AI 在**中文技术语境下对闭源微内核操作系统的理解、设计和编码能力**，真实反映该闭源操作系统中文开发团队的使用体验。

---

## 附录

### A. 术语表

| 术语 | 定义 |
|------|------|
| Baseline | 半年前稳定版本，AI 唯一可见的旧代码库 |
| Feature | 近半年真实开发的独立新特性 |
| Oracle / Ground Truth | 真实实现代码和相关数据，用于评测对比 |
| Rubric | 由专家定义的评分规则和标准 |
| Black-box Testing | 仅通过公开接口验证行为的测试 |
| Sandbox | 隔离的编译和测试运行环境 |
| Contamination | AI 训练数据中混入测试内容导致的评测失真 |
| Chinese-Language Compatibility | 基准对中文开发环境的适配能力，包括中文注释理解、中文需求解析、中文技术表达评估 |
| zh-CN Mode | Benchmark 运行在中文模式下的配置，所有输入输出默认为简体中文 |

### B. 参考文献

1. SWE-bench: Can Language Models Resolve Real-World GitHub Issues? (Jimenez et al., 2024)
2. Can Large Language Models Reason and Plan? (Subbarao et al., 2023)
3. Measuring Coding Challenge Competence with APPS (Hendrycks et al., 2021)
4. The Microkernel Architecture Pattern (Tanenbaum et al.)
5. Benchmark Design Principles for AI Code Evaluation (Industry Best Practices)

### C. 文档版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0 | 2026-05-08 | 初始完整方案，包含 T1-T5 五类任务 |
| v1.1 | 2026-05-08 | 新增中文语言环境适配：新增 1.4 节适配总则，T1/T2/T5 能力矩阵增加中文理解维度，3.9 节中文兼容性原则，增加中文需求描述示例，所有 Schema 增加 language 字段和中文支持，评分体系增加中文表达质量和术语一致性维度 |
| v1.2 | 2026-05-08 | 新增 T6 自进化能力评估：新增 T6 任务类型体系（T6.1 迭代改进/T6.2 知识迁移/T6.3 自主进化），更新任务架构图和关系图，新增 8.7 节 T6 评分细则，调整综合评分权重，更新 MVP 范围和实施路线图新增 Phase 5 |

---

**文档结束**