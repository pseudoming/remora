# Long-Cycle Code Agent Cognitive Architecture — Concept

> [中文版本](./concept_CN.md)

## 0. TL;DR

We are not trying to create an agent that "never forgets." We are building a collaboration system where **forgetting is controllable, and errors are reversible**.

By applying mature software engineering ideas — high cohesion and low coupling, layered storage, defensive programming — to LLM memory management, this architecture transforms the AI coding assistant from a "short-term memory powerhouse with chaotic thinking" into a **senior partner with structured long-term memory, traceable decision processes, and the resilience for high-intensity, long-cycle collaboration**.

**The core difference is not technological flashiness, but a clear-eyed understanding of costs:**  
Tokens are cheap, flow state is expensive. Computation can be wasted; trust cannot be broken.

---

## 1. The Problem: Context Rot

During extended collaboration with an AI coding agent — say, a project spanning multiple days and hundreds of conversation turns — a phenomenon called **Context Rot** inevitably sets in:

- Early key decisions, constraints, and rejected approaches are gradually diluted by subsequent logs, trial-and-error, and tangential chatter.
- The agent's attention drifts in the bloated context. It forgets previously confirmed rules, repeats mistakes, and even overturns settled architecture.
- The user is forced to repeatedly correct, re-explain, and restart sessions. Flow state continuously fractures. Project maintenance costs balloon.

This is not a problem of individual model capability. It is the inevitable result of **context as a cognitive resource trending toward entropy without active management**. Existing solutions either brutally truncate history (losing information) or indiscriminately summarize it (destroying structure). Neither fundamentally addresses the problem of cognitive decay.

---

## 2. Core Philosophy: Trade Compute for Cognitive Safety

**In long-cycle, high-value agent collaboration, tokens are cheap computational resources. The developer's flow state continuity and the traceability of project state are the truly expensive assets.**

Therefore, we proactively pay a "cognitive housekeeping tax" — the system periodically and structurally spends extra tokens to organize, compress, and index conversation history, in exchange for clarity, accuracy, and correctability of memory over long collaborations.

The ROI calculation is straightforward: the cost of spending a few hundred thousand extra tokens on high-quality memory organization is dwarfed by the cost of one project rework, production incident, or developer flow-state reset caused by context rot. In serious code maintenance scenarios, this trade is overwhelmingly worthwhile.

---

## 3. Core Design Principle: High Cohesion, Low Coupling in Memory Management

We apply traditional software engineering modularity principles to the design of an agent's long-term memory:

- **High cohesion**: Each topic contains a complete and self-consistent decision lineage — what approach was adopted, what was overturned and why, what constraints remain, and what open questions persist.
- **Low coupling**: Topics connect through explicit dependency relationships, rather than letting information implicitly scatter across hundreds of messages.

This transforms conversation history from a chaotic "message web" into a structured **memory map**. Each topic becomes an independently understandable, individually updatable, source-traceable memory module.

---

## 4. How the System Works: Three Core Mechanisms

### 4.1 Layered Memory: Hot, Warm, Cold

- **Hot Memory**: Content placed directly into the LLM's inference window. Static global instructions + structured summary index of all active topics (detailed summaries loaded on demand) + recent uncompressed conversations since the last compaction.
- **Warm Memory**: Complete raw conversation records, indexed by topic and time, stored in a local SQLite database. It does not enter context but can be recalled at any time. **Compression is reversible — summaries may have bias, but the original facts are never lost.**
- **Cold Memory**: General rules, lessons learned, and patterns extracted from past projects, accumulated as long-term reusable knowledge.

### 4.2 Topic-Based Compression: Post-Priority Principle & Verification

When uncompressed messages in hot memory accumulate past a threshold, the system compresses them by topic, converting raw conversations into structured summaries.

At the heart of this mechanism are **structured decisions** — key judgments distilled into non-drifting constraints:

```json
{
  "decision": "Do not introduce Redis",
  "status": "confirmed",
  "rationale": "Project is small, local in-memory cache is sufficient. Rejected because: Redis operational complexity outweighs any performance benefit",
  "negated": "Any external caching layer, including scenarios claiming 'just for this one feature'",
  "user_confirmed": true,
  "evidence_msg_ids": [52]
}
```

Key design elements:
- **Mandatory `rationale`**: Records not just "what was done" but also "why" — and explicitly "what was decided NOT to do."
- **User hard anchors**: Decisions with `user_confirmed: true` enter a non-bypassable compression verification checklist and cannot be silently dropped.

The compression process itself follows four principles:

1. **Boundaries are user-controlled.** Topic segmentation is managed through explicit commands — this is a design choice, not a compromise. The system may suggest a new topic when it detects topic drift, but the decision always remains with the user.

2. **Post-priority principle.** During compression, the LLM is explicitly instructed that "later decisions override earlier ones." Overturned approaches retain only their final state and the reason for being overturned — they do not pollute the summary's main conclusion.

3. **Decision checklist verification.** Compression reliability does not depend on LLM self-assessment. Before compression, the system extracts a structured decision checklist. After compression, it verifies that all entries are preserved, calculating an objective, verifiable confidence score. If the pass rate falls below threshold, the summary is flagged as "low confidence" and the user is prompted to review. This mechanism provides a deterministic backstop for the LLM's probabilistic output.

4. **Background execution.** Compression runs in a background thread without blocking user interaction.

### 4.3 Stateless Execution: Fire-and-Forget Sub-Agents

When concrete code operations are needed, the main agent dispatches tasks to a **temporary, environment-isolated sub-agent**:

- **Isolated sandbox**: The sub-agent runs in an independent Git Worktree, completely untouched by the user's in-progress edits.
- **Fire and forget**: The sub-agent is destroyed immediately upon completion, leaving no residual state. This fundamentally prevents execution noise from gradually polluting the main cognitive context.
- **User confirmation**: After task completion, the change diff is presented for the user to decide whether to apply.
- **Complete traceability**: The sub-agent reports all modified and referenced files, ensuring implicit code dependencies are not missed.

---

## 5. Defensive Design: Acknowledging Imperfection, Ensuring Correctability

The core philosophy is **defensive design**: we accept that LLM summaries and memories will contain errors, so we build multiple correction mechanisms rather than pursuing the unrealistic goal of "perfect memory."

- **Warm storage as source of truth**: Any compressed information can be recalled as a complete conversation through keyword search.
- **Active recall**: When the system detects user input that challenges historical decisions, it prompts the LLM in the System Prompt to prioritize using the recall tool to verify facts rather than relying on summaries.
- **Manual recall by user**: An explicit `/recall` command is provided, allowing the user to proactively browse any historical segment at any time.

This makes the system analogous to a senior engineer: it takes notes, it summarizes, but it also knows notes can be wrong — so it always keeps the original records and knows to consult them when needed.
