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
<summary>中文说明</summary>

# 结构化 Auto Memory 召回：60 条完整评测

本报告覆盖完整大样本运行：60 条相互独立的 case、120 次 A/B 任务执行、60 次答案成对评审和 120 次召回评审。A 为平铺记忆基线，B4 为树状结构加按需正文读取。

## 主要结果

- 任务质量：A 7.703，B4 7.649，基本持平。
- 召回质量：A 8.633，B4 8.667，基本持平。
- 延迟：B4 的 P50 从 51.0 秒降至 39.1 秒。
- Token：包含后台整理后，B4 总 Token 相比 A 变化 -21.2%。
- 稳定性：A 失败 4 条，B4 失败 5 条。
- 召回贡献：B4 的 rescue 从 21 增至 30，但出现 2 条 missed。

23 条答案任务不可评估，是因为隔离的 answer judge 缺少真实仓库状态、用户配置、Git remote 或题目引用的代码，不应猜测。它们仍完整保留在逐 case 数据中；全部 60 条的召回质量均可评估。

当前产物没有保存 selector timeline，因此不能据此报告 selector 命中率、abort 率和 focused subtree 的交付时序。

</details>
