# AI 使用策略

> 针对"不稳定、低速"的第三方 LLM API 的工程化对策——检查点、上下文交接、模型路由与优雅降级。

🌐 **语言**：[English](./README.md) · **简体中文**

## 本仓库要解决什么

这个仓库聚焦一个具体问题：**当依赖的 LLM 供应商不可靠时，如何构建稳定的 Agent 工作流？**

仓库提出的答案：
1. **尽早识别** —— 用硬阈值量化"上下文耗尽"和"超时"，而不是凭感觉。
2. **干净交接** —— 当一个会话即将结束时，写一份结构化快照，让接手方（人或模型）能无损续接。
3. **绕开失败** —— 显式定义主备模型链，配合熔断机制。

完整工作约定见 **[AGENTS.md](./AGENTS.md)** —— 这是仓库里每个 AI 助手启动时第一份读取的文件。

## 文件结构

| 文件 | 用途 |
|------|------|
| [`AGENTS.md`](./AGENTS.md) | 给 AI Agent 的工作约定（工具默认加载的文件） |
| [`AGENTS.zh-CN.md`](./AGENTS.zh-CN.md) | 上者的中文镜像；必须保持同步 |
| `.claude/handoffs/` | 本地交接快照（已 gitignore，详见 `AGENTS.md` §3） |

## 维护约定

- 英文 `AGENTS.md` 是工具默认加载的真理源。
- 中文版供团队人工阅读；**两份文件必须在同一 commit 内同步修改**。
- 完整元规则见 [`AGENTS.md` §8](./AGENTS.md#8-meta-rules-about-this-file-itself)。

## 参考资源

- [AGENTS.md 开放标准](https://agents.md/)——由 Linux Foundation 旗下 Agentic AI Foundation 治理
- [Claude Code 官方最佳实践](https://code.claude.com/docs/en/best-practices)
- [GitHub 对 2500+ 个 AGENTS.md 的分析](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)

## 许可证

本项目以 [MIT License](./LICENSE) 发布。© 2026 各贡献者。
