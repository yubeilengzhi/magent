# 2026 年新项目竞品分析（补充）

> 基于刚才下载的 8 个新项目源码分析，补充 ARCHITECTURE.md 和 COMPETITIVE_ANALYSIS.md。

## 新增的 7 个项目

| 项目 | Stars | 语言 | License | 核心定位 |
|------|-------|------|---------|---------|
| **smtg-ai/claude-squad** | 8,167 | Go | AGPL-3.0 | "管理多个 AI 终端 agent（Claude Code/Codex/Gemini）" |
| **Kilo-Org/kilocode** | 26,473 | TypeScript | MIT | "all-in-one agentic engineering platform" |
| **Gentleman-Programming/engram** | 5,643 | Go | MIT | **"Persistent memory for AI agents" - 单 binary + SQLite + MCP** |
| **iOfficeAI/AionUi** | 30,709 | TypeScript | Apache 2.0 | "24/7 Cowork app，20+ CLI 集成" |
| **1jehuang/jcode** | 10,895 | Rust | MIT | "最智能的 agent harness for code" |
| **esengine/DeepSeek-Reasonix** | 27,638 | Go | MIT | "DeepSeek-native coding agent" |
| **chenhg5/cc-connect** | 14,308 | Go | MIT | **"桥接 16+ agent 到 messaging 平台"** |
| **jnMetaCode/superpowers-zh** | 7,163 | Markdown | MIT | **"superpowers 中文版 + 4 个中国原创 skills"** |

## 9. claude-squad（直接竞品）

### 定位
> "Manage multiple AI terminal agents like Claude Code, Codex, OpenCode, and Amp"

### 架构（Go 写的）

```
claude-squad/
├── app/              # 主 TUI（应用层）
│   ├── app.go
│   └── help.go
├── cmd/              # 子命令
│   └── cmd.go
├── daemon/           # 后台进程（不同 OS）
│   ├── daemon.go
│   ├── daemon_unix.go
│   └── daemon_windows.go
└── config/           # 配置
```

**核心抽象**：
```go
type Instance struct {
    Title     string  // 用户标签
    Path      string  // workspace
    Program   string  // claude/codex/...
    Status    Status  // running/paused/done
    Branch    string  // git branch
    DiffStats string  // 改动统计
}
```

### 借鉴清单

✅ **直接借鉴**：
- "Instance 概念"（每个 session 是一个 instance）
- 多 agent 在独立 workspace 并行
- 后台 daemon 进程（macOS launchd / Linux systemd）
- TUI 状态展示（实例列表、状态切换）

⚠️ **避免**：
- AGPL-3.0 协议（要避免传染）
- 单一 workspace 隔离（我们要做"跨 workspace"）

### 差异化机会

- **claude-squad** 是"多 instance 并行"（同一 agent 多次）
- **magent** 是"跨 agent 切换"（一次跑一个）

**互补关系**：可以基于 claude-squad 包装一层"跨 agent 路由"

---

## 10. engram（持久化记忆，**最相关**）

### 定位
> "Persistent memory for AI coding agents - Go binary + SQLite + FTS5 + MCP"

### 架构（174 个 Go 文件，113K 行！）

```
engram/
├── cmd/                  # CLI
│   ├── engram/           # 主命令
│   ├── cloud/            # Cloud 版本
│   ├── diagnostic/       # 诊断
│   ├── llm/              # LLM 客户端
│   ├── mcp/              # MCP server
│   ├── obsidian/         # Obsidian 集成
│   ├── project/          # 项目管理
│   ├── server/           # HTTP server
│   ├── setup/            # 引导
│   ├── store/            # 数据层
│   └── sync/             # 同步
├── internal/
│   ├── store/            # SQLite + FTS5 存储
│   ├── llm/              # LLM 客户端（Claude/OpenCode）
│   ├── mcp/              # MCP server
│   └── ...
├── docker/               # 自托管
├── plugin/               # 插件
├── docs/                 # 完整文档
└── openspec/             # OpenSpec 规范
```

### 核心特性

**1. 单一 binary**：
- 一个 Go 二进制
- SQLite + FTS5（全文搜索）
- 自带 MCP server
- 自带 HTTP API
- 自带 CLI
- 自带 TUI

**2. Agent 无关**：
- 任何支持 MCP 的 agent 都能用
- Claude Code、OpenCode、Gemini CLI、Codex、VS Code、Antigravity、Cursor、Windsurf

**3. 工作原理**：
```
Agent (Claude Code / OpenCode / ...)
    ↓ MCP stdio
Engram (single Go binary)
    ↓
SQLite + FTS5 (~/.engram/engram.db)
```

### 借鉴清单

✅ **直接借鉴（最大价值）**：
- **MCP server 设计**（让所有 agent 通过 MCP 访问记忆）
- **SQLite + FTS5 全文搜索**（比 JSONL 强 100 倍）
- **单一 binary**（部署简单）
- **Agent 无关**（通过 MCP）

✅ **借鉴架构**：
```go
type Store struct {
    db *sql.DB
    // 记忆存储
    // 关系图（engram 有 relations）
    // FTS5 全文搜索
}

type MCPServer struct {
    store *Store
    // 提供 memory_save, memory_search, memory_get 等 tools
}
```

⚠️ **避免**：
- engram 太重（113K 行，5 年开发）
- 我们 MVP 用简化版

### 对 magent 的影响

**engram 实际上**是 magent 想要做的"记忆层"的**直接参考实现**！

我们要做：
- MVP：本地 JSONL（够用）
- v1.0：可以 fork engram 的 MCP server 设计
- 或集成 engram 作为后端

---

## 11. AionUi（30k stars，桌面 app）

### 定位
> "Free, local, open-source 24/7 Cowork app for OpenClaw, Hermes Agent, Claude Code, Codex, OpenCode, Gemini CLI and 20+ more CLI"

### 关键特性

- **桌面 app**（Electron + UnoCSS）
- **多 agent 集成**（20+ CLI）
- **24/7 自动化**（cron）
- **远程访问**（WebUI + Telegram/Lark/DingTalk/WeChat）
- **零配置**（内置 agent engine）

### 借鉴清单

✅ **借鉴远程访问**：
- WebUI（基础）
- Telegram/Lark 集成（v2.0）

✅ **借鉴多 agent 集成方式**：
- 21 个内置 assistants
- 用户加 API key 就用
- 不强求装 CLI

⚠️ **避免**：
- Electron 太重（我们是 CLI 优先）
- 24/7 是亮点，但我们是开发工具不是监控

### 差异化机会

- AionUi 是"桌面 app"（用户跑在自己电脑）
- **magent** 是"CLI 工具"（用户通过命令行）

**互补关系**：magent 可以是 AionUi 的后端（CLI 模式）

---

## 12. cc-connect（桥接 16+ agent）

### 定位
> "Bridge local AI coding agents (Claude Code, Cursor, Gemini CLI, Codex) to messaging platforms"

### 关键特性

- **16 个 agent adapter**：
  - acp, antigravity, claudecode, codex, copilot, cursor, devin, gemini, iflow, kimi, opencode, pi, qoder, reasonix, tmux

- **8+ messaging 平台**：
  - Feishu, Lark, DingTalk, Telegram, Slack, Discord, WeCom

### 借鉴清单

✅ **直接借鉴**：
- **16 个 agent adapter 设计**（架构上完整）
- **MCP-based 桥接**（不直接 spawn，更稳定）
- **多平台 abstraction**（adapter pattern）

✅ **借鉴 adapter 列表**（我们要支持）：
- claude-code, codex, opencode, cursor, gemini, kimi, pi, qoder, reasonix, copilot, devin

⚠️ **避免**：
- cc-connect 主要是 messaging 桥接（不是 CLI 工具）
- 我们要 CLI 优先

### 对 magent 的影响

**cc-connect 给了我们完整的 16 个 agent 列表**——这就是市场上有需求的所有 CLI 工具。

**必支持的 8 个**（基于 star 数 + 活跃度）：
1. Claude Code
2. Codex
3. OpenCode
4. Cursor
5. Gemini CLI
6. Pi
7. Qoder（字节）
8. Copilot

**可支持的 5 个**（长尾）：
- Antigravity, Aider, Cline, Continue, Devin

---

## 13. superpowers-zh（中文增强版）

### 定位
> "superpowers（116k+ ⭐）完整汉化 + 4 个中国原创 skills"

### 关键差异

| 原版 superpowers | superpowers-zh |
|------------------|----------------|
| 14 skills | 20 skills |
| 英文 | 简体中文 + 繁体 |
| 9 harness | **20 harness**（加了国产 IDE） |
| 西方项目流程 | **4 个中国原创** |

### 4 个中国原创 skills

1. `chinese-code-review` - 中国团队代码审查
2. `chinese-commit-conventions` - 中文 commit 规范
3. `chinese-documentation` - 中文文档写作
4. `chinese-git-workflow` - Git workflow 中文

### 借鉴清单

✅ **直接借鉴**：
- **20 个 skills 全部**（中文适配版更友好）
- **支持 20 个 harness**（cc-connect 的 16 + 国产 IDE）
- **npm 安装** `npm i -g superpowers-zh`

⚠️ **避免**：
- 中文项目维护活跃度要观察
- 部分 skill 可能不适用国际项目

### 对 magent 的影响

**我们不需要自己写 skills**——直接集成 superpowers-zh 的 20 个 skills！

或：
- **v1.0 直接用 npm 包**
- **magent skill install superpowers-zh**（一行命令）

---

## 14. 其他 3 个项目

### Kilo Code（26k stars）

**定位**：VS Code fork + CLI
**借鉴价值**：UI 设计、500+ 模型集成
**关系**：是竞品不是组件

### jcode（10k stars，Rust）

**定位**："The most intelligent agent harness for code"
**借鉴价值**：Rust 写的性能优势
**关系**：设计思路参考

### DeepSeek-Reasonix（27k stars）

**定位**：DeepSeek-native coding agent
**借鉴价值**：prefix-cache 优化（让 agent 长时间运行）
**关系**：是 provider 不是工具

---

## 重大调整：基于 2026 项目的差异化重定位

### 原版 ARCHITECTURE 假设

> magent = 1 个 CLI 入口 + 智能路由 + 跨工具记忆 + 共享配置

### 2026 现实

| 项目 | 已经做了 |
|------|---------|
| **engram** | ✅ 跨 agent 持久化记忆（通过 MCP） |
| **claude-squad** | ✅ 多 agent 并行管理（Go TUI） |
| **AionUi** | ✅ 多 agent 集成（20+ CLI 桌面 app） |
| **cc-connect** | ✅ 16+ agent 桥接 + messaging 集成 |
| **superpowers-zh** | ✅ 跨 20 harness skills 框架 |
| **Kilo Code** | ✅ VS Code + CLI + 500 模型 |

### 我们的真正差异化

**选项 A：做"engram 的 CLI 版"**
- 跨工具记忆是核心
- 不做路由（让用户自己选）
- **轻量、CLI 优先**

**选项 B：做"claude-squad + 智能路由"**
- 多 agent 并行 + 智能选哪个
- **更复杂的 UI**

**选项 C：做"AionUi 的 CLI 版"**
- 多 agent 集成（20+ CLI）
- 24/7 自动化
- **桌面 app + CLI 双形态**

**选项 D：做"engram + claude-squad + 智能路由"（全做）**
- 记忆 + 多 agent + 路由
- **最完整，但最难做**

### 我的建议：**选项 A**（最聚焦）

**magent = engram 的 CLI 轻量版 + 智能路由**

理由：
1. engram 已经做了 60%（记忆 + MCP）
2. claude-squad 已经做了 80%（多 agent 管理）
3. **真正空白的是"跨 agent 路由 + 跨 agent 记忆的统一 CLI"**
4. 我们聚焦这个，价值密度最高

**具体定位**：
- 不重做 engram（用 engram 作为记忆后端）
- 不重做 claude-squad（他们的多 agent GUI 更好）
- **做"智能路由器"，让用户在多 agent 之间无缝切换**

---

## 调整后的产品形态

### 方案：magent = "Router + 统一 CLI"

```
User
  ↓
magent run "task"  ← 一个 CLI 入口
  ↓
[magent router]
  ├─ → Claude Code (via claude-squad / 直接)
  ├─ → Codex (via @openai/codex-sdk)
  ├─ → OpenCode (via binary)
  ├─ → Cursor CLI
  └─ → Pi / Qoder / etc.
  ↓
[共享记忆层]
  ├─ → engram (via MCP) - 长期记忆
  └─ → 本地 JSONL (MVP) - 短期
  ↓
[共享 Skills]
  └─ → superpowers-zh (20 skills, 20 harness)
```

### 借鉴策略（更新）

| 来源 | 借鉴 | 集成方式 |
|------|------|---------|
| **engram** | 跨 agent 记忆 | **MCP 客户端**（用 engram 作为后端） |
| **claude-squad** | 多 agent 管理 | **参考**（不重复造轮子） |
| **AionUi** | 多 agent 集成 | **参考 agent 列表** |
| **cc-connect** | 16+ agent adapter | **参考 adapter 设计** |
| **superpowers-zh** | 20 skills | **npx 安装**（subprocess） |
| **Kilo Code** | 500+ 模型 | **参考**（我们不需要） |
| **mem0** | 记忆后端 | **备选**（v2.0） |

### 不再重复造轮子

- ❌ 不重做 engram 的记忆层
- ❌ 不重做 claude-squad 的 TUI
- ❌ 不重写 16 个 agent adapter
- ✅ **聚焦：智能路由 + 统一 CLI + 跨工具切换**

### 新的"魔数"（核心能力）

**让用户在 1 个 CLI 里，跨 16+ agent 切换，记忆共享，配置共享**

具体 demo：
```bash
# 1. 在 Codex 跑 30 分钟
magent run "重构 X" --provider codex

# 2. token 满了，迁移到 Claude Code
magent migrate --to claude-code

# 3. 切到 Pi 跑测试
magent run "跑测试" --provider pi

# 4. 跨工具记忆
magent memory add "我更喜欢用 TypeScript"
# 自动应用到所有 16 个 agent
```

**这就是 engram + claude-squad + 智能路由的组合**

---

## 调整后的实施路线图

### MVP（4 周）✅ 最小可行

**Week 1：基础设施**
- [ ] CLI 入口（Node.js + Commander）
- [ ] 配置系统（YAML）
- [ ] Provider adapter 接口
- [ ] **集成 engram**（作为记忆后端，通过 MCP）
- [ ] **集成 superpowers-zh**（作为 skills 库，npx）

**Week 2：Codex 集成**
- [ ] CodexAdapter（用 @openai/codex-sdk）
- [ ] Session 管理
- [ ] CLI 基础命令（run, session, model, provider）
- [ ] **不写本地记忆**（用 engram）

**Week 3：Claude Code 集成**
- [ ] ClaudeCodeAdapter（用 spawn）
- [ ] 跨 provider session 迁移
- [ ] 路由决策（基础 LLM）

**Week 4：发布**
- [ ] 智能路由
- [ ] 第一个 release
- [ ] npm publish
- [ ] 集成文档

### v1.0（3 个月）

- [ ] 16+ agent adapter（基于 cc-connect 设计）
- [ ] 完整智能路由
- [ ] Web UI（轻量）
- [ ] superpowers-zh 深度集成
- [ ] 跨 agent 记忆迁移

### v2.0（6 个月）

- [ ] 多 agent 并行（参考 claude-squad）
- [ ] messaging 集成（参考 cc-connect）
- [ ] 远程访问（参考 AionUi）
- [ ] Team features
- [ ] 商业版

---

## 总结：基于 2026 新项目的关键决策

### 决策 1：用 engram 作为记忆后端
- **不再自己实现记忆层**
- engram 已经是产品级（5.6k stars, 113K 行 Go）
- 通过 MCP 集成

### 决策 2：用 superpowers-zh 作为 skills 库
- **不再自己写 skills**
- superpowers-zh 已经是 20 skills × 20 harness
- 通过 npx subprocess 集成

### 决策 3：支持 16+ agent
- 参考 cc-connect 的 agent 列表
- 必支持 8 个：Claude Code、Codex、OpenCode、Cursor、Gemini CLI、Pi、Qoder、Copilot

### 决策 4：聚焦核心
- 不做桌面 app（AionUi 做了）
- 不做 messaging 桥接（cc-connect 做了）
- 不做 TUI 多 instance（claude-squad 做了）
- **只做"智能路由 + 统一 CLI + 跨工具切换"**

### 决策 5：MVP 路径
- Week 1：engram + superpowers-zh 集成
- Week 2-3：2 个 provider 跑通
- Week 4：发布 alpha
