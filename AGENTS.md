# AGENTS.md

> Working contract for all AI coding assistants (Claude Code / OpenAI Codex / Cursor / Copilot / Gemini, etc.).
> Follows the [agents.md](https://agents.md/) open standard (governed by the Agentic AI Foundation, Linux Foundation).
> 🌐 **Languages**: **English** · [简体中文](./AGENTS.zh-CN.md)
> Claude Code users: when both this file and `CLAUDE.md` exist, the one nearest the edited file wins; explicit chat instructions always override.

---

## 0. Non-Negotiable Rules (Highest Priority)

🚫 **NEVER** commit secrets, tokens, `.env`, or credentials to any file or log.
🚫 **NEVER** execute irreversible operations without explicit confirmation (`rm -rf`, force-push, branch deletion, table drops, sending email, closing PRs).
🚫 **NEVER** silently retry an unstable third-party LLM API more than 3 times — invoke the handoff protocol in §3.
⚠️ **ASK FIRST**: adding production dependencies, changing CI/CD, modifying database schema, calling paid APIs.
✅ **ALWAYS**: use structured output / use idempotency keys / use absolute paths / restate the goal before acting.

---

## 1. Project Overview

- **Purpose**: research and implement engineering countermeasures for "depending on unstable third-party LLM APIs"
- **Core problem domain**: context-window handoff, model routing, degradation, and circuit-breaking
- **Tech stack**: [TBD: Python / TypeScript / Go ?]
- **Key directories**:
  - `/docs/` — strategy documents and decision records
  - `/src/` — implementation code (if any)
  - `/tests/` — verification scripts
  - `.claude/handoffs/` — **handoff snapshot storage** (see §3)

---

## 2. Command Cheatsheet (commands first — tool names alone are not enough)

```bash
# Environment
[TBD: e.g., uv sync / pnpm install --frozen-lockfile]

# Validation (must all pass before declaring done)
[TBD: e.g., pnpm -w typecheck && pnpm -w lint && pnpm -w test]

# Run a single test file
[TBD: e.g., pytest tests/test_handoff.py -v]

# Start local server
[TBD]
```

---

## 3. Context Budget & Handoff Protocol ⭐ Core

### 3.1 Triggers (any one fires the protocol)

| Signal | Threshold | Action |
|--------|-----------|--------|
| Token usage | ≥ 70% | Enter "prep mode": avoid new long quotes, start condensing output |
| Token usage | ≥ 85% | **Immediately** generate the §3.2 handoff snapshot to `.claude/handoffs/` |
| Token usage | ≥ 92% | **Hard stop** main task; finish only the handoff snapshot, then exit |
| API calls | 3 consecutive timeouts or 5xx | Write snapshot, stop retrying, report to user |
| Tool loop | Same tool + same args ≥ 3 times | Treat as stuck; write snapshot and request human intervention |
| User cue | "save state" / "context is getting full" / "handoff" / "pause" | Execute §3.2 immediately |

> ⚠️ LLMs have poor self-awareness of remaining tokens. **When in doubt, hand off early.**

### 3.2 Handoff Snapshot Structure (HANDOFF Schema v1)

Filename: `.claude/handoffs/YYYY-MM-DD-HHMMSS-<slug>.md`

```markdown
---
schema_version: 1.0
created_at: <ISO8601>
origin_model: <model-id>
tokens_consumed: <int>
continues_from: <previous handoff filename, empty if none>
confidence: <0.0-1.0>
---

## Objective (required, lossless)
- primary_goal: <one sentence — the user's end goal>
- success_criteria: <verifiable definition of done>
- hard_constraints: <inviolable constraints>

## Progress
- completed: <finished steps + artifact refs; do not restate full content>
- current: <current step + status + blocker (if any)>
- pending: <to-do list, ordered by priority>
- ruled_out: <approaches verified as non-viable + reason — prevents re-walking dead ends>

## Working Context
- key_facts: <facts that affect downstream decisions>
- user_preferences: <preferences the user stated or implied>
- external_refs: <file paths, URLs, msg_ids — reference, don't inline>

## Next Action (required, lossless)
- immediate: <what the receiver should do first>
- expected_input: <inputs required>
- fallback: <retreat path if the first step fails>

## Gotchas
- <known pitfalls, temporary workarounds, mines not to step on again>
```

### 3.3 Receiver Onboarding Prompt (first message in the new session)

```
You are taking over an interrupted task. Please:
1. Read the latest file in .claude/handoffs/
2. Restate Objective.primary_goal in one sentence and wait for my confirmation
3. Review ruled_out to avoid repeating dead ends
4. Start from Next Action.immediate
5. If the snapshot has contradictions or gaps, tag [NEED_CLARIFY] and bounce back to me
```

---

## 4. Code Style

> Principle: show, don't tell (GitHub's 2500-repo analysis finding).

[TBD: paste a real snippet from your codebase as a paired good/bad example]

```python
# ✅ Good
def fetch_user(user_id: str) -> User | None:
    ...

# ❌ Bad
def fetch_user(id):
    ...
```

General constraints:
- Naming: [TBD: snake_case / camelCase]
- Error handling: validate at system boundaries; trust types in interior code
- Comments: none by default; write only "why", not "what"
- Tests: use real dependencies at boundaries, fakes for units; **no DB mocking**

---

## 5. Three-Tier Boundaries (Always / Ask first / Never)

### ✅ Always do
- Read adjacent files' naming/structure conventions before writing code
- Use absolute paths over relative ones (more robust across tools)
- Run §2 validation commands after finishing
- Attach a minimal reproduction when fixing bugs

### ⚠️ Ask first
- Adding production dependencies (even small ones)
- Changing public APIs / database schema / config files
- Deleting more than 50 lines of code
- Batch changes spanning multiple directories
- Any LLM call that affects billing

### 🚫 Never do
- Commit `.env`, `*.pem`, `credentials.*`, `secrets.*`
- Use bypass flags like `--no-verify`, `--force`
- Mock the database in tests (use a real test database)
- Silently swallow exceptions (`except: pass`)
- Fabricate file paths, API names, or dependency versions — grep when unsure

---

## 6. Working with Third-Party LLM APIs (project-specific)

For "unstable + slow" providers:

- **Timeouts**: connect 5s / TTFT soft 15s, hard 30s / total 60s (simple) or 180s (complex)
- **Retries**: exponential backoff + jitter; max 3; idempotency key required
- **Circuit breaker**: trip when failure rate ≥ 50% in a 60s window; OPEN for 30s
- **Routing priority**: [TBD: primary → fallback#1 → fallback#2 with specific models]
- **On every degradation**: preserve request ID, log switch timestamp, write a §3 snapshot
- **Reasoning effort**: prefer a higher effort tier when the upstream is unstable — every retry is expensive, so trade single-call latency for fewer reruns. Reserve `low` / `minimal` for deterministic, near-zero-retry-cost work (formatting, schema fills, deterministic transforms). For Claude Code, the keywords `think` / `think harder` / `ultrathink` map to roughly 4k / 10k / 32k thinking-token budgets — they are real budget tiers, not hints.

---

## 7. PR / Commit Convention

```
<type>(<scope>): <summary>

<body: problem / approach / tests / risk / rollback>
```

- `type`: feat | fix | refactor | docs | test | chore
- Before committing: run §2 validation commands
- PR title ≤ 70 chars; the body benefits from §3.2 schema-style structured fields

---

## 8. Meta Rules (about this file itself)

- This file ≤ 150 lines; if it grows, split into subdirectory `AGENTS.md` files
- Review quarterly; delete items that have become muscle memory
- Edits to this file must use commit type `docs(agents)`
- When conflicting with `CLAUDE.md`: this file owns "engineering conventions"; `CLAUDE.md` owns "Claude-specific instructions"
- Keep this file and `AGENTS.zh-CN.md` in sync; any change to one must be mirrored in the other within the same commit

---

<!--
References (do not delete):
- AGENTS.md spec: https://agents.md/
- OpenAI Codex AGENTS.md guide: https://developers.openai.com/codex/guides/agents-md
- Claude Code best practices: https://code.claude.com/docs/en/best-practices
- GitHub 2500-repo analysis: https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/
- Session handoff skill: https://github.com/softaworks/agent-toolkit/tree/main/skills/session-handoff
-->
