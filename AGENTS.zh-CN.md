# AGENTS.md（中文版）

> 给所有 AI 编码助手（Claude Code / OpenAI Codex / Cursor / Copilot / Gemini 等）的工作约定。
> 本文件遵循 [agents.md](https://agents.md/) 开放标准（Linux Foundation / Agentic AI Foundation 治理）。
> 🌐 **语言**：[English](./AGENTS.md) · **简体中文**
> Claude Code 用户：本文件与 `CLAUDE.md` 同时存在时，以更靠近被修改文件的为准；显式 chat 指令永远胜出。
> ⚠️ **注意**：AI 工具默认读取的是 [`AGENTS.md`](./AGENTS.md)（英文版）。本文件为中文协作者参考用；两份文件内容必须同步。

---

## 0. 最高优先级规则（NON-NEGOTIABLE）

🚫 **绝不**提交密钥、token、`.env`、凭据到任何文件或日志。
🚫 **绝不**未经确认执行不可逆操作（`rm -rf`、强推、删分支、删表、发邮件、关 PR）。
🚫 **绝不**在第三方 LLM API 不稳定时静默重试超过 3 次——按 §3 协议交接。
⚠️ **必须先问**：新增生产依赖、改 CI/CD、改数据库 schema、调用付费 API。
✅ **总是**：用结构化输出 / 用幂等键 / 用绝对路径 / 在动手前复述目标。

---

## 1. 项目概览

- **目的**：研究并落地"依赖不稳定第三方 LLM API 时的工程化对策"
- **核心问题域**：上下文耗尽前交接、模型路由、降级与熔断
- **技术栈**：[待补充：Python / TypeScript / Go ？]
- **关键目录**：
  - `/docs/` — 策略文档与决策记录
  - `/src/` — 实现代码（如有）
  - `/tests/` — 验证脚本
  - `.claude/handoffs/` — **交接快照存放处**（见 §3）

---

## 2. 命令速查（命令优先，禁止只写工具名）

```bash
# 环境
[待补充：例如 uv sync / pnpm install --frozen-lockfile]

# 验证（在声明完成前必须全部通过）
[待补充：例如 pnpm -w typecheck && pnpm -w lint && pnpm -w test]

# 单测一个文件
[待补充：例如 pytest tests/test_handoff.py -v]

# 启动本地服务
[待补充]
```

---

## 3. 上下文预算与交接协议 ⭐ 核心

### 3.1 触发器（任一命中即执行）

| 信号 | 阈值 | 动作 |
|------|------|------|
| Token 使用率 | ≥ 70% | 进入"准备模式"：避免新增长引用，开始精炼输出 |
| Token 使用率 | ≥ 85% | **立即**生成 §3.2 交接快照，写入 `.claude/handoffs/` |
| Token 使用率 | ≥ 92% | **强制**停止主任务，仅完成交接快照后退出 |
| API 调用 | 连续 3 次超时或 5xx | 写入快照，停止重试，向用户报告 |
| 工具循环 | 同工具+同参数 ≥ 3 次 | 视为卡死，写入快照并请求人工介入 |
| 用户呼叫 | 出现 "save state" / "context is getting full" / "交接" / "停一下" | 立即执行 §3.2 |

> ⚠️ 大模型对自身 token 剩余感知很差。**宁可早交接，不要赌**。

### 3.2 交接快照结构（HANDOFF Schema v1）

文件名：`.claude/handoffs/YYYY-MM-DD-HHMMSS-<slug>.md`

```markdown
---
schema_version: 1.0
created_at: <ISO8601>
origin_model: <model-id>
tokens_consumed: <int>
continues_from: <前一份 handoff 文件名，无则留空>
confidence: <0.0-1.0>
---

## Objective（必填，无损）
- primary_goal: <一句话，用户的最终目标>
- success_criteria: <可验证的完成判定>
- hard_constraints: <不能违反的约束>

## Progress
- completed: <已完成步骤 + artifact 引用，不复述全文>
- current: <当前步骤 + 状态 + 阻塞原因（如有）>
- pending: <按优先级排序的待办>
- ruled_out: <已验证不可行的方案 + 原因，避免重走弯路>

## Working Context
- key_facts: <影响后续决策的事实>
- user_preferences: <用户明示/暗示的偏好>
- external_refs: <文件路径、URL、msg_id——用引用代替全文>

## Next Action（必填，无损）
- immediate: <接手方第一步要做什么>
- expected_input: <需要哪些输入>
- fallback: <第一步失败时的退路>

## Gotchas
- <已知的坑、临时 workaround、不要再踩的雷>
```

### 3.3 接手方启动语（写在新会话首条）

```
你正在接手被中断的任务，请：
1. 读取 .claude/handoffs/ 下最新文件
2. 用一句话复述 Objective.primary_goal，等我确认
3. 检查 ruled_out 避免重复尝试
4. 从 Next Action.immediate 开始执行
5. 若发现快照矛盾或缺失，标 [NEED_CLARIFY] 后回退给我
```

---

## 4. 代码风格

> 原则：show, don't tell（GitHub 2500 仓库分析结论）。

[待补充：贴一段你的代码作为正反例]

```python
# ✅ Good
def fetch_user(user_id: str) -> User | None:
    ...

# ❌ Bad
def fetch_user(id):
    ...
```

通用约束：
- 命名：[待补充：snake_case / camelCase]
- 错误处理：只在系统边界校验；内部代码信任类型
- 注释：默认不写；只写"为什么"，不写"做什么"
- 测试：边界用真实依赖，单元用 fake；**禁用 mock 数据库**

---

## 5. 三级边界（Always / Ask first / Never）

### ✅ Always do
- 在每次写代码前先看相邻文件的命名/结构约定
- 用绝对路径而非相对路径（跨工具更稳定）
- 完成后跑完 §2 验证命令
- 修复 bug 时附最小复现

### ⚠️ Ask first
- 新增生产依赖（即便看起来很小）
- 改公共 API / 数据库 schema / 配置文件
- 删除超过 50 行的代码
- 跨多个目录的批量改动
- 任何可能影响计费的 LLM 调用

### 🚫 Never do
- 提交 `.env`、`*.pem`、`credentials.*`、`secrets.*`
- 使用 `--no-verify`、`--force` 等绕过校验的标志
- 在测试里 mock 数据库（用真实测试库）
- 静默吞掉异常（`except: pass`）
- 编造文件路径、API 名称、依赖版本号——不确定就 grep

---

## 6. 与第三方 LLM API 协作（项目专属）

针对"不稳定 + 慢"的供应商：

- **超时**：连接 5s / TTFT 软 15s 硬 30s / 总响应 60s（简单）或 180s（复杂）
- **重试**：指数退避 + 抖动；max 3 次；幂等键必填
- **熔断**：60s 窗口内失败率 ≥50% 触发，OPEN 30s
- **路由优先级**：[待补充：主→备1→备2 的具体模型]
- **降级时务必**：保留请求 ID、记录切换时间戳、按 §3 写快照

---

## 7. PR / Commit 规范

```
<type>(<scope>): <summary>

<body：问题 / 方案 / 测试 / 风险 / 回滚>
```

- `type`: feat | fix | refactor | docs | test | chore
- 提交前必跑：§2 验证命令
- PR 标题 ≤ 70 字符；正文用 §3.2 同款结构化字段更佳

---

## 8. 元规则（关于本文件本身）

- 本文件 ≤ 150 行；超出时拆到子目录 AGENTS.md
- 每季度回顾一次，删除"已经成为肌肉记忆"的条目
- 修改本文件需明确标注 commit 类型为 `docs(agents)`
- 与 `CLAUDE.md` 冲突时：本文件管"工程约定"，CLAUDE.md 管"Claude 专属指令"
- 中英双版必须同步：修改其中一份时，必须在同一 commit 内镜像修改另一份

---

<!--
References (do not delete):
- AGENTS.md spec: https://agents.md/
- OpenAI Codex AGENTS.md guide: https://developers.openai.com/codex/guides/agents-md
- Claude Code best practices: https://code.claude.com/docs/en/best-practices
- GitHub 2500-repo analysis: https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/
- Session handoff skill: https://github.com/softaworks/agent-toolkit/tree/main/skills/session-handoff
-->
