# 长周期 Code Agent 认知架构与工程实施方案（v6）

> [English](./architecture.md)
>
> **状态**：五轮架构推演、十五份独立 Review 交叉验证通过，进入 Phase 1 编码
> **核心哲学**：用计算换认知安全 — Token 是廉价资源，开发者的心流与项目返工成本才是昂贵的

---

## 1. 概述

### 1.1 项目定位
构建一个本地 CLI/TUI 客户端形态的 Coding Agent，通过云端大模型 API 进行推理，能与开发者进行跨数天、数百轮对话的长周期协作，维持对项目上下文、决策历史和约束条件的连贯认知，避免"上下文腐烂"导致返工或错误决策。

### 1.2 核心价值
- **结构化记忆**：按话题组织对话历史，形成可追溯、可纠错的知识库。
- **无状态执行**：代码修改、测试等操作委派给隔离的、用完即焚的子代理，避免污染主干认知。
- **分层存储**：热记忆、温记忆、冷记忆各司其职。
- **主动纠错**：检测到可能涉及历史记忆时，提示 LLM 回溯原始记录。
- **版本化可审计**：记忆压缩全生命周期可追溯，支持一致性校验与回滚。

### 1.3 核心原则
- **高内聚低耦合**：每个话题内部包含完整决策脉络及深层原因，话题之间通过显式依赖关联。
- **工程确定性对抗模型概率性**：用哈希校验、异步守护、意图检测等传统软件工程手段给 LLM 的输出托底。
- **防御性设计**：承认压缩必然有损，但通过温存储原文保留和召回机制确保错误可逆。
- **渐进式演进**：MVP 严格收窄边界，先跑通最小闭环，再逐步叠加高级特性。

---

## 2. 系统架构

### 2.1 架构总览
```
┌──────────────────── 用户界面 (CLI / TUI) ────────────────────┐
│                                                                │
│  ┌──────────────────┐        ┌────────────────────────────┐   │
│  │  Orchestrator    │◄──────►│  Context Manager           │   │
│  │  (主干循环)       │        │  - 记忆生命周期管理        │   │
│  │  - Planner        │        │  - 异步后台压缩（Phase 3） │   │
│  │  - Dispatcher     │        │  - 压缩校验与版本控制      │   │
│  │  - State Tracker  │        │  - 文件状态锚定            │   │
│  └───┬──────────────┘        └───────────┬────────────────┘   │
│      │ 分发子任务                          │                    │
│      ▼                                    │                    │
│  ┌──────────────────┐                     │                    │
│  │ Sub-agent Factory│                     │                    │
│  │ (Git Worktree    │                     │                    │
│  │  执行沙箱)       │                     │                    │
│  └───┬──────────────┘                     │                    │
│      │ 日志/结果                          │                    │
│      ▼                                    ▼                    │
│  ┌───────────────────────────────────────────┐                │
│  │            Memory Bus                     │                │
│  │  - 温存储 (SQLite + FTS5 trigram)         │                │
│  │  - 冷存储 (经验库，Phase 3)               │                │
│  └───────────────────────────────────────────┘                │
│                                                                │
│   ┌─────────── 云端 LLM API ───────────┐                       │
│   │  DeepSeek / Kimi / OpenAI 等        │                       │
│   └────────────────────────────────────┘                       │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 数据流概要
1. **用户输入** → Orchestrator。
2. Orchestrator 请求 Context Manager 构造当前热记忆上下文（静态 System Prompt + 话题摘要索引 + 自上次压缩水位线以来全部未压缩消息）。
3. 若需执行具体任务，Task Dispatcher 生成任务指令，Context Manager 根据任务涉及的文件筛选关联话题，打包为子代理上下文快照，传给 Sub-agent Factory。
4. 子代理在 Git Worktree 隔离环境中执行，流式读取日志并熔断超限，完成后返回结构化结果（含 modified 和 referenced 文件列表）。
5. 当主干上下文 Token 超过阈值，Orchestrator 通知 Context Manager 触发后台异步压缩（Phase 3 实现）。
6. 用户输入可能涉及历史记忆时，意图检测在 System Prompt 中插入提示，引导 LLM 主动调用温存储检索。

---

## 3. 核心模块设计

### 3.1 Orchestrator（主干）

**职责**：管理对话流，理解用户意图，规划任务，分发执行，维护短期状态。

**组件**：
- **Planner**：调用 LLM 分解任务，识别步骤关联的话题。
- **Task Dispatcher**：组装子代理上下文快照，并在子代理执行后将其实际修改和读取的文件路径回写到当前活跃话题的 `associated_files` 和 `referenced_files`。
- **State Tracker**：维护当前活跃话题、自上次压缩水位线以来的动态消息缓冲、任务执行状态。
- **意图检测旁路**：用户输入进入主逻辑前，先用改进的正则进行匹配。若命中，不直接硬拦截注入上下文，而是在 System Prompt 中插入提示："用户提及了历史内容，如果你对相关事实不确定，建议调用 recall_memory 工具确认事实，不要凭印象回答。"同时保留显式 `/recall` 命令作为最终兜底。

**System Prompt 核心规则**：
> "当用户的输入显示其可能涉及对历史记忆的质疑或引用时，你必须优先使用 `recall_memory` 工具获取原始对话记录，而不能仅依赖上下文中已有的摘要。如果你不确定，请主动建议用户使用 `/recall` 命令。"

**上下文结构**：
```
[System Prompt（全局共享，绝对静态，可被缓存）]
[话题结构化摘要索引（ID + 标题 + 关联文件）]
[当前任务相关话题的详细摘要（按需加载）]
[自上次压缩水位线以来的全部未压缩消息]
[当前用户输入]
```

### 3.2 Context Manager（记忆管理器）—— 核心模块

#### 3.2.1 核心数据结构：Topic
```json
{
  "topic_id": "t_001",
  "title": "B站爬虫频率限制",
  "status": "open",
  "version": 3,
  "summary": "最终方案：动态调整间隔，基于403响应。…",
  "facts": ["B站API限制单IP每秒5次请求"],
  "key_decisions": [
    {
      "decision": "采用动态间隔",
      "status": "confirmed",
      "rationale": "固定间隔在B站场景下仍触发429，动态调整能适应不同时段的限制策略",
      "evidence_msg_ids": [45, 67]
    },
    {
      "decision": "不引入 Redis",
      "status": "confirmed",
      "rationale": "项目规模小，引入 Redis 增加运维复杂度，本地内存缓存足够",
      "evidence_msg_ids": [52]
    }
  ],
  "constraints": [
    {"constraint": "单IP每秒请求数不超过5", "evidence_msg_ids": [45]},
    {"constraint": "必须处理403并回退", "evidence_msg_ids": [67]}
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
- `facts`：客观的、可验证的信息（如 API 文档限制、环境版本号）。
- `key_decisions`：团队/Agent 主动做出的选择，必须包含 `rationale`（深层原因），不仅记录"做了什么"，也记录"明确决定不做什么"及其原因。
- `constraints`：仍生效的约束条件，每条必须附带 `evidence_msg_ids`。
- `associated_files`：记录被修改（modified）的文件；`referenced_files`：记录被读取（referenced）的文件。两者分开存储，在话题关联匹配时 modified 权重更高，referenced 权重次之。
- `compression_confidence`：定义为 Checksum 通过率（保留的关键决策数 / 压缩前提取的关键决策总数），是客观比值，不依赖 LLM 自评。

#### 3.2.2 Topic 生命周期管理（Phase 2 引入）
- **手动打标为主**：用户/主干使用 `/topic new <名称>` 创建，后续消息自动挂载。
- **自动建议为辅**：LLM 检测到话题转变时建议新话题，用户确认。
- **关联文件回写**：子代理执行后，Task Dispatcher 将本次任务实际修改的文件写入 `associated_files`，读取的文件写入 `referenced_files`。
- **话题拆分提醒**：当单个话题消息数超过阈值（如 50 条）时，主动提示用户考虑拆分。

#### 3.2.3 压缩策略（Phase 2 手动，Phase 3 自动）
- **触发**：动态缓冲超阈值（默认 70% 模型窗口）或用户 `/compact`。
- **后优先压缩**：Prompt 明确指示"后发生的决策覆盖早期决策"，并要求记录每个关键决策的 `rationale`（深层原因）、明确记录"决定不做什么"及其原因。
- **决策清单 Checksum**：压缩前提取结构化决策点；压缩后比对保留情况，计算 `compression_confidence`。若信心低于 0.7，在主干下次响应开头插入警告行。
- **异步执行与水位线**（Phase 3）：压缩基于快照，完成后合并水位线之后的新消息。提交时采用 CAS 版本校验，若话题版本已变则丢弃重算。
- **话题依赖链预警**（Phase 3）：当某个话题被更新后，自动在所有依赖它的话题摘要中插入提示。

#### 3.2.4 记忆-代码一致性锚定
- 压缩或事后回写时记录 `associated_files` 和 `referenced_files`。
- 引用话题时校验文件 hash：若变更小则静默更新；若实质变更则插入警告。

#### 3.2.5 温存储检索与 Recall
- **存储**：SQLite + FTS5 trigram 分词器。
- **检索 API**：`search_messages(query, topic_id=None, limit=10)`，结果在代码层强制按 `message_id` 升序排列。
- **触发**：意图检测命中 → System Prompt 插入提示引导 LLM 主动调用；手动 `/recall <关键词>` 作为兜底。

### 3.3 Sub-agent Factory（子代理工厂）

**设计目标**：无状态、环境隔离的一次性执行单元。

**隔离沙箱**：
- **核心方案：Git Worktree**，在项目外部创建独立工作目录。
- **环境探测与回退**：有 Git 且支持 worktree → 使用 worktree；有 Git 但不支持 → stash + checkout + stash pop；无 Git → 文件快照备份。
- **脏工作区处理**：创建 Worktree 前探测目标文件在主工作区的未提交改动。若存在，提供选项：① 先 stash 自动暂存；② 继续（接受后续手动合并）；③ 取消。默认推荐①。
- **结果回传协议**：执行完成后展示 diff → 用户确认 `y/n` → `y` 则复制文件回主工作区（不做 merge），`n` 则彻底丢弃沙箱。
- **失败恢复**：自动清理临时工作树，回退 stash（若已执行），报告失败原因。

**日志与防 OOM**：流式读取 + 定长缓冲区（2MB）+ 硬超时熔断（3 分钟）。

**上下文快照结构**：
```
[System Prompt (全局静态，享受缓存)]
[任务相关话题的详细摘要（通过 associated_files 和 referenced_files 匹配）]
[具体任务指令]
[执行环境配置]
```

**返回协议**：
```json
{
  "status": "success | failure | timeout",
  "summary": "修复了超时逻辑...",
  "changed_files": ["src/crawler/bilibili.py"],
  "referenced_files": ["src/crawler/types.py", "src/config.py"],
  "commands_run": ["python test_crawler.py"],
  "risks": ["频率限制可能仍不完善"],
  "raw_log_ref": "msg_456"
}
```
- `changed_files` 对应 `associated_files`（modified），`referenced_files` 对应 `referenced_files`。

### 3.4 Memory Bus（记忆总线）

**温存储**：
- **数据库**：SQLite，启用 WAL 模式。
- **消息表**：含 `message_id`, `timestamp`, `role`, `content`, `topic_id`, `metadata`。
- **全文索引**：FTS5 trigram 分词器，检索结果强制按 `message_id` 升序排列。
- **备份**：压缩前自动备份数据库文件。

**冷存储**（Phase 3）：本地 Markdown/JSON 或向量数据库，人工策展 + 自动推荐经验规则。

---

## 4. 关键技术决策与风险缓解

| 决策 | 方案 | 风险缓解 |
|------|------|----------|
| **执行隔离** | Git Worktree 为主，三级回退，脏状态检测 | 彻底隔离用户工作区，避免代码丢失 |
| **日志处理** | 流式读取 + 硬超时 + 定长缓冲区 | 防止子进程输出撑爆内存 |
| **全文检索** | FTS5 trigram + 结果时序重排 | 支持中文/代码符号子串匹配，保证因果顺序 |
| **缓存优化** | System Prompt 绝对静态，动态内容后置 | 确保云端前缀缓存最高命中率 |
| **话题创建** | 手动打标为主，事后文件锚定 | 避免 LLM 切分错误，精准关联代码 |
| **关联过滤** | modified + referenced 双列表，按权重匹配 | 不遗漏隐式依赖，控制上下文噪声 |
| **决策记录** | `key_decisions` 强制包含 `rationale` 和否定性决策 | 保留深层原因，避免未来重复已否决方案 |
| **压缩策略** | 后优先原则，决策清单 Checksum，信心阈值告警 | 防止早期错误覆盖后期修正 |
| **异步压缩** | 水位线 + CAS 版本校验（Phase 3） | 防止覆盖新消息，保证并发安全 |
| **Recall 触发** | 意图检测提示 + LLM 自主决策 + 手动兜底 | 避免假阳性打断心流，保证防御性 |

---

## 5. 技术栈与工程建议

- **语言**：Python 3.11+，`asyncio` 异步核心。
- **LLM API**：`litellm` 统一适配，内置成本追踪。
- **终端交互**：Phase 1 用 `prompt_toolkit`；Phase 3 可升级 `textual` TUI。
- **沙箱**：`git worktree` + `asyncio.subprocess`，命令白名单。
- **存储**：`aiosqlite` 异步读写 SQLite（WAL + trigram FTS5）。
- **工具标准**：建议工具接口遵循 MCP 协议。

---

## 6. 分阶段 MVP 实施计划

### Phase 1：基础无状态协作与温存储（2 周）
**目标**：跑通主干-子代理最小闭环，验证执行隔离与日志存储。

**明确不做**：话题化、压缩、后台任务、经验库。

**实现列表**：
- [ ] CLI 界面（`prompt_toolkit`），接入云端模型（`litellm`）。
- [ ] Orchestrator 主循环：接收输入 → 调用 LLM 规划 → 分发子代理。
- [ ] Sub-agent Factory：Git Worktree 隔离（含环境探测、脏状态提示、回退策略）；流式日志读取 + 2MB 硬限制 + 3 分钟超时熔断；变更回传协议（展示 diff → 用户确认 → 文件复制）；结构化结果返回（含 changed_files 和 referenced_files）。
- [ ] SQLite 数据库：消息表 + trigram FTS5 + WAL。
- [ ] 手动 `/recall` 命令（结果按 ID 升序）。
- [ ] 上下文：动态缓冲 + 静态 System Prompt。

### Phase 2：话题化记忆与手动压缩（2-3 周）
**目标**：引入结构化话题，验证分层记忆和纠错闭环。

**新增**：
- [ ] `/topic new/switch/close` 命令。
- [ ] Topic 数据结构落地，消息自动关联，子代理执行后回写 `associated_files` 和 `referenced_files`。
- [ ] 手动 `/compact` 触发同步压缩：后优先原则 + `rationale` 记录 + 决策清单 Checksum。
- [ ] 文件哈希锚定与静默降级比对。
- [ ] 意图检测提示 + 手动 `/recall` 双保险。
- [ ] 话题超阈值提醒拆分。

**测试**：跨 3-5 天的真实项目开发，统计记忆纠正次数。

### Phase 3：全自动认知引擎与冷存储（4-6 周）
**目标**：系统自主维护记忆健康，接入经验库。

**新增**：
- [ ] 异步压缩队列：水位线 + CAS 版本校验。
- [ ] 上下文超阈值自动触发后台压缩。
- [ ] 话题依赖链断裂预警。
- [ ] 记忆有效期与主动失效（对有时效性的约束设置 expiry）。
- [ ] 多维文件锚定拓展。
- [ ] 冷存储经验库（人工策展 + 向量检索）。
- [ ] TUI 升级为 `textual`。
- [ ] 监控面板。

---

## 7. 监控与成功指标

- **记忆准确性**：用户主动纠正次数 / 总交互轮次（长期下降趋势）。
- **压缩保真度**：人工抽查 20% 压缩摘要，关键决策点保留率 > 90%。
- **召回有效性**：Recall 触发后成功解决用户质疑的比例。
- **Token 效率**：每次任务平均 Token 消耗。
- **任务完成率**：50+ 轮复杂任务的成功比例。

---

## 8. 版本修正记录

| 版本 | 主要修正 |
|------|----------|
| v1-v3 | 初始架构设计，分层存储、话题压缩、子代理隔离 |
| v4 | Git Worktree 替代 checkout，流式日志熔断，FTS5 trigram，静态前缀缓存 |
| v5 | 变更回传协议，脏工作区处理，动态缓冲，压缩置信度客观化 |
| v6 | `key_decisions` 增加 `rationale` 字段，意图检测改为提示模式，`referenced_files` 双列表，话题索引懒加载 |
