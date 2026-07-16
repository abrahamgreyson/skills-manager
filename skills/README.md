# Skills

个人 Claude Code / Codex 技能的**唯一事实来源**。

本仓库只管"源"——编写、评估、迭代技能本身。把技能分发到具体项目和编码代理（Claude Code、Codex 等）由外部工具 **Skill Manager** 动态完成；本仓库不做任何安装或软链接。

## 现有技能

| 技能 | 说明 |
|------|------|
| [`supervisor`](./supervisor/) | 通过 tmux 管理 Worker agent 会话——自动批准权限、响应交互问题，无人值守 |
| [`tag`](./tag/) | 分析变更、判断 SemVer 版本、打 annotated git tag 做发版标记 |

## 开发规范

新增或修改技能的完整约定见 [CLAUDE.md](./CLAUDE.md)。要点：

- 每个技能是根目录下的自包含子目录（kebab-case）
- `SKILL.md` 必需；`scripts/`、`README.md`、`evals/` 可选
- 评估使用 **skill-creator** 技能提供的机制（通过 Skill Manager 按需加载）
- 分发归 Skill Manager，不在此仓库内安装
