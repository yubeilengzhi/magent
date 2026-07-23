# Contributing to magent

欢迎贡献！以下是流程。

## 快速开始

1. Fork 仓库
2. 创建特性分支：`git checkout -b feature/amazing-feature`
3. 提交变更：`git commit -m "Add amazing feature"`
4. 推送分支：`git push origin feature/amazing-feature`
5. 创建 Pull Request

## 开发

```bash
# 克隆
git clone https://github.com/yubeilengzhi/magent.git
cd magent

# 安装依赖
npm install

# 开发模式
npm run dev

# 测试
npm test

# 构建
npm run build
```

## 提交规范

使用 conventional commits：
- `feat:` 新功能
- `fix:` 修复
- `docs:` 文档
- `refactor:` 重构
- `test:` 测试
- `chore:` 杂项

例如：
```
feat: add mem0 integration
fix: routing decision timeout
docs: update architecture diagram
```

## 项目结构

```
src/
├── cli/          # CLI 入口
├── core/         # 核心（路由、记忆、配置）
├── providers/    # Provider 适配器
└── web/          # Web UI（可选）
```

## 添加新 Provider

1. 在 `src/providers/` 创建 `{name}.ts`
2. 实现 `ProviderAdapter` 接口
3. 添加配置到 `~/.magent/config.yaml`
4. 写测试
5. 提 PR

## 添加新 Skill

1. 创建 `skills/{name}/SKILL.md`
2. 遵循 [ECC](https://github.com/affaan-m/ecc) 的 SKILL.md 格式
3. 提交

## 行为准则

- 友好、包容、专业
- 接受建设性批评
- 关注用户价值，不炫技
- 中文友好（项目作者是中文母语者）

## License

贡献默认采用 MIT License。
