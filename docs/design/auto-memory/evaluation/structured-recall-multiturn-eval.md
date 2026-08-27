# Structured Auto Memory Recall: Multi-Turn Evaluation

This report covers six eight-turn conversations: 48 turns per variant, 96 A/B task turns, 48 blind answer judgments, 96 recall judgments, and six session-level judgments. A is the flat-memory baseline; B4 is structured tree recall with focused subtrees and on-demand body access.

## Evaluation setup

- Repository revision: `4c21f39ac02faf5323d353454414b09548c5cd7e`.
- Task model: `DeepSeek/deepseek-v4-flash`.
- Selector model: `qwen3.7-max` through the Idealab OpenAI-compatible endpoint.
- Judge model: `qwen3.7-max`.
- Each conversation used one persistent headless `stream-json` process and session id. Turns within a conversation were sequential; independent session/variant jobs ran concurrently.
- A and B4 used isolated project copies, runtime directories, and `QWEN_HOME` settings. User permissions, hooks, MCP servers, custom tools, and user skills were not inherited. Dream, Extraction, and background memory maintenance were disabled.
- Answer and session judging was blind and position-randomized. The answer judge saw only the anonymous conversation histories, final answers, and critical facts; it was explicitly instructed not to infer or reward the recall mechanism.
- The benchmark intentionally stresses memory recall. It is not sampled from natural product traffic.

## Quality results

| Metric | A baseline | B4 structured | B4 - A |
| --- | ---: | ---: | ---: |
| Task score (0-10, 46 evaluable turns) | 7.587 | 8.348 | +0.761 |
| Recall score (0-10, 95 evaluable runs) | 4.000 | 6.809 | +2.809 |
| Session score (0-10, 6 sessions) | 7.667 | 8.167 | +0.500 |
| Answer wins | 7 | 12 | +5 |
| Session wins | 2 | 1 | -1 |

There were 27 answer ties and three session ties. Three of 150 judge jobs failed because the judge returned malformed JSON; all six session judgments completed. The exported dataset retains these failures instead of silently dropping them.

## Runtime and token results

| Metric | A baseline | B4 structured | B4 - A |
| --- | ---: | ---: | ---: |
| Runtime failures | 0/48 | 0/48 | 0 |
| Main-model requests | 169 | 150 | -11.2% |
| Main-path total tokens | 12,492,207 | 11,014,046 | -11.8% |
| Main-path uncached input | 398,502 | 401,803 | +0.8% |
| Main-path output tokens | 125,321 | 101,011 | -19.4% |
| P50 latency | 13.5 s | 17.3 s | +28.9% |
| P95 latency | 113.8 s | 102.5 s | -9.9% |
| Average latency | 44.0 s | 34.3 s | -22.0% |
| Maximum latency | 570.3 s | 171.5 s | -69.9% |

The 94 selector calls consumed 535,366 additional tokens: 511,200 input, 25,137 cache-read input, and 24,166 output. Including selector usage, B4 consumed 11,549,412 total tokens, 7.5% below A. However, combined uncached input rose from 398,502 to 887,866 because selector prompts had little cache reuse. Monetary cost therefore depends on the relative main-model, selector-model, and cache pricing; total-token reduction alone does not prove lower billing cost.

B4 improved average and tail latency but regressed median latency. It adds fixed recall work to simple turns while reducing some long repository/tool loops. A also had one 570-second outlier, so the average-latency delta should not be read without the percentile data.

## Tool and memory behavior

| Tool | A calls / failures | B4 calls / failures |
| --- | ---: | ---: |
| All tools | 189 / 2 | 157 / 1 |
| `read_file` | 84 / 0 | 59 / 0 |
| `grep_search` | 54 / 0 | 57 / 0 |
| `glob` | 25 / 0 | 15 / 0 |
| `run_shell_command` | 19 / 2 | 3 / 0 |
| `search_memory` | 0 / 0 | 23 / 1 |

B4 fetched 30 memory bodies. It reduced newly introduced memory prompt text from 159,523 to 96,338 characters and the corresponding heuristic estimate from 46,821 to 35,773 tokens. It returned 1,291 duplicate body characters, so body reuse was good but not perfect.

The selector made 94 runtime calls: 61 succeeded, 33 were best-effort aborts after the owning turn completed or cleaned up, and none failed because of authentication, quota, or model availability. All selector requests used `qwen3.7-max`.

## Benchmark bias and over-recall

The case mix is recall-heavy: 12 body turns, nine metadata turns, 14 history-reuse turns, two current-state turns, five repository turns, and only six no-memory controls. Thirteen prompts explicitly mention memory, and seven explicitly ask to read or explain a body. The observed memory-tool rate therefore should not be treated as a normal-product traffic estimate.

Of 23 `search_memory` calls, 10 occurred on body turns and 13 on non-body turns. Metadata turns triggered ten calls, while history, repository, and current-state turns triggered one each. This shows that the model still fetches too often when metadata or conversation history should be sufficient.

None of the six no-memory controls called `search_memory`, but four received newly focused memory text. Recall judges consistently penalized this as irrelevant context. The benchmark therefore exposes B4 over-recall rather than only rewarding structured recall.

## Interpretation

In this recall-heavy multi-turn benchmark, B4 improves task, recall, and average session scores while reducing main-model requests, total tokens including selector, average latency, and P95 latency. The strongest quality evidence is the blind task score, because that judge cannot see the recall mechanism.

The result does not establish the same gain for natural Qwen Code traffic. The benchmark overrepresents explicit memory requests, B4 doubles combined uncached input after selector usage, median latency regresses, metadata turns over-fetch bodies, and no-memory turns sometimes receive irrelevant focused subtrees. A natural-traffic benchmark with a much lower memory-need base rate is required before making a product-wide cost or quality claim.

The exported data intentionally excludes raw API logs, complete transcripts, memory bodies, credentials, and hidden model reasoning.

- [Case definitions](./structured-recall-multiturn-eval-cases.json)
- [Per-turn results](./structured-recall-multiturn-eval-results.json)

<details>
<summary>中文翻译</summary>

# 结构化 Auto Memory 召回：多轮评测

本报告覆盖 6 组、每组 8 轮的连续对话：每个变体 48 轮，共 96 次 A/B 任务执行、48 次匿名答案评审、96 次召回评审和 6 次 session 级评审。A 为平铺记忆基线，B4 为完整树、Focused Subtree 与按需正文读取组成的结构化召回链路。

## 评测设置

- 仓库版本：`4c21f39ac02faf5323d353454414b09548c5cd7e`。
- 主模型：`DeepSeek/deepseek-v4-flash`。
- Selector：通过 Idealab OpenAI-compatible endpoint 调用 `qwen3.7-max`。
- Judge：`qwen3.7-max`。
- 每组对话使用一个持续存活的 headless `stream-json` 进程和固定 session id。同一 session 内按顺序执行，不同 session/variant 并行。
- A/B 使用隔离的项目副本、runtime 和 `QWEN_HOME`，不继承用户 permissions、hooks、MCP、自定义工具和 user skills。Dream、Extraction 和后台记忆维护关闭。
- 答案与 session 评审采用匿名、确定性换位。答案 Judge 只能看到匿名历史、最终答案和关键事实，并被明确要求不得推断或奖励召回机制。
- 本评测有意对记忆召回施压，不是自然产品流量抽样。

## 质量结果

| 指标 | A 基线 | B4 结构化召回 | B4 - A |
| --- | ---: | ---: | ---: |
| 任务得分（46 个可评估轮次） | 7.587 | 8.348 | +0.761 |
| 召回得分（95 个可评估运行） | 4.000 | 6.809 | +2.809 |
| Session 得分（6 组） | 7.667 | 8.167 | +0.500 |
| 答案胜场 | 7 | 12 | +5 |
| Session 胜场 | 2 | 1 | -1 |

答案评审包含 27 次平局，session 评审包含 3 次平局。150 个 Judge 任务中有 3 个因返回 JSON 格式错误而失败；6 个 session 总评全部完成。导出的数据保留这些失败，没有静默丢弃。

## 性能与 Token

| 指标 | A 基线 | B4 结构化召回 | B4 - A |
| --- | ---: | ---: | ---: |
| 运行失败 | 0/48 | 0/48 | 0 |
| 主模型请求 | 169 | 150 | -11.2% |
| 主链路总 Token | 12,492,207 | 11,014,046 | -11.8% |
| 主链路未缓存输入 | 398,502 | 401,803 | +0.8% |
| 主链路输出 Token | 125,321 | 101,011 | -19.4% |
| P50 延迟 | 13.5 秒 | 17.3 秒 | +28.9% |
| P95 延迟 | 113.8 秒 | 102.5 秒 | -9.9% |
| 平均延迟 | 44.0 秒 | 34.3 秒 | -22.0% |
| 最大延迟 | 570.3 秒 | 171.5 秒 | -69.9% |

94 次 selector 调用额外消耗 535,366 Token，其中输入 511,200、缓存输入 25,137、输出 24,166。计入 selector 后，B4 总 Token 为 11,549,412，比 A 低 7.5%。但由于 selector prompt 缓存复用很少，合并后的未缓存输入从 398,502 增至 887,866。真实费用取决于主模型、selector 和缓存 Token 的相对价格，不能只凭总 Token 下降宣称账单成本降低。

B4 改善了平均和长尾延迟，但中位延迟退化。简单轮次承担了固定召回开销，复杂仓库/工具循环则有时缩短。A 还存在一次 570 秒异常长尾，因此平均值必须结合分位数解读。

## 工具与记忆行为

| 工具 | A 调用/失败 | B4 调用/失败 |
| --- | ---: | ---: |
| 全部工具 | 189 / 2 | 157 / 1 |
| `read_file` | 84 / 0 | 59 / 0 |
| `grep_search` | 54 / 0 | 57 / 0 |
| `glob` | 25 / 0 | 15 / 0 |
| `run_shell_command` | 19 / 2 | 3 / 0 |
| `search_memory` | 0 / 0 | 23 / 1 |

B4 共返回 30 个记忆正文。新引入的记忆 prompt 文本从 159,523 降到 96,338 字符，相应启发式 Token 估算从 46,821 降到 35,773。重复返回正文为 1,291 字符，正文复用总体有效但并不完美。

Selector 共发起 94 次运行时调用：61 次成功，33 次在所属 turn 完成或清理后 best-effort abort，没有鉴权、额度或模型不可用错误。全部请求使用 `qwen3.7-max`。

## Case 偏向与过召回

Case 分布明显偏召回：12 个正文轮次、9 个 metadata 轮次、14 个 history 复用轮次、2 个 current-state 轮次、5 个 repository 轮次，只有 6 个 no-memory 对照。13 个问题直接提到记忆，7 个问题直接要求读取或解释正文。因此本次 `search_memory` 频率不能视为正常产品流量估计。

23 次 `search_memory` 中只有 10 次发生在 body 轮次，另外 13 次发生在非正文场景。Metadata 轮次触发 10 次，history、repository、current-state 各触发 1 次。这说明 metadata 或上下文历史已经足够时，模型仍会过度 fetch。

6 个 no-memory 对照均未调用 `search_memory`，但其中 4 个仍新增了 Focused memory。Recall Judge 一致将其视为无关上下文并扣分。因此这套评测能暴露 B4 过召回，并非只奖励结构化召回。

## 结论

在这套偏记忆召回的多轮压力评测中，B4 提高了任务、召回和平均 session 得分，同时减少主模型请求、包含 selector 的总 Token、平均延迟和 P95 延迟。最有价值的质量证据是匿名 Task Score，因为该 Judge 看不到召回机制。

该结果不能证明自然 Qwen Code 流量可获得相同收益。评测过度代表显式记忆需求；计入 selector 后未缓存输入翻倍；P50 延迟退化；metadata 场景存在正文过取；no-memory 场景仍可能注入无关 Focus。要形成产品级成本和质量结论，还需要增加低记忆需求占比的自然流量评测集。

导出的数据有意排除原始 API 日志、完整 transcript、记忆正文、凭据和模型隐藏 reasoning。

- [Case 定义](./structured-recall-multiturn-eval-cases.json)
- [逐轮结果](./structured-recall-multiturn-eval-results.json)

</details>
