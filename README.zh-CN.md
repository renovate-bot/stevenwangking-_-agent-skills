# stevenwangking agent skills

[![skills.sh](https://skills.sh/b/stevenwangking/agent-skills)](https://skills.sh/stevenwangking/agent-skills)

个人维护的 [Agent Skills](https://agentskills.io) 技能合集，可通过 [skills CLI](https://skills.sh)（`npx skills`）安装到多种编程智能体。English: [README.md](README.md)

## 设计哲学

仓库里每个技能都遵循同一个原则：**自动化带上守门**。

- **commit —— 有风险绝不默默提交。** 默认全自动执行；密钥、无关文件、依赖变更一律暂停等确认。
- **reply-polish —— 过度润色即失败。** 只做减法：保留说话人的语气和立场，去掉 AI 腔和公文腔。
- **simplify —— 行为零变更。** 每次简化都先经过 lint、类型或测试验证；证明不了安全就不动手。
- **skill-manager —— 安全是硬门槛，不是评分项。** 装不上验证不了完成就不许报成功；删不掉链接不许说干净。

## 技能列表

| 技能 | 说明 | 语言 |
| --- | --- | --- |
| [commit](skills/commit/SKILL.md) | 分析 Git 变更，生成符合规范的 commit message，主题不合时自动按功能拆分提交。 | 中文 |
| [reply-polish](skills/reply-polish/SKILL.md) | 润色已写好的中文职场回复，自然、专业、像作者本人。 | 中文 |
| [simplify](skills/simplify/SKILL.md) | 在不改行为的前提下简化近期改动的代码，提升清晰度与复用。 | English |
| [skill-manager](skills/skill-manager/SKILL.md) | Agent Skills 全生命周期管理：多源发现、去重溯源、四维评测（含安全硬门槛）、经 skills CLI 安装/更新/卸载、多 Agent 链接验证与状态诊断。 | 中文 |

### commit

检查变更 → 基于真实 diff 生成 message → 需要时按功能拆分 → 回报 hash。message 不虚构背景，拆分不做 hunk 级。

> 触发方式：`/commit` · "提交一下代码" · "帮我 commit" · `/commit amend`

### reply-polish

接收已写好的草稿（IM、邮件或评论），保留原意和说话方式，去掉 AI 腔与公文腔，按受众（领导/产品/后端/客户）校准分寸。默认只给一版可直接复制的成品。

> 触发方式："帮我改一下这段话" · "太官腔了" · "这样回复合适吗"

### simplify

只审查本次改动，从复用、质量、效率三个视角做克制简化，收尾前用 lint、类型或测试验证。

> 触发方式："simplify this" · "clean up the changes" · "refactor before commit"

### skill-manager

一条链路管到底：多源发现 → 去重溯源 → 四维评测（功能匹配、工程质量、安全硬门槛、实用性）→ 经 `npx skills` CLI 安装/更新/卸载 → canonical 与多 Agent 链接验证。安全是硬门槛判定（PASS / WARNING / DANGER），不是评分项；验证没跑完的安装不许报成功。

> 触发方式："找个适合 X 的技能" · "安装这个技能" · "检查技能状态" · "skill manager"

## 安装

```bash
# 安装全部技能
npx skills add stevenwangking/agent-skills

# 只安装某一个
npx skills add stevenwangking/agent-skills --skill commit
```

支持所有遵循 [Agent Skills 标准](https://agentskills.io) 的智能体——Claude Code、Codex、Cursor、ZCode 等 [70+ 款](https://github.com/vercel-labs/skills#supported-agents)。

## 维护

技能的唯一编辑源在作者本机的 `~/.agents/skills/`。本地修改后运行 `./sync.sh`——脚本自动拷贝变更的 `SKILL.md`、提交并推送。

## 许可

[MIT](LICENSE)
