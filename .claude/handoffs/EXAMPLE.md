# EXAMPLE: Handoff Snapshot

> 本文件是 `AGENTS.md` §3.2 定义的 HANDOFF Schema v1 的**示范样例**。
> 不要在真实任务中复用本文件——它仅作为新手理解字段如何填写的参考。
> 真实 handoff 写入路径：`.claude/handoffs/YYYY-MM-DD-HHMMSS-<slug>.md`
>
> This file is a **reference example** of the HANDOFF Schema v1 defined in
> `AGENTS.md` §3.2. Do not reuse this snapshot for real tasks — it exists
> only so newcomers can see how each field should be filled in.

---

```yaml
---
schema_version: 1.0
created_at: 2026-05-28T14:32:17+08:00
origin_model: claude-opus-4.7
tokens_consumed: 152384
continues_from:
confidence: 0.82
---
```

## Objective

- **primary_goal**: 实现 `handoff_writer` 模块，使其能在触发条件命中时按 §3.2 schema 落盘快照。
- **success_criteria**:
  - 三类触发器（token_ratio ≥ 0.85 / 连续 3 次 API 失败 / 同工具同参数循环 3 次）都能正确激活
  - 输出的 markdown 能被 `handoff_loader.parse()` 无损解析回 Pydantic 对象
  - 写入是原子的：进程崩溃不会留下半截文件
- **hard_constraints**:
  - 仅依赖 Python stdlib + Pydantic v2，禁用第三方 LLM SDK
  - 不能 mock 数据库（项目硬约束，见 `AGENTS.md` §5）

## Progress

- **completed**:
  - `src/handoff/models.py` — Pydantic v2 模型，对应 schema v1 全部字段
  - `src/handoff/triggers.py` — 三个触发器函数 + 单元测试
  - `tests/handoff/test_triggers.py` — 12 个用例，覆盖率 87%
  - 决策：用 markdown 而非 JSON 存储（理由见 ruled_out）
- **current**: 实现 `write_snapshot()` 中的原子写入；卡在跨平台 `fsync` 行为差异
  - macOS 的 `os.fsync()` 不真正刷盘，需要 `fcntl(F_FULLFSYNC)`
  - Linux 直接用 `os.fsync()` 即可
- **pending**（按优先级）:
  1. 完成 `atomic_write(path, content)` 工具函数
  2. 实现 `load_latest_handoff()` 加载器
  3. 与 `model_router` 集成：路由所有 fallback 失败时主动触发 handoff
  4. 写一份 README 章节描述 trigger 行为
- **ruled_out**:
  - **SQLite 存 snapshot** → 查询不是热点，markdown 文件足够且利于人工审阅与 PR review
  - **JSON 而非 markdown** → 损失人可读性；schema 校验通过 frontmatter + Pydantic 已足够
  - **APScheduler 定期写快照** → 改动太大，且与"按事件触发"的设计哲学冲突

## Working Context

- **key_facts**:
  - Python 3.11+；`from __future__ import annotations` 默认启用
  - Pydantic v2 而非 v1（团队偏好，已在 §5 备案）
  - `.claude/handoffs/` 在 `.gitignore` 内，但 `EXAMPLE.md` 例外
  - CI 用 GitHub Actions，见 `.github/workflows/agents-sync-check.yml`
- **user_preferences**:
  - 错误只在系统边界处理，内部代码信任类型签名
  - 注释只写"为什么"，不写"做什么"
  - 测试用真实依赖；fakes 仅限纯内存对象
- **external_refs**:
  - 设计文档：`docs/handoff-design.md`（commit `a1b2c3d`）
  - 触发阈值定义：`AGENTS.md` §3.1
  - 上游 issue：#42（用户最初的需求描述）

## Next Action

- **immediate**: 在 `src/handoff/io.py` 中实现 `atomic_write(path: Path, content: str) -> None`：
  1. 用 `tempfile.NamedTemporaryFile(dir=path.parent, delete=False)` 创建临时文件
  2. 写入并 flush
  3. 调 `os.fsync(fd)`（macOS 上额外调 `fcntl.F_FULLFSYNC`）
  4. `os.replace(tmp_path, path)` 原子重命名
- **expected_input**: 无；所有依赖已就位
- **fallback**: 若 `os.replace` 在 Windows 共享文件夹上行为异常，回退到 `portalocker` 库（已在 ruled_out 之外，可引入）

## Gotchas

- `.claude/handoffs/` 目录可能不存在 → 写入前 `path.parent.mkdir(parents=True, exist_ok=True)`
- Windows 上 CRLF 转换会污染 frontmatter 解析；已在 `.gitattributes` 锁定 LF：`AGENTS*.md text eol=lf`
- `tokens_consumed` 字段在 Anthropic SDK v0.40+ 才有可靠估计，旧版返回 None — 已在 models.py 中标记 Optional
- Pydantic v2 的 `model_validate` 对 YAML frontmatter 不直接支持，需要先 `yaml.safe_load` 再传入

---

<!--
This example demonstrates the right level of detail:
- Each field is filled with concrete, verifiable content
- "ruled_out" prevents the receiver from re-walking dead ends
- "external_refs" uses pointers (commit hash, file path, issue #) instead of inlining content
- "immediate" is a step the receiver can execute without further clarification

Anti-patterns to avoid in real handoffs:
- Vague goals ("improve the system")
- Dumping full file contents into the snapshot
- Listing every micro-decision instead of the ones that actually constrain future work
- Empty "ruled_out" — almost every non-trivial task has at least one dead end worth recording
-->
