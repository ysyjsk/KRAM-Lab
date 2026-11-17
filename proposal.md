# Research Proposal

# 面向 Linux Kernel Repair Agent 的任务内记忆管理与反馈驱动决策优化

**Intra-Task Memory Management and Feedback-Driven Decision Optimization for Linux Kernel Repair Agents**

- **研究平台：** Live-kBench（ICML 2026）
- **应用场景：** Linux Kernel Crash Diagnosis and Repair
- **研究方向：** Agent Memory × Agentic Systems × Feedback Learning / RL
- **研究阶段：** 论文与源码学习 → 基准复现 → 真实轨迹分析 → 受控记忆实验 → 学习式策略
- **项目性质：** 独立工程项目与探索性科研相结合
- **Proposal 版本：** V1，2026 年 11 月

---

## 1. 研究摘要（Abstract）

长期运行的 Linux Kernel Repair Agent 需要持续进行代码探索、故障诊断、补丁修改和内核验证。在这一过程中，Agent 会积累大量来自源码、工具调用、Crash Report、编译日志与 Crash Resolution Feedback（CRF）的信息。随着修复过程推进，早期诊断证据、失败尝试以及已确认的状态可能需要在后续决策中重新使用。

现有 Live-kBench 提供了标准化的内核修复环境与真实执行反馈，但尚未专门研究 Agent 在单次修复任务中如何管理和利用诊断历史。

本研究首先基于 Live-kBench 已有的真实 Agent 轨迹，分析历史证据的保留情况、重复访问行为和后续决策之间的关系。随后，在固定 Agent、内核工作区和执行预算的条件下，比较完整历史、普通复查、历史重新呈现以及摘要检索等策略对后续修复行为的影响。

在验证记忆干预确有独立价值之后，进一步研究能否根据 Agent 当前任务状态和执行反馈，学习何时检索、呈现或跳过历史信息，改善修复质量与执行成本之间的权衡。

研究以证据驱动、分阶段验证为原则，不预设上下文遗忘、记忆收益或强化学习收益已经成立。

## 2. 研究背景与核心动机

### 2.1 Linux Kernel Repair 为什么是有价值的 Agent 研究场景？

相比普通软件修复，Linux 内核崩溃修复通常涉及：

- 大规模 C 语言代码库及复杂的子系统依赖。
- Crash Stack Trace、Reproducer、内核配置和历史代码状态。
- 需要理解并发、对象生命周期、引用计数、锁和资源释放。
- 补丁正确性依赖编译、虚拟机运行和 Crash 复现。
- 一次验证可能耗费较长时间，并且 Crash 复现具有非确定性。

因此，这类任务不仅要求模型具备代码生成能力，还要求 Agent 在多轮交互中维护并正确使用诊断证据。

Live-kBench 是适合研究此类问题的平台，因为它提供真实内核 Bug、标准化修复环境和可以记录的执行轨迹。

### 2.2 Live-kBench 已经解决了哪些问题？

根据原论文，Live-kBench 主要解决三个问题：

1. **Benchmark Freshness：** 持续从 Syzbot 收集内核 Bug，降低静态数据集带来的污染风险。
2. **Standardized Execution：** 使用 kEnv 为不同 Agent 提供统一的内核源码、任务输入和执行接口。
3. **Execution Feedback：** 通过 kGym 和 CRF 支持补丁编译、Crash 复现及最终评估。

其主要组件包括：

- **kArena：** Bug 数据、任务调度与结果管理。
- **kEnv：** Agent 运行容器及统一协议。
- **kGym：** 内核编译、虚拟机运行和 Crash 复现。
- **Agent Scaffold：** mini-SWE-agent、SWE-agent、OpenHands 等。

这些组件将被直接复用，不作为本研究的原创成果。

### 2.3 原论文留下了什么空间？

Live-kBench 关注最终 Patch 的生成和评测，以及 CRF 是否能够提高修复成功率。

对于单次修复任务中的历史信息，仍然可以进一步研究：

- 早期诊断证据在后续模型请求中是否保持可访问。
- Agent 是否在需要时重新参考已有证据。
- 重复读取、重复诊断与历史利用是否存在可解释的联系。
- 通过主动呈现历史证据，是否可以改善下一步决策。
- 在额外 Token、模型推理和工具执行成本下，Memory 是否值得使用。

这些是待验证的问题，并不意味着原论文存在已经证实的记忆缺陷。

## 3. 现有真实数据与事实边界

### 3.1 当前可用的数据

根据此前整理的实验记录，现阶段具有以下数据基础：

| 项目 | 当前记录 |
|---|---|
| Agent Attempts | 20 |
| Unique Bugs | 9 |
| 模型/工具交互统计 | 985 条相关调用统计，需复核具体字段口径 |
| Context Records | 996 |
| CRF 调用 | 34 |
| 最终评测记录 | 9 |
| Provider Prompt Tokens | 约 3317 万 |
| 历史上下文重复比例 | 约 97.62% |
| 已检测 Context Reduction | 0/976 |

这些数据来自此前的轨迹统计，本 proposal 不将其视为本轮重新运行的实验结果。

同时，需注意现有记录中对于 985 条调用的命名存在过不同口径。下一阶段必须重新核对 Model Call、Tool Call 和 Message 三类计数，避免将不同统计对象混用。

### 3.2 当前已经观察到的事实

**事实 F1：** 历史上下文存在大量重复携带，整体上下文开销较高。

**事实 F2：** 已检查记录中没有检测到明确的原生 Context Reduction，不能据此认定 Agent 因上下文截断丢失早期证据。

**事实 F3：** 部分轨迹存在重复读取或重复操作，但尚未建立它们与 Memory 失效的因果关系。

**事实 F4：** 已有多轮 CRF 与候选补丁变化，能够用于研究执行反馈和后续修复决策。

**事实 F5：** 当前最终评测尚不足以计算可靠的整体 CRR/EPR，不能将中途未复现 Crash 的 CRF 结果直接当成最终成功。

### 3.3 重点分析案例

以此前分析的 Bluetooth L2CAP 案例 `027` 为例：

- 具有较长的 Agent 执行轨迹。
- 包含 142 次模型调用。
- 包含 6 次 CRF。
- 单次请求的历史上下文峰值约为 81,684 tokens。
- 已有证据表明早期诊断信息仍保留在可见历史中。
- 最终任务状态为 aborted/error，而非已确认的成功修复。

该案例适合作为历史证据分析与决策检查点实验的起点。

已有候选检查点 `m147` 位于候选补丁形成之后、后续反馈出现之前，可以用于研究 Agent 在获得后续结果之前如何使用历史诊断信息。

**研究约束：** 不允许将该案例直接解释为已经发生 Memory Loss，也不能把后续失败结果提前注入检查点。

## 4. 研究目标与核心问题

### 总体目标

建立一个基于 Live-kBench 的可复现 Agent 执行和记忆实验平台，研究任务内诊断历史在后续修复决策中的作用，并探索低成本的反馈驱动记忆使用策略。

### RQ1：历史诊断信息是否被有效利用？

在完整历史仍可访问的情况下，Agent 后续决策是否正确利用了此前观察到的关键诊断证据？

需要明确区分：

- 证据不存在。
- 证据被上下文处理机制删除。
- 证据仍存在，但未被主动检索。
- 证据被检索或提及，但未正确使用。
- 证据已被新的执行结果推翻，因而不应继续使用。

RQ1 主要是描述性分析，不直接做因果结论。

### RQ2：Memory Re-activation 是否具有独立收益？

对于同一个修复检查点，主动重新呈现已有诊断证据，是否比直接使用完整历史或者进行普通代码复查更有效？

这是第一阶段的核心因果问题。

重点判断：

1. 原有历史是否已经足够。
2. Memory 提醒是否真正改变后续行为。
3. 改善是否来自额外的信息处理预算，而非特殊的 Memory 表示。
4. 重新呈现过时或错误证据是否导致负面影响。

### RQ3：不同 Memory 表示的质量和成本有何差异？

在证实历史重新呈现有价值之后，比较：

- 原始证据片段。
- 普通摘要。
- 带来源引用的结构化摘要。
- 可检索的历史诊断记录。

重点考察信息完整性、证据准确性、额外推理成本以及后续行为差异。

### RQ4：反馈学习能否改善 Memory 使用策略？

若 RQ2/RQ3 已观察到不同策略的稳定收益差异，进一步研究 Agent 能否根据当前诊断阶段、历史内容和剩余预算，学习何时调用 Memory、呈现何种历史证据，或者选择不干预。

RQ4 为后续扩展，不作为第一阶段必须成立的研究命题。

## 5. 研究假设

### H1：已有证据的重新呈现可能影响后续修复决策

在部分决策点，虽然关键证据已经出现在可见历史中，但将其重新呈现可能改变 Agent 的下一步诊断选择。

该假设需要通过同状态受控继续执行来验证。

### H2：Memory 的收益依赖于信息类型与当前修复阶段

代码探索、根因假设、补丁实现和验证阶段可能需要不同的历史信息。

例如，补丁验证阶段可能更依赖此前已确认的 CRF 结果，而非早期的一般代码搜索记录。

这只是候选假设，必须通过阶段标注和具体轨迹检验。

### H3：选择性 Memory 可能优于持续注入

在部分情况下，主动检索与提醒有价值；在另外一些情况下，额外历史信息可能造成干扰或浪费。

因此允许 Memory Policy 选择“不干预”。

### H4：学习式 Memory Policy 可能改善质量—成本权衡

如果存在能够预测 Memory 收益的状态特征，并且不同动作的实际结果具有可学习差异，则反馈学习可能比固定规则更有效。

如果前述条件不成立，则不应继续投入 RL 训练。

## 6. 系统设计

### 6.1 总体架构

系统采取独立扩展方式，不修改 Live-kBench 官方评测器。

主执行链路：

`Syzbot Task → kArena → kEnv → Coding Agent → kGym/CRF → Patch Evaluation`

新增模块：

`Trajectory Collector → Intra-Task Memory Store → Memory Intervention → Checkpoint Evaluation → Feedback Learner`

主要组成如下。

**A. Trajectory Collector**

记录模型请求、工具调用、工具反馈、代码修改和执行状态，保留每条记录的来源。

**B. Intra-Task Memory Store**

仅保存当前修复任务内部的历史，不混入其他 Bug 的诊断经验。

**C. Memory Intervention Adapter**

在 Agent 的下一次模型调用之前，根据预先定义的策略选择是否呈现历史证据。

**D. Checkpoint Runner**

从固定的工作区状态继续执行不同对照分支，记录后续行为和任务结果。

**E. Evaluation and Analysis**

整理 CRR、EPR、执行成本、历史利用情况和失败案例。

**F. Feedback Learner（可选）**

在具有真实干预数据之后，训练轻量 Memory Policy。

### 6.2 Memory 数据结构

建议初期使用保留来源引用的简化记录：

```json
{
  "memory_id": "case027-m001",
  "task_id": "027",
  "episode_id": "attempt-01",
  "kind": "observation",
  "content": "原始诊断证据摘录",
  "source_message_ids": ["m041", "m042"],
  "status": "observed",
  "git_state_hash": "<state-id>",
  "created_at_step": 42,
  "last_verified_step": 42
}
```

其中 `kind` 可取：

- `observation`：实际工具观察。
- `hypothesis`：尚未验证的诊断假设。
- `attempt`：已执行的操作与结果。
- `verification`：编译、测试或 CRF 结果。
- `open_issue`：尚未完成的诊断事项。

`status` 必须能够区分已观察事实、未验证假设和已被推翻的结论。

第一版优先保留原始证据引用，不要求 LLM 自动抽象完整根因。

### 6.3 不主动修改 Solver

初期固定：

- 原始 Coding Agent 模型。
- Agent System Prompt。
- 原始工具集合。
- Kernel Git Commit。
- Crash Report 和 Reproducer。
- 内核配置及评测协议。

Memory 作为旁路组件介入下一次决策。

这样能尽量将观察到的行为变化归因于历史信息的呈现方式，而不是基础模型或执行环境变化。

## 7. 研究方法与实验设计

### E0：真实轨迹审计

**目标：** 确定是否存在值得进一步研究的历史使用问题。

步骤：

1. 解析现有 Agent 轨迹与工具反馈。
2. 统计当前调用能访问哪些历史证据。
3. 标记代码读取、诊断、修改和 CRF 行为。
4. 检查重复动作的上下文。
5. 记录候选的历史证据复用失败现象。
6. 人工复核关键案例，排除正常复查和新状态变化。

输出：

- 统一 Trajectory Schema。
- 行为统计报告。
- 来源可追溯的诊断事件。
- 经人工复核的候选决策检查点。

不能仅凭重复命令数量推断 Memory Failure。

### E1：同状态决策检查点对照

**目标：** 验证历史信息的重新呈现是否具有独立因果作用。

初期选用已规划的 7 个候选检查点，核对后保留满足环境恢复要求的样本。

设置四个主要实验组：

| 组别 | 具体设置 |
|---|---|
| A0：Full History | 保持原有完整历史，不额外插入 Memory |
| A1：Ordinary Review | 允许 Agent 使用相同预算重新检查当前代码与反馈 |
| A2：History Re-presentation | 自动检索本次任务已有的相关原始证据并重新呈现 |
| A3：Summary + Retrieval | 提供可追溯的简短历史摘要及原始证据引用 |

其中 A1 是重要的强对照。

如果 A1 与 A2 效果相同，就不能把收益直接归因于特殊的记忆机制。

条件允许时，可以增加等长度但无决策相关性的安慰剂文本对照，用于衡量“额外提醒”本身的影响。

**控制条件：**

- 同一检查点。
- 同一历史前缀。
- 同一工作区代码状态。
- 同一模型、工具和解码配置。
- 同等后续推理及工具预算。
- 所有注入信息只能来自检查点之前。
- 不向 Solver 提供 Developer Fix 或未来 CRF 结果。

同一条件至少进行若干次重复采样，记录运行间差异。

### E2：Memory 内容与策略比较

只有 E1 显示存在稳定的历史信息干预空间，才进入 E2。

主要比较：

1. 原始历史片段直接呈现。
2. 普通任务进度摘要。
3. 来源关联的结构化诊断记忆。
4. 按当前任务状态检索记忆。
5. 选择性介入与始终介入。

初期不同时改写检索器、摘要器、触发器和 Agent Prompt，保证每轮实验主要只有一个可解释的变化因素。

### E3：完整任务运行

在少量真实 Linux Kernel Bug 上，比较：

- 原始 Live-kBench Agent。
- 普通复查策略。
- 受控 Memory Intervention。
- E2 阶段最优的可复现 Memory 方法。

每组都从相同初始内核状态启动独立任务，不能直接把已有轨迹回放结果当作新的完整 Agent Run。

最终使用官方评测流程计算修复指标。

### E4：反馈驱动策略学习（条件性阶段）

只有满足以下条件才进入：

- 存在真实、可重复的 Memory 干预收益或损失。
- 已收集到足够的不同状态和动作反馈。
- 能够在训练与评测任务之间建立有效隔离。
- 学习策略的额外推理成本处于可接受范围。

第一版不训练完整 Kernel Repair LLM，而是训练独立 Memory Policy。

候选动作：

`NONE / ORDINARY_REVIEW / RETRIEVE_RAW / RETRIEVE_SUMMARY`

输入状态包括当前阶段、最近观察、历史信息索引、上下文压力和剩余预算。

奖励优先由实际后续任务结果及成本计算，而不是奖励“使用了 Memory”这一行为本身。

如果数据不足，则只实现启发式或 Contextual Bandit 版本；不强行运行 GRPO。

## 8. 关键实验控制：恢复真实 Agent 状态

对照实验的可信度高度依赖环境恢复。

一个检查点至少应当包括：

```text
Checkpoint
├── bug_id
├── base_kernel_commit
├── current_git_diff
├── workspace_state
├── visible_context_prefix
├── agent_harness_config
├── model_config
├── tool_configuration
├── crash_report
├── reproducer
├── kernel_configuration
├── pending_tool_state
└── experiment_budget
```

仅恢复 `traj.json` 中的文本历史，不等于恢复真实 Linux 执行环境。

必须处理：

- 当前 Git Diff 与未提交文件。
- 中间生成的辅助文件。
- 后台进程及异步工具调用。
- Kernel Build/CRF 的待完成任务。
- 相同模型的配置和调用预算。

如果某检查点无法可靠恢复工作区，则只能用于离线下一步决策分析，不能将其作为完整端到端反事实修复结果。

## 9. 评估指标

### 9.1 最终修复质量

**CRR：** Crash Resolution Rate，采用官方终态评估结果。

**EPR：** Equivalent Patch Rate，使用官方补丁等价性评估口径。

**Localization IoU：** 比较修改文件和函数的定位重叠程度。

需注意，CRF 的 `notReproduced` 不能直接替代最终 CRR；EPR 也不是对补丁语义正确性的形式化证明。

### 9.2 Agent 行为指标

- 无效或完全重复的工具操作比例。
- 历史已知事实被重新发现的次数。
- 对已否定诊断假设的重复尝试。
- Memory 注入后后续动作的变化。
- 诊断证据在后续决策中的实际引用与使用。
- 错误历史提醒造成的 Negative Transfer。

此类指标必须定义清楚标注标准，保留对应原始 Trace。

### 9.3 Memory 质量指标

- Evidence Retrieval Recall / Precision。
- Memory 引用来源正确率。
- 已失效证据被错误使用的比例。
- Memory 呈现对下一步行为的实际影响。
- Memory 构建和检索耗时。

不把“检索到很多历史”作为成功标准。

### 9.4 资源与效率指标

- 真实 Provider Prompt / Completion Tokens。
- 缓存命中相关信息及可计费 Token。
- Memory 额外模型调用次数。
- Tool Calls / CRF Calls。
- 内核编译与 Crash 复现耗时。
- End-to-End Wall Time。
- 按实际定价计算的模型及算力成本。

原始 Prompt Tokens 高，不代表能够按相同比例节约实际费用；缓存、Prefix 复用和 Memory 额外开销都应计入。

## 10. 统计与结果分析

初期样本数量较少，研究应首先采用配对实验和案例级分析。

- 以相同检查点比较不同处理组。
- 尽可能固定模型、环境与预算。
- 对随机生成结果进行重复运行。
- 将同一 Bug 的多个检查点视为相关观测。
- 汇报各条件的结果分布，不只选择最好的样本。
- 在样本扩大后，使用按 Bug 聚类的 Bootstrap 或适当的配对分析。

不得将 7 个相关检查点直接解释为 7 个独立的 Linux Bug。

若观察到收益，应同时报告没有改善和发生退化的案例。

## 11. RL 扩展方案

### 11.1 为什么不直接训练整个 Repair Agent？

完整 Kernel Repair RL 需要大量交互式 Rollout，每次 Rollout 又可能调用内核编译与虚拟机测试。

因此相比数学答案式 RL，训练成本主要受到真实执行环境约束。

在缺少足够训练轨迹和可靠 Reward 的情况下，直接进行模型级 GRPO 并不经济。

### 11.2 先使用轻量反馈学习

如果 E1/E2 已经获得状态—动作—结果数据，可以先使用：

- 静态规则策略。
- 基于历史收益的简单动作估值。
- Contextual Bandit。
- 小型监督策略。

比较它们能否在不同检查点选择更合适的 Memory 行为。

### 11.3 再考虑参数化 RL

必要时使用小型语言模型或轻量控制器作为 Memory Policy。

完整 Agent 保持冻结，仅训练是否使用 Memory、使用哪类 Memory 的决策。

训练时需要解决：

- 反馈稀疏。
- 动作间 Reward 差异较小。
- 同组采样奖励零方差。
- 不同决策点难度不一致。
- 训练期间的评测信息泄漏。

若简单策略已经足够，则可以停止在反馈式学习阶段，不强行增加 GRPO。

## 12. 项目资源与可行性

### P0：论文与源码学习

资源：

- Linux 开发环境。
- Python、Git、Docker。
- 官方 Live-kBench 代码和文档。

无额外 GPU 需求。

### P1：已有轨迹分析

使用已保存的 Agent Trajectory、CRF 日志和评测记录。

主要消耗 CPU、系统内存与存储空间。

不需要重新执行全部 Linux Crash 修复任务。

### P2：检查点继续执行

需要：

- 可恢复的 Kernel 工作区。
- Agent 模型 API 或等效本地推理服务。
- kGym 运行环境。
- 内核编译和虚拟化资源。

这是主要成本阶段。

初期应限制在少量检查点、少量处理组及固定工具调用预算。

### P3：轻量学习策略

规则策略和简单 Bandit 可以在 CPU 上开发。

如果训练 0.5B–1.5B Memory Policy，可以尝试单张 RTX 3090 Ti（24GB），必要时使用两张卡；但完整长程 Kernel Repair LLM 的 RL 训练不列为本 proposal 的必需工作。

## 13. 分阶段实施计划

| 阶段 | 主要任务 | 必须交付 |
|---|---|---|
| M0：基础学习 | Live-kBench 论文、Linux Crash、Agent Loop | 学习笔记、论文—源码映射 |
| M1：基准复现 | 固定环境运行至少一条修复任务 | 配置、日志、Patch、结果 |
| M2：轨迹审计 | 分析已有任务历史与行为 | Trajectory Analyzer、证据表 |
| M3：受控实验 | 同状态比较普通复查与 Memory | Checkpoint Runner、配对结果 |
| M4：Memory 模块 | 实现可追溯记忆与选择性介入 | 独立代码模块、测试、消融 |
| M5：反馈学习 | 满足进入条件后实现学习式策略 | 训练/评测记录、对照结果 |
| M6：成果交付 | 整理开源项目、技术报告与简历 | README、实验报告、代码仓库 |

不要求所有阶段在固定时间内完成。

建议以 M0–M3 作为第一阶段，先形成可信的工程和实验基础。

## 14. 研究风险与替代路径

### 风险一：真实轨迹中没有可确认的 Memory 失效

处理方式：如实报告现有证据可以持续访问，不预设遗忘问题。

继续分析重复诊断、状态可见性和普通复查行为，而不是构造人为场景证明 Memory 一定必要。

### 风险二：Memory 干预没有优于普通复查

处理方式：保留负结果，分析额外信息预算是否已经解释收益。

项目仍可产出可复现 Checkpoint 评测平台，不强行升级成新算法。

### 风险三：Checkpoint 环境无法恢复

处理方式：先完成轨迹解析与离线下一步决策评测；优先在新运行的任务中增加工作区状态采集。

不能将文本回放当成真实环境复现。

### 风险四：样本数量不足

处理方式：继续收集符合筛选规则的真实长轨迹和检查点，不通过复制模板制造独立样本数量。

### 风险五：RL 训练收益不稳定

处理方式：比较规则、监督学习和 Bandit，检查动作空间、奖励与训练数据；必要时终止参数化 RL 阶段。

## 15. 预期成果与贡献边界

### 工程成果

1. 基于 Live-kBench 的 Kernel Repair Agent 轨迹分析模块。
2. 支持内核工作区恢复和受控继续执行的检查点实验工具。
3. 可插拔的任务内诊断 Memory 模块。
4. 修复质量与系统成本的统一评测程序。
5. 如条件允许，独立的反馈学习策略模块。

这些模块要与官方 Live-kBench 的原有实现明确区分。

### 研究成果

**现阶段可合理提出的研究目标：**

- 对真实 Linux Kernel Repair Agent 的任务内诊断历史利用进行实证分析。
- 检验历史重新呈现相对于普通复查的实际作用。
- 在真实结果支持的前提下探索状态相关的 Memory 使用策略。

**现阶段不能提前声明的创新：**

- 已经发现 Linux Repair Agent 普遍发生长期记忆遗忘。
- 提出能够显著提升 CRR/EPR 的新 Memory 算法。
- 提出新的 GRPO 或 Credit Assignment 理论。
- 在跨任务经验复用方面取得了新进展。

是否存在论文级创新，需要通过后续数据和近邻工作的严格比较确定。

## 16. 求职项目交付标准

项目至少需要达到：

- 能运行 Live-kBench 的真实任务。
- 能解释 kArena、kEnv、kGym 与 Agent 的调用关系。
- 自己开发一个具有独立功能的模块。
- 有可靠的回归测试与实验结果。
- 能清楚说明原始开源框架和个人贡献边界。
- 能展示真实的资源开销及错误排查过程。

达到这一标准之后，就有条件作为独立工程项目写入简历，而不必依赖论文发表或 RL 训练一定成功。

**候选简历项目名称：**

*面向 Linux Kernel Repair 的 Agent 诊断记忆与执行评测系统*

若 M5 阶段取得可靠结果，可调整为：

*基于反馈学习的 Linux Kernel Repair Agent 任务内记忆管理与优化*

## 17. 学习材料与技术依赖

| 学习顺序 | 资料 | 对应任务 |
|---|---|---|
| 1 | Live-kBench，ICML 2026 | 理解内核修复 Benchmark 与评价指标 |
| 2 | Live-kBench 官方 kEnv 文档 | 理解 Agent 执行环境和工具接口 |
| 3 | Hugging Face Agents Course Unit 1 | 理解 Agent Loop 与 Tool Calling |
| 4 | Linux Kernel Documentation / Syzkaller | 理解 Crash、Reproducer、Kernel Build |
| 5 | Remember When It Matters，2026 | 理解任务内主动记忆与介入 |
| 6 | PMCoder，2026 | 理解软件修复过程中的状态与记忆 |
| 7 | Compact-Bench / CompactionBench，2026 | 学习检查点实验和上下文管理评测 |
| 8 | MemRL / H-EPM | 学习反馈驱动的经验价值和记忆选择 |
| 9 | Hugging Face Deep RL Course / TRL | 在需要时学习 Policy、Reward 与 GRPO |

### 核心资料链接

1. Live-kBench: https://arxiv.org/abs/2602.02690
2. Live-kBench Code: https://github.com/ARiSE-Lab/live-kbench
3. kEnv Documentation: https://github.com/ARiSE-Lab/live-kbench/blob/oss-releases/docs/kEnv.md
4. Hugging Face Agents Course: https://huggingface.co/learn/agents-course/en/unit1/introduction
5. Remember When It Matters: https://arxiv.org/abs/2607.08716
6. PMCoder: https://arxiv.org/abs/2608.06811
7. Compact-Bench: https://arxiv.org/abs/2610.31710
8. CompactionBench: https://www.preprints.org/manuscript/202611.0780
9. MemRL: https://arxiv.org/abs/2601.03192
10. H-EPM: https://proceedings.mlr.press/v306/li26bn.html
11. CAMG-RL: https://arxiv.org/abs/2609.34422
12. CAMP-II: https://arxiv.org/abs/2611.14009
13. TRL GRPOTrainer: https://huggingface.co/docs/trl/grpo_trainer

## 18. 最终研究原则

本项目遵循以下原则：

**第一，固定研究场景。** 研究 Live-kBench 的 Intra-Task Agent Memory，不主动转向跨任务经验迁移。

**第二，以真实轨迹为依据。** 先确认问题，再设计 Memory 机制，不用虚构的失败故事代替 Motivation。

**第三，区分完整历史与有效使用。** 历史存在、历史被读取、历史真正影响决策是不同层次。

**第四，建立强对照。** Memory 必须与普通复查、完整历史及等预算方法进行比较。

**第五，先工程、后学习。** 先跑通 Agent 和 Memory 实验，再决定是否引入反馈学习与 GRPO。

**第六，不预设正结果。** 无论 Memory 或 RL 是否有效，都完整记录实验事实及其适用边界。

**第七，科研与求职成果分离验收。** 工程功能、实验可复现性与技术贡献可以独立成立；学术创新必须另行证明。

---

**一句话概括：**

以 Live-kBench 为真实 Linux Kernel Repair 实验平台，首先通过同状态受控实验研究 Agent 在单次修复任务中对历史诊断证据的有效利用，再依据实验结果探索任务内记忆介入与反馈驱动策略，在此过程中构建可复现的 Agent Memory 工程与评测系统。