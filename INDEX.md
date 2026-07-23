# 文档索引

## 📚 必读

1. [README.md](README.md) - 项目入口
2. [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) - **1482 行完整架构**
3. [docs/COMPETITIVE_ANALYSIS.md](docs/COMPETITIVE_ANALYSIS.md) - **756 行竞品分析**
4. [docs/POC_CODE.md](docs/POC_CODE.md) - **可运行 MVP 代码**
5. [CONTRIBUTING.md](CONTRIBUTING.md) - 贡献指南

## 📁 项目结构

```
magent/
├── README.md                       # 项目入口
├── LICENSE                         # MIT License
├── CONTRIBUTING.md                 # 贡献指南
├── INDEX.md                        # 本文件
├── .gitignore
├── docs/                           # 产品文档
│   ├── ARCHITECTURE.md             # 完整产品架构
│   ├── COMPETITIVE_ANALYSIS.md     # 8 个项目竞品分析
│   └── POC_CODE.md                 # MVP 代码骨架
└── (源码将在 MVP 阶段加入 src/)
```

## 🎯 核心决策

1. **产品名**：`magent`（meta-agent / multi-agent）
2. **核心差异化**：跨工具的智能编程助手
3. **记忆后端**：MVP 用本地 JSONL，v1.0 集成 mem0
4. **路由策略**：规则 + LLM fallback
5. **借鉴策略**：fork cloudcli + 集成 ECC skills + 集成 mem0

## 📋 下一步

- [ ] 在 GitHub 创建 Issue 跟踪 MVP 任务
- [ ] 复制 PoC 代码搭第一个版本
- [ ] 跑通 Codex 集成
- [ ] 加智能路由
- [ ] 加本地记忆
- [ ] 写第一个 release
- [ ] npm publish alpha
