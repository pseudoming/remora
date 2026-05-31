# Contributing

> [中文版本](./CONTRIBUTING_CN.md)

This project is currently in **concept / RFC stage**. There is no runnable code.

## Most Useful Ways to Contribute

### 1. Challenge Design Assumptions

What we need most is someone to say "This assumption doesn't hold because..."

Open an Issue with the title format: `[Challenge] <the assumption you question>`

Particularly welcome:
- Real-world long-cycle AI collaboration issues you've encountered, compared against this design
- Scenarios where this design would fail
- Features you think are overvalued or undervalued

### 2. Share Your Scenario

If you're working on something similar (long-cycle AI agent collaboration, LLM memory management), even if your approach is completely different, we'd love to hear about it.

Open a Discussion describing your scenario and pain points.

### 3. Help Implement Phase 1

Phase 1 focuses on four things: warm storage, topic management, manual compression, recall.

If you want to write code, mention it in an Issue first to avoid duplicate work.

Tech stack: Python 3.11+, asyncio, aiosqlite, litellm, prompt_toolkit.

---

## Code Contribution Principles (Starting from Phase 1)

- Each PR corresponds to one clear feature or fix — don't mix concerns
- For architecture-level changes, discuss in an Issue before writing code
- Test coverage for core paths (compression, recall, topic management)

---

> No CLA, no bureaucracy. MIT license — you own the code you write.
