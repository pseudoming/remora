# 设计评审记录

> [English](./README.md)
>
> Remora 架构的批判性分析与修正记录。

架构已经历了六轮正式评审，覆盖六个模型，每次评审针对逐步深入的不同工程关切。

## Claude 评审

| 轮次 | 焦点 | 识别出的关键问题 |
|------|------|-----------------|
| [v1](v1-claude.md) | 可行性评估 | 三个硬骨头：压缩验证、话题边界识别、冷存储质量 |
| [v2](v2-claude.md) | 工程细化 | Checksum 格式、异步压缩边界、话题相关性过滤 |
| [v3](v3-claude.md) | 实现前就绪度 | Git 隔离边缘案例、`associated_files` 冷启动、倒序压缩 Prompt 澄清 |
| [v4](v4-claude.md) | 工程完备性 | Worktree 接受变更协议、`facts` 字段定义、`compression_confidence` 计算公式 |

## ChatGPT 评审

| 轮次 | 焦点 | 识别出的关键问题 |
|------|------|-----------------|
| [v1](v1-chatgpt.md) | 核心价值评估 | 话题切分难度、摘要漂移、复杂度成本、需要可衡量指标 |
| [v2](v2-chatgpt.md) | MVP 就绪度 | 话题可计算性、版本化压缩、异步一致性、语义级校验 |
| [v3](v3-chatgpt.md) | Phase 1 范围收窄 | 状态机合同、子代理输出 Schema、话题拆分/合并规则、CAS 版本提交 |
| [v4](v4-chatgpt.md) | 最终实现前检查 | 状态机规范、证据链格式、FTS5 trigram 局限性、Worktree 恢复协议 |

## Gemini 评审

| 轮次 | 焦点 | 识别出的关键问题 |
|------|------|-----------------|
| [v1](v1-gemini.md) | 工业可行性 | 话题纠缠、代码-记忆状态脱节、STW 压缩、LLM 拒绝召回 |
| [v2](v2-gemini.md) | 施工蓝图 | 异步竞争条件、哈希锚定脆弱性、CLI/异步线程冲突、日志爆炸 |
| [v3](v3-gemini.md) | Day 1 编码指南 | Git Worktree 替代 Checkout、流式 OOM 熔断、FTS5 trigram 中文支持、Prompt 缓存字节洁癖 |
| [v4](v4-gemini.md) | 净室红队推演 | 脏工作区悖论、热记忆时间黑洞、意图误杀风暴、关联爆炸、时序幻觉 |

## DeepSeek、Doubao、Qwen 评审（v5）

| 评审 | 焦点 | 识别出的关键问题 |
|------|------|-----------------|
| [DeepSeek v5](v5-deepseek.md) | 最终实现前验证 | 否定性决策 Checksum、记忆有效期机制、话题依赖链预警、Phase 1 最小会话摘要 |
| [Doubao v5](v5-doubao.md) | 全面架构验证 | 700 倍 ROI 分析、referenced-files 依赖缺口、多话题复杂度天花板、UX 告警疲劳缓解 |
| [Qwen v5](v5-qwen.md) | 深层工程陷阱与 V5 批判 | 压缩中 Rationale 丢失、Worktree 环境依赖、意图检测假阳性、FTS5 语义盲区、话题索引爆炸与懒加载 |

每次评审对应架构文档的特定版本（v1 到 v5）。当前架构为 [v6](../architecture.md)，已吸收了全部十五份评审的反馈及额外迭代。

---

关于当前架构状态，参见：

- [设计哲学](../concept_CN.md)
- [工程实施方案 v6](../architecture_CN.md)
