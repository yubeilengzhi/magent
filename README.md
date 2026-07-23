# magent 项目

> **第一个跨工具的智能编程助手**（CLI 入口 + 智能路由 + 跨工具记忆 + 共享配置）

## 📚 文档

1. **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — 完整产品架构（1482 行）
   - 愿景、竞品分析、技术架构
   - 5 个核心子系统设计
   - 数据流、存储格式、API
   - 实施路线图、风险评估

2. **[docs/COMPETITIVE_ANALYSIS.md](docs/COMPETITIVE_ANALYSIS.md)** — 竞品深度分析（756 行）
   - 8 个开源项目源码分析
   - 每个项目的可借鉴点 + 我们要改进什么

3. **[docs/POC_CODE.md](docs/POC_CODE.md)** — MVP 代码骨架（826 行）
   - 完整可运行的 PoC
   - 包含 package.json、TypeScript 代码、CLI 入口

4. **[INDEX.md](INDEX.md)** — 文档索引

## 🎯 核心定位

**magent = 1 个 CLI 入口 + 智能路由 + 跨工具记忆（mem0）+ 共享配置层（ECC/superpowers 模型）+ Web UI（可选）**

## 💡 与现有项目的差异化

| 维度 | cloudcli | ECC | superpowers | Cline | Pi | **magent** |
|------|---------|-----|-------------|-------|-----|---------|
| 统一 CLI 入口 | ✅（Web） | ❌ | ❌ | ✅（CLI） | ✅（TUI） | ✅ **跨工具 CLI** |
| 智能路由 | ❌ | ⚠️ 内嵌 | ❌ | ❌ | ❌ | ✅ **LLM 决策** |
| 跨工具记忆 | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ **mem0 集成** |
| MCP 共享中心 | ❌ | ⚠️ 文件复制 | ⚠️ 文件复制 | ❌ | ❌ | ✅ **中心化** |
| Skills 共享中心 | ❌ | ✅ | ✅ | ⚠️ | ✅ | ✅ **中心化** |
| 模型池统一 | ⚠️ 部分 | ❌ | ❌ | ❌ | ❌ | ✅ **统一** |
| Provider 数量 | 4 | 9+ | 9+ | 1 | 1 | ✅ **4 起步，按需加** |

## 🔬 本地源码分析（不提交到 git）

调研时下载了 8 个开源项目的源码，本地分析后未上传到仓库：

| 项目 | 版本 | 用途 |
|------|------|------|
| [cloudcli](https://github.com/cloudcli-ai/cloudcli) | 1.36.3 | Provider 抽象、WebSocket |
| [ECC](https://github.com/affaan-m/ecc) | latest | 278 skills、SKILL.md 格式 |
| [superpowers](https://github.com/obra/superpowers) | latest | HARD-GATE、systematic-debugging |
| [mem0](https://github.com/mem0ai/mem0) | 3.1.1 | 记忆后端 SDK |
| [Zep](https://github.com/getzep/zep) | 3.25.0 | Episode 概念、时序图谱 |
| [Pi](https://pi.dev/) | 0.81.1 | 极简哲学、tree history、扩展系统 |
| [Cline](https://github.com/cline/cline) | CLI 0.0.13 | Plan/Act、Scheduled、Connect |
| [OpenCode](https://github.com/anomalyco/opencode) | 1.18.4 | 75+ provider、跨平台 |

## 🚀 立即开始

### 查看架构

```bash
# 完整架构
cat docs/ARCHITECTURE.md

# 竞品分析
cat docs/COMPETITIVE_ANALYSIS.md

# PoC 代码（可运行）
cat docs/POC_CODE.md
```

### 复制 PoC 跑起来

```bash
mkdir magent
cd magent
# 复制 docs/POC_CODE.md 里的代码到对应文件
npm install
npm run dev -- run "重构这个文件"
```

## 📋 实施路线图

**MVP（4 周）**：
- Week 1: CLI 入口 + 配置系统
- Week 2: Codex 集成 + 路由
- Week 3: Claude Code 集成
- Week 4: Web UI + npm publish

**v1.0（3 个月）**：
- mem0 完整集成
- 4 个 provider
- MCP 共享中心
- tree session
- 完整 Web UI

**v2.0（6 个月）**：
- Team features
- Kanban 多 agent 编排
- Scheduled agents
- Connect 平台（Telegram、Slack）
- VS Code / JetBrains 插件

## 🤝 借鉴策略

| 来源 | 借鉴 | 改进 |
|------|------|------|
| cloudcli | 7-文件 Provider 抽象、SQLite | + CLI + 路由 + 记忆 |
| ECC | 278 skills、SKILL.md 格式 | - 减重，只取核心 |
| superpowers | HARD-GATE、debugging 流程 | - 减方法论 |
| mem0 | 完整记忆 SDK | v1.0 集成 |
| Pi | 极简哲学、tree history、扩展 | - 不全照搬 |
| Cline | Plan/Act、Scheduled、Connect | - 跨 provider |
| OpenCode | 75+ provider、跨平台 | - 只做核心 |

## 📜 License

MIT
