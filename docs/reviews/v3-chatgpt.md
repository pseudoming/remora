# Review v3 (ChatGPT)

> 这版我给的判断是：**可以开工，且比前两版更接近可跑的产品方案**。它已经不是"概念拼图"，而是一个能落到代码结构里的系统设计了。真正的问题不在大方向，而在 **Phase 1 需要再收窄边界**，否则你会很快被状态管理、工具失败、压缩一致性这些工程细节拖住。

---

## 最成熟的部分

四层职责已经基本清晰：Orchestrator 负责决策，Context Manager 负责记忆生命周期，Sub-agent 负责一次性执行，Memory Bus 负责持久化。这个分层是对的。

SQLite + WAL + FTS5 作为本地温存储是合理的：WAL 通过 `PRAGMA journal_mode=WAL;` 开启，FTS5 就是 SQLite 的全文检索虚拟表模块，适合本地 CLI/TUI 这种单机优先场景。([SQLite](https://sqlite.org/wal.html))

MCP 适合作为后续工具适配层，不适合作为 Phase 1 的核心依赖。

---

## 四个硬缺口

### 1）缺一个明确的"状态机合同"

现在文档把模块讲清楚了，但没有把一轮交互的状态转移写死。你需要明确这些状态：`idle / planning / tool_running / waiting_user / compacting / recalling / error_recovering`，以及每个状态允许哪些输入、哪些输出、哪些重试、哪些取消。没有这个，后面并发一上来，系统会变成一堆互相抢写上下文的协程。

### 2）子代理的输出协议还不够硬

现在你说"返回精简结果 + 完整日志落盘"，方向没问题，但要补一个强制 schema。至少要有：`status / changed_files / commands_run / tests / risks / next_actions / raw_log_ref`。不然主干只能读自然语言总结，仍然会被污染。这个地方建议直接把错误类型也结构化掉，比如 `timeout / malformed_output / command_denied / sandbox_violation / git_conflict / test_fail`。

### 3）Topic 机制还要再加"拆分/合并规则"

你已经解决了"怎么创建 topic"，但还没解决"什么时候一个 topic 该拆成两个，或者两个该合并"。这个问题不做，长期下来 topic 会越来越脏。建议 Phase 2 先上人工打标，自动只做建议；并且给每个 topic 加一个"边界证据"，记录它为什么被认为是同一主题，而不是只存标题和摘要。

### 4）异步压缩必须引入版本提交协议

你已经加了 `watermark`，这是对的，但还不够。后台压缩拿的是快照，主线程又在继续写，最后提交时必须做 compare-and-swap：如果 `topic.version` 或 `watermark` 已变化，这次压缩结果就只能丢弃或重算，不能直接覆盖。否则异步压缩会把新消息吞掉。SQLite 的 WAL 适合并发读写，但并不替你解决业务层的一致性问题。([SQLite](https://sqlite.org/wal.html))

---

## 对 Phase 1 的建议：再砍一刀

Phase 1 不要碰自动 topic、自动压缩、语义校验、经验库。就做这五件事：

1. CLI/TUI
2. Orchestrator 主循环
3. 子代理沙箱 + Git 临时分支回滚
4. 消息入库 + FTS 检索 + 手动 `/recall`
5. 结构化日志落盘

只要这五件事跑通，你就已经拿到一个真正能用的"长周期协作底座"了。后面的 topic 化、压缩、版本化校验，都是在这个底座上加能力，不是 Phase 1 的前置条件。

---

## 结论

这版方案已经从"有想法"进化到"有工程骨架"了，**值得开工**。但它还不是"自动稳态系统"，它现在更像一个**需要严格边界控制的本地协作内核**。先把状态机、输出协议、版本提交这三根钉子钉死，后面才不会散。
