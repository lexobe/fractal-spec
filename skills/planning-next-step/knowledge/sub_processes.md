# 子处理阶段定义

本文件定义了 Planner 在单次调用内部依次完成的子处理逻辑。
它们通过 SKILL.md 的工作流 Phase 引用，不是独立的 API 调用。

> **决策流程参考**（伪代码，帮助理解整体思考流程）：
> ~~~pseudo
> function planner_step(context):
>   ended = isend_thinkon(context)       # Phase 1: 内部推理参考
>   new_goal = make_next(context)         # Phase 2: 生成下一步目标
>   plan_type, plan_items, executor_call = two_gate_judge(new_goal, context)  # Phase 3
>   update_plan = update_current_plan(new_goal, plan_items, context)          # Phase 4
>   return {
>     "type": "plan-next",              # 规划阶段唯一合法输出
>     "plan_type": plan_type,
>     "new_block": {"goal": new_goal, "plan": plan_items, "done": []},
>     "executor_call": executor_call,    # 仅 EXECUTE
>     "update_plan": update_plan          # 可选
>   }
> ~~~

## 1. isend_thinkon(context) — 完成判定

**任务**: 判断当前 block 是否已完成（仅作为**内部推理参考**）。

**判定条件**（Core4 视角）：
*   `done` 是否已从 Intent / Deliverable / Metric / Constraint 四个维度覆盖 `goal`。
*   在现有约束下是否无法安全继续推进（如目标冲突、缺乏必要上下文等）。

**输出**: `ended: boolean`（内部使用，不直接影响输出格式）

**重要约束 — Runtime 输出类型契约**:
*   `plan-next`: 规划阶段的**唯一合法输出**。必须包含合法 `new_block`（含非空 goal、plan list、done list），否则 Runtime 报错。
*   `plan-return`: **仅在执行阶段**被 Runtime 接受，必须包含 `result`（string 或 object），否则 Runtime 报错。
*   其他 type 或阶段不匹配：Runtime 报错。

因此，无论 `isend_thinkon` 判定结果如何，Planner **必须**输出合法的 `plan-next`。`isend_thinkon` 的判定结果仅用于辅助 Planner 决定 `new_block` 的内容策略（例如：当判定接近完成时，可以生成一个验证/收尾性质的目标）。

---

## 2. make_result(context) — 结果构建

仅在 `ended == true` 时在内部使用。

**任务**: 将当前 block 的完成情况总结为 Core4 风格的 `result` 文本。

**输出结构**（Markdown 格式）：
1.  **完成情况**: 已完成 / 部分完成 / 未完成。未完成需说明阻塞原因。
2.  **提交物**:
    *   工件交付型 → 罗列文件名和极简说明。
    *   文本交付型 → 直接给出结论文本。
3.  **指标情况**: 对照 Metric 说明通过/未通过的依据。
4.  **约束符合性**: 是否符合约束，有偏离需说明原因。

**示例 A（工件交付型）**:
~~~
### 完成情况
已完成：已生成求解脚本。
### 提交物
- solve.py：支持回溯算法。
### 指标情况
- 通过：check.sh 退出码 0。
### 约束符合性
符合。
~~~

**示例 B（文本交付型）**:
~~~
### 完成情况
已完成：完成代码评审。
### 提交物
- 评审结论：逻辑清晰但缺注释。
### 指标情况
- 依据：人工走查。
### 约束符合性
符合。
~~~

---

## 3. make_next(context) — 下一步目标生成

**任务**: 生成下一步要执行的子 block 的 `new_goal`。

**行为规则**:
*   优先选择仍未在 `done` 中体现的计划项。
*   将该计划项重写为一个更清晰、可验收的 Core4 目标。
*   **默认策略**: 生成的 `new_block` 的 `plan` 默认为 `[]`，让子 block 进入执行阶段（除非 Two-Gate 判定需要拆分）。
*   若 `done` 中已有类似目标的失败记录，**必须**拆分为更细粒度的子步骤，不可重试同一粒度。

**输出**: `new_goal: string | Core4Object`

---

## 4. update_current_plan(new_goal, plan_items, context) — 计划调整

**任务**: 根据 `done` 中的执行反馈，决定是否调整剩余 `plan`。

**默认行为**: 不返回 `update_plan`，Runtime 自动 FIFO 消费 `plan` 首项。

**语义**: `update_plan` 是**全量覆盖**——它表示"接下来要做的全部计划"，与已完成的 `done` 无关。
Runtime 会用 `update_plan` **直接替换**当前 block 的整个 `plan` 字段。

**需要调整的场景**:
*   发现依赖关系，需要重新排序。
*   根据执行结果，需要添加/删除步骤。
*   对剩余计划项做细化或重写，以更好对齐 Core4。

**示例**: 若原 `plan = [P1, P2, P3]`，P1 已完成，发现需要调整：
*   `update_plan: [P2_modified, P3, P4_new]` → 替换整个 `plan`。

**输出**: `update_plan: List[str] | null`

---

## 5. make_plan_items(plan_type, new_goal, context) — 计划项生成

根据 `plan_type` 生成对应的 `plan` 内容：

### PLAN_PROBES
*   生成 3–7 条 hypotheses。
*   每条假设应可被验证或证伪。
*   附带 `success_signal`: 何种信号代表“证据已足以裁决”。

### PLAN_STEPS
*   生成 3–7 条 steps。
*   每一步可改写为一个可执行的子 goal（Core4）。
*   附带 `success_signal`: 何种信号代表“步骤链推进到满足 metric 的交付物”。

### EXECUTE
*   `plan` 设为 `[]`。
*   生成 `executor_call` 对象：

~~~json
{
  "command": "shell: <command>" ,
  "inputs": { },
  "expected_observations": [
    "退出码/错误码",
    "生成的文件/变更点",
    "日志摘要或检查项"
  ]
}
~~~
