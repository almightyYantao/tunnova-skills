# Tunnova skills

给 Claude Code 用的两个 skill：

| 目录 | 用途 |
|---|---|
| [tunnova-plugin](tunnova-plugin/SKILL.md) | 为 Tunnova 隧道网关编写、打包、签名、测试和发布插件（`.tnp`） |
| [tunnova-component](tunnova-component/SKILL.md) | 组件热更新（`.tnc`）：开发、打包、影响评估、界面、实测和发布 |

## 安装

克隆到任意目录，再把两个文件夹软链接进 `~/.claude/skills/`：

```sh
git clone git@github.com:almightyYantao/tunnova-openwrt-sdwan-plugin-skill.git ~/.claude/skill-repos/tunnova-skills
ln -s ~/.claude/skill-repos/tunnova-skills/tunnova-plugin    ~/.claude/skills/tunnova-plugin
ln -s ~/.claude/skill-repos/tunnova-skills/tunnova-component ~/.claude/skills/tunnova-component
```

以后 `git pull` 就能更新。
