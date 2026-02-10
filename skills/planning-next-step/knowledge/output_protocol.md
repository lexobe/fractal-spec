# plan-next 输出协议

本文件定义了 Planner 的**唯一合法输出格式**。
输出必须是一个严格符合以下 Schema 的 JSON 对象，对象外不可存在任何额外文本。

---

## JSON Schema

~~~json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "plan_type", "new_block"],
  "properties": {
    "type": {"const": "plan-next"},
    "plan_type": {"enum": ["PLAN_PROBES", "PLAN_STEPS", "EXECUTE"]},
    "new_block": {
      "type": "object",
      "required": ["goal", "plan", "done"],
      "properties": {
        "goal": {
          "oneOf": [
            {"type": "string", "minLength": 1},
            {
              "type": "object",
              "required": ["intent", "deliverable", "metric", "constraint"],
              "properties": {
                "intent": {"type": "string"},
                "deliverable": {"type": "string"},
                "metric": {"type": "string"},
                "constraint": {"type": "string"}
              }
            }
          ]
        },
        "plan": {"type": "array", "items": {"type": "string"}},
        "done": {"type": "array", "maxItems": 0}
      },
      "additionalProperties": false
    },
    "success_signal": {"type": "string"},
    "executor_call": {
      "type": "object",
      "properties": {
        "command": {"type": "string"},
        "inputs": {"type": "object"},
        "expected_observations": {"type": "array", "items": {"type": "string"}}
      }
    },
    "update_plan": {"type": "array", "items": {"type": "string"}}
  },
  "additionalProperties": false
}
~~~

---

## 字段约束

| 字段 | 约束 |
| :--- | :--- |
| `type` | **必须**为 `"plan-next"`。Runtime 在规划阶段**仅接受** `plan-next`；`plan-return` 仅在执行阶段接受。类型不匹配时 Runtime 报错。 |
| `plan_type` | **必须**为 `"PLAN_PROBES"` / `"PLAN_STEPS"` / `"EXECUTE"` 三者之一。 |
| `new_block` | **必须**存在。**仅**包含 `goal` / `plan` / `done`，**不可**包含 `id`、`new_id`、`path`、`children` 等结构控制字段。 |
| `new_block.goal` | 非空字符串或 Core4 对象（含 intent/deliverable/metric/constraint）。 |
| `new_block.plan` | 字符串数组。PLAN_PROBES 时为假设列表（禁顺序词）；PLAN_STEPS 时为步骤列表（禁猜测词）；EXECUTE 时**必须**为 `[]`。 |
| `new_block.done` | **必须**为空数组 `[]`。 |
| `success_signal` | PLAN_PROBES / PLAN_STEPS 时**应当**存在。描述可进入下一步的信号。 |
| `executor_call` | 仅 EXECUTE 时**必须**存在。含 `command` / `inputs` / `expected_observations`。不可编造未提供的工具。 |
| `update_plan` | 可选。若存在，**必须**为字符串数组。语义为**全量覆盖**——表示"接下来要做的全部计划"，Runtime 用其直接替换当前 block 的整个 `plan` 字段。 |

---

## 全局输出约束

*   输出中 **MUST NOT** 出现任何用于 TCB 结构或文件路径控制的字段（如 `id`、`new_id`、`path`、`children` 等）。即使这些字段出现在输入 Context 中，也只能被阅读，不能被复制或改写到输出。
*   输出对象外 **MUST NOT** 存在任何额外文本（包括解释、日志、前后缀或 Markdown 代码块）。
*   不得编造未在 `executors.yaml` 中提供的工具；若能力缺失，用通用形式表达（如 `"shell: <command>"`）。

---

## 示例

### 示例 A: PLAN_PROBES（探针/假设）

~~~json
{
  "type": "plan-next",
  "plan_type": "PLAN_PROBES",
  "new_block": {
    "goal": {
      "intent": "定位登录报错原因",
      "deliverable": "根因分析报告",
      "metric": "能复现并解释错误",
      "constraint": "不修改生产数据"
    },
    "plan": [
      "假设：鉴权服务返回 401，Token 过期或无效",
      "假设：API 网关超时，后端服务未响应",
      "假设：前端请求参数格式错误",
      "假设：数据库连接池耗尽导致查询失败"
    ],
    "done": []
  },
  "success_signal": "能通过日志或测试复现并确认至少一个假设"
}
~~~

### 示例 B: PLAN_STEPS（步骤拆分）

~~~json
{
  "type": "plan-next",
  "plan_type": "PLAN_STEPS",
  "new_block": {
    "goal": {
      "intent": "实现登录页面前端",
      "deliverable": "login.html + login.css + login.js",
      "metric": "表单可提交且有基本校验",
      "constraint": "不引入前端框架"
    },
    "plan": [
      "创建 login.html 骨架结构",
      "编写 login.css 样式",
      "实现 login.js 表单验证逻辑",
      "集成测试"
    ],
    "done": []
  },
  "success_signal": "login.html 可在浏览器正常渲染，表单提交触发校验"
}
~~~

### 示例 C: EXECUTE（直接执行）

~~~json
{
  "type": "plan-next",
  "plan_type": "EXECUTE",
  "new_block": {
    "goal": {
      "intent": "创建 login.html 骨架",
      "deliverable": "login.html 文件",
      "metric": "包含表单元素",
      "constraint": "HTML5 语法"
    },
    "plan": [],
    "done": []
  },
  "executor_call": {
    "command": "shell: cat > login.html << 'EOF'\n<!DOCTYPE html>...\nEOF",
    "inputs": {},
    "expected_observations": [
      "退出码 0",
      "login.html 文件存在",
      "文件包含 <form> 标签"
    ]
  }
}
~~~
