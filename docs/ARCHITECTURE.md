# magent - 产品架构设计文档 v0.1

> **核心定位**：第一个跨工具的智能编程助手（CLI 入口 + 智能路由 + 跨工具记忆 + 共享配置）

## 目录

1. [愿景与差异化](#1-愿景与差异化)
2. [竞品深度分析](#2-竞品深度分析)
3. [产品形态](#3-产品形态)
4. [技术架构](#4-技术架构)
5. [核心子系统设计](#5-核心子系统设计)
6. [数据流](#6-数据流)
7. [存储格式](#7-存储格式)
8. [API 设计](#8-api-设计)
9. [实施路线图](#9-实施路线图)
10. [风险与权衡](#10-风险与权衡)
11. [借鉴的开源项目细节](#11-借鉴的开源项目细节)

---

## 1. 愿景与差异化

### 1.1 一句话定位

> **magent = 1 个 CLI 入口 + 智能路由 + 跨工具记忆（mem0）+ 共享配置层（ECC/superpowers 模型）+ Web UI（可选）**

### 1.2 核心问题（用户真实痛点）

| 痛点 | 现状 | 用户反馈 |
|------|------|---------|
| **工具启动繁琐** | 每个 CLI 单独 `cd` + 启动 | "我每天要在 4 个工具之间切换" |
| **MCP 配置重复** | 5 个 harness 各写一遍 | "filesystem MCP 我配了 5 次" |
| **记忆不共享** | Claude 知道的偏好 Codex 不知道 | "我每次都要重新告诉 agent 我喜欢 TS" |
| **路由靠记忆** | 用户自己判断用哪个 | "我得自己记住哪个任务用哪个工具" |
| **Session 不互通** | Codex 跑了 50%，切 Claude 重新开始 | "上下文断了很烦" |

### 1.3 我们的差异化（与已有项目对比）

| 维度 | cloudcli | ECC | superpowers | Cline | Pi | **magent** |
|------|---------|-----|-------------|-------|-----|---------|
| 统一 CLI 入口 | ✅（Web） | ❌ | ❌ | ✅（CLI） | ✅（TUI） | ✅ **跨工具 CLI** |
| 智能路由 | ❌ | ⚠️ 内嵌 | ❌ | ❌ | ❌ | ✅ **LLM 决策** |
| 跨工具记忆 | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ **mem0 集成** |
| MCP 共享中心 | ❌ | ⚠️ 文件复制 | ⚠️ 文件复制 | ❌ | ❌ | ✅ **中心化** |
| Skills 共享中心 | ❌ | ✅ | ✅ | ⚠️ | ✅ | ✅ **中心化** |
| 模型池统一 | ⚠️ 部分 | ❌ | ❌ | ❌ | ❌ | ✅ **统一** |
| Web UI | ✅ | ❌ | ❌ | ⚠️ Kanban | ❌ | ✅ **可选** |
| Provider 数量 | 4 | 9+ | 9+ | 1 | 1 | ✅ **4 起步，按需加** |
| 安装方式 | npm install | 插件市场 | 插件市场 | npm install | npm install | **npm install + 适配器** |

### 1.4 价值主张

**对个人开发者**：
- 1 个命令用所有 AI 工具
- 不用记配置（自动共享）
- 不用选工具（智能路由）
- 不用重复说偏好（跨工具记忆）

**对团队**：
- 统一 MCP 配置（一次配，全员用）
- 统一技能库（团队最佳实践共享）
- 统一审计（所有 agent 行为可追溯）
- 统一计费（统计每个 provider 用量）

---

## 2. 竞品深度分析

### 2.1 cloudcli（51K 行 Node.js）

**已下载源码**：`sources/cloudcli.tgz` (4.3MB)

**架构**：
\`\`\`
dist-server/
├── server/
│   ├── index.js              # Express + WebSocket 入口
│   ├── cli.js                # CLI 命令
│   ├── claude-sdk.js         # Claude Code SDK 封装
│   ├── openai-codex.js       # Codex SDK 封装
│   ├── cursor-cli.js         # Cursor CLI 封装
│   ├── opencode-cli.js       # OpenCode CLI 封装
│   ├── load-env.js           # 环境变量加载
│   ├── modules/
│   │   ├── providers/        # 4 个 provider 的统一抽象
│   │   │   ├── list/{claude,codex,cursor,opencode}/
│   │   │   ├── services/
│   │   │   │   ├── provider-models.service.js
│   │   │   │   ├── provider-auth.service.js
│   │   │   │   └── sessions.service.js
│   │   │   ├── provider.routes.js
│   │   │   └── provider.registry.js
│   │   ├── websocket/        # WS 实时通信
│   │   ├── database/         # SQLite schema
│   │   ├── notifications/
│   │   ├── assets/
│   │   ├── projects/
│   │   └── browser-use/
│   ├── routes/               # Express routes
│   ├── shared/               # 共享工具
│   ├── utils/                # 工具函数
│   └── middleware/
├── shared/                   # 前后端共享
│   └── networkHosts.js
└── dist/                     # 前端（传统 HTML）
\`\`\`

**核心接口**（每个 provider 都有）：
- `*.provider.js` - 主入口（~18 行）
- `*-auth.provider.js` - 认证
- `*-mcp.provider.js` - MCP 配置
- `*-models.provider.js` - 模型列表
- `*-sessions.provider.js` - session 流处理
- `*-session-synchronizer.provider.js` - 文件监控
- `*-skills.provider.js` - 技能系统

**借鉴的价值**：
- ✅ Provider 抽象模式（每个 provider 拆 6 个文件）
- ✅ WebSocket 实时通信
- ✅ SQLite schema 设计
- ✅ Session 文件同步机制

**改进**：
- ❌ 单一入口（Web only，CLI 没有）
- ❌ 没有智能路由
- ❌ 没有跨工具记忆
- ❌ MCP 配置不集中

### 2.2 mem0（v3.1.1，JavaScript SDK + Python）

**已下载源码**：`sources/mem0.tgz` (788KB)

**核心 API**：
\`\`\`typescript
interface MemoryClient {
  add(messages: Message[], options?: AddMemoryOptions): Promise<Memory[]>
  search(query: string, options?: SearchMemoryOptions): Promise<SearchResult>
  get(memoryId: string): Promise<Memory>
  getAll(options?: GetAllMemoryOptions): Promise<PaginatedMemories>
  update(memoryId: string, body: MemoryUpdateBody): Promise<Memory>
  delete(memoryId: string, options?: DeleteMemoryOptions): Promise<void>
  deleteAll(options?: DeleteAllMemoryOptions): Promise<void>
}
\`\`\`

**借鉴的价值**：
- ✅ 完整的多维度记忆管理（user/agent/app/run）
- ✅ 语义检索 + 重排序
- ✅ 时间衰减
- ✅ 结构化数据提取
- ✅ **支持自托管（OSS）和云服务**

**借鉴到 magent**：
- 用 `add()` 自动从对话提取记忆
- 用 `search()` 在 agent 启动前注入相关记忆
- 用 `userId` 隔离多用户
- 用 `agentId` 隔离多 agent

### 2.3 Zep（v3.25.0，TypeScript SDK）

**核心结构**：
\`\`\`
api/resources/
├── thread/         # 会话管理
├── graph/          # 知识图谱
│   └── resources/
│       ├── episode/    # 记忆事件
│       ├── edge/       # 关系
│       ├── node/       # 实体
│       └── threadSummary/
├── user/           # 用户管理
└── context/        # 上下文窗口
\`\`\`

**特点**：
- ✅ **Temporal Knowledge Graph（时序知识图谱）**
- ✅ 每个对话是 episode
- ✅ 实体之间有关联边
- ✅ 自动生成摘要
- ✅ 时序推理（"上周用户说了什么"）

### 2.4 ECC（affaan-m/ECC）

**核心结构**：
\`\`\`
agents/        # 67 个 Markdown agent 定义
skills/        # 278 个 Markdown skill 定义
commands/      # 94 个 slash 命令
rules/         # always-follow 规则
hooks/         # 运行时钩子
mcp-configs/   # MCP 配置
contexts/      # 动态上下文注入
\`\`\`

**Skill 格式**：
\`\`\`markdown
---
name: skill-name
description: Use when... (触发条件)
metadata:
  origin: ECC
---

# Skill Title
主体内容...
\`\`\`

**借鉴的价值**：
- ✅ 278 个现成 skills 可直接复用
- ✅ cross-harness 支持（Claude Code、Cursor、Codex、OpenCode、Pi 等 7+ 个）
- ✅ SKILL.md 标准格式

### 2.5 superpowers（obra/superpowers）

**支持的 9+ harness**：
Claude Code、Codex、Cursor、OpenCode、Pi、Antigravity、Factory Droid、GitHub Copilot CLI、Kimi Code

**借鉴价值**：
- ✅ 跨 9+ harness 的统一 skills 框架
- ✅ 每个 skill 有 `<HARD-GATE>` 强制执行点

### 2.6 Pi（earendil-works/pi-coding-agent，v0.81.1）

**核心特点**：
- **极简设计**：不做 sub-agent、plan mode、permission popups、background bash
- **Tree-structured history**：会话可分支、合并
- **15+ providers**：Anthropic、OpenAI、Google、Azure、Bedrock、Mistral、Groq、Cerebras、xAI、HuggingFace、Kimi、MiniMax、NVIDIA、OpenRouter、Ollama
- **Mid-session model switch**
- **Context engineering**：AGENTS.md、SYSTEM.md、Skills、Prompt templates
- **4 种模式**：interactive、print/JSON、RPC、SDK
- **扩展系统**：基于 TypeScript

**Pi 的核心创新**：
- ✅ **Tree-structured session history**（其他都没有）
- ✅ **可分享的会话 URL**（`/share` 生成 GitHub gist）
- ✅ **导出 HTML**（`/export`）
- ✅ **扩展系统**（用户可以构建自己的功能）

### 2.7 Cline（cline/cline）

**核心特点**：
- ✅ **CLI + Kanban + VS Code + JetBrains + SDK** 全平台
- ✅ **Plan/Act 模式**：先 plan 后 act
- ✅ **Multi-Agent Teams**：coordinator 分发任务
- ✅ **Scheduled Agents**：cron 调度
- ✅ **Connect 平台**：Telegram、Slack、Discord
- ✅ **Headless CLI**：CI/CD 集成

### 2.8 OpenCode（opencode-ai）

**核心特点**：
- ✅ **15 万+ GitHub stars**
- ✅ **75+ LLM provider 支持**
- ✅ **三平台**：TUI、Desktop、IDE
- ✅ **隐私优先**：本地部署

---


## 3. 产品形态

### 3.1 CLI（核心入口）

```bash
# 安装
npm install -g magent

# 基本使用
magent run "重构这个文件"              # 智能路由
magent run --provider codex "快速 fix" # 显式指定
magent run --provider claude-code "深度分析" 

# 会话管理
magent session list                  # 列出所有 session（跨工具）
magent session show <id>            # 显示 session 详情
magent session share <id>            # 分享 session URL

# 记忆管理
magent memory search "用户偏好"       # 搜索记忆
magent memory add "我喜欢 TS"        # 添加记忆
magent memory list                   # 列出所有记忆

# 配置管理
magent config show                   # 显示当前配置
magent config edit                   # 编辑配置
magent mcp list                      # 列出 MCP
magent mcp install <name>            # 安装 MCP
magent skill list                    # 列出 skills
magent skill install <name>          # 安装 skill

# 模型管理
magent model list                    # 列出所有可用模型
magent model route "这个任务该用啥"  # 智能路由

# 提供商管理
magent provider list                 # 列出已配置 provider
magent provider add codex            # 添加 provider
magent provider auth codex           # 认证 provider
```

### 3.2 Web UI（可选）

```bash
magent web                            # 启动 Web UI（默认 20001）
magent web --port 8080
```

**Web UI 功能**：
- 聊天界面（实时显示 agent 思考过程）
- 多 session 列表
- 记忆浏览器（可视化、可编辑）
- 路由决策日志
- Provider 状态
- 用量统计

### 3.3 SDK（给其他工具嵌入）

```typescript
import { Agent, Router, Memory } from 'magent/sdk';

const router = new Router({
  model: 'haiku',
  providers: ['codex', 'claude-code'],
  memory: new Memory({ backend: 'mem0' }),
});

const session = await router.run({
  task: '帮我重构这个文件',
  context: { files: ['./src/foo.ts'] },
});

for await (const event of session.stream()) {
  console.log(event);
}
```

---

## 4. 技术架构

### 4.1 总体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        magent CLI / Web                          │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│ Router       │    │ Memory Layer     │    │ Config Layer │
│ (智能路由)    │    │ (mem0)           │    │ (中心化配置)  │
└──────────────┘    └──────────────────┘    └──────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Provider Abstraction Layer                     │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│   │ Codex   │  │Claude   │  │ Pi      │  │ Cursor  │  ...   │
│   │ Adapter │  │Code     │  │ Adapter │  │ Adapter │         │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘         │
└─────────────────────────────────────────────────────────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│ Codex SDK    │    │ Claude Code SDK  │    │ Pi SDK       │
└──────────────┘    └──────────────────┘    └──────────────┘
```

### 4.2 核心原则

1. **不重写 wheel**：每个 provider 直接调用官方 SDK
2. **Plugin 化架构**：新 provider 通过 adapter 接入
3. **中心化数据**：配置、记忆、skills 都中心化存储
4. **智能路由**：基于 LLM 决策（不硬编码）
5. **跨工具**：所有数据可在 provider 之间共享

### 4.3 技术栈

| 层 | 技术选择 | 理由 |
|---|---------|------|
| CLI | Node.js 20+ + Commander.js | 用户环境兼容 |
| 包管理 | npm / pnpm | 与下游兼容 |
| 数据库 | SQLite (better-sqlite3) | 本地、无依赖、快 |
| HTTP 服务 | Express + WebSocket | 与 cloudcli 一致 |
| Memory 后端 | mem0（OSS + 自托管） | 业界最强 |
| LLM（路由） | 任意 OpenAI 兼容（cliproxy） | 用户灵活 |
| 前端 | React + Vite（可选） | 现代化 |
| 配置格式 | YAML（可读）+ JSON（机器） | 友好 |
| 测试 | Vitest + 真实 SDK 集成 | 端到端 |

---

## 5. 核心子系统设计

### 5.1 Router（智能路由引擎）

#### 职责
- 接收用户任务
- 决定用哪个 provider
- 决定用哪个模型
- 决定是否需要 sub-agent 协作
- 持续观察 session，必要时切换

#### 路由决策算法

```typescript
async function route(task: Task, context: Context): Promise<RouteDecision> {
  // 1. 快速规则匹配（毫秒级）
  const ruleMatch = matchRules(task, context);
  if (ruleMatch.confidence > 0.95) {
    return ruleMatch;
  }

  // 2. LLM 决策（200-500ms）
  const prompt = buildRouterPrompt(task, context, providers, history);
  const decision = await llm.complete(prompt, { model: 'haiku' });
  
  return parseDecision(decision);
}
```

#### Router Prompt 模板

```typescript
const ROUTER_PROMPT = `
你是 AI 工具路由器。根据任务和上下文，选择最合适的 provider 和模型。

## 可用 Providers
${providers.map(p => `
- ${p.name}:
  - 能力: ${p.capabilities.join(', ')}
  - 速度: ${p.avgLatency}ms
  - 当前负载: ${p.currentLoad}
  - 状态: ${p.status}
  - 配额剩余: ${p.quotaRemaining ?? 'unlimited'}
`).join('\n')}

## 历史表现（按任务类型）
${history.map(h => `
- 任务类型: ${h.taskType}
- Provider: ${h.provider}
- 成功: ${h.success ? '✓' : '✗'}
- 延迟: ${h.latency}ms
- 用户评分: ${h.userRating ?? '-'}/5
`).join('\n')}

## 当前任务
${task.description}

## 上下文
- 涉及文件: ${context.files?.join(', ') ?? 'none'}
- 估算大小: ${context.estimatedSize ?? 'unknown'} tokens
- 时间约束: ${context.timeConstraint ?? 'normal'}

## 输出 JSON
{
  "provider": "推荐 provider 名称",
  "model": "推荐模型",
  "reason": "为什么这样选",
  "confidence": 0.0-1.0,
  "fallback": "备选 provider"
}
`;
```

#### 持续路由（session 中切换）

```typescript
async function monitor(session: Session): Promise<RouteChange | null> {
  if (session.toolCallCount > 10 && session.latency > 30_000) {
    return shouldReRoute(session);
  }
  if (session.totalTokens > session.contextWindow * 0.7) {
    return suggestModelDowngrade(session);
  }
  if (session.errorCount > 2) {
    return suggestFallback(session);
  }
  return null;
}
```

### 5.2 Memory Layer（记忆层）

#### 职责
- 跨 session 持久化用户偏好
- 跨 provider 共享项目上下文
- 跨工具路由历史
- 主动检索相关记忆注入到 prompt

#### 数据分类

```typescript
interface MemoryRecord {
  id: string;
  type: 'preference' | 'project' | 'decision' | 'routing' | 'session' | 'pattern';
  scope: 'user' | 'project' | 'workspace' | 'session';
  content: string;
  metadata: Record<string, any>;
  embedding?: number[];
  createdAt: Date;
  expiresAt?: Date;
  source: string;
}
```

#### 存储位置

```
~/.magent/
├── memory/
│   ├── user-preferences.jsonl     # 用户偏好（永久）
│   ├── project-context.jsonl      # 项目背景（项目级）
│   ├── decisions.jsonl            # 决策历史
│   ├── routing-history.jsonl      # 路由历史
│   ├── session-summaries/         # 会话摘要（30 天）
│   └── patterns.jsonl             # 学习模式
└── mem0-data/                     # mem0 自有存储
```

#### 注入时机

```typescript
async function startProvider(provider: Provider, task: Task): Promise<Session> {
  const memories = await memory.search(task.description, {
    topK: 10,
    filters: {
      scope: ['user', 'project'],
      expiresAt: { $gt: new Date() },
    },
  });

  const systemPrompt = buildSystemPrompt({
    basePrompt: provider.defaultSystemPrompt,
    userPreferences: memories.filter(m => m.type === 'preference'),
    projectContext: memories.filter(m => m.type === 'project'),
    recentDecisions: memories.filter(m => m.type === 'decision'),
    routingHistory: memories.filter(m => m.type === 'routing'),
  });

  const session = await provider.startSession({
    task,
    systemPrompt,
  });

  session.on('complete', async () => {
    await saveSessionSummary(session);
  });

  return session;
}
```

#### 主动提取

```typescript
async function extractMemories(session: Session): Promise<void> {
  const messages = session.transcript;
  await mem0.add(messages, {
    userId: getUserId(),
    agentId: 'magent',
    metadata: {
      sessionId: session.id,
      provider: session.provider,
      duration: session.duration,
    },
    infer: true,
    customInstructions: `
      优先记忆：
      1. 用户偏好（编程语言、框架、风格）
      2. 项目背景（项目类型、技术栈）
      3. 重要决策（为什么选这个方案）
      4. 工作习惯（喜欢先写测试再写代码）
      忽略：
      - 临时任务（"今天帮我重构 X"）
      - 工具调用细节
      - 错误信息
    `,
  });
}
```

### 5.3 Config Layer（配置层）

#### 数据结构

```typescript
interface MagentConfig {
  version: string;
  user: {
    id: string;
    name?: string;
    preferences: Record<string, any>;
  };
  
  providers: Record<string, ProviderConfig>;
  
  memory: {
    backend: 'mem0' | 'local';
    config: {
      apiKey?: string;
      host?: string;
      topK?: number;
      threshold?: number;
    };
  };
  
  router: {
    model: string;
    rules?: RoutingRule[];
    enableLearning: boolean;
  };
  
  mcp: {
    servers: Record<string, MCPServerConfig>;
  };
  
  skills: {
    sources: SkillSource[];
  };
  
  web: {
    port: number;
    enabled: boolean;
  };
}
```

#### 配置文件位置

```
~/.magent/
├── config.yaml                # 主配置
├── providers/
│   ├── codex.yaml             # Codex 配置
│   ├── claude-code.yaml       # Claude Code 配置
│   └── ...
├── mcp/
│   └── servers.yaml           # MCP 服务器配置（中心化）
├── skills/
│   ├── installed/             # 已安装技能
│   └── custom/                # 用户自定义
└── routing/
    ├── rules.yaml             # 路由规则
    └── history.jsonl          # 路由历史
```

#### MCP 配置分发

```typescript
class MCPDistributor {
  private servers = loadMCPConfig('~/.magent/mcp/servers.yaml');
  
  async distributeTo(providerName: string): Promise<void> {
    const config = this.servers;
    switch (providerName) {
      case 'codex':
        await this.distributeToCodex(config);
        break;
      case 'claude-code':
        await this.distributeToClaudeCode(config);
        break;
    }
  }
  
  private async distributeToCodex(config: MCPConfig): Promise<void> {
    const target = `${process.env.HOME}/.codex/config.toml`;
    const existing = await fs.readFile(target, 'utf8');
    const merged = mergeTOML(existing, configToTOML(config));
    await fs.writeFile(target, merged);
  }
}
```

### 5.4 Provider Adapter（适配器层）

#### 接口设计

```typescript
interface ProviderAdapter {
  name: string;
  capabilities: Capability[];
  models: ModelInfo[];
  status: 'ready' | 'busy' | 'error' | 'auth_required';
  
  initialize(config: ProviderConfig): Promise<void>;
  authenticate(): Promise<AuthResult>;
  
  startSession(opts: StartSessionOptions): Promise<ProviderSession>;
  resumeSession(id: string, opts: ResumeOptions): Promise<ProviderSession>;
  
  watchFiles?(path: string): AsyncIterable<FileEvent>;
  
  shutdown(): Promise<void>;
}

interface ProviderSession {
  id: string;
  provider: string;
  model: string;
  
  events: AsyncIterable<SessionEvent>;
  send(message: string, attachments?: Attachment[]): Promise<void>;
  abort(): Promise<void>;
  
  status: 'starting' | 'running' | 'paused' | 'completed' | 'failed';
  toolCalls: ToolCall[];
  totalTokens: number;
  
  startedAt: Date;
  completedAt?: Date;
}

interface SessionEvent {
  type: 'thinking' | 'tool-call' | 'tool-result' | 'message' | 'error' | 'done';
  content: any;
  timestamp: Date;
}
```

#### Codex Adapter（参考实现）

```typescript
class CodexAdapter implements ProviderAdapter {
  name = 'codex';
  capabilities = ['file-ops', 'long-context', 'fast', 'reasoning'];
  
  async startSession(opts: StartSessionOptions): Promise<CodexSession> {
    const { Codex } = await import('@openai/codex-sdk');
    const codex = new Codex({
      env: {
        ...process.env,
        OPENAI_API_KEY: this.config.apiKey,
        CODEX_INTERNAL_ORIGINATOR_OVERRIDE: 'magent',
      }
    });
    
    const thread = codex.startThread({
      model: opts.model || this.config.defaultModel,
      workingDirectory: opts.cwd,
      sandboxMode: opts.permissionMode || 'workspace-write',
    });
    
    return new CodexSession(thread, opts);
  }
}

class CodexSession extends BaseSession {
  async *events(): AsyncIterable<SessionEvent> {
    const stream = await this.thread.runStreamed(this.currentInput);
    for await (const event of stream.events) {
      yield this.transformEvent(event);
    }
  }
}
```

### 5.5 Skills Manager（技能管理）

```typescript
interface SkillSource {
  type: 'marketplace' | 'git' | 'local';
  url: string;
  name: string;
  installedAt: Date;
  enabled: boolean;
}

class SkillsManager {
  private sources: SkillSource[] = [];
  private installed: Map<string, Skill> = new Map();
  
  async install(source: string): Promise<void> {
    if (source.startsWith('ecc:')) {
      return this.installFromECC(source);
    }
    if (source.startsWith('git:') || source.startsWith('https://')) {
      return this.installFromGit(source);
    }
  }
  
  async distribute(skill: Skill): Promise<void> {
    for (const provider of this.getEnabledProviders()) {
      await this.distributeTo(provider, skill);
    }
  }
  
  async invoke(skillName: string, context: any): Promise<string> {
    const skill = this.installed.get(skillName);
    return skill.body;
  }
}
```

### 5.6 Session Manager（会话管理）

借鉴 Pi 的 tree-structured history：

```typescript
interface SessionTree {
  id: string;
  root: SessionNode;
  currentPath: string[];
}

interface SessionNode {
  id: string;
  parentId?: string;
  children: string[];
  provider: string;
  model: string;
  startedAt: Date;
  endedAt?: Date;
  transcript: SessionEvent[];
  summary?: string;
  branchPoint?: number;
  label?: string;
}

class SessionManager {
  async branch(sessionId: string, branchPoint: number, provider: string): Promise<SessionNode> {
    const parent = this.getNode(sessionId);
    const newNode: SessionNode = {
      id: generateUUID(),
      parentId: parent.id,
      provider,
    };
    return newNode;
  }
  
  // 跨 provider session 迁移
  async migrateTo(nodeId: string, newProvider: string): Promise<SessionNode> {
    const old = this.getNode(nodeId);
    const transcript = this.serializeTranscript(old);
    const summary = await this.summarizeForMigration(old);
    
    const newSession = await this.providers[newProvider].startSession({
      cwd: old.cwd,
      systemPrompt: `${summary}\n\n继续之前的任务：`,
    });
    
    return newSession;
  }
}
```

---


## 6. 数据流

### 6.1 典型用户旅程

```
1. 用户输入: magent run "重构 src/foo.ts"
   ↓
2. CLI 解析参数，查询记忆
   ↓
3. Memory.search("重构 src/foo.ts")
   ↓ 返回: [
     {type: 'preference', content: '用户喜欢 TypeScript strict mode'},
     {type: 'project', content: '项目 X 用 Next.js + tRPC'},
     {type: 'decision', content: '2026-07-20 决定用 Prisma'}
   ]
   ↓
4. Router.route(task, context)
   ↓ Router prompt + memory context + provider info + history
   ↓ LLM 返回: { provider: 'claude-code', model: 'opus', reason: '重构任务，需要深度理解', confidence: 0.92 }
   ↓
5. ClaudeCodeAdapter.startSession({
     task,
     systemPrompt: built with memories,
   })
   ↓
6. Session 启动，WebSocket 流式输出到 UI
   ↓
7. Session 完成，自动提取记忆
   ↓
8. mem0.add(transcript, {infer: true})
   ↓
9. 路由历史记录: { taskType: 'refactor', provider: 'claude-code', success: true, latency: 45s }
   ↓
10. 用户可在 UI 看 / routing 看决策过程
```

### 6.2 跨 Provider 会话迁移流程

```
1. 用户在 Codex 跑了 30 分钟，token 接近上限
   ↓
2. SessionManager.monitor() 检测到 token > 70%
   ↓
3. 询问用户: "Token 接近上限。要切换到 claude-code 继续吗？"
   ↓ 用户确认
4. SessionManager.migrateTo('claude-code')
   ↓
5. 生成 session 摘要（用 mem0）
   ↓
6. claude-code adapter 用摘要作为 system prompt 启动新 session
   ↓
7. 在 UI 显示 "迁移完成，新 session ID: xxx"
   ↓
8. 保留原 Codex session（标记为已迁移）
```

### 6.3 MCP 配置同步流程

```
1. 用户运行: magent mcp install filesystem
   ↓
2. MCP 配置中心化存储: ~/.magent/mcp/servers.yaml
   ↓
3. 检测已配置的 providers
   ↓
4. 对每个 provider 转换格式：
   - Codex → ~/.codex/config.toml 的 [mcp_servers.filesystem]
   - Claude Code → ~/.claude.json 的 mcpServers.filesystem
   - Cursor → ~/.cursor/mcp.json
   - Pi → ~/.pi/mcp.json
   ↓
5. 写入各 provider 配置（不删除已有，只添加）
   ↓
6. 重启相应 provider（如果它正在运行）
```

---

## 7. 存储格式

### 7.1 主配置文件

`~/.magent/config.yaml`：
```yaml
version: 1
user:
  id: dustking
  name: "Dust King"

providers:
  codex:
    enabled: true
    defaultModel: qwen3.7-plus
    apiKey: sk-xxx
    baseUrl: http://43.137.15.66:8627/v1
    capabilities:
      - file-ops
      - long-context
      - reasoning
  claude-code:
    enabled: true
    defaultModel: sonnet
    capabilities:
      - file-ops
      - deep-reasoning
      - plan-mode
  pi:
    enabled: false
  cursor:
    enabled: false

memory:
  backend: mem0
  config:
    apiKey: sk-mem0-xxx
    host: http://localhost:8000
    topK: 10
    threshold: 0.7

router:
  model: haiku
  enableLearning: true
  rules:
    - pattern: "^(fix|改|小修)"
      provider: codex
      confidence: 0.9
    - pattern: "^(重构|refactor|重写)"
      provider: claude-code
      confidence: 0.85

mcp:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/data/root"]
  context7:
    command: npx
    args: ["-y", "@upstash/context7-mcp"]

skills:
  sources:
    - type: marketplace
      url: ecc
      enabled: true
    - type: git
      url: https://github.com/obra/superpowers
      enabled: true

web:
  port: 20001
  enabled: true
```

### 7.2 记忆文件

`~/.magent/memory/user-preferences.jsonl`：
```jsonl
{"id": "u-1", "type": "preference", "scope": "user", "content": "用户喜欢 TypeScript strict mode", "source": "codex", "createdAt": "2026-07-23T10:00:00Z"}
{"id": "u-2", "type": "preference", "scope": "user", "content": "用户在早上工作效率高", "source": "claude-code", "createdAt": "2026-07-23T11:00:00Z"}
{"id": "u-3", "type": "decision", "scope": "user", "content": "2026-07-23: 项目 X 决定用 Prisma 而非 Drizzle，因为团队更熟悉", "source": "codex", "createdAt": "2026-07-23T15:00:00Z"}
```

### 7.3 路由历史

`~/.magent/routing/history.jsonl`：
```jsonl
{"timestamp": "2026-07-23T10:00:00Z", "taskType": "refactor", "provider": "claude-code", "model": "opus", "success": true, "latencyMs": 45000, "userRating": 5}
{"timestamp": "2026-07-23T11:00:00Z", "taskType": "fix", "provider": "codex", "model": "qwen3.7-plus", "success": true, "latencyMs": 5000, "userRating": 5}
```

---

## 8. API 设计

### 8.1 CLI 命令

```bash
magent <command> [options]

Commands:
  run [task]               启动一个新任务（智能路由）
  continue [session-id]    继续一个 session
  session                  会话管理
  memory                   记忆管理
  model                    模型管理
  provider                 provider 管理
  mcp                      MCP 管理
  skill                    技能管理
  route                    路由管理
  config                   配置管理
  web                      启动 Web UI
  daemon                   后台守护进程
  update                   更新 magent
  version                  版本信息
  help                     帮助

Options:
  -p, --provider <name>     强制指定 provider
  -m, --model <name>        强制指定 model
  -c, --cwd <path>           工作目录
  --headless                无 UI 模式
  --auto-approve            自动批准（危险）
  --memory <query>          注入额外记忆
  --no-memory               不使用记忆
  --router-model <name>     路由决策用哪个 model
  --verbose                  详细输出
```

### 8.2 编程接口（SDK）

```typescript
import { Agent, Router, Memory } from 'magent';

// 高级：智能路由
const session = await Agent.run({
  task: '重构这个文件',
  cwd: './',
  overrideProvider: 'codex',
});

// 流式获取输出
for await (const event of session.events) {
  console.log(event);
}

// 低级：手动控制
const router = new Router({ /* config */ });
const decision = await router.route({
  task: '...',
  context: { /* ... */ },
});

const codex = new CodexAdapter({ /* config */ });
const session = await codex.startSession({
  task: '...',
  systemPrompt: '...',
});
```

---


## 9. 实施路线图

### 9.1 MVP（4 周）

**Week 1：基础设施**
- [ ] 项目脚手架（TypeScript + pnpm）
- [ ] 配置系统（YAML 解析）
- [ ] Provider adapter 接口定义
- [ ] 记忆抽象层（不接 mem0，先本地 JSONL）
- [ ] 路由抽象层（先简单规则）

**Week 2：Codex 集成**
- [ ] CodexAdapter（调用 @openai/codex-sdk）
- [ ] Session 管理
- [ ] CLI 基础命令（run, session, model, provider）
- [ ] 本地记忆存储
- [ ] 智能路由（基础 LLM 决策）

**Week 3：Claude Code 集成**
- [ ] ClaudeCodeAdapter
- [ ] 跨 provider session 迁移
- [ ] 路由历史学习
- [ ] 配置文件分发

**Week 4：发布准备**
- [ ] Web UI（基础聊天界面）
- [ ] 文档（README、quickstart）
- [ ] 第一个 release
- [ ] npm publish

### 9.2 v1.0（3 个月）

- [ ] mem0 完整集成（替换本地 JSONL）
- [ ] 4 个 provider（Codex、Claude Code、Pi、Cursor）
- [ ] MCP 共享中心（自动分发到所有 provider）
- [ ] Skills manager（ECC + superpowers 集成）
- [ ] tree-structured session history
- [ ] 路由学习（从历史提升准确率）
- [ ] 完整 Web UI
- [ ] 文档站

### 9.3 v2.0（6 个月）

- [ ] Team features（共享记忆）
- [ ] Cline Kanban 风格的多 agent 编排
- [ ] Scheduled agents（cron）
- [ ] Connect 平台（Telegram、Slack）
- [ ] VS Code / JetBrains 插件
- [ ] 企业版（SSO、审计）

---

## 10. 风险与权衡

### 10.1 风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| mem0 API 变化 | 中 | 用 SDK 抽象，定期测试 |
| Provider SDK 升级 | 高 | 锁定版本，semver 严格 |
| 路由决策慢 | 中 | 用小模型 + 缓存 |
| 记忆污染 | 中 | 让用户能编辑/删除 |
| Token 成本 | 中 | 默认不记忆只主动提取关键内容 |

### 10.2 权衡

**Q1：记忆后端用本地还是 mem0？**
- 本地：免费、私密、但功能弱
- mem0：强大、但要 cloud 或 self-host
- **决定**：MVP 先本地 JSONL，v1.0 接 mem0（OSS 模式可自托管）

**Q2：智能路由用小模型 vs 大模型？**
- 小模型（haiku、flash）：快、便宜
- 大模型：准确
- **决定**：用小模型，cache 历史决策，复杂任务升级

**Q3：CLI 还是 Web UI？**
- CLI：开发者友好
- Web：可视化
- **决定**：CLI 主、Web 辅

**Q4：fork cloudcli 还是重写？**
- fork：复用 5 万行代码、省 6 个月
- 重写：完全控制、但工作量大
- **决定**：fork cloudcli 的 provider 代码，重写 CLI 入口和路由

**Q5：开源协议？**
- MIT：宽松、商业友好
- Apache 2.0：专利友好
- **决定**：MIT（参考 ECC、superpowers）

---

## 11. 借鉴的开源项目细节

### 11.1 cloudcli 借鉴清单

**直接复用**：
- `dist-server/server/modules/database/` (SQLite schema)
- `dist-server/server/modules/websocket/` (WebSocket 框架)
- `dist-server/server/middleware/auth.js` (JWT)
- `dist-server/server/routes/` (Express routes)
- `dist-server/server/modules/providers/services/` (provider 抽象)
- 4 个 provider 的具体实现（Codex、Claude Code、Cursor、OpenCode）

**重新设计**：
- 加 CLI 入口（cloudcli 只有 Web）
- 加智能路由（cloudcli 没有）
- 加跨 provider 记忆（cloudcli 没有）
- 加 MCP 共享中心（cloudcli 每个 provider 各自配置）
- 加 tree-structured session（cloudcli 是线性的）

**不参考**：
- 传统 HTML 前端（用 React + Vite 重写）
- 单一 provider registry（改为 plugin 系统）

### 11.2 mem0 借鉴清单

**直接使用 SDK**：
```bash
npm install mem0ai
```

**核心 API**：
```typescript
import { MemoryClient } from 'mem0ai';

const client = new MemoryClient({ apiKey: 'xxx' });

// 存
await client.add(messages, {
  userId: 'dustking',
  agentId: 'magent',
  infer: true,
  customInstructions: '优先记忆用户偏好和项目背景',
});

// 查
const memories = await client.search('用户喜欢什么编程语言', {
  userId: 'dustking',
  topK: 10,
  rerank: true,
});
```

### 11.3 ECC 借鉴清单

**直接复用 278 个 skills**：
- ECC marketplace URL：`/plugin marketplace add https://github.com/affaan-m/ECC`
- magent 解析 SKILL.md frontmatter
- 注入到对应 provider 的 prompt

### 11.4 superpowers 借鉴清单

**借鉴的 HARD-GATE 概念**：
```markdown
<HARD-GATE>
Do NOT invoke any implementation skill until you have presented a design
</HARD-GATE>
```

### 11.5 Pi 借鉴清单

**借鉴的扩展系统**：
```typescript
interface MagentExtension {
  registerTool(tool: Tool): void;
  registerCommand(cmd: Command): void;
  registerProvider(adapter: ProviderAdapter): void;
  registerRoutingStrategy(strategy: RoutingStrategy): void;
}
```

### 11.6 Zep 借鉴清单

**借鉴的 episode 概念**：
- 每个 session 是一次 episode
- 自动生成 threadSummary

### 11.7 Cline 借鉴清单

- Plan/Act 模式（路由前先判断）
- Headless CLI
- Scheduled Agents
- Connect 平台

---

## 附录 A：代码示例

### A.1 完整路由决策流程

```typescript
// src/core/router.ts
import OpenAI from 'openai';
import type { Task, Context, Provider, RouteDecision } from './types';

export class Router {
  private llm: OpenAI;
  
  constructor(private config: RouterConfig) {
    this.llm = new OpenAI({
      baseURL: config.baseUrl,
      apiKey: config.apiKey,
    });
  }
  
  async route(task: Task, context: Context): Promise<RouteDecision> {
    const history = await this.loadHistory(task.taskType);
    const providers = await this.loadProviders();
    
    // 规则匹配（fast path）
    const rule = this.matchRule(task, context);
    if (rule && rule.confidence > 0.95) {
      await this.recordDecision(rule, 'rule');
      return rule;
    }
    
    // LLM 决策（slow path）
    const decision = await this.llmDecide(task, context, providers, history);
    await this.recordDecision(decision, 'llm');
    return decision;
  }
  
  private async llmDecide(
    task: Task,
    context: Context,
    providers: Provider[],
    history: RoutingHistory[],
  ): Promise<RouteDecision> {
    const prompt = `
你是 AI 工具路由器。基于任务和上下文，选择最合适的 provider 和模型。

## 可用 Providers
${providers.map(p => this.formatProvider(p)).join('\n')}

## 历史表现
${history.map(h => this.formatHistory(h)).join('\n')}

## 当前任务
${task.description}

## 上下文
- Files: ${context.files?.join(', ') ?? 'none'}
- Size: ${context.estimatedSize ?? 'unknown'} tokens
- Constraint: ${context.timeConstraint ?? 'normal'}

## 输出（仅 JSON）
{
  "provider": "name",
  "model": "model-name",
  "reason": "explanation",
  "confidence": 0.0-1.0,
  "fallback": "backup-provider"
}
    `;
    
    const response = await this.llm.chat.completions.create({
      model: this.config.model,
      messages: [{ role: 'system', content: prompt }],
      response_format: { type: 'json_object' },
    });
    
    return JSON.parse(response.choices[0].message.content);
  }
}
```

### A.2 跨工具记忆注入

```typescript
// src/core/memory-injector.ts
import { MemoryClient } from 'mem0ai';
import type { Provider, Task, Context } from './types';

export class MemoryInjector {
  constructor(private memory: MemoryClient) {}
  
  async buildSystemPrompt(
    basePrompt: string,
    task: Task,
    context: Context,
    userId: string,
  ): Promise<string> {
    const memories = await this.memory.search(task.description, {
      filters: { user_id: userId },
      topK: 10,
      rerank: true,
      categories: ['preference', 'project_context', 'recent_decisions'],
    });
    
    if (!memories.results?.length) {
      return basePrompt;
    }
    
    const grouped = this.groupMemories(memories.results);
    
    return `${basePrompt}

## User Preferences
${grouped.preferences.map(m => `- ${m.memory}`).join('\n') || 'None'}

## Project Context
${grouped.project.map(m => `- ${m.memory}`).join('\n') || 'None'}

## Recent Decisions
${grouped.decisions.map(m => `- ${m.memory}`).join('\n') || 'None'}
`;
  }
  
  private groupMemories(memories: any[]) {
    return {
      preferences: memories.filter(m => m.metadata?.category === 'preference'),
      project: memories.filter(m => m.metadata?.category === 'project_context'),
      decisions: memories.filter(m => m.metadata?.category === 'recent_decisions'),
    };
  }
}
```

---

## 附录 B：参考链接

| 项目 | URL | 借鉴内容 |
|------|-----|---------|
| cloudcli | https://github.com/cloudcli-ai/cloudcli | Provider 抽象、Web UI、SQLite |
| ECC | https://github.com/affaan-m/ecc | 278 skills、SKILL.md 格式 |
| superpowers | https://github.com/obra/superpowers | HARD-GATE 概念、跨 9 harness |
| mem0 | https://github.com/mem0ai/mem0 | 记忆 SDK、自托管 |
| Zep | https://github.com/getzep/zep | Episode 概念、threadSummary |
| Pi | https://pi.dev/ | 扩展系统、tree history、4 模式 |
| Cline | https://github.com/cline/cline | Plan/Act、Scheduled、Connect |
| OpenCode | https://github.com/anomalyco/opencode | 跨平台部署、15 万+ stars |
| james AgentHub | https://github.com/jamesrochabrun/AgentHub | 多 session 监控（参考） |

---

## 附录 C：决策记录

### C.1 用什么记忆后端？

**评估**：
- mem0：业界标准、有 cloud 和 OSS、API 简洁 ✅
- Zep：时序图谱、重，但对我们要"跨工具"太重 ❌
- LangMem：绑死 LangChain ❌
- 自建：质量差 ❌

**决定**：mem0（v1.0 集成）

### C.2 用什么路由策略？

**评估**：
- 硬编码规则：不灵活 ❌
- LLM 决策：成本 + 延迟 ⚠️
- 规则 + LLM fallback（先用规则，高置信度返回；否则 LLM）：✅

**决定**：规则 + LLM fallback

### C.3 用什么 provider SDK？

**评估**：
- Codex SDK：`@openai/codex-sdk` ✅
- Claude Code SDK：需要鉴权（cloudcli 用 spawn）⚠️
- Pi SDK：Node.js SDK（@earendil-works/pi-coding-agent）✅
- Cursor CLI：CLI 包装 ⚠️

**决定**：
- Codex → `@openai/codex-sdk`
- Claude Code → 借鉴 cloudcli 的 spawn 方式
- Pi → Pi SDK
- Cursor → CLI 包装

### C.4 命名

**评估**：
- `agent-hub`：被占用 ⚠️
- `agent-router`：聚焦路由 ✅
- `agentconsole`：聚焦控制台 ✅
- `agentmesh`：太抽象 ⚠️
- `magent`：简短好记 ✅

**决定**：`magent`（meta-agent / multi-agent）

---

**最后更新**：2026-07-23
**作者**：dustking + Claude
**版本**：0.1
**状态**：设计阶段，待实施
