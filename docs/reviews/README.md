# Design Reviews

> [中文版本](./README_CN.md)
>
> Critical analysis and revision records for the Remora architecture.

The architecture has undergone four formal review rounds each by Claude and ChatGPT, with each round addressing progressively deeper engineering concerns.

## Claude Reviews

| Review | Focus | Key Issues Identified |
|--------|-------|----------------------|
| [v1](v1-claude.md) | Initial viability assessment | Three hard problems: compression verification, topic boundary identification, cold storage quality |
| [v2](v2-claude.md) | Engineering refinement | Checksum format, async compression boundaries, topic relevance filtering |
| [v3](v3-claude.md) | Pre-implementation readiness | Git isolation edge cases, `associated_files` cold start, reverse-compression prompt clarity |
| [v4](v4-claude.md) | Engineering completeness | Worktree acceptance protocol, `facts` field definition, `compression_confidence` formula |

## ChatGPT Reviews

| Review | Focus | Key Issues Identified |
|--------|-------|----------------------|
| [v1](v1-chatgpt.md) | Core value assessment | Topic segmentation difficulty, summary drift, complexity cost, need for measurable metrics |
| [v2](v2-chatgpt.md) | MVP readiness | Topic computability, versioned compression, async consistency, semantic-level verification |
| [v3](v3-chatgpt.md) | Phase 1 scope narrowing | State machine contract, sub-agent output schema, topic split/merge rules, CAS version commit |
| [v4](v4-chatgpt.md) | Final pre-implementation check | State machine spec, evidence chain format, FTS5 trigram limitations, worktree recovery protocol |

## Gemini Reviews

| Review | Focus | Key Issues Identified |
|--------|-------|----------------------|
| [v1](v1-gemini.md) | Industrial viability | Topic entanglement, code-memory state desync, STW compression, LLM refusal to recall |
| [v2](v2-gemini.md) | Construction blueprint | Async race conditions, brittle hash anchoring, CLI/async thread conflict, log explosion |
| [v3](v3-gemini.md) | Day 1 coding guide | Git worktree over checkout, streaming log OOM prevention, FTS5 trigram for CJK, prompt cache byte hygiene |
| [v4](v4-gemini.md) | Cleanroom red-team review | Dirty workspace paradox, hot memory time black hole, intent false-positive storm, association explosion, chronological hallucination |

## DeepSeek, Doubao & Qwen Reviews (v5)

| Review | Focus | Key Issues Identified |
|--------|-------|----------------------|
| [DeepSeek v5](v5-deepseek.md) | Final pre-implementation validation | Negative-decision checksum, memory expiration mechanism, topic dependency chain warnings, minimal session summary for Phase 1 |
| [Doubao v5](v5-doubao.md) | Comprehensive architecture validation | 700x ROI analysis, referenced-files dependency gap, multi-topic complexity ceiling, UX warning fatigue mitigation |
| [Qwen v5](v5-qwen.md) | Deep engineering pitfalls & V5 critique | Rationale loss in compression, Worktree environment dependency, intent detection false positives, FTS5 semantic blind spot, topic index explosion & lazy loading |

Each review corresponds to a specific version of the architecture document (v1 through v5). The current architecture is at [v6](../architecture.md), incorporating feedback from all fifteen reviews plus additional iterations.

---

For the current state of the architecture, see:

- [Design Philosophy](../concept.md)
- [Engineering Architecture v6](../architecture.md)
