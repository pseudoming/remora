# Roadmap

> [中文版本](./ROADMAP_CN.md)

## Phase 0 — Concept Release (Current)

**Goal**: Publish the design, accept discussion and critique.

- [x] Concept documentation
- [x] Engineering architecture v6
- [x] Design review records
- [ ] Incorporate community feedback, iterate on design

## Phase 1 — Minimum Viable Closed Loop

**Goal**: Validate the core hypothesis — does structured decision memory actually reduce rework and correction frequency?

**Explicitly out of scope**: Auto-compression, experience library, TUI, full execution isolation.

- [ ] CLI interface with cloud model access (litellm)
- [ ] SQLite warm storage (WAL + trigram FTS5)
- [ ] `/topic new/switch/close` manual topic management
- [ ] `/compact` manual compression (post-priority principle + rationale recording + negative decisions)
- [ ] Hard anchor mechanism: `user_confirmed` flag + mandatory compression retention check
- [ ] `/recall <keyword>` recall command

**Validation metrics**:
- User correction count / total interaction turns (should trend downward)
- Key decision retention rate after compression (target > 90%)

## Phase 2 — Structured Memory Refinement

- [ ] Topic dependency associations (explicit `dependencies` field)
- [ ] Sub-agent execution isolation (Git Worktree)
- [ ] File association anchoring (`associated_files` / `referenced_files`)
- [ ] Intent detection (challenge/negation patterns trigger recall prompts)
- [ ] Topic over-threshold split reminders

## Phase 3 — Autonomous Cognitive Engine

- [ ] Async background compression (watermark + CAS version check)
- [ ] Cold storage experience library (cross-project lessons learned)
- [ ] Topic dependency chain breakage warnings
- [ ] Memory expiration and proactive invalidation
- [ ] TUI upgrade (textual)
- [ ] Monitoring dashboard

---

> If you think a feature's priority should be adjusted, open an Issue explaining why.
