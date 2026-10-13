# Tunnova skills

给 Claude Code 用的 skill 和子 Agent。

## Skill

| 目录 | 用途 |
|---|---|
| [tunnova-plugin](tunnova-plugin/SKILL.md) | 为 Tunnova 隧道网关编写、打包、签名、测试和发布插件（`.tnp`） |
| [tunnova-component](tunnova-component/SKILL.md) | 组件热更新（`.tnc`）：开发、打包、影响评估、界面、实测和发布 |

## 子 Agent

| 文件 | 用途 |
|---|---|
| [agents/tunnova-dev.md](agents/tunnova-dev.md) | 产品开发与发版：固件、Cloud、插件、组件、管理后台、测试、发版 |
| [agents/tunnova-biz.md](agents/tunnova-biz.md) | 商务：报价、定价测算、合同、竞品、合规、公司事务、供应商、硬件选型 |

## 安装

克隆到任意目录，再把 skill 文件夹软链接进 `~/.claude/skills/`，Agent 文件软链接进 `~/.claude/agents/`：

```sh
git clone git@github.com:almightyYantao/tunnova-skills.git ~/.claude/skill-repos/tunnova-skills
ln -s ~/.claude/skill-repos/tunnova-skills/tunnova-plugin    ~/.claude/skills/tunnova-plugin
ln -s ~/.claude/skill-repos/tunnova-skills/tunnova-component ~/.claude/skills/tunnova-component
ln -s ~/.claude/skill-repos/tunnova-skills/agents/tunnova-dev.md ~/.claude/agents/tunnova-dev.md
ln -s ~/.claude/skill-repos/tunnova-skills/agents/tunnova-biz.md ~/.claude/agents/tunnova-biz.md
```

以后 `git pull` 就能更新。
