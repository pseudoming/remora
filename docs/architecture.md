# Long-Cycle Code Agent Cognitive Architecture & Engineering Implementation (v6)

> [中文版本](./architecture_CN.md)
>
> **Status**: Five rounds of architecture iteration, fifteen independent reviews cross-validated, entering Phase 1 implementation
> **Core Philosophy**: Trade compute for cognitive safety — tokens are cheap, the developer's flow state and project rework costs are expensive

---

## 1. Overview

### 1.1 Project Positioning
Build a local CLI/TUI client acting as a Coding Agent, leveraging cloud LLM APIs for reasoning, capable of multi-day, hundreds-of-turn collaboration with developers, maintaining coherent awareness of project context, decision history, and constraints — preventing "context rot" from causing rework or erroneous decisions.

### 1.2 Core Value
- **Structured Memory**: Organize conversation history by topic into a traceable, correctable knowledge base.
- **Stateless Execution**: Delegate code modifications, testing, etc. to isolated, fire-and-forget sub-agents, preventing pollution of the main cognitive context.
- **Layered Storage**: Hot, warm, and cold memory each serve their distinct role.
- **Active Error Correction**: When potential historical memory references are detected, prompt the LLM to trace back to original records.
- **Versioned & Auditable**: Full lifecycle traceability of memory compression, supporting consistency verification and rollback.

### 1.3 Core Principles
- **High Cohesion, Low Coupling**: Each topic contains a complete decision lineage with deep rationale; topics connect through explicit dependencies.
- **Engineering Determinism vs. Model Probabilism**: Use hash verification, async daemons, intent detection, and other traditional software engineering techniques to backstop LLM outputs.
- **Defensive Design**: Acknowledge that compression is inherently lossy, but ensure errors are reversible through warm storage retention and recall mechanisms.
- **Progressive Evolution**: MVP strictly narrows scope — run the minimum closed loop first, then gradually layer on advanced features.

---

## 2. System Architecture

### 2.1 Architecture Overview
```
┌──────────────────── User Interface (CLI / TUI) ───────────────────┐
│                                                                    │
│  ┌──────────────────┐        ┌────────────────────────────┐       │
│  │  Orchestrator    │◄──────►│  Context Manager           │       │
│  │  (Main Loop)     │        │  - Memory lifecycle mgmt   │       │
│  │  - Planner        │        │  - Async bg compression   │       │
│  │  - Dispatcher     │        │    (Phase 3)              │       │
│  │  - State Tracker  │        │  - Compression verification│       │
│  └───┬──────────────┘        │  - File state anchoring   │       │
│      │ dispatch subtasks     └───────────┬────────────────┘      │
│      ▼                                   │                         │
│  ┌──────────────────┐                    │                         │
│  │ Sub-agent Factory│                    │                         │
│  │ (Git Worktree    │                    │                         │
│  │  execution       │                    │                         │
│  │  sandbox)        │                    │                         │
│  └───┬──────────────┘                    │                         │
│      │ logs/results                      │                         │
│      ▼                                   ▼                         │
│  ┌──────────────────────────────────────────────┐                 │
│  │              Memory Bus                      │                 │
│  │  - Warm Storage (SQLite + FTS5 trigram)      │                 │
│  │  - Cold Storage (experience library, Phase 3)│                 │
│  └──────────────────────────────────────────────┘                 │
│                                                                    │
│   ┌─────────── Cloud LLM API ───────────┐                          │
│   │  DeepSeek / Kimi / OpenAI / etc.    │                          │
│   └─────────────────────────────────────┘                          │
└────────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow Summary
1. **User input** → Orchestrator.
2. Orchestrator requests Context Manager to construct current hot memory context (static System Prompt + topic summary index + all uncompressed messages since last compaction watermark).
3. If task execution is required, Task Dispatcher generates task instructions. Context Manager filters associated topics by the files involved in the task, packages a sub-agent context snapshot, and passes it to the Sub-agent Factory.
4. Sub-agent executes in an isolated Git Worktree environment, streaming logs with fuse-based cutoffs, returning structured results (including modified and referenced file lists) upon completion.
5. When main context token count exceeds threshold, Orchestrator notifies Context Manager to trigger background async compression (Phase 3).
6. When user input potentially involves historical memory, intent detection inserts a prompt in the System Prompt, guiding the LLM to actively invoke warm storage retrieval.

---

## 3. Core Module Design

### 3.1 Orchestrator (Main Agent)

**Responsibilities**: Manage conversation flow, understand user intent, plan tasks, dispatch execution, maintain short-term state.

**Components**:
- **Planner**: Calls LLM to decompose tasks and identify associated topics for each step.
- **Task Dispatcher**: Assembles sub-agent context snapshots, and after sub-agent execution, writes back the actually modified and read file paths to the current active topic's `associated_files` and `referenced_files`.
- **State Tracker**: Maintains current active topic, dynamic message buffer since last compaction watermark, task execution status.
- **Intent Detection Bypass**: Before user input enters main logic, match against improved regex patterns. On match, do not hard-intercept — instead insert a hint in the System Prompt: "The user has referenced historical content. If you are uncertain about related facts, it is recommended to invoke the `recall_memory` tool to verify, rather than answering from memory." An explicit `/recall` command is also preserved as the final fallback.

**System Prompt Core Rule**:
> "When the user's input indicates potential challenge or reference to historical memory, you MUST prioritize using the `recall_memory` tool to retrieve original conversation records — do not rely solely on existing summaries in context. If unsure, proactively suggest the user use `/recall`."

**Context Structure**:
```
[System Prompt (globally shared, absolutely static, cacheable)]
[Topic structured summary index (ID + title + associated files)]
[Detailed summaries of topics relevant to current task (loaded on demand)]
[All uncompressed messages since last compaction watermark]
[Current user input]
```

### 3.2 Context Manager (Memory Manager) — Core Module

#### 3.2.1 Core Data Structure: Topic
```json
{
  "topic_id": "t_001",
  "title": "Bilibili Crawler Rate Limiting",
  "status": "open",
  "version": 3,
  "summary": "Final approach: dynamic interval adjustment based on 403 responses...",
  "facts": ["Bilibili API limits single IP to 5 requests per second"],
  "key_decisions": [
    {
      "decision": "Use dynamic interval adjustment",
      "status": "confirmed",
      "rationale": "Fixed intervals still trigger 429 in Bilibili's case; dynamic adjustment adapts to different time periods' rate-limiting policies",
      "evidence_msg_ids": [45, 67]
    },
    {
      "decision": "Do not introduce Redis",
      "status": "confirmed",
      "rationale": "Project is small; introducing Redis adds operational complexity. Local in-memory cache is sufficient",
      "evidence_msg_ids": [52]
    }
  ],
  "constraints": [
    {"constraint": "Single IP requests ≤ 5/sec", "evidence_msg_ids": [45]},
    {"constraint": "Must handle 403 with fallback", "evidence_msg_ids": [67]}
  ],
  "open_questions": [],
  "dependencies": ["t_002"],
  "associated_files": [
    {
      "path": "src/crawler/bilibili.py",
      "action": "modified",
      "hash": "abc123def",
      "last_verified_commit": "3f2a1b0"
    }
  ],
  "referenced_files": [
    {
      "path": "src/crawler/types.py",
      "action": "referenced"
    }
  ],
  "original_message_range": [45, 89],
  "compression_watermark_id": 89,
  "compression_confidence": 0.92,
  "created_at": "...",
  "last_updated": "..."
}
```
- `facts`: Objective, verifiable information (e.g., API documentation limits, environment version numbers).
- `key_decisions`: Choices actively made by the team/agent. Must include `rationale` (deep reasoning). Records not only "what was done" but also "what was explicitly decided NOT to do" and why.
- `constraints`: Active constraints, each with mandatory `evidence_msg_ids`.
- `associated_files`: Records modified files. `referenced_files`: Records read files. Stored separately; modified carries higher weight in topic association matching, referenced carries secondary weight.
- `compression_confidence`: Defined as Checksum pass rate (number of key decisions retained / total key decisions extracted before compression). An objective ratio, not reliant on LLM self-assessment.

#### 3.2.2 Topic Lifecycle Management (Phase 2)
- **Manual tagging as primary**: User/main agent creates topics via `/topic new <name>`. Subsequent messages are automatically attached.
- **Auto-suggestion as auxiliary**: LLM suggests new topic upon detecting topic shift; user confirms.
- **Associated file write-back**: After sub-agent execution, Task Dispatcher writes actually modified files to `associated_files` and read files to `referenced_files`.
- **Topic split reminder**: When a single topic exceeds a message threshold (e.g., 50 messages), actively prompt the user to consider splitting.

#### 3.2.3 Compression Strategy (Phase 2 manual, Phase 3 automatic)
- **Trigger**: Dynamic buffer exceeds threshold (default 70% of model window) or user `/compact`.
- **Post-priority compression**: Prompt explicitly instructs "later decisions override earlier ones," requires recording `rationale` (deep reasoning) for each key decision, and explicitly records "what was decided NOT to do" and why.
- **Decision checklist Checksum**: Extract structured decision points before compression; compare retention after compression, calculate `compression_confidence`. If confidence < 0.7, insert a warning line at the beginning of the main agent's next response.
- **Async execution & watermark** (Phase 3): Compression operates on snapshots; merges new messages after the watermark upon completion. CAS version check on commit — discard and recompute if topic version has changed.
- **Topic dependency chain warning** (Phase 3): When a topic is updated, automatically insert a hint in all topics that depend on it.

#### 3.2.4 Memory-Code Consistency Anchoring
- Record `associated_files` and `referenced_files` during compression or write-back.
- Verify file hash when referencing a topic: silently update if change is small; insert warning if substantial change.

#### 3.2.5 Warm Storage Retrieval & Recall
- **Storage**: SQLite + FTS5 trigram tokenizer.
- **Retrieval API**: `search_messages(query, topic_id=None, limit=10)`. Results are forced into ascending `message_id` order at the code layer.
- **Trigger**: Intent detection hit → System Prompt inserts hint guiding LLM to actively invoke; manual `/recall <keyword>` as fallback.

### 3.3 Sub-agent Factory

**Design Goal**: Stateless, environment-isolated, one-shot execution units.

**Isolation Sandbox**:
- **Core approach: Git Worktree**, creating an independent working directory outside the project.
- **Environment detection & fallback**: Has Git and supports worktree → use worktree; Has Git but doesn't support → stash + checkout + stash pop; No Git → file snapshot backup.
- **Dirty workspace handling**: Before creating worktree, detect uncommitted changes to target files in the main workspace. If present, offer options: ① auto-stash first; ② continue (accept subsequent manual merge); ③ cancel. Default recommendation: ①.
- **Result delivery protocol**: After execution, display diff → user confirms `y/n` → `y` copies files back to main workspace (no merge), `n` completely discards the sandbox.
- **Failure recovery**: Automatically clean up temporary worktree, revert stash (if executed), report failure cause.

**Logging & OOM Prevention**: Streamed reads + fixed-size buffer (2MB) + hard timeout cutoff (3 minutes).

**Context Snapshot Structure**:
```
[System Prompt (globally static, cacheable)]
[Detailed summaries of task-relevant topics (matched via associated_files and referenced_files)]
[Specific task instructions]
[Execution environment configuration]
```

**Return Protocol**:
```json
{
  "status": "success | failure | timeout",
  "summary": "Fixed timeout logic...",
  "changed_files": ["src/crawler/bilibili.py"],
  "referenced_files": ["src/crawler/types.py", "src/config.py"],
  "commands_run": ["python test_crawler.py"],
  "risks": ["Rate limiting may still be imperfect"],
  "raw_log_ref": "msg_456"
}
```
- `changed_files` maps to `associated_files` (modified). `referenced_files` maps to `referenced_files`.

### 3.4 Memory Bus

**Warm Storage**:
- **Database**: SQLite, WAL mode enabled.
- **Message table**: Contains `message_id`, `timestamp`, `role`, `content`, `topic_id`, `metadata`.
- **Full-text index**: FTS5 trigram tokenizer. Retrieval results forced into ascending `message_id` order.
- **Backup**: Auto-backup database file before compression.

**Cold Storage** (Phase 3): Local Markdown/JSON or vector database, manual curation + auto-suggested experience rules.

---

## 4. Key Technical Decisions & Risk Mitigation

| Decision | Approach | Risk Mitigation |
|----------|----------|-----------------|
| **Execution Isolation** | Git Worktree as primary, three-tier fallback, dirty state detection | Complete isolation of user workspace, prevents code loss |
| **Log Handling** | Streamed reads + hard timeout + fixed-size buffer | Prevents subprocess output from blowing up memory |
| **Full-Text Search** | FTS5 trigram + result chronological reordering | Supports Chinese/code symbol substring matching, guarantees causal order |
| **Cache Optimization** | System Prompt absolutely static, dynamic content appended | Ensures maximum cloud prefix cache hit rate |
| **Topic Creation** | Manual tagging as primary, post-hoc file anchoring | Avoids LLM segmentation errors, precise code association |
| **Association Filtering** | modified + referenced dual lists, weighted matching | Does not miss implicit dependencies, controls context noise |
| **Decision Recording** | `key_decisions` requires `rationale` and negative decisions | Preserves deep reasoning, prevents re-proposal of rejected solutions |
| **Compression Strategy** | Post-priority principle, decision checklist Checksum, confidence threshold alerts | Prevents early errors from overwriting later corrections |
| **Async Compression** | Watermark + CAS version check (Phase 3) | Prevents overwriting new messages, ensures concurrency safety |
| **Recall Trigger** | Intent detection hint + LLM autonomous decision + manual fallback | Avoids false positives disrupting flow, ensures defensive coverage |

---

## 5. Tech Stack & Engineering Recommendations

- **Language**: Python 3.11+, `asyncio` async core.
- **LLM API**: `litellm` unified adapter, built-in cost tracking.
- **Terminal Interaction**: Phase 1 uses `prompt_toolkit`; Phase 3 may upgrade to `textual` TUI.
- **Sandbox**: `git worktree` + `asyncio.subprocess`, command whitelist.
- **Storage**: `aiosqlite` async SQLite reads/writes (WAL + trigram FTS5).
- **Tool Standard**: Tool interface recommended to follow MCP protocol.

---

## 6. Phased MVP Implementation Plan

### Phase 1: Basic Stateless Collaboration & Warm Storage (2 weeks)
**Goal**: Run the main-agent/sub-agent minimum closed loop, verify execution isolation and log storage.

**Explicitly out of scope**: Topic-based organization, compression, background tasks, experience library.

**Implementation checklist**:
- [ ] CLI interface (`prompt_toolkit`), connected to cloud models (`litellm`).
- [ ] Orchestrator main loop: receive input → call LLM to plan → dispatch sub-agents.
- [ ] Sub-agent Factory: Git Worktree isolation (with environment detection, dirty state prompts, fallback strategy); streamed log reads + 2MB hard limit + 3-minute timeout cutoff; change delivery protocol (display diff → user confirm → file copy); structured result return (with `changed_files` and `referenced_files`).
- [ ] SQLite database: message table + trigram FTS5 + WAL.
- [ ] Manual `/recall` command (results by ascending ID).
- [ ] Context: dynamic buffer + static System Prompt.

### Phase 2: Topic-Based Memory & Manual Compression (2-3 weeks)
**Goal**: Introduce structured topics, verify layered memory and correction closed loop.

**New additions**:
- [ ] `/topic new/switch/close` commands.
- [ ] Topic data structure implementation, auto message association, sub-agent execution write-back to `associated_files` and `referenced_files`.
- [ ] Manual `/compact` triggering synchronous compression: post-priority principle + `rationale` recording + decision checklist Checksum.
- [ ] File hash anchoring with silent degradation comparison.
- [ ] Intent detection hints + manual `/recall` dual safeguard.
- [ ] Topic over-threshold split reminders.

**Testing**: 3-5 day real project development, tracking memory correction frequency.

### Phase 3: Autonomous Cognitive Engine & Cold Storage (4-6 weeks)
**Goal**: System autonomously maintains memory health, integrates experience library.

**New additions**:
- [ ] Async compression queue: watermark + CAS version check.
- [ ] Context over-threshold auto-trigger background compression.
- [ ] Topic dependency chain breakage warnings.
- [ ] Memory expiration & proactive invalidation (set expiry on time-sensitive constraints).
- [ ] Multi-dimensional file anchoring expansion.
- [ ] Cold storage experience library (manual curation + vector retrieval).
- [ ] TUI upgrade to `textual`.
- [ ] Monitoring dashboard.

---

## 7. Monitoring & Success Metrics

- **Memory Accuracy**: User proactive corrections / total interaction turns (long-term downward trend).
- **Compression Fidelity**: Manual spot-check of 20% of compression summaries; key decision retention rate > 90%.
- **Recall Effectiveness**: Ratio of recall triggers that successfully resolve user challenges.
- **Token Efficiency**: Average token consumption per task.
- **Task Completion Rate**: Success ratio for 50+ turn complex tasks.

---

## 8. Version Revision History

| Version | Key Revisions |
|---------|---------------|
| v1-v3 | Initial architecture design: layered storage, topic compression, sub-agent isolation |
| v4 | Git Worktree replacing checkout, streamed log fuse, FTS5 trigram, static prefix caching |
| v5 | Change delivery protocol, dirty workspace handling, dynamic buffer, objective compression confidence |
| v6 | `key_decisions` adds `rationale` field, intent detection changed to hint mode, `referenced_files` dual list, topic index lazy loading |

---
