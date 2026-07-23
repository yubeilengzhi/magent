# magent 项目

> **第一个跨工具的智能编程助手**（CLI 入口 + 智能路由 + 跨工具记忆 + 共享配置）

## 📚 文档

1. **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — 完整产品架构（1506 行）
   - 愿景、竞品分析、技术架构
   - 5 个核心子系统设计
   - 数据流、存储格式、API
   - 实施路线图、风险评估

2. **[docs/COMPETITIVE_ANALYSIS.md](docs/COMPETITIVE_ANALYSIS.md)** — 8 个项目竞品分析（756 行）
3. **[docs/COMPETITIVE_ANALYSIS_2026.md](docs/COMPETITIVE_ANALYSIS_2026.md)** — **8 个 2026 新项目**（504 行）
   - engram、claude-squad、AionUi、cc-connect、superpowers-zh 等

4. **[docs/POC_CODE.md](docs/POC_CODE.md)** — MVP 代码骨架（826 行）

5. **[INDEX.md](INDEX.md)** — 文档索引

## 🎯 核心定位（2026 修订版）

**magent = 1 个 CLI 入口 + 智能路由 + engram 持久化记忆 + 共享 skills 层（superpowers-zh）+ Web UI（可选）**

**核心差异化**（在 2026 年的 16+ 类似项目中）：
- 别人做"单一 agent"（claude-squad、AionUi、cc-connect）
- 别人做"单一功能"（engram 是记忆、superpowers 是 skills）
- **magent 做"统一 CLI + 智能路由"，让用户 1 个命令跨 16+ agent**

## 💡 与 2026 年最新项目的对比

| 我们的能力 | engram | claude-squad | AionUi | cc-connect | superpowers-zh |
|----------|--------|--------------|--------|-----------|----------------|
| 统一 CLI 入口 | ❌ | ✅（Go TUI） | ✅（Electron） | ❌ | ❌ |
| **智能路由** | ❌ | ❌ | ❌ | ❌ | ❌ |
| 跨工具记忆 | ✅（自做） | ❌ | ⚠️（弱） | ❌ | ❌ |
| 16+ agent 集成 | ❌ | ✅（4 个） | ✅（20+） | ✅（16） | ✅（20 harness） |
| 共享 skills | ❌ | ❌ | ⚠️（21 个） | ❌ | ✅（20 skills） |

**结论**：**没有人同时做"统一 CLI + 智能路由 + 跨工具记忆"**——这是我们的真正差异化。

## 🔬 调研过的开源项目（已下载源码分析）

### 第一批：8 个项目
| 项目 | Stars | 语言 | 用途 |
|------|-------|------|------|
| [cloudcli](https://github.com/cloudcli-ai/cloudcli) | 1.36k | Node.js | Provider 抽象、WebSocket |
| [ECC](https://github.com/affaan-m/ecc) | 232k | Markdown | 278 skills、SKILL.md 格式 |
| [superpowers](https://github.com/obra/superpowers) | 259k | Markdown | HARD-GATE、systematic-debugging |
| [mem0](https://github.com/mem0ai/mem0) | 35k | Python/JS | 记忆后端 SDK |
| [Zep](https://github.com/getzep/zep) | 5k | TypeScript | Episode 概念、时序图谱 |
| [Pi](https://pi.dev/) | 76k | TypeScript | 极简哲学、tree history、扩展系统 |
| [Cline](https://github.com/cline/cline) | 35k+ | TypeScript | Plan/Act、Scheduled、Connect |
| [OpenCode](https://github.com/anomalyco/opencode) | 140k+ | Go/TS | 75+ provider、跨平台 |

### 第二批：8 个 2026 新项目
| 项目 | Stars | 语言 | 用途 |
|------|-------|------|------|
| [claude-squad](https://github.com/smtg-ai/claude-squad) | 8.2k | Go | 多 agent 终端管理 |
| [Kilo Code](https://github.com/Kilo-Org/kilocode) | 26k | TypeScript | VS Code + CLI + 500 模型 |
| [engram](https://github.com/Gentleman-Programming/engram) | 5.6k | Go | **持久化记忆 + MCP**（直接相关！）|
| [AionUi](https://github.com/iOfficeAI/AionUi) | 30k | TypeScript | 24/7 Cowork app |
| [jcode](https://github.com/1jehuang/jcode) | 11k | Rust | "最智能的 agent harness" |
| [DeepSeek-Reasonix](https://github.com/esengine/DeepSeek-Reasonix) | 27k | Go | DeepSeek-native |
| [cc-connect](https://github.com/chenhg5/cc-connect) | 14k | Go | **桥接 16+ agent 到 messaging** |
| [superpowers-zh](https://github.com/jnMetaCode/superpowers-zh) | 7.2k | Markdown | **superpowers 中文版 + 4 个中国原创** |

## 🚀 立即开始

```bash
# 看架构
cat docs/ARCHITECTURE.md

# 看 2026 竞品分析（关键）
cat docs/COMPETITIVE_ANALYSIS_2026.md

# 复制 PoC 跑起来
mkdir magent
cd magent
# 复制 docs/POC_CODE.md 里的代码
npm install
npm run dev -- run "重构这个文件"
```

## 📋 实施路线图（2026 修订版）

**MVP（4 周）** - **不重复造轮子**：
- Week 1: 集成 engram（记忆）+ superpowers-zh（skills），CLI 入口
- Week 2: Codex 集成（@openai/codex-sdk）
- Week 3: Claude Code 集成（spawn）
- Week 4: 智能路由 + npm publish

**v1.0（3 个月）**：
- 16+ agent adapter（基于 cc-connect 设计）
- 完整智能路由
- 跨 provider 记忆迁移
- superpowers-zh 深度集成
- 完整 Web UI

**v2.0（6 个月）**：
- 多 agent 并行（参考 claude-squad）
- messaging 集成（参考 cc-connect）
- 远程访问（参考 AionUi）
- Team features
- 商业版

## 🤝 借鉴策略（核心决策）

**不重复造轮子，集成现成项目**：

| 项目 | 角色 | 集成方式 |
|------|------|---------|
| **engram** | 记忆后端 | MCP client |
| **superpowers-zh** | skills 库 | npx subprocess |
| **claude-squad** | 多 agent GUI（参考） | 不集成 |
| **AionUi** | 20+ agent 列表（参考） | 自己实现 adapter |
| **cc-connect** | 16 agent adapter（参考） | 自己实现 |
| **mem0** | 记忆备选 | v2.0 考虑 |

## 📜 License

MIT
