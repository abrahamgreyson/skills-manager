# CLAUDE.md — skills 仓库

## 仓库职能

这是个人 Claude Code Skill 的**唯一事实来源**。本仓库只做一件事：**管理技能本身**——编写、评估、迭代、演进。

把技能分发到不同项目和编码代理（Claude Code / Codex 等）是外部工具 **Skill Manager** 的职责，不属于本仓库。这里只产出"源"，交付由 Skill Manager 动态读取应用。

## 边界（明确不做）

- **不**在 `~/.claude/skills/`、`~/.codex/` 等 home 目录建立副本
- **不**在具体项目里内联 skill 副本
- **不**做 symlink 安装（旧 `README.md` 提到的 `ln -sf … ~/.claude/skills/` 模式已废弃，待清理）

> 副本和分发归 Skill Manager。在本仓库里建任何"安装到某处"的逻辑，都是越界。

## Skill 目录结构

每个 skill 是仓库根目录下的一个自包含子目录，目录名 kebab-case：

```
{skill-name}/
├── SKILL.md              # 必需。frontmatter + 给 AI 的编排指令
├── scripts/              # 可选。执行脚本（如 watch.mjs）
├── README.md             # 可选。给人看的说明（区别于给 AI 看的 SKILL.md）
└── evals/                # 可选但建议。评估材料（对齐 skill-creator）
    ├── trigger-eval.json # 触发评估用例（正例 + near-miss 反例），入库
    └── evals.json        # 行为评估用例（prompt + assertions），按需
```

## SKILL.md 规范

**frontmatter 字段**：

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | 是 | kebab-case，**与目录名一致** |
| `description` | 是 | 触发条件写足——这是 eval 优化的核心对象。把"什么时候该用 / 什么时候不该用"讲清楚 |
| `argument-hint` | 否 | 参数提示 |
| `allowed-tools` | 否 | 显式列出该 skill 需要的工具 |

**正文**：给 AI 的编排指令，不是给人看的文档。人在意的说明放 `README.md`。

## Eval 评估

评估使用 **skill-creator** 技能提供的机制（通过 Skill Manager 按需加载到会话，不内置在本仓库）。新建或重大修改一个 skill 时默认要做评估。

两套评估，按开销选：

| 评估 | 测什么 | 产物 | 开销 | 何时做 |
|------|--------|------|------|--------|
| **触发评估** | `description` 触发准不准 | `evals/trigger-eval.json` | 轻量 | **默认做** |
| **行为评估** | skill 执行任务的实际效果 | `evals/evals.json`（prompt + assertions） | 重量（subagent + 大量 API） | **按需** |

### 触发评估（默认）

编 `evals/trigger-eval.json`——一组真实感强的 query，标 `should_trigger`：

```json
[
  {"query": "给当前仓库打个 tag，这版做了 monorepo 重构", "should_trigger": true},
  {"query": "把 v2.0.0 这个 tag push 到 origin", "should_trigger": false}
]
```

- **正例 8-10 条**：覆盖同一意图的不同说法，含用户不直呼 skill 名的场景。
- **反例 8-10 条**：最值钱的是 **near-miss**——共享关键词但实际需要别的工具（如"push tag"≠"打 tag"）。明显无关的反例（如"写个 fibonacci"）测不出东西，不要。
- query 要**具体带上下文**（文件路径、版本号、commit hash、技术栈、背景），不要抽象。skill-creator 的原话：抽象的"打个 tag"是坏 query，带版本号和改动背景的才是好 query。

写好后用 skill-creator 的优化循环跑（自动 train/test 划分、每条 query 跑多次取触发率、多轮改写 description、按 held-out test 分数选最优防过拟合），把 `best_description` 写回 SKILL.md。`trigger-eval.json` 作为回归资产入库。

> **⚠️ 别在 skill 仓库内自测触发评估**：优化循环用临时 command 模拟 skill 触发，但 claude 会优先发现并读取仓库里的真 `SKILL.md`，临时 command 永不被调用 → 触发率全 0（假象，不是 description 的问题）。要在**不含该 skill 源的干净项目目录**下跑，让临时 command 成为唯一来源。验证手段：跑一个正例 query 看 `claude -p` 的工具序列，若它直接 `Read` 了真 `SKILL.md` 即说明中招——此时 skill 实际触发正常，是评估机制的盲区。

### 行为评估（按需）

测 skill 执行任务的实际效果。spawn subagent 跑 with_skill vs baseline，grader 按 assertions 打分，aggregate 成 benchmark 对比。开销大，API 额度紧张时省略，或只用人工跑几个 prompt 抽检。

> 两套评估的完整 JSON schema、grader/analyzer subagent、benchmark viewer 用法见 skill-creator 自身文档（`references/schemas.md`、`agents/`）。

## 新增 Skill 流程

1. 建目录 `{skill-name}/`（kebab-case）
2. 写 `SKILL.md`（`name` = 目录名）
3. 编 `evals/trigger-eval.json`
4. 用 skill-creator 跑触发评估，优化 `description`
5. （可选）补 `scripts/`、`README.md`、`evals/evals.json`
6. 提交（见下）

## 命名与提交

- **skill 名**：kebab-case；目录名、frontmatter `name`、commit scope 三者一致。
- **commit**：conventional commit，scope 用 skill 名。
  - `feat(tag): 支持按历史补多 tag`
  - `fix(supervisor): 消除误报和被动通知`
  - `refactor(supervisor): 自动发现 + 项目本地日志`
- **原子提交**：一个逻辑变更一个 commit（继承全局规范）。

## 与全局规范的关系

本仓库继承 `~/.claude/CLAUDE.md` 的通用规范（简洁优先、精准修改、目标驱动执行、编码前思考）。此处仅补充技能仓库特有的约定，不重复通用条款。
