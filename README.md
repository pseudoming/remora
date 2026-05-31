# Remora

> **Remora** — An attachable long-term memory layer for coding agents. Cleans up context rot, locks down key decisions, and keeps your AI partner remembering across sessions.
>
> [中文版本](./README_CN.md)

[![Status](https://img.shields.io/badge/status-concept%20%2F%20RFC-blue)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen)]()

---

## Have You Experienced This?

Day four of a project. Your AI agent starts undoing the architecture you settled on day one.

Two weeks ago you spent an hour explaining why Redis is overkill. Today it suggests Redis again.

You keep correcting the same mistake. Next session, it happens again.

This is not your model getting dumber. This is **Context Rot** — the inevitable cognitive decay that sets in during long-running AI collaborations.

---

## The Problem Runs Deeper Than You Think

Context Rot is actually two distinct problems:

| Problem | Nature | Existing Tools |
|---------|--------|---------------|
| **Information Entropy** | Conversations get long, attention dilutes | Truncation, rolling summaries |
| **Decision Drift** | LLM silently reinterprets settled decisions | ❌ Almost no tool addresses this |

Claude Code has `CLAUDE.md`. Cursor has Rules. But both are **static text**. No tool tackles this at an engineering level:

> *You explicitly discussed, rejected, and documented why you rejected a solution three months ago. Why should the AI be allowed to propose it again six months later?*

**This repo exists to solve the second problem — preventing cognitive drift through structured decision anchoring.**

---

## Core Design

### Decision Anchoring, Not Just Compression

We don't just "summarize history." We distill key judgments into **non-drifting structured constraints**:

```json
{
  "decision": "Do not introduce Redis",
  "status": "confirmed",
  "rationale": "Project is small, local in-memory cache is sufficient. Rejected because: the operational complexity Redis introduces far outweighs any performance gain",
  "user_confirmed": true,
  "evidence_msg_ids": [52, 67]
}
```

`rationale` captures both *why* and *why not*. Decisions with `user_confirmed: true` enter a **hard anchor checklist** that compression cannot silently bypass — something no existing tool provides.

### Hot / Warm / Cold Three-Tier Memory

```
Hot Memory (in LLM inference window)
  ├── Static System Prompt (cacheable)
  ├── Structured topic summary index
  └── Uncompressed messages since last compaction

Warm Memory (local database, recallable on demand)
  └── Complete raw conversations + FTS5 full-text index

Cold Memory (cross-project experience library, Phase 3)
  └── Lessons learned + reusable rules
```

Compression is lossy, but the original records are never lost. Errors are reversible.

### Topic-Based High Cohesion, Low Coupling

Conversation history is not a chaotic web of messages — it's a **structured memory map**. Each topic contains a complete decision lineage, connected through explicit dependencies rather than scattered across hundreds of messages.

### Stateless Sub-Agent Execution

Code modifications are delegated to **fire-and-forget** isolated sub-agents (Git Worktree). The main cognitive context stays clean — execution noise never pollutes it. Diffs are presented for user confirmation before application.

---

## What Existing Tools Do and Don't Do

| Capability | Claude Code | Cursor | Remora |
|------------|:-----------:|:------:|:------:|
| Cross-session memory | CLAUDE.md (static) | Rules (static) | ✅ Dynamic structured |
| Negative decision recording | ❌ | ❌ | ✅ |
| Decision drift protection | ❌ | ❌ | ✅ |
| Raw conversation recall | ❌ | ❌ | ✅ |
| Code execution isolation | ✅ | Partial | ✅ |
| Compression confidence verification | ❌ | ❌ | ✅ |

---

## What Remora Explicitly Does NOT Do

- **Not a general-purpose memory system.** Remora is built specifically for decision management in long-cycle coding agents. It does not serve chatbots, personal knowledge bases, or general Q&A scenarios.
- **No built-in model.** We don't train memory models or embed any LLM. All reasoning goes through cloud APIs (DeepSeek, Kimi, OpenAI, etc.).
- **Not a code generator.** This is a memory layer, not a coding agent itself. It attaches to existing agents (Claude Code, Cursor, etc.) and manages decision history and context structure — it does not directly generate or modify code.
- **No automatic topic segmentation.** Manual topic tagging is an intentional design choice, not a temporary compromise. Letting users explicitly manage topic boundaries is far more reliable than having an LLM guess.

---

## Documentation

- [Concept & Design Philosophy](docs/concept.md) — Why this design, what are the core judgments
- [Engineering Architecture v6](docs/architecture.md) — System architecture, module design, phased plan
- [Design Review Discussions](docs/reviews/) — Critical analysis and revision records

---

## Roadmap

```
Phase 0 (current)   Concept release, accepting discussion and feedback
Phase 1 (planned)   Minimum closed loop: message storage → topic management → manual compression → recall
Phase 2             Structured topics, hard anchor verification, file association
Phase 3             Auto-compression, experience library, TUI
```

See [ROADMAP.md](ROADMAP.md) for details.

---

## Current Stage

**This is currently in concept / RFC stage.** There is no runnable code.

The purpose of this repo is to:
1. Open-source the design for critique
2. Find others who are also frustrated by Context Rot
3. Build it together if the direction is validated

If you're working on long-cycle AI agent collaboration or have thoughts on LLM memory management, open an Issue.

---

## Get Involved

- Have ideas? → [Start a Discussion](../../discussions)
- Found a design flaw? → [Open an Issue](../../issues)
- Want to build it? → See [CONTRIBUTING.md](CONTRIBUTING.md)

---

## License

MIT
