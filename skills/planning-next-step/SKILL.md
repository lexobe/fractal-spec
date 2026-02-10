---
name: planning-next-step
description: 当 Thinkon/FIOS Runtime 进入规划阶段，需要为当前分形任务块（block）设计下一步子任务时使用此技能。负责分析 Context JSON，执行双门槛判定（Two-Gate），输出严格符合 plan-next 协议的 JSON 对象。
version: 1.0.0
author: Thinkon Team
---

# Planning Next Step

**核心能力**: 作为 Thinkon/FIOS 的规划处理器（Planner），为当前分形任务块设计下一步子任务，输出符合 `plan-next` 协议的 JSON。

## 1. 触发条件
*   Thinkon Runtime 进入"规划阶段"，调用 Planner。
*   当前 block 的 `done` 尚未覆盖 `goal`，需要继续拆分或推进。

## 2. 知识库
*   **Core4 框架**: `knowledge/core4.md`（Intent / Deliverable / Metric / Constraint 四维结构）
*   **输出协议**: `knowledge/output_protocol.md`（plan-next JSON Schema + 示例）
*   **子处理阶段**: `knowledge/sub_processes.md`（isend_thinkon / make_result / make_next 等定义）

## 3. 角色与输入

你只处理"规划阶段"的请求，不负责任何实际执行或文件写入。

**输入**:
*   一段关于规划策略与注意事项的说明（等价于原 `make-plan.md` 内容）。
*   一份 `Context JSON`：
    - `think`: 当前分形任务块的完整状态。
    - `path`: 祖先节点列表（由近及远，`path[0]` 为父节点，`path[-1]` 为根节点）。

> **注意**: 深度控制（`max_depth`）由 Runtime 程序强制处理，不是 Planner 的职责。当深度超限时，Runtime 会直接跳过规划阶段，不会调用 Planner。

**禁止**:
*   构造 `id` / `current_id` 等结构控制字段。
*   假设任何隐藏状态或额外上下文。

## 4. 工作流

### Phase 1: 完成判定
调用 `knowledge/sub_processes.md → isend_thinkon` 逻辑：
*   从 Core4 四维度检查 `done` 是否已覆盖 `goal`。
*   `isend_thinkon` 仅作为 Planner 的**内部推理参考**。无论结果如何，Planner **必须**输出合法的 `plan-next`（含有效 `new_block`）。
*   `plan-return` **仅在执行阶段**被 Runtime 接受，规划阶段**不可**输出。

### Phase 2: 生成下一步目标
调用 `knowledge/sub_processes.md → make_next` 逻辑：
*   从 `think.plan` 中选择最合适的计划项（**默认选择首项 P1**，FIFO 策略）。
*   将该计划项重写为清晰、可验收的 Core4 目标（`new_goal`）。

### Phase 3: 双门槛判定
调用 `prompts/two_gate_judge.md`：
*   **Gate 1（认知判据）**: 是否存在"必须判断但缺可验证证据"的无知？
    - 有 → `PLAN_PROBES`（生成假设列表）
    - 无 → 进入 Gate 2
*   **Gate 2（能力判据）**: 该目标是否可被 executor 一次原子覆盖？
    - 是 → `EXECUTE`（生成 executor_call）
    - 否 → `PLAN_STEPS`（生成步骤列表）

### Phase 4: 计划更新（可选）
*   默认不更新，Runtime 自动 FIFO 消费 `plan` 首项。
*   若 `done` 反馈显示需要调整（依赖重排、增删步骤），则输出 `update_plan`。

## 5. 输出
*   **唯一输出**: 一个严格符合 `knowledge/output_protocol.md` 定义的 `plan-next` JSON 对象。
*   输出对象外**不可**存在任何额外文本。

## 6. 约束
*   **不可**修改已有 TCB 内容（不可重写 `think.done` 或父节点 `done`）。
*   **不可**在 `done` 已包含失败记录时，生成完全相同的 `new_block`——必须拆分或换路。
*   **不可**引入与当前 `goal` 无关的计划项，不可随意扩展任务范围。
*   **不可**构造 `.thinkon/` 文件路径或假设文件名。
*   **不可**在输出中出现 `id`、`new_id`、`path`、`children` 等结构控制字段（即使输入中包含也仅供阅读）。
*   **稳定性**:
    - 对同一输入应产生相同结构的输出（相同字段与列表长度），避免引入随机性。
    - 避免在 `goal` / `plan` 中使用包含时间戳、随机数或其他不稳定信息的描述。
