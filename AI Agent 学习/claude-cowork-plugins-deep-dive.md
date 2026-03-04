# Claude Cowork 及其插件系统深度研究

> 研究日期：2026-03-04  
> 数据来源：claude.com/cowork、claude.com/plugins、support.claude.com、github.com/anthropics/knowledge-work-plugins

---

## 目录

1. [概述：Cowork 是什么](#1-概述cowork-是什么)
2. [核心功能拆解](#2-核心功能拆解)
3. [与普通 Claude Chat 的区别](#3-与普通-claude-chat-的区别)
4. [插件系统技术实现（重点）](#4-插件系统技术实现重点)
5. [开源插件目录（逐个详解）](#5-开源插件目录逐个详解)
6. [插件开发者指南](#6-插件开发者指南)
7. [插件与 Agent/Tool 的关系](#7-插件与-agenttool-的关系)
8. [典型用例](#8-典型用例)
9. [安全与权限模型](#9-安全与权限模型)
10. [与 OpenClaw 的对比和启发](#10-与-openclaw-的对比和启发)

---

## 1. 概述：Cowork 是什么

**Claude Cowork** 是 Anthropic 于 2026 年推出的桌面端 AI Agent 产品（目前处于 Research Preview 阶段），定位是**将 Claude Code 的 Agentic 能力带入知识工作领域**。

### 一句话定位

> "Cowork brings Claude Code's agentic capabilities to your desktop. Give Claude access to your local files, set a task, and step away. Come back to completed work."

Cowork 的核心思路：你描述目标 → Claude 自主规划并执行 → 完成后汇报结果。用户无需盯着屏幕逐步引导，Claude 在后台自主完成多步骤任务。

### 发布背景

- Cowork 与 Claude Chat 和 Claude Code 并列，出现在 Claude 桌面 App 的顶部导航栏
- 它是 Claude Code 的"平民版"——不需要打开终端，面向非开发者知识工作者
- 目前支持 macOS 和 Windows 桌面客户端

### 订阅计划与定价（2026年）

| 计划 | 月费 | Cowork 说明 |
|------|------|------------|
| Pro | $17/月（年付）/ $20/月 | 包含，适合轻量任务 |
| Max 5x | $100/月 | 包含，适合日常中等复杂任务 |
| Max 20x | $200/月 | 包含，适合重度用户全天挂机 |
| Team | $20/seat/月 | 包含 + Slack Connector，支持 5-75 人团队 |
| Enterprise | 企业定价 | 包含，管理员可关闭，目前不记录审计日志 |

**注意**：Cowork 的任务消耗 Token 远多于普通 Chat，因为它需要协调多个子 Agent 和工具调用。

---

## 2. 核心功能拆解

### 2.1 文件访问与自主执行

Cowork 运行在本地隔离虚拟机（VM）中，用户授权后 Claude 可以：
- **读取、编辑、创建**本地文件夹中的文件
- 支持的文件类型极为广泛（见下表）

| 文件类别 | 支持格式 |
|---------|---------|
| 文档/文本 | .docx, .doc, .pdf, .txt, .md, .html, .json, .csv, .tsv |
| 表格 | .xlsx, .xls, .xlsm |
| 演示文稿 | .pptx, .ppt |
| 图片 | .png, .jpg, .jpeg, .gif, .svg, .webp |
| 数据/配置 | .yaml, .yml, .xml, .toml |
| 笔记本 | .ipynb |
| 代码文件 | .py, .js, .ts, .jsx, .tsx, .java, .go, .rs, .rb, .php, .sql, shell 脚本等 |

### 2.2 连接器（Connectors）

除本地文件外，Cowork 还支持通过**连接器**拉取外部上下文：

- **Slack** — 读取频道消息和 DM
- **Notion** — 访问知识库和文档
- **Figma** — 读取设计文件
- **Claude in Chrome** — 在浏览器中浏览网页、抓取数据、填表
- **Microsoft 365** — Word、Excel、Outlook、Teams
- 更多 MCP 兼容服务均可接入

### 2.3 规划 → 执行 → 反馈 循环

当用户发起一个任务，Cowork 的工作流程如下：

```
用户描述任务
     ↓
Claude 分解为多个步骤（规划）
     ↓
Claude 展示计划，等待用户批准
     ↓
开始执行（可并行调用多工具/子 Agent）
     ↓
实时展示进度（用户可随时介入调整）
     ↓
完成后汇报结果
```

**关键设计**：在执行重大操作前，Claude 会主动展示计划并等待用户确认，而不是悄悄做完。

### 2.4 定时任务（Scheduled Tasks）

这是 Cowork 区别于普通 Chat 的重要特性之一：

- 支持 `/schedule` 命令创建定时任务
- 可配置每日、每周、每月自动执行
- 任务在电脑唤醒且桌面 App 打开时运行
- 例如：每天早上 8 点生成当日工作 Briefing；每周五生成周报

### 2.5 Office 深度集成（Research Preview）

Cowork 与 Excel 和 PowerPoint 的 Office Add-in 集成（目前仅 macOS）：
- Claude 可以直接打开、编辑 Excel 和 PowerPoint 文件
- 支持跨应用传递上下文（如在 Excel 中分析数据后，直接将图表插入 PowerPoint 演示文稿）

### 2.6 全局指令与文件夹指令

- **全局指令**：对所有 Cowork Session 生效，可设置偏好格式、语气、背景角色信息
- **文件夹指令**：针对特定文件夹的上下文，类似于代码仓库的 `.cursorrules` 或 `AGENTS.md`
- Claude 在任务执行过程中也可以自动更新文件夹指令

---

## 3. 与普通 Claude Chat 的区别

| 维度 | Claude Chat | Claude Cowork |
|------|------------|---------------|
| **执行模式** | 一问一答，逐步引导 | 自主执行多步骤任务 |
| **文件访问** | 需要手动上传文件 | 可直接读写本地文件夹 |
| **外部工具** | 有限（Web Search 等） | 完整 MCP 连接器生态 |
| **定时任务** | 不支持 | 支持，可自动化周期性任务 |
| **子 Agent** | 不支持 | 支持多 Agent 协作 |
| **Token 消耗** | 标准 | 更高（多步执行） |
| **适用场景** | 即时问答、创作辅助 | 复杂工作流、文件处理、自动化 |
| **运行环境** | 云端对话 | 本地 VM + 桌面 App |
| **历史记录** | 云端同步 | 存储在本地设备 |
| **跨设备同步** | 支持 | 暂不支持 |
| **Projects/Memory** | 支持 | 暂不支持（Research Preview 限制） |

---

## 4. 插件系统技术实现（重点）

### 4.1 插件是什么

**插件（Plugin）** 是将技能（Skills）、连接器（Connectors）、斜杠命令（Slash Commands）和子 Agent（Sub-agents）打包在一起的**单一可安装单元**，使 Claude 在安装后立即以特定领域专家的身份工作。

官方定义：
> "Plugins bundle your team's tools, knowledge, and workflows into a single install so Claude shows up as a specialist from day one."

### 4.2 插件的文件结构（标准规范）

每个插件都遵循相同的目录结构，**全部是 Markdown 和 JSON 文件，无需代码、无需基础设施、无需构建步骤**：

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # 插件清单（必须）
├── .mcp.json                 # MCP 工具连接配置（必须）
├── README.md                 # 插件说明文档
├── CONNECTORS.md             # 连接器详细说明
├── commands/                 # 斜杠命令（显式触发）
│   ├── command-name.md
│   └── ...
└── skills/                   # 技能（上下文相关自动触发）
    ├── skill-name/
    │   └── SKILL.md
    └── ...
```

### 4.3 plugin.json 清单格式

这是插件的元数据文件，格式简洁：

```json
{
  "name": "sales",
  "version": "1.1.0",
  "description": "Prospect, craft outreach, and build deal strategy faster. Prep for calls, manage your pipeline, and write personalized messaging that moves deals forward.",
  "author": {
    "name": "Anthropic"
  }
}
```

字段说明：
- `name`：插件唯一标识符
- `version`：语义化版本号
- `description`：插件功能描述（会显示在插件目录中）
- `author`：作者信息

### 4.4 .mcp.json 连接器配置

这是插件最重要的配置文件，定义了插件所需的所有 MCP Server 连接。以 Sales 插件为例：

```json
{
  "mcpServers": {
    "slack": {
      "type": "http",
      "url": "https://mcp.slack.com/mcp"
    },
    "hubspot": {
      "type": "http",
      "url": "https://mcp.hubspot.com/anthropic"
    },
    "close": {
      "type": "http",
      "url": "https://mcp.close.com/mcp"
    },
    "clay": {
      "type": "http",
      "url": "https://api.clay.com/v3/mcp"
    },
    "zoominfo": {
      "type": "http",
      "url": "https://mcp.zoominfo.com/mcp"
    },
    "notion": {
      "type": "http",
      "url": "https://mcp.notion.com/mcp"
    },
    "fireflies": {
      "type": "http",
      "url": "https://api.fireflies.ai/mcp"
    },
    "ms365": {
      "type": "http",
      "url": "https://microsoft365.mcp.claude.com/mcp"
    },
    "google-calendar": {
      "type": "http",
      "url": "https://gcal.mcp.claude.com/mcp"
    },
    "gmail": {
      "type": "http",
      "url": "https://gmail.mcp.claude.com/mcp"
    }
  }
}
```

**关键特性**：
- 所有 MCP Server 连接均为 **HTTP 类型**（Streamable HTTP MCP 协议）
- 连接的是各 SaaS 厂商自己提供的 MCP Endpoint（如 `mcp.slack.com`、`mcp.hubspot.com`）
- 用户只需要有对应服务的账号和授权，无需自己部署
- 插件在**优雅降级**——即使某些连接器未配置，插件仍可以以"独立模式"工作（用户手动提供数据）

### 4.5 Skills（技能）的设计哲学

Skills 是**域知识的编码**，Claude 在相关上下文中会自动激活它们。

以 Sales 插件的 `call-prep` 技能为例，SKILL.md 包含：

1. **触发条件**（YAML 前置元数据）：
   ```yaml
   ---
   name: call-prep
   description: Prepare for a sales call...
   # 触发词：
   # "prep me for my call with [company]"
   # "I'm meeting with [company] prep me"
   # "call prep [company]"
   ---
   ```

2. **工作流程说明**（Markdown 详细描述）：
   - 独立模式下如何收集信息
   - 连接工具后如何自动拉取数据
   - 分步执行逻辑（Step 1 → Step 2 → Step 3）

3. **输出格式规范**：用 Markdown 模板定义标准化输出结构

4. **关联技能**：指向其他互补的技能

**这意味着**：Skills 本质上是精心设计的**系统提示词片段**，打包成可复用的文件格式。Claude 在执行时会将激活的技能内容注入上下文窗口。

### 4.6 Commands（斜杠命令）的设计

Commands 是用户**显式触发**的工作流。以 `/call-summary` 命令为例：

```markdown
---
description: Process call notes or a transcript — extract action items, draft follow-up email, generate internal summary
argument-hint: "<call notes or transcript>"
---

# /call-summary

Process these call notes: $ARGUMENTS

If a file is referenced: @$1
```

命令文件的关键元素：
- **YAML 前置元数据**：描述和参数提示（用于 UI 展示）
- **`$ARGUMENTS`**：用户输入的参数占位符
- **`@$1`**：文件引用语法
- **正文**：详细的执行指令和输出格式规范

### 4.7 MCP 协议的核心作用

插件系统完全基于 **Model Context Protocol (MCP)**——Anthropic 开源的 AI 工具标准化协议。

```
插件 (.mcp.json) → MCP Server → 第三方 SaaS API
     ↑                                    ↑
  声明连接               按需调用（用户触发时）
```

MCP 的角色：
- **工具统一接口**：所有外部工具都暴露为标准化的 MCP Tool
- **权限隔离**：Claude 通过 MCP 调用工具，而不是直接调用 API
- **可审计**：每次工具调用都有记录
- **可扩展**：任何服务都可以实现 MCP Server

目前已有大量 SaaS 厂商提供原生 MCP Endpoint：
- Slack: `https://mcp.slack.com/mcp`
- HubSpot: `https://mcp.hubspot.com/anthropic`
- Notion: `https://mcp.notion.com/mcp`
- Atlassian: `https://mcp.atlassian.com/v1/mcp`
- Fireflies: `https://api.fireflies.ai/mcp`
- Microsoft 365: `https://microsoft365.mcp.claude.com/mcp`

### 4.8 组织级插件分发

对于 Team 和 Enterprise 计划：
- 管理员可以通过**插件市场**向全组织分发插件
- 组织管理的插件与个人插件工作方式相同，但由中央管控
- 支持自定义组织内部专属插件（包含公司特有的术语、流程、系统）

### 4.9 插件安全机制

- **Anthropic Verified 徽章**：经过 Anthropic 额外质量与安全审核的插件
- **社区插件**：仅通过基本自动化审查，用户需自行判断信任度
- 每个插件在目录中都附有源代码链接，供用户审查
- 插件可能安装本地 MCP Server 或额外软件，需仔细检查

---

## 5. 开源插件目录（逐个详解）

Anthropic 在 GitHub 上开源了 11 个插件（https://github.com/anthropics/knowledge-work-plugins）：

---

### 5.1 productivity（生产力）

**版本**：1.1.0  
**定位**：个人效率助手，管理任务、日历、日常工作流和重要上下文，减少重复劳动

**支持的连接器**：
- Slack、Notion、Asana、Linear、Jira、Monday、ClickUp、Microsoft 365

**核心能力**：
- 管理任务和待办事项（跨平台）
- 日历规划和日程管理
- 构建个人工作记忆（类似 MEMORY.md 机制）
- 每日工作简报生成
- 邮件和消息优先级整理

---

### 5.2 sales（销售）

**版本**：1.1.0  
**定位**：销售全流程助手，从潜客研究到成交跟进

**支持的连接器**：
- CRM: HubSpot、Close
- 通话记录: Fireflies、Gong
- 数据富化: Clay、ZoomInfo、Apollo
- 邮件/日历: Gmail、Google Calendar、Microsoft 365
- 其他: Slack、Notion、Atlassian、Outreach、SimilarWeb

**斜杠命令**：
| 命令 | 功能 |
|------|------|
| `/call-summary` | 处理通话记录，提取行动事项，起草跟进邮件 |
| `/forecast` | 生成加权销售预测（支持 CSV 上传） |
| `/pipeline-review` | 分析管道健康度，标记风险，制定周行动计划 |

**自动技能（自然语言触发）**：
| 技能 | 触发场景 |
|------|---------|
| account-research | 研究目标公司/联系人 |
| call-prep | 准备销售拜访（议程、问题、对手资料） |
| daily-briefing | 生成每日销售工作简报 |
| draft-outreach | 研究后起草个性化外联邮件/LinkedIn 消息 |
| competitive-intelligence | 竞争对手分析、对比矩阵、话术 |
| create-an-asset | 生成定制销售素材（落地页、演示文稿、单页纸） |

**本地配置文件** (`settings.local.json`)：
```json
{
  "name": "Your Name",
  "title": "Account Executive",
  "company": "Your Company",
  "quota": { "annual": 1000000, "quarterly": 250000 },
  "product": {
    "name": "Your Product",
    "value_props": ["价值点1", "价值点2"],
    "competitors": ["竞品A", "竞品B"]
  }
}
```

---

### 5.3 customer-support（客户支持）

**版本**：最新版  
**定位**：客服团队副驾驶，处理工单分类、回复起草、知识库维护

**支持的连接器**：
- Slack（内部讨论）
- Intercom（工单和对话）
- HubSpot（客户信息）
- Guru、Notion（知识库）
- Atlassian（Bug 追踪）
- Microsoft 365（邮件）

**斜杠命令**：
| 命令 | 功能 |
|------|------|
| `/triage` | 工单分类（P1-P4）、优先级评估、路由建议 |
| `/research` | 跨源研究客户问题，带置信度分数 |
| `/draft-response` | 为各种情况起草客户回复 |
| `/escalate` | 打包升级材料（含重现步骤、业务影响评估） |
| `/kb-article` | 从已解决问题自动生成知识库文章 |

**技能亮点**：
- 工单分类使用 P1-P4 优先级框架
- 回复草稿自动适配紧急程度和沟通渠道
- 升级包含结构化格式，方便工程/产品团队快速理解

---

### 5.4 product-management（产品管理）

**版本**：最新版  
**定位**：PM 全流程助手，从需求文档到路线图，再到干系人沟通

**支持的连接器**：
- Slack、Linear、Asana、Monday、ClickUp、Jira（项目追踪）
- Notion（知识库）
- Figma（设计上下文）
- Amplitude、Pendo（产品分析）
- Intercom（用户反馈）
- Fireflies（会议记录）

**斜杠命令**：
| 命令 | 功能 |
|------|------|
| `/write-spec` | 从问题陈述生成 PRD/功能规格（含用户故事、验收标准） |
| `/roadmap-update` | 创建或更新路线图（支持多种格式） |
| `/stakeholder-update` | 生成面向不同受众的状态更新 |
| `/synthesize-research` | 综合多份用研材料，识别主题和机会点 |
| `/competitive-brief` | 竞品分析报告（功能对比、定位分析） |
| `/metrics-review` | 分析产品指标趋势 |

---

### 5.5 marketing（市场营销）

**版本**：最新版  
**定位**：市场营销全栈助手，从内容创作到活动规划再到绩效分析

**支持的连接器**：
- Slack、Canva、Figma（内容和设计）
- HubSpot、Amplitude、Klaviyo（营销自动化和分析）
- Notion（文档）
- Ahrefs、Similarweb（SEO 和竞品）

**斜杠命令**：
| 命令 | 功能 |
|------|------|
| `/draft-content` | 起草博客、社媒、Newsletter、落地页、案例研究 |
| `/campaign-plan` | 生成完整活动 Brief（目标、渠道、内容日历、KPI） |
| `/brand-review` | 检查内容是否符合品牌声音和风格指南 |
| `/competitive-brief` | 竞品定位和消息对比 |
| `/performance-report` | 营销绩效报告（指标、趋势、优化建议） |
| `/seo-audit` | SEO 全面审计（关键词、页内、内容缺口、竞品对比） |
| `/email-sequence` | 设计并起草多封邮件序列（Nurture、Onboarding、Drip） |

---

### 5.6 legal（法律）

**版本**：最新版  
**定位**：企业内法律团队助手（商业顾问、产品法律、合规、诉讼支持）

**⚠️ 免责声明**：本插件辅助法律工作流，**不提供法律建议**，所有输出需由执照律师审查后方可依赖。

**支持的连接器**：
- Slack（团队通讯）
- Box、Egnyte（文件存储）
- Microsoft 365（邮件、日历、文档）
- Atlassian（事项追踪）

**斜杠命令**：
| 命令 | 功能 |
|------|------|
| `/review-contract` | 按照组织谈判 Playbook 审查合同，标记偏差，生成批注 |
| `/triage-nda` | NDA 快速分类（GREEN/YELLOW/RED）|
| `/vendor-check` | 查询与特定供应商的现有协议状态 |
| `/brief` | 生成法律简报（每日/专题/突发情况） |
| `/respond` | 从配置模板生成标准回复（DSR、保全通知等） |

**高度可定制化**——通过 `legal.local.md` 文件定义组织的谈判立场：
```markdown
# Legal Playbook Configuration

## Contract Review Positions

### Limitation of Liability
- Standard position: Mutual cap at 12 months of fees paid/payable
- Acceptable range: 6-24 months of fees
- Escalation trigger: Uncapped liability
```

---

### 5.7 finance（财务）

**版本**：1.1.0  
**定位**：财务和会计工作流自动化（月末关账、日记账、对账、财报）

**⚠️ 免责声明**：辅助财务工作流，**不提供财务、税务或审计建议**，所有输出需合格财务专业人员审查。

**支持的连接器**：
- 数据仓库: Snowflake、Databricks、BigQuery
- 办公: Microsoft 365（Excel 工作底稿）
- 沟通: Slack

**斜杠命令**：
| 命令 | 功能 |
|------|------|
| `/journal-entry` | 生成日记账分录（应计、固定资产、预付、薪资、收入确认） |
| `/reconciliation` | 账户对账（GL 与子账、银行、第三方） |
| `/income-statement` | 生成损益表（含期间对比和差异分析） |
| `/variance-analysis` | 差异/弹性分析（分解为驱动因素，含瀑布图叙述） |
| `/sox-testing` | SOX 合规测试（样本选择、测试底稿、控制评估） |

**月末关账工作流示例**：
```
1. /journal-entry ap-accrual 2024-12 → 生成应付账款应计
2. /journal-entry prepaid 2024-12  → 摊销预付费用
3. /journal-entry fixed-assets 2024-12 → 计提折旧
4. /reconciliation cash 2024-12    → 银行对账
5. /income-statement monthly 2024-12 → 生成含差异分析的损益表
```

---

### 5.8 data（数据分析）

**版本**：最新版  
**定位**：数据分析师协作工具，SQL 查询、可视化、仪表板构建、洞察生成

**支持的连接器**：
- 数据仓库: Snowflake、Databricks、BigQuery
- 分析: Amplitude、Hex
- 项目追踪: Jira

**斜杠命令**：
| 命令 | 功能 |
|------|------|
| `/analyze` | 回答数据问题（从快速查询到完整分析） |
| `/explore-data` | 数据集 Profiling（形状、质量、模式） |
| `/write-query` | 按指定 SQL 方言写优化查询 |
| `/create-viz` | 用 Python 创建出版质量的可视化 |
| `/build-dashboard` | 构建交互式 HTML 仪表板（含 Chart.js、过滤器） |
| `/validate` | 分享前 QA 检查（方法论、准确性、偏差检查） |

**两种运行模式**：
- **有数据仓库连接**：直接查询、迭代分析、端到端执行
- **无连接**：粘贴 SQL 结果或上传 CSV/Excel 文件，Claude 分析并生成可视化

---

### 5.9 enterprise-search（企业搜索）

**版本**：最新版  
**定位**：跨所有企业工具的统一搜索，一次查询找遍全公司

**核心理念**：
> "知识工作者每周花数小时寻找散落在各工具中的信息。答案就在某处——Slack 线程、邮件链、文档、Wiki 页面——但找到它需要逐个工具搜索。Enterprise Search 把所有工具当作一个可搜索的知识库。"

**工作原理**：
```
用户提问："我们关于 API 重设计做了什么决定？"
    ↓ Claude 分解查询
    ↓ 并行搜索各数据源
  Slack → #engineering 周二的讨论
  Email → Sarah 的规格说明邮件
  云存储 → 更新后的 API 设计文档
    ↓ 综合结果
"团队周二决定用 REST 而非 GraphQL。Sarah 周四发了更新规格..."
```

**斜杠命令**：
| 命令 | 功能 |
|------|------|
| `/search` | 跨所有连接数据源的统一查询 |
| `/digest` | 生成每日/每周跨源活动摘要 |

**支持的查询过滤器**：`from:`、`in:`、`after:`、`before:`、`type:`

---

### 5.10 bio-research（生物研究）

**版本**：最新版  
**定位**：生命科学 R&D 加速器，连接前临床研究工具和数据库

**包含的 MCP Server**（10 个）：
| 数据源 | 用途 |
|--------|------|
| PubMed (NLM) | 生物医学文献检索 |
| bioRxiv/medRxiv (deepsense.ai) | 预印本访问 |
| Wiley | 学术论文全文 |
| Synapse (Sage Bionetworks) | 协作研究数据管理 |
| ChEMBL | 生物活性药物化合物数据库 |
| Open Targets | 药物靶点发现和优先级排序 |
| ClinicalTrials.gov | 临床试验数据 |
| BioRender | 科学插图制作 |
| Owkin | AI 组织病理学和药物发现 |
| Benchling | 实验室数据管理平台 |

**可选二进制 MCP**（需单独下载）：
- 10X Genomics txg-mcp（单细胞基因组学）
- ToolUniverse（Harvard MIMS 科学发现工具）

**分析技能（Skills）**：
1. **Single-Cell RNA QC** — scRNA-seq 数据质控，基于 MAD 过滤
2. **scvi-tools** — 深度学习单细胞组学（scVI、scANVI 等模型）
3. **Nextflow Pipelines** — nf-core 生信流程（RNA-seq、变异检测、ATAC-seq）
4. **Instrument Data to Allotrope** — 仪器数据转换（40+ 仪器类型）
5. **Scientific Problem Selection** — Fischbach & Walsh 研究问题选择框架

---

### 5.11 cowork-plugin-management（插件管理）

**版本**：0.2.2  
**定位**：创建和管理插件的元插件，帮助用户为自己的组织构建定制插件

**这是一个特殊的"引导器"插件**：
- 引导用户从零创建新插件
- 帮助定制现有 Anthropic 插件
- 配置 MCP Server 连接
- 调整插件行为以匹配团队实际工作方式
- 无需了解插件规范即可完成创建

---

## 6. 插件开发者指南

### 6.1 创建插件的三种方式

1. **使用 cowork-plugin-management 插件**（推荐新手）：直接在 Cowork 中通过对话创建
2. **Fork 开源仓库**：从 Anthropic 的模板修改定制
3. **从零手工创建**：按照文件结构规范自行构建

### 6.2 开发规范

#### 必须文件
- `.claude-plugin/plugin.json`（插件清单）
- `.mcp.json`（MCP 连接配置，即使没有连接器也需要存在）

#### Skills 编写规范

SKILL.md 文件结构：
```markdown
---
name: skill-name
description: 简短描述（用于触发条件判断）
# 触发词示例（帮助 Claude 决定何时激活此技能）
---

# Skill Name

## 概述
[技能的用途和价值]

## 工作原理
[独立模式 vs 连接器模式的对比说明]

## 执行流程
[分步骤的执行逻辑]

## 输出格式
[标准化的 Markdown 输出模板]

## 相关技能
[链接到互补技能]
```

#### Commands 编写规范

命令文件（`commands/command-name.md`）结构：
```markdown
---
description: 命令功能描述（显示在 / 菜单中）
argument-hint: "<参数提示>"
---

# /command-name

[命令执行指令]

Process: $ARGUMENTS

[详细的执行逻辑、输出格式、注意事项]
```

### 6.3 定制已有插件

**步骤**：
1. **替换连接器**：编辑 `.mcp.json`，指向自己的工具栈（如替换 HubSpot 为 Salesforce）
2. **添加公司上下文**：在 skill 文件中加入公司术语、组织结构、流程规范
3. **调整工作流**：修改 skill 指令，使其符合团队实际做法而非教科书方式
4. **添加本地配置**：在 `.claude/settings.local.json` 中设置个性化参数

### 6.4 插件发布

- 提交到 claude.com/plugins 的社区插件目录（Anthropic 审查）
- 通过 GitHub PR 贡献到 anthropics/knowledge-work-plugins

### 6.5 Claude Code 中的安装方式

```bash
# 添加市场
claude plugin marketplace add anthropics/knowledge-work-plugins

# 安装特定插件
claude plugin install sales@knowledge-work-plugins

# 安装后自动激活，斜杠命令立即可用
# 例如：/sales:call-prep, /data:write-query
```

### 6.6 参考文档

- 官方插件技术规范：https://code.claude.com/docs/en/plugins-reference

---

## 7. 插件与 Agent/Tool 的关系

### 7.1 概念层级

```
Cowork（Agentic 运行时）
    ├── 插件（Plugin）
    │     ├── Skills（知识/提示词片段）
    │     ├── Commands（用户触发的工作流）
    │     ├── Sub-agents（专门任务的子代理）
    │     └── Connectors（通过 MCP 连接工具）
    │           └── MCP Tools（具体工具调用）
    └── 本地文件访问（VM 沙箱）
```

### 7.2 Skills vs Tools vs Agents

| 概念 | 本质 | 触发方式 | 示例 |
|------|------|---------|------|
| **Skill** | 领域知识提示词 | 上下文相关自动触发 | call-prep 技能 |
| **Command** | 显式工作流程序 | 用户 `/命令` 触发 | `/call-summary` |
| **Tool (MCP)** | 外部能力接口 | Claude 决策调用 | Slack 搜索、HubSpot 查询 |
| **Sub-agent** | 专门任务的子 Claude | 主 Agent 委托 | 并行研究多个竞品 |
| **Plugin** | 上述四者的打包 | 安装即激活 | Sales 插件 = 全套销售能力 |

### 7.3 执行机制

当用户在 Cowork 中触发任务：

1. **主 Agent** 接收用户指令，分析已安装插件的可用能力
2. **Skills 激活**：相关技能的 SKILL.md 被注入上下文
3. **规划**：主 Agent 制定执行计划，可能委托给 Sub-agents
4. **工具调用**：通过 MCP 协议调用外部工具（Slack、HubSpot 等）
5. **文件操作**：直接读写用户授权的本地文件
6. **结果综合**：汇总所有输出，生成最终结果

### 7.4 "Standalone + Supercharged" 设计模式

Anthropic 在每个插件中都贯彻了一个设计原则：

```
✓ 独立模式（Standalone）：用户手动粘贴数据，插件仍可工作
+ 增强模式（Supercharged）：连接外部工具后，自动拉取数据
```

这确保了插件的普适性——即使没有任何 MCP 连接，插件仍有价值。

---

## 8. 典型用例

### 8.1 管理类

**场景：创建每日工作简报**
```
使用插件：productivity / enterprise-search
操作：设置定时任务，每天 8 点自动从 Slack、Notion、GitHub 拉取
      当日重点，整合为优先级排列的简报
```

**场景：整理混乱的 Downloads 文件夹**
```
直接 Cowork 任务：指向 Downloads 文件夹
Claude 自动：按类型分类、按规范重命名、清理重复文件
无需任何插件
```

### 8.2 研究与分析类

**场景：市场规模分析**
```
直接 Cowork 任务："分析 X 市场规模"
Claude 自动：网络调研 → 数据计算 → 生成 Excel/PPT 可交付物
```

**场景：数据仓库分析**
```
使用插件：data
操作：/analyze 过去 12 个月各产品线收入趋势
Claude：写 SQL → 执行查询 → 生成图表 → 验证结果 → 输出报告
```

### 8.3 文档创建类

**场景：从散落笔记生成报告**
```
直接 Cowork 任务：给 Claude 访问笔记文件夹权限
Claude 自动：读取所有相关内容 → 综合关键信息 → 生成草稿
```

**场景：合同审查**
```
使用插件：legal
操作：/review-contract，上传合同 PDF
Claude：按公司 Playbook 逐条分析 → GREEN/YELLOW/RED 标记 → 生成批注建议
```

### 8.4 销售/GTM 类

**场景：综合客户反馈**
```
使用插件：sales / customer-support
操作：综合通话记录、Slack、CRM Notes、Linear 问题
Claude：识别跨平台模式 → 生成优先级排列的产品建议
```

**场景：法律文件管理**
```
直接 Cowork 任务：将诉讼文件夹整理为按时间排列的证据集
Claude：自动分类、命名、排序、生成战略重要性评估
```

---

## 9. 安全与权限模型

### 9.1 VM 隔离

- Cowork 在本地隔离虚拟机中运行
- 代码安全隔离执行，但 Claude **可以**对授权文件做真实改动
- 网络访问通过白名单（Allowlist）控制，用户在设置中配置

### 9.2 权限原则

- 只有用户**明确授权**的文件夹和连接器才可访问
- Claude 在执行重大操作前展示计划，等待批准
- 用户可在任何步骤重定向、细化或改变方向

### 9.3 数据存储

- Cowork 对话历史存储在**本地设备**，不在 Anthropic 服务器
- Team/Enterprise 的审计日志、合规 API、数据导出**目前不捕获** Cowork 活动

### 9.4 合规限制

- **不适用于**：HIPAA、FedRAMP 或 FSI 监管工作负载
- 管理员可随时在 Admin Settings 中关闭 Cowork

---

## 10. 与 OpenClaw 的对比和启发

### 10.1 定位对比

| 维度 | Claude Cowork | OpenClaw (内网版) |
|------|---------------|-----------------|
| **核心定位** | 桌面端知识工作 Agent | 企业内网 AI 助手（工蜂生态） |
| **运行环境** | 本地 VM + 桌面 App | 工蜂内网云端 + 容器 |
| **文件访问** | 本地文件系统 | 工作空间文件 + 代码仓库 |
| **插件生态** | 社区 + Anthropic 官方 | 待建设 |
| **工具集成** | MCP 协议，连接商业 SaaS | 工蜂内网服务 |
| **代码能力** | Claude Code 同等 Agentic | exec/process 工具 |
| **多 Agent** | Sub-agent 协作 | 单主 Agent |
| **定时任务** | `/schedule` 本地执行 | cron 工具 |

### 10.2 关键启发

#### 启发 1：插件 = 角色专业化的打包机制

Cowork 最聪明的设计是将"专家角色"打包成可安装单元。OpenClaw 目前通过 `SOUL.md` + `USER.md` 定义 Agent 人格，但缺乏**领域专家插件**的概念。

**可借鉴**：设计 OpenClaw 插件系统，每个插件包含：
- 领域知识提示词（对应 SKILL.md）
- 工蜂内网服务连接（对应 .mcp.json）
- 斜杠命令（对应 commands/）

#### 启发 2：Standalone + Supercharged 渐进增强模式

每个插件在没有外部连接时仍能工作，有连接时能力更强。这是优秀的降级设计——类似于 OpenClaw 中即使没有任何工具也能对话，但连上工具后能力更强。

**可借鉴**：所有 OpenClaw 技能都应明确区分"基础能力"和"增强能力（需要特定工具）"。

#### 启发 3：MCP 作为工具统一接口

Cowork 通过 MCP 协议统一接入所有外部工具，而不是每个工具都写专有集成代码。这让社区可以自由扩展工具生态。

**可借鉴**：OpenClaw 的工具层可以采用类似 MCP 的标准化接口，让工蜂生态的各个服务（工单系统、代码仓库、知识库等）可插拔接入。

#### 启发 4：文件即配置，无需编写代码

Cowork 插件完全用 Markdown 和 JSON 实现，任何会写文档的人都能创建插件。这极大降低了插件开发门槛。

**可借鉴**：OpenClaw 的配置文件（SOUL.md、USER.md、AGENTS.md）已经体现了这个思路。可以进一步将"技能"标准化为可分享的 Markdown 文件。

#### 启发 5：组织级插件分发

Team/Enterprise 计划允许管理员统一分发插件，确保团队使用标准化的工作方式。

**可借鉴**：在工蜂内网场景中，可以让项目/部门级别的 `AGENTS.md` 扮演类似组织插件的角色，定义团队标准工作流。

#### 启发 6：插件管理插件（Meta-plugin）

`cowork-plugin-management` 插件是个精妙设计——用 Claude 来创建 Claude 插件，降低创建门槛，同时保证格式规范。

**可借鉴**：OpenClaw 可以有一个"工作流创建助手"，帮助用户通过对话定义自己的工作流、命令和技能。

### 10.3 架构差异分析

**Cowork 的优势**：
- 完整的本地文件访问能力（VM 沙箱）
- 成熟的 MCP 生态（大量 SaaS 厂商已接入）
- 明确的多 Agent 架构（Sub-agents）
- 桌面 App 提供定时任务和后台执行能力

**OpenClaw 的优势**：
- 与工蜂内网生态深度集成（无法被 Cowork 替代）
- 支持代码仓库级别的上下文理解
- 企业内网安全合规（数据不出内网）
- 跨设备/平台的消息通道（企业微信/工蜂等）

**OpenClaw 的发展方向**：
- 对标 Cowork 的插件系统，建设领域专家插件生态
- 深化 MCP 协议支持，连接工蜂内网各服务
- 完善 Agentic 执行能力（多步任务、子 Agent）
- 建设组织级配置分发机制

---

## 附录：资源链接

| 资源 | 链接 |
|------|------|
| Cowork 产品页 | https://claude.com/product/cowork |
| 插件目录 | https://claude.com/plugins |
| 官方帮助文档 | https://support.claude.com/en/articles/13345190-get-started-with-cowork |
| 插件使用指南 | https://support.claude.com/en/articles/13837440-use-plugins-in-cowork |
| 开源插件仓库 | https://github.com/anthropics/knowledge-work-plugins |
| 插件技术规范 | https://code.claude.com/docs/en/plugins-reference |
| MCP 协议官网 | https://modelcontextprotocol.io/ |
| Claude 下载 | https://claude.com/download |

---

*本文档由 OpenClaw (内网版) 研究生成，基于 2026-03-04 的公开资料整理。*
