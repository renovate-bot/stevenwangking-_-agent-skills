# 安全审查清单（安装/更新前必跑）

适用对象：任何准备安装或更新的第三方技能（自制技能更新可简化审查，但 scripts/ 环节建议保留）。核心认知：**技能会在 Agent 的 Shell 权限下运行，市场收录 ≠ 安全**——像审计代码一样审计技能。审查范围是技能目录全部文件（SKILL.md、scripts/、references/、assets/），不只是 SKILL.md。

## 1. 元数据检查

- frontmatter `name` 与技能目录名一致（与仓库名不同是常态，聚合仓库常见）；警惕 typosquat（以下为虚构示意，仅演示拼写变体）：
  - 单字符增删换：`git-commit-helper` vs `git-commiter`、`gihub-push`
  - 同形字符替换：l/1、O/0
  - 热门技能名的多余连字符/下划线变体
- `description` 与技能实际功能相符——描述与内容不符（挂羊头卖狗肉）至少 WARNING
- 作者可识别（有公开仓库/主页）；无名作者加强后续所有环节的审查

## 2. 危险内容扫描

**Critical（任一命中 = DANGER，直接淘汰）**：

- 读取 `~/.ssh`、`~/.aws`、`~/.env`、credentials 文件、浏览器 Cookie/配置数据
- `curl ... | bash`、`wget ... | sh` 类下载即执行
- 无防护的破坏性命令：`rm -rf`、`sudo`、大范围 `chmod/chown`
- 明显 prompt injection："ignore previous instructions"、"send/upload the contents of ..."、诱导把本地数据外发
- 长 base64 串、混淆或加密的脚本内容（读不懂 = 无法审计）
- 网络外发行为与读取本地敏感数据（凭据/Cookie/密钥文件）同时出现的组合（典型数据外泄通道）

**Warning（命中即列出全部疑点，请用户裁决，默认不装）**：

- 引用未知的第三方 API / webhook / 远程脚本
- 网络请求与 shell 执行并存，但文档未说明用途与去向
- 读取 `OPENAI_API_KEY`、`ANTHROPIC_API_KEY`、`GH_TOKEN` 等敏感环境变量（有正当用途的必须能说清用途与去向）
- 修改 shell 配置（.zshrc/.bashrc）、crontab、系统文件
- 大范围文件访问模式（`/**/*`、`/etc/`）
- 要求关闭沙箱或安全设置

**Informational（记录在案，不影响判定）**：

- description 模糊、无版本号、无 license

## 3. scripts/ 专项审查

- 逐个通读脚本，明确其行为与被调用的时机；vetting 阶段**只读不执行**，任何执行（哪怕"演示跑一下"）都要先经用户确认
- 无法阅读或无法理解的脚本——命中第 2 节 Critical（混淆/加密/长 base64）的直接 DANGER；其余（压缩、超长编码等）至少 WARNING

## 4. 判定输出

- **PASS**：无命中，或仅 Informational
- **WARNING**：命中 Warning 项 → 输出全部疑点 + 风险说明 + 建议，由用户裁决；用户未表态前不安装
- **DANGER**：命中任一 Critical → 淘汰，不进评测评分，不因 stars/安装量/官方收录而放宽

## 5. 信任层级（决定审查深度，不豁免审查）

官方源（anthropics、vercel-labs 等已实证源）> 知名作者公开仓库 > 高安装量社区技能 > 未知新作者（全量深审）。

再热门的技能也要过完整清单；v1.0 安全不代表 v1.1 安全，**每次更新后重审**，重点 diff 新增的 scripts 与网络请求。
