# AI Usage Strategy

> Engineering countermeasures for working with unstable, slow third-party LLM APIs — checkpointing, context handoff, model routing, and graceful degradation.

🌐 **Languages**: **English** · [简体中文](./README.zh-CN.md)

## What's in this repo

This repository is a working playbook for one specific problem: **how do you build reliable agent workflows on top of an unreliable LLM provider?**

The answer this repo proposes:
1. **Detect early** — quantify "context exhaustion" and "timeout" with hard thresholds, not vibes.
2. **Hand off cleanly** — when a session is about to die, write a structured snapshot so any successor (human or model) can resume losslessly.
3. **Route around failure** — explicit primary/fallback model chain with circuit-breaking.

The full working contract lives in **[AGENTS.md](./AGENTS.md)** — that's the file every AI coding assistant in this repo reads first.

## Files

| File | Purpose |
|------|---------|
| [`AGENTS.md`](./AGENTS.md) | Working contract for AI agents (the spec they follow) |
| [`AGENTS.zh-CN.md`](./AGENTS.zh-CN.md) | Chinese mirror of the above; must stay in sync |
| `.claude/handoffs/` | Local handoff snapshots (gitignored; see `AGENTS.md` §3) |

## Conventions

- The English `AGENTS.md` is the source of truth that tools load by default.
- The Chinese version is for human readers on the team; **both files must change together in the same commit**.
- See [`AGENTS.md` §8](./AGENTS.md#8-meta-rules-about-this-file-itself) for the full meta-rules.

## References

- [AGENTS.md open standard](https://agents.md/) — governed by the Agentic AI Foundation (Linux Foundation)
- [Claude Code best practices](https://code.claude.com/docs/en/best-practices)
- [GitHub's analysis of 2,500+ AGENTS.md files](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)

## License

Released under the [MIT License](./LICENSE). © 2026 the contributors.
