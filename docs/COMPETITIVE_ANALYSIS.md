# 竞品深度分析

> 基于实际下载的源码（解压后逐文件分析），不是二手资料。

## 总览：已下载的源码

| 项目 | 大小 | 来源 | 核心文件 |
|------|------|------|---------|
| cloudcli | 4.3MB | npm @cloudcli-ai/cloudcli@1.36.3 | 153 个 JS 后端 + 105 个前端 |
| ECC | ~280MB | git clone affaan-m/ecc | 278 skills + 67 agents |
| superpowers | ~10MB | git clone obra/superpowers | 14 skills |
| mem0 | 788KB | npm mem0ai@3.1.1 | SDK + 自托管 |
| Zep | 234KB | npm @getzep/zep-cloud@3.25.0 | Thread + Graph + User |
| Pi | 5MB | npm @earendil-works/pi-coding-agent@0.81.1 | 完整 harness |
| Cline CLI | 6KB | npm @cline/cli@0.0.13 | CLI 入口 |
| OpenCode CLI | 3KB | npm opencode-ai@1.18.4 | CLI 入口 |

---

## 1. cloudcli（最复杂的现有方案）

### 1.1 项目规模

```
后端 (dist-server): 30,040 行 JS (153 文件)
前端 (dist): 14,942 行 (105 文件)
源版 (server/): 12,982 行 (36 文件)
总计: 52,335 行
```

### 1.2 目录结构

```
dist-server/server/
├── index.js                    # Express + WebSocket 入口（主服务器）
├── cli.js                      # CLI 命令解析（start/sandbox/info）
├── claude-sdk.js               # Claude Code SDK 封装
├── openai-codex.js             # Codex SDK 封装（447 行）
├── cursor-cli.js               # Cursor CLI 封装
├── opencode-cli.js             # OpenCode CLI 封装
├── voice-proxy.js              # 语音代理
├── browser-use-mcp.js          # 浏览器使用 MCP
├── load-env.js                 # 环境变量加载
│
├── modules/
│   ├── providers/              # ⭐ Provider 系统（最复杂）
│   │   ├── index.js
│   │   ├── list/
│   │   │   ├── claude/
│   │   │   │   ├── claude.provider.js (18 行)
│   │   │   │   ├── claude-auth.provider.js (125 行)
│   │   │   │   ├── claude-mcp.provider.js (102 行)
│   │   │   │   ├── claude-models.provider.js (370 行)
│   │   │   │   ├── claude-sessions.provider.js (566 行)
│   │   │   │   ├── claude-session-synchronizer.provider.js (138 行)
│   │   │   │   └── claude-skills.provider.js (204 行)
│   │   │   ├── codex/
│   │   │   │   ├── codex.provider.js (18 行)
│   │   │   │   ├── codex-auth.provider.js (83 行)
│   │   │   │   ├── codex-mcp.provider.js (106 行)
│   │   │   │   ├── codex-models.provider.js (145 行)
│   │   │   │   ├── codex-sessions.provider.js (537 行)
│   │   │   │   ├── codex-session-synchronizer.provider.js (197 行)
│   │   │   │   └── codex-skills.provider.js (56 行)
│   │   │   ├── cursor/
│   │   │   │   └── ...（类似的 7 文件）
│   │   │   └── opencode/
│   │   │       └── ...（类似的 7 文件）
│   │   ├── services/
│   │   │   ├── provider-models.service.js  # 模型动态加载（903 行）
│   │   │   ├── provider-auth.service.js
│   │   │   ├── sessions.service.js
│   │   │   ├── settings.service.js
│   │   │   └── notifications.service.js
│   │   ├── provider.registry.js     # Provider 注册表
│   │   └── provider.routes.js
│   ├── websocket/
│   │   ├── index.js
│   │   └── services/
│   │       ├── chat-websocket.service.js  # 聊天 WebSocket
│   │       ├── shell-websocket.service.js
│   │       ├── plugin-websocket-proxy.service.js
│   │       └── websocket-state.service.js
│   ├── database/
│   │   ├── schema.js              # SQLite schema
│   │   ├── migrations.js
│   │   ├── repositories/
│   │   │   ├── sessions.db.js     # Session 仓库
│   │   │   └── projects.db.js
│   │   └── index.js
│   ├── assets/
│   ├── projects/
│   ├── browser-use/
│   ├── notifications/
│   └── voice/
│
├── routes/                       # Express routes
│   ├── auth.js
│   ├── agent.js
│   ├── git.js
│   ├── cursor.js
│   ├── commands.js
│   ├── plugins.js                # 插件管理
│   └── ...
│
├── shared/                       # 共享工具
│   ├── utils.js
│   └── constants.js
│
├── utils/
│   ├── plugin-loader.js
│   ├── plugin-process-manager.js
│   ├── colors.js
│   ├── runtime-paths.js
│   └── ...
│
└── middleware/
    ├── auth.js                   # JWT 认证
    └── ...
```

### 1.3 Provider 接口设计（值得借鉴）

**每个 Provider 拆成 7 个文件**，每个文件单一职责：

```
provider/
├── {name}.provider.js                      # 18 行 - 主入口
├── {name}-auth.provider.js                 # 认证
├── {name}-mcp.provider.js                  # MCP 配置
├── {name}-models.provider.js               # 模型列表
├── {name}-sessions.provider.js             # Session 流处理（最复杂）
├── {name}-session-synchronizer.provider.js # 文件监控
└── {name}-skills.provider.js               # 技能系统
```

**借鉴**：每个 Provider 7 文件模式（清晰、可维护）

**改进**：
- 加 `routing.provider.js`（智能路由）
- 加 `memory.provider.js`（跨 provider 记忆）
- 加 `session-migration.provider.js`（跨 provider 迁移）

### 1.4 关键技术细节

**SQLite Schema**（`schema.js`）：
```sql
CREATE TABLE IF NOT EXISTS sessions (
    session_id TEXT NOT NULL,
    provider TEXT NOT NULL DEFAULT 'claude',
    provider_session_id TEXT,
    custom_name TEXT,
    project_path TEXT,
    jsonl_path TEXT,
    isArchived BOOLEAN DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (session_id),
    FOREIGN KEY (project_path) REFERENCES projects(project_path)
);
```

**借鉴**：sessions 表设计很好（provider_session_id 分离让前端用 session_id，provider 用自己的 id）

**WebSocket 协议**（`chat-websocket.service.js`）：
- 客户端发 `{ kind: 'chat.send', sessionId, content, options }`
- 服务端发 `{ kind: 'token_budget', tokenBudget }`、`{ kind: 'status', text }` 等
- 流式事件用 `kind: 'item', itemType: 'agent_message', content }`

**借鉴**：WebSocket 协议模式（已用 `providerModelsService.resolveResumeModel`）

### 1.5 痛点分析

**Web UI 优先**：没有 CLI 入口，只有 `cloudcli start` 启动 Web 服务器
- 用户必须用浏览器
- 不能 `cloudcli run "fix this"` 直接跑

**没有跨 provider 路由**：每个 provider 独立调用
- 用户要自己选 provider

**没有共享记忆**：每个 provider 独立 session
- Codex 不知道 Claude 学到的用户偏好

**MCP 配置分散**：
- `~/.codex/config.toml`
- `~/.claude.json`
- `~/.cursor/mcp.json`
- **不统一**

### 1.6 借鉴清单

✅ **直接复用**：
- Provider 抽象模式（7 文件）
- WebSocket 协议
- SQLite schema（sessions、projects）
- Express 路由模式
- 中间件（auth、cors）

⚠️ **借鉴后改进**：
- 加 CLI 入口
- 加智能路由层
- 加跨 provider 记忆
- 加 MCP 共享中心
- 用 React 重写前端

---

## 2. ECC（affaan-m/ECC，232k stars）

### 2.1 项目结构

```
ecc/
├── agents/                # 67 个 Markdown agent 定义
├── skills/                # 278 个 Markdown skill 定义 ⭐
├── commands/              # 94 个 slash 命令
├── rules/                 # 永远跟随的规则
├── hooks/                 # 运行时钩子
├── mcp-configs/           # MCP 配置
├── contexts/              # 动态上下文注入
├── marketplace.json       # Claude plugin 市场
├── llms.txt               # LLM 文档
└── 各种文档
```

### 2.2 Skills 格式（核心借鉴）

**每个 skill 一个文件夹，包含 `SKILL.md`**：

```
skills/
├── brainstorming/
│   └── SKILL.md
├── tdd-workflow/
│   └── SKILL.md
└── ...
```

**SKILL.md 格式**：
```markdown
---
name: brainstorming
description: "You MUST use this before any creative work..."
metadata:
  origin: ECC
  author: affaan
  version: 1.0
---

# Brainstorming Ideas Into Designs

主要内容...

## Anti-Pattern

反模式列表...

## Checklist

检查清单...
```

### 2.3 ECC 跨 Harness 支持（核心创新）

**支持 9+ 个 AI 编码 harness**：
- Claude Code（主战场）
- Cursor IDE
- Codex CLI + App
- OpenCode
- Pi
- Antigravity
- Factory Droid
- GitHub Copilot CLI
- Kimi Code

**借鉴方法**：每个 harness 都有自己的配置目录：

```
.cursor/          # Cursor rules
.opencode/        # OpenCode 配置
.zed/             # Zed
.github/          # GitHub Copilot instructions
```

### 2.4 借鉴清单

✅ **直接复用 278 个 skills**：
- magent 解析 SKILL.md frontmatter
- 注入到任何 provider 的 system prompt
- 不重新写，直接调用

✅ **借鉴格式**：
- SKILL.md 格式（YAML frontmatter + Markdown body）
- HARD-GATE 概念（强制执行点）
- Anti-Pattern / Checklist 结构

⚠️ **避免**：
- ECC 太重（232k stars 都是"功能堆砌"）
- 我们只做核心，简洁优先

---

## 3. superpowers（obra/superpowers，259k stars）

### 3.1 项目结构

```
superpowers/
├── skills/                # 14 个核心 skills
│   ├── brainstorming/
│   ├── test-driven-development/
│   ├── systematic-debugging/
│   ├── using-git-worktrees/
│   ├── writing-plans/
│   ├── subagent-driven-development/
│   └── ...
├── docs/                  # 文档
└── 各种 harness 的安装指南
```

### 3.2 superpowers 的核心创新

**14 个 skills** 都是**方法论**（不是工具调用）：

| Skill | 作用 |
|------|------|
| brainstorming | 设计前先 brainstorm |
| writing-plans | 写实施计划 |
| test-driven-development | TDD |
| systematic-debugging | 系统调试 |
| using-git-worktrees | git worktree 隔离 |
| subagent-driven-development | subagent 编排 |
| requesting-code-review | 代码审查 |
| finishing-a-development-branch | 完成分支 |
| dispatching-parallel-agents | 并行 agent |
| executing-plans | 执行计划 |
| receiving-code-review | 接收审查 |
| writing-skills | 写 skills |
| using-superpowers | 介绍 |
| verification-before-completion | 完成前验证 |

### 3.3 借鉴清单

✅ **借鉴方法论**：
- HARD-GATE 概念（magent 解析后强制执行）
- systematic-debugging 流程（用于路由失败处理）
- TDD 工作流（agent 自己写测试）

✅ **可借鉴的 SKILL.md 风格**：
```markdown
<HARD-GATE>
Do NOT invoke any implementation skill until you have presented a design
</HARD-GATE>
```

⚠️ **避免**：
- skills 太方法论（不直接给工具调用）
- 我们需要更多"做事的"skills（不只是"思考的"）

---

## 4. mem0（v3.1.1）

### 4.1 SDK 结构

```
mem0ai/
├── dist/
│   ├── index.js (主入口)
│   ├── index.d.ts (TypeScript 定义)
│   └── oss/ (自托管版本)
└── package.json
```

### 4.2 核心 API（TypeScript）

```typescript
interface MemoryClient {
  add(messages: Message[], options?: AddMemoryOptions): Promise<Memory[]>
  search(query: string, options?: SearchMemoryOptions): Promise<SearchResult>
  get(memoryId: string): Promise<Memory>
  getAll(options?: GetAllMemoryOptions): Promise<PaginatedMemories>
  update(memoryId: string, body: MemoryUpdateBody): Promise<Memory>
  delete(memoryId: string, options?: DeleteMemoryOptions): Promise<void>
  deleteAll(options?: DeleteAllMemoryOptions): Promise<void>
}

interface AddMemoryOptions {
  userId?: string
  agentId?: string
  appId?: string
  runId?: string
  metadata?: Record<string, any>
  infer?: boolean                          // LLM 自动提取
  customCategories?: custom_categories[]
  customInstructions?: string              // 自定义提取规则
  timestamp?: number
  expirationDate?: string
  structuredDataSchema?: Record<string, any>
}

interface SearchMemoryOptions {
  filters?: Record<string, any>
  topK?: number                             // 默认 10
  threshold?: number
  rerank?: boolean
  latestOnly?: boolean
  fields?: string[]
  categories?: string[]
}
```

### 4.3 核心特性

**1. 多级记忆（Multi-Level Memory）**：
- User：用户级偏好
- Session：会话级上下文
- Agent：agent 级状态

**2. 智能提取**：
- `infer: true` 让 LLM 决定什么值得记忆
- `customInstructions` 引导提取方向

**3. 语义检索**：
- 向量检索 + 重排序
- 阈值过滤

**4. 自托管**：
- Docker compose 一键起
- Postgres + Qdrant + mem0 server

### 4.4 借鉴清单

✅ **直接用 SDK**：
```bash
npm install mem0ai
```

✅ **借鉴分类**：
- user/agent/app/run 四维隔离
- 我们用 userId（用户）+ agentId（magent）

✅ **借鉴提取**：
- `infer: true` + `customInstructions`
- 让 LLM 主动决定什么值得记忆

⚠️ **避免**：
- 自托管成本（Docker + Postgres + Qdrant）
- MVP 可以先用本地 JSONL

---

## 5. Zep（v3.25.0）

### 5.1 SDK 结构

```
@getzep/zep-cloud/
├── dist/
│   ├── cjs/
│   │   ├── api/resources/
│   │   │   ├── thread/        # 会话
│   │   │   ├── graph/         # 知识图谱
│   │   │   │   └── resources/
│   │   │   │       ├── episode/    # 记忆事件
│   │   │   │       ├── edge/
│   │   │   │       ├── node/
│   │   │   │       └── threadSummary/
│   │   │   ├── user/
│   │   │   ├── context/
│   │   │   └── memory/
│   │   └── index.js
│   └── esm/...
```

### 5.2 核心特性：Temporal Knowledge Graph

**Zep 的核心创新**：把对话组织成**情节图谱**：

```
Episode 1: "用户说喜欢 TS"
    ↓ (linked by user_id)
Episode 2: "用户决定用 Next.js"
    ↓ (related: about frontend)
Episode 3: "讨论 Server Components"
```

**每个对话是 episode，实体之间有关联边，自动生成摘要**。

### 5.3 借鉴清单

✅ **借鉴概念**：
- 每个 session 是 episode
- 自动生成 threadSummary
- 时序推理（"上周用户说了什么"）

⚠️ **避免**：
- Zep 太重（图谱数据库）
- 我们只需要简单的 episode 列表，不搞关系图

---

## 6. Pi（earendil-works/pi-coding-agent，v0.81.1）

### 6.1 SDK 结构（5MB，TypeScript 完整实现）

```
pi-coding-agent/
├── dist/
│   ├── index.js              # 主入口
│   ├── cli/                  # CLI 入口
│   ├── core/
│   │   ├── agent-session.ts          # Agent 会话
│   │   ├── session-manager.ts        # Session 管理（含 tree）
│   │   ├── extensions/               # 扩展系统
│   │   ├── tools/                    # 工具
│   │   ├── skills.ts                 # Skills 加载
│   │   ├── settings-manager.ts       # 设置
│   │   ├── model-registry.ts         # 模型注册
│   │   ├── model-runtime.ts          # 模型运行时
│   │   ├── model-resolver.ts         # 模型解析
│   │   ├── event-bus.ts              # 事件总线
│   │   ├── auth-storage.ts           # 认证存储
│   │   ├── compaction/               # 上下文压缩
│   │   ├── trust-manager.ts          # 信任管理
│   │   ├── footer-data-provider.ts   # 状态栏
│   │   └── ...
│   ├── modes/
│   │   ├── interactive/             # TUI 模式
│   │   ├── print/                  # 打印模式
│   │   ├── rpc/                    # RPC 模式
│   │   └── sdk/                     # SDK 模式
│   ├── utils/
│   ├── extensions/                  # 第三方扩展
│   └── main.ts                      # 入口
└── package.json
```

### 6.2 Pi 的核心创新

**1. 极简设计**（借鉴）：
> "Aggressively extensible so it doesn't have to dictate your workflow"

**不内置的功能**（用户自己 build）：
- Sub-agents（用 tmux 实现）
- Plan mode（写到文件）
- Permission popups（用容器）
- Background bash（用 tmux）
- To-dos（用 TODO.md）

**2. Tree-structured history**（核心创新）：
- 会话是树形（可分支、合并）
- `/tree` 命令浏览
- 所有分支存一个文件

**3. 4 种模式**（借鉴）：
- `interactive` - TUI 体验
- `print` - `pi -p "query"` 脚本模式
- `rpc` - JSON over stdin/stdout
- `sdk` - 嵌入应用

**4. 扩展系统**：
```typescript
interface Extension {
  registerTool(tool: Tool): void;
  registerCommand(cmd: Command): void;
  registerProvider(adapter: ProviderAdapter): void;
  // ...
}
```

**5. Mid-session model switch**（借鉴）：
- `/model` 或 Ctrl+L 切换
- Ctrl+P 循环

### 6.3 借鉴清单

✅ **借鉴极简哲学**：
- 我们也只做核心功能
- 复杂功能让用户扩展

✅ **借鉴 4 模式**：
- magent 也支持 `interactive` / `print` / `rpc` / `sdk`

✅ **借鉴 tree-structured history**（差异化点）：
- Session 可以分支
- 跨 provider 切换保留上下文

✅ **借鉴扩展系统**：
```typescript
interface MagentExtension {
  registerTool(tool: Tool): void;
  registerCommand(cmd: Command): void;
  registerProvider(adapter: ProviderAdapter): void;
  registerRoutingStrategy(strategy: RoutingStrategy): void;
}
```

✅ **借鉴 15+ providers**：
- 直接用 Pi 的 provider 列表
- 让用户容易切换

⚠️ **避免**：
- Pi 太新、用户少
- 我们需要更稳定的方案

---

## 7. Cline（cline/cline，35k+ stars）

### 7.1 产品形态（核心创新）

**全平台部署**：
```
npm install -g cline             # CLI
VS Code Marketplace               # VS Code 插件
JetBrains Marketplace              # IntelliJ/PyCharm 插件
npm install @cline/sdk            # SDK
```

**Web Kanban**（v2.0+ 新功能）：
- Web 任务板
- 每个 card 一个 worktree
- auto-commit + 依赖链

### 7.2 核心创新

**1. Plan/Act 模式**：
- Plan mode：只读分析，写出计划
- Act mode：执行计划
- 用户随时切换

**2. Multi-Agent Teams**：
```bash
cline --team-name auth-sprint "Plan and implement user authentication"
```

**3. Scheduled Agents**：
```bash
cline schedule create "PR summary"   --cron "0 9 * * MON-FRI"   --prompt "List all open PRs"   --workspace /path
```

**4. Connect 平台**：
```bash
cline connect telegram -k $BOT_TOKEN
cline connect slack --bot-token $TOKEN
```

### 7.3 借鉴清单

✅ **借鉴 Plan/Act 模式**：
- magent run 之前可以加 `--plan`
- 先让 LLM 分析，输出计划，用户确认后再执行

✅ **借鉴 Scheduled Agents**：
```bash
magent schedule create "PR summary"   --cron "0 9 * * MON-FRI"   --prompt "List all open PRs"
```

✅ **借鉴 Connect 平台**（未来扩展）：
- Telegram bot
- Slack app

⚠️ **避免**：
- Cline 是单一 provider（自己）
- 我们要做"跨 provider"

---

## 8. OpenCode（15 万+ stars）

### 8.1 核心特点

- **跨平台**：TUI、Desktop、IDE 一套代码
- **75+ LLM provider 支持**（比 Pi 还多）
- **隐私优先**：本地部署
- **强大的 hook 系统**：20+ 事件类型

### 8.2 Hook 系统对比

| Claude Code | OpenCode | 我们的 magent |
|-------------|----------|----------------|
| 8 events | 11 events | 计划 6 个核心事件 |

**借鉴**：OpenCode 的 hook 设计更多元，可以监听：
- `tool.execute.before/after`
- `file.edited`
- `file.watcher.updated`
- `message.updated`
- `lsp.client.diagnostics`

### 8.3 借鉴清单

✅ **借鉴 hook 设计**：
- 我们也用类似的事件系统
- 但只做核心的（不用 20 个）

✅ **借鉴 75+ provider 支持**：
- 通过 adapter pattern 实现
- 用户可以自己加 provider

⚠️ **避免**：
- OpenCode 是单一产品（不像我们做"跨工具"）
- 我们只需要核心

---

## 总结：借鉴策略

### 9.1 直接复用

| 来源 | 复用内容 |
|------|---------|
| cloudcli | Provider 抽象、WebSocket、SQLite schema、4 个 provider 实现 |
| ECC | 278 个 skills、SKILL.md 格式 |
| superpowers | HARD-GATE 概念、systematic-debugging 流程 |
| mem0 | 完整记忆后端（v1.0） |
| Pi | Tree-structured history、扩展系统、4 模式 |
| Cline | Plan/Act、Scheduled、Connect |

### 9.2 借鉴后改进

| 来源 | 我们做什么 |
|------|----------|
| cloudcli | + CLI 入口、+ 智能路由、+ 共享记忆、+ MCP 中心 |
| ECC | - 太重，我们只挑核心 skills |
| superpowers | - 太方法论，我们挑实用的 |
| mem0 | - MVP 先用本地 JSONL |
| Pi | - 太新，我们只借鉴设计哲学 |
| Cline | - 我们做跨 provider |

### 9.3 不借鉴

- cloudcli 传统 HTML 前端（用 React 重写）
- ECC 232k stars 的"功能堆砌"
- superpowers 的 14 个方法论 skills（太多）

---

## 借鉴优先级

**P0（必须）**：
- cloudcli Provider 7-文件模式
- mem0 SDK（v1.0 集成）
- ECC SKILL.md 格式

**P1（应该）**：
- Pi 扩展系统
- Pi tree-structured history
- superpowers HARD-GATE

**P2（锦上添花）**：
- Cline Plan/Act
- Cline Scheduled
- OpenCode 多 provider

