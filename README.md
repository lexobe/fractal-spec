# Skills

> **Agent Skills 开发仓库** — 标准化、模块化的 Agent 能力包集合。

每个 Skill 是一个独立的目录，包含 `SKILL.md` 入口文件和可选的 `prompts/`、`knowledge/`、`scripts/` 子目录。

## 目录结构

```
.
├── skills/                          # 所有 Skill 包
│   ├── planning-next-step/          # Thinkon Planner — 分形规划器
│   ├── skills-engineer/             # Skills Engineer — Skill 创建/转换工具
│   ├── recursive-core/              # Recursive Core — 递归核心能力
│   └── fractal-spec/                # Fractal Spec — 分形规范驱动开发
│       ├── SKILL.md
│       └── commands/                # /spec /plan /do /check /refactor /score
│
├── resources/                       # 原始源文件、参考素材
│   └── planner.md                   # Thinkon Planner System Prompt 源文件
│
└── references/                      # 外部参考项目
    ├── planning-with-files/
    └── spec-kit/
```

## Skills 总览

| Skill | 描述 | 版本 |
| :--- | :--- | :--- |
| **planning-next-step** | Thinkon/FIOS 规划处理器，双门槛判定，输出 plan-next JSON | v1.0.0 |
| **skills-engineer** | Agent Skill 创建/转换工具，支持 Create + Convert 双模式 | v1.2.0 |
| **recursive-core** | 递归核心能力单元 | v1.0.0 |
| **fractal-spec** | Spec-Driven 开发工作流（/spec → /plan → /do → /check） | v1.0.0 |

## Skill 标准结构

```
skill-name/
├── SKILL.md              # 入口文件（YAML Frontmatter + 触发/工作流/约束）
├── prompts/              # 子提示词（逻辑层）
├── knowledge/            # 知识文件（概念定义、协议、框架）
└── scripts/              # 辅助脚本（可选）
```

## 快速开始

1. 浏览 `skills/` 目录选择需要的 Skill。
2. 阅读对应 `SKILL.md` 了解触发条件和使用方式。
3. 在 Agent 配置中引用对应 Skill 目录。
