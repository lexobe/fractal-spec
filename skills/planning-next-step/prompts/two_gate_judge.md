# 双门槛判定 (Two-Gate Judge)

本提示词是 Planning Next Step 的**核心决策逻辑**。
在生成 `new_goal` 后，通过两道门槛决定推进方式。

---

## 输入
*   `new_goal`: 下一步要执行的子目标（Core4 格式）。
*   `context`: 完整的 Context JSON（含 `think` 和 `path`）。
*   `executors.yaml`: 可用执行器能力列表（从 `.thinkon/executors/executors.yaml` 读取）。

## Gate 1: 认知判据 (Cognitive Criterion)

**核心问题**: 当前 `new_goal` 是否存在"必须判断但缺可验证证据"的无知（Gap）？

**判定要点**:
*   不能仅依赖关键词（如"查明/为什么"），但这些是强提示。
*   硬门槛："若继续推进只能猜"。

**若 Has Gap = true → `PLAN_PROBES`**:
*   生成 3-7 条 **hypotheses**（竞争性假设）。
*   假设必须**可证伪**（falsifiable）、尽量**可区分**（discriminable）。
*   **禁止使用顺序词**（先/再/然后/第一步/接下来）。
*   附带 `success_signal`: 何种信号代表"证据已足以裁决"。

**若 Has Gap = false → 进入 Gate 2**。

## Gate 2: 能力判据 (Capability Criterion)

仅当 Gate 1 通过（Has Gap = false）时评估。

**核心问题**: 该 `new_goal` 是否可被 executor **一次原子覆盖**？

**判定要点**:
*   需要多次工具调用、多文件大改动、或风险高需检查点 → 非原子。
*   无对应工具 / 能力描述不足 → 默认非原子。
*   参照 `executors.yaml` 中的 `guarantees` / `limits` / `side_effects` 判定。

**若 Atomic = true → `EXECUTE`**:
*   生成 `executor_call` 对象（command + inputs + expected_observations）。
*   `plan` 设为 `[]`。

**若 Atomic = false → `PLAN_STEPS`**:
*   生成 3-7 条 **steps**（顺序步骤）。
*   允许顺序词（先/再/然后）。
*   **禁止使用猜测词**（可能/也许/原因/假设）。
*   附带 `success_signal`: 何种信号代表"步骤链推进到满足 metric 的交付物"。

## 语义约束

| 约束 | 要求 |
| :--- | :--- |
| **混杂禁止** | PLAN_PROBES 的 plan 必须是纯 hypotheses（无步骤语义）；PLAN_STEPS 的 plan 必须是纯 steps（无假设语义）。若存在混杂风险，回退到 PLAN_PROBES。 |
| **反死循环** | 若 `done` 中已存在同粒度失败尝试，不允许输出完全相同的计划项或执行指令。必须改为：更细的 steps、新的 hypotheses、或补充证据策略。 |
