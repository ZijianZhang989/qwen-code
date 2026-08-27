# Structured Auto Memory Recall: 60-Case Evaluation

This report covers the complete large-sample run: 60 independent cases, 120 A/B task runs, 60 pairwise answer judgments, and 120 recall judgments. A is the flat-memory baseline; B4 is structured tree recall with on-demand memory access.

## Evaluation setup

- Repository context: full Qwen Code checkout at the recorded case revision.
- Task model: `DeepSeek/deepseek-v4-flash`.
- Variants, queries, repository context, and memory corpus were held constant; only the recall path changed.
- Every task run and every judge item used a fresh context.
- Answer judging was blind and position-randomized. The exported results restore the deterministic A/B position mapping.

## Results

| Metric | A baseline | B4 structured | B4 - A |
| --- | ---: | ---: | ---: |
| Task score (0-10, 37 evaluable cases) | 7.703 | 7.649 | -0.054 |
| Exclusive wins | 10 | 8 | -2 |
| Recall quality (0-10, 60 cases) | 8.633 | 8.667 | +0.033 |
| Rescue contributions | 21 | 30 | +9 |
| Helpful contributions | 34 | 24 | -10 |
| Missed contributions | 0 | 2 | +2 |
| P50 latency | 51.0 s | 39.1 s | -23.4% |
| P95 latency | 194.3 s | 189.1 s | -2.7% |
| Runtime failures | 4/60 | 5/60 | +1 |
| Main-path total tokens | 24,355,312 | 18,860,402 | -22.6% |
| Main-path uncached input | 3,217,774 | 2,825,328 | -12.2% |
| Background memory tokens | 38,148 | 373,040 | 334,892 |
| Total tokens including background | 24,393,460 | 19,233,442 | -21.2% |
| All tool calls | 573 | 444 | -129 |
| Tool call failures | 22 | 23 | 1 |

Answer outcomes: 17 ties, 2 both-fail cases, 23 not-evaluable cases, and 0 judge execution failures.

The paired 95% confidence interval is [-0.978, 0.870] for the B4-minus-A task score and [-0.406, 0.473] for recall quality. Both cross zero.

## Tool behavior

| Tool | A calls / failures | B4 calls / failures |
| --- | ---: | ---: |
| All tools | 573 / 22 | 444 / 23 |
| `read_file` | 171 / 5 | 94 / 0 |
| `grep_search` | 145 / 4 | 99 / 1 |
| `run_shell_command` | 162 / 12 | 94 / 2 |
| `search_memory` | 0 / 0 | 97 / 18 |

B4 made 73 memory fetch calls and 24 memory search calls. The 18 failed `search_memory` calls are retained in the evidence and mainly reflect tool-protocol misuse observed in the run, including passing search-only fields to fetch and using a cursor with search.

## Interpretation

Task quality and recall quality are effectively tied in this sample. B4 has more rescue contributions and more high-scoring recalls, but it also has two misses. The strongest measured benefits are lower median latency and lower total token use. Runtime reliability is slightly worse by one failed run.

The 23 not-evaluable answer cases are retained rather than discarded. They require repository state, user configuration, Git remotes, or omitted code that the isolated answer judge could not independently verify. Recall quality remains evaluable for all 60 cases.

Selector-level hit rate, abort rate, and focused-subtree delivery timing are not reported because the saved evidence contains no selector timeline records. Tool-level and per-case runtime data remain available in the JSON dataset.

The per-case dataset intentionally excludes raw API logs, full transcripts, and hidden model reasoning.

<details>
<summary>中文翻译</summary>

# 结构化 Auto Memory 召回：60 条完整评测

本报告覆盖完整大样本运行：60 条相互独立的 case、120 次 A/B 任务执行、60 次答案成对评审和 120 次召回评审。A 为平铺记忆基线，B4 为树状结构加按需正文读取。

## 评测设置

- 仓库上下文：在每条 case 记录的代码版本上检出的完整 Qwen Code 仓库。
- 任务模型：`DeepSeek/deepseek-v4-flash`。
- 两组实验保持版本、问题、仓库上下文和记忆语料不变，仅改变召回链路。
- 每次任务执行和每个 judge 项目都使用全新的上下文。
- 答案评审采用盲测，并随机排列答案位置。导出的结果恢复了确定性的 A/B 位置映射。

## 结果

| 指标 | A 基线 | B4 结构化召回 | B4 - A |
| --- | ---: | ---: | ---: |
| 任务得分（0-10，37 条可评估 case） | 7.703 | 7.649 | -0.054 |
| 独占胜利数 | 10 | 8 | -2 |
| 召回质量（0-10，60 条 case） | 8.633 | 8.667 | +0.033 |
| Rescue 贡献数 | 21 | 30 | +9 |
| Helpful 贡献数 | 34 | 24 | -10 |
| Missed 贡献数 | 0 | 2 | +2 |
| P50 延迟 | 51.0 秒 | 39.1 秒 | -23.4% |
| P95 延迟 | 194.3 秒 | 189.1 秒 | -2.7% |
| 运行失败数 | 4/60 | 5/60 | +1 |
| 主链路总 Token | 24,355,312 | 18,860,402 | -22.6% |
| 主链路未缓存输入 Token | 3,217,774 | 2,825,328 | -12.2% |
| 后台记忆 Token | 38,148 | 373,040 | 334,892 |
| 包含后台任务的总 Token | 24,393,460 | 19,233,442 | -21.2% |
| 全部工具调用数 | 573 | 444 | -129 |
| 工具调用失败数 | 22 | 23 | 1 |

答案评审结果包括：17 条平局、2 条双方均失败、23 条不可评估，以及 0 次 judge 执行失败。

B4 减 A 的任务得分差值，其配对 95% 置信区间为 [-0.978, 0.870]；召回质量差值的配对 95% 置信区间为 [-0.406, 0.473]。两个区间都跨越 0。

## 工具行为

| 工具 | A 调用数 / 失败数 | B4 调用数 / 失败数 |
| --- | ---: | ---: |
| 全部工具 | 573 / 22 | 444 / 23 |
| `read_file` | 171 / 5 | 94 / 0 |
| `grep_search` | 145 / 4 | 99 / 1 |
| `run_shell_command` | 162 / 12 | 94 / 2 |
| `search_memory` | 0 / 0 | 97 / 18 |

B4 共进行了 73 次记忆 fetch 调用和 24 次记忆 search 调用。证据中保留了 18 次失败的 `search_memory` 调用。这些失败主要反映本次运行中观察到的工具协议误用，包括向 fetch 传递仅属于 search 的字段，以及在 search 中使用 cursor。

## 结果解读

在这个样本中，任务质量和召回质量实际上基本持平。B4 产生了更多 rescue 贡献和更多高分召回，但也出现了两次漏召回。最明显的实测收益是降低了中位延迟和总 Token 消耗。运行可靠性略有下降，比 A 多一次失败。

23 条不可评估的答案 case 被保留，而不是被丢弃。这些 case 依赖仓库状态、用户配置、Git remote 或题目中未提供的代码，隔离运行的答案 judge 无法独立验证这些信息。全部 60 条 case 的召回质量仍然可以评估。

由于保存的证据中没有 selector timeline 记录，本报告不提供 selector 层面的命中率、abort 率和 focused subtree 交付时序。工具层面和逐 case 的运行数据仍可在 JSON 数据集中查看。

逐 case 数据集有意排除了原始 API 日志、完整 transcript 和模型的隐藏推理过程。

</details>
