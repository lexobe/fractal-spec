---
name: skills-engineer
description: 当用户需要设计、生成或优化一个 Agent Skill 时使用此技能。支持从零创建（Create）或从已有 Prompt/文档转换（Convert）为完整的 Skill 目录结构，并内置安全审查和质量优化闭环。
version: 1.2.0
author: AI Architect
---

# Skills Engineer

**核心能力**: 将用户意图转化为符合 Agent Skills 开放标准的完整 Skill 包（目录结构 + 文件内容）。

## 1. 触发条件
*   用户明确要求设计、生成或优化一个 **Agent Skill**。
*   用户提供了一个已有的 Prompt / System Prompt / 文档，要求将其**转换**为标准 Skill 包。
*   当前任务过于复杂，需要拆解为独立的、可复用的 Agent 能力单元。
*   需要将业务逻辑固化为标准化工作流。

## 2. 知识库
*   **Skill 设计模式**: `knowledge/taxonomy.md`
*   **最佳实践**: `knowledge/best_practices.md`

## 3. 工作流

### Phase 1: 理解与规划 (Architect)
1.  **分析意图**: 识别用户的核心目标、输入/输出、约束条件。
2.  **判定工作模式**:
    *   **Create 模式**: 无源文件，从零构建。
    *   **Convert 模式**: 有源文件（已有 Prompt/文档），需拆分并保真转换。
3.  **架构设计**: 调用 `prompts/planner.md` 决定 Skill 的文件结构和组件类型。
    *   需要完整目录还是单文件？
    *   需要辅助脚本还是纯提示词？
    *   需求是否足够清晰？（不清晰时必须先澄清，不可强行设计。）

### Phase 2: 构建 (Build)
1.  **生成文件**: 调用 `prompts/alpha_generator.md`，按照规划生成所有文件内容。
2.  **结构校验**: 确保生成的 `SKILL.md` 包含正确的 YAML Frontmatter 和触发描述。

### Phase 3: 质量增强 (Quality Control)
1.  **评审**: 调用 `prompts/omega_optimizer.md` 对生成结果进行 4 阶段评审。
    *   **Phase 1 — 量化打分**: 标准合规、指令质量、架构合理、健壮性、安全性（+ Convert 模式的源保真度）
    *   **Phase 2 — 歧义检测**: 扫描边界行为未定义、隐含偏好、语义二义性、默认行为不清晰等歧义，整理为 A/B 选择题提交用户裁决（**阻塞步骤**）
    *   **Phase 3 — 修正**: 针对低分项和用户裁决结果执行修正
    *   **Phase 4 — 交付**: 输出最终 Skill 文件 + Changelog
2.  **修正**: 若评审总分低于阈值（Create: 20/25, Convert: 24/30），强制回退 Phase 2 重写。

## 4. 交付物
*   完整的 Skill 目录结构图。
*   所有核心文件的代码块（`SKILL.md`, `prompts/*`, `scripts/*`）。
*   简短的使用说明（触发方式、变量填写指引）。

## 5. 约束
*   **不可**在需求模糊时强行生成——必须先提出澄清问题。
*   **不可**将多个不相关能力捆绑进一个 Skill。
*   **不可**在任何文件中硬编码敏感信息（API Key、密码等）。
