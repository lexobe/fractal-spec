---
name: fractal-spec
description: 基于分形架构的规范驱动开发工作流。通过 /spec → /plan → /do → /check 闭环，确保需求→规范→代码→验证的全流程可追溯。
version: 1.0.0
author: Thinkon Team
---

# Fractal Spec

**核心能力**: Spec-Driven 开发工作流编排器，将用户需求转化为可执行的分形规范，并驱动代码实现与质量验证。

## 1. 触发条件
*   用户使用 `/spec`、`/plan`、`/do`、`/check`、`/refactor`、`/score` 任一命令。
*   需要将需求文档化为严格的分形规范结构。

## 2. 命令映射
*   `/spec` → `commands/fractal.spec.md` — 将需求转化为 Spec 文档
*   `/plan` → `commands/fractal.plan.md` — 基于 Git Diff 生成实施计划
*   `/do` → `commands/fractal.do.md` — 执行代码实现
*   `/check` → `commands/fractal.check.md` — 验证并提交
*   `/refactor` → `commands/fractal.refactor.md` — 治理 Spec 结构
*   `/score` → `commands/fractal.score.md` — 6 维质量评分

## 3. 工作流
```
User Input → /spec → specs/*.md → git diff → /plan → tasks.md → /do → Code → /check → git commit
```

## 4. 约束
*   **Spec First**: 严禁跳过 `/spec` 直接写代码。
*   **原子化命名**: 文件名片段必须是单英文单词。
*   **分形生长**: 任何 Spec > 300 行时必须拆分为子 Spec。
