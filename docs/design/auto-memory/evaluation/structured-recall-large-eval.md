# Structured Auto Memory Recall: Large-Sample Evaluation

This report contains only completed validated entries from the large-sample run. An entry is included only when the paired A/B run is available, both A and B4 finished successfully, answer judging passed, and recall judging passed for both variants.

## Evaluation Setup

- Repository context: full Qwen Code repository checkout.
- Memory corpus: copied project/user memory set used by the large-sample run.
- Variants: A is the main-branch baseline memory path; B4 is the structured recall path.
- Task model: DeepSeek/deepseek-v4-flash for the recorded large-sample run.
- Judge model: the same judge pipeline used for the run, evaluating answer quality pairwise and recall quality per variant.
- Included entries: 36.

## Metrics

- Recall quality: LLM-as-judge score for whether the memory path surfaced useful, relevant memory for the task. Higher is better.
- Contribution: judge label for memory impact: rescue, helpful, neutral, harmful, or missed.
- Memory need: judge label for how much memory evidence was needed: metadata or body.
- Latency: end-to-end task runtime from the headless result event. Lower is better.
- Token usage: API usage from the final result event, including input, output, total, and cache-read input tokens when reported by the provider. Lower is better for comparable answer quality.

## Summary

| Metric                      | A baseline | B4 structured recall |        Delta B4 - A |
| --------------------------- | ---------: | -------------------: | ------------------: |
| Completed validated entries |         36 |                   36 |                   0 |
| Recall quality avg          |       8.47 |                 9.14 |               +0.67 |
| Rescue contributions        |         10 |                   17 |                   7 |
| Helpful contributions       |         23 |                   19 |                  -4 |
| Neutral contributions       |          3 |                    0 |                  -3 |
| Harmful contributions       |          0 |                    0 |                   0 |
| P50 latency                 |  48,687 ms |            34,022 ms | -14,665 ms (-30.1%) |
| P95 latency                 | 191,893 ms |           276,311 ms |           84,418 ms |
| Avg total tokens / entry    |    399,413 |              359,922 |     -39,490 (-9.9%) |
| Avg input tokens / entry    |    392,886 |              354,096 |     -38,790 (-9.9%) |
| Avg output tokens / entry   |      6,527 |                5,827 |                -700 |
| Total tokens                | 14,378,866 |           12,957,209 |          -1,421,657 |

## Result Interpretation

On the completed validated entries, B4 improves average recall quality and increases rescue-labeled memory contributions, while reducing median latency and average token usage. The result supports the direction of replacing flat forced memory injection with structured router/subtree metadata and on-demand body access for this workload.

The result should be read as a recall-path evaluation rather than a full product-quality benchmark. The case set intentionally focuses on memory-sensitive tasks in a full repository checkout, and the judge evaluates both memory relevance and answer quality from the recorded run artifacts.

The per-case data is available in [structured-recall-large-eval-cases.json](./structured-recall-large-eval-cases.json). It contains only case metadata, judge summaries, latency, and token usage; raw API logs, model reasoning, and full transcripts are intentionally not included.

<details>
<summary>中文说明</summary>

# 结构化 Auto Memory 召回：大样本评测

这份报告只包含大样本运行中已经完成完整验证的条目。只有当成对 A/B 运行可用、A 和 B4 都成功结束、答案评审通过、并且 A 与 B4 的召回评审都通过时，该条目才会被纳入。

## 评测设置

- 仓库上下文：完整的 Qwen Code 仓库检出。
- 记忆语料：大样本运行使用的 project/user 记忆副本。
- 对比版本：A 是 main 分支基线记忆链路；B4 是结构化召回链路。
- 任务模型：本次大样本记录使用 DeepSeek/deepseek-v4-flash。
- 评审模型：使用本次运行对应的同一套 judge pipeline，对答案质量做成对比较，并对每个版本单独评估召回质量。
- 纳入条目：36 条。

## 指标含义

- 召回质量：LLM-as-judge 给出的评分，用来判断记忆链路是否为任务提供了有用且相关的记忆。越高越好。
- 贡献类型：judge 对记忆影响的标签，包括 rescue、helpful、neutral、harmful 或 missed。
- 记忆需求：judge 对任务所需记忆证据粒度的判断，包括 metadata 或 body。
- 延迟：headless 最终 result 事件记录的端到端任务运行时间。越低越好。
- Token 消耗：最终 result 事件中的 API usage，包括输入、输出、总 token，以及 provider 返回时的 cache-read input tokens。在答案质量可比时越低越好。

## 汇总结果

| 指标               |     A 基线 | B4 结构化召回 |         B4 - A 差值 |
| ------------------ | ---------: | ------------: | ------------------: |
| 完整验证条目       |         36 |            36 |                   0 |
| 平均召回质量       |       8.47 |          9.14 |               +0.67 |
| rescue 贡献数      |         10 |            17 |                   7 |
| helpful 贡献数     |         23 |            19 |                  -4 |
| neutral 贡献数     |          3 |             0 |                  -3 |
| harmful 贡献数     |          0 |             0 |                   0 |
| P50 延迟           |  48,687 ms |     34,022 ms | -14,665 ms (-30.1%) |
| P95 延迟           | 191,893 ms |    276,311 ms |           84,418 ms |
| 每条平均总 token   |    399,413 |       359,922 |     -39,490 (-9.9%) |
| 每条平均输入 token |    392,886 |       354,096 |     -38,790 (-9.9%) |
| 每条平均输出 token |      6,527 |         5,827 |                -700 |
| 总 token           | 14,378,866 |    12,957,209 |          -1,421,657 |

## 结果解读

在这些完整验证条目上，B4 提高了平均召回质量，增加了被标为 rescue 的记忆贡献，同时降低了中位延迟和平均 token 消耗。这个结果支持当前方向：用结构化的 router/subtree metadata 和按需正文访问，替代平铺式强制记忆注入。

这个结果应被理解为召回链路评测，而不是完整产品质量基准。评测集有意聚焦于完整仓库检出中的记忆敏感任务，judge 会根据记录下来的运行产物同时评估记忆相关性和答案质量。

逐条数据见 [structured-recall-large-eval-cases.json](./structured-recall-large-eval-cases.json)。其中只包含 case metadata、judge 摘要、延迟和 token 使用量；原始 API 日志、模型推理过程和完整 transcript 均未上传。

</details>
