# 01 · Agent 轨迹评测：三层模型与六个指标

> 来源：DeepEval 官方指南《AI Agent Evaluation Metrics》《Tracing AI Agents》、Promptfoo《Tracing》、NVIDIA NeMo Agentic Metrics（2026）
> 学习目标：看懂 think / tool_call / final_response 三段轨迹怎么评，直接对标"智能体搜索 SFT 标注员"岗位
> 与项目映射：穿搭导购的多步规划（天气→风格→商品→购物闭环）、客服的 6 路路由 + 工具调用

---

## 一、为什么 Agent 不能只评最终答案

普通问答只看 final_response 就够，但 Agent 的失败方式完全不同：

- 工具选对了，**参数传错了**；
- 计划做得很漂亮，**执行时没按计划走**；
- 任务最终完成了，但**多绕了五步、浪费了 token 和时间**。

答案只告诉你"到没到终点"，轨迹（trajectory / trace）告诉你"走的是哪条路、在哪一步摔的"。所以行业共识：**Agent 评测必须记录完整有序轨迹——每次模型调用、每次工具调用、工具返回、下一步决策，按顺序留痕。**

## 二、三层模型（必背框架）

DeepEval 把 Agent 拆成三层，每层失败模式不同、评测粒度不同：

| 层 | 干什么 | 评测范围 | 核心指标 |
| --- | --- | --- | --- |
| **推理层 Reasoning** | 理解任务、做计划、定策略 | 整条轨迹 | PlanQuality（计划质量）、PlanAdherence（计划遵循） |
| **动作层 Action** | 选工具、生成参数、执行调用 | 单个 span（组件级） | ToolCorrectness（工具正确性）、ArgumentCorrectness（参数正确性） |
| **执行层 Execution** | 跑完整循环、达成目标 | 整条轨迹 | TaskCompletion（任务完成度）、StepEfficiency（步骤效率） |

记忆法：**想得对（推理）→ 做得对（动作）→ 做成了且不啰嗦（执行）**。

## 三、六个指标逐个理解

### 推理层（轨迹级）
1. **PlanQualityMetric 计划质量**：计划本身是否逻辑合理、完整、高效。算法是"任务↔计划"的对齐打分。注意：轨迹里检测不到显式计划时默认给满分——没有计划就不罚，但也意味着该指标只在 Agent 显式规划时有用。
2. **PlanAdherenceMetric 计划遵循**：实际执行步骤有没有偏离自己定下的计划。计划好却不执行 = 白搭，必须和 PlanQuality 成对使用。

### 动作层（组件级，挂在做工具决策的那个 span 上）
3. **ToolCorrectnessMetric 工具正确性** = 正确使用的工具数 ÷ 调用的工具总数。严格度分四级可调：仅工具名匹配 → 加参数匹配 → 加返回值匹配 → 加调用顺序匹配。
4. **ArgumentCorrectnessMetric 参数正确性**：工具选对了，参数对不对（城市名、日期、订单号）。

### 执行层（轨迹级）
5. **TaskCompletionMetric 任务完成度**：看完整步骤与结果，判定目标是否真正达成——不是"模型说完成了"，而是证据表明完成了。
6. **StepEfficiencyMetric 步骤效率**：这些步骤是否必要、有没有冗余调用和重复试错。任务完成但绕路，此项低分。

> 设计精髓：**打分与归因解耦**。TaskCompletion 回答"好不好"，ToolCorrectness/错误类型回答"错在哪、该修哪"。

## 四、轨迹判分的通用四档

行业标注常用：`correct` / `partially_correct` / `incorrect` / `unnecessary`（正确 / 部分正确 / 错误 / 多余步骤）。

错误类型枚举（归因标签，与分数分开记）：

```
reasoning  : logical_error | factual_error | planning_error   逻辑错/事实错/规划错
execution  : wrong_tool | wrong_args | api_error              选错工具/参数错/接口报错
termination: premature_stop | redundant_step                 提前终止/冗余步骤
```

边界样例：工具调用失败但模型正确降级并向用户说明 → 判 `partially_correct`，**不惩罚合理的失败恢复**。

## 五、映射到我的项目

- **客服项目**：6 路分类器路由 = ToolCorrectness 的简化版（路由对不对、要不要订单号）；"退款多久到账 vs 我要退款"就是动作层判据；拒答转人工 = 失败恢复的 `partially_correct` 逻辑。
- **穿搭项目**：天气模块→风格识别→商品检索→搭配生成→购物闭环，是天然多步轨迹，可以用三层模型复盘：风格误判属 reasoning（planning_error），商品检索召回错属 execution 的检索 span，最终推荐不合适是 TaskCompletion 低但步骤效率可能正常。

## 六、自测三问（盖住答案口述）

1. 为什么 Agent 评测不能只看 final_response？
2. 三层模型各层的指标是什么？哪两个必须成对用？
3. ToolCorrectness 的四级严格度是什么？

---
来源：
- https://deepeval.com/guides/guides-ai-agent-evaluation-metrics
- https://deepeval.com/guides/guides-tracing-ai-agents
- https://www.promptfoo.dev/docs/tracing/
- https://docs.nvidia.com/nemo-platform/documentation/evaluate-models/metrics/agentic-metrics
