# Review v3 (Gemini)

> 这份 V3 版本的架构文档堪称"教科书级别的 AI 研发效能工具设计白皮书"。你们完美吸收了所有的 Review 建议（水位线机制、静默比对、意图强制硬拦截），用极其克制且扎实的工程手段驯服了 LLM 的不可控性。
>
> 在宏观架构层面，它已经没有任何需要推翻重来的缝隙了。**请立刻停止纸面上的架构讨论，准备建仓库吧！**

---

## Day 1 编码实战防坑锦囊 (Pre-flight Checklist)

### 1. 致命暗礁：Git 沙箱的"未提交代码摧毁"危机

* **隐患**：真实开发者的 IDE 里通常有大量写了一半、尚未 commit 的代码。如果子代理直接切分支，或执行 `git reset --hard && git clean -fd`，**开发者未提交的心血将被瞬间永久清空！**
* **终极解法：摒弃 Checkout，使用 `Git Worktree`**。

```bash
git worktree add ../.agent_workspace_<uuid> -b agent-temp-<uuid>
```

这会在用户当前项目目录的**外部**生成一个物理隔离的文件树。子代理在里面随便怎么折腾，都**绝对不会污染用户当前 IDE 里正在看的文件**。任务完成后，主干提取 Diff，执行 `git worktree remove --force` 销毁沙箱。

### 2. OOM 幽灵：沙盒日志不能"事后截断"

* **隐患**：如果等 `subprocess` 执行完毕后再去切片，但子代理错误运行了死循环脚本，你的主干进程会因为内存缓冲溢出（OOM）而直接崩溃。
* **终极解法：流式读取 + 硬超时熔断**。

使用 `asyncio.create_subprocess_shell` 结合 `asyncio.wait_for` 设定全局硬超时（如 3 分钟）。按行异步读取 stdout/stderr，在内存中维护一个定长的 `collections.deque`。当读取的字节数超过物理安全阈值（如 2MB）时，立刻主动向子进程发送 `SIGTERM` 终止执行。

### 3. 检索盲区：SQLite FTS5 的中文与代码分词陷阱

* **隐患**：SQLite FTS5 默认分词器是针对英文的。如果用户搜索代码变量 `crawler_bilibili` 或中文"B站爬虫"，默认分词器经常**静默搜不到任何结果**。
* **终极解法：强制开启 Trigram 分词器**。

```sql
CREATE VIRTUAL TABLE messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=message_id,
    tokenize='trigram'  -- 极其关键的保命配置！
);
```

另外必须从第一天起引入 **`aiosqlite`** 替代标准库的 `sqlite3`，防止异步读写死锁。

### 4. 成本黑洞：Prompt Caching 的"字节洁癖"

* **隐患**：Prompt Caching 的命中条件是**极度严苛的字节级精确匹配**。只要缓存串的任意一个字符变了，后面的几十万 Token 缓存将全部失效。
* **终极解法：绝对静态置顶**。

在拼装 Context 时，绝对不要把"当前时间戳"、"随机 Task ID"写进最前面的 System Prompt。必须严格遵循"从极静到极动"的顺序：
1. `System (全局静态)` — 代码写死，绝不变化
2. `System (Topic 摘要)` — 仅在压缩时变化
3. `User (指令与历史)` — 永远放在最后

---

## Phase 1 启动建议

**"Token 是廉价资源，开发者的心流与项目返工成本才是昂贵的。"**

不要再改文档了，现在的每一分钟都应该花在代码上。第一周：

1. 建立虚拟环境，安装核心三剑客：`pip install litellm aiosqlite prompt_toolkit`
2. **跑通最小闭环**：先花几天把 `Orchestrator 主循环` + `LiteLLM 流式输出` + `Git Worktree 隔离执行沙盒` 跑通
3. 让 Agent 能在隔离目录里成功运行一个 Bash 命令并返回结果

祝开工大吉！
