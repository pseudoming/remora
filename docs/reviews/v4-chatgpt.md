# Review v4 (ChatGPT)

> 这版已经到 **"可以开工，但要带着两条红线开工"** 的程度了。方向上比 v3 更稳，尤其是你把 **Git Worktree 沙箱、流式日志熔断、静态 System Prompt、FTS5 trigram、WAL** 这些工程点都落到了明确实现上；这些组件本身都是有依据的：SQLite 支持 WAL，FTS5 支持 trigram tokenizer，Git worktree 也确实支持同一仓库的多个 working tree。([SQLite](https://sqlite.org/wal.html))

---

## 总体判断

v4 可以进入 Phase 1，但 Phase 1 只能做"最小闭环"，不能顺手把 Phase 2/3 的东西偷跑进去。你现在的方案已经具备落地骨架，但还没有到"边开工边自动演进也不会乱"的程度。

我认为这版最好的地方有四个：

**第一，沙箱方案终于现实了。** Git Worktree 做隔离，比在主工作区里 checkout 来 checkout 去安全得多，因为它允许同一仓库存在多个工作树，天然适合一次性子代理执行。([Git](https://git-scm.com/docs/git-worktree))

**第二，日志处理补对了。** 流式读取 + 定长缓冲区 + 硬超时，这一套是防 OOM 的正解。你不再依赖"输出别太多"这种软约束，而是用工程手段兜底。

**第三，检索层终于开始像产品了。** SQLite FTS5 的 trigram tokenizer 是官方支持的，适合做子串式检索；对代码符号、路径片段、中文混排这种场景，比纯分词思路更实际。([SQLite](https://sqlite.org/fts5.html))

**第四，你把动态内容和静态前缀分开了。** 这对云端模型的前缀缓存很重要，尤其是子代理高频创建时，减少重复前缀能直接省成本、减延迟。

---

## 四个必须先钉死的问题

### 1）还没有把"主干状态机合同"写成硬规范

现在模块分得很清楚，但一轮交互到底允许哪些状态转移，还没定死。至少要明确：`idle / planning / tool_running / waiting_user / recalling / error_recovering`。每个状态的输入、输出、可重试动作、超时后的回退都要写出来。否则协程一多，主干会开始互相抢上下文。

### 2）`facts / decisions / constraints / open_questions` 虽然已经分开，但还不够"可验证"

你现在的数据结构比之前强很多，但还缺一个"证据链字段"的统一规范。比如每个 decision、constraint 都应该强制带 `evidence_msg_ids` 或同等引用，不然后续 checksum 只能比对"有没有这句话"，不能比对"这句话是不是还站得住"。

### 3）"FTS5 trigram = 完美支持中文" 这个表述要收回

它支持更一般的子串匹配，这很适合中文和代码符号检索，但"完美"太满了。实际效果还要看你怎么做查询分解、怎么处理短词、怎么处理代码路径和自然语言混检索。官方文档只保证 trigram 的工作方式是按连续三字符切分来支持 substring matching，不该把它说成万能中文分词。([SQLite](https://sqlite.org/fts5.html))

### 4）Worktree 沙箱还缺一个清晰的失败恢复协议

你现在已经有 worktree add/remove 的方向了，这没问题；但实现里必须规定：worktree 创建失败怎么办、子代理异常退出后怎么清理、有未提交改动时如何处理、用户本地同时修改同文件时怎么避免误删。Git worktree 机制本身支持多工作树，但它不替你处理业务层的恢复策略。([Git](https://git-scm.com/docs/git-worktree))

---

## 对 Phase 1 的建议

Phase 1 只做四件事：

1. CLI / TUI
2. Orchestrator 主循环
3. 子代理 worktree 沙箱 + 日志熔断
4. SQLite 温存储 + `/recall`

**不要在 Phase 1 做：Topic、压缩、自动召回、经验库、CAS、异步后台压缩。** 这些都很好，但它们属于 Phase 2/3。现在硬塞进去，只会把第一版拖死。

---

## 最终评级

| 维度 | 评分 |
|------|------|
| 方向 | 8.5/10 |
| 工程可开工性 | 7.5/10 |
| 离可维护产品还差的关键件 | 状态机合同、证据链规范、失败恢复协议 |

**一句话结论**：这版已经可以作为 Phase 1 的开发蓝图，但开工前必须先把"状态机 + 输出协议 + 沙箱失败恢复"三份合同写死。这三份不写，后面一定返工。
